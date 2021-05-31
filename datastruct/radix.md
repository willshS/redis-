# 数据结构
## 基数树
基数树是字典树的优化版本，将单链压缩到一个节点中，例如：httpa，httpb，httpc组成的字典树前面4个节点h->t->t->p单链表，基数树会压缩相同前缀http成为一个节点，这样就省掉3个节点，并且大大降低树的高度，内存和性能都高于字典树。  
![radix](./radix.png)

### radix说明
```
typedef struct raxNode {
    uint32_t iskey:1;     /* Does this node contain a key? */
    uint32_t isnull:1;    /* Associated value is NULL (don't store it). */
    uint32_t iscompr:1;   /* Node is compressed. */
    uint32_t size:29;     /* Number of children, or compressed string len. */
    /* 当不压缩的时候，data中有size个子节点
     * [header iscompr=0][abc][padding][a-ptr][b-ptr][c-ptr](value-ptr?)
     * 当压缩的时候，data中只有一个子节点
     * [header iscompr=1][xyz][padding][z-ptr](value-ptr?)
     */
    unsigned char data[]; // 柔性数组
} raxNode;

typedef struct rax {
    raxNode *head;
    uint64_t numele;
    uint64_t numnodes;
} rax;
```
`data` 是一个柔性数组，中间存储了元素+对齐+子节点指针+value指针（如果存在的话），因此以下宏要弄明白：  

1.
```
#define raxPadding(nodesize) ((sizeof(void*)-((nodesize+4) % sizeof(void*))) & (sizeof(void*)-1))
```
返回需要添加的对齐内存。以64位为例，nodesize是node->size，例如http就是4。因此我们可以确定4字节的header（sizeof(raxNode)）+ size这一部分要对齐，因此8-(size+4)对8字节取余即可。32位同理。  

2.
```
#define raxNodeCurrentLength(n) ( \
    sizeof(raxNode)+(n)->size+ \
    raxPadding((n)->size)+ \
    ((n)->iscompr ? sizeof(raxNode*) : sizeof(raxNode*)*(n)->size)+ \
    (((n)->iskey && !(n)->isnull)*sizeof(void*)) \
)
```
返回节点长度。压缩：header+size+padding+一个子节点+一个value指针。非压缩：header+size+padding+size个子节点+一个value指针。  

3.
```
#define raxNodeLastChildPtr(n) ((raxNode**) ( \
    ((char*)(n)) + \
    raxNodeCurrentLength(n) - \
    sizeof(raxNode*) - \
    (((n)->iskey && !(n)->isnull) ? sizeof(void*) : 0) \
))
```
返回最后一个子节点指针。节点指针+长度-最后一个子节点指针-value指针  

4.
```
#define raxNodeFirstChildPtr(n) ((raxNode**) ( \
    (n)->data + \
    (n)->size + \
    raxPadding((n)->size)))
```
返回第一个子节点指针。data+size+padding  

**注意：value可能为空，如果为空则内存不包含value。**  
以上几个宏完全说明白了节点内部的内存结构。通过使用柔性数组，内存利用非常高效便捷。

### 初始化
```
// 直接包含了一个header，但是header中没有key，没有value
rax *raxNew(void) {
    rax *rax = rax_malloc(sizeof(*rax));
    if (rax == NULL) return NULL;
    rax->numele = 0;
    rax->numnodes = 1;
    rax->head = raxNewNode(0,0);
    if (rax->head == NULL) {
        rax_free(rax);
        return NULL;
    } else {
        return rax;
    }
}

/* Set the node auxiliary data to the specified pointer. */
void raxSetData(raxNode *n, void *data) {
    n->iskey = 1;
    if (data != NULL) {
        n->isnull = 0;
        void **ndata = (void**)
            ((char*)n+raxNodeCurrentLength(n)-sizeof(void*));
        // 这里*ndata = data也可以
        memcpy(ndata,&data,sizeof(data));
    } else {
        n->isnull = 1;
    }
}

/* Get the node auxiliary data. */
void *raxGetData(raxNode *n) {
    if (n->isnull) return NULL;
    void **ndata =(void**)((char*)n+raxNodeCurrentLength(n)-sizeof(void*));
    void *data;
    memcpy(&data,ndata,sizeof(data));
    return data;
}
```
基数树中存储的全部是指针。

### 查找
无论是插入还是删除都需要使用到查找
```
// 此函数查找s，返回停止查找的节点即不匹配的节点，返回值为s匹配的字节数
// 若返回值为0，则h与s完全不匹配，h可能为空节点，例如第一次插入返回header
// 若返回值>0,<len,则部分匹配即路径是s的子串，splitpos有值则h为匹配的压缩节点，反之h可能为空节点或完全不匹配的节点（部分匹配肯定会继续遍历，匹配失败自然返回子节点，比如：现在有两个节点h->i,查找ha,h匹配，子节点i（无论是否压缩）不匹配被返回）。
// 若返回值==len，则完全匹配，返回节点h，此时splitpos有值，则不存在
static inline size_t raxLowWalk(rax *rax, unsigned char *s, size_t len, raxNode **stopnode, raxNode ***plink, int *splitpos, raxStack *ts) {
    raxNode *h = rax->head;
    raxNode **parentlink = &rax->head; // 1? 这个变量好像没有任何作用，一直等于h，而且外部也没有进行使用，根据命名应该是h的父亲？那么后面应该是parentlink = h;memcpy(&h,children+j,sizeof(h));吧


    size_t i = 0; /* Position in the string. */
    size_t j = 0; /* Position in the node children (or bytes if compressed).*/
    while(h->size && i < len) {
        debugnode("Lookup current node",h);
        unsigned char *v = h->data;

        if (h->iscompr) {
            // 如果是压缩的，就与h节点中的数据逐个对比
            for (j = 0; j < h->size && i < len; j++, i++) {
                if (v[j] != s[i]) break;
            }
            // 不是完全相同，直接返回，因为是压缩的，这里j就是要分割此压缩节点的地方
            // 完全相同，继续子节点查找
            if (j != h->size) break;
        } else {
            // 如果不是压缩的，有一个相同即可，继续遍历第j个子节点
            for (j = 0; j < h->size; j++) {
                if (v[j] == s[i]) break;
            }
            // j==h->size 没有一个相同，直接返回，完全不匹配
            // 否则有相同的，继续子节点，并s的下一位继续匹配
            if (j == h->size) break;
            i++;
        }

        if (ts) raxStackPush(ts,h); /* Save stack of parent nodes. */
        // 获取子节点
        raxNode **children = raxNodeFirstChildPtr(h);
        if (h->iscompr) j = 0; // 2? 要去遍历子节点j怎么样都要变成0，没看懂这个操作
        memcpy(&h,children+j,sizeof(h));
        parentlink = children+j;
        j = 0; /* If the new node is non compressed and we do not
                  iterate again (since i == len) set the split
                  position to 0 to signal this node represents
                  the searched key. */
    }
    debugnode("Lookup stop node is",h);
    if (stopnode) *stopnode = h;
    if (plink) *plink = parentlink;
    if (splitpos && h->iscompr) *splitpos = j;
    return i;
}

void *raxFind(rax *rax, unsigned char *s, size_t len) {
    raxNode *h;

    debugf("### Lookup: %.*s\n", (int)len, s);
    int splitpos = 0;
    size_t i = raxLowWalk(rax,s,len,&h,NULL,&splitpos,NULL);
    // i不等于len 代表没有完全匹配  匹配到的h是压缩的，并且分割点不等于0，代表key不存在
    if (i != len || (h->iscompr && splitpos != 0) || !h->iskey)
        return raxNotFound;
    return raxGetData(h);
}
```
### 插入
