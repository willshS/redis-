# 数据结构
## 字符串
　　字符串声明与实现都在sds.h和sds.c中
### 字符串的声明
```
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```
redis将字符串声明为不同类型的结构体来存储，len代表使用长度，alloc代表分配空间大小，flags是使用了哪种结构体。Redis中真正使用的是sds，sds为char *类型。通过对sds指针进行操作获取上述结构体的数据。`#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))` 注意sdshdr5官方注释为不使用，因此以下不讨论在内  
注：sdshdr5的flags中存有长度信息，其它类型flags为与5保持一致，只使用前3位。  
以下为长度对应的结构体类型:
```
static inline char sdsReqType(size_t string_size) {
    if (string_size < 1<<5)
        return SDS_TYPE_5;
    if (string_size < 1<<8)
        return SDS_TYPE_8;
    if (string_size < 1<<16)
        return SDS_TYPE_16;
#if (LONG_MAX == LLONG_MAX)
    if (string_size < 1ll<<32)
        return SDS_TYPE_32;
    return SDS_TYPE_64;
#else
    return SDS_TYPE_32;
#endif
}
```

### 字符串操作  
字符串简单操作如获取len，获取alloc等函数都是根据sds类型获得结构体中字段的值，比较简单不进行解释。sdshdr的创建与释放，过程如下：
```
sds _sdsnewlen(const void *init, size_t initlen, int trymalloc) {
    //1.根据字符串长度获取不同类型 （sdshdr8 即 长度 小于 1 << 8），然后加上对应类型的sdshdr的结构体大小，使用malloc分配内存（此处使用的zmalloc）

    char type = sdsReqType(initlen);
    int hdrlen = sdsHdrSize(type);
    sh = s_malloc_usable(hdrlen+initlen+1, &usable);
    memset(sh, 0, hdrlen+initlen+1);

    //2.根据类型初始化结构体

    case SDS_TYPE_32: {
        SDS_HDR_VAR(32,s);
        sh->len = initlen;
        sh->alloc = usable;
        *fp = type;
        break;
    }

    //3.拷贝结构体

    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';
    return s;
}

void sdsfree(sds s) {
    if (s == NULL) return;
    s_free((char*)s-sdsHdrSize(s[-1]));
}
```
上面代码只有部分，需要注意的是sds中自带'\0'，因此分配内存的时候需要+1。  
字符串的扩容和缩容：
```
sds sdsMakeRoomFor(sds s, size_t addlen) {

    //1.查看剩余可用空间是否大于需要扩容的空间，大于直接返回

    void *sh, *newsh;
    size_t avail = sdsavail(s);
    size_t len, newlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;
    size_t usable;

    /* Return ASAP if there is enough space left. */
    if (avail >= addlen) return s;

    //2.计算扩容后需要的大小，并获取对应类型

    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    newlen = (len+addlen);
    assert(newlen > len);   /* Catch size_t overflow */
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;

    type = sdsReqType(newlen);

    /* Don't use type 5: the user is appending to the string and type 5 is
     * not able to remember empty space, so sdsMakeRoomFor() must be called
     * at every appending operation. */
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    //3.若扩容后类型不变，在当前空间则重新分配内存即可  
    //  若扩容后类型改变，则重新分配内存，旧内存数据拷贝到新内存，并销毁旧内存，对新结构体进行赋值  
    //最终更改可用空间即可

    hdrlen = sdsHdrSize(type);
    assert(hdrlen + newlen + 1 > len);  /* Catch size_t overflow */
    if (oldtype==type) {
        newsh = s_realloc_usable(sh, hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        /* Since the header size changes, need to move the string forward,
         * and can't use realloc */
        newsh = s_malloc_usable(hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    usable = usable-hdrlen-1;
    if (usable > sdsTypeMaxSize(type))
        usable = sdsTypeMaxSize(type);
    sdssetalloc(s, usable);
    return s;
}
```
扩容与缩容大同小异。字符串的创建销毁，扩容缩容就是一个字符串的骨架，其余的拷贝，拼接，格式化，获取子串等大多都是基于上面介绍的功能。

### 总结
sds即为redis使用c实现的 string 类。为什么要实现字符串类型而不直接使用c的char*?
* redis根据字符串的长度用不同的结构体来存储，根据长度进行扩容缩容，使用内存更高效。  
* 存储长度，获取长度直接返回。并且根据长度判断结束位置，具有**二进制安全性**（即字符串中即便有'\0'也能正确返回）
