# 模型
## db
redisdb是redis存储数据的模型。redis默认有16个db，可以进行配置。
```
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
    unsigned long expires_cursor; /* Cursor of the active expire cycle. */
    list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
} redisDb;
```
`dict` 主键哈希表，key-value都存储在这里。  
`expires` 有过期设置的key  
`blocking_keys` 阻塞键  
`ready_keys` 阻塞就绪键  
`watched_keys`  
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
