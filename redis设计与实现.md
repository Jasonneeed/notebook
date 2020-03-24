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

#### 1.4 跳跃表

* 跳跃表支持平均时间复杂度O(logN)，最坏时间复杂度为O(n)的节点查找。redis使用跳跃表作为有序集合键的底层实现之一，当一个有序集合包含的元素数量比较多，又或者有序集合中的元素的成员是比较长的字符串，redis会使用跳跃表作为有序集合键的底层实现。redis仅在两处使用了跳跃表结构：①有序集合键；②集群节点中作内部数据结构。
* 跳跃表结构如下:
``` C
  //跳跃表节点结构
  typedef struct skiplistNode{
      //层
      struct zskiplistNodeLevel{
        zskiplistNode * foreward;   //前进指针；
        unsigned int span;  //跨度，用以记录当前两节点差值，可快速查找节点位置。
      } level[];
      
      struct zskiplistNode *backward; //后退指针：指向前一节点，但不指向header
      double score; //分值，各节点以分值大小从小到大排序，若两节点分值相同，则按照成员对象在字典序中的从小到大排序
      robj* obj;//成员对象，指向一个字符串对象，保存一个SDS
  }

  //跳跃表结构
  typedef struct skiplist{
      skiplistNode *header;   //头节点位置，通常使用幂次法则初始化header节点
      skiplistNode *tail;     //尾节点位置
      int length;    //记录跳跃表节点数，不包含header节点
      unsigned long level;     //记录最大节点层数，不包含header节点
    
  }
```

#### 1.5 整数集合

    整数集合是redis集合键的底层实现之一，当一个集合只包含整数类型的值，且集合数量不多，redis采用整数集合作为集合键的实现；

##### 1.5.1 整数集合结构
``` C
    typedef struct intset{
      uint32_t encoding;    //编码方式，规定contents数组的类型
      uint32_t length;      //数组长度
      int contents[];       //保存元素的数组，按照元素大小从小到大排序，不包括重复项
    }
```
##### 1.5.2 整数集合升级
* 需要添加一个新元素到整数集合中，且该元素的类型比整数集合中现有的所有元素都长时，需要进行升级操作后，才能添加新元素到整数集合中；整数集合不支持降级
* 整数集合的升级过程:
  1. 根据新元素类型，扩展底层数组的空间大小，并新元素分配空间
  2. 将底层数组中的现有元素全部按照新元素类型进行转换，将转换后的元素放到正确位置，且需保持整数集合有序特性不变
  3. 将新元素添加进底层数组中(要么最大，要么最小)
* 升级的优点:
  1. 提升整体整数集合的灵活性；
  2. 尽可能的节省内存空间；
  
#### 1.6 压缩列表

  压缩列表时列表见和哈希键的底层实现之一，当一个列表键只包含少量列表项，并且每项要么是小整数，要么是长度较短的字符串。那么redis就会使用压缩列表作为列表键的底层实现。
##### 1.6.1 压缩列表的构成

| 属性 | 类型 | 长度 | 用途 |
| --- | --- | --- | --- |
| zlbytes | uint32_t | 4字节 | 记录整个压缩列表占用的字节数；在对压缩列表进行内存重分配或计算zlend位置时使用 |
| zltail | uint32_t | 4字节 | 记录压缩列表表尾节点距离压缩列表起始地址的字节数；通过整个偏移量，程序无需遍历就可以确定表尾节点位置 |
| zllen | uint32_t | 4字节 | 记录压缩列表包含的节点数量: 当此属性值小于65535时，该属性表示此压缩列表包含的节点数量；若大于整个值，需遍历列表才可得知 |
| entryX | uint16_t | 不定 | 压缩列表包含的各个节点，节点长度由节点保存的内容决定 |
| zlend | uint8_t | 1字节 | 特殊值0xff ,用于标记压缩列表的末端 |
##### 1.6.2 压缩列表节点构成
* 压缩列表是一种为节约内存而开发的顺序型数据结构。

| previous_entry_length | encoding | content |
| - | - | - |
| 以字节位单位，记录压缩列表前一节点的长度；①当前一节点长度小于254字节，使用1字节保存该长度，② 前一节点长度大于等于254字节，此属性长度为5，第一个字节设置为0xFE ,后四个字节用于保存长度 | 记录该节点的content属性所保存的数据类型和长度: 1字节，2字节或5字节，值高位为00，01，10的是字节数组；1字节长，值最高位为11的是整数编码 | 保存节点值，值的类型和长度由encoding决定 |

* 每个压缩节点结构如上，content可以保存一个字节数组或者一个整数值，字节数组可以是一下三种长度中的一种：
    1. 长度<= 63（2^6 -1)字节的字节数组。
    2. 长度<= 2^14 -1 字节的字节数组。
    3. 长度<= 2^32 -1字节的字节数组。
   
  整数值可以是一下几种长度的一种:
    1. 4位长，介于0-12之间的的无符号整数
    2. 1字节长的有符号整数
    3. 3字节长的有符号整数
    4. int16_t，int32_t,int64_t类型的整数。
* 通过previous_entry_length可以获取前一节点的地址，加上压缩列表中尾节点获取的便利，可以很好的反向遍历压缩列表的各个节点。
##### 1.6.3 压缩列表连锁更新
* 场景： 当压缩列表中存在多个连续的，长度介于250至253字节之间的节点，①新增一个长度大于254字节的节点，后续节点需使用5字节的previous_entry_length来保存。②当删除节点后后续节点的previous_entry_length节点字长变化导致连锁更新。
* 每次空间重分配的时间复杂度为O(N),连锁更新的最坏时间复杂度为O(N^2)

#### 1.7 对象
  
  这一部分暂未想好如何总结
