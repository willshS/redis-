# 数据结构

## 哈希表
　　哈希表dict：dict.h,dict.c
### 哈希表的定义
```
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v; // union， 存储s64, u64, double优化
    struct dictEntry *next;
} dictEntry;

typedef struct dictType {
    uint64_t (*hashFunction)(const void *key); // 哈希函数
    void *(*keyDup)(void *privdata, const void *key); // key复制
    void *(*valDup)(void *privdata, const void *obj); // val复制
    int (*keyCompare)(void *privdata, const void *key1, const void *key2); //key比较
    void (*keyDestructor)(void *privdata, void *key); // key释放
    void (*valDestructor)(void *privdata, void *obj); // val释放
    int (*expandAllowed)(size_t moreMem, double usedRatio); //是否允许扩容，参数1扩容需要的内存大小，参数2负载 used/size。可以自定义扩容标准
} dictType;

/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
typedef struct dictht {
    dictEntry **table; // 数组，存储dictEntry*
    unsigned long size; // table数组长度
    unsigned long sizemask; // size-1, idx = hash%sizemash
    unsigned long used; // 已经存储的节点数
} dictht;

typedef struct dict {
    dictType *type; // 用户定制哈希表的函数指针集合
    void *privdata;
    dictht ht[2]; // 两个哈希表，在rehash的时候ht[0]表示旧的，ht[1]表示新的，不reahsh，则只使用ht[0]
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
} dict;
```
`dictEntry` 哈希节点，存储key,value,next三个字段(next用于拉链法解决冲突)
`dictType` 哈希字典操作
`dictht` 哈希表
`dict` redis哈希表
### 哈希表的创建和销毁
```
/* Create a new hash table */
dict *dictCreate(dictType *type, void *privDataPtr);
void dictRelease(dict *d);
```
根据传入的dictType初始化一个空的哈希表，其中没有任何元素（此时真正存储数据的数组为空）
### 哈希表的增删改查
#### 增
##### 接口
```
// 添加key value到哈希表中
int dictAdd(dict *d, void *key, void *val);
// 添加key到哈希表中返回哈希节点由用户自己设置值，如果已经存在，返回NULL，存在的节点通过existing返回
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing);
// 同上 对上面函数的封装，一定返回哈希节点（无论是新加的还是已经存在）
dictEntry *dictAddOrFind(dict *d, void *key);
```
##### 流程
添加元素最终调用dictAddRaw获取到节点，再赋值
```
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    long index;
    dictEntry *entry;
    dictht *ht;

    // 如果正在rehash，做一次rehash
    if (dictIsRehashing(d)) _dictRehashStep(d);

    // -1已经存在，节点赋值给existing
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;

    // 在放入下标为index的桶中，赋值为头节点
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    /* Set the hash entry fields. */
    dictSetKey(d, entry, key);
    return entry;
}

static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existing)
{
    unsigned long idx, table;
    dictEntry *he;
    if (existing) *existing = NULL;

    // 如果需要扩容，进行扩容，只有扩容失败才返回DICT_ERR
    // 哈希表创建是空的，这里也会在空的时候进行第一次分配内存，默认是4个桶
    if (_dictExpandIfNeeded(d) == DICT_ERR)
        return -1;
    // 查询是否存在key，若存在，通过existing返回节点，若不存在且没有正在rehash，返回。若不存在且正在rehash，去第二个table中查找，因此如果在第二个table中未找到，返回的自然是第二个桶中的位置，即如果正在rehash，返回新桶的位置
    for (table = 0; table <= 1; table++) {
        idx = hash & d->ht[table].sizemask;
        /* Search if this slot does not already contain the given key */
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                if (existing) *existing = he;
                return -1;
            }
            he = he->next;
        }
        if (!dictIsRehashing(d)) break;
    }
    return idx;
}
```
插入流程总结：
1. 是否正在rehash，是则rehash一次
2. 获取key经过hash后的下标，如果正在rehash，则返回ht[1]中的桶下标。若key存在，则返回节点指针（这时候无需在意在哪个哈希表中，使用节点指针赋值就可以了）
3. 根据下标和是否在rehash，创建节点放入对应的哈希表中的对应的桶中

#### 删
##### 接口
```
// 直接从d中删除key，并对k所在节点，key和value进行释放
int dictDelete(dict *d, const void *key);

// 从dict中删除key，返回key对应的节点指针，不释放key，value和节点
dictEntry *dictUnlink(dict *ht, const void *key);
// 释放从上面函数返回的节点以及节点中的k和v
void dictFreeUnlinkedEntry(dict *d, dictEntry *he);
```
删除有上面两种方式，主要区别在于如果我们想获取某个数据，使用后再删除，如果只有第一种，就需要先find，再删除。如下：
```
entry = dictFind(...);
// Do something with entry
dictDelete(dictionary,entry);
```
两次查找过程。第二种方法如下：
```
entry = dictUnlink(dictionary,entry);
// Do something with entry
dictFreeUnlinkedEntry(entry); // <- This does not need to lookup again.
```
一次查找过程即可。  
##### 流程
删除过程：
```
// 两种删除方法都调用此函数，根据第三个参数nofree进行区分
static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree) {
    uint64_t h, idx;
    dictEntry *he, *prevHe;
    int table;

    /* dict is empty */
    if (dictSize(d) == 0) return NULL;

    if (dictIsRehashing(d)) _dictRehashStep(d);
    h = dictHashKey(d, key);

    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask;
        he = d->ht[table].table[idx];
        prevHe = NULL;
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                // 从桶中的链表中删除
                if (prevHe)
                    prevHe->next = he->next;
                else
                    d->ht[table].table[idx] = he->next;
                // 如果要free，直接free，否则不管，直接返回，需要调用者free
                if (!nofree) {
                    dictFreeUnlinkedEntry(d, he);
                }
                d->ht[table].used--;
                return he;
            }
            prevHe = he;
            he = he->next;
        }
        // 没有在rehash，只在ht[0]中找，找不到直接返回NULL
        if (!dictIsRehashing(d)) break;
    }
    return NULL; /* not found */
}
```
#### 改
```
int dictReplace(dict *d, void *key, void *val)
```
修改其实与添加并无不同，存在则修改值，不存在则添加，需要注意的是如果存在，先设置新值，再释放原值，必须按照此顺序，比如引用计数，先递增再递减。
#### 查
```
dictEntry *dictFind(dict *d, const void *key)
```
查找过程其实在添加，删除等过程中没有任何不同，仅仅是业务原因是否从链表中删除或者获取值所在桶的idx。

### 哈希表的rehash
redis的rehash是渐进式，否则可能会因为哈希表太大，需要rehash太久而导致停止处理客户请求。redis通过使用两张哈希表来进行rehash。
```
int dictRehash(dict *d, int n)
```
#### 扩容的时机和条件
扩容是在我们空间不够用的情况下才需要，因此扩容的时机就是在插入元素的时候，进行是否需要扩容的判断。
```
/* Expand the hash table if needed */
static int _dictExpandIfNeeded(dict *d)
{
    /* Incremental rehashing already in progress. Return. */
    if (dictIsRehashing(d)) return DICT_OK;

    /* If the hash table is empty expand it to the initial size. */
    // 哈希表的真正创建也是在这里进行，初始大小为4
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    // 扩容的三个条件
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio) &&
        dictTypeExpandAllowed(d))
    {
        // 扩容，即生成新的更大的哈希表，并未做元素转移(将rehashidx赋值为0，即从下标0的桶开始rehash)
        return dictExpand(d, d->ht[0].used + 1);
    }
    return DICT_OK;
}
```
扩容的三个条件：  
1. 当前哈希表存储的元素大于桶的个数
2. 允许扩容（动态设置）或存储的元素数量/桶的个数 > 阈值（默认是5，被称为强制扩容）
3. 用户自定义允许扩容函数`int (*expandAllowed)(size_t moreMem, double usedRatio);`即dictType中的函数指针  

以上三个条件全部满足才进行扩容。redis的扩容是生成新的哈希表，因此在子进程的状态变换时设置条件2中的是否允许扩容，即在有子进程做RDB保存，或AOF的时候禁用，以此更好的做copy-on-write（否则在扩容的时候会产生大量内存页的复制）。  
扩容的大小一般为2的x次方，即2^x >= size = used+1。手动扩容可以指定size
#### 扩容的过程
```
/* Performs N steps of incremental rehashing. Returns 1 if there are still
 * keys to move from the old to the new hash table, otherwise 0 is returned.
 *
 * Note that a rehashing step consists in moving a bucket (that may have more
 * than one key as we use chaining) from the old to the new hash table, however
 * since part of the hash table may be composed of empty spaces, it is not
 * guaranteed that this function will rehash even a single bucket, since it
 * will visit at max N*10 empty buckets in total, otherwise the amount of
 * work it does would be unbound and the function may block for a long time. */
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    if (!dictIsRehashing(d)) return 0;

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            uint64_t h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```
注释：参数n，为移动n个桶后返回。因为是渐进式rehash，所以不能将全部元素直接移动到新的哈希表中，而且哈希表中可能有过多的空桶，为了防止访问太多桶而造成长时间的阻塞，访问空桶数量最大为n*10。每次rehash，将一个桶中的所有元素重新计算在新的哈希表中的idx，然后移动过去，修改rehashidx和used。保证其他操作不受影响。
rehash操作在增删查等操作中都会进行移动一个桶的操作，直至rehash结束
### 迭代器
```
dictIterator *dictGetIterator(dict *d);
dictIterator *dictGetSafeIterator(dict *d);
```
dict提供了两种迭代器，一种是安全的，一种是不安全的。  
不安全的迭代器：迭代过程中，若果rehash，rehash可能导致迭代器跳过很多元素，或重复访问某些元素。因此不安全的迭代器只能进行遍历，不能进行哈希表的操作。在初始化不安全的迭代器时，会将哈希表的状态xor异或生成一个“指纹”，析构的时候会检查“指纹”是否相同，若不同就说明用户在迭代过程中做了哈希表的操作如add，delete等。  
安全的迭代器：初始化安全的迭代器会将哈希表的rehash暂停。此时就可以做任何哈希操作而不安全了。(迭代器中存储当前遍历的元素entry和当前元素的下一个nextEntry来保证删除操作不会越界，删除当前元素entry，下次迭代会将nextEntry赋值给entry，若为空，则继续循环下个桶。但是如果我在迭代到entry时，恰巧将nextEntry给删除了，就会访问越界，**因此循环时请仅使用当前entry**)

## 总结
dict通过使用两张哈希表来实现渐进式哈希，通过组合used，size，rehashidx来实现各种功能，并提供暂停哈希功能。通过使用uion存储v进行优化某些元素的存储，sizemask优化速度，rehashidx保证rehash逐步进行保证性能不会被block。并且提供dicttype可以让用户灵活定制自己的哈希表。
