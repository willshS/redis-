# 多线程
最新的redis已经支持多线程了，在[bio](./bio.md)中已经有后台线程的介绍，主要做close,fsync和free三种操作，可以看到基本都是耗费时间的阻塞调用。那么read和write函数同样消耗时间，最新的redis就已经支持这两个多线程调用了。  
## 定义与初始化
```
pthread_t io_threads[IO_THREADS_MAX_NUM];  // 所有的thread
pthread_mutex_t io_threads_mutex[IO_THREADS_MAX_NUM]; // thread对应的锁
redisAtomic unsigned long io_threads_pending[IO_THREADS_MAX_NUM]; // 线程任务数
int io_threads_op; // 线程要做的操作
// 线程要处理的任务，list中是client对象
list *io_threads_list[IO_THREADS_MAX_NUM];


void initThreadedIO(void) {
   server.io_threads_active = 0; /* We start with threads not active. */

   // 用户配置的线程数量，1就是只用main thread（不包括bio，bio的线程一定会工作）
   if (server.io_threads_num == 1) return;
   // 太多也不行，最大是128
   if (server.io_threads_num > IO_THREADS_MAX_NUM) {
       serverLog(LL_WARNING,"Fatal: too many I/O threads configured. "
                            "The maximum number is %d.", IO_THREADS_MAX_NUM);
       exit(1);
   }

   /* Spawn and initialize the I/O threads. */
   for (int i = 0; i < server.io_threads_num; i++) {
       /* Things we do for all the threads including the main thread. */
       io_threads_list[i] = listCreate();
       if (i == 0) continue; /* Thread 0 is the main thread. */

       /* Things we do only for the additional threads. */
       pthread_t tid;
       pthread_mutex_init(&io_threads_mutex[i],NULL);
       setIOPendingCount(i, 0);
       // 这个锁就是为了让main thread可以阻塞子线程的 这里给锁住
       pthread_mutex_lock(&io_threads_mutex[i]); /* Thread will be stopped. */
       if (pthread_create(&tid,NULL,IOThreadMain,(void*)(long)i) != 0) {
           serverLog(LL_WARNING,"Fatal: Can't initialize IO thread.");
           exit(1);
       }
       io_threads[i] = tid;
   }
}
```

## 线程主函数
```
void *IOThreadMain(void *myid) {
    long id = (unsigned long)myid;
    char thdname[16];
    // 与bio一样 设置亲和等
    snprintf(thdname, sizeof(thdname), "io_thd_%ld", id);
    redis_set_thread_title(thdname);
    redisSetCpuAffinity(server.server_cpulist);
    makeThreadKillable();

    while(1) {
        // 等任务
        for (int j = 0; j < 1000000; j++) {
            if (getIOPendingCount(id) != 0) break;
        }

        // 没任务的时候 这里就提供一个机会让主线程能暂停此线程
        if (getIOPendingCount(id) == 0) {
            pthread_mutex_lock(&io_threads_mutex[id]);
            pthread_mutex_unlock(&io_threads_mutex[id]);
            continue;
        }

        serverAssert(getIOPendingCount(id) != 0);

        // 处理工作
        listIter li;
        listNode *ln;
        listRewind(io_threads_list[id],&li);
        while((ln = listNext(&li))) {
            client *c = listNodeValue(ln);
            if (io_threads_op == IO_THREADS_OP_WRITE) {
                writeToClient(c,0);
            } else if (io_threads_op == IO_THREADS_OP_READ) {
                readQueryFromClient(c->conn);
            } else {
                serverPanic("io_threads_op value is unknown");
            }
        }
        listEmpty(io_threads_list[id]);
        // 工作处理完之后，将任务数改为0
        setIOPendingCount(id, 0);
    }
}
```
从上面两个函数可以看到，redis初始化的时候会根据配置初始化相应的线程，并将它们阻塞，这时候`server.io_threads_active == 0`表示线程都在睡觉，合适的时机再把它们叫醒。可以从线程主函数看到，线程的操作就两种，写和读，每个线程要处理一个list的client，互不干扰。前面介绍过，读的入口在`readQueryFromClient`，写的操作在`beforeSleep`，可以在其中看到如何使用这些线程来进行读写。  
## read io
首先是有数据来了，我们要收集需要read的客户端都有哪些。
```
// readQueryFromClient函数最一开始就调用下面的函数
int postponeClientRead(client *c) {
    // 线程是活动的，允许使用线程读（配置），没有在block的时候处理事件，客户端不是主或从或已经在线程里面了。
    if (server.io_threads_active &&
        server.io_threads_do_reads &&
        !ProcessingEventsWhileBlocked &&
        !(c->flags & (CLIENT_MASTER|CLIENT_SLAVE|CLIENT_PENDING_READ)))
    {
        c->flags |= CLIENT_PENDING_READ;
        listAddNodeHead(server.clients_pending_read,c);
        return 1;
    } else {
        return 0;
    }
}
```
*ProcessingEventsWhileBlocked 这个暂时可以不管，其主要是为了在block的时候也可以处理事件，目前用于加载aof和rdb，执行script超时三个地方*  
正常情况下我们有read事件就直接调用read读数据然后处理数据，然后将处理结果放入回复列表中，都是一个线程在做，现在我们直接将需要调用read的放入一个链表。然后在beforeSleep的时候进行处理。
```
int handleClientsWithPendingReadsUsingThreads(void) {
    if (!server.io_threads_active || !server.io_threads_do_reads) return 0;
    int processed = listLength(server.clients_pending_read);
    if (processed == 0) return 0;

    // 把需要读的client给子线程们分一分
    listIter li;
    listNode *ln;
    listRewind(server.clients_pending_read,&li);
    int item_id = 0;
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id],c);
        item_id++;
    }

    // 注意 放到队列之后，再修改任务数量
    io_threads_op = IO_THREADS_OP_READ;
    for (int j = 1; j < server.io_threads_num; j++) {
        int count = listLength(io_threads_list[j]);
        setIOPendingCount(j, count);
    }

    // main线程也不能看着，得干自己该干的活
    listRewind(io_threads_list[0],&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        readQueryFromClient(c->conn);
    }
    listEmpty(io_threads_list[0]);

    // 等子线程把任务全部做完
    while(1) {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += getIOPendingCount(j);
        if (pending == 0) break;
    }

    // 子线程把数据都处理完了，全都生成了命令，但是命令没处理（命令都是主线程处理，因为涉及到了db数据结构，redis的数据结构不支持多线程，所以redis多线程仅仅是io）
    while(listLength(server.clients_pending_read)) {
        ln = listFirst(server.clients_pending_read);
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_READ;
        listDelNode(server.clients_pending_read,ln);
        // 处理命令
        if (processPendingCommandsAndResetClient(c) == C_ERR) {
            continue;
        }
        // 新来的数据也顺便处理一下
        processInputBuffer(c);

        // 子线程是没办法reply的，这里我们把都给reply了
        if (!(c->flags & CLIENT_PENDING_WRITE) && clientHasPendingReplies(c))
            clientInstallWriteHandler(c);
    }
    server.stat_io_reads_processed += processed;

    return processed;
}
```
上面就是redis多线程处理read io。  
1. 如果多线程可以用，将read io分配给多线程
2. 多线程处理事情，读取数据处理数据（不执行命令）
3. 主线程等待多线程做完事情，统一回复。此时多线程闲下来了  

因此从第一步开始天然就不需要加锁，只需要任务队列填满之后再去原子写任务数量就可以了。每次多线程开始执行任务，主线程都会等待所有子线程执行完，再往下进行，下次再次分配任务的时候子线程一定是没有在执行任务的。主线程与子线程通信就是通过一个原子变量就可以了，非常高效！
## write io
```
int stopThreadedIOIfNeeded(void) {
    int pending = listLength(server.clients_pending_write);

    if (server.io_threads_num == 1) return 1;
    // 客户端太少了，用不上多线程
    if (pending < (server.io_threads_num*2)) {
        if (server.io_threads_active) stopThreadedIO();
        return 1;
    } else {
        return 0;
    }
}

int handleClientsWithPendingWritesUsingThreads(void) {
    int processed = listLength(server.clients_pending_write);
    if (processed == 0) return 0;
    // 用不到多线程
    if (server.io_threads_num == 1 || stopThreadedIOIfNeeded()) {
        return handleClientsWithPendingWrites();
    }

    // 打开多线程
    if (!server.io_threads_active) startThreadedIO();

    // 分配任务
    listIter li;
    listNode *ln;
    listRewind(server.clients_pending_write,&li);
    int item_id = 0;
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_WRITE;
        // 客户端要关了，不写了
        if (c->flags & CLIENT_CLOSE_ASAP) {
            listDelNode(server.clients_pending_write, ln);
            continue;
        }

        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id],c);
        item_id++;
    }
    // 子线程开始
    io_threads_op = IO_THREADS_OP_WRITE;
    for (int j = 1; j < server.io_threads_num; j++) {
        int count = listLength(io_threads_list[j]);
        setIOPendingCount(j, count);
    }

    // 老规矩 main thread不要闲着
    listRewind(io_threads_list[0],&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        writeToClient(c,0);
    }
    listEmpty(io_threads_list[0]);

    // 等活完成
    while(1) {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += getIOPendingCount(j);
        if (pending == 0) break;
    }

    // 有些write失败了，注册事件等下一次
    listRewind(server.clients_pending_write,&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        if (clientHasPendingReplies(c) &&
                connSetWriteHandler(c->conn, sendReplyToClient) == AE_ERR)
        {
            freeClientAsync(c);
        }
    }
    listEmpty(server.clients_pending_write);
    server.stat_io_writes_processed += processed;
    return processed;
}
```
跟读其实一模一样，唯一不同的就是一个控制子线程是否运行的开关`startThreadedIO`和`stopThreadedIO`。根据需要写的客户端数量来控制多线程是否用。因此多线程是否活动其实依赖于需要写的客户端数量，从理论上来讲读客户端数量跟写客户端数量基本相同，因为redis协议基本都有回复的，我认为可能是存在消息队列广播，主从复制等功能，所以写的io数量可能是读的io数量的几倍（一个从就要广播一次客户端的命令）。因此针对最大的瓶颈来开启关闭多线程
## 总结
redis多线程仅仅是用于io，io前分配任务，io后主线程要等待多线程全部都完成任务。  
redis的数据结构全都不支持多线程，因此多线程仅仅读数据写数据分析数据，不进行任何处理命令的操作，也不需要它们处理命令，因为redis所有数据都在内存中，速度非常快，一个main速度就够了，真正瓶颈在于io，多线程搞io就可以了。
