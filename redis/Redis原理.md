# Redis原理

## 底层数据结构

### 1.SDS简单动态字符串

SDS简单动态字符串是redis自己实现的一种字符串数据结构，他有以下特点。**1.O1时间获取字符串长度。2.二进制安全，不会跳过/0这个字符3.可以修改**

结构中的每个成员变量分别介绍下：

- **len，记录了字符串长度**。这样获取字符串长度的时候，只需要返回这个成员变量值就行，时间复杂度只需要 O（1）。

- **alloc，分配给字符数组的空间长度**。这样在修改字符串的时候，可以通过 `alloc - len` 计算出剩余的空间大小，可以用来判断空间是否满足修改需求，如果不满足的话，就会自动将 SDS 的空间扩展至执行修改所需的大小，然后才执行实际的修改操作，所以使用 SDS 既不需要手动修改 SDS 的空间大小，也不会出现前面所说的缓冲区溢出的问题。

  如果所需的 sds 长度**小于 1 MB**，那么最后的扩容是按照**翻倍扩容**来执行的，即 2 倍的newlen

  如果所需的 sds 长度**超过 1 MB**，那么最后的扩容长度应该是 newlen **+ 1MB**。

- **flags，用来表示不同类型的 SDS**。一共设计了 5 种类型，分别是 sdshdr5、sdshdr8、sdshdr16、sdshdr32 和 sdshdr64，后面在说明区别之处。

- **buf[]，字符数组，用来保存实际数据**。不仅可以保存字符串，也可以保存二进制数据。

**之所以 SDS 设计5种不同类型的结构体，是为了能灵活保存不同大小的字符串，从而有效节省内存空间**。比如，在保存小字符串时，结构头占用空间也比较少。

除了设计不同类型的结构体，Redis 在编程上还**使用了专门的编译优化来节省内存空间**，即在 struct 声明了 `__attribute__ ((packed))` ，它的作用是：**告诉编译器取消结构体在编译过程中的优化对齐，按照实际占用字节数进行对齐**。





### 2.IntSet整数集合

整数集合是Set的底层结构之一，如果set只包含数值并且元素数量不大，可以用IntSet，**它有以下特点1.不重复的有序集合，2.是一种特殊的数组3.支持动态升级**

```c
typedef struct intset {
    //编码方式
    uint32_t encoding;
    //集合包含的元素数量
    uint32_t length;
    //保存元素的数组
    int8_t contents[];
} intset;
```

现在，往这个整数集合中加入一个新元素 65535，这个新元素需要用 int32_t 类型来保存，所以整数集合要进行升级操作，首先需要为 contents 数组扩容，**在原本空间的大小之上再扩容多 80 位（4x32-3x16=80），这样就能保存下 4 个类型为 int32_t 的元素**。

![img](https://cdn.xiaolincoding.com//mysql/other/e2e3e19fc934e70563fbdfde2af39a2b.png)

![img](https://cdn.xiaolincoding.com//mysql/other/e84b052381e240eeb8cc97d6b729968b.png)

整数集合升级的好处是**节省内存资源**，一开始只用最小的，后面遇到大的再升级转换

### 3.Dict哈希表

```c
typedef struct dictht {//哈希表
    //哈希表数组
    dictEntry **table;
    //哈希表大小
    unsigned long size;  
    //哈希表大小掩码，用于计算索引值
    unsigned long sizemask;
    //该哈希表已有的节点数量
    unsigned long used;
} dictht;
```

```c
typedef struct dictEntry {//键值对
    //键值对中的键
    void *key;
  
    //键值对中的值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    //指向下一个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry;
```

```c
typedef struct dict {//
    …
    //两个Hash表，交替使用，用于rehash操作
    dictht ht[2]; 
    …
} dict;
```

![image-20240811215845722](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240811215845722.png)

关于哈希表的数据结构看源码和上图就行了，以下是一些注意点

**采用头插法，方便快捷**

**哈希表的size的2的整数倍，这样一来做哈希运算的时候才等价于模运算。**

**Dict的rehash**
Dict的rehash并不是一次性完成的。

因拷贝数据的耗时，影响 Redis 性能的情况，所以 Redis 采用了**渐进式 rehash**，也就是将数据的迁移的工作不再是一次性迁移完成，而是分多次迁移。

渐进式 rehash 步骤如下：

- 给「哈希表 2」 分配空间；
- **在 rehash 进行期间，每次哈希表元素进行新增、删除、查找或者更新操作时，Redis 除了会执行对应的操作之外，还会顺序将「哈希表 1 」中索引位置上的一整个链表 迁移到「哈希表 2」 上**；
- 随着处理客户端发起的哈希表操作请求数量越多，最终在某个时间点会把「哈希表 1 」的所有 key-value 迁移到「哈希表 2」，从而完成 rehash 操作。
- 新增直接操作哈希表2，其他都会在两个表同时查找，查到了再操作，肯定只有一个表会有数据

#### 触发条件：

​		收缩或者扩展都有可能，收缩时则used/size<0.1且size大于4.

​		扩容时：used/size>=1 &&没有做bgsave或bgrewrite。 used/size>5时强制扩容





### 4.ZipList压缩列表

压缩列表是 Redis 为了节约内存而开发的，它是**由连续内存块组成的顺序型数据结构**，有点类似于数组。

![img](https://cdn.xiaolincoding.com//mysql/other/ab0b44f557f8b5bc7acb3a53d43ebfcb.png)

压缩列表在表头有三个字段：

- ***zlbytes***，记录整个压缩列表占用对内存字节数；

- ***zltail***，记录压缩列表「尾部」节点距离起始地址由多少字节，也就是列表尾的偏移量；

- ***zllen***，记录压缩列表包含的节点数量；

- ***zlend***，标记压缩列表的结束点，固定值 0xFF（十进制255）。

  每一个entry又包含：

  - ***prevlen***，记录了「前一个节点」的长度，目的是为了实现从后向前遍历；
  - ***encoding***，记录了当前节点实际数据的「类型和长度」，类型主要有两种：字符串和整数。
  - ***data***，记录了当前节点的实际数据，类型和长度都由 `encoding` 决定；

- 如果**前一个节点的长度小于 254 字节**，那么 prevlen 属性需要用 **1 字节的空间**来保存这个长度值；
- 如果**前一个节点的长度大于等于 254 字节**，那么 prevlen 属性需要用 **5 字节的空间**来保存这个长度值

**会发生连锁更新问题，如果一串250-253字节的entry，头插了一个255字节以上的entry，那么每个entry的Prevlen信息都会加4，导致像多米诺骨牌一洋造成连锁反应。**



### 5.QuickList双端链表（List的主要结构）

是一个结点为ZipList的双端链表

节点采用ZipList，解决传统链表内存占用问题

控制ZipList大小，解决ZipList要申请太大连续内存的问题

中间节点可以压缩，节省空间





### 6.SkipList跳表

是一个双向链表，有score和ele字符串按照score升序，相同则按ele的字典序

每个节点包含多层指针，是1-32的随机数

不同层指针跨度不同

增删改查效率和红黑树基本一致，实现更简单



### 7.RedisObject



### 不同数据类型对于的编码格式

##### String：

![image-20240812224907039](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240812224907039.png)

当SDS对象小于64字节会转换成连续空间存储EMBSTR

RaW：用一个ptr指向SDS对象，这个SDS+RedisObject会超过64字节

INT：如果字符串是整形，则也是直接连续空间存。



**List：**

底层使用QuickList

![image-20240812232230292](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240812232230292.png)



**Set**

底层使用Dict或者Intset

![image-20240812233620182](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240812233620182.png)

当所有数字都是整数而且数量不超过设定值，就会采用Intset，否则用DIct，dict的key用来存储具体字符串，value统一是null



**ZSet**：dict加跳表

![image-20240813001254730](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240813001254730.png)

还有第二种编码：当元素数量小于128而且都小于64字节，则改用ZipList。**小数据会采用这种ZipList编码**

ZipList本身没有排序功能，也没有键值对，因此需要他自己通过业务逻辑实现插入时排序，并且两个紧挨着的Entry一个作为Score一个作为Element





**Hash**

跟Zset差不多，只是去掉了跳表

![image-20240813002220287](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240813002220287.png)

---

### IO模型

首先我们得了解，内存空间分为用户空间和内存空间，两个内存空间由各自的缓冲区，只有内核空间才能调用系统指令操作硬盘，当我们读数据的时候，我们需要等待硬盘将数据传送到内核缓冲区，**这样才称之为数据就绪**。数据就绪之后还要拷贝到用户空间，再处理数据。

**1.阻塞式IO**

等待数据就绪和等内核拷贝到用户空间这两个阶段都是阻塞的，效率低下

**2.非阻塞IO**

它在第一个阶段等待数据时不阻塞，而是返回失败数据并且循环重复问，第二阶段依旧阻塞。**其实没什么用，反而还增加了复杂度**

**3.IO多路复用**

 ![image-20240813125625895](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240813125625895.png)

**select**

![image-20240813130454317](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240813130454317.png)

**POLL**

![image-20240813130935346](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240813130935346.png)

EPOLL

![image-20240813132252933](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240813132252933.png)

![image-20240813133543648](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240813133543648.png)

![image-20240813162433470](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240813162433470.png)

### 过期key删除策略

![image-20240813163916311](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240813163916311.png)