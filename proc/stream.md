# 消息队列
## 实现
stream是redis实现的消息队列。底层类型是rax+listpack。  
streamid由两部分组成，4字节的unix毫秒time+4字节的seq。我们可以指定streamid但是必须自己保持递增，或由redis帮我们递增，每往stream增加一条消息，redis获取当前时间与last_id比较，如果相同，last_id的seq++就是新的streamid，否则seq从0开始。  
rax的key是StreamId，因此每个有val的节点都是一个streamid，val是一个listpack（类似于ziplist的字符串压缩表），listpack会存储多条消息直到配置的stream_node_max_bytes和stream_node_max_entries（和ziplist一样，O(n)，条目越多效率越低，内存越大效率越低）
```
typedef struct stream {
    rax *rax;               /* The radix tree holding the stream. */
    uint64_t length;        /* Number of elements inside this stream. */
    streamID last_id;       /* Zero if there are yet no items. */
    rax *cgroups;           /* Consumer groups dictionary: name -> streamCG */
} stream;


int streamAppendItem(stream *s, robj **argv, int64_t numfields, streamID *added_id, streamID *use_id) {

    // 根据last_id生成新id
    streamID id;
    if (use_id)
        id = *use_id;
    else
        streamNextID(&s->last_id,&id);

    // 检查id 必须保证递增
    if (streamCompareID(&id,&s->last_id) <= 0) return C_ERR;

    // 找到字典树的尾节点 （树的最右叶子节点，也是当前streamid最大的叶子节点）
    raxIterator ri;
    raxStart(&ri,s->rax);
    raxSeek(&ri,"$",NULL,0);

    size_t lp_bytes = 0;        /* Total bytes in the tail listpack. */
    unsigned char *lp = NULL;   /* Tail listpack pointer. */

    // 获取listpack
    if (raxNext(&ri)) {
        lp = ri.data;
        lp_bytes = lpBytes(lp);
    }
    raxStop(&ri);

    uint64_t rax_key[2];
    streamID master_id;
    // 不为空的话检查配置大小
    if (lp != NULL) {
        if (server.stream_node_max_bytes &&
            lp_bytes >= server.stream_node_max_bytes)
        {
            lp = NULL;
        } else if (server.stream_node_max_entries) {
            unsigned char *lp_ele = lpFirst(lp);
            /* Count both live entries and deleted ones. */
            int64_t count = lpGetInteger(lp_ele) + lpGetInteger(lpNext(lp,lp_ele));
            if (count >= server.stream_node_max_entries) {
                // 如果内存有多，但是条目限制，缩内存（listpack是预分配内存）
                lp = lpShrinkToFit(lp);
                if (ri.data != lp)
                    raxInsert(s->rax,ri.key,ri.key_len,lp,NULL);
                lp = NULL;
            }
        }
    }

    int flags = STREAM_ITEM_FLAG_NONE;
    if (lp == NULL) {
        master_id = id;
        // streamID转换成字典树的key
        streamEncodeID(rax_key,&id);
        // 创建master节点，每个rax的val都有一个master节点，即每个listpack包含一个master节点
        // master节点其实就是我们要插入的节点，但是只插入field不插入value，后面再新增消息，将会以master节点为基准
        size_t prealloc = STREAM_LISTPACK_MAX_PRE_ALLOCATE;
        if (server.stream_node_max_bytes > 0 && server.stream_node_max_bytes < prealloc) {
            prealloc = server.stream_node_max_bytes;
        }
        lp = lpNew(prealloc);
        lp = lpAppendInteger(lp,1); /* One item, the one we are adding. */
        lp = lpAppendInteger(lp,0); /* Zero deleted so far. */
        lp = lpAppendInteger(lp,numfields);
        for (int64_t i = 0; i < numfields; i++) {
            sds field = argv[i*2]->ptr;
            // 只存储master节点的field
            lp = lpAppend(lp,(unsigned char*)field,sdslen(field));
        }
        lp = lpAppendInteger(lp,0);
        // 插入字典树
        raxInsert(s->rax,(unsigned char*)&rax_key,sizeof(rax_key),lp,NULL);
        // same标志 意思是可以跟master一样
        flags |= STREAM_ITEM_FLAG_SAMEFIELDS;
    } else {
        serverAssert(ri.key_len == sizeof(rax_key));
        memcpy(rax_key,ri.key,sizeof(rax_key));

        // 如果listpack还能插入数据，获取masterid
        streamDecodeID(rax_key,&master_id);
        unsigned char *lp_ele = lpFirst(lp);
        // listpack相关操作，拿到master节点
        int64_t count = lpGetInteger(lp_ele);
        lp = lpReplaceInteger(lp,&lp_ele,count+1);
        lp_ele = lpNext(lp,lp_ele); /* seek deleted. */
        lp_ele = lpNext(lp,lp_ele); /* seek master entry num fields. */

        int64_t master_fields_count = lpGetInteger(lp_ele);
        lp_ele = lpNext(lp,lp_ele);
        // 新增消息的field数与master一样
        if (numfields == master_fields_count) {
            int64_t i;
            for (i = 0; i < master_fields_count; i++) {
                sds field = argv[i*2]->ptr;
                int64_t e_len;
                unsigned char buf[LP_INTBUF_SIZE];
                unsigned char *e = lpGet(lp_ele,&e_len,buf);
                /* Stop if there is a mismatch. */
                if (sdslen(field) != (size_t)e_len ||
                    memcmp(e,field,e_len) != 0) break;
                lp_ele = lpNext(lp,lp_ele);
            }
            // 所有field都跟master一样 same标志
            if (i == master_fields_count) flags |= STREAM_ITEM_FLAG_SAMEFIELDS;
        }
    }

    /* 新增消息编码如下:
     *
     * +-----+--------+----------+-------+-------+-/-+-------+-------+--------+
     * |flags|entry-id|num-fields|field-1|value-1|...|field-N|value-N|lp-count|
     * +-----+--------+----------+-------+-------+-/-+-------+-------+--------+
     *
     * 如果有same标志，即与master节点field一样:
     *
     * +-----+--------+-------+-/-+-------+--------+
     * |flags|entry-id|value-1|...|value-N|lp-count|
     * +-----+--------+-------+-/-+-------+--------+
     *
     * entryid存的是与master的ms差和seq差
     *
     * lp-count是组成条目的和，可以从后往前遍历 */
    lp = lpAppendInteger(lp,flags);
    lp = lpAppendInteger(lp,id.ms - master_id.ms);
    lp = lpAppendInteger(lp,id.seq - master_id.seq);
    if (!(flags & STREAM_ITEM_FLAG_SAMEFIELDS))
        // 不是same 需要多加一个
        lp = lpAppendInteger(lp,numfields);
    for (int64_t i = 0; i < numfields; i++) {
        sds field = argv[i*2]->ptr, value = argv[i*2+1]->ptr;
        if (!(flags & STREAM_ITEM_FLAG_SAMEFIELDS))
            lp = lpAppend(lp,(unsigned char*)field,sdslen(field));
        lp = lpAppend(lp,(unsigned char*)value,sdslen(value));
    }
    /* Compute and store the lp-count field. */
    int64_t lp_count = numfields;
    lp_count += 3; /* Add the 3 fixed fields flags + ms-diff + seq-diff. */
    if (!(flags & STREAM_ITEM_FLAG_SAMEFIELDS)) {
        lp_count += numfields+1;
    }
    lp = lpAppendInteger(lp,lp_count);

    // 如果listpack地址变了 重新进去
    if (ri.data != lp)
        raxInsert(s->rax,(unsigned char*)&rax_key,sizeof(rax_key),lp,NULL);
    s->length++;
    s->last_id = id;
    if (added_id) *added_id = id;
    return C_OK;
}
```
上面一个add其实将stream的内存结构讲的很清楚了，xread和xrange就是根据streamid转换成的key其实是一串数字来查找val，然后遍历listpack中的数据找到对应的streamid即可。
## 消费组
