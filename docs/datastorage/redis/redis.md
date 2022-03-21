# Redis

## Redis 单线程 & 为什么快

Redis **执行命令阶段是完全单线程的**。在较新的版本中引入了多线程。**多线程主要用于网络 I/O 阶段**，也就是接收命令和写回结果阶段，而在执行命令阶段，还是由单线程串行执行。由于执行时还是串行，因此无需考虑并发安全问题。

。。。。。。。。。。最后再搞一下

### Redis 为什么使用单进程、单线程也很快

1、基于内存的操作

2、使用了 I/O 多路复用模型，select、epoll 等，基于 reactor 模式开发了自己的网络事件处理器

3、单线程可以避免不必要的上下文切换和竞争条件，减少了这方面的性能消耗。

## 场景

+ 缓存（核心）

  + 不失效的情况下作为查找表：DNS
  + 会话缓存：可以使用 Redis 来统一存储多台应用服务器的会话信息 session。一个用户可以请求任意一个应用服务器，从而更容易实现高可用性以及可伸缩性
  + 正常缓存：将热点数据放到内存中，设置内存的最大使用量以及淘汰策略来保证缓存的命中率。

+ 计数器

  可以对 String 进行自增自减运算，从而实现计数器功能。Redis 这种内存型数据库的读写性能非常高，很适合存储频繁读写的计数量。

+ 分布式锁（setnx + lua 脚本）

  可以使用 Redis 自带的 SETNX 命令实现分布式锁，除此之外，还可以使用官方提供的 RedLock 分布式锁实现

+ 消息队列

  List 是一个双向链表，可以通过 lpush 和 rpop 写入和读取消息。也可以用最新的 Stream。

+ 排行榜（zset）

+ 访客统计（hyperloglog）

## 与 memcached 的对比

两者都是非关系型内存键值数据库，主要有以下不同：

+ 数据类型

  Memcached 仅支持字符串类型，而 Redis 支持五种不同的数据类型，可以更灵活地解决问题。

+ 数据持久化

  Redis 支持两种持久化策略：RDB 快照和 AOF 日志，而 Memcached 不支持持久化。

+ 分布式

  Memcached 不支持分布式，只能通过在客户端使用一致性哈希来实现分布式存储，这种方式在存储和查询时都需要先在客户端计算一次数据所在的节点。

  Redis Cluster 实现了分布式的支持。

+ 内存管理机制

  - 在 Redis 中，并不是所有数据都一直存储在内存中，可以将一些很久没用的 value 交换到磁盘，而 Memcached 的数据则会一直在内存中。

  - Memcached 将内存分割成特定长度的块来存储数据，以完全解决内存碎片的问题。但是这种方式会使得内存的利用率不高，例如块的大小为 128 bytes，只存储 100 bytes 的数据，那么剩下的 28 bytes 就浪费掉了。

## 数据淘汰策略

可以设置内存最大使用量，当内存使用量超出时，会施行数据淘汰策略。

Redis 具体有 8（6+2） 种淘汰策略：

| 策略            | 描述                                                     |
| :-------------- | :------------------------------------------------------- |
| volatile-lru    | 从 已设置过期时间的数据集 中挑选 最近最少使用 的数据淘汰 |
| volatile-ttl    | 从 已设置过期时间的数据集 中挑选 将要过期 的数据淘汰     |
| volatile-random | 从 已设置过期时间的数据集 中 任意 选择数据淘汰           |
| allkeys-lru     | 从 所有数据集 中挑选 最近最少使用 的数据淘汰             |
| allkeys-random  | 从 所有数据集 中 任意 选择数据进行淘汰                   |
| noeviction      | 禁止驱逐数据                                             |

作为内存数据库，出于对性能和内存消耗的考虑，Redis 的淘汰算法实际实现上并非针对所有 key，而是抽样一小部分并且从中选出被淘汰的 key。

使用 Redis 缓存数据时，为了提高缓存命中率，需要保证缓存数据都是热点数据。可以将内存最大使用量设置为热点数据占用的内存量，然后启用 allkeys-lru 淘汰策略，将最近最少使用的数据淘汰。

Redis 4.0 引入了 volatile-lfu 和 allkeys-lfu 淘汰策略，LFU 策略通过**统计访问频率，将访问频率最少的键值对淘汰**。

## 数据类型

### String：字符串

<img src="image-20220318140042505.png" alt="image-20220318140042505" style="zoom: 33%;" />

```c
struct sdshdr {
    int len;
    int free;
    char buf[];
};
```

+ 获取长度的时间复杂度为 O(1)

+ 是安全的不会缓冲区溢出

+ 修改字符串长度不一定会导致内存重分配：字符串小于1M，采用的是加倍扩容的策略，也就是多分配100%的剩余空间，当大于1M，每次扩容，只会多分配1M的剩余空间

+ 可以保存文本数据或者二进制数据。

### List：列表

<img src="image-20220318140102416.png" alt="image-20220318140102416" style="zoom:33%;" />

```
> rpush list-key item
> rpush list-key item2
> rpush list-key item

lrange list-key 0 -1
1) "item"
2) "item2"
3) "item"

> lindex list-key 1
"item2"
```



### Hash：哈希对象

<img src="image-20220318140343095.png" alt="image-20220318140343095" style="zoom:33%;" />

```
hset
hgetall
hdel
```





### Set：集合

Set 是一个特殊的 value 为空的 Hash。

<img src="image-20220318140238755.png" alt="image-20220318140238755" style="zoom:33%;" />

```
> sadd set-key item
> sadd set-key item2
> sadd set-key item3
> sadd set-key item

> smembers set-key
1) "item"
2) "item2"
3) "item3"

> sismember set-key item4
(integer) 0
> sismember set-key item
(integer) 1
```



### Zset：有序集合

<img src="image-20220318140613664.png" alt="image-20220318140613664" style="zoom:33%;" />

ZSet（有序集合）当前有两种编码：ziplist、skiplist。ziplist使用压缩列表实现，当保存的元素长度都小于64字节，同时数量小于128时，使用该编码方式;否则会使用 skiplist。详见下方的数据结构部分

```
> zadd zset-key 728 member1
> zadd zset-key 982 member0
> zadd zset-key 982 member0

> zrange zset-key 0 -1 withscores
1) "member1"
2) "728"
3) "member0"
4) "982"

> zrangebyscore zset-key 0 800 withscores
1) "member1"
2) "728"
```



### HyperLogLog

通常用于基数统计（使用少量固定大小的内存，**用来统计集合中唯一元素的数量）**。统计结果不是精确值，而是一个带有0.81%标准差（standard error）的近似值。所以，HyperLogLog 适用于一些对于统计结果精确度要求不是特别高的场景，例如网站的 UV 统计。

> too much math work!

### Bitmap

位图。其实就是 Redis 实现的布隆过滤器。（大致判断某个 key 是不是存在，防止缓存穿透问题。不一定能判断某个 key 一定存在，但是可以判断其一定不存在）

它本质上是 String 数据结构，只不过操作的粒度变成了位，即 bit。

### Stream

主要用于消息队列，类似于 kafka，可以认为是 pub/sub 的改进版。提供了消息的持久化和主备复制功能，可以让任何客户端访问任何时刻的数据，并且能记住每一个客户端的访问位置，还能保证消息不丢失。

<img src="v2-4103fa7c6a73fcfdea12b310a1df034a_1440w.jpg" alt="img"  />

它有一个消息链表，将所有加入的消息都串起来，每个消息都有一个唯一的ID和对应的内容。消息是持久化的，Redis重启后，内容还在。

+ 每个 Stream 都有唯一的名称，它就是 Redis 的key，在我们首次使用 `xadd` 指令追加消息时自动创建。
+ 每个 Stream 都可以挂多个消费组，每个消费组会有个游标 `last_delivered_id` 在Stream数组之上往前移动，表示当前消费组已经消费到哪条消息了。每个消费组都有一个Stream内唯一的名称。**每个消费组(Consumer Group)的状态都是独立的，相互不受影响。也就是说同一份Stream内部的消息会被每个消费组都消费到。**
+ 同一个消费组 (Consumer Group) 可以挂接多个消费者 (Consumer)，**这些消费者之间是竞争关系**，任意一个消费者读取了消息都会使游标 `last_delivered_id` 往前移动。每个消费者者有一个组内唯一名称。

### Geo

可以将用户给定的地理位置信息储存起来， 并对这些信息进行操作：获取2个位置的距离、根据给定地理位置坐标获取指定范围内的地理位置集合。

## 数据结构

![preview](v2-1811a380e3f75ffdfb33869c3e359e6c_r.jpg)

### 压缩列表 ziplist

> redis 压缩列表ziplist、quicklist - 诚毅的文章 - 知乎 https://zhuanlan.zhihu.com/p/375414918

一整块连续内存，没有多余的内存碎片，更新会导致内存 realloc 与内存复制，平均时间复杂度为 `O(N)`

<img src="v2-696f6096610911ef3dd63c5904a3eb50_b.jpg" alt="img" style="zoom: 50%;" />

#### 结构

1、**zlbytes**：压缩列表的字节长度，占4个字节，因此压缩列表最长(2^32)-1字节；

2、**zltail**：压缩列表尾元素相对于压缩列表起始地址的偏移量，占4个字节；

3、**zllen**：压缩列表的元素数目，占两个字节；那么当压缩列表的元素数目超过 `(2^16)-1` 怎么处理呢？此时通过 zlen 无法获得压缩列表的元素数目，必须遍历整个压缩列表才能获取到元素数目；

4、**entryX**：压缩列表存储的若干个元素，可以为字节数组或者整数

<img src="image-20220318143932564.png" alt="image-20220318143932564" style="zoom: 25%;" />

+ previous_entry_length：该属性根据前一个节点的大小不同可以是 1 个字节或者 5 个字节。
  + 如果前一个节点的长度小于254个字节，那么 previous_entry_length 的大小为 1 个字节，**即前一个节点的长度可以使用 1 个字节表示**
  + 如果前一个节点的长度大于等于254个字节，那么 previous_entry_length 的大小为 5 个字节**，第 1 个字节会被设置为 0xFE(十进制的254），之后的四个字节则用于保存前一个节点的长度。**
+ encoding：编码类型
+ content：保存节点的值

5、**zlend**：压缩列表的结尾，占一个字节，恒为**0xFF**。

#### 操作

**逆向遍历**：从表尾开始向表头进行遍历（但是插入还是在表头）

在 p 之后插入元素：计算新 entry 需要的空间，移动 p 以后的 entry 腾出空间，修改 p 和 p 以后的 prelen，然后插入新 entry，修改 tail 值。

> 连锁更新：当添加或删除节点时，可能就会因为previous_entry_length的变化导致发生连锁的更新操作。假设 **e1 的 previous_entry_length 只有1个字节**，而新插入的节点大小超过了 254 字节，此时由于 e1 的 previous_entry_length 无法存下该长度，就会将 **previous_entry_length 的长度更新为5字节**。假设 e1 原本的大小为252字节，**当 previous_entry_length 更新后它的大小则超过了 254，此时又会引发对 e2 的更新**。顺着这个思路，一直更新下去……
>
> 因为连锁更新在最坏情况下需要对压缩列表执行N次空间重分配操作，而每次空间重分配的最坏复杂度为o(N)，所以连锁更新的最坏复杂度为O($N^2$)

| <img src="v2-fcdd83a13e97d88f5c1e6a8747380726_b.jpg" alt="img" style="zoom: 33%;" /> | <img src="v2-c0760d9162c4850db7c5ec2b00adcf98_b.jpg" alt="img" style="zoom: 50%;" /> |
| ------------------------------------------------------------ | ------------------------------------------------------------ |



### 双向链表 linkedlist

发布与订阅、慢查询、监视器等功能也用到了链表， Redis 服务器本身还使用链表来保存多个客户端的状态信息， 以及使用链表来构建客户端输出缓冲等。

![img](v2-8f0014f1080d060a991f5acf143b1ef6_b.jpg)

```c
typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 链表所包含的节点数量
    unsigned long len;
    // 节点值复制函数
    void *(*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr, void *key);
} list;
```

特点：

+ 双端： 链表节点带有 prev 和 next 指针， 获取某个节点的前置节点和后置节点的复杂度都是 O(1) 。

+ 无环： 表头节点的 prev 指针和表尾节点的 next 指针都指向 NULL ， 对链表的访问以 NULL 为终点。

+ 带表头指针和表尾指针： 通过 list 结构的 head 指针和 tail 指针， 程序获取链表的表头节点和表尾节点的复杂度为 O(1) 。

+ 带长度计数器： 程序使用 list 结构的 len 属性来对 list 持有的链表节点进行计数， 程序获取链表中节点数量的复杂度为 O(1) 。

+ 多态： 链表节点使用 void* 指针来保存节点值， 并且可以通过 list 结构的 dup 、 free 、 match 三个属性为节点值设置类型特定函数， 所以链表可以用于保存各种不同类型的值。

### quicklist

<img src="v2-ffc658aae5b8a0ba92d31bc7bd1e3724_1440w.jpg" alt="img" style="zoom:50%;" />

在 redis 3.2 之后，list 的底层实现变为快速列表 quicklist。快速列表是 linkedlist 与 ziplist 的结合: quicklist 包含多个内存不连续的节点，但每个节点本身就是一个 ziplist。

虽然压缩列表是通过紧凑型的内存布局节省了内存开销，但是因为它的结构设计，如果保存的元素数量增加，或者元素变大了，压缩列表会有连锁更新的风险，一旦发生，会造成性能下降。

quicklist 解决办法是：**通过控制每个链表节点中的压缩列表的大小或者元素个数，来规避连锁更新的问题。**

### listpack 

<img src="image-20220318164348917.png" alt="image-20220318164348917" style="zoom:33%;" />

与 ziplist entry 不同的是，len 保存的是 encoding+data 的总长度。listpack 只记录当前节点的长度，当向 listpack 加入一个新元素的时候，不会影响其他节点的长度字段的变化，从而避免了压缩列表的连锁更新问题。

### hash table & 字典

dictht 是一个散列表结构，使用拉链法解决哈希冲突。

<img src="image-20220318185033506.png" alt="image-20220318185033506" style="zoom:50%;" />

```c
/* 哈希表节点 */
typedef struct dictEntry {
    void *key; /* 键名 */
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v; /* 值 */
    struct dictEntry *next; /* 指向下一个节点, 将多个哈希值相同的键值对连接起来*/
} dictEntry;

/* 哈希表结构 */
typedef struct dictht {
    dictEntry **table; /* 哈希表节点数组, 个人理解是 dictEntry* 的一维数组，用指针表示 */ 
    unsigned long size; /* 哈希表大小 */
    unsigned long sizemask; /* 哈希表大小掩码，用于计算哈希表的索引值，大小总是dictht.size - 1，由于 size 通常很少 2^n，所以 sizemask 其实就是每一位都是 1*/
    unsigned long used; /* 哈希表已经使用的节点数量 */
} dictht;
```

dict

```c
/* 字典结构 每个字典有两个哈希表，实现渐进式哈希时需要用在将旧表 rehash 到新表 */
typedef struct dict {
    dictType *type; /* 类型特定函数 */
    void *privdata; /* 保存类型特定函数需要使用的参数 */
    dictht ht[2]; /* 保存的两个哈希表，ht[0]是真正使用的，ht[1]会在rehash时使用 */
    long rehashidx; /* rehashing not in progress if rehashidx == -1 rehash进度，如果不等于-1，说明还在进行rehash */
    unsigned long iterators; /* number of iterators currently running 正在运行中的遍历器数量 */
} dict;

/* 保存一连串操作特定类型键值对的函数 */
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key); /* 哈希函数 */
    void *(*keyDup)(void *privdata, const void *key); /* 复制键函数 */
    void *(*valDup)(void *privdata, const void *obj); /* 复制值函数 */
    int (*keyCompare)(void *privdata, const void *key1, const void *key2); /* 比较键函数 */
    void (*keyDestructor)(void *privdata, void *key); /* 销毁键函数 */
    void (*valDestructor)(void *privdata, void *obj); /* 销毁值函数 */
} dictType;
```

type 属性和 privdata 属性是针对不同类型的键值对，为创建多态字典而设置的：type 属性是一-个指向 dictType 结构的指针，每个 dictrype 结构保存了一簇用
于操作特定类型键值对的函数，Redis 会为用途不同的字典设置不同的类型特定函数。而 privdata 属性则保存了需要传给那些类型特定函数的可选参数。

ht 属性是一个包含两个项的数组，数组中的每个项都是一个 dictht 哈希表，一般情况下，字典只使用 ht [0] 哈希表，ht [1] 哈希表只会在对 ht[0] 哈希表进行 rehash 时使用。另一个和 rehash 有关的属性是 rehashidx，它记录了 rehash 目前的进度，如果目前没有在进行 rehash，那么它的值为-1。

<img src="image-20220318185829168.png" alt="image-20220318185829168" style="zoom: 50%;" />

#### 哈希算法

当要将一个新的键值对添加到字典里时，程序需要先根据键值对的键计算出哈希值和索引值，将键值对添加到哈希表数组的指定索引上去。

> 通常 size 是 $2^n$，sizemask 其实就是每一位都是 1 的表示。通过这样的 & 操作就可以保证下标在合法范围内。

```c
hash = dict->type->hashFunction(key);
index = hash & dict->hx[x].sizemask;
```

#### rehash 和渐进式 rehash

##### rehash

随着操作的不断执行，哈希表保存的键值对会逐渐地增多或者减少，为了让哈希表的负载因子（已用已存键值对数量 used 和槽数 size 的比值，默认为 5）维持在一个合理的范围之内，当哈希表保存的键值对数量太多或者太少时，程序需要对哈希表的大小进行相应的扩展或者收缩。

rehash 的步骤如下：

+ 为字典的 ht[1] 哈希表分配空间，这个哈希表的空间大小取决于要执行的操作，以及 ht[0] 当前包含的键值对数量（也即是ht [0].used 属性的值)
  + 如果执行的是扩展操作，那么 ht [1] 的大小为第一个大于等于 `ht [0].used *2`的 $2^n$;
  + 如果执行的是收缩操作，那么 ht[1] 的大小为第一个大于等于 ht[0]. used的$2^n$.
+ 将保存在 ht [0] 中的所有键值对 rehash 到 ht[1] 上面：rehash指的是重新计算键的哈希值和索引值，然后将键值对放置到 ht[1] 哈希表的指定位置上。
+ 当 ht[0] 包含的所有键值对都迁移到了 ht[1] 之后，释放ht[0]，将ht [1] 设置为 ht[0]，并在 ht[1] 新创建一个空白哈希表，为下一次 rehash 做准备。

##### 渐进式 hash

hash 并不是一次性完成的，如果服务器存储的 (k,v) 数量过多，庞大的计算量可能导致服务器在一段时间内停止服务。因此是分多次、渐进式地将 ht[0] 里面的键值对慢慢的 rehash 到 ht[1]，具体步骤：

+ 为 ht[1] 分配空间
+ 在字典中维持一个索引计数器变量 rehashidx，并将它的值设置为 0，表示 rehash 工作正式开始。
+ 在 rehash 进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将 ht[0] 哈希表在 rehashidx 索引上的所有键值对 rehash 到 ht[1]，再将 rehashidx 的值 +1。
+ 随着字典操作的不断执行，最终在某个时间点上，ht[0] 的所有键值对都会被 rehash 至 ht[1]，这时程序将 rehashidx 属性的值设为 -1，表示 rehash 操作已完成。

在渐进式 rehash 进行期间，字典的删除、查找、更新等操作会在两个哈希表上进行。例如，要在字典里面查找一个键的话，程序会先在 ht[0] 里面进行查找，如果没找到的话，就会继续到 ht[1] 里面进行查找。**新添加到字典的键值对一律会被保存到 ht[1] 里面,而 ht[0] 则不再进行任何添加操作**，这一措施保证了 ht[0] 包含的键值对数量会只减不增，并随着 rehash 操作的执行而最终变成空表。

### skiplist 

> Redis系列(七)底层数据结构之跳跃表 - 呼延十的文章 - 知乎 https://zhuanlan.zhihu.com/p/103368713
>
> 跳跃表的用途：
> 1、实习有序集合键
> 2、作为集群节点中的内部数据结构

+ 跳跃表是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。
+ 跳跃表支持平均 O(logN)、最坏 O(N) 复杂度的节点查找，还可以通过顺序性操作来批量处理节点。在大部分情况下，跳跃表的效率可以和平衡树相媲美，并且因为跳跃表的实现比平衡树要来得更为简单，所以有不少程序都使用跳跃表来代替平衡树。
+ 跳跃表是基于多指针有序链表实现的，可以看成多个有序链表。在查找时，从上层指针开始查找，找到对应的区间之后再到下一层去查找。

与红黑树等平衡树相比，跳跃表具有以下优点：

- 插入速度非常快速，不需要进行旋转等操作来维护平衡性；
- 更容易实现；

```c
typedef struct zskiplist{
    // 表头结点和尾节点
    struct zskiplistNode *header, *tail;
    // 表中节点的数量
    unsigned int length;
    // 表中层数最大的节点的层数
    int level;
} zskiplist;
```

```c
typedef struct zskiplistNode{
    struct zskiplistLevel{
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[]; // (本节点，以及本节点在所有层的索引)

    struct zskiplistNode *backward; // 后退指针
    double score; // 分值
    sds ele; // 成员对象
} zskiplistNode;
```

+ **ele**：用于记录跳跃表节点的值，类型是 **SDS**。
+ **score**：保存跳跃表节点的分值，在跳跃表中，节点按各自所保存的分值**从小到大**排列。
+ **backward**：后退指针，它指向当前节点的前一个节点，在程序从表尾向表头遍历时使用。**因为一个跳跃表节点只有一个后退指针，所以每次只能后退至前一个节点。**
+ **level**：跳跃表节点的层（每个跳跃表节点最多有  32 层），每个层都带有两个属性：**前进指针 `forward`** 和**跨度 `span`**。
  + **前进指针**用于访问位于表尾方向的其他节点
  + 而**跨度**则记录了**前进指针**所指向节点和当前节点的距离

#### 一些问题

**插入时的层高问题**：如果上层链表节点的个数是下层节点个数的一半，整个查找过程就类似于二分查找，复杂的是 O(logn)。但是在插入和删除的时候有很大的问题，插入和删除会打乱上下两层链表上个数2：1对应关系，需要对后面的节点重新调整，插入复杂度会退化到 O(n)。因此，为了避免这一问题，skiplist 不要求上下相邻两层链表之间的节点个数有严格的对应关系，而是 **为每个节点随机出一个层数(level)**。因此，**插入操作只需要修改节点前后的指针，而不需要对多个节点都进行调整**，这就降低了插入操作的复杂度。直观上期望的目标是 50% 的概率被分配到 `Level 1`，25% 的概率被分配到 `Level 2`，12.5% 的概率被分配到 `Level 3`，以此类推。

**顺序问题**：在 zset 中，是可以存储分数一样的值的。Redis 除了按照分值排序之外，还会按照字符串的字典序来存储。

**排名问题**：前面提到了 `跨度` 这个属性，当我们需要查找某个元素的排名时，跳跃表首先开始一次查询过程，找到该节点时，也可以找到从顶层索引找到该节点的 **查找路径**, 将 路径上的所有节点的 **跨度** 值相加就是该节点的排名。
