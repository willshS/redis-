# 内存处理
redis内存使用超过配置文件设置的最大内存，就需要删除某些数据以降低内存，redis有以下内存管理策略
```
#define MAXMEMORY_FLAG_LRU (1<<0)         // least recently used
#define MAXMEMORY_FLAG_LFU (1<<1)         // least frequently used
#define MAXMEMORY_FLAG_ALLKEYS (1<<2)     // all key
#define MAXMEMORY_FLAG_NO_SHARED_INTEGERS \
    (MAXMEMORY_FLAG_LRU|MAXMEMORY_FLAG_LFU)

#define MAXMEMORY_VOLATILE_LRU ((0<<8)|MAXMEMORY_FLAG_LRU)
#define MAXMEMORY_VOLATILE_LFU ((1<<8)|MAXMEMORY_FLAG_LFU)
#define MAXMEMORY_VOLATILE_TTL (2<<8)
#define MAXMEMORY_VOLATILE_RANDOM (3<<8)
#define MAXMEMORY_ALLKEYS_LRU ((4<<8)|MAXMEMORY_FLAG_LRU|MAXMEMORY_FLAG_ALLKEYS)
#define MAXMEMORY_ALLKEYS_LFU ((5<<8)|MAXMEMORY_FLAG_LFU|MAXMEMORY_FLAG_ALLKEYS)
#define MAXMEMORY_ALLKEYS_RANDOM ((6<<8)|MAXMEMORY_FLAG_ALLKEYS)
#define MAXMEMORY_NO_EVICTION (7<<8)
```
内存管理首先分为两大类：  
1. VOLATILE代表只对有过期时间的key进行逐出
    * lfu // 使用val中的lfu
    * lru // 使用val中的lfu
    * random // 完全随机
    * ttl // 使用db->expire中的when
2. ALLKEYS代表对所有key进行逐出
    * lfu
    * lru
    * random

最后一种不逐出。  
redis就是通过key的管理+是否allkey组合为8种内存管理方法  
逐出数据不可能遍历所有数据，这样性能太差，因此redis进行随机采样，对采样的数据进行对比，再根据策略进行逐出。lfu和lru的说明在robj.md中。
```
// 逐出池
#define EVPOOL_SIZE 16
#define EVPOOL_CACHED_SDS_SIZE 255
struct evictionPoolEntry {
    unsigned long long idle;
    sds key;                 
    sds cached;
    int dbid;
};

// redis对内存的管理是在每一个cmd进来的时候，都会检查是否需要逐出。
int performEvictions(void) {
    if (!isSafeToPerformEvictions()) return EVICT_OK;

    int keys_freed = 0;
    // 使用内存量 需要释放的内存量
    size_t mem_reported, mem_tofree;
    long long mem_freed; /* May be negative */
    mstime_t latency, eviction_latency;
    long long delta;
    int slaves = listLength(server.slaves);
    int result = EVICT_FAIL;
    // 获取当前使用的最大内存信息，判断是否需要逐出，不需要就返回。
    // 因此即便每个命令都会进入此函数，但是并不会影响性能。
    // 其实可以通过cmd的flag是否含有use_memory来进行一个小优化
    if (getMaxmemoryState(&mem_reported,NULL,&mem_tofree,NULL) == C_OK)
        return EVICT_OK;

    // 内存超过max了，但是用户不让逐出
    if (server.maxmemory_policy == MAXMEMORY_NO_EVICTION)
        return EVICT_FAIL;

    // 计算逐出过程最大能耗费的时间，不能过长阻塞
    unsigned long eviction_time_limit_us = evictionTimeLimitUs();
    mem_freed = 0;
    latencyStartMonitor(latency);

    monotime evictionTimer;
    elapsedStart(&evictionTimer);

    while (mem_freed < (long long)mem_tofree) {
        int j, k, i;
        static unsigned int next_db = 0;
        sds bestkey = NULL;
        int bestdbid;
        redisDb *db;
        dict *dict;
        dictEntry *de;
        // lfu或lru策略 或者 ttl策略
        if (server.maxmemory_policy & (MAXMEMORY_FLAG_LRU|MAXMEMORY_FLAG_LFU) ||
            server.maxmemory_policy == MAXMEMORY_VOLATILE_TTL)
        {
            // 获取全局变量
            struct evictionPoolEntry *pool = EvictionPoolLRU;

            while(bestkey == NULL) {
                unsigned long total_keys = 0, keys;

                // 遍历所有db，不能只逐出一个db的
                for (i = 0; i < server.dbnum; i++) {
                    db = server.db+i;
                    // 这里判断是否有allkey，有的话逐出主空间的，没有的话只逐出有过期时间的
                    dict = (server.maxmemory_policy & MAXMEMORY_FLAG_ALLKEYS) ?
                            db->dict : db->expires;
                    if ((keys = dictSize(dict)) != 0) {
                        // 装pool
                        evictionPoolPopulate(i, dict, db->dict, pool);
                        total_keys += keys;
                    }
                }
                if (!total_keys) break; /* No keys to evict. */

                // pool中数据 idle由低到高排序，因此从高开始逐出
                for (k = EVPOOL_SIZE-1; k >= 0; k--) {
                    if (pool[k].key == NULL) continue;
                    // 要删除的key所在dbid
                    bestdbid = pool[k].dbid;

                    if (server.maxmemory_policy & MAXMEMORY_FLAG_ALLKEYS) {
                        de = dictFind(server.db[pool[k].dbid].dict,
                            pool[k].key);
                    } else {
                        de = dictFind(server.db[pool[k].dbid].expires,
                            pool[k].key);
                    }

                    // 从pool中删除
                    if (pool[k].key != pool[k].cached)
                        sdsfree(pool[k].key);
                    pool[k].key = NULL;
                    pool[k].idle = 0;

                    // 判断数据是否真的存在，存在break。
                    if (de) {
                        // 要删除的key
                        bestkey = dictGetKey(de);
                        break;
                    } else {
                        /* Ghost... Iterate again. */
                    }
                }
            }
        }

        /* volatile-random and allkeys-random policy */
        else if (server.maxmemory_policy == MAXMEMORY_ALLKEYS_RANDOM ||
                 server.maxmemory_policy == MAXMEMORY_VOLATILE_RANDOM)
        {
            // 根据next_db获取db，next_db是静态变量，因此我们最终遍历所有db
            // 此时获取随机key，仅仅是从当前db中
            for (i = 0; i < server.dbnum; i++) {
                j = (++next_db) % server.dbnum;
                db = server.db+j;
                dict = (server.maxmemory_policy == MAXMEMORY_ALLKEYS_RANDOM) ?
                        db->dict : db->expires;
                if (dictSize(dict) != 0) {
                    // 随机获取一个key
                    de = dictGetRandomKey(dict);
                    // 要删除的key和db号
                    bestkey = dictGetKey(de);
                    bestdbid = j;
                    break;
                }
            }
        }

        // 这里最终删除
        if (bestkey) {
            db = server.db+bestdbid;
            robj *keyobj = createStringObject(bestkey,sdslen(bestkey));
            propagateExpire(db,keyobj,server.lazyfree_lazy_eviction);
            // 这里计算一下释放了多少空间，不考虑主从和aof使用的内存，因为这两个操作使用的内存迟早释放，只关注主空间使用的内存即可。
            // 否则主从和aof使用内存大于我们要释放的内存，那就永远也释放不完了
            delta = (long long) zmalloc_used_memory();
            latencyStartMonitor(eviction_latency);
            if (server.lazyfree_lazy_eviction)
                dbAsyncDelete(db,keyobj);
            else
                dbSyncDelete(db,keyobj);
            latencyEndMonitor(eviction_latency);
            latencyAddSampleIfNeeded("eviction-del",eviction_latency);
            delta -= (long long) zmalloc_used_memory();
            mem_freed += delta;
            server.stat_evictedkeys++;
            // 通知修改
            signalModifiedKey(NULL,db,keyobj);
            notifyKeyspaceEvent(NOTIFY_EVICTED, "evicted",
                keyobj, db->id);
            decrRefCount(keyobj);
            keys_freed++;

            if (keys_freed % 16 == 0) {
                // 如果释放足够大的内存，可能在这里花的时间太久了，我们的删除操作到不了从，这里强制刷一下
                if (slaves) flushSlavesOutputBuffers();

                // 如果是异步删除 这里检查一下是不是已经符合要求了
                if (server.lazyfree_lazy_eviction) {
                    if (getMaxmemoryState(NULL,NULL,NULL,NULL) == C_OK) {
                        break;
                    }
                }

                // 检查一下时间，如果跑的时间太长也不行，剩下的放到时间事件里面去
                if (elapsedUs(evictionTimer) > eviction_time_limit_us) {
                    // We still need to free memory - start eviction timer proc
                    if (!isEvictionProcRunning) {
                        isEvictionProcRunning = 1;
                        // 0 尽快触发
                        aeCreateTimeEvent(server.el, 0,
                                evictionTimeProc, NULL, NULL);
                    }
                    break;
                }
            }
        } else {
            goto cant_free; /* nothing to free... */
        }
    }
    // 结果 还在逐出过程中 还是已经完成了
    result = (isEvictionProcRunning) ? EVICT_RUNNING : EVICT_OK;

cant_free:
    if (result == EVICT_FAIL) {
        // 跑到这里可能是异步删除。如果后台线程删除作业还有，这里sleep一下，然后检查一下内存结果
        if (bioPendingJobsOfType(BIO_LAZY_FREE)) {
            usleep(eviction_time_limit_us);
            if (getMaxmemoryState(NULL,NULL,NULL,NULL) == C_OK) {
                result = EVICT_OK;
            }
        }
    }

    latencyEndMonitor(latency);
    latencyAddSampleIfNeeded("eviction-cycle",latency);
    return result;
}

void evictionPoolPopulate(int dbid, dict *sampledict, dict *keydict, struct evictionPoolEntry *pool) {
    int j, k, count;
    // 采样数据 可配置 采样越多逐出越精准 但是速度越慢
    dictEntry *samples[server.maxmemory_samples];

    // 随机获取采样数据
    count = dictGetSomeKeys(sampledict,samples,server.maxmemory_samples);
    for (j = 0; j < count; j++) {
        unsigned long long idle;
        sds key;
        robj *o;
        dictEntry *de;

        de = samples[j];
        key = dictGetKey(de);

        // 如果我们从db->expire采样，同样要从主空间中删除
        if (server.maxmemory_policy != MAXMEMORY_VOLATILE_TTL) {
            if (sampledict != keydict) de = dictFind(keydict, key);
            o = dictGetVal(de);
        }

        // 根据策略不同 算出数据的idletime 有lru肯定不是ttl 因此o一定有值
        if (server.maxmemory_policy & MAXMEMORY_FLAG_LRU) {
            // lru数据转换为idletime
            idle = estimateObjectIdleTime(o);
        } else if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
            // 因为我们要逐出最大的idletime，lru计算就是当前时间减去val中的lru，越大代表越不频繁使用，lfu就要计算最近使用次数，因此要反转一下，255是最大使用次数
            idle = 255-LFUDecrAndReturn(o);
        } else if (server.maxmemory_policy == MAXMEMORY_VOLATILE_TTL) {
            // 如果是ttl 直接从expire获取when即可
            idle = ULLONG_MAX - (long)dictGetVal(de);
        } else {
            serverPanic("Unknown eviction policy in evictionPoolPopulate()");
        }

        // 将数据插入到pool中，根据idle进行排序。
        k = 0;
        while (k < EVPOOL_SIZE &&
               pool[k].key &&
               pool[k].idle < idle) k++;
        if (k == 0 && pool[EVPOOL_SIZE-1].key != NULL) {
            // pool满了，并且idle小于pool中最小的idle即 pool[0].idle > idle
            continue;
        } else if (k < EVPOOL_SIZE && pool[k].key == NULL) {
            // 要插入的地方有空位
        } else {
            // 要插入的地方有值，但是后面还有位置
            if (pool[EVPOOL_SIZE-1].key == NULL) {
                // cached是实现分配好的内存
                sds cached = pool[EVPOOL_SIZE-1].cached;
                memmove(pool+k+1,pool+k,
                    sizeof(pool[0])*(EVPOOL_SIZE-k-1));
                pool[k].cached = cached;
            } else {
                // pool满了，pool[0]移除，剩下的左移
                k--;
                sds cached = pool[0].cached; /* Save SDS before overwriting. */
                if (pool[0].key != pool[0].cached) sdsfree(pool[0].key);
                memmove(pool,pool+1,sizeof(pool[0])*k);
                pool[k].cached = cached;
            }
        }

        // 复用cached内存，注释说明，释放和分配空间性能很差。
        // 每次为key分配空间销毁空间的话，分配释放非常频繁，因此尽量复用内存。
        int klen = sdslen(key);
        if (klen > EVPOOL_CACHED_SDS_SIZE) {
            pool[k].key = sdsdup(key);
        } else {
            memcpy(pool[k].cached,key,klen+1);
            sdssetlen(pool[k].cached,klen);
            pool[k].key = pool[k].cached;
        }
        pool[k].idle = idle;
        pool[k].dbid = dbid;
    }
}
```
整个逐出过程就是两个函数，逐出总的来说还是随机的，我们通过对随机出来的采样数据进行层层比较，然后再删除，比如lfu，会从所有db中进行采样，每个db随机maxmemory_samples个采样，所有在一起比较，选出一个最大idletime的进行删除。因此采样越多，逐出就越精准，但是速度就越慢。
## 总结
逐出保证了redis的内存使用不大于配置的maxmemory，这里面涉及到多个配置因素，因此合理配置redis是高性能的关键。
