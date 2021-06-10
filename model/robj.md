# 模型
## robj
### 定义
redis中基本都是通过robj来存储数据，因此robj更像是一个数据接口的封装。
```
#define OBJ_STRING 0    /* String object. */
#define OBJ_LIST 1      /* List object. */
#define OBJ_SET 2       /* Set object. */
#define OBJ_ZSET 3      /* Sorted set object. */
#define OBJ_HASH 4      /* Hash object. */
#define OBJ_MODULE 5    /* Module object. */
#define OBJ_STREAM 6    /* Stream object. */

typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;

    // 内存逐出方式，lru least recently used 最近最少使用 lfu least frequently used 最不经常使用。
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```
`type` 4bit 对象的类型 如上7种基本类型 redis存储都是以基础类型进行存储  
`encoding` 4bit 对象的编码类型 基础类型底层使用的数据结构即为编码类型  
`lru` 24bit   
`refcount` 引用计数  
`ptr` 数据  
```
// 例如我们的dict的类型就是OBJ_SET，编码类型是OBJ_ENCODING_HT
robj *createSetObject(void) {
    dict *d = dictCreate(&setDictType,NULL);
    robj *o = createObject(OBJ_SET,d);
    o->encoding = OBJ_ENCODING_HT;
    return o;
}
```
redis还有intset，整数集合，类型为OBJ_SET，编码类型OBJ_ENCODING_INTSET。
以及zset类型为OBJ_ZSET，编码类型有OBJ_ENCODING_SKIPLIST，OBJ_ENCODING_ZIPLIST。redis将用户数据存在对应的数据结构中并生成robj，统一存储在内存中（db）。  
robj是通过引用计数来进行内存管理，引用计数需要通过自己调用函数来增加和减小。全局变量为INT_MAX，不存在引用计数。
### lfu
lfu : 分钟 << 8 | LFU_INIT_VAL 5 前16位表示时间，单位分钟，后8位表示访问次数的对数LOG_C
```
// 计算对象的LOG_C（减小）
unsigned long LFUDecrAndReturn(robj *o) {
    unsigned long ldt = o->lru >> 8;
    unsigned long counter = o->lru & 255;
    unsigned long num_periods = server.lfu_decay_time ? LFUTimeElapsed(ldt) / server.lfu_decay_time : 0;
    if (num_periods)
        counter = (num_periods > counter) ? 0 : counter - num_periods;
    return counter;
}
```
16位分钟可以表示65536-1分钟。 此函数根据配置的lfu-decay-time 来计算需要减少的数  
1. 首先算当前时间与对象存储的时间差，若当前时间小于记录的则算一轮即65535-robj->lfu+now。  
2. 根据时间差除以lfu-decay-time衰减时间等于周期，counter减去周期即可。如果配置为0则不减。最近最少使用可以通过lfu-decay-time配置周期，比如一个key，他在0-10m使用了100次，但是10m-50m没有任何使用，如果配置为0，则相当于从0-50m这个时间段的counter，如果不为0，则根据时间计算一个衰减的counter = counter - (50m-10m)/lfu-decay-time。这样越久远的它的counter就越少。

```
// lfu 增加
/* 1. A random number R between 0 and 1 is extracted.
   2. A probability P is calculated as 1/(old_value*lfu_log_factor+1).
   3. The counter is incremented only if R < P. */
uint8_t LFULogIncr(uint8_t counter) {
    if (counter == 255) return 255;
    // 第一步： 随机0-1
    double r = (double)rand()/RAND_MAX;
    double baseval = counter - LFU_INIT_VAL;
    if (baseval < 0) baseval = 0;
    // 第二步： 计算增长可能性， counter越大 配置的对数因子越大 则增长可能性越小
    double p = 1.0/(baseval*server.lfu_log_factor+1);
    if (r < p) counter++;
    return counter;
}

// 更新robj的lfu 根据时间先递减，再递增。并更新前16bit的时间
// 此函数是访问robj的时候调用
void updateLFU(robj *val) {
    unsigned long counter = LFUDecrAndReturn(val);
    counter = LFULogIncr(counter);
    val->lru = (LFUGetTimeInMinutes()<<8) | counter;
}
```
### lru
```
unsigned int LRU_CLOCK(void) {
    unsigned int lruclock;
    if (1000/server.hz <= LRU_CLOCK_RESOLUTION) {
        atomicGet(server.lruclock,lruclock);
    } else {
        lruclock = getLRUClock();
    }
    return lruclock;
}

unsigned long long estimateObjectIdleTime(robj *o) {
    unsigned long long lruclock = LRU_CLOCK();
    if (lruclock >= o->lru) {
        return (lruclock - o->lru) * LRU_CLOCK_RESOLUTION;
    } else {
        return (lruclock + (LRU_CLOCK_MAX - o->lru)) *
                    LRU_CLOCK_RESOLUTION;
    }
}
```
lru是24bit表示的秒。计算时间也比较简单，直接获取当前lru-robj的lru即可。访问key时直接获取新的LRU_CLOCK赋值即可。  

## 总结
以上是两种内存管理方法在robj中的实现。其中lfu使用的是16bit的分钟，取值来源于server对象的unixtime，lru使用的是24bit的秒，取值来源于server的lru。无论是哪一种来源，都是在serverCron中更新，serverCron是我们注册的第一个时间事件，并且根据配置的hz来表示执行频率即1000/hz毫秒一次，默认为10，即100ms执行一次。可以配置动态调整，根据客户端连接数进行调整hz，hz越大，则执行频率越高，因此会使用更多的cpu以及更精确地timeout控制。
