# 模型
## db
redisdb是redis存储数据的模型。redis默认有16个db，可以进行配置。
```
typedef struct redisDb {
    dict *dict;                 /* k-v */
    dict *expires;              /* k-when */
    dict *blocking_keys;        /* k-client list*/
    dict *ready_keys;           /* k-null*/
    dict *watched_keys;         /* k-client list */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
    unsigned long expires_cursor; /* Cursor of the active expire cycle. */
    list *defrag_later;         /* redis6.2.3 好像未启用 */
} redisDb;
```
`dict` 主键哈希表，key-value都存储在这里。  
`expires` 有过期设置的key value是过期时间   
`blocking_keys` 阻塞键 value是等待这个key的client链表  
`ready_keys` 阻塞就绪键 value是NULL  
`watched_keys` 监听key value是监听这个key的client链表  
`id` id  

### 初始化
```
server.db = zmalloc(sizeof(redisDb)*server.dbnum);
for (j = 0; j < server.dbnum; j++) {
        server.db[j].dict = dictCreate(&dbDictType,NULL);
        server.db[j].expires = dictCreate(&dbExpiresDictType,NULL);
        server.db[j].expires_cursor = 0;
        server.db[j].blocking_keys = dictCreate(&keylistDictType,NULL);
        server.db[j].ready_keys = dictCreate(&objectKeyPointerValueDictType,NULL);
        server.db[j].watched_keys = dictCreate(&keylistDictType,NULL);
        server.db[j].id = j;
        server.db[j].avg_ttl = 0;
        server.db[j].defrag_later = listCreate();
        listSetFreeMethod(server.db[j].defrag_later,(void (*)(void*))sdsfree);
    }
```
### db
#### dict && expires
dict是我们所有key存储的空间，redis是kv库，那么命令的处理回调函数中肯定要对key进行查询，通过调用`lookupKey`来进行查询的，找到更新对应的lfu，lru。这里就使用了db.dict。而db就是server.db中的一个（根据客户端select的结果，默认为0号db）。  
```
// set key value 命令set 来分析db是如何运作的
void setGenericCommand(client *c, int flags, robj *key, robj *val, robj *expire, int unit, robj *ok_reply, robj *abort_reply) {
    long long milliseconds = 0, when = 0;

    // 如果命令中有过期时间，处理过期时间，存入when
    if (expire) {
        ...... // 代码太多 省略了
    }

    // 处理异常情况，nx xx getset
    ......

    genericSetKey(c,c->db,key, val,flags & OBJ_KEEPTTL,1);
    server.dirty++;
    notifyKeyspaceEvent(NOTIFY_STRING,"set",key,c->db->id);
    if (expire) {
        // 设置过期，即放入db->expire
        setExpire(c,c->db,key,when);
        notifyKeyspaceEvent(NOTIFY_GENERIC,"expire",key,c->db->id);
        robj *exp = (flags & OBJ_PXAT) || (flags & OBJ_EXAT) ? shared.pxat : shared.px;
        robj *millisecondObj = createStringObjectFromLongLong(milliseconds);
        rewriteClientCommandVector(c,5,shared.set,key,val,exp,millisecondObj);
        decrRefCount(millisecondObj);
    }
    if (!(flags & OBJ_SET_GET)) {
        addReply(c, ok_reply ? ok_reply : shared.ok);
    }

    if ((flags & OBJ_SET_GET) && !expire) {
        int argc = 0;
        int j;
        robj **argv = zmalloc((c->argc-1)*sizeof(robj*));
        for (j=0; j < c->argc; j++) {
            char *a = c->argv[j]->ptr;
            if (j >= 3 &&
                (a[0] == 'g' || a[0] == 'G') &&
                (a[1] == 'e' || a[1] == 'E') &&
                (a[2] == 't' || a[2] == 'T') && a[3] == '\0')
                continue;
            argv[argc++] = c->argv[j];
            incrRefCount(c->argv[j]);
        }
        replaceClientCommandVector(c, argc, argv);
    }
}

void genericSetKey(client *c, redisDb *db, robj *key, robj *val, int keepttl, int signal) {
    if (lookupKeyWrite(db,key) == NULL) {
        dbAdd(db,key,val);
    } else {
        dbOverwrite(db,key,val);
    }
    // 上层函数增加引用计数，dbAdd不管理
    incrRefCount(val);
    // 不过期，从db->expires中删除
    if (!keepttl) removeExpire(db,key);
    // 通知此key变化了
    if (signal) signalModifiedKey(c,db,key);
}

robj *lookupKeyWriteWithFlags(redisDb *db, robj *key, int flags) {
    expireIfNeeded(db,key);
    return lookupKey(db,key,flags);
}

int expireIfNeeded(redisDb *db, robj *key) {
    if (!keyIsExpired(db,key)) return 0;

    // 如果过期了，但是不是master，直接返回过期。不进行删除，等master给消息
    if (server.masterhost != NULL) return 1;

    if (checkClientPauseTimeoutAndReturnIfPaused()) return 1;

    /* Delete the key */
    if (server.lazyfree_lazy_expire) {
        // 异步删除，计算value的大小，如果大于阈值，value在后台bio线程删除。key直接删除
        dbAsyncDelete(db,key);
    } else {
        // 直接释放key-value
        dbSyncDelete(db,key);
    }
    server.stat_expiredkeys++;
    // 通知从删除，并写入aof
    propagateExpire(db,key,server.lazyfree_lazy_expire);
    notifyKeyspaceEvent(NOTIFY_EXPIRED,
        "expired",key,db->id);
    signalModifiedKey(NULL,db,key);
    return 1;
}

void dbAdd(redisDb *db, robj *key, robj *val) {
    sds copy = sdsdup(key->ptr);
    // 终于把数据加到db里面了 val是robj，可以存储任何类型
    int retval = dictAdd(db->dict, copy, val);

    serverAssertWithInfo(NULL,key,retval == DICT_OK);
    // 有client在等待，通知一下
    signalKeyAsReady(db, key, val->type);
    // 集群相关
    if (server.cluster_enabled) slotToKeyAdd(key->ptr);
}
```
redis命令所有的key都会在db->dict中存储，value根据不同命令使用不同的数据结构
#### blocking_keys && ready_keys
阻塞相关命令的哈希表，下面是blpop,brpop
```
/* Blocking RPOP/LPOP */
void blockingPopGenericCommand(client *c, int where) {
    robj *o;
    mstime_t timeout;
    int j;
    // 获取阻塞的超时时间
    if (getTimeoutFromObjectOrReply(c,c->argv[c->argc-1],&timeout,UNIT_SECONDS)
        != C_OK) return;

    // 可以从多个list pop
    for (j = 1; j < c->argc-1; j++) {
        //从dict找数据
        o = lookupKeyWrite(c->db,c->argv[j]);
        if (o != NULL) {
            //value type与命令不匹配
            if (checkType(c,o,OBJ_LIST)) {
                return;
            } else {
                // list不为空，直接pop一个出来送回去
                if (listTypeLength(o) != 0) {
                    robj *value = listTypePop(o,where);
                    serverAssert(value != NULL);
                    //返回两项 先返回2
                    addReplyArrayLen(c,2);
                    // 返回哪个链表pop的
                    addReplyBulk(c,c->argv[j]);
                    // 返回pop出来的数据
                    addReplyBulk(c,value);
                    decrRefCount(value);
                    listElementsRemoved(c,c->argv[j],where,o,1);

                    /* 转换成lpop或rpop */
                    rewriteClientCommandVector(c,2,
                        (where == LIST_HEAD) ? shared.lpop : shared.rpop,
                        c->argv[j]);
                    return;
                }
            }
        }
    }

    // 没找到 不等待 直接返回空
    if (c->flags & CLIENT_DENY_BLOCKING) {
        addReplyNullArray(c);
        return;
    }

    /* If the keys do not exist we must block */
    struct listPos pos = {where};
    blockForKeys(c,BLOCKED_LIST,c->argv + 1,c->argc - 2,timeout,NULL,&pos,NULL);
}

// 这个函数不仅仅是blpop 任何阻塞流都走这里 stream zset等
void blockForKeys(client *c, int btype, robj **keys, int numkeys, mstime_t timeout, robj *target, struct listPos *listpos, streamID *ids) {
    dictEntry *de;
    list *l;
    int j;

    c->bpop.timeout = timeout;
    c->bpop.target = target;

    if (listpos != NULL) c->bpop.listpos = *listpos;

    if (target != NULL) incrRefCount(target);

    for (j = 0; j < numkeys; j++) {
        bkinfo *bki = zmalloc(sizeof(*bki));
        if (btype == BLOCKED_STREAM)
            bki->stream_id = ids[j];

        // 每个client有一个blockingstate 里面是这个用户阻塞相关的信息，如果这个key已经在等了，循环看下一个key。value是阻塞信息，如果是stream阻塞，保存streamid和等待这个key的链表的尾节点
        if (dictAdd(c->bpop.keys,keys[j],bki) != DICT_OK) {
            zfree(bki);
            continue;
        }
        incrRefCount(keys[j]);

        /* blocking_keys出场了，将等待的key放到 db里 */
        de = dictFind(c->db->blocking_keys,keys[j]);
        if (de == NULL) {
            int retval;

            /* 每个等待的key都是一个client链表，因此blocking_key是key-list（client）的哈希表 */
            l = listCreate();
            retval = dictAdd(c->db->blocking_keys,keys[j],l);
            incrRefCount(keys[j]);
            serverAssertWithInfo(c,keys[j],retval == DICT_OK);
        } else {
            l = dictGetVal(de);
        }
        listAddNodeTail(l,c);
        bki->listnode = listLast(l);
    }
    // 把用户加入到server.clients_timeout_table。
    blockClient(c,btype);
}

void blockClient(client *c, int btype) {
    c->flags |= CLIENT_BLOCKED;
    c->btype = btype;
    // block的数量+1
    server.blocked_clients++;
    // 等待对应的type的client+1，比如等待list 则blocked_clients_by_type[BLOCKED_LIST]++
    server.blocked_clients_by_type[btype]++;
    // 加入server的clients_timeout_table。
    addClientToTimeoutTable(c);
    if (btype == BLOCKED_PAUSE) {
        listAddNodeTail(server.paused_clients, c);
        c->paused_list_node = listLast(server.paused_clients);
        /* Mark this client to execute its command */
        c->flags |= CLIENT_PENDING_COMMAND;
    }
}
```
此时我们的客户端就是block状态了，等待数据或者超时。关于超时：超时是redis的时间事件吗？不是，而是每次事件循环时去检查server.clients_timeout_table，超时的返回超时错误。  
数据来了如何通知？看上一节的dbAdd中的signalKeyAsReady。
```
void signalKeyAsReady(redisDb *db, robj *key, int type) {
    readyList *rl;

    // value的type不是那几个有阻塞状态的，直接返回
    int btype = getBlockedTypeByType(type);
    if (btype == BLOCKED_NONE) {
        return;
    }
    // 再次过滤，这个类型是否有client在等。比如现在只有一个BLPOP，那么有人往stream里面添加数据了，不是用户等待的数据，直接返回。
    if (!server.blocked_clients_by_type[btype] &&
        !server.blocked_clients_by_type[BLOCKED_MODULE]) {
        return;
    }

    // 然后再检查这个key有人等没，没人等，返回
    if (dictFind(db->blocking_keys,key) == NULL) return;

    // 这个key已经通知过了 返回
    if (dictFind(db->ready_keys,key) != NULL) return;

    // 这是用户等待的了，加入server的就绪链表
    rl = zmalloc(sizeof(*rl));
    rl->key = key;
    rl->db = db;
    incrRefCount(key);
    listAddNodeTail(server.ready_keys,rl);

    // 加入db的就绪哈希
    incrRefCount(key);
    serverAssert(dictAdd(db->ready_keys,key,NULL) == DICT_OK);
}
```
在`processCommand`函数处理命令中，每个命令执行完自己的回调函数后，都会检查server.ready_keys
```
void handleClientsBlockedOnKeys(void) {
    while(listLength(server.ready_keys) != 0) {
        list *l;

        l = server.ready_keys;
        server.ready_keys = listCreate();

        while(listLength(l) != 0) {
            listNode *ln = listFirst(l);
            readyList *rl = ln->value;

            // 先从read_keys删除，这样signalKeyAsReady()调用安全？
            // 单线程的 好像不应该存在安全不安全 这点不太明白注释意思
            dictDelete(rl->db->ready_keys,rl->key);


            /* Even if we are not inside call(), increment the call depth
             * in order to make sure that keys are expired against a fixed
             * reference time, and not against the wallclock time. This
             * way we can lookup an object multiple times (BLMOVE does
             * that) without the risk of it being freed in the second
             * lookup, invalidating the first one.
             * See https://github.com/redis/redis/pull/6554. */
            server.fixed_time_expire++;
            updateCachedTime(0);

            /* Serve clients blocked on the key. */
            robj *o = lookupKeyWrite(rl->db,rl->key);

            if (o != NULL) {
                if (o->type == OBJ_LIST)
                    serveClientsBlockedOnListKey(o,rl);
                else if (o->type == OBJ_ZSET)
                    serveClientsBlockedOnSortedSetKey(o,rl);
                else if (o->type == OBJ_STREAM)
                    serveClientsBlockedOnStreamKey(o,rl);
                /* We want to serve clients blocked on module keys
                 * regardless of the object type: we don't know what the
                 * module is trying to accomplish right now. */
                serveClientsBlockedOnKeyByModule(rl);
            }
            server.fixed_time_expire--;

            /* Free this item. */
            decrRefCount(rl->key);
            zfree(rl);
            listDelNode(l,ln);
        }
        listRelease(l); /* We have the new list on place at this point. */
    }
}

void serveClientsBlockedOnListKey(robj *o, readyList *rl) {
    // 从blocking_keys取出来block的client链表
    dictEntry *de = dictFind(rl->db->blocking_keys,rl->key);
    if (de) {
        list *clients = dictGetVal(de);
        int numclients = listLength(clients);

        while(numclients--) {
            listNode *clientnode = listFirst(clients);
            client *receiver = clientnode->value;

            if (receiver->btype != BLOCKED_LIST) {
                /* Put at the tail, so that at the next call
                 * we'll not run into it again. */
                listRotateHeadToTail(clients);
                continue;
            }
            // 取出这个client阻塞的信息
            robj *dstkey = receiver->bpop.target;
            int wherefrom = receiver->bpop.listpos.wherefrom;
            int whereto = receiver->bpop.listpos.whereto;
            robj *value = listTypePop(o, wherefrom);

            if (value) {
                /* Protect receiver->bpop.target, that will be
                 * freed by the next unblockClient()
                 * call. */
                if (dstkey) incrRefCount(dstkey);

                monotime replyTimer;
                elapsedStart(&replyTimer);
                // 回复给等待的client
                if (serveClientBlockedOnList(receiver,
                    rl->key,dstkey,rl->db,value,
                    wherefrom, whereto) == C_ERR)
                {
                    /* If we failed serving the client we need
                     * to also undo the POP operation. */
                    listTypePush(o,value,wherefrom);
                }
                updateStatsOnUnblock(receiver, 0, elapsedUs(replyTimer));
                // 把blockforkeys和blickclient函数做的 反着来一遍
                // 注意上面的blockforkeys是个循环，BLPOP可以阻塞多个list，这里都给去掉block了
                unblockClient(receiver);

                if (dstkey) decrRefCount(dstkey);
                decrRefCount(value);
            } else {
                break;
            }
        }
    }

    if (listTypeLength(o) == 0) {
        dbDelete(rl->db,rl->key);
        notifyKeyspaceEvent(NOTIFY_GENERIC,"del",rl->key,rl->db->id);
    }
    /* We don't call signalModifiedKey() as it was already called
     * when an element was pushed on the list. */
}
```
阻塞的也通知完了，为什么是dbAdd这个时间点通知呢？因为对于redis来说，如果存储的list的长度为0，就将这个key从db的dict中删除，这时候我来blpop，dict中如果存在，就退化成lpop，如果不存在那就block，等别人dbadd进来。从上面的blockingPopGenericCommand也能看到，有数据就直接返回，所有block等待的list都没有数据，才阻塞client。
#### watched_keys
服务于watch命令，watch命令是监视一个key，然后可以开始事务，当提交事务的时候，如果监听的key变化了，则事务失败。下面将redis的事务和watch一起看。  
redis的事务是不具有原子性的，搭配watch可以进行原子性。redis的事务其实仅仅是将命令缓存，提交事务的时候进行执行。
```
// watch跟blocking非常相似，client有个监听key的list，db中有某个key的监听client链表. unwatch跟这个相反，不贴了
void watchForKey(client *c, robj *key) {
    list *clients = NULL;
    listIter li;
    listNode *ln;
    watchedKey *wk;

    /* Check if we are already watching for this key */
    listRewind(c->watched_keys,&li);
    while((ln = listNext(&li))) {
        wk = listNodeValue(ln);
        if (wk->db == c->db && equalStringObjects(key,wk->key))
            return; /* Key already watched */
    }
    /* This key is not already watched in this DB. Let's add it */
    clients = dictFetchValue(c->db->watched_keys,key);
    if (!clients) {
        clients = listCreate();
        dictAdd(c->db->watched_keys,key,clients);
        incrRefCount(key);
    }
    listAddNodeTail(clients,c);
    /* Add the new key to the list of keys watched by this client */
    wk = zmalloc(sizeof(*wk));
    wk->key = key;
    wk->db = c->db;
    incrRefCount(key);
    listAddNodeTail(c->watched_keys,wk);
}

// 事务开始，给client增加MULTI状态
void multiCommand(client *c) {
    if (c->flags & CLIENT_MULTI) {
        addReplyError(c,"MULTI calls can not be nested");
        return;
    }
    c->flags |= CLIENT_MULTI;

    addReply(c,shared.ok);
}

// 可以看到事务处理在processCommand最后面，此时已经查找命令了，如果命令错误直接就是事务失败，这里还没有处理数据，所以如果数据错误，事务并不失败
int processCommand(client *c) {
    ......

    /* Exec the command */
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand &&
        c->cmd->proc != resetCommand)
    {
        // 事务 缓存命令
        queueMultiCommand(c);
        addReply(c,shared.queued);
    } else {
        // 非事务直接执行
        call(c,CMD_CALL_FULL);
        c->woff = server.master_repl_offset;
        if (listLength(server.ready_keys))
            handleClientsBlockedOnKeys();
    }
    return C_OK;
}

typedef struct multiState {
    multiCmd *commands;
    int count;
    int cmd_flags;
    int cmd_inv_flags;
} multiState;

void queueMultiCommand(client *c) {
    multiCmd *mc;
    int j;

    /* 如上所述，如果我们的命令有问题，在回复我们命令出错的同时会给client挂上这个标志，当提交事务的时候，事务中所有语句全都不执行，这里连存都懒得存了 */
    if (c->flags & CLIENT_DIRTY_EXEC)
        return;
    // client中有个事务结构体
    c->mstate.commands = zrealloc(c->mstate.commands,
            sizeof(multiCmd)*(c->mstate.count+1));
    // 找到当前该放command的位置
    mc = c->mstate.commands+c->mstate.count;
    // 命令以及参数放好
    mc->cmd = c->cmd;
    mc->argc = c->argc;
    mc->argv = zmalloc(sizeof(robj*)*c->argc);
    memcpy(mc->argv,c->argv,sizeof(robj*)*c->argc);
    for (j = 0; j < c->argc; j++)
        incrRefCount(mc->argv[j]);
    // 命令数量和flag放好
    c->mstate.count++;
    c->mstate.cmd_flags |= c->cmd->flags;
    c->mstate.cmd_inv_flags |= ~c->cmd->flags;
}

void execCommand(client *c) {
    int j;
    robj **orig_argv;
    int orig_argc;
    struct redisCommand *orig_cmd;
    int was_master = server.masterhost == NULL;
    // 不是事务 返回
    if (!(c->flags & CLIENT_MULTI)) {
        addReplyError(c,"EXEC without MULTI");
        return;
    }

    // 如果watch的有改变，或者命令有错误 直接返回
    // 当有键改变的时候，都会调用`signalModifiedKey`，在此函数中会对watch这个key的client加上dirtycas的标志
    if (c->flags & (CLIENT_DIRTY_CAS|CLIENT_DIRTY_EXEC)) {
        addReply(c, c->flags & CLIENT_DIRTY_EXEC ? shared.execaborterr :
                                                   shared.nullarray[c->resp]);
        discardTransaction(c);
        goto handle_monitor;
    }

    uint64_t old_flags = c->flags;

    /* we do not want to allow blocking commands inside multi */
    c->flags |= CLIENT_DENY_BLOCKING;

    /* Exec all the queued commands */ 这里为什么不取消浪费cpu？
    unwatchAllKeys(c); /* Unwatch ASAP otherwise we'll waste CPU cycles */

    server.in_exec = 1;

    orig_argv = c->argv;
    orig_argc = c->argc;
    orig_cmd = c->cmd;
    addReplyArrayLen(c,c->mstate.count);
    for (j = 0; j < c->mstate.count; j++) {
        c->argc = c->mstate.commands[j].argc;
        c->argv = c->mstate.commands[j].argv;
        c->cmd = c->mstate.commands[j].cmd;

        /* 权限控制 */
        int acl_errpos;
        int acl_retval = ACLCheckAllPerm(c,&acl_errpos);
        if (acl_retval != ACL_OK) {
            char *reason;
            switch (acl_retval) {
            case ACL_DENIED_CMD:
                reason = "no permission to execute the command or subcommand";
                break;
            case ACL_DENIED_KEY:
                reason = "no permission to touch the specified keys";
                break;
            case ACL_DENIED_CHANNEL:
                reason = "no permission to access one of the channels used "
                         "as arguments";
                break;
            default:
                reason = "no permission";
                break;
            }
            addACLLogEntry(c,acl_retval,acl_errpos,NULL);
            addReplyErrorFormat(c,
                "-NOPERM ACLs rules changed between the moment the "
                "transaction was accumulated and the EXEC call. "
                "This command is no longer allowed for the "
                "following reason: %s", reason);
        } else {
            // 执行
            call(c,server.loading ? CMD_CALL_NONE : CMD_CALL_FULL);
            serverAssert((c->flags & CLIENT_BLOCKED) == 0);
        }

        /* Commands may alter argc/argv, restore mstate. */
        c->mstate.commands[j].argc = c->argc;
        c->mstate.commands[j].argv = c->argv;
        c->mstate.commands[j].cmd = c->cmd;
    }

    // restore old DENY_BLOCKING value
    if (!(old_flags & CLIENT_DENY_BLOCKING))
        c->flags &= ~CLIENT_DENY_BLOCKING;

    c->argv = orig_argv;
    c->argc = orig_argc;
    c->cmd = orig_cmd;
    // 释放client事务相关的所有变量
    discardTransaction(c);

    /* Make sure the EXEC command will be propagated as well if MULTI
     * was already propagated. */
    if (server.propagate_in_transaction) {
        int is_master = server.masterhost == NULL;
        server.dirty++;
        /* 这点没看太懂，后面主从再分析 */
        if (server.repl_backlog && was_master && !is_master) {
            char *execcmd = "*1\r\n$4\r\nEXEC\r\n";
            feedReplicationBacklog(execcmd,strlen(execcmd));
        }
        afterPropagateExec();
    }

    server.in_exec = 0;

handle_monitor:
    /* 后面分析 */
    if (listLength(server.monitors) && !server.loading)
        replicationFeedMonitors(c,server.monitors,c->db->id,c->argv,c->argc);
}
```
从上面看到，如果watch的key改变或者命令错误，所有事务命令都不执行，但是如果数据错误，事务依然会执行所有命令。以上就是redis的watch和事务
#### expire
上面db->expire讲到，存储的是有过期时间的key。查找某个key的时候，会判断是否过期，如果过期就删除，如果有一个key有过期时间，但是再也没有访问过，那么redis会主动管理删除。
```
void activeExpireCycle(int type) {
    unsigned long
    // 根据配置
    effort = server.active_expire_effort-1,
    // 一次循环检查多少个key
    config_keys_per_loop = ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP +
                           ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP/4*effort,
    config_cycle_fast_duration = ACTIVE_EXPIRE_CYCLE_FAST_DURATION +
                                 ACTIVE_EXPIRE_CYCLE_FAST_DURATION/4*effort,
    config_cycle_slow_time_perc = ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC +
                                  2*effort,
    config_cycle_acceptable_stale = ACTIVE_EXPIRE_CYCLE_ACCEPTABLE_STALE-
                                    effort;

    // 3个全局变量
    static unsigned int current_db = 0; /* Next DB to test. */
    static int timelimit_exit = 0;      /* Time limit hit in previous call? */
    static long long last_fast_cycle = 0; /* When last fast cycle ran. */

    int j, iteration = 0;
    int dbs_per_call = CRON_DBS_PER_CALL;
    long long start = ustime(), timelimit, elapsed;

    if (checkClientPauseTimeoutAndReturnIfPaused()) return;

    if (type == ACTIVE_EXPIRE_CYCLE_FAST) {
        // 上一个循环不是因为时间过期退出的，并且过期key的比率不高于配置的
        if (!timelimit_exit &&
            server.stat_expired_stale_perc < config_cycle_acceptable_stale)
            return;
        // 上次循环过期没多久，再攒攒
        if (start < last_fast_cycle + (long long)config_cycle_fast_duration*2)
            return;

        last_fast_cycle = start;
    }

    // 每次检查的db数量不能超过总数，如果上次循环是因为时间过期，这次全扫描
    if (dbs_per_call > server.dbnum || timelimit_exit)
        dbs_per_call = server.dbnum;

    // 计算每次循环用的最多的时间
    timelimit = config_cycle_slow_time_perc*1000000/server.hz/100;
    timelimit_exit = 0;
    if (timelimit <= 0) timelimit = 1;

    if (type == ACTIVE_EXPIRE_CYCLE_FAST)
        timelimit = config_cycle_fast_duration; /* in microseconds. */

    long total_sampled = 0;
    long total_expired = 0;

    // 多个db的循环
    for (j = 0; j < dbs_per_call && timelimit_exit == 0; j++) {
        unsigned long expired, sampled;

        // 取当前的db
        redisDb *db = server.db+(current_db % server.dbnum);
        current_db++;

        do {
            unsigned long num, slots;
            long long now, ttl_sum;
            int ttl_samples;
            iteration++;

            // 这个db没有过期key，直接下一个db
            if ((num = dictSize(db->expires)) == 0) {
                db->avg_ttl = 0;
                break;
            }
            // 过期db有多少桶
            slots = dictSlots(db->expires);
            now = mstime();

            // 过期db里面数据小于百分之1
            if (slots > DICT_HT_INITIAL_SIZE &&
                (num*100/slots < 1)) break;

            // 删除过期key的数量  取样数量
            expired = 0;
            sampled = 0;
            ttl_sum = 0;
            ttl_samples = 0;

            if (num > config_keys_per_loop)
                num = config_keys_per_loop;

            // 如果空桶特别多，最多检查应该检查的key数量*20个空桶
            long max_buckets = num*20;
            long checked_buckets = 0;

            while (sampled < num && checked_buckets < max_buckets) {
                // 取哈希表两个table的expires_cursor
                for (int table = 0; table < 2; table++) {
                    if (table == 1 && !dictIsRehashing(db->expires)) break;

                    unsigned long idx = db->expires_cursor;
                    idx &= db->expires->ht[table].sizemask;
                    dictEntry *de = db->expires->ht[table].table[idx];
                    long long ttl;

                    while(de) {
                        // 遍历这个桶
                        dictEntry *e = de;
                        de = de->next;

                        ttl = dictGetSignedIntegerVal(e)-now;
                        // 过期就删除
                        if (activeExpireCycleTryExpire(db,e,now)) expired++;
                        if (ttl > 0) {
                            ttl_sum += ttl;
                            ttl_samples++;
                        }
                        sampled++;
                    }
                }
                // 继续下一个桶
                db->expires_cursor++;
            }
            total_expired += expired;
            total_sampled += sampled;

            // 计算db中过期key的平均ttl。权重平均
            if (ttl_samples) {
                long long avg_ttl = ttl_sum/ttl_samples;

                if (db->avg_ttl == 0) db->avg_ttl = avg_ttl;
                db->avg_ttl = (db->avg_ttl/50)*49 + (avg_ttl/50);
            }

            // 每16次遍历 检查是否到时间限制了
            if ((iteration & 0xf) == 0) { /* check once every 16 iterations. */
                elapsed = ustime()-start;
                if (elapsed > timelimit) {
                    timelimit_exit = 1;
                    server.stat_expired_time_cap_reached_count++;
                    break;
                }
            }
            // 如果当前db空桶过多或者过期率太高，继续这个db
        } while (sampled == 0 ||
                 (expired*100/sampled) > config_cycle_acceptable_stale);
    }

    elapsed = ustime()-start;
    server.stat_expire_cycle_time_used += elapsed;
    latencyAddSampleIfNeeded("expire-cycle",elapsed/1000);

    // 算一下整个过程过期key的比率
    double current_perc;
    if (total_sampled) {
        current_perc = (double)total_expired/total_sampled;
    } else
        current_perc = 0;
    server.stat_expired_stale_perc = (current_perc*0.05)+
                                     (server.stat_expired_stale_perc*0.95);
}
```
整个过程其实就是遍历db->expire哈希表来判断key是否过期。中间增加了很多条件来达成此函数不会占用过多的cpu，并且尽可能的达到用户配置的过期比率。此函数会在beforeSleep调用快速过程，在serverCron时间事件中调用slowtype的过程。

## 总结
redisdb主要结构为哈希表，因此查找速度极快，保证了redis的性能。上面介绍了主要db的使用方式，顺带介绍了普通命令，阻塞命令，事务，过期key处理等重点。
