# 后台线程
redis后台常驻线程目前有3个，分别是关闭file，aof，lazy_free。  
每个线程有自己的锁，条件变量和job。
```
static pthread_t bio_threads[BIO_NUM_OPS];
static pthread_mutex_t bio_mutex[BIO_NUM_OPS];
static pthread_cond_t bio_newjob_cond[BIO_NUM_OPS];
static pthread_cond_t bio_step_cond[BIO_NUM_OPS];
// 3个list，装job用的
static list *bio_jobs[BIO_NUM_OPS];
// 挂起的作业数量
static unsigned long long bio_pending[BIO_NUM_OPS];

// 要执行的工作
struct bio_job {
    time_t time; // job创建时间
    int fd; // close file和aof 的参数 文件句柄
    lazy_free_fn *free_fn; // lazy_free的释放函数
    void *free_args[]; // lazy_free的释放函数的参数
};
```
后台线程的主函数
```
void *bioProcessBackgroundJobs(void *arg) {
    struct bio_job *job;
    unsigned long type = (unsigned long) arg;
    sigset_t sigset;

    // 只支持3个
    if (type >= BIO_NUM_OPS) {
        serverLog(LL_WARNING,
            "Warning: bio thread started with wrong type %lu",type);
        return NULL;
    }

    switch (type) {
    case BIO_CLOSE_FILE:
        redis_set_thread_title("bio_close_file");
        break;
    case BIO_AOF_FSYNC:
        redis_set_thread_title("bio_aof_fsync");
        break;
    case BIO_LAZY_FREE:
        redis_set_thread_title("bio_lazy_free");
        break;
    }

    // 设置cpu亲和，线程可终止，定时器信号等 (todo:了解的不深入)
    redisSetCpuAffinity(server.bio_cpulist);
    makeThreadKillable();
    pthread_mutex_lock(&bio_mutex[type]);
    /* Block SIGALRM so we are sure that only the main thread will
     * receive the watchdog signal. */
    sigemptyset(&sigset);
    sigaddset(&sigset, SIGALRM);
    if (pthread_sigmask(SIG_BLOCK, &sigset, NULL))
        serverLog(LL_WARNING,
            "Warning: can't mask SIGALRM in bio.c thread: %s", strerror(errno));

    while(1) {
        listNode *ln;

        // 等待新工作
        if (listLength(bio_jobs[type]) == 0) {
            pthread_cond_wait(&bio_newjob_cond[type],&bio_mutex[type]);
            continue;
        }
        // 取一个job出来
        ln = listFirst(bio_jobs[type]);
        job = ln->value;
        // 可以解锁了 因为线程要执行一个独立工作了
        pthread_mutex_unlock(&bio_mutex[type]);

        /* Process the job accordingly to its type. */
        if (type == BIO_CLOSE_FILE) {
            // close系统调用就完事了
            close(job->fd);
        } else if (type == BIO_AOF_FSYNC) {
            // fd可能被关了，被主线程拿来用作其它用途，忽略这些错误，aof并不会真正的失败
            if (redis_fsync(job->fd) == -1 &&
                errno != EBADF && errno != EINVAL)
            {
                int last_status;
                atomicGet(server.aof_bio_fsync_status,last_status);
                atomicSet(server.aof_bio_fsync_status,C_ERR);
                atomicSet(server.aof_bio_fsync_errno,errno);
                if (last_status == C_OK) {
                    serverLog(LL_WARNING,
                        "Fail to fsync the AOF file: %s",strerror(errno));
                }
            } else {
                atomicSet(server.aof_bio_fsync_status,C_OK);
            }
        } else if (type == BIO_LAZY_FREE) {
            // 释放
            job->free_fn(job->free_args);
        } else {
            serverPanic("Wrong job type in bioProcessBackgroundJobs().");
        }
        zfree(job);

        // 循环再次开始的时候锁住
        pthread_mutex_lock(&bio_mutex[type]);
        // 删除完成的作业
        listDelNode(bio_jobs[type],ln);
        // 挂起的少了一个
        bio_pending[type]--;

        // 做完一个作业 给大家说一下
        pthread_cond_broadcast(&bio_step_cond[type]);
    }
}

// 提交作业
void bioSubmitJob(int type, struct bio_job *job) {
    job->time = time(NULL);
    // 加锁 往作业队列push一个作业，通知有新作业了 解锁
    pthread_mutex_lock(&bio_mutex[type]);
    listAddNodeTail(bio_jobs[type],job);
    bio_pending[type]++;
    pthread_cond_signal(&bio_newjob_cond[type]);
    pthread_mutex_unlock(&bio_mutex[type]);
}

// 返回还没写的作业数量
unsigned long long bioPendingJobsOfType(int type) {
    unsigned long long val;
    pthread_mutex_lock(&bio_mutex[type]);
    val = bio_pending[type];
    pthread_mutex_unlock(&bio_mutex[type]);
    return val;
}

// 这个函数是为了其它线程等bio做完作业用的，但是现在没有用到
unsigned long long bioWaitStepOfType(int type) {
    unsigned long long val;
    pthread_mutex_lock(&bio_mutex[type]);
    val = bio_pending[type];
    if (val != 0) {
        pthread_cond_wait(&bio_step_cond[type],&bio_mutex[type]);
        val = bio_pending[type];
    }
    pthread_mutex_unlock(&bio_mutex[type]);
    return val;
}
```
## 总结
bio是非常简单的后台线程，每个线程有一个自己的作业队列，没有作业久等新作业，有作业久做作业，做完一个作业通知一下就可以了。不支持通知作业的创建者做完作业了（bioPendingJobsOfType太粗糙了，作业队列那么多，你不知道是不是你创建的作业被完成了，当然，可以通过作业列表的数量来查看）。redis后台线程主要是关闭文件io和释放内存，关闭文件系统调用速度比较慢，可能会导致服务卡，future可能会使用libeio
