# 发布订阅
pub sub命令可以作为消息队列
```
int pubsubSubscribeChannel(client *c, robj *channel) {
    dictEntry *de;
    list *clients = NULL;
    int retval = 0;

    // 增加订阅的channel到客户端的订阅列表
    if (dictAdd(c->pubsub_channels,channel,NULL) == DICT_OK) {
        retval = 1;
        incrRefCount(channel);
        // 增加客户端到server的订阅列表 k - client list
        de = dictFind(server.pubsub_channels,channel);
        if (de == NULL) {
            clients = listCreate();
            dictAdd(server.pubsub_channels,channel,clients);
            incrRefCount(channel);
        } else {
            clients = dictGetVal(de);
        }
        listAddNodeTail(clients,c);
    }
    // 订阅成功
    addReplyPubsubSubscribed(c,channel);
    return retval;
}

int pubsubPublishMessage(robj *channel, robj *message) {
    int receivers = 0;
    dictEntry *de;
    dictIterator *di;
    listNode *ln;
    listIter li;

    // 获取订阅此channel的所有client
    de = dictFind(server.pubsub_channels,channel);
    if (de) {
        list *list = dictGetVal(de);
        listNode *ln;
        listIter li;

        listRewind(list,&li);
        while ((ln = listNext(&li)) != NULL) {
            // 回复所有client新发布的client
            client *c = ln->value;
            addReplyPubsubMessage(c,channel,message);
            receivers++;
        }
    }
    // 检查订阅的模式
    di = dictGetIterator(server.pubsub_patterns);
    if (di) {
        channel = getDecodedObject(channel);
        while((de = dictNext(di)) != NULL) {
            robj *pattern = dictGetKey(de);
            list *clients = dictGetVal(de);
            // 模式匹配的话 给订阅此模式的所有客户端发消息
            if (!stringmatchlen((char*)pattern->ptr,
                                sdslen(pattern->ptr),
                                (char*)channel->ptr,
                                sdslen(channel->ptr),0)) continue;

            listRewind(clients,&li);
            while ((ln = listNext(&li)) != NULL) {
                client *c = listNodeValue(ln);
                addReplyPubsubPatMessage(c,pattern,channel,message);
                receivers++;
            }
        }
        decrRefCount(channel);
        dictReleaseIterator(di);
    }
    return receivers;
}
```
订阅发布及其简单，客户端订阅某个channel或者pattern，当有客户端向这个channel发布消息的时候，此消息直接转发给订阅的客户端即可。  
## 总结
pubsub与list的blpop：  
1. blpop已经介绍过了，它所操作的数据最终还是还写入db->dict，对key的所有push操作都会存储到数据库中，如果有客户端来pop，就会将数据库中的数据取出，如果没有数据则等待直到有数据或者超时。因此push发消息的时候完全可以不需要有客户端去pop，它会将数据存储，客户端可以在任意时间取数据。
2. pubsub可以从上面代码看到，pub的任何消息都会直接去sub列表里面找sub的客户端，然后将消息直接发送，如果没有客户端sub，那么这条消息是直接丢掉的，不进行任何存储的，也就不占用数据库空间的。
