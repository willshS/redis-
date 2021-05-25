# 模型
## 连接请求
　　[network](../net/network.md)中解释了如何accept一个客户端连接，会创建一个redisclient的对象，通过这个对象注册读事件回调函数`readQueryFromClient`处理客户端请求。
## 请求处理
1. 读数据：`readQueryFromClient`中调用read函数从socket中读出数据到`querybuf`字段
2. 处理协议：`processMultibulkBuffer`处理协议数据，根据协议创建对应的`robj`对象存储到`argc和argv`字段
3. 过滤命令：调用`processCommand`查找命令获取`redisCommand`对象，过滤错误命令等
4. 处理命令：过滤完成调用`redisCommand`中预先定义好的处理函数进行处理
5. 生成结果：调用`addReply`将数据写入`buf`并将client对象加入到server的`clients_pending_write`链表
6. 返回结果：在事件循环before钩子中对此链表进行处理，将reply发送回给客户端。如果发送失败，则注册写事件到事件循环中，当有可写事件的时候再次进行发送（此时事件多了CONN_FLAG_WRITE_BARRIER，即下次此socket事件发生先进行写，再读）。

## 总结
上面流程就是我们客户端如何与服务交互，从数据读取到数据写回的整个流程。这个流程中还有非常多的状态变量，而且对于集群，监控，消息队列等对于当前服务端都是客户端，针对不同客户端类型在此流程中都会进行相应的状态变化和数据处理。还有非常多的细节，比如buf满了添加reply到client的reply链表下次再次进行处理等，比如redis数据类型的序列化和反序列化即robj数据结构的处理，比如命令的处理，set命令会处理先get再set，会处理过期等。  
此流程太过于复杂，目前先梳理主流程，毕竟对于命令的处理过程全部是字符串处理过程，也要对大多命令非常熟悉。对于集群的处理也比普通客户端复杂的多，后续再分块一一分析。
