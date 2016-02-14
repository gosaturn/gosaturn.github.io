---
layout: post
author: GoSaturn
title: redis源码学习——字符串&双端链表
category: 源码学习
tag: [redis]
---
# 字符串（SDS）

> 简单动态字符串 simple dynamic string ,SDS

## sds结构体

```c
struct sds{
	int len; //记录buf数组中已使用字节的数目，即sds所保存字符串的长度
	int free; //记录buf数组中未使用字节数
	char buf[];//长度为0，不占用内存空间
};//sizeof(struct sds) = 8B,即 int len + int free的长度
```

>[c零长数组](http://www.cnblogs.com/Anker/p/3744127.html)
>[c内存分配方式](http://blog.csdn.net/shuaishuai80/article/details/6140979)

## sds源码

```c
源文件——sds.h
typedef char *sds;//定义一种数据类型，名叫sds，它是一种指向char类型的指针
```
```c
//sds.h文件
//返回sds字符串长度
static inline size_t sdslen(const sds s) {
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    return sh->len;
}
```
解释说明：

 - sizeof(struct sdshdr)) 值为8B，即len + free的长度；
 - sds是`指向char类型的指针`类型，因此s是指向sds结构中buf数组的指针
 - s-(sizeof(struct sdshdr)))就是说s指针向上移动8B，即指向了sds结构体的首地址
 - 因此sh->len就能得到sds结构体中len的值

这里用到了零长数组的特性。
## sds特点
### 1. 空间预分配
当增加sds字符串长度时：
 - 如果扩展后，sds字符串的长度<1M（即len值），则分配相同长度的free空间（eg. len长度变为50B，则分配同样长度50B的空间给free字段，即50B+50B+1B，1B用来保存字符串结束符'\0'）
 - 如果扩展后，sds字符串长度>=1M，则分配1M的空间给free字段
### 2. 惰性空间释放
当缩短sds字符串长度时，释放的空间不会立即由内存收回，而是放到free属性中，以备将来使用（如果需要真正释放，sds也提供了相应的api）。
### 3. 二进制安全
当字符串中有空字符时，如果用c字符串来保存，空字符会被当做字符串的结束符，导致空字符后面的字符被截断。
sds字符串，通过len属性保存字符串长度，不会根据空字符截取，读取字符串时，会根据len长度读取相应长度的字符串，因此是二进制安全的。
### 4. 兼容部分c字符串函数
由于sds保留了c字符串的结束符'\0'，因此可以重用一部分string.h库定义的c字符串函数。
### 5. 常数复杂度获取字符串长度
sds记录了字符长度len，因此获取字符串长度的复杂度为O(1)
### 6. 杜绝缓冲区溢出
sds api在对字符串进行修改时，会先检查sds的空间是否满足需求，如果不满足，会自动分配相应的空间，因此不会出现缓冲区溢出的问题。

# 链表
## 链表数据结构
```c
//adlist.h文件
//双端链表节点
typedef struct listNode{
	struct listNode *prev;
	struct listNode *next;
	void *value;//链表节点可以用来保存各种类型的值
}listNode;
```
```c
//链表迭代器
typedef struct listIter{
    listNode *next;
    int direction;
}listIter;
```
```c
typedef struct list{
	listNode *header;
	listNode *tail;	 
    void *(*dup)(void *ptr);  // 节点值复制函数  
    void (*free)(void *ptr);  // 节点值释放函数   
    int (*match)(void *ptr, void *key);// 节点值对比函数
	unsigned long len;	
}list;
//根据listNode中value的不同类型，实现相应的dum,free,match，从而实现“多态”
```
>当列表的元素较多或每个元素对应的字符串长度较长时，使用list进行存储，否则使用ziplist

