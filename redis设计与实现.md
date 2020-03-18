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
