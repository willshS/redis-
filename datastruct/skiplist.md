# 数据结构
## 跳表
增加多层索引的链表结构  
![sikp](./skiplist.png)
### skiplist
跳表，多层链表。
```
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele; // 存储的数据
    double score; // 分数 用户传入 依此排序
    struct zskiplistNode *backward; // 当前节点的前一个节点 只存在与0层
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 当前节点的当前层(level的idx为第几层)的下一个节点
        unsigned long span; // 到下一个节点的距离(n到m的距离即0层的m-n+1)
    } level[]; // 层 数组
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail; // 头尾指针
    unsigned long length; // 链表长度
    int level; // 总层数
} zskiplist;
```

#### 创建和释放
```
zskiplistNode *zslCreateNode(int level, double score, sds ele) {
    zskiplistNode *zn =
        zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score;
    zn->ele = ele;
    return zn;
}

/* Create a new skiplist. */
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1;
    zsl->length = 0;
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}

/* Free the specified skiplist node. The referenced SDS string representation
 * of the element is freed too, unless node->ele is set to NULL before calling
 * this function. */
void zslFreeNode(zskiplistNode *node) {
    sdsfree(node->ele);
    zfree(node);
}

/* Free a whole skiplist. */
void zslFree(zskiplist *zsl) {
    zskiplistNode *node = zsl->header->level[0].forward, *next;

    zfree(zsl->header);
    while(node) {
        next = node->level[0].forward;
        zslFreeNode(node);
        node = next;
    }
    zfree(zsl);
}
```
