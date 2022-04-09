## 学习指南
https://jishuin.proginn.com/p/763bfbd57a6d
https://zhuanlan.zhihu.com/p/91539644
https://blog.csdn.net/ThinkWon/article/details/103522351
https://segmentfault.com/a/1190000038748352
https://juejin.cn/post/6844904110840348680

## 缓存相关
https://segmentfault.com/a/1190000015146983

## 了解redis

## 数据结构
Redis主要有5种数据类型，包括String，List，Set，Zset，Hash，满足大部分的使用要求
https://blog.csdn.net/a745233700/article/details/113449889
### redisObject
https://www.jianshu.com/p/6390aa7a33cb
https://xindoo.blog.csdn.net/article/details/112449822
#### 数据结构与编码
- string
OBJ_ENCODING_RAW
OBJ_ENCODING_INT
OBJ_ENCODING_EMBSTR
- hash
OBJ_ENCODING_ZIPLIST
OBJ_ENCODING_HT
- list
Redis3.2之前的底层实现方式：OBJ_ENCODING_ZIPLIST 或者 双向循环链表linkedlist
Redis3.2及之后的底层实现方式：OBJ_ENCODING_QUICKLIST
- set
OBJ_ENCODING_INTSET
OBJ_ENCODING_HT
- zset
OBJ_ENCODING_ZIPLIST
OBJ_ENCODING_HT + OBJ_ENCODING_SKIPLIST

### 字符串编码
https://www.cnblogs.com/chenchuxin/p/14204452.html
http://zhangtielei.com/posts/blog-redis-robj.html

### sds
https://www.jianshu.com/p/25739b690cf6
https://developer.huawei.com/consumer/cn/forum/topic/0203469526305220198
源码
https://juejin.cn/post/6844903986584092686（讲得很一般）
- sds加长
空间预分配
- sds缩短
空间惰性释放
有三个函数:sdsclear、sdstrim、sdsrange，他们都不会改变alloc的大小即不会释放任何内存，这就是sds字符串内存管理的一种方式：惰性释放。额外调用sdsRemoveFreeSpace释放内存，这样就节省了每次sds缩减长度而导致的内存释放开销。sdsRemoveFreeSpace这个函数压缩内存，让alloc=len。


sds删除剩余空间
```
/* Reallocate the sds string so that it has no free space at the end. The
 * contained string remains not altered, but next concatenation operations
 * will require a reallocation.
 *
 * After the call, the passed sds string is no longer valid and all the
 * references must be substituted with the new pointer returned by the call. */
sds sdsRemoveFreeSpace(sds s) {
    void *sh, *newsh;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen, oldhdrlen = sdsHdrSize(oldtype);
    size_t len = sdslen(s);
    size_t avail = sdsavail(s);
    sh = (char*)s-oldhdrlen;

    /* Return ASAP if there is no space left. */
    if (avail == 0) return s;

    /* Check what would be the minimum SDS header that is just good enough to
     * fit this string. */
    type = sdsReqType(len);
    hdrlen = sdsHdrSize(type);

    /* If the type is the same, or at least a large enough type is still
     * required, we just realloc(), letting the allocator to do the copy
     * only if really needed. Otherwise if the change is huge, we manually
     * reallocate the string to use the different header type. */
    if (oldtype==type || type > SDS_TYPE_8) {
        newsh = s_realloc(sh, oldhdrlen+len+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+oldhdrlen;
    } else {
        newsh = s_malloc(hdrlen+len+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, len);
    return s;
}
```
![image.png](https://upload-images.jianshu.io/upload_images/24337324-6fe5043413a4cab8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 跳跃表
https://juejin.cn/post/6844904004498128903
skiplist与平衡树、哈希表的比较
- skiplist和各种平衡树（如AVL、红黑树等）的元素是有序排列的，而哈希表不是有序的。因此，在哈希表上只能做单个key的查找，不适宜做范围查找。所谓范围查找，指的是查找那些大小在指定的两个值之间的所有节点。
- 在做范围查找的时候，平衡树比skiplist操作要复杂。在平衡树上，我们找到指定范围的小值之后，还需要以中序遍历的顺序继续寻找其它不超过大值的节点。如果不对平衡树进行一定的改造，这里的中序遍历并不容易实现。而在skiplist上进行范围查找就非常简单，只需要在找到小值之后，对第1层链表进行若干步的遍历就可以实现。
平衡树的插入和删除操作可能引发子树的调整，逻辑复杂，而skiplist的插入和删除只需要修改相邻节点的指针，操作简单又快速。
- 从内存占用上来说，skiplist比平衡树更灵活一些。一般来说，平衡树每个节点包含2个指针（分别指向左右子树），而skiplist每个节点包含的指针数目平均为1/(1-p)，具体取决于参数p的大小。如果像Redis里的实现一样，取p=1/4，那么平均每个节点包含1.33个指针，比平衡树更有优势。
- 查找单个key，skiplist和平衡树的时间复杂度都为O(log n)，大体相当；而哈希表在保持较低的哈希值冲突概率的前提下，查找时间复杂度接近O(1)，性能更高一些。所以我们平常使用的各种Map或dictionary结构，大都是基于哈希表实现的。
- 从算法实现难度上来比较，skiplist比平衡树要简单得多。

### 快速列表
https://juejin.cn/post/6844904020616675335

### intset
http://zhangtielei.com/posts/blog-redis-intset.html
intset与ziplist相比：
- ziplist可以存储任意二进制串，而intset只能存储整数。
- ziplist是无序的，而intset是从小到大有序的。因此，在ziplist上查找只能遍历，而在intset上可以进行二分查找，性能更高。
- ziplist可以对每个数据项进行不同的变长编码，而intset只能整体使用一个统一的编码。
#### 大小端
https://www.php.cn/faq/465632.html

###  字典
hash冲突
渐进式hash（增删改查，移动一步。定时任务也会进行1ms的迁移）
![image.png](https://upload-images.jianshu.io/upload_images/24337324-7e6421fd738d6d66.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

扩容：负载因子1、5（aof、rdb，写时复制避免内存过多变化）
收缩：负载因子0.1、如果正在进行rdb或aof不允许进行收缩操作（定时任务）
https://juejin.cn/post/6844903991290101773

### 压缩列表
普通双向链表：内存单独分配，容易造成空间碎片
压缩链表：多个元素内存统一分配，不容易造成空间碎片

级联更新问题（小概率）
元素太大，元素数量太多，修改时，空间分配，元素拷贝效率低。
https://juejin.cn/post/6844904012819464206
前一个节点长度 + 当前节点长度 + 编码 + 内容

## 内存管理
https://juejin.cn/post/6844903982628864007
### 内存优化
https://www.jianshu.com/p/d39f087850c9 赞赞赞（多看几遍）
https://www.modb.pro/db/46670 TODO
https://www.dazhuanlan.com/toforever/topics/981864 TODO

### 内存淘汰源码
当前使用内存 - aof、主从缓冲区 > maxmemory
每个命令都会检查
采样（不精确）
LRU（now - lru）、LFU（255 - （now - lru前16位计算衰减计数)）、TTL（now - ttl）计算空闲时间放到淘汰池里面，取淘汰池最后一个数据bestKey。释放bestKey空间。
https://cloud.tencent.com/developer/article/1794482

### serverCron
https://www.cnblogs.com/flypighhblog/p/7757996.html
https://zhuanlan.zhihu.com/p/353548838
https://blog.csdn.net/mytt_10566/article/details/98476617


### 异步线程background io
https://blog.csdn.net/qq_35542218/article/details/107508872
bio_close_file用于关闭文件、bio_aof_fsync用于文件在os buffer缓冲区的刷盘、bio_lazy_free异步删除key。

## 功能
### 事务(multi.c)
https://juejin.cn/post/6855129005827227661
### 分布式锁
https://xie.infoq.cn/article/556aaceb68789b9de4807f1c2
①nx   表示互斥性
②ex   防止死锁
------  ①与②需要保证原子操作。不然会有死锁问题
③value 用户身份识别，防止误删
④del     用来解锁
------  ③与④需要保证原子操作。不然会有误删问题

> 另外需要考虑容错性，redis只支持异步复制，主节点挂了，切到从节点，锁还没同步过去怎么办？
①redLock：客户端多个节点都要获得锁，需要更多redis实例，从客户端代码实现，运维成本来看并不是一个很好的方案。
②zookeeper：保证强一致性。

#### RedLock
https://www.cnblogs.com/rgcLOVEyaya/p/RGC_LOVE_YAYA_1003days.html

#### Redission
https://zhuanlan.zhihu.com/p/135864820 非常赞
- 加锁
①基于hash结构实现，key=锁标志，hash_key=身份标识（uuid+线程id）,hash_value=计数
②支持可重入锁，锁存在，可以判断当前是不是本人加锁，如果是递增计数
③订阅锁释放事件，避免无效等待、申请操作
④过期时间续约机制，watchdog针对没有显示指定过期时间的锁，每10秒钟检查一次，如果锁存在就重置 过期时间为30秒。参考：https://cloud.tencent.com/developer/article/1844942

- 解锁
①判断锁是不是我加的，如果是，则递减hash_value计数值
②如果hash_value==0，删除key，并且发布锁释放事件
③watchDog删除相应的锁信息

### 消息队列
https://zhuanlan.zhihu.com/p/344269737
单播；
不可靠，无ack。
#### 延迟队列
https://juejin.cn/post/6845166890877190157
zset；
#### 发布订阅
https://redisbook.readthedocs.io/en/latest/feature/pubsub.html
https://xie.infoq.cn/article/34a190ee13a3436ae20759723
频道/模式（两种机制）
多播；
无持久化机制，不存储历史数据，消费者或redis下线都会导致消息丢失。
不可靠，无ack。

#### Stream
https://www.cnblogs.com/wmyskxz/p/12499532.html
https://zhuanlan.zhihu.com/p/60501638
支持持久化；
消费组；
历史数据；
消费进度last_delivered_id ；
ack机制pending_ids（PEL存储待ack消息） 。

### 位图（参考算法部分）
统计一整年某个人签到情况，365 * 4B
如果用位图 356 * 4 / 32B
### HyperLogLog（暂时放过）
https://www.modb.pro/db/43389
### 布隆过滤器（参考算法部分）
https://cloud.tencent.com/developer/article/1812826
L：限制位图的长度，越长错误率越低，但越占用空间
K：hash函数的数量，越多错误率越低，但约耗性能
只有不存在的情况才有 错误率

场景：是否存在，去重
防止缓存穿透。
简单理解为：高级位图
### 限流
https://blog.csdn.net/lmx125254/article/details/90700118
https://segmentfault.com/a/1190000040570911
http://dockone.io/article/10137
注意：漏桶算法，市面上给的代码都有问题（实现类似令牌桶）。
### GeoHash
https://segmentfault.com/a/1190000038529554
### scan
https://blog.csdn.net/u014439693/article/details/108325632
http://www.langdebuqing.com/redis%20notebook/redis%E6%BA%90%E7%A0%81%E9%9A%BE%E7%82%B9%EF%BC%9A%E5%AD%97%E5%85%B8%E7%9A%84%E9%81%8D%E5%8E%86dictScan.html
#### 为什么是高进位？
- redis字典扩容或者缩容规律
字典的容量=2^n； sizemake=2^n - 1，二进制表示... 111111；所以这个时候hash取模是可以用&操作的。
扩容（重复遍历）：插槽位的高位加k位；
缩容（漏掉数据）：插槽位的高位减k位；
redis字典扩容或者缩容规律就是：发生变化的是高位。

高进位：拆分两个子数组，双向从高往低去做推进。

> ①redis扩容规律，先讲下低进位为啥有问题。②画图，先表述高进位的机制
③高进位如何解决扩容、缩容问题

### 管道
https://www.jianshu.com/p/007509df7bd5
https://juejin.cn/post/6844904127001001991 主要看这篇
https://juejin.cn/post/6904433426560974856
https://www.cnblogs.com/jabnih/p/7157921.html

## 集群
### 主从复制
https://www.cnblogs.com/kismetv/p/9236731.html
https://www.cnblogs.com/wangcuican/p/12915856.html 看这篇比较简洁
### 哨兵
https://www.cnblogs.com/gunduzi/p/13160448.html
https://blog.csdn.net/a745233700/article/details/112451629 这篇可以
https://segmentfault.com/a/1190000039766545

- 三个通信；
- 主观下线
- 客观下线
- 哨兵leader选举
- 故障转移

> 哨兵之前通信pub/sub：任意一redis节点__sentinel__:hello频道

> 脑裂问题：
min-slavers-to-write 1 比如三台
min-slavers-max-lag 10

#### 客户端主从切换状态感知
https://blog.csdn.net/wrongyao/article/details/104956538
https://jishuin.proginn.com/p/763bfbd58538（有些讲得很有问题）

### redis cluster
https://blog.csdn.net/a745233700/article/details/112691126
https://blog.51cto.com/u_14279308/2484807   写的一般
https://www.cnblogs.com/detectiveHLH/p/14154665.html 写的一般
gossip协议 TODO

- move：会更新客户端hash槽缓存
- ask：不会更新
> 迁移中状态，客户端先向目标节点发送asking，再发送操作命令的原因？
避免循环重定向。如果目标节点没有找到对应数据，则会返回MOVE +源节点。asking相当于强制在目标节点执行。

### codis
https://my.oschina.net/fileoptions/blog/3007972

### 集群方案对比
https://jishuin.proginn.com/p/763bfbd5e686

## 持久化
https://segmentfault.com/a/1190000039208726
### 源码分析 TODO

## 事件循环
https://www.cnblogs.com/shijingxiang/articles/13112275.html
https://zhuanlan.zhihu.com/p/92739237
https://goalong.github.io/2019/04/13/Redis%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF/
https://blog.csdn.net/j00105840/article/details/108183316
### redis 与 IO多路复用
https://jishuin.proginn.com/p/763bfbd5f0b1

## 通信协议
https://zhuanlan.zhihu.com/p/345327284

## 监控

## 常见问题
### 使用问题
https://blog.csdn.net/ailiandeziwei/article/details/104347554

## jedis、lettuce、 redisTemplate、 redission
https://blog.csdn.net/CSDN2497242041/article/details/102675435
https://www.cnblogs.com/54chensongxia/p/13815761.html


## 超卖问题
https://cloud.tencent.com/developer/article/1399872
https://blog.csdn.net/weixin_39524247/article/details/110984749
https://blog.csdn.net/weixin_45480785/article/details/118224842

- redis + 内存标记？
一旦库存扣完，就不会再访问redis。缺点就是没法解决凸峰问题。比如当前redis库存只有1个，现在有1w个并发过来了。当然有更好的方案，可以参考12306的做法。

- 为什么不建议使用redis decr先扣减库存，而是采用lua先判断库存情况，才decr扣减？
从技术角度来看，两种方式都能保证并发安全性。但是从业务角度上看先decr会造成库存变成负数，如果后面加库存那就有问题了。

### 12306高性能
https://blog.csdn.net/g6U8W7p06dCO99fQ3/article/details/120792793
引入本地库存（还要加上冗余库存）、远程库存（redis）的思考？
一般来说不需要本地库存，所有库存都放到远程库存上。
这种方式的一个致命缺点：如果假设现在库存=1，有1w个并发过来，redis就要承担这1w并发压力。
对于本地库存+远程库存的好处：如果假设现在库存=1，有10w个并发过来，在服务层面就可以先控制一波流量，比如服务有100台，10w并发负载均衡每台是100个并发，那在不考虑冗余库存的情况下此时只有一台在提供服务接收100个并发，这个100个并发最终只会1个请求到redis
这一层。
所以根据个人理解，本地库存实际上是起到流控作用，避免所有流量都转到redis。

