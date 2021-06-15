# 主从
主从最重要的就是数据一致性，redis数据同步主要分为两种，全量和增量。  
redis通过replid + master_repl_offset 来确定数据偏移，当这slave中的这两个数据与master相同，代表主从数据一致。replid一串40字节的随机字符串，在服务启动的时候进行生成，在主从状态改变的时候会生成新的replid，类似于raft协议中任期的那种感觉。如果slave和master的replid不同，则需要进行全量
## 全量
首先从服务器链接到master，肯定要全量。
```
void syncCommand(client *c) {
    .....

    // 尝试增量同步
    if (!strcasecmp(c->argv[0]->ptr,"psync")) {
        if (masterTryPartialResynchronization(c) == C_OK) {
            server.stat_sync_partial_ok++;
            return; /* No full resync needed, return. */
        } else {
            char *master_replid = c->argv[1]->ptr;
            if (master_replid[0] != '?') server.stat_sync_partial_err++;
        }
    } else {
        c->flags |= CLIENT_PRE_PSYNC;
    }

    /* Full resynchronization. */
    server.stat_sync_full++;

    c->replstate = SLAVE_STATE_WAIT_BGSAVE_START;
    if (server.repl_disable_tcp_nodelay)
        connDisableTcpNoDelay(c->conn); /* Non critical if it fails. */
    c->repldbfd = -1;
    c->flags |= CLIENT_SLAVE;
    // 加入到从列表
    listAddNodeTail(server.slaves,c);

    if (listLength(server.slaves) == 1 && server.repl_backlog == NULL) {
        // 生成新的replid，创建backlog
        changeReplicationId();
        clearReplicationId2();
        createReplicationBacklog();
        serverLog(LL_NOTICE,"Replication backlog created, my new "
                            "replication IDs are '%s' and '%s'",
                            server.replid, server.replid2);
    }

    // 第一种情况：这个slave来同步的时候，后台已经在保存rdb文件了，目标disk
    if (server.child_type == CHILD_TYPE_RDB &&
        server.rdb_child_type == RDB_CHILD_TYPE_DISK)
    {
        // 这里找一个slave跟新来的slave对比
        client *slave;
        listNode *ln;
        listIter li;

        listRewind(server.slaves,&li);
        while((ln = listNext(&li))) {
            slave = ln->value;
            if (slave->replstate == SLAVE_STATE_WAIT_BGSAVE_END &&
                (!(slave->flags & CLIENT_REPL_RDBONLY) ||
                 (c->flags & CLIENT_REPL_RDBONLY)))
                break;
        }
        // 如果当前bgsave的cap与新来的slave要支持的capa一样，只需要把slave的输出buffer拷贝给新来的slave即可
        // 这个输出buffer就是下面所述的全量的增量
        if (ln && ((c->slave_capa & slave->slave_capa) == slave->slave_capa)) {
            if (!(c->flags & CLIENT_REPL_RDBONLY)) copyClientOutputBuffer(c,slave);
            replicationSetupSlaveForFullResync(c,slave->psync_initial_offset);
            serverLog(LL_NOTICE,"Waiting for end of BGSAVE for SYNC");
        } else {
            // 否则等下次同步
            serverLog(LL_NOTICE,"Can't attach the replica to the current BGSAVE. Waiting for next BGSAVE for SYNC");
        }

    // 第二种情况：后台子进程在保存rdb文件，目标是socket
    } else if (server.child_type == CHILD_TYPE_RDB &&
               server.rdb_child_type == RDB_CHILD_TYPE_SOCKET)
    {
        // socket是现读现写，跟不上这次了，等下次吧
        serverLog(LL_NOTICE,"Current BGSAVE has socket target. Waiting for next BGSAVE for SYNC");

    // 第三种情况
    } else {
        if (server.repl_diskless_sync && (c->slave_capa & SLAVE_CAPA_EOF) &&
            server.repl_diskless_sync_delay)
        {
            // 不用磁盘的同步，等多点slave到再一起
            serverLog(LL_NOTICE,"Delay next BGSAVE for diskless SYNC");
        } else {
            // 没有子进程保存rdb，保存一个rdb，此时是磁盘capa
            if (!hasActiveChildProcess()) {
                startBgsaveForReplication(c->slave_capa);
            } else {
                serverLog(LL_NOTICE,
                    "No BGSAVE in progress, but another BG operation is active. "
                    "BGSAVE for replication delayed");
            }
        }
    }
    return;
}

void updateSlavesWaitingBgsave(int bgsaveerr, int type) {
    listNode *ln;
    listIter li;

    listRewind(server.slaves,&li);
    while((ln = listNext(&li))) {
        client *slave = ln->value;

        if (slave->replstate == SLAVE_STATE_WAIT_BGSAVE_END) {
            struct redis_stat buf;

            if (bgsaveerr != C_OK) {
                freeClient(slave);
                serverLog(LL_WARNING,"SYNC failed. BGSAVE child returned an error");
                continue;
            }
            if (type == RDB_CHILD_TYPE_SOCKET) {
                serverLog(LL_NOTICE,
                    "Streamed RDB transfer with replica %s succeeded (socket). Waiting for REPLCONF ACK from slave to enable streaming",
                        replicationGetSlaveName(slave));
            } else {
                if ((slave->repldbfd = open(server.rdb_filename,O_RDONLY)) == -1 ||
                    redis_fstat(slave->repldbfd,&buf) == -1) {
                    freeClient(slave);
                    serverLog(LL_WARNING,"SYNC failed. Can't open/stat DB after BGSAVE: %s", strerror(errno));
                    continue;
                }
                slave->repldboff = 0;
                slave->repldbsize = buf.st_size;
                slave->replstate = SLAVE_STATE_SEND_BULK;
                slave->replpreamble = sdscatprintf(sdsempty(),"$%lld\r\n",
                    (unsigned long long) slave->repldbsize);
                connSetWriteHandler(slave->conn,NULL);
                // 发送rdb文件到slave
                if (connSetWriteHandler(slave->conn,sendBulkToSlave) == C_ERR) {
                    freeClient(slave);
                    continue;
                }
            }
        }
    }
}
```
redis提供两种全量方式，一种是生成rdb文件到磁盘（capa_none），一种是不使用磁盘（capa_eof），直接使用socket。进行全量前会现将master当前的replid+offset发送给slave。
### 磁盘
磁盘全量是通过生成rdb文件，然后将rdb文件发送给从服务器。调用`startBgsaveForReplication` 生成rdb文件，子进程执行完毕，调用 `updateSlavesWaitingBgsave` 将rdb文件发送给从服务器。发送结束后，slave的状态就被改为online。
### socket
socket与磁盘一样，都是通过`startBgsaveForReplication` 来进行同步（但是是在servercron中，为了尽可能等待多的需要全量的slave，否则每来一个slave，就全量一次太浪费），子进程与父进程通过管道通信，将rdb数据不写入文件，而是写入管道，父进程将读管道注册到事件循环中，在每次有数据读的时候调用`rdbPipeReadHandler`函数来进行处理，从管道读取数据，遍历slave的连接，write数据即可。socket的rdb结束会有一个特殊的EOF标志  
### 全量期间的增量
在slave接收master的全量同步的时候，此时如果有用户命令处理，master直接将数据写入slave类型的client的buffer中，等待全量完成（全量完成slave会发送ack），slave的状态为online并且repl_put_online_on_ack为0，再有数据要写才会加入clients_pending_write,在beforeSleep中处理client中的buffer数据，发送给slave。  
***全量分为两种，磁盘和非磁盘，但是其实都是rdb类型，只不过socket传rdb不知道预计大小，磁盘文件的话可以获取大小，因此socket全量会有一个eof，不同类型的同步能力的slave连接到master，master得两次bgsave。如果优化成一次bgsave呢？其实bgsave速度非常快，是对db内存数据的一个序列化，写入pipe还是写入磁盘都一样，都需要io，如果合并成一次，io数量大大增加，用户有意配置只使用磁盘的话，速度反而会变慢，得不偿失***
## 增量
增量同步的基础是slave和master全量过，中间slave掉线重连，slave发送自己的replid+offset给master，master判断全量还是增量。增量的基础是通过backlog，上面我们可以看到backlog的创建，当有slave连上来，就创建backlog，将后续的命令通过传播（就是写aof那个传播）存入backlog。backlog是一个预分配的环形队列，写满了从头开始写，起点`repl_backlog_off`是创建backlog的时候server的offset（replid+offset中的offset），长度`repl_backlog_histlen`是backlog中的数据。写入点`repl_backlog_idx`是下一个要写入的位置。总大小`repl_backlog_size`是预分配的内存大小。
```
int masterTryPartialResynchronization(client *c) {
    long long psync_offset, psync_len;
    char *master_replid = c->argv[1]->ptr;
    char buf[128];
    int buflen;

    // slave传的数据读取
    if (getLongLongFromObjectOrReply(c,c->argv[2],&psync_offset,NULL) !=
       C_OK) goto need_full_resync;

    // 对比replid和offset，如果不同就全量
    if (strcasecmp(master_replid, server.replid) &&
        (strcasecmp(master_replid, server.replid2) ||
         psync_offset > server.second_replid_offset))
    {
        ......
        goto need_full_resync;
    }

    // 没有backlog，请求的offset没有落在backlog里面
    if (!server.repl_backlog ||
        psync_offset < server.repl_backlog_off ||
        psync_offset > (server.repl_backlog_off + server.repl_backlog_histlen))
    {
        ......
        goto need_full_resync;
    }

    // 增量同步，全量发送的是fullsync  增量发送continue
    c->flags |= CLIENT_SLAVE;
    c->replstate = SLAVE_STATE_ONLINE;
    c->repl_ack_time = server.unixtime;
    c->repl_put_online_on_ack = 0;
    listAddNodeTail(server.slaves,c);

    if (c->slave_capa & SLAVE_CAPA_PSYNC2) {
        buflen = snprintf(buf,sizeof(buf),"+CONTINUE %s\r\n", server.replid);
    } else {
        buflen = snprintf(buf,sizeof(buf),"+CONTINUE\r\n");
    }
    // 写continue
    if (connWrite(c->conn,buf,buflen) != buflen) {
        freeClientAsync(c);
        return C_OK;
    }
    // 将backlog数据给新来的slave
    psync_len = addReplyReplicationBacklog(c,psync_offset);
    refreshGoodSlavesCount();
    moduleFireServerEvent(REDISMODULE_EVENT_REPLICA_CHANGE,
                          REDISMODULE_SUBEVENT_REPLICA_CHANGE_ONLINE,
                          NULL);

    return C_OK; /* The caller can return, no full resync needed. */

need_full_resync:
    return C_ERR;
}
```
增量非常简单，主要就是对backlog这个环形队列的维护。
## 总结
同步过程其实非常复杂，因为涉及到非常多的状态数据，这里并没有列出状态，也仅仅列出核心代码，其实整个过程涉及的变量非常多，当然，我们知道主要思想其实也够了。在这些代码中其实也涉及了哨兵模式和集群模式，后面一一分析  
***主从是可以对从再次进行主从的，也就是说是树状结构，这样可以减轻master的工作量，但是原理都是相同的，第一级的slave会将master的数据转发给自己的slave。同步过程也大同小异***
