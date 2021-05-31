# 模型
## 命令
用户数据从socket读取出来后，解析为redisCommand，再进行执行。
```
typedef void redisCommandProc(client *c);
typedef int redisGetKeysProc(struct redisCommand *cmd, robj **argv, int argc, getKeysResult *result);
struct redisCommand {
    char *name;
    redisCommandProc *proc;
    int arity;
    char *sflags;   /* Flags as string representation, one char per flag. */
    uint64_t flags; /* The actual flags, obtained from the 'sflags' field. */
    /* Use a function to determine keys arguments in a command line.
     * Used for Redis Cluster redirect. */
    redisGetKeysProc *getkeys_proc;
    /* What keys should be loaded in background when calling this command? */
    int firstkey; /* The first argument that's a key (0 = no keys) */
    int lastkey;  /* The last argument that's a key */
    int keystep;  /* The step between first and last key */
    long long microseconds, calls, rejected_calls, failed_calls;
    int id;     /* Command ID. This is a progressive ID starting from 0 that
                   is assigned at runtime, and is used in order to check
                   ACLs. A connection is able to execute a given command if
                   the user associated to the connection has this command
                   bit set in the bitmap of allowed commands. */
};
```
redisCommand会事先定义好redis支持的各种命令以及他们的回调函数
```
{"set",setCommand,-3,
     "write use-memory @string",
     0,NULL,1,1,1,0,0,0},
```
首先检查key是否过期，如果过期进行删除。通过lookupKey找到要处理的key，如果找到，则更新他们的lru和lfu，没找到则添加，然后调用对应的数据结构接口对数据进行修改。修改完成后添加回复到客户端。
`name` set get 等命令字符串
`proc` 命令处理函数指针
`arity` 参数数目， -N表示>=N
`sflags flags` 命令标志，定义了这个命令是什么类型的命令（写，只读，使用内存，随机，脚本，排序等等）
`getkeys_proc` 从命令中获取键参数的可选函数
`firstkey lastkey keystep` 第一个参数，最后一个参数，参数的步数，例如mset key,value,key,value 步数是2
`microseconds` 命令总执行时间
`calls` 命令调用总数
`id` 访问控制
注：flags、microseconds和calls字段由Redis计算，应该始终设置为零。
