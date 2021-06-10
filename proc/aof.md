# 持久化
[redis持久化](https://redis.io/topics/persistence)官网说明
## aof
aof是redis持久化的一种方式，其是为了redis的数据恢复，首先看一下aof相关变量（server中关于aof的变量）
```
int aof_enabled;                // aof开关
int aof_state;                  /* 状态 AOF_(ON|OFF|WAIT_REWRITE) */
int aof_fsync;                  // 策略 每秒 每query
char *aof_filename;             // aof文件名
int aof_no_fsync_on_rewrite;    // 有子进程重写的时候不进行刷数据到磁盘
// 重写阈值，当前aof文件/启动时或上次重写后aof文件 > aof_rewrite_perc 则重写
int aof_rewrite_perc;           
off_t aof_rewrite_min_size;     // redis重写的最小标准
off_t aof_rewrite_base_size;    // 上次重写后的大小
off_t aof_current_size;         // 当前aof write 大小
off_t aof_fsync_offset;         // aof fsync大小
int aof_flush_sleep;            // test
int aof_rewrite_scheduled;      /* Rewrite once BGSAVE terminates. */
list *aof_rewrite_buf_blocks;   /* Hold changes during an AOF rewrite. */
sds aof_buf;                    // buffer 在事件循环的开始运行
int aof_fd;                     // aof文件
int aof_selected_db;            // aof当前dbid
time_t aof_flush_postponed_start;  // 上次延迟flush的时间
time_t aof_last_fsync;          // 上次调用fsync的时间
time_t aof_rewrite_time_last;   // 上次重写使用的时间
time_t aof_rewrite_time_start;  // 当前重写开始的时间
int aof_lastbgrewrite_status;   /* C_OK or C_ERR */
unsigned long aof_delayed_fsync;  // 延迟fsync的次数
int aof_rewrite_incremental_fsync;// 重写的时候递增fsync  默认写32M fsync一下
int aof_last_write_status;      // 写入状态
int aof_last_write_errno;       // 有错无的时候 错误代码
int aof_load_truncated;         // aof文件被截断的时候，尽可能加载更多数据
int aof_use_rdb_preamble;       // 使用rdb写aof
redisAtomic int aof_bio_fsync_status; // bio fsync的状态
redisAtomic int aof_bio_fsync_errno;  // bio 错误码

/* AOF pipes used to communicate between parent and child during rewrite. */
int aof_pipe_write_data_to_child;     // 成对 用来重写子进程与父进程通信
int aof_pipe_read_data_from_parent;
int aof_pipe_write_ack_to_parent;
int aof_pipe_read_ack_from_child;
int aof_pipe_write_ack_to_child;
int aof_pipe_read_ack_from_parent;
int aof_stop_sending_diff;     // 父进程是否给重写子进程发送新的数据
sds aof_child_diff;             // 子进程中存储父进程发来的新数据
```
### aof flush
aof在redis数据被改变是才需要写，如果仅仅是读取，不影响数据，aof完全不需要写（与主从一模一样，保持数据的一致性）。因此当有命令改变数据的时候会写aof，在cmd.md:63行就是通知主从的时候，也通知aof写
```
void feedAppendOnlyFile(struct redisCommand *cmd, int dictid, robj **argv, int argc) {
    sds buf = sdsempty();
    // 首先检查数据库id对不对
    if (dictid != server.aof_selected_db) {
        char seldb[64];
        snprintf(seldb,sizeof(seldb),"%d",dictid);
        buf = sdscatprintf(buf,"*2\r\n$6\r\nSELECT\r\n$%lu\r\n%s\r\n",
            (unsigned long)strlen(seldb),seldb);
        server.aof_selected_db = dictid;
    }

    // 这一段就是重写命令，将命令转成协议写入buf
    ......

    // 加入aof的buf
    if (server.aof_state == AOF_ON)
        server.aof_buf = sdscatlen(server.aof_buf,buf,sdslen(buf));

    // 如果有子进程在重写aof，一并放入
    if (server.child_type == CHILD_TYPE_AOF)
        aofRewriteBufferAppend((unsigned char*)buf,sdslen(buf));

    sdsfree(buf);
}
```
每次命令会改变数据的都会调用上面函数写入aof_buf，在事件循环之前beforeSleep中刷此buf
```
void flushAppendOnlyFile(int force) {
    ssize_t nwritten;
    int sync_in_progress = 0;
    mstime_t latency;

    if (sdslen(server.aof_buf) == 0) {
        // aofbuf为0，但是有数据还没有fsync到磁盘上，尝试一次fsync
        if (server.aof_fsync == AOF_FSYNC_EVERYSEC &&
            server.aof_fsync_offset != server.aof_current_size &&
            server.unixtime > server.aof_last_fsync &&
            !(sync_in_progress = aofFsyncInProgress())) {
            goto try_fsync;
        } else {
            return;
        }
    }

    if (server.aof_fsync == AOF_FSYNC_EVERYSEC)
        sync_in_progress = aofFsyncInProgress();

    if (server.aof_fsync == AOF_FSYNC_EVERYSEC && !force) {
        // 后台线程有fsync
        if (sync_in_progress) {
            if (server.aof_flush_postponed_start == 0) {
                // 上次没有延时刷，这次等等，减少io，记录延迟刷的时间
                server.aof_flush_postponed_start = server.unixtime;
                return;
            } else if (server.unixtime - server.aof_flush_postponed_start < 2) {
                // 上次延迟刷的时间小于2，再等等
                return;
            }
            // 等时间超过两秒了，磁盘可能很忙，我们就不等后台fsync结束了
            server.aof_delayed_fsync++;
            serverLog(LL_NOTICE,"Asynchronous AOF fsync is taking too long (disk is busy?). Writing the AOF buffer without waiting for fsync to complete, this may slow down Redis.");
        }
    }
    /* We want to perform a single write. This should be guaranteed atomic
     * at least if the filesystem we are writing is a real physical one.
     * While this will save us against the server being killed I don't think
     * there is much to do about the whole server stopping for power problems
     * or alike */
    // 测试用的 根据注释 大概就是我们在写之前休眠一下，让操作系统有时间将前面写的数据刷到磁盘上，这样即便系统异常（断电），可尽可能多的保证数据落盘，而我们等着系统刷aof_buf数据不就全丢了？如果这时候系统异常，aof_buf即便调用了写也没时间落盘。
    if (server.aof_flush_sleep && sdslen(server.aof_buf)) {
        usleep(server.aof_flush_sleep);
    }

    // 调用write系统调用
    latencyStartMonitor(latency);
    nwritten = aofWrite(server.aof_fd,server.aof_buf,sdslen(server.aof_buf));
    latencyEndMonitor(latency);
    // 收集监控数据
    if (sync_in_progress) {
        latencyAddSampleIfNeeded("aof-write-pending-fsync",latency);
    } else if (hasActiveChildProcess()) {
        latencyAddSampleIfNeeded("aof-write-active-child",latency);
    } else {
        latencyAddSampleIfNeeded("aof-write-alone",latency);
    }
    latencyAddSampleIfNeeded("aof-write",latency);

    // 写入了，延迟归0
    server.aof_flush_postponed_start = 0;

    // write遇到error了，没写完
    if (nwritten != (ssize_t)sdslen(server.aof_buf)) {
        static time_t last_write_error_log = 0;
        int can_log = 0;

        // 限制日志频率
        if ((server.unixtime - last_write_error_log) > AOF_WRITE_LOG_ERROR_RATE) {
            can_log = 1;
            last_write_error_log = server.unixtime;
        }

        if (nwritten == -1) {
            if (can_log) {
                serverLog(LL_WARNING,"Error writing to the AOF file: %s",
                    strerror(errno));
                server.aof_last_write_errno = errno;
            }
        } else {
            if (can_log) {
                serverLog(LL_WARNING,"Short write while writing to "
                                       "the AOF file: (nwritten=%lld, "
                                       "expected=%lld)",
                                       (long long)nwritten,
                                       (long long)sdslen(server.aof_buf));
            }
            // 写入部分成功，我们回退写入部分，就当没写入
            if (ftruncate(server.aof_fd, server.aof_current_size) == -1) {
                if (can_log) {
                    serverLog(LL_WARNING, "Could not remove short write "
                             "from the append-only file.  Redis may refuse "
                             "to load the AOF the next time it starts.  "
                             "ftruncate: %s", strerror(errno));
                }
            } else {
                nwritten = -1;
            }
            server.aof_last_write_errno = ENOSPC;
        }

        /* Handle the AOF write error. */
        if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
            // 一直在fsync，所有写和读都会及时被fsync，这时候写失败了，可能已经被刷到磁盘上了，没办法回滚了
            serverLog(LL_WARNING,"Can't recover from AOF write error when the AOF fsync policy is 'always'. Exiting...");
            exit(1);
        } else {
            // write error
            server.aof_last_write_status = C_ERR;

            // ftruncate没回退成功，那就接着写入成功的部分数据
            if (nwritten > 0) {
                server.aof_current_size += nwritten;
                sdsrange(server.aof_buf,nwritten,-1);
            }
            return; /* We'll try again on the next call... */
        }
    } else {
        // 写成功了，状态改为ok，因为前面一次写可能失败改成了err，这次写成功相当于解决了err
        if (server.aof_last_write_status == C_ERR) {
            serverLog(LL_WARNING,
                "AOF write error looks solved, Redis can write again.");
            server.aof_last_write_status = C_OK;
        }
    }
    server.aof_current_size += nwritten;

    // 如果aofbuf小于4k，就重用。过于大的话，下次可能只需要很小的aof，我们释放掉节省内存
    if ((sdslen(server.aof_buf)+sdsavail(server.aof_buf)) < 4000) {
        sdsclear(server.aof_buf);
    } else {
        sdsfree(server.aof_buf);
        server.aof_buf = sdsempty();
    }

try_fsync:
    // 如果子进程在做磁盘io，fsync可能很慢，就先不做fsync了
    if (server.aof_no_fsync_on_rewrite && hasActiveChildProcess())
        return;

    if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
        latencyStartMonitor(latency);
        // 如果是always，就要write结束后及时fsync
        if (redis_fsync(server.aof_fd) == -1) {
            serverLog(LL_WARNING,"Can't persist AOF for fsync error when the "
              "AOF fsync policy is 'always': %s. Exiting...", strerror(errno));
            exit(1);
        }
        latencyEndMonitor(latency);
        latencyAddSampleIfNeeded("aof-fsync-always",latency);
        // 更新fsync数据
        server.aof_fsync_offset = server.aof_current_size;
        server.aof_last_fsync = server.unixtime;
    } else if ((server.aof_fsync == AOF_FSYNC_EVERYSEC &&
                server.unixtime > server.aof_last_fsync)) {
        // 没有后台线程fsync，搞一个
        if (!sync_in_progress) {
            aof_background_fsync(server.aof_fd);
            server.aof_fsync_offset = server.aof_current_size;
        }
        server.aof_last_fsync = server.unixtime;
    }
}
```
可以看到redis为了保证持久化数据在系统故障的时候尽量少丢数据，将持久化分为两步，write和fsync。write函数将数据写入系统缓冲区，fsync将文件的系统缓冲区的数据刷到磁盘上。根据fsync策略不同，对待error和fsync时机也不同，AOF_FSYNC_EVERYSEC配置兼顾性能和持久化，即便是系统故障，也只会丢掉一两秒的数据，而且redis团队表示此配置性能也很高。  
刷数据是在beforeSleep，非强制表示如果是每秒刷配置，则会推迟刷，之后会在系统时间事件serverCron中再次刷，保证数据一致性。
### aof rewrite
aof不停的将命令数据追加到aof文件，因为多了命令和协议数据，aof比仅存数据的rdb文件大了非常多，这样不停地追加aof文件，将会很快膨胀到非常大，有些数据set进去后又del了，但是还是存在aof文件中，这些其实都是无效数据了，redis会在合适时机使用子进程进行rewrite，以减少aof文件大小
```
int rewriteAppendOnlyFileBackground(void) {
    pid_t childpid;

    if (hasActiveChildProcess()) return C_ERR;
    // 初始化管道
    if (aofCreatePipes() != C_OK) return C_ERR;
    // 调用fork
    if ((childpid = redisFork(CHILD_TYPE_AOF)) == 0) {
        char tmpfile[256];
        /* Child */
        redisSetProcTitle("redis-aof-rewrite");
        redisSetCpuAffinity(server.aof_rewrite_cpulist);
        snprintf(tmpfile,256,"temp-rewriteaof-bg-%d.aof", (int) getpid());
        // 重写
        if (rewriteAppendOnlyFile(tmpfile) == C_OK) {
            sendChildCowInfo(CHILD_INFO_TYPE_AOF_COW_SIZE, "AOF rewrite");
            exitFromChild(0);
        } else {
            exitFromChild(1);
        }
    } else {
        /* Parent */
        if (childpid == -1) {
            serverLog(LL_WARNING,
                "Can't rewrite append only file in background: fork: %s",
                strerror(errno));
            aofClosePipes();
            return C_ERR;
        }
        serverLog(LL_NOTICE,
            "Background append only file rewriting started by pid %ld",(long) childpid);
        server.aof_rewrite_scheduled = 0;
        server.aof_rewrite_time_start = time(NULL);

        // 要求强刷一下select命令，可以更安全的合并
        server.aof_selected_db = -1;
        replicationScriptCacheFlush();
        return C_OK;
    }
    return C_OK; /* unreached */
}

int rewriteAppendOnlyFile(char *filename) {
    rio aof;
    FILE *fp = NULL;
    char tmpfile[256];
    char byte;

    // 这使用的名字不是父进程传进来的
    snprintf(tmpfile,256,"temp-rewriteaof-%d.aof", (int) getpid());
    fp = fopen(tmpfile,"w");
    if (!fp) {
        serverLog(LL_WARNING, "Opening the temp file for AOF rewrite in rewriteAppendOnlyFile(): %s", strerror(errno));
        return C_ERR;
    }

    server.aof_child_diff = sdsempty();
    rioInitWithFile(&aof,fp);
    // 默认写32M 自动刷一次
    if (server.aof_rewrite_incremental_fsync)
        rioSetAutoSync(&aof,REDIS_AUTOSYNC_BYTES);

    startSaving(RDBFLAGS_AOF_PREAMBLE);

    if (server.aof_use_rdb_preamble) {
        // 使用rdb方式写aof
        int error;
        if (rdbSaveRio(&aof,&error,RDBFLAGS_AOF_PREAMBLE,NULL) == C_ERR) {
            errno = error;
            goto werr;
        }
    } else {
        // 重写aof 重写过程为读取所有db数据，根据数据类型不同组成不同命令写入aof文件
        // 这样只有当前有的数据，如果删除命令较多，会大大减少文件大小
        if (rewriteAppendOnlyFileRio(&aof) == C_ERR) goto werr;
    }

    // 写完了 调用fflush将数据写入内核 会使最后一次fsync更快
    if (fflush(fp) == EOF) goto werr;
    if (fsync(fileno(fp)) == -1) goto werr;

    // 尽量从父进程读数据出来
    int nodata = 0;
    mstime_t start = mstime();
    while(mstime()-start < 1000 && nodata < 20) {
        if (aeWait(server.aof_pipe_read_data_from_parent, AE_READABLE, 1) <= 0)
        {
            nodata++;
            continue;
        }
        nodata = 0;
        // 读数据到child_diff
        aofReadDiffFromParent();
    }

    // 写入 父进程ack  父进程收到子进程的ack 就不会再发新的diff数据了
    if (write(server.aof_pipe_write_ack_to_parent,"!",1) != 1) goto werr;
    if (anetNonBlock(NULL,server.aof_pipe_read_ack_from_parent) != ANET_OK)
        goto werr;
    // 读取 父进程ack 超时5s
    if (syncRead(server.aof_pipe_read_ack_from_parent,&byte,1,5000) != 1 ||
        byte != '!') goto werr;
    serverLog(LL_NOTICE,"Parent agreed to stop sending diffs. Finalizing AOF...");

    // 最后再读一下新数据
    aofReadDiffFromParent();

    /* Write the received diff to the file. */
    serverLog(LL_NOTICE,
        "Concatenating %.2f MB of AOF diff received from parent.",
        (double) sdslen(server.aof_child_diff) / (1024*1024));

    // 把收到的diff数据重写
    size_t bytes_to_write = sdslen(server.aof_child_diff);
    const char *buf = server.aof_child_diff;
    long long cow_updated_time = mstime();
    long long key_count = dbTotalServerKeyCount();
    while (bytes_to_write) {
        // 一次写8M数据
        size_t chunk_size = bytes_to_write < (8<<20) ? bytes_to_write : (8<<20);

        if (rioWrite(&aof,buf,chunk_size) == 0)
            goto werr;

        bytes_to_write -= chunk_size;
        buf += chunk_size;

        // 父进程在子进程生命周期中传过来的数据一定是copy的，这个copy操作非常费时费力。
        // 子进程每次写8M，统计时间和cow的内存数发给父进程。不过从代码来看父进程完全不care
        long long now = mstime();
        if (now - cow_updated_time >= 1000) {
            sendChildInfo(CHILD_INFO_TYPE_CURRENT_INFO, key_count, "AOF rewrite");
            cow_updated_time = now;
        }
    }

    // 所有数据都写到磁盘
    if (fflush(fp)) goto werr;
    if (fsync(fileno(fp))) goto werr;
    if (fclose(fp)) { fp = NULL; goto werr; }
    fp = NULL;

    // 改名成父进程传进来的 只有在成功重写之后 才会生成父进程想要的文件，
    // 否则直接走err，不会生成父进程想要的文件
    if (rename(tmpfile,filename) == -1) {
        serverLog(LL_WARNING,"Error moving temp append only file on the final destination: %s", strerror(errno));
        unlink(tmpfile);
        stopSaving(0);
        return C_ERR;
    }
    serverLog(LL_NOTICE,"SYNC append only file rewrite performed");
    stopSaving(1);
    return C_OK;

werr:
    serverLog(LL_WARNING,"Write error writing append only file on disk: %s", strerror(errno));
    if (fp) fclose(fp);
    unlink(tmpfile);
    stopSaving(0);
    return C_ERR;
}
```
以上为重写aof文件的流程。  
最后还有子进程结束后父进程监听到后进行的收尾工作，比如在子进程进行重写的时候，父进程先将diff数据放入rewritebuf，然后才写入pipe，子进程结束后，rewritebuf可能又有数据了，写入新文件，然后将新文件改成配置文件中的aof文件的名字，以接收下面的新命令追加。这些就不详细解释了，都很好理解。
## 总结
aof是redis的持久化的方式之一，它是通过将用户命令追加到文件，还原通过读取aof文件，按顺序执行其中的每一条命令来进行还原。重写过程中的各种边界条件都要考虑到，任何数据都不能丢，要重新写入新的aof文件中，并且对于系统调用，redis想办法加快或使用子线程来做。
