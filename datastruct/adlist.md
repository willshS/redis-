# 数据结构
## 双向链表
　　双向链表：adlist.h|c
### 链表定义
```
// 链表节点
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

// 迭代器
typedef struct listIter {
    listNode *next;
    int direction;
} listIter;

// 链表
typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;

```
链表定义包含头节点，尾节点，链表长度和三个可以自定义的节点操作函数  
链表节点包含前指针，后指针和节点值，节点值为void\*，因此可以存放任何类型数据，但是同一链表不能存储不同类型值，因为没办法判断void\*中到底是什么类型，使用此链表必须自己进行约束  
### 链表创建和销毁
```
// 创建空链表，返回空链表指针
list *listCreate(void);
// 释放链表（释放所有链表中的节点）
void listRelease(list *list);
// 释放链表结构体
void listEmpty(list *list);
```
**注：必须在调用此函数前自己释放节点中的值，或设置链表中的free函数释放节点，否则内存泄露**
### 链表节点添加和删除
```
// 添加节点到链表头节点
list *listAddNodeHead(list *list, void *value);
// 添加节点到链表尾节点
list *listAddNodeTail(list *list, void *value);
// 根据after添加节点到old_node之前或之后
list *listInsertNode(list *list, listNode *old_node, void *value, int after);
// 删除节点 内存管理同链表销毁
void listDelNode(list *list, listNode *node);
```
### 链表迭代器
迭代器中只包含下一元素和方向两个字段  
```
// 获取参数list的迭代器，direction为0则从头节点开始遍历，反之从尾节点开始
listIter *listGetIterator(list *list, int direction);
// 根据迭代器获取下一个节点
listNode *listNext(listIter *iter);
// 释放迭代器
void listReleaseIterator(listIter *iter);
```
获取迭代器是将头节点或者尾节点赋值给迭代器的next，遍历只需要调用listnext根据方向获取pre或者next即可，使用方式如下：
```
iter = listGetIterator(list,<direction>);
while ((node = listNext(iter)) != NULL) {
   doSomethingWith(listNodeValue(node));
}
```
通过迭代器获取的节点可以进行删除，这是安全的，但是不能删除其他节点，否则迭代器可能不安全。
### 链表的其他操作
链表的复制，查询等都与链表的复制匹配函数指针有关，若复制函数不为空（需要你自己在此函数中进行深拷贝并返回新的数据的指针），则进行深拷贝，为空则仅仅复制指针。查询同理。

## 总结
双向链表操作比较简单，redis提供设置复制、查询、释放三个函数指针来保证对用户数据的操作的正确性，否则需要用户自己进行管理，比较复杂。
