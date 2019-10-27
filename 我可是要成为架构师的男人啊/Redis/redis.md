

## Redis数据结构

可以从两个层面来理解

**使用者**

* string

* list

* hash

* set

* zset

**底层实现**

* dict
* sds
* ziplist
* quicklist
* skiplisst

### RedisObject

使用层和实现层的中介，Redis数据库中，键和值都有对应的redisObject。其结构如下：

```c
typedef struct redisObject {
    // 类型(使用层)
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    // 指向底层数据结构的指针(实现层)
    void *ptr;
    // 引用计数
    int refcount;
    // 最后一次被访问的时间
    unsigned lru:22;
} robj;
```

每种类型(`type`)的对象都至少使用了两种不同的编码

![robj类型和编码映射]( https://ftp.bmp.ovh/imgs/2019/10/99dc620722b32b7c.png )

#### 字符串对象

* int

  字符串是整数，ptr类型为long。

  long和double在Redis中也是作为字符串值来保存的。

* raw

  长度大于32字节的字符串时，ptr类型为SDS，encoding为raw

* embstr

  适用于短字符。效果与raw完全一样，但是，**embstr编码只调用一次内存分配函数来获取一块连续的空间，依次包含redisObject和ssdshdr两个结构**

  ![embstr编码结构]( https://i.bmp.ovh/imgs/2019/10/50538f55bc1c9bad.png )

对于int类型，修改其值为字符串后，编码会变成raw；

embstr是只读的，对应的任何修改，编码都会变成raw。

#### 列表对象

当列表满足下面两个条件，使用ziplist编码：

- 列表对象保存的所有字符串长度小于`64字节`
- 列表元素小于`512`个

#### 哈希对象

* ziplist

  键值对都小于`64字节`;

  元素个数小于`512`个。

  ![ziplist编码的哈希对象]( https://i.bmp.ovh/imgs/2019/10/b74ac5f220ccc41e.png )

* hashtable

  同Java中的HashMap

#### 集合对象

* intset

  所有值都是整数；

  元素个数小于512个

* hashtable

  值为null

#### 有序集合

* ziplist

  元素小于64字节；

  元素个数小于128个。

  ![压缩列表的实现]( https://ftp.bmp.ovh/imgs/2019/10/a2bcecc9afb5c52a.jpg )

* skiplist

  skiplist编码的有序集合对象使用zset结构作为底层实现

  ```c
  typedef struct zset {
      // 跳表
  	zskiplist *zsl;
      // key->score映射
      dict *dict;
  }
  ```

  zsl用于实现排序、范围查找等有序集合通用功能；

  dict用于加快 value->score 查找。

#### 多态

![多态]( https://ftp.bmp.ovh/imgs/2019/10/3d6f8c7a37b5697a.jpg )

#### 内存回收

* 新对象创建时，初始值为1
* 对象呗其它程序使用，引用计数+1
* 不在引用，-1
* 为0时，内存回收

#### 对象共享



---

### SDS

字符串是程序语言最常用的数据结构，Redis对外暴露了一个字符串结构，叫做string，string的底层实现就是sds,全称 Simple Dynamic String。

sds.h/sdshdr

```c++
struct sdshdr {
    // 已使用的字节数量
    int len;
    // 未使用的字节数量
    int free;
    // 字节数组
    char buf[];
};
```

![SDS结构图]( https://ftp.bmp.ovh/imgs/2019/10/70e70d9015a6f394.jpg )

以空字符'\0'结尾，占一字节，不计入sds的len属性。从而可以直接重用C函数库里的字符操作。  

---

#### SDS与C字符串的区别

* 常数复杂度获取字符串长度(len)

* 杜绝缓冲区溢出

  > 当修改SPS时，API会先检测SDS空间是否满足修改所需的要求，如果不满足，API会自动扩容，然后再执行。

* 减少修改字符串时带来的内存重分配

  >1. 空间预分配
  >
  >   * 对SDS进行修改后，SDS的长度小于1MB，free=len
  >   * 。。。大于1MB，free=1MB
  >
  >2. 惰性空间释放
  >
  >   字符串缩短后，free变大，不会重新分配内存。

* 二进制安全，对空字符'\0'的解析

* 兼容部分C字符函数

---

### 链表

存放数据的集合，C语言中并不存在。还应用在`发布与订阅`、`慢查询`、`监视器`等功能，Redis服务器本身还使用链表来`保存多个客户端的连接信息`，以及使用链表来`构建客户端输出缓冲区`

adlist.h/listNode

```c
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点值
    void *value;
}listNode;
```

多个listNode通过prev和next组成双端链表：adlist.h/list

```c
typedef struct list {
    // 表头节点
	listNode * head;
    // 表尾节点
    listNode * tail;
    // 链表所包含的节点数量
    unsigned long len;
    
    // ...
    // 节点复制函数
    void *(*dup)(void *ptr);
    // 节点释放函数
    void (*free)(void *ptr);
    // 节点比对函数
    int (*match)(void *ptr,void *key);
} list;
```

![链表结构图]( https://i.bmp.ovh/imgs/2019/10/b0689196b957d2c4.png )

特性

* 双端
* 无环
* 带表头和表尾指针
* 带链表长度计数器
* 多态：动态实现dup、free、match三个函数

---

### 字典

Redis的字典使用哈希表作为底层实现

dict.h/dictht

```c
typedef struct dictht {
    // 哈希表数组
    dictEntry **table;
    // 哈希表数组大小
    unsigned long size;
    // size-1;用于取模计算索引值
    unsigned long sizemask;
    // 已有节点的数量
    unsigned long used;
}
```

![空哈希表]( https://i.bmp.ovh/imgs/2019/10/929bf01db698ea58.png )

哈希表节点

```c
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union{
        void *val;
        uint64_tu64;
        int64_tu64;
    } v;
    // 指向下个节点
    struct dictEntry *next;
}
```

Redis中的数据库结构——字典：dict.h/dict

```c
typedef struct dict {
    // 类型
    dictType *type;
    // 私有数据
    void *privdata;
    // 哈希表
    dictht ht[2];
    // rehash索引，-1表示没有进行
    int rehashidx;
} dict;
```

type属性和privdata属性针对不同类型的键值对，用于创建多态字典。

* type是一个指向dictType结构的指针

```c
typedef struct dictType {
    // 哈希函数
    unsigned int (*hashFunction)(consst void *key);
    // 复制key函数
    void *(*keyDup)(void *privdata, const void *key);
    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);
    // 对比键的函数
    int (*keyCompare)(void * privdata, const void *key1, const void *key2);
    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```

* privdata属性保存需要传给特定函数的可选参数
* ht属性包含两个数组

![普通状态下的字典]( https://i.bmp.ovh/imgs/2019/10/495a5bf042d7ba65.png )

#### 哈希算法

Redis计算hash的算法和Java 中的HashMap类似

1. 使用字典设置的哈希函数，计算键key的哈希值

   hash = dict->type->hashFunction(key);

2. 取模（按位与）计算出索引值，x可能是0或1

   index = hash & dict->ht[x].sizemask

3. 放在ht[x]索引链表上的第一位

#### rehash

当哈希表保存的键值对数量太多或者太少时，程序需要对哈希表进行扩容或缩容。具体步骤如下：

1. 为字典的ht[1]哈希表分配空间，扩大或缩小`used`两倍
2. ht[0]中的所有键值对，按索引位置，rehash到h[1]上
3. ht[0]遍历完后，将ht[1]设置为ht[0]

#### 渐进式rehash

数组的迁移并不是一次性、集中式地完成的，而是分多次、渐进式地完成的，将rehash键值对所需的计算工作均摊到对字典的每个操作上，从而避免了集中式rehash而带来的庞大计算量。以下是rehash的详细步骤：

1. 为h[1]分配空间，字典同时持有ht[0]和ht[1]
2. rehashidx设置为0，表示是rehash工作正式开始
3. rehash期间，执行过增删改查的键所在的索引，会迁移到ht[1]
4. 遍历结束后，rehashidx重置为-1

**rehash期间查找键时，会现在ht[0]里查找，没找到，就会继续在ht[1]里面查找；新添加的键值对一律保存到ht[1] **

---

### 跳跃表

跳跃表是一种有序数据结构，效率可以和平衡树相媲美。

zskiplist

```c
typedef struct zskiplist {
    // 表头、表尾
    structz skiplistNode *header, *tail;
    // 表中节点的数量
    unsigned long length;
    // 最大层数
    int level;
} zskiplist;
```

redis.h/zskiplistNode

```c
typedef struct zskiplistNode {
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } levle[];
    // 后退指针
    struct zskiplistNode *backward;
    // 分值
    double score;
    // 成员对象(key)
    robj *obj;
}
```

![跳跃表]( https://i.bmp.ovh/imgs/2019/10/acbd39d6cb4ae728.png )

每个节点包含一个level数组，表示当前节点到下一特定节点的跨度，如上图，表头节点到o1，层数组长度为4，跨度为1。

调表对应Java中的Set，构造过程如下

* 新增一个key时（`*obj`），如果存在，覆盖更新，保证成员对象唯一
* 计算节点level，越高概率越小
* 按score排列，score相同，则根据key的字母序

***

### 整数集合

当一个集合中只包含不多的整数元素，Redis会使用整数集合作为集合键的底层实现

intset.h/intsset

```c
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组,从小到大排列，且不重复
    int8_t contents[];
} intset;
```

#### 升级

将一个超过当前编码集的整数添加到列表时，会将当前所有的元素都升级到集合内最大值适用的编码。带来的好处是`提升灵活性`、`节约内存`

#### 降级

整数集合不支持降级操作

---

### 压缩列表

当一个列表键或哈希键只包含少量元素，并且每个元素要么是`小整数`，要么是`短字符`，Redis 会使用压缩列表来实现。

| zlbytes  | zltail         | zllen    | entry1 | entry2 | ……   | entryN | zlend            |
| -------- | -------------- | -------- | ------ | ------ | ---- | ------ | ---------------- |
| 总字节数 | 到表头的偏移量 | 节点数量 | 节点   |        |      |        | 0xFF用于标记末端 |

 **当zllen为16位，当节点数量等于65535时，节点数量需要遍历整个列表来统计**

节点组成：

| previous_entry_length | encoding       | content              |
| --------------------- | -------------- | -------------------- |
| 前一节点的长度        | 数据类型及长度 | 内容。字节数组或整数 |



#### 连锁更新

前一个节点的值改变时（从一字节扩展到5字节，或相反），后续节点的previous_entry_length也要申请新的空间，最坏情况下，需要对压缩列表执行N次重分配操作，每次重分配复杂度为O（N）。



