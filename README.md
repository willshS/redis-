# redis-study

## this is redis study record

### 数据结构总结
数据结构总结的差不多了，ziplist和zipmap的代码没有进行深入分析，仅仅根据官方注释看懂了原理。  
redis还有一个quicklist没有进行分析，其是通过list+ziplist实现高效使用内存，即每个ziplist是一个节点，根据阈值，插入时超过ziplist大小，则新建ziplist节点，两者通过指针引用形成链表。radix中每个entry使用一块内存存储key，在内存中紧密排列，与quicklist一模一样的思想。从这些可以看到要不是dict这样时间复杂度低，要不是ziplist，quicklist，radix这种内存利用效率极高，普通双向链表adlist都被抛弃，包括zset的底层如果数据量少也用ziplist，redis对内存的高效使用让人感到吃鲸。  
redis速度快，与这些底层的数据结构密不可分，在分析代码的过程中，遇到令人费解的，可以将代码抽出来进行gdb，因为所有数据结构都能独立出来进行编译。例如radix：  
rax.h,rax.c,zmalloc.h,zmalloc.c,rax_malloc.h，atomicvar.h几个文件即可闭环进行编译调试
