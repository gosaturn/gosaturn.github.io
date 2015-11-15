---
layout: post
author: GoSaturn
title: redis源码学习——压缩列表ziplist
category: 技术学习
tag: [redis]
---
# 压缩列表ziplist
##ziplist简介

 - 压缩列表是由一系列 `特殊编码的` `连续内存块` 组成的 `顺序型` 存储结构。主要是为了`节省内存空间`而设计的数据结构。
 - 压缩列表可以用来存储字符串或整数。
 - 压缩列表存储的数据特点： 数据量比较小，并且每项数据要么是比较小的整数，要么是比较短的字符串。
 - redis中的 列表list，哈希表hashtable，有序集合sorted set 的底层实现中都用到了ziplist。
 - 对于用户来说，其实不用关心什么时候用ziplist，redis底层会根据数据量多少以及每项数据大小，判断是否采用ziplist结构存储数据（当数据量超过限定值时，会传化成其他的数据结构）。


{% highlight C %}
//redis.conf文件
list-max-ziplist-entries 512 //ziplist中entry个数
list-max-ziplist-value 64 //每个entry所占的字节大小
{% endhighlight %}


##ziplist结构
```
<zlbytes>|<zltail>|<zllen>|<entry>...<entry>|<zlend>
4B       |   4B   |    2B |                 |  1B
|<------header----------->|  
```

zlbytes： 存储整个压缩列表所占字节数，4B(unsigned int)

zltail: 存储压缩列表尾节点相对于首地址的偏移量， 4B(unsigned int)

zllen：存储压缩列表中节点数目，2B(unsigned int)，最长2^16-2，当节点数目大于此值，需要遍历所有节点才能得到节点数

entry: 压缩列表节点，节点大小根据存储的内容确定

zlend： 压缩列表结尾符，值为255，1B(unsigned int)

##entry结构

```
typedef struct zlentry{
	unsigned int   prevrawlensize, prevrawlen;
	//prevrawlen前一个节点长度， prevrawlensize 前一个节点编码所占字节数
	unsigned int   lensize, len;
	//len 当前节点长度，lensize 当前节点编码所占字节数
	unsigned int   headersize;
	//
	unsigned char  encoding;
	unsigned char  *p;
}
```


```
|前一个节点的编码&长度    |当前节点编码&长度     |当前节点内容|
<previous_entry_length>|<encoding & length>|<content> |
```
`previous_entry_length`有两种长度：

 - 如果前一个字节长度小于2^8-2=254 B，则previous_entry_length长度为1B
 - 如果前一个字节长度大于254B，则previous_entry_length长度为5B，第一个字节为254(1111 1110)，后面4个字节保存实际长度值

`encoding&length` ziplist有两种类型的编码：字符串&整数。
>通过encoding的前两个比特位判断content保存的是字符串还是整数，00，01，10表示字符串，11表示整数

**字符串：**

 -  |00aaaaaa|  `1字节`，长度<= 2^6-1=63B
 -  |01aaaaaa|bbbbbbbb|  `2字节`，长度<=2^14-1=16383B
 -  |10--------|aaaaaaaa|bbbbbbbb|cccccccc|dddddddd|  `5字节`，长度>=16383B，用后面4个字节存储字符串长度

**整数：**
 
 - 1100 0000 —— >1字节 ——> int16_t类型整数
 - 1101 0000 ——> 1字节 ——> int32_t类型整数
 - 1110 0000 —— >1字节 ——> int64_t类型整数
 - 1111 0000 ——> 1字节 ——> 24bit有符号整数
 - 1111 1110 —— >1字节 ——> 8bit 有符号整数
 - 1111 xxxx ——> 1字节 ——> 4bit无符号整数，0~12，不需要content字段


`定义压缩列表节点的encoding值`

```
/* Different encoding/length possibilities */定义不同的encoding
//str
#define ZIP_STR_06B (0 << 6)
#define ZIP_STR_14B (1 << 6)
#define ZIP_STR_32B (2 << 6)
//int
#define ZIP_INT_16B (0xc0 | 0<<4) // '|'按位或，1&0=1，0&0=0
#define ZIP_INT_32B (0xc0 | 1<<4)
#define ZIP_INT_64B (0xc0 | 2<<4)
```

`计算数据类型`

```
/* Macro's to determine type */计算encoding对应的数据类型，str 或 int
//将上面的宏带入enc，得到true或false。
#define ZIP_IS_STR(enc) (((enc) & 0xc0) < 0xc0) //'&'按位与，1&1=1，1&0=0
#define ZIP_IS_INT(enc) (!ZIP_IS_STR(enc) && ((enc) & 0x30) < 0x30)

```

##ziplist应用
###创建空的ziplist
```
//创建并返回一个新的ziplist，其中entry为空
unsigned char *ziplistNew(void) {
    // ZIPLIST_HEADER_SIZE 是 ziplist 表头的大小,固定长度4B+4B+2B
    // 1 字节是表末端 ZIP_END 的大小
    unsigned int bytes = ZIPLIST_HEADER_SIZE+1;
    // 为表头和表末端分配空间
    unsigned char *zl = zmalloc(bytes);
    // 初始化表属性
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);//整个ziplist所占字节数,intrev32ifbe大小端转换
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);//ziplist尾节点相对首节点的偏移量
    ZIPLIST_LENGTH(zl) = 0;//entry为空
    // 设置表末端
    zl[bytes-1] = ZIP_END;
    return zl;
}
```

###给ziplist插入entry节点
```
//将长度为slen的字符串s，插入zl中，返回插入了entry元素的ziplist
unsigned char *ziplistPush(unsigned char *zl, unsigned char *s, unsigned int slen, int where) {
    // 根据 where 参数的值，决定将值推入到表头还是表尾
    unsigned char *p;
    p = (where == ZIPLIST_HEAD) ? ZIPLIST_ENTRY_HEAD(zl) : ZIPLIST_ENTRY_END(zl);
    // 返回添加新值后的 ziplist
    // T = O(N^2)
    return __ziplistInsert(zl,p,s,slen);
}
```
其中，插入节点可能会涉及到`连锁更新`
##初始化entry
```
//将p指向的压缩列表节点所对应的属性信息保存到zlentry结构体中，并返回zlentry结构体。(相当于给zlentry结构体的各个字段赋值)
static zlentry zipEntry(unsigned char *p) {
    zlentry e;
    // e.prevrawlensize 保存着编码前一个节点的长度所需的字节数
    // e.prevrawlen 保存着前一个节点的长度
    // T = O(1)
    ZIP_DECODE_PREVLEN(p, e.prevrawlensize, e.prevrawlen);
    // p + e.prevrawlensize 将指针移动到列表节点本身
    // e.encoding 保存着节点值的编码类型
    // e.lensize 保存着编码节点值长度所需的字节数
    // e.len 保存着节点值的长度
    // T = O(1)
    ZIP_DECODE_LENGTH(p + e.prevrawlensize, e.encoding, e.lensize, e.len);
    // 计算头结点的字节数
    e.headersize = e.prevrawlensize + e.lensize;
    // 记录指针
    e.p = p; 
    return e;
}
```
