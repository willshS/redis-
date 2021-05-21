# 网络模型
## 主流程
### 事件循环的数据模型
```
/* State of an event based program */
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* max number of file descriptors tracked */
    long long timeEventNextId;
    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
    int flags;
} aeEventLoop;
```
redis是基于事件的网络模型。
初始化：
```
//根据setsize初始化aeEventLoop结构体，aeFileEvent与aeFiredEvent大小均为setsize（可以看成约等于redis支持最大连接数）
aeEventLoop *aeCreateEventLoop(int setsize);
```
### redis事件循环处理事件  
redis主函数中调用`aeMain`来进行事件循环，`aeMain`中循环调用`aeProcessEvents`
 ```
int aeProcessEvents(aeEventLoop *eventLoop, int flags);
 ```
主线程事件处理流程：  
1. 获取时间事件列表中的最小值即时间轴中最小的时间事件
2. 根据最小时间事件距离当前的时间作为参数进行`aeApiPoll`(多路复用，即epoll_wait)，若有时间事件小于当前时间则为0，即NoWait
3. 多路复用前钩子函数->多路复用->多路复用后钩子函数
4. 文件事件处理->时间事件处理  
注：若文件事件没有`AE_BARRIER`标志的先进行读再进行写，可以一次循环将客户端请求读取并写回处理结果。若有则先处理写事件再处理读事件  

```
//参数flags为过程处理的参数，主线程事件处理过程和其它线程的参数不同
//flags指定了此过程处理中要处理的事件
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    //flags即不处理时间事件，也不出处理文件事件，直接返回，无意义
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    //这里主线程一定会进，后面比较其它的线程处理过程再来看(TODO:)
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        struct timeval tv, *tvp;
        int64_t usUntilTimer = -1;

        //计算距离最近的时间事件的timer
        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            usUntilTimer = usUntilEarliestTimer(eventLoop);

        if (usUntilTimer >= 0) {
            tv.tv_sec = usUntilTimer / 1000000;
            tv.tv_usec = usUntilTimer % 1000000;
            tvp = &tv;
        } else {
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                tvp = NULL; //阻塞调用 即不处理时间事件 专心等待文件事件
            }
        }


        //调用前钩子
        if (eventLoop->beforesleep != NULL && flags & AE_CALL_BEFORE_SLEEP)
            eventLoop->beforesleep(eventLoop);

        //根据上述计算的timer进行多路复用
        //aeApiPoll将触发事件的触发类型与fd存储到eventLoop的fired字段中
        numevents = aeApiPoll(eventLoop, tvp);

        //后钩子
        if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
            eventLoop->aftersleep(eventLoop);

        for (j = 0; j < numevents; j++) {
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int fired = 0; /* Number of events fired for current fd. */

            //有AE_BARRIER先处理写再处理读，官方解释为如果在多路复用前钩子函数中有刷盘行为，这样处理非常有用。暂时不知道为什么
            int invert = fe->mask & AE_BARRIER;

            //todo：检查fe中的mask是否还有读
            //处理读事件
            if (!invert && fe->mask & mask & AE_READABLE) {
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
                fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
            }

            //处理写事件
            if (fe->mask & mask & AE_WRITABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            //AE_BARRIER的反转
            if (invert) {
                fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
                if ((fe->mask & mask & AE_READABLE) &&
                    (!fired || fe->wfileProc != fe->rfileProc))
                {
                    fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            processed++;
        }
    }
    /* Check time events */
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
```

## 文件事件
文件事件是redis处理socket的事件模型，存储在`aeEventLoop`的`events`字段
```
/* File event structure */
typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE|BARRIER) */
    aeFileProc *rfileProc;
    aeFileProc *wfileProc;
    void *clientData;
} aeFileEvent;

int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData);
void aeDeleteFileEvent(aeEventLoop *eventLoop, int fd, int mask);
```
`mask` 事件类型，读或写等  
`rfileProc` 读取事件的回调函数  
`wfileProc` 写事件的回调函数  
`clientData` 存储client类型数据，表示redisCli的数据结构  
创建文件事件是通过fd进行索引，fd对应的`aeFileEvent`存储数据指针以及事件类型等，当有事件发生，通过fd取出对应的`aeFileEvent`来进行处理，`aeFileProc`就是我们存入到`aeFileEvent`，事件发生取出来执行的回调函数。
```
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }
    //fd作为index绑定eventLoop字段中events 因此在处理事件的时候从events根据fd取出注册进来的aeFileEvent数据
    aeFileEvent *fe = &eventLoop->events[fd];

    //加入事件循环中即epoll_ctl
    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;
    //赋值fe
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    fe->clientData = clientData;
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}
```
我们从监听socket开始看文件事件模型，首先redis根据配置监听端口（bind、listen、设置非阻塞属性等）调用`createSocketAcceptHandler`创建文件事件注册读取回调函数`acceptTcpHandler`，处理客户端tcp连接，在有socket链接上来调用注册的`acceptTcpHandler`回调函数获取fd，使用fd创建一个[connection](#connection)的结构体（其实有cpp对象的味道），然后创建client结构体（todo：client），`connection`结构体中有读取回调，写回调等，设置回调函数就会生成fileEvent注册到事件循环中，当事件发生就会调用对应的回调函数，例如redis-cli链接上来，生成client对象会设置`connection`的读取回调，注册回调函数`readQueryFromClient`，当有读取事件发生，将会调用`readQueryFromClient`函数进行处理。
## 时间事件
时间事件与文件事件比较类似
```
/* Time event structure */
typedef struct aeTimeEvent {
    long long id; /* time event identifier. */
    monotime when;
    aeTimeProc *timeProc;
    aeEventFinalizerProc *finalizerProc;
    void *clientData;
    struct aeTimeEvent *prev;
    struct aeTimeEvent *next;
    int refcount; /* refcount to prevent timer events from being
  		   * freed in recursive time event calls. */
} aeTimeEvent;

long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
        aeTimeProc *proc, void *clientData,
        aeEventFinalizerProc *finalizerProc);
int aeDeleteTimeEvent(aeEventLoop *eventLoop, long long id);
```
`id` 时间事件id是自增的，在每次创建时间事件进行自增，存储在`aeEventLoop`字段`timeEventNextId`  
`when` timer的超时时间，此时间取系统`CLOCK_MONOTONIC`时间（时间使用统一的）  
`timeProc` 时间事件回调函数  
`finalizerProc` 时间事件从事件循环中取消注册时回调函数  
`clientData`  时间事件回调函数处理时的私有数据，作为参数传入回调函数  
`prev` `next` 时间事件为双向链表，head存储在`aeEventLoop`  
`refcount` 防止在递归时间事件调用中释放计时器事件  
时间事件处理比较简单，遍历事件循环中的时间事件列表，如果当前时间事件id为-1，且为非递归，从链表中删除，并调用`finalizerProc`，若果当前时间事件id不为-1，且小于等于now时间，则调用`timeProc`，调用前`refcount++`，调用前`refcount--`，根据回调函数返回值判断此事件是一次性timer还是持续timer，若不为-1则增加`when`，否则id改为-1，惰性删除。

## connection
```
typedef struct ConnectionType {
    void (*ae_handler)(struct aeEventLoop *el, int fd, void *clientData, int mask);
    int (*connect)(struct connection *conn, const char *addr, int port, const char *source_addr, ConnectionCallbackFunc connect_handler);
    int (*write)(struct connection *conn, const void *data, size_t data_len);
    int (*read)(struct connection *conn, void *buf, size_t buf_len);
    void (*close)(struct connection *conn);
    int (*accept)(struct connection *conn, ConnectionCallbackFunc accept_handler);
    int (*set_write_handler)(struct connection *conn, ConnectionCallbackFunc handler, int barrier);
    int (*set_read_handler)(struct connection *conn, ConnectionCallbackFunc handler);
} ConnectionType;

struct connection {
    ConnectionType *type;
    ConnectionState state;
    short int flags;
    short int refs;
    int last_errno;
    void *private_data;
    ConnectionCallbackFunc conn_handler;
    ConnectionCallbackFunc write_handler;
    ConnectionCallbackFunc read_handler;
    int fd;
};
```
可以把`connection`看成一个对象`ConnectionType`便是它的成员函数，type是默认的CT_Socket  
`ae_handler` 注册到事件循环中的任何事件的回调函数都是它，在这个函数中对事件进行区分再次调用`ConnectionCallbackFunc`函数来真正的读或者写。需要明白的是与事件处理循环中相同，对单个fd，当有事件触发时会在事件循环中对读或写进行分开处理，但是普遍情况下我们的注册回调函数都是此函数，因此在这个函数中也有读或写的分开处理，自然也有循环处理中的`WRITE_BARRIER`    
`set_write_handler``set_read_handler` 即为注册文件事件循环，分别注册读和写  
其他的函数其实都对应着相应的系统调用，因此可以看出connection其实就是对tcp的操作的封装

## 总结
redis的事件模型是封装了多个多路复用的框架，整个框架小巧精致，其中对于很多都充满了面向对象的味道，还有很多小细节没有分析到，后面分析集群还会根据过程管理参数不同进行再次分析。
