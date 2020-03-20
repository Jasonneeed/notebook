# Redis 设计与实现


## 一、Redis的数据结构
#### 1.1 redis 基本字符串类型SDS（simple dynamic string）
##### 1.1.1 SDS的代码结构
``` C
  typedef struct SDS{
    int len;    //已存入字符长度，作为判断字符结束标志
    int free;   //未使用的字符空间长度
    char buf[]; //底层字符数组
  }
```

* SDS与C语言字符串结构对比

| C语言字符串 | SDS |
| --------- | --------- |
| 以'/0'为字符串结束标志,获取字符串长度的时间复杂度为O(N) | 以自定义的len标注字符串长度，获取字符串长度的时间复杂度为O(1) |
| API 不是二进制安全的，可能造成缓冲区溢出 | API 是二进制安全的，不会造成缓冲区溢出 |
| 修改N次字符串内容时间复杂度为O(N) | 修改N次字符串内容时间复杂度最多为O(N) |
| 仅能保存文本数据 | 可以保存文本或二进制数据 |
| 可以使用全部<String.h>中的函数 | 能使用<String.h>中的部分函数 |

* SDS 内存分配策略
1. 空间预分配：
  ```
  当修改后的buf长度<1MB时，len长度=free长度,buf.len = len+free+1;
  当修改后的buf长度>=1MB时，free长度=1MB,buf.len = len + free +1bit;
  ```
2. 惰性空间释放
  ```
  当缩短SDS保存的字符串时，程序不会立即回收多余的空间，通过free属性将这些字符记录下来，以便后续使用
  ```
  
#### 1.2 redis链表类型
##### 1.2.1 链表的代码结构

``` C
  // 链表节点的代码实现
  typedef struct ListNode{
    ListNode* pre;
    ListNode* next;
    void* value;
  }ListNode;
  
  // 链表
  typedef struct list{
    // 头指针
    ListNode* head;
    // 尾指针
    ListNode* tail;
    // 链表所包含的节点数
    unsigned long len;
    //节点释放值函数；节点复制值函数；节点对比值函数；略
  }
  
```
##### 1.2.2 redis链表特性总结
* 双向链表：链表节点带有pre和next，获取某节点的前置节点或后置节点的时间复杂度为O(1);
* 无环：头结点的pre指向null，尾节点的next指向null，链表访问以null为结束;
* 带有头尾节点指针：list记录有头尾节点，获取头尾节点的时间复杂度为O(1)；
* 链表长度计数： list的len字段标记了当前链表中的节点数量，获取链表长度的时间复杂度为O(1);
* 多态： 使用void* 保存节点值，链表可保存不同类型的值。

#### 1.3 redis字典类型(dict)
##### 1.3.1 redis字典实现
``` C
  //哈希表结构
  typedef struct dictht{
    //哈希表数组
    dictEntry **table;
    //哈希表大小
    unsigned long size;
    //哈希表大小掩码，用于计算索引，值为size-1
    unsigned long sizemask;
    //哈希表已有节点数量
    unsigned long used；
  }
  
  //哈希表节点结构
  typedef struct dictEntry{
    void* key;
    //值
    union{
      void* val;
      uint64_tu64;
      int64_ts64;
    }v;
    //指向下一个哈希表节点，形成链表，用于解决哈希冲突。
    struct dictEntry *next;
  }
  //redis 字典结构
  typedef struct dict{
    //类型特定函数（这个结构我懒得记了)
    dictType* type;
    // 私有数据
    void* privdata
    //哈希表（一般情况下，仅使用ht[0]表，ht[1]表在对ht[0]rehash时使用),元素为链表
    dictht ht[2];
    //rehash 索引；当rehash不进行时，值为-1
    int rehashidx;
  }
```
##### 1.3.2 redis哈希算法(常用MurmurHash2,DJB)
1. redis计算哈希值：hash=dict.Type.hashFunction(key);
2. redis通过哈希值和hash表的sizemask计算索引值。
3. 通过计算的索引值找到ht[0]中的索引位置。若该索引位置不为空，则在链表头加入dictEntry节点。
##### 1.3.3 字典中的rehash
* rehash的执行依据：
```
  hash表的负载因子定义：load_factor = ht[0].used/ht[1].size;
  1.当redis未进行bgsave或bgrewriteaof操作时，负载因子大于等于1，则需要进行扩展的rehash操作。
  2.当redis正在进行bgsave或bgrewriteaof命令时，负载因子大于5，则需要进行扩展的rehash操作。
  3.当负载因子小于0.1时，则需要进行收缩的rehash操作。
```
* rehash的收缩/扩容操作：
```
  1.为字典中的ht[1]分配内存空间：
    1)扩容操作：ht[1]分配的内存空间 = 第一个大于或等于ht[0].used*2的2^n;
    2)收缩操作：ht[1]分配的内存空间 = 第一个大于或等于ht[0].used的2^n;
  2.将保存到ht[0]中的所有键值对保存到ht[1]中(rehash指重新计算hash值和索引值，然后重新分配元素至ht[1]中)
  3.释放ht[0]，并将ht[1]设置为ht[0]，将ht[1]设置为空。rehashidx设为-1标志此次rehash结束。
```
* 渐进式rehash
```
  redis中rehash过程不是一次性，集中式的完成，而是分多次，渐进式的完成。当ht[0]存在多个键值对时，一次性完成rehash操作可能导致服务器停止服务。
  
  渐进式rehash过程：
  1. 为ht[1]分配地址空间，字典同时持有ht[0]和ht[1]；
  2.将字典中的rehashidx设置为0，标志rehash开始。
  3.rehash过程中，对字典执行增删查改操作。新增键值对直接将键值对写入ht[1]表中，查找操作先查找ht[0]，再查找ht[1]；每有一个键值对从ht[0]rehash到ht[1]上，rehashidx+1;
  4.当ht[0]表上元素全部置入ht[1]中，将rehashidx设置为-1，标志此次rehash操作结束。
```
