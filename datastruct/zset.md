# 数据结构
## 有序集合
zset是redis实现的有序集合，它并不是一个基础数据结构，而是在基础数据结构上实现的，其所有操作都基于基础数据结构的操作。
### 创建与释放
```
robj *createZsetObject(void) {
    zset *zs = zmalloc(sizeof(*zs));
    robj *o;

    zs->dict = dictCreate(&zsetDictType,NULL);
    zs->zsl = zslCreate();
    o = createObject(OBJ_ZSET,zs);
    o->encoding = OBJ_ENCODING_SKIPLIST;
    return o;
}

robj *createZsetZiplistObject(void) {
    unsigned char *zl = ziplistNew();
    robj *o = createObject(OBJ_ZSET,zl);
    o->encoding = OBJ_ENCODING_ZIPLIST;
    return o;
}

void freeZsetObject(robj *o) {
    zset *zs;
    switch (o->encoding) {
    case OBJ_ENCODING_SKIPLIST:
        zs = o->ptr;
        dictRelease(zs->dict);
        zslFree(zs->zsl);
        zfree(zs);
        break;
    case OBJ_ENCODING_ZIPLIST:
        zfree(o->ptr);
        break;
    default:
        serverPanic("Unknown sorted set encoding");
    }
}
```
可以从api看出zset有两种，一种是基于ziplist的，一种是基于dict和skiplist的。zset对象是存储为robj，根据type和encodeing来确定编码类型。robj后续会进行梳理。
[ziplist](./ziplist.md)  
[skiplist](./skiplist.md)  
[dict](./dict.md)
### 插入和删除

#### 插入
```
int zsetAdd(robj *zobj, double score, sds ele, int in_flags, int *out_flags, double *newscore) {
    // 根据命令的状态
    /* ZADD_INCR: 增量操作，不存在就新增
    * ZADD_NX:   不更新存在的成员。只添加新成员。
    * ZADD_XX:   仅仅更新存在的成员，不添加新成员。
    * ZADD_GT:   更新新的分值比当前分值大的成员，不存在则新增。
    * ZADD_LT:   更新新的分值比当前分值小的成员，不存在则新增。
    int incr = (in_flags & ZADD_IN_INCR) != 0;
    int nx = (in_flags & ZADD_IN_NX) != 0;
    int xx = (in_flags & ZADD_IN_XX) != 0;
    int gt = (in_flags & ZADD_IN_GT) != 0;
    int lt = (in_flags & ZADD_IN_LT) != 0;
    *out_flags = 0; /* We'll return our response flags. */
    double curscore;

    /* score不是NAN */
    if (isnan(score)) {
        *out_flags = ZADD_OUT_NAN;
        return 0;
    }

    /* 当底层是ziplist的时候 */
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        unsigned char *eptr;

        // ziplist做有序集合的底层，存储方式为ele后紧接score，即两个ziplist entry为一组，组成一个有序集合的节点
        // 查找ele，获取相同ele的指针
        if ((eptr = zzlFind(zobj->ptr,ele,&curscore)) != NULL) {
            // ziplist中存在ele
            /* NX? Return, same element already exists. */
            if (nx) {
                *out_flags |= ZADD_OUT_NOP;
                return 1;
            }

            /* Prepare the score for the increment if needed. */
            if (incr) {
                score += curscore;
                if (isnan(score)) {
                    *out_flags |= ZADD_OUT_NAN;
                    return 0;
                }
            }

            /* GT/LT? Only update if score is greater/less than current. */
            if ((lt && score >= curscore) || (gt && score <= curscore)) {
                *out_flags |= ZADD_OUT_NOP;
                return 1;
            }

            if (newscore) *newscore = score;

            /* 要插入的数据不同于已经存在的，删除，重新插入 */
            if (score != curscore) {
                zobj->ptr = zzlDelete(zobj->ptr,eptr);
                zobj->ptr = zzlInsert(zobj->ptr,ele,score);
                *out_flags |= ZADD_OUT_UPDATED;
            }
            return 1;
        } else if (!xx) {
            /* Optimize: check if the element is too large or the list
             * becomes too long *before* executing zzlInsert. */
            // 插入后检查ziplist中节点（此节点是有序集合节点，即ziplist entry/2）个数和插入元素大小，若超过阈值，转换为skiplist的zset
            zobj->ptr = zzlInsert(zobj->ptr,ele,score);
            if (zzlLength(zobj->ptr) > server.zset_max_ziplist_entries ||
                sdslen(ele) > server.zset_max_ziplist_value)
                zsetConvert(zobj,OBJ_ENCODING_SKIPLIST);
            if (newscore) *newscore = score;
            *out_flags |= ZADD_OUT_ADDED;
            return 1;
        } else {
            // 有xx选项，但是元素不存在
            *out_flags |= ZADD_OUT_NOP;
            return 1;
        }
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
      // skiplist + dict 时
        zset *zs = zobj->ptr;
        zskiplistNode *znode;
        dictEntry *de;
        // 先从字典中查找
        de = dictFind(zs->dict,ele);
        if (de != NULL) {
            /* NX? Return, same element already exists. */
            if (nx) {
                *out_flags |= ZADD_OUT_NOP;
                return 1;
            }
            // 获取找到节点的score
            curscore = *(double*)dictGetVal(de);

            /* Prepare the score for the increment if needed. */
            if (incr) {
                score += curscore;
                if (isnan(score)) {
                    *out_flags |= ZADD_OUT_NAN;
                    return 0;
                }
            }

            /* GT/LT? Only update if score is greater/less than current. */
            if ((lt && score >= curscore) || (gt && score <= curscore)) {
                *out_flags |= ZADD_OUT_NOP;
                return 1;
            }

            if (newscore) *newscore = score;

            /* Remove and re-insert when score changes. */
            if (score != curscore) {
                // 更新skiplist中的值（skiplist自带排序）
                znode = zslUpdateScore(zs->zsl,curscore,ele,score);
                // 更新dict中的数据
                dictGetVal(de) = &znode->score; /* Update score ptr. */
                *out_flags |= ZADD_OUT_UPDATED;
            }
            return 1;
        } else if (!xx) {
            // 注意 这里会复制一次ele
            ele = sdsdup(ele);
            znode = zslInsert(zs->zsl,score,ele);
            serverAssert(dictAdd(zs->dict,ele,&znode->score) == DICT_OK);
            *out_flags |= ZADD_OUT_ADDED;
            if (newscore) *newscore = score;
            return 1;
        } else {
            *out_flags |= ZADD_OUT_NOP;
            return 1;
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return 0; /* Never reached. */
}
```
无论底层是ziplist还是skiplist，主流程都一样。这里需要额外说明的三点如下：
1. ziplist是压缩列表，内部存储的时候会进行编码并将数据复制一份。因此新增元素时会拷贝ele。
skiplist为底层的时候，上面手动复制一份ele，存入skiplist和dict。因此skiplist和dict中相同元素指向的数据ele是同一份。  
综上所述，无论是什么底层，zset传进来的ele的内存还是归调用者自己管理。  
2. skiplist为底层，是通过dict来进行去重，skiplist来实现排序，两者综合就是有序集合。  
3. ziplist为底层的时候，元素是<ele><score>对，两个相邻entry表示一个有序集合的节点。从第0个entry开始，[0,1],[2,3]....[n-1][n]，n % 2 == 0为true。

#### 删除
```
int zsetDel(robj *zobj, sds ele) {
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        unsigned char *eptr;

        // ziplist中找到节点，并删除
        if ((eptr = zzlFind(zobj->ptr,ele,NULL)) != NULL) {
            zobj->ptr = zzlDelete(zobj->ptr,eptr);
            return 1;
        }
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        // 从skiplist中删除
        if (zsetRemoveFromSkiplist(zs, ele)) {
            if (htNeedsResize(zs->dict)) dictResize(zs->dict);
            return 1;
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return 0; /* No such element found. */
}

static int zsetRemoveFromSkiplist(zset *zs, sds ele) {
    dictEntry *de;
    double score;

    de = dictUnlink(zs->dict,ele);
    if (de != NULL) {
        /* Get the score in order to delete from the skiplist later. */
        score = *(double*)dictGetVal(de);

        /* Delete from the hash table and later from the skiplist.
         * Note that the order is important: deleting from the skiplist
         * actually releases the SDS string representing the element,
         * which is shared between the skiplist and the hash table, so
         * we need to delete from the skiplist as the final step. */
        dictFreeUnlinkedEntry(zs->dict,de);

        /* Delete from skiplist. */
        int retval = zslDelete(zs->zsl,score,ele,NULL);
        serverAssert(retval);

        return 1;
    }

    return 0;
}
```
从注释可以看到先从dict删除，再从skiplist删除，因为skiplist是真正释放ele的地方。

## 总结
zset是有序集合，有两种实现方式，当节点过多或某个节点大小过大，就会强制转换为skiplist+dict来实现。（todo:集合合并尚未看到，找到在补充）
