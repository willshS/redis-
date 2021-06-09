# 持久化
[redis持久化](https://redis.io/topics/persistence)官网说明
## rdb
rdb存储的直接是数据，不包含命令。文件中的数据是根据数据类型紧密排列在一起的，因此文件大小相比较于aof非常小。rdb保存的数据是全量数据，每次rdbsave的时候都会将redis内存中所有数据遍历一遍序列化写入到一个rdb文件，因此一次rdb操作是比aof仅仅追加一条命令的代价高得多，因此rdb持久化并不能随时持久化，只能隔一段时间进行一次rdb全量保存数据。因此当恢复数据的时候，rdb可能比aof丢失更多的数据。但是因为直接存储的数据，数据恢复的速度非常快，而aof还需要再次解析命令才能回复数据。代码暂时不分析了todo。

aof rewrite提供了rdb方式重写。aof因为基本每秒一次追加，因此io远远高于rdb（aof追加是在主进程，重写才是在子进程），因此对于redis的性能是有稍微影响的。