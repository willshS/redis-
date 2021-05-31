# 模型
## 连接请求
　　[network](../net/network.md)中解释了如何accept一个客户端连接，会创建一个redisclient的对象，通过这个对象注册读事件回调函数`readQueryFromClient`处理客户端请求。
## 请求处理
1. 读数据：`readQueryFromClient`中调用read函数从socket中读出数据到`querybuf`字段
2. 处理协议：`processMultibulkBuffer`处理协议数据，根据协议创建对应的`robj`对象存储到`argc和argv`字段
3. 查找命令：调用`processCommand`查找命令获取`redisCommand`对象，过滤错误命令等
4. 处理命令：过滤完成调用`redisCommand`中预先定义好的处理函数进行处理
5. 生成结果：调用`addReply`将数据写入`buf`并将client对象加入到server的`clients_pending_write`链表
6. 返回结果：在事件循环before钩子中对此链表进行处理，将reply发送回给客户端。如果发送失败，则注册写事件到事件循环中，当有可写事件的时候再次进行发送（此时事件多了CONN_FLAG_WRITE_BARRIER，即下次此socket事件发生先进行写，再读）。

### 读数据
从socket中读取数据，操作比较简单
```
// 将数据读取到querybuf中
qblen = sdslen(c->querybuf);
c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
nread = connRead(c->conn, c->querybuf+qblen, readlen);
```
### 处理协议
```
/*
set key addd
*3
$3
set
$3
key
$4
addd
*/
// qb_pos 处理的位置
while(c->qb_pos < sdslen(c->querybuf)) {
    if (!c->reqtype) {
        if (c->querybuf[c->qb_pos] == '*') {
            // 普通客户端来的数据，多行
            c->reqtype = PROTO_REQ_MULTIBULK;
        } else {
            c->reqtype = PROTO_REQ_INLINE;
        }
    }

    ...

    // 分行处理 将数据转换成long long 存储到 multibulklen
    newline = strchr(c->querybuf+c->qb_pos,'\r');
    ok = string2ll(c->querybuf+1+c->qb_pos,newline-(c->querybuf+1+c->qb_pos),&ll);
    // 就是*3 代表有3行数据
    c->multibulklen = ll;

    ...

    // 逐两行处理数据 例如 $3 \r set 将协议转换为robj对象 存入到argv
    c->argv[c->argc++] = createObject(OBJ_STRING,c->querybuf);
}
```
到此为止协议就处理完了，我们已经将用户数据 "set" "key" "addd" 转换为robj存入了argv中。

### 查找命令
```
// 第一个就是命令 set
c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
```

### 处理命令
```
// 首先根据服务器状态，客户端命令状态，客户端类型等等过滤，命令是否正确等等。
// 处理命令
call(c,CMD_CALL_FULL); // c->cmd->proc(c)
```

### 生成结果
```
// 调用不同命令的回调函数处理命令，并在处理结束后 生成结果放入队列
void addReply(client *c, robj *obj) {
    // 将c放入server.clients_pending_write
    if (prepareClientToWrite(c) != C_OK) return;

    // 结果为sds 或者 string
    if (sdsEncodedObject(obj)) {
        // 放入c->buf，如果满了放入c->reply（双向链表）
        if (_addReplyToBuffer(c,obj->ptr,sdslen(obj->ptr)) != C_OK)
            _addReplyProtoToList(c,obj->ptr,sdslen(obj->ptr));
    } else if (obj->encoding == OBJ_ENCODING_INT) {
        /* For integer encoded strings we just convert it into a string
         * using our optimized function, and attach the resulting string
         * to the output buffer. */
        char buf[32];
        size_t len = ll2string(buf,sizeof(buf),(long)obj->ptr);
        if (_addReplyToBuffer(c,buf,len) != C_OK)
            _addReplyProtoToList(c,buf,len);
    } else {
        serverPanic("Wrong obj->encoding in addReply()");
    }
}
```

### 返回结果
```
// 遍历所有要返回数据的客户端进行处理
listRewind(server.clients_pending_write,&li);
while((ln = listNext(&li))) {
   client *c = listNodeValue(ln);
   c->flags &= ~CLIENT_PENDING_WRITE;
   listDelNode(server.clients_pending_write,ln);


   // 直接写，将c->buf和reply全写回去
   if (writeToClient(c,0) == C_ERR) continue;

   // 上面没写完，还有数据没有发送出去，即socket发送窗口满了，注册事件等待下次发送
   if (clientHasPendingReplies(c)) {
       int ae_barrier = 0;
       /* For the fsync=always policy, we want that a given FD is never
        * served for reading and writing in the same event loop iteration,
        * so that in the middle of receiving the query, and serving it
        * to the client, we'll call beforeSleep() that will do the
        * actual fsync of AOF to disk. the write barrier ensures that. */
       if (server.aof_state == AOF_ON &&
           server.aof_fsync == AOF_FSYNC_ALWAYS)
       {
           ae_barrier = 1;
       }
       if (connSetWriteHandlerWithBarrier(c->conn, sendReplyToClient, ae_barrier) == C_ERR) {
           freeClientAsync(c);
       }
   }
}
```

## 总结
上面流程就是我们客户端如何与服务交互，从数据读取到数据写回的整个流程。这个流程中还有非常多的状态变量，而且对于集群，监控，消息队列等对于当前服务端都是客户端，针对不同客户端类型在此流程中都会进行相应的状态变化和数据处理。还有非常多的细节，比如buf满了添加reply到client的reply链表下次再次进行处理等，比如redis数据类型的序列化和反序列化即robj数据结构的处理，比如命令的处理，set命令会处理先get再set，会处理过期等。  
此流程太过于复杂，目前先梳理主流程，毕竟对于命令的处理过程全部是字符串处理过程，也要对大多命令非常熟悉。对于集群的处理也比普通客户端复杂的多，后续再分块一一分析。
