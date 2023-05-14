# Redis 之 ziplist
ziplist 是特殊的双向链表,优点为内存占用少。可保存整数和字符串。在链表的两侧进行增删操作的复杂度为O(1)。

## Redis 的五种类型
Redis 类型hash 、list 、ziplist 底层的数据结构都有ziplist。本文的研究主题为ziplist。

- string
    - sds
- hash
    - ziplist
    - hashtable
- list
    - linkedlist
    - quicklist
    - ziplist
- set 
    - hashtable
    - intset
- zset
    - skiplist
    - ziplist

## ziplist 编码规则
下图使用`plantuml` 语法绘制，编辑器为neovim，markdown插件为`iamcco/markdown-preview.nvim` 。

```plantuml

:<zlbytes><zltail><zllen><entry><entry><zlend>;
note right
    ziplist的整体布局
endnote

split
    :zlbytes;
    note left
       ziplist 占用总字节数
       占用4字节
       避免重新分配时遍历内存
    endnote
split again
    :ztail;
    note left
        最后节点的偏移
        占用4字节
    endnote
split again
    :zllen;
    note left
        节点的长度
        占用2字节
        当长度小于2**16-1，为长度
        当长度等于2**16-1，需遍历
    endnote
split again
    :entry;
    note left
        节点，包含头信息和保存体
        头信息包含前一节点占用字节长度和编码
    endnote
    split
        :一： header 头信息;
        split
            :1.1, 保存前一节点占用字节数;
            if (第一节点数值小于254) then (小于)
                :1字节;
            else (大于)
                :5字节，接下来4字节为长度;
            endif
            :1.2, 编码;
            split
                :字符串;
                split
                    :00pppppp;
                    :body小于等于63字节;
                split again
                    :01pppppp,qqqqqqqq;
                    :body小于等于16383字节;
                split again
                    :10______,qqqqqqq,rrrrrrrr,ssssssss,tttttttt;
                    :body大于等于16384字节;
                end split
            split again
                :整数;
                split
                    :11000000;
                    :body 占用 2 字节;
                split again
                    :11010000;
                    :body 占用 4 字节;
                split again
                    :11100000;
                    :body 占用 8 字节;
                split again
                    :11110000;
                    :body 占用 3 字节;
                split again
                    :11111110;
                    :body 占用 1 字节;
                split again
                    :1111XXXX;
                    :body 占用 0 字节;
                end split
            end split
        end split
        :二, body;
        note right
            保存的整数
            或者字符串
        end note
    end split
split again
    :zlend;
    note left
        结束标记
        占用1字节
        内容为：11111111
    endnote
end split

 ``` 

## zlentry

```c
typedef struct zlentry {
    unsigned int prevrawlensize, prevrawlen;
    unsigned int lensize, len;
    unsigned int headersize;
    unsigned char encoding;
    unsigned char *p;
} zlentry;

```
- prevrawlensize
    前一节点的属性长度
- prevrawlen
    前一节点占用字节长度
- lensize
    当前节点的属性长度, 通过encoding 解析
- len
    当前保存数据占用的字节长度，通过encoding 解析
- headersize
    prevrawlensize + lensize
- encoding
    编码
- p
    char数组

zlentry中的`len`和`lensize`在编码规则中未描述。上面两个属性是通过encoding编码解析出，非核心流程。编码规则见上图。


```c
#define ZIP_DECODE_LENGTH(ptr, encoding, lensize, len) do {                    \
    ZIP_ENTRY_ENCODING((ptr), (encoding));                                     \
    if ((encoding) < ZIP_STR_MASK) {                                           \
        if ((encoding) == ZIP_STR_06B) {                                       \
            (lensize) = 1;                                                     \
            (len) = (ptr)[0] & 0x3f;                                           \
        } else if ((encoding) == ZIP_STR_14B) {                                \
            (lensize) = 2;                                                     \
            (len) = (((ptr)[0] & 0x3f) << 8) | (ptr)[1];                       \
        } else if (encoding == ZIP_STR_32B) {                                  \
            (lensize) = 5;                                                     \
            (len) = ((ptr)[1] << 24) |                                         \
                    ((ptr)[2] << 16) |                                         \
                    ((ptr)[3] <<  8) |                                         \
                    ((ptr)[4]);                                                \
        } else {                                                               \
            assert(NULL);                                                      \
        }                                                                      \
    } else {                                                                   \
        (lensize) = 1;                                                         \
        (len) = zipIntSize(encoding);                                          \
    }                                                                          \
} while(0);

```

## 总结
本文主要通过流程图的表达方式，从全局角度解释了ziplist的编码。可以看到ziplist就是数组的升级版本,保存的数据长度不固定。
