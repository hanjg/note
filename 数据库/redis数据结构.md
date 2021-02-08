[toc]
## redis特点 ##
- 优点：
  1. **性能极高**。Redis能读的速度是110000次/s,写的速度是81000次/s 。
  2. **原子性**。Redis的所有操作都是原子性的，同时Redis还支持对几个操作全并后的原子性执行。
  3. **数据类型丰富**。Redis支持Strings, Lists, Hashes, Sets 及 Ordered Sets 等数据类型操作。
  4. **可持久化**。相比于其他内存数据库，可以将数据保存在磁盘中，启动时再次加载。
  5. **支持主从**。master-slave模式的数据备份。
  6. 其他特性。Redis还支持 publish/subscribe, 通知, key 过期等等特性。
- 缺点：
  1. 受限于**物理内存**。通常用于小数据量的高性能操作。
  2. [持久化可能丢失数据](https://blog.csdn.net/qq_40369829/article/details/107829437)。尽量做缓存而不是存储。
- [官网](http://www.redis.net.cn/)

## 数据结构 ##
### SDS ###
- 简单动态字符串。
- 数据结构。<br>![210204.sds.png](https://img-blog.csdnimg.cn/20210208131429643.png)
```c
struct sdshdr {
    // 字符串长度
    int len;
    // buf数组中未使用的字节数
    int free;
    // 字节数组，用于保存字符串
    char buf[];
};
```

- 对比C字符串。
  - O(1)获取字符串长度。
  - 杜绝缓冲区溢出。C手动分配内存和拼接字符串可能导致。
  - 减少修改时重新分配内存的次数。空间预分配、惰性释放。
  - 二进制安全。\0在C中会被识别为结束。

### linkedlist ###
- 双向无环链表。<br>![210204.list.png](https://img-blog.csdnimg.cn/20210208131508937.png)
```c
typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 链表包含的节点数量
    unsigned long len;
    // 节点复制函数
    void *(*dup)(void *ptr);
    // 节点释放函数
    void (*free)(void *ptr);
    // 节点对比函数
    int (*match)(void *ptr, void *key);
} list;
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点值
    void *value;
} listNode
```

### dict ###
- 字典。链表解决hash冲突。两块空间rehash。<br>![210204.dict.png](https://img-blog.csdnimg.cn/20210208131537633.png)
```c
typedef struct dict {
    // 类型特定函数
    dictType *type;
    // 私有数据
    void *privdata;
    // 哈希表
    dictht ht[2];
    //rehash索引
    // 当rehash不在进行时，值为-1
    int rehashidx;
}
typedef struct dictht{
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小
    unsigned long size;
    // 哈希表大小掩码，用于计算索引值，总是等于 size-1
    unsigned long sizemask;
    // 该哈希表已有节点数量
    unsigned long used;
} dictht
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {
        void *val;
        unit64_t u64;
        nit64_t s64;
    } v;
    // 指向下一个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```

- rehash时机：
  - 扩展：负载因子>=1，有BGSAVE则>=5.
  - 收缩：负载因子<=0.1.
- **渐进式rehash**流程：
  - ht[1]分配空间。
  - rehashidx置为1。
  - 对字典增删改查时，附带将ht[0].table[rehashidx]上的所有键值对迁移到ht[1]，rehashidx++。
  - 键值对全迁移之后rehashidx重置为-1
- 渐进式rehash过程中：
  - 查找：先查ht[0]再查ht[1]。
  - 新增：只在ht[1]。

### skiplist ###
- 多层链表，每个节点层数[1,32]之间的随机数，每层单独的前进指针和跨度。<br>![210204.skiplist.png](https://img-blog.csdnimg.cn/20210208131604886.png)
```c
typedef struct zskiplistNode {
	// 头尾节点
	struct zskiplistNode *header, *tail;
	// 节点数量
	unsigned long length;
	// 层数最大的节点层数
	int level;
}
typedef struct zskiplistNode {
    // 成员对象
    robj *obj;
    // 分值
    double score;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度，沿途累加的跨度为节点排位
        unsigned int span;
    } level[];
    // 后退指针，逆向遍历使用
    struct zskiplistNode *backward;
} zskiplistNode;
```

- 成员对象必须唯一。
- 分值相同时按照对象字典序排序。
- [leetcode简化版实现](https://github.com/hanjg/leetcode/blob/master/src/design/Skiplist.java)。
- [redis源码](https://github.com/redis/redis/blob/e9aba28932c6a0450c14ecd984603d41f31e0f0d/src/t_zset.c)。

### intset ###
- 整数集合。保存唯一类型的整数值，int16_t或int32_t或int64_t。<br>![210204.intset.png](https://img-blog.csdnimg.cn/2021020813163740.png)
```c
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组
    int8_t contents[];
} intset;
```

- **集合升级**：新元素类型比集合中现有类型长
  - 扩展contents数组空间。
  - 现有元素转换成新元素类型，并移到对应的位置。
  - 新元素加到数组中（最前或者最后）。
- 升级的优势：
  - 提升灵活性。一种结构存储多种类型（同时一种）而不必担心类型错误。
  - 节约内存。
- 不支持降级操作。

### ziplist ###
- 压缩列表。连续内存块组成的顺序存储结构。<br>![210205.ziplist.png](https://img-blog.csdnimg.cn/20210208131732207.png)
- ![210205.ziplistkey.png](https://img-blog.csdnimg.cn/20210208131750649.png)
- entry的结构。<br>![210205.ziplistentry.png](https://img-blog.csdnimg.cn/20210208193112426.png)
  - previous_entry_length：前一个节点长度，字段长度为1或5个字节。逆向遍历使用。
- **连锁更新**。
  - 插入和删除节点，导致自身和下游的previous_entry_length长度不够，节点均需要扩展。
  - 连续的扩展概率较低。

## 对象类型 ##
- 对象数据结构。
```c
typedef struct redisObject {
    // 类型
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    // 底层数据结构的指针
    void *ptr;
	// 引用计数，0则释放空间
	int refcount
	// 最后一次被命令程序访问的时间
	unsigned lru;
} robj;
```

- 共享0~9999的字符串对象，节约空间。

### String ###
- 数据结构为3种SDS编码：
  - int。保存可以用long类型表示的整数。<br>![210207.int.png](https://img-blog.csdnimg.cn/20210208132032695.png)
  - embstr。字符串长度<=32字节，分配为连续的内存空间，只需一次分配和回收。<br>![210207.embstr.png](https://img-blog.csdnimg.cn/20210208132032616.png)
  - raw。字符串长度>32字节。<br>![210207.raw.png](https://img-blog.csdnimg.cn/20210208132032617.png)
- int->raw：追加字符等改变类型的操作。
- embstr->raw：embstr只读，任何修改命令都会转成raw。

### List ###
- 分为两种编码：
  - ziplist：元素个数小于512且长度都小于64字节。<br>![210207.set.ziplist.png](https://img-blog.csdnimg.cn/20210208191538643.png)
  - linkedlist。<br>![210207.set.linkedlist.png](https://img-blog.csdnimg.cn/20210208191637593.png)
- ziplist->linkedlist。条件不满足时单向转换。


### Hash ###
- 分为两种编码：
  - ziplist：同一键值对键和值相邻，键在前。<br>![210207.hash.ziplist.png](https://img-blog.csdnimg.cn/20210208191701647.png)
  - dict<br>![210207.hash.hashtable.png](https://img-blog.csdnimg.cn/20210208191701694.png)
- 元素个数小于512且长度都小于64字节时，ziplist，超出转为dict。

### Set ###
- 分为两种编码：
  - intset：所有元素都是整数且数量不超过512。<br>![210207.set.intset.png](https://img-blog.csdnimg.cn/20210208191754309.png)
  - dict<br>![210207.set.hashtable.png](https://img-blog.csdnimg.cn/20210208191815208.png)
- intset->hashtable：不满足条件时单向转换。

### Zset ###
- 分两种编码：
  - ziplist：按分值从小到大保存元素和分值。<br>![210207.zset.ziplist.png](https://img-blog.csdnimg.cn/20210208191849540.png)
  - skiplist：同时使用跳表和字典，两个**共享**成员和分值。
    - 字典：O(1)随机查分值，O(NlogN)范围查分值。
    - 跳表：O(logN)范围查询，O(logN)随机查询。<br>![210207.zset.skiplist.png](https://img-blog.csdnimg.cn/20210208191850768.png)
- 元素个数小于128且长度都小于64字节时，ziplist，超出转为skiplist+dict。



## 相关参考 ##
- [一文读懂Redis常见对象类型的底层数据结构](https://mp.weixin.qq.com/s/JFRmw7ko5CogdZJWTCxDnQ)
- [跳表](https://mp.weixin.qq.com/s/45WU1EUAvfNg5o_NZ0DVgQ)

