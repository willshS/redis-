# 数据结构
## 压缩链表
压缩链表是使用内存非常高效的一种数据结构，以小端序编码，可以存储string和int类型，允许在o(1)时间对头和尾进行push和pop操作，但是每次操作都需要重新分配内存，所以真正复杂度取决于链表使用的内存大小  
以下为redis官方注释的说明和例子

### list
以下是链表的内存结构
```
<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>
```
`zlbytes` 整个链表占用的字节数，类型为uint32  
`zltail` 尾节点的相对偏移，因此可以直接对尾节点进行操作，类型为uint32  
`zllen` 链表中的entry数量，类型为uint16，因此若数量超过2^16-2，此字段值为2^16-1，此时需要遍历整个链表获取数量  
`entry` 节点  
`zlend` 链表结束标志，uint8，恒为255  

#### entry
以下是entry的结构
```
<prevlen> <encoding> <entry-data>
```
`prevlen` 前一个entry的长度，可以从后往前遍历
`encoding` 编码格式，int还是string类型，如果是string类型，还存储了string的长度
`entry-data` 负载数据
如果存储的是一个小型整数，则entry-data可能不存在。

##### prevlen
1. 如果前一个数据长度小于254，则prevlen为1个字节，存储前一个数据的长度
2. 如果前一个数据长度大于253，则prevlen为5个字节，第一个字节为254，剩下4个字节存储前一个数据的长度  

##### encoding
第一个字节的前两位表示此entry存储的是什么类型。  
1. 如果是string类型，第一个字节的前两位表示string的长度的编码，以下为redis官方例子二进制：
> |00pppppp| - 1 byte 表示string长度小于等于63字节即2^6-1，后6位pppppp表示string的长度  
> |01pppppp|qqqqqqqq| - 2 bytes 表示string长度小于等于16383即2^14-1，后6位pppppp+下一个字节8位qqqqqqqq表示string的长度(**14位表示长度的存储方式为大端序**)\
> |10000000|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt| - 5 bytes 第1个字节的后6位不使用，后4个字节表示string长度即大于等于16384，小于等于2^32-1（**32位表示长度的存储方式为大端序**）  

2. 如果是int类型，即使大端序的系统，编码也是小端序，第一个字节的前两位恒为11，表示为int类型，例子：
> |11000000| - 3 bytes int类型编码为 int16_t (2 bytes).  
> |11010000| - 5 bytes int类型编码为 int32_t (4 bytes).  
> |11010000| - 9 bytes int类型编码为 int64_t (8 bytes).  
> |11110000| - 4 bytes int类型编码为 24 bit 有符号 (3 bytes).  
> |11111110| - 2 bytes int类型编码为 8 bit 有符号 (1 byte).  
> |1111xxxx| - 1 byte即小int数，没有entry-data负载，表示0到12，因为11110000和11111110都已经被使用，因此表示的值为0001-1101再减去1即(1到13)-1  
> |11111111| - zlend  

### list例子
| [0f 00 00 00] | [0c 00 00 00] | [02 00] | [00 f3] | [02 f6] | [ff] |
|  ----  | ---- |  ----         |  ----   | ----    |  ----   |
| zlbytes  | zltail| entries         | "2"   | "5"   |  end   |

解释（按字节顺序）：  
0-4字节即0x0f000000为zlbytes，表示整个链表的长度为15个字节  
5-8字节即0x0c000000表示尾节点即"5"相对于整个链表的偏移12个字节  
9-10字节即0x0200表示节点个数，为2个entry  
11-12字节即0x00f3表示第一个entry：  
> 因为是第一个节点，因此prevlen为0，占用一个字节  
> 0xf3二进制为1111 0011，从前两位知道为int类型，在根据1111 xxxx表示为小int数，即后4位表示int大小，0011-0001 = 0010 即 数字2  

13-14字节即0x02f6表示第二个节点：  
> 前一个节点长度为2(prevlen为1+encoding为1，小int数没有额外负载)，因此prevlen占用一个字节即0x02
> 0xf6二进制为1111 0110，从前两位知道为int类型，在根据1111 xxxx表示为小int数，即后4位表示int大小，0110-0001 = 0101 即 数字5  

15字节即0xff 表示链表结束标志zlend  

假如我们向此链表中数字"5"的后面加入字符串"Hello World"：
> 首先加入prevlen，前一个节点长度为2，因此占1个字节 [02]
> 其次我们的字符串小于63字节符合|00pppppp|编码格式，字符串长度为11，因此encoding为0000 1011即0x0b
> 最后加入字符串的ASCII编码[48 65 6c 6c 6f 20 57 6f 72 6c 64] 11个字节

最后需要修改zlbyte整个链表的长度，修改zltail最后一个节点的偏移即可，这样我们就完成了一个节点的插入。

## 总结
ziplist通过对int和string类型进行二进制encoding，来极致压缩内存，保证占用内存量极少，因此是压缩链表，其实ziplist的内存结构在内存中是连续的，更像数组。通过zltail能在o(1)时间找到尾节点，并且通过每个entry中的prevlen可以进行从后到前的遍历。这样既能从前也能从后遍历，更像是双向链表。这种用数组来实现链表并通过二进制的编解码来高效利用内存的结构体就是压缩链表（设计巧妙）