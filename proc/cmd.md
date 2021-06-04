# 命令处理
[request](../net/request.md)中介绍了客户端和服务端整个交互流程  
## 命令预定义
在server.c中定义了redisCommand redisCommandTable[]数组，此数组定义了redis支持的命令以及其处理函数。redis服务启动会遍历解析此数组，将命令加入到server.commands哈希表中。  
## 命令查询
处理完协议后，根据协议查找命令。查找命令就是从commands哈希表中进行匹配，返回redisCommand对象。  
## 命令处理
根据redisCommand对象中的预定义的条件，对用户传入数据进行鉴别，如果不符合此命令，返回错误。如果用户数据无误，调用回调函数进行处理。  
首先根据命令的`flags`进行分类，根据redis自己分类分的比较多，每个命令是一个或多个flags组成。
`write` 可能会修改键空间  
`read-only` 只读，不会修改数据。有些命令不会从键空间读数据，没有被标记为此flag，如select，time，info等  
`use-memory` 可能会增加内存使用，不允许oom  
`admin` 管理员命令  
`pub-sub` 订阅命令  
`no-script` 命令不允许在脚本中使用  
`random` 随机命令  
`to-sort` 脚本调用的时候，对命令排序，结果输出一定  
`ok-loading` 加载数据时允许  
`ok-stale` 从有脏数据时允许  
`no-monitor` 不自动将命令发送给monitor  
`no-slowlog` 不自动将命令发送到slowlog  
`cluster-asking` Perform an implicit ASKING for this command, so the command will be accepted in cluster mode if the slot is marked as 'importing'  
`fast` 不被延时的O(1) O(log(N))命令  
`may-replicate` 命令可能会产生复制通信量 与 write互斥  

get命令 read-only|fast
```
int getGenericCommand(client *c) {
    robj *o;

    // 去db->dict中查找，没找到直接返回
    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.null[c->resp])) == NULL)
        return C_OK;

    // 检查value类型 如果类型不对 那就不是get命令能拿的
    if (checkType(c,o,OBJ_STRING)) {
        return C_ERR;
    }

    // 返回结果
    addReplyBulk(c,o);
    return C_OK;
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
    propagateExpire(db,key,server.lazyfree_lazy_expire);
    notifyKeyspaceEvent(NOTIFY_EXPIRED,
        "expired",key,db->id);
    signalModifiedKey(NULL,db,key);
    return 1;
}
```
set命令 write|use-memory
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
