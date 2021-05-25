# 数据结构
## 压缩哈希
与ziplist非常相似，既然我们可以用连续内存来实现一个高效使用内存的双向链表，也可以实现一个散列表。当我们需要一个存储少量节点的散列表的时候，就可以使用zipmip，它相比于dict使用非常少的内存。但是其查找时间复杂度为o(n)，因此只使用少量节点的时候使用zipmap

以下是map[foo]=bar,map[hello]=world的一个散列表的内存结构：
```
<zmlen><len>"foo"<len><free>"bar"<len>"hello"<len><free>"world"<zmend>
```
`zmlen` 占1个字节，表示map中节点个数，最大为253，当为254的时候就需要遍历zipmap获取实际长度  
`len` 表示后面字符串的长度，同ziplist一样，如果为0到253大小的长度则占用1个字节，否则占用5个字节，第1个字节为254，则后4个字节为长度  
`free` 8bit空闲空间，比如如果设置foo的值为barr则可以直接使用，如果设置foo为b，则free还是8bit，因为如果超过1字节会重新调整内存来使zipmap占用内存尽量小  
`zmend` 恒为255 同ziplist，占1个字节  
因此上面的zipmap内存中最紧凑的表示为`\x02\x03foo\x03\x00bar\x05hello\x05\x00world\xff`

## 总结
zipmap是压缩哈希表，但是只适合用来存储少量节点，当达到一定规模的节点的时候，转换为dict。
