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
  
  五种对象类型:字符串对象，列表对象，哈希对象，集合对象和有序集合对象。每个对象由一个RedisObject结构表示：
  ``` C
  typedef struct redisObject{
    //类型
    unsigned type:4;
    //编码
    unsigned encoding：4；
    //指向底层实现数据结构的指针
    void *ptr；
  }
  ```
  不同类型和编码的对象：
  
  |类型|编码|对象|
  |-|-|-|
  |REDIS_STRING|REDIS_ENCODING_INT|使用整数值实现的字符串对象|
  |REDIS_STRING|REDIS_ENCODING_EMBSTR|使用embstr编码的简单动态字符串实现的字符串对象|
  |REDIS_STRING|REDIS_ENCODING_RAW|使用简单动态字符串实现的字符串对象|
  |REDIS_LIST|REDIS_ENCODING_ZIPLIST|使用压缩列表实现的列表对象|
  |REDIS_LIST|REDIS_ENCODING_LINKEDLIST|使用链表实现的列表对象|
  |REDIS_HASH|REDIS_ENCODING_ZIPLIST|使用压缩列表实现的哈希对象|
  |REDIS_HASH|REDIS_ENCODING_HT|使用字典实现的哈希对象|
  |REDIS_SET|REDIS_ENCODING_INTSET|使用整数集合实现的集合对象|
  |REDIS_SET|REDIS_ENCODING_HT|使用字典实现的集合对象|
  |REDIS_ZSET|REDIS_ENCODING_ZIPLIST|使用压缩列表实现的有序集合对象|
  |REDIS_ZSET|REDIS_ENCODING_SKIPLIST|使用跳跃表实现的有序集合对象|
  
##### 1.7.1 字符串对象
* 如果一个字符串对象保存的是整数值且可以使用long类型表示，字符串编码类型为int；
* 如果字符串对象保存的是一个字符串值，且字符串长度大于32字节，那么将用一个SDS来保存该字符串值，编码设置为raw;
* 字符串对象保存的是一个字符串值，且字符长度小于等于32字节，那么编码设置为embstr(实际上该编码对象为只读)；
* 使用embstr编码保存短字符串的优势：
  1. embstr编码将创建字符串对象所需的内存分配次数从raw编码的两次降为一次；
  2. 释放embstr编码的字符串对象只需调用一次内存释放函数，而释放raw编码需要调用两次内存释放函数。
  3. embstr编码的字符串对象所有数据都保存在一块连续的内存里面。
##### 1.7.2 列表对象

同时满足这两个条件列表对象使用ziplist编码，否则使用linkedlist编码:

* 列表对象的所有字符串元素的长度都小于64字节
* 列表对象保存的元素数量小于512个。
##### 1.7.3 哈希对象

同时满足一下两个条件时使用ziplist编码，否则使用hashtable编码：
* 哈希对象保存的所有键值对的键和值的字符串长度都小于64字节；
* 哈希对象保存的键值对数量小于512个。
##### 1.7.4 集合对象

同时满足以下两个条件时使用intset编码，否则使用hashtable编码：
* 集合对象保存的所有元素都是整数值；
* 集合对象保存的元素数量不超过512个；
##### 1.7.5 有序集合对象

同时满足以下两个条件时使用ziplist编码，否则使用skiplist编码：
* 有序集合保存的元素数量小于128个；
* 有序集合保存的所有元素成员的长度都小于64字节；
##### 1.7.6 内存回收

redis构建了一个引用计数技术实现的内存回收机制。
## 二、单机Redis
#### 2.1 redis数据库概述
##### 2.1.1 服务器中的数据库结构

* redis中不保存当前使用的数据库编号，命令窗可以看到当前数据库编号，默认为0号数据库(建议多执行select database命令)。

``` C
  //redis数据库结构
  typedef struct redisDB{
    dict *dict;   //数据库键空间，保存着数据库所有的键值对
    dict *expires;  //数据库过期键空间，保存着数据库所有的设定过期时间的键值对，值为Unix时间戳
  }
  
  typedef struct redisServer{
    ...
    int dbnum;      //服务器数据库数量，默认为16
    redisDB *redisdb;   //数据库数组，保存着服务器中的所有数据库
    struct saveparam *saveparams;   //记录保存条件的数组(内含秒数和修改数两个参数)
    long long dirty;    //修改计数器(记录着自上次save或bgsave命令后，服务器对数据库进行的状态修改次数。
    time_t lastsave；   //上一次保存的时间
    ...
  }
  
  typedef struct redisClient{
    ...
    redisDB *redisdb;   //redisdb对象，存储当前客户端所使用的数据库
    ...
  }
```
##### 2.1.2 数据库键空间
* 键空间中的键保存着数据库的键，每个键都是一个字符串对象。
* 键空间中的值保存着数据库的值，每个值可以使字符串对象，链表对象，哈希对象，集合对象，有序集合对象。
* 添加，更新，删除，取值等操作。
* 键的生存时间或过期时间:
  1. expire(秒)命令和pexpire(毫秒)命令，设置键的生存时间，经过指定时间后，redis会删除生存时间为0的键。
  2. expreat(秒)命令和pexpireat(毫秒)命令，设置键的过期时间(时间戳)，过期时间到来后，服务器自动删除该键。
  3. expire,pexpire, expireat 都是通过pexpireat命令实现的。
 ##### 2.1.3 键空间中键的过期
 * 键的过期判定:
    1. 检查给定键是否存在于过期键；存在取得过期键的过期时间；
    2. 检查当前Unix时间戳是否大于过期时间；
 * 过期键的删除策略
 

| 策略 | 概述 | 优点 | 缺点 |
| - | - | - | - |
|定时删除 | 通过使用定时器，定时删除策略可以尽快删除过期键，释放内存 | 对内存友好 | 对CPU时间不友好 |
|惰性删除 | 程序只会在取出键的时候进行过期检查 | 对CPU时间友好 | 对内存不友好|
|定期删除 | 间隔指定时间后，程序执行一次删除过期操作，通过限制删除操作的时长和频率减少删除操作对CPU时间的影响，需合理设置操作的执行时长和频率| 

##### 2.1.4 dis的过期删除策略

    reids服务器使用惰性删除和定期删除两种策略相互配合使用。合理使用CPU时间和避免浪费内存空间。
* redis的惰性删除策略: 在所有读写reids数据库命令执行之前调用expireIfNeeded 函数，进行过期检查；若键已过期，删除键，避免命令与过期键进行接触。
* redis的定期删除策略
```
  1. 默认检查16个数据库，每个数据库默认删除20个过期键，全局变量current_db记录当前检查进度
  2. 每个数据库中随机获取20个键进行过期判断，期间若是操作时间过长，也会停止处理。
```
##### 2.1.5 AOF，RDB和复制功能对工期间的处理
* RBD对过期键的处理:
  1. 创建新的RDB文件时，会忽略已过期的键
  2. 载入RDB文件时，主服务器会忽略过期键，从服务器不论键是否过期都会载入RDB文件。不过主数据库进行数据同步的时候，从数据库数据会被清空。
* AOF对过期键的处理:
  
  1. 服务器以AOF持久化模式运行时，只要过期键未被删除，服务器都会将其写入到文件中，当过期键被惰性删除或定期删除后，AOF追加一条del 命令，显示表明该键已被删除。
  2. AOF重写时，已过期的键不会被保存到重写的AOF文件中。
* 复制模式:
  1. 主服务器删除一个过期键后，会显示的向所有从服务器发送一个del命令，告知所有从服务器删除该键。
  2. 从服务器在执行客户端读命令时，即使遇到过期键也不会删除，当做未过期键正常处理。从服务器需接受到主服务器发送的del命令才删除过期键。(主从一致性)
##### 2.1.6 数据库通知
* 键空间通知:某个键执行了什么命令
* 键事件通知:某个命令被什么键执行了

#### 2.2 RDB持久化

  RDB持久化功能所生成的RDB文件是一个保存在硬盘，经过压缩的二进制文件，记录着redis数据库的状态。
##### 2.2.1 RDB文件的创建与载入
* save命令和bgsave命令可用于生成RDB文件。save会阻塞服务器进程，bgsave开启子进程来进行save命令。
* RDB文件的载入在服务器启动时执行，没有用于载入RDB文件的命令；当服务器载入RDB文件时，会一直处于阻塞状态，直到载入完成为止。
* AOF的更新频率会比RDB的更新频率高，若服务器开启了AOF功能，服务器会优先使用AOF还原数据库状态；只有AOF功能关闭才会使用RDB功能还原数据库状态。
* 当服务器执行save命令期间，redis服务器会被阻塞，所有客户端发送的所有命令都会被拒绝。当服务器执行bgsave时，save命令和bgsave命令都会被服务器拒绝。
* bgsave和bgrewriteaof不能同时执行，执行bgsave时，bgrewriteaof命令需等bgsave命令完成后执行；执行bgrewriteaof时，bgsave会被拒绝。

##### 2.2.2 RDB文件结构

| REDIS | db_version | databases | EOF | check_sum|
| - | - | - | - | - |
* REDIS表示RDB文件开头
* db_version：长度为4字节，值是一个字符串表示的整数，记录RDB文件版本。
* databases: 数据库状态为空时，这部分为空。
* EOF：1字节长，标志着RDB文件内容结束。
* check_sum：8字节长的无符号整数，通过前四个部分内容计算得出，用于检验RDB文件是否出错或损坏。

databases的结构:
  
  | SELECTDB | db_number | key_value_pairs|
  | - | - | - |

key_value_pairs的结构:
  
| TYPE | key | value|
| - | - | - |
  
带过期时间的键值对结构:
 
 | EXPIRETIME_MS | ms | TYPE |key | value|
| - | - | - | - | - |

###### value的编码
1. 字符串对象:
    长度不超过32位的整数，使用INT编码结构

    | ENCODING| integer |
    | - | - |
    
    长度<=20字节，字符直接原样保存；长度>20字节，该字符串会被压缩后再保存(若服务器关闭了RDB文件压缩功能，则全部以未压缩形式保存)
    
    无压缩字符串结构
    
    | len | string |
    | - | - |
    
    压缩字符串结构
    
    | REDIS_RDB_ENC_LZF | compressed_len | origin_len | compressed_string |
    |-|-|-|-| 
2. 列表对象

| list_length | item1 |item2 | ... | itemN|
|-|-|-|-|-|

3. 集合对象

|set_size | elem1 |elem2 |...| elemN|
|-|-|-|-|-|
4. 哈希对象

|hash_size | key1|value1|key2|value2|
|-|-|-|-|-|
5. 有序集合对象

|sorted_set_size | item1|score1|item2|score2|
|-|-|-|-|-|
6. 整数集合

将整数集合转化为字符串对象，再保存字符串对象。

7. 压缩表编码的列表，有序集合或哈希表

将压缩列表转换为字符串，以字符串形式保存

#### 2.3 AOF持久化
