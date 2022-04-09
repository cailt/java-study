## 结构介绍
https://xie.infoq.cn/article/ed531f74ecfd44eacb1a98258 赞赞赞

## 后台线程
### Master Thread
#### 作用
- 脏页的刷新；（1.2版本以后，分离到Page Cleaner Thread）
- 合并插入缓冲；
- undo页的回收（1.1版本以后，分离到Purge Thread）；
- 重做日志刷新；
#### 流程
 - idle（空闲）：
     - 重做日志缓冲写入磁盘（总是）;
     - 合并变更缓冲（总是），5% * innodb_io_capacity;
- active（活跃）：
     - 重做日志缓冲写入磁盘（总是）;
     - 合并变更缓冲（可能），需要评估当前IO负载，负载低则会触发合并变更缓冲5% * innodb_io_capacity;

> 注意：InnoDB提供了innodb_io_capacity配置参数，表示磁盘IO的吞吐量，默认是200。用于控制刷新到磁盘页的数量，因此如果用户使用了固态硬盘（SSD）等高速磁盘，完全可以调高innodb_io_capacity，直到符合磁盘IO的吞吐量。同理，innodb_purge_threads参数可以控制会瘦undo页的数量，默认20。


### IO Thread
在InnoDB存储引擎中大量使用了AIO来处理IO请求，这样可以极大提高数据库性能。而IO Thread主要负责这些IO请求的回调处理。IO Thread分为：

- write Thread：负责用户写入操作，默认有4个；可以通过innodb_write_io_threads参数进行配置；
- read Thread：负责用户的读取操作，默认有4个；可以通过innodb_read_io_threads参数进行配置；
- insert buffer Thread：负责插入缓冲区的合并；
- log Thread：负责将redo log刷新到磁盘；

### Purge Thread
事务被提交后，其所使用的undo log可能不再需要了，因此需要Purge Thread来回收已经使用并分配的undo页。默认有4个，可以通过innodb_purge_threads进行配置；

### Page Cleaner Thread
主要用于脏页的刷新，主要目的是为了减轻Master Thread的工作；


## 缓冲池
缓冲池，是主存中InnoDB缓存被访问的表和索引数据的区域。缓冲池允许直接从内存中处理经常使用的数据，从而加快处理速度。
- 流程：
   - 读取操作：首先从磁盘上读取到的页放在缓冲池中，下一次读取相同的页时，先判断该页是否在缓冲池中，若在，则称作页在缓冲池被命中，否则需要去磁盘上读取；
   - 修改操作：首先修改在缓冲池中的页（修改过的页，称为脏页），然后通过Checkpoint机制，按照一定的频率刷新到磁盘上。

- 参数：innodb_buffer_pool_size配置缓冲池的大小（默认8M），innodb_buffer_pool_instances配置缓冲池的数量（默认1个），innodb_old_blocks_pct配置LRU old子列表占比（默认3/8），innodb_old_blocks_time加入热点页的延迟时间（默认1000ms）。
- 数据对象：
![image.png](https://upload-images.jianshu.io/upload_images/24337324-b442628f43d44a6b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 内存管理
   - LRU List：
     中点插入策略将LRU划分为两个子列表：
     - New Sublist：在midpoint前面，new子列表，即热点页，5/8；
     - Old Sublist：在midpoint后面，old子列表， 3/8。
    假设用户读取某个缓冲池中没有的页。先去磁盘读取页到缓冲池；
该页对LRU来说是一个新的页，因此会把它加到midpoint，也就是old子列表的首部；该页在old子列表停留innodb_old_blocks_time后被访问了，LRU算法会认为它是热点页，将它加到new子列表的首部，这个操作被称为page made young；

   - Free List：当需要从缓冲池中分配页时，首先从Free List中查找是否有空闲页，如果有则从Free List移到LRU 列表中；否则，根据LRU算法，淘汰末尾的页，腾出来给新的页用。缓冲池页的大小默认是16K，对于非16K(原本16K的页压缩为1K,2K,4K,8K)，是通过unzip_LRU列表进行管理。unzip_LRU管理方式如下：假设需要申请一个4K的页。
      - 检查unzip_LRU列表是否有空闲的4K页，有则直接使用；如果没有，则往下走；
      - 检查是否有空闲的8K页，有则切割成两个4K页，一个直接使用，一个存放到unzip_LRU列表；如果没有，则往下走；
      - 从LRU列表申请一个16K的空闲页，切割成一个8K，两个4K，分别存放到unzip_LRU列表中

   - Flush List：管理脏页列表，负责将脏页刷回磁盘。当然LRU列表也会存储在脏页，保证缓冲池页的可用性，两者并不冲突。

### Change Buffer
https://zhuanlan.zhihu.com/p/86085906
https://www.leviathan.vip/2021/04/06/innodb-change-buffer/
使用change buffer来保存对非唯一索引的插入/更新，从而减少了IO次数，提高了性能。
- 条件
   - 用户设置了选项innodb_change_buffering
   - 非唯一的辅助索引,对于唯一辅助索引(unique key)可以缓存删除操作
   - 叶子节点 
   - 可用空间充足并且记录数>1
   - 表上没有flush操作
- 配置参数
   - innodb_change_buffering：inserts、deletes、purges、changes、all、none
   - innodb_change_buffer_max_size：默认25，最大不超过50，单位百分比。
- 原理
   对于符合上述条件的辅助索引的插入和更新时，不是每一次直接插入到索引页，而是先判断目标索引页是否在缓冲池，如果存在直接操作，否则，先放到Change Buffer。等到一定的时机，将Change Buffer与辅助索引页进行合并。对于写多的场景，能讲多个操作合并到一个操作中，从而减少了IO次数，提高了性能。
   Change Buffer数据结构是一个颗B+Tree：
   > 非叶子结点：space_id（表空间id），marker（新旧版本标识），page_no（目标页的偏移量）；
      叶子节点：space_id（表空间id），marker（新旧版本标识），page_no（目标页的偏移量），metadata（操作顺序，操作类型，row format），辅助索引记录。

  Change Buffer合并操作触发的时机
   > ①辅助索引页被读取到缓冲池：当辅助索引页被读取到缓冲池时，需要检查Insert Buffer Bitmap页IBUF_BITMAP_BUFFERED==1？如果是，则触发合并操作；
     ②Insert Buffer Bitmap页追踪到改辅助索引页无可用空间时：插入辅助索引页时，检测到此次插入后的辅助索引页大小会<1/32，就会强制触发合并操作；
     ③类型为IBUF_OP_DELETE的操作缓存时，会预估在apply完该page上所有的ibuf entry后还剩下多少记（ibuf_get_volume_buffered），如果只剩下一条记录，则拒绝本次purge操作缓存，强制触发合并操作；
     ④Master Thread：每秒或每10秒可能触发一次合并操作，需要评估IO负载；Master Thread会随机选择Change Buffer B+ Tree的一个节点，读取从该节点开始到之后所需要的节点数量。
     ⑤在 Recover 阶段会对 ibuf Page 的 Records 和数据 Page 进行 merge。
    ⑥当执行 slow shutdown 时，会强制做一次全部的ibuf merge。
- 适合场景
   - 数据库大部分是非唯一索引
   - 业务是写多读少
   - 写入数据之后并不会立即读取它
- 疑问解答
   - Change Buffer B+Tree索引key（space_id, page_no），page_no是怎么计算来的？
     > 首先，需要明确一点，Change Buffer对于辅助索引并不是只是缓存这个操作，还需要检索辅助索引上的B+Tree，找到对应叶子节点的父节点，也就是说它还是进行了多次的IO，但是不需要加载叶子节点，这一步的IO是可以优化的。因此可以通过父节点找到对应叶子节点的page_no。
   - 所有的更新或删除操作都会使用Change Buffer么？
     > 如果用辅助索引key作为更新或者操作条件，必然需要先索引，再回表才能计算出返回行数，这种情况无法使用Change Buffer；如果更新或者删除操作不是没有改到辅助索引列，也不需要使用Change Buffer。
  - 为啥要跟踪索引页的可用空间不足或者只剩下一条记录时，触发Change Buffer merge操作？
    > 索引页的空间被填满（索引页分裂），索引页内只剩下一条记录（索引页合并）。Change Buffer B+Tree存放操作的索引页号，如果磁盘索引页分裂或者合并了，此时会导致Change Buffer B+Tree存放操作的索引页号与磁盘索引页对应不上，导致Change Buffer merge开销变大。
换句话说：当可用空间不足或者只剩下一条记录时，是不能使用Change Buffer。
可用空间不足：使用Insert Buffer Map去跟踪每个索引页的可用空间大小；
只剩下一条记录：在准备插入类型为IBUF_OP_DELETE的操作缓存时，会预估在apply完该page上所有的ibuf entry后还剩下多少记（ibuf_get_volume_buffered），如果只剩下一条记录，则拒绝本次purge操作缓存，改走正常的读入物理页逻辑。
  - 为什么一定要非唯一索引呢？
    > 唯一，就需要唯一性校验，必须读取磁盘索引页。
  - 为什么Change Buffer要使用B+Tree作为数据结构存储呢？
    > BTree或者B+Tree的，本质上就是为了减少IO次数。Change Buffer不仅仅在缓冲池，它也在系统表空间进行持久化存储。基于B+Tree数据结构，Change Buffer可以存储大量的操作。
   

### doublewrite
使用doublewrite来保证数据页的可靠性。
- 问题：数据库宕机，某个页正在写入，造成部分写失效，并且此时页损坏了，无法通过redo log进行恢复（记录的是对页的物理操作）？
- 原理：
   doublewrite由两部分组成：内存中的doublewrite，磁盘共享表空间连续的128个页，大小2M。对缓冲池脏页进行刷新时，并不直接刷新等到磁盘，而是memcpy到内存中的doublewrite，之后分两次，每次1M写入到共享表空间的磁盘及目标表磁盘，然后马上调用fsync，同步磁盘。
   因此，当数据库宕机重新恢复时，如果此时写入失效（页损坏），先通过共享表空间的副本还原该页，再进行redo log重做。当然有些文件系统本身就提供了部分写失效的防范机制，在这种情况下就不需要启用doublewrite了。
- 配置参数：innodb_doublewrite （默认开启）
- 疑问解答：
   - 为什么log write不需要doublewrite的支持？
   > 因为redolog写入的单位就是512字节，也就是磁盘IO的最小单位，所以无所谓数据损坏。

### 自适应hash索引
Innodb存储引擎会监控对表上二级索引的查找，如果发现某二级索引被频繁访问，二级索引成为热数据，建立哈希索引可以带来速度的提升。

条件：二级索引页在相同的where条件下访问了N次，其中N=页中记录 * 1/16

特点
- 哈希索引，查询消耗 O(1)
- 降低对二级索引树的频繁访问资源
- 自适应

缺点
- hash自适应索引会占用innodb buffer pool；
- 自适应hash索引只适合搜索等值的查询，如select * from table where index_col='xxx'，而对于其他查找类型，如范围查找，是不能使用的；
- 无法用于排序。

由于innodb不支持hash索引，但是在某些情况下hash索引的效率很高，于是出现了adaptive hash index功能，但是通过状态监控，可以计算其收益以及付出，控制该功能开启与否。

参数：innodb_adaptive_hash_index，默认开启
　
### AIO
为了提高磁盘操作性能，当前的数据库系统都采用异步IO（Asynchronous IO，AIO）的方式来处理磁盘操作。
- Sync IO：每进行一次IO操作，需要等待此操作结束才能继续接下来的操作；
- AIO：用户可以在发出一个IO请求后立即再发出另一个IO请求，当全部IO请求发送完毕后，等待所有IO操作的完成。

AIO另一个优势是进行IO Merge操作，也就是将多个IO（连续的页）合并为1个IO,这样可以提高IOPS的性能。

在InnoDB1.1.x之前，AIO的实现是通过InnoDB存储引擎中的代码来模拟实现。从InnoDB1.1.x开始，提供了内核级别AIO的支持，称为Native AIO。因此在编译或者运行该版本的MySql时，需要libaio库的支持。

- 配置参数：innodb_use_native_aio控制是否启动Native AIO，在Linux系统下，默认值为ON。

### Checkpoint技术
#### 背景：
- 页每一次发生变化，就将页刷新到磁盘，开销非常大，innodb会缓存脏页，但是当缓冲池内存不够时，怎么办？
- 将页刷新到磁盘时发生宕机，数据无法恢复，为了解决这个问题，当前事务数据库普遍采用Write Ahead Log（日志先行）策略，innodb的redo log采用循环使用方式，因此当redo log不可用时怎么办？
- redo log很大，数据库恢复时间长怎么办？

Checkpoint技术就是为了解决以上几个问题，作用：
- 缓冲池不够用，将脏页刷回磁盘；
> 缓冲池不够用时，根据LRU算法会淘汰掉最近最少使用的页，如果该页是脏页的话，会强制执行CheckPoint，将该脏页刷回磁盘；
- redo log不可用，刷新磁盘；
> 如果重做日志经过一圈写到当前Checkpoint的位置，导致redo log不可用，将强制执行CheckPoint，将缓冲池的页至少刷新到当前重做日志的位置。
- 缩短数据库恢复时间。
> 当数据库发生宕机时，不需要重放整个redo log，只需要重放Checkpoint之后的redo log，这样就可以大大缩短恢复时间；


#### 分类
- Sharp Checkpoint：发生在数据库关闭时将所有的脏页都刷新回磁盘，如果此时数据库脏页非常多，则会很大影响性能；

- Fuzzy Checkpoint：模糊检查点，默认方式，只刷新一部分脏页，不是刷新所有脏页；主要有以下几种情况：

   - Master Thread Checkpoint：每秒或者10秒的频率异步刷新缓冲池的脏页到磁盘。（由Page Cleaner Thread完成）
   - FLUSH_LRU_LIST Checkpoint：缓冲池不够用时，根据LRU算法会淘汰掉最近最少使用的页，如果该页是脏页的话，会强制执行CheckPoint，将该脏页刷回磁盘（由Page Cleaner Thread完成）；
   - Dirty Page too much Checkpoint：即脏页数量太多，导致强制进行Checkpoint。由参数`innodb_max_dirty_pages_pt`来控制，默认75（即75%）。当脏页数量占据75%缓冲池时，刷新一部分脏页到磁盘。（由Page Cleaner Thread完成）
   - Async/Sync Flush Checkpoint：重做日志不可用的情况，需要强制从脏页列表中选取一些脏页刷新磁盘到缓存（由Page Cleaner Thread完成）。
> 假设将已经写到redo log的LSN记为redo_lsn，将已经刷新回磁盘最后一个脏页的LSN记为checkpoint_lsn（严格讲是最后一个脏页的第一次修改时LSN），则可定义：
checkpoint_age = redo_lsn - checkpoint_lsn;
async_water_mark = 75% * total_redo_log_file_size; < checkpoint_age 则Async Flush 
sync_water_mark = 90% * total_redo_log_file_size; < checkpoint_age 则Sync Flush 

#### LSN
LSN(Log Sequence Number)用来标记版本，8个字节的数字。重做日志每增加多少个字节，LSN就递增多少。存放位置：

- redo log：每次事务提交，先在log buffer生成LSN，然后刷新到磁盘的redo log，递增LSN；
- 缓冲池中的页：保存两个LSN，第一次修改时LSN，最后一次修改时LSN；每次修改页，递增最后一次修改时LSN；
- Checkpoint：每次将页刷新到磁盘，保存该页的第一次修改时LSN；
因此，数据库恢复时，只需要重放redo log的LSN - Checkpoint的LSN之间的操作。

LSN可以获取如下等几个信息：

- 数据页的版本信息
- 日志的总量
- CheckPoint的位置

LSN四个重要指标：
- Log sequence number：最新的LSN，存放在Log Buffer;
- Log flushed up to：redo log上面的LSN，从Log Buffer刷新到redo log；
- Pages flushed up to：最后一个脏页被刷回磁盘的第一次修改时LSN；
- Last checkpoint at：checkpoint LSN，即最后一个脏页被刷回磁盘的第一次修改时LSN；

四者之间的关系：Log sequence number >= Log flushed up to >= Pages flushed up to >= Last checkpoint at。

Last checkpoint at的lsn比上面三个少9的原因：checkpoint LSN自身也要存储在redo log，占用9个字节。

## redo log
https://www.cnblogs.com/WangXianSCU/p/15061753.html
https://segmentfault.com/a/1190000017888478
https://jimmy2angel.github.io/2019/05/07/InnoDB-redo-log/ 赞赞赞
### redo log并发写入
http://mysql.taobao.org/monthly/2019/03/03/
### redo log记录undo log
https://www.zhihu.com/question/338308334
### 为什么需要redo log
为了取得更好的读写性能，InnoDB会将数据缓存在内存中（InnoDB Buffer Pool），对磁盘数据的修改也会落后于内存，这时如果进程或机器崩溃，会导致内存数据丢失，为了保证数据库本身的一致性和持久性，InnoDB维护了REDO LOG。修改Page之前需要先将修改的内容记录到REDO中，并保证REDO LOG早于对应的Page落盘，也就是常说的WAL，Write Ahead Log。当故障发生导致内存数据丢失后，InnoDB会在重启时，通过重放REDO，将Page恢复到崩溃前的状态。
### redo log中记录了什么内容
到MySQL 8.0为止，已经有多达65种的REDO记录。用来记录这不同的信息，恢复时需要判断不同的REDO类型，来做对应的解析。根据REDO记录不同的作用对象，可以将这65中REDO划分为三个大类：作用于Page，作用于Space以及提供额外信息的Logic类型。如图以作用于Page的REDO为例：
![image.png](https://upload-images.jianshu.io/upload_images/24337324-adb5727815b9203d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其中，Type就是MLOG_REC_UPDATE_IN_PLACE类型，Space ID和Page Number唯一标识一个Page页，这三项是所有REDO记录都需要有的头信息，后面的是MLOG_REC_UPDATE_IN_PLACE类型独有的，其中Record Offset用给出要修改的记录在Page中的位置偏移，Update Field Count说明记录里有几个Field要修改，紧接着对每个Field给出了Field编号(Field Number)，数据长度（Field Data Length）以及数据（Filed Data）。
由于Physiological Logging的方式采用了物理Page中的逻辑记法，导致两个问题：
- 需要基于正确的Page状态上重放REDO
> 由于在一个Page内，REDO是以逻辑的方式记录了前后两次的修改，因此重放REDO必须基于正确的Page状态。然而InnoDB默认的Page大小是16KB，是大于文件系统能保证原子的4KB大小的，因此可能出现Page内容成功一半的情况。InnoDB中采用了Double Write Buffer的方式来通过写两次的方式保证恢复的时候找到一个正确的Page状态。
- 需要保证REDO重放的幂等
> InnoDB给每个REDO记录一个全局唯一递增的标号LSN(Log Sequence Number)。Page在修改时，会将对应的REDO记录的LSN记录在Page上（FIL_PAGE_LSN字段），这样恢复重放REDO时，就可以来判断跳过已经应用的REDO，从而实现重放的幂等。
### 如何安全地清除REDO
由于REDO文件空间有限，同时为了尽量减少恢复时需要重放的REDO，InnoDB引入log_checkpointer线程周期性的打Checkpoint。重启恢复的时候，只需要从最新的Checkpoint开始回放后边的REDO，因此Checkpoint之前的REDO就可以删除或被复用。
### 如何高效地写REDO
作为维护数据库正确性的重要信息，REDO日志必须在事务提交前保证落盘，否则一旦断电将会有数据丢失的可能，因此从REDO生成到最终落盘的完整过程成为数据库写入的关键路径，其效率也直接决定了数据库的写入性能。这个过程包括：
- REDO内容的产生
> 事务在写入数据的时候会产生REDO，一次原子的操作可能会包含多条REDO记录，InnoDB有一套完整的机制来保证涉及一次原子操作的多条REDO记录原子，即恢复的时候要么全部重放，要不全部不重放。InnoDB中通过min-transaction实现，简称mtr，需要原子操作时，调用mtr_start生成一个mtr，mtr中会维护一个动态增长的m_log，这是一个动态分配的内存空间，将这个原子操作需要写的所有REDO先写到这个m_log中，当原子操作结束后，调用mtr_commit将m_log中的数据拷贝到InnoDB的Log Buffer。
- REDO写入InnoDB Log Buffer
> 高并发的环境中，会同时有非常多的min-transaction(mtr)需要拷贝数据到Log Buffer，如果通过锁互斥，那么毫无疑问这里将成为明显的性能瓶颈。为此，从MySQL 8.0开始，设计了一套无锁的写log机制，其核心思路是允许不同的mtr，同时并发地写Log Buffer的不同位置
- 从InnoDB Log Buffer写入操作系统Page Cache
> 写入到Log Buffer中的REDO数据需要进一步写入操作系统的Page Cache，InnoDB中有单独的log_writer来做这件事情。
- REDO刷盘
> log_writer提升write_lsn之后会通知log_flusher线程，log_flusher线程会调用fsync将REDO刷盘，至此完成了REDO完整的写入过程
- 唤醒等待的用户线程完成Commit。
> 为了保证数据正确，只有REDO写完后事务才可以commit，因此在REDO写入的过程中，大量的用户线程会block等待，直到自己的最后一条日志结束写入。
① `innodb_flush_log_at_trx_commit = 1`（默认），事务每次提交都会将 redo log buffer 中的日志写入 os buffer 并调用 fsync() 刷到 redo log file 中。这种方式即使系统崩溃也不会丢失任何数据，但是因为每次提交都写入磁盘，IO的性能较差。
②`innodb_flush_log_at_trx_commit = 2`，每次提交都仅写入到 os buffer ，然后是每秒调用 fsync() 将 os buffer 中的日志写入到 redo log file 。
③`innodb_flush_log_at_trx_commit = 0`，事务提交时不会将 redo log buffer 中日志写入到 os buffer ，而是每秒写入 os buffer 并调用 fsync() 写入到 redo log file 中。也就是说设置为0时是(大约)每秒刷新写入到磁盘中的，当系统崩溃，会丢失1秒钟的数据。
### 什么是Mini-Transaction
Innodb引擎对底层页的一次原子访问的过程叫做Mini-Transaction。什么意思呢？我们来举一个例子，假如我们在创建有索引的数据表中插入一条记录，那么我们需要在Innodb引擎的聚簇索引B+树的数据页中插入这条记录，更改这条记录的上一条记录的next_record的内容，使其指向这条记录；然后还需要在辅助索引B+树的数据页的中插入这条记录的索引信息；上边描述的这个过程就称为对底层页的一次原子访问，也就是说这次原子访问可能修改了多个数据页的信息，这个过程是不可分割的。
参考：https://juejin.cn/post/6895265596985114638

### Mini-Transaction和redo log的关系
假设向有索引的数据表中插入一条记录会产生多条redo log，这些redo log只插入了其中一部分时，数据库宕机了。我们知道redo log是做数据页恢复用的，假如我们只恢复了一部分redo log,那么这张数据表所对应的B+树就处于不正确的状态了。Innodb引擎当然不会允许这种事情发生，所以对于一个Mini-Transactoin产生的redo log日志都会被划分到一个组当中去，在进行系统崩溃重启恢复时，`针对某个组中的redo日志`，要么把全部的日志都恢复掉，要么就一条也不恢复。

假如Innodb引擎中的一个Transaction由多条Sql语句组成，每条Sql语句又可以由多个mtr组成，每个mtr又包含一组不可分割的redo log日志，所以Transaction、Mini-Transaction、redo log之间的关系就是这样一种包含的对应关系：

![image.png](https://upload-images.jianshu.io/upload_images/24337324-cb9de20de7c639d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 数据库恢复流程
 1.当MySQL启动的时候，先会从共享表空间中读取出上次保存最大的LSN。
    2.然后调用recv_recovery_from_checkpoint_start，并将最大的LSN作为参数传入函数当中。
    3.函数会先最近建立checkpoint的日志组，并读取出对应的checkpoint信息
    4.通过`checkpoint lsn`和传入的`最大LSN`进行比较，如果相等，不进行日志恢复数据，如果不相等，进行日志恢复。
    5.在启动恢复之前，先会同步各个日志组的archive归档状态
    6.在开始恢复时，先会从日志文件中读取2M的日志数据到log_sys->buf，然后对这2M的数据进行scan,校验其合法性，而后将去掉block header的日志放入recv_sys->buf当中，这个过程称为scan,会改变scanned lsn.
    7.在对2M的日志数据scan后，innodb会对日志进行mtr操作解析，并执行相关的mtr函数。如果mtr合法，会将对应的记录数据按`space page_no`作为KEY存入recv_sys->addr_hash当中。
    8.当对scan的日志数据进行mtr解析后，innodb对会调用recv_apply_hashed_log_recs对整个recv_sys->addr_hash进行扫描，并按照日志相对应的操作进行对应page的数据恢复。这个过程会改变recovered_lsn。
    9.如果完成第8步后，会再次从日志组文件中读取2M数据，跳到步骤6继续相对应的处理，直到日志文件没有需要恢复的日志数据。
    10.innodb在恢复完成日志文件中的数据后，会调用recv_recovery_from_checkpoint_finish结束日志恢复操作，主要是释放一些开辟的内存。并进行事务和binlog的处理。 
 上面过程的示意图如下：
![image.png](https://upload-images.jianshu.io/upload_images/24337324-7001bc75d4b70804.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 崩溃恢复
http://mysql.taobao.org/monthly/2015/06/01/
https://www.leviathan.vip/2021/04/16/innodb-trx-recover/
## redo log 与 bin log
https://segmentfault.com/a/1190000023827696
### XA和组提交
http://mysql.taobao.org/monthly/2020/05/07/
https://cloud.tencent.com/developer/article/1429685

## bin log
MySQL 的 Binlog 日志是一种二进制格式的日志，Binlog 记录所有的 DDL 和 DML 语句(除了数据查询语句SELECT、SHOW等)，以 Event 的形式记录，同时记录语句执行时间。

### Binlog 的主要作用有两个：

- 数据恢复：
因为 Binlog 详细记录了所有修改数据的 SQL，当某一时刻的数据误操作而导致出问题，或者数据库宕机数据丢失，那么可以根据 Binlog 来回放历史数据。

- 主从复制
想要做多机备份的业务，可以去监听当前写库的 Binlog 日志，同步写库的所有更改。

### Binlog 包括两类文件：
二进制日志索引文件(.index)：记录所有的二进制文件。
二进制日志文件(.00000*)：记录所有 DDL 和 DML 语句事件。

### 参数
Binlog 日志功能默认是开启的，线上情况下 Binlog 日志的增长速度是很快的，在 MySQL 的配置文件 my.cnf 中提供一些参数来对 Binlog 进行设置。
```
设置此参数表示启用binlog功能，并制定二进制日志的存储目录
log-bin=/home/mysql/binlog/

#mysql-bin.*日志文件最大字节（单位：字节）
#设置最大100MB
max_binlog_size=104857600

#设置了只保留7天BINLOG（单位：天）
expire_logs_days = 7

#binlog日志只记录指定库的更新
#binlog-do-db=db_name

#binlog日志不记录指定库的更新
#binlog-ignore-db=db_name

#写缓冲多少次，刷一次磁盘，默认0。表示该操作由操作系统根据自身负载自行决定多久写一次磁盘。
#sync_binlog = 1 表示每一条事务提交都会立刻写盘。sync_binlog=n 表示 n 个事务提交才会写盘
sync_binlog=0
```

### 写 Binlog 的时机
SQL transaction 执行完，但任何相关的 Locks 还未释放或事务还未最终 commit 前。这样保证了 Binlog 记录的操作时序与数据库实际的数据变更顺序一致。

### Binlog 日志格式
针对不同的使用场景，Binlog 也提供了可定制化的服务，提供了三种模式来提供不同详细程度的日志内容。

- Statement 模式：基于 SQL 语句的复制(statement-based replication-SBR)
 保存每一条修改数据的SQL。该模式只保存一条普通的SQL语句，不涉及到执行的上下文信息。因为每台 MySQL 数据库的本地环境可能不一样，那么对于依赖到本地环境的函数或者上下文处理的逻辑 SQL 去处理的时候可能同样的语句在不同的机器上执行出来的效果不一致。

比如像 sleep()函数，last_insert_id()函数，等等，这些都跟特定时间的本地环境
- Row 模式：基于行的复制(row-based replication-RBR)
 MySQL V5.1.5 版本开始支持Row模式的 Binlog，它与 Statement 模式的区别在于它不保存具体的 SQL 语句，而是记录具体被修改的信息。比如一条 update 语句更新10条数据，如果是 Statement 模式那就保存一条 SQL 就够，但是 Row 模式会保存每一行分别更新了什么，有10条数据。Row 模式的优缺点就很明显了。保存每一个更改的详细信息必然会带来存储空间的快速膨胀，换来的是事件操作的详细记录。所以要求越高代价越高。
- Mixed 模式：混合模式复制(mixed-based replication-MBR)
  Mixed 模式即以上两种模式的综合体。既然上面两种模式分别走了极简和一丝不苟的极端，那是否可以区分使用场景的情况下将这两种模式综合起来呢？
  在 Mixed 模式中，一般的更新语句使用 Statement 模式来保存 Binlog，但是遇到一些函数操作，可能会影响数据准确性的操作则使用 Row 模式来保存。这种方式需要根据每一条具体的 SQL 语句来区分选择哪种模式。
  MySQL 从 V5.1.8 开始提供 Mixed 模式，V5.7.7 之前的版本默认是Statement 模式，之后默认使用Row模式， 但是在 8.0 以上版本已经默认使用 Mixed 模式了。

## 环形复制与双主复制缺点
双主复制本身没有冲突检测机制，所以如果两个主节点都同时更新了相同的主键，就会有冲突导致复制中断。如果两个主的修改表是错开的，读看情况，如果是实时性要求高的，那么需要读写节点一致，如果允许读延迟，那么可以任意选择一个节点。目前配置双主复制，一般都是一个主有写入。另一个主是为了在当前真主故障后切换到新主不需要配置复制关系。其实如果因为多个节点多有写入而配置双主，那么还是建议用MySQL Group Replication的多主模式。这才是长远的方案。

https://www.cnblogs.com/gaogao67/p/10931313.html
不具备冲突检测机制；
若其中有一台服务器宕机，则该宕机节点发起的事件就会绕着服务器链无限循环：https://blog.csdn.net/weixin_30271335/article/details/96554087

## 主从复制
https://www.cnblogs.com/kevingrace/p/10228694.html
Binlog 日志主要作用是数据恢复和主从复制。本身就是二进制格式的日志文件，网络传输无需进行协议转换。MySQL 集群的高可用，负载均衡，读写分离等功能都是基于Binlog 来实现的。
### 主流架构模型
- 一主一从 / 一主多从
 最常见的主从架构方式，一般实现主从配置或者读写分离都可以采用这种架构。
如果是一主多从的模式，当 Slave 增加到一定数量时，Slave 对 Master 的负载以及网络带宽都会成为一个严重的问题。
- 多主一从
 MySQL 5.7 开始支持多主一从的模式，将多个库的数据备份到一个库中存储。
- 双主复制
 理论上跟主从一样，但是两个MySQL服务器互做对方的从，任何一方有变更，都会复制对方的数据到自己的数据库。双主适用于写压力比较大的业务场景，或者 DBA 做维护需要主从切换的场景，通过双主架构避免了重复搭建从库的麻烦。（主从相互授权连接，读取对方binlog日志并更新到本地数据库的过程；只要对方数据改变，自己就跟着改变）
- 级联复制
级联模式下因为涉及到的 slave 节点很多，所以如果都连在 master 上对主服务器的压力肯定是不小的。所以部分 slave 节点连接到它上一级的从节点上。这样就缓解了主服务器的压力。
级联复制解决了一主多从场景下多个从库复制对主库的压力，带来的弊端就是数据同步延迟比较大。

### 主从复制原理
![image.png](https://upload-images.jianshu.io/upload_images/24337324-27015b9af5fceb0e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 在从节点上执行 start slave 命令开启主从复制开关，开始进行主从复制。从节点上的 I/O 进程连接主节点，并请求从指定日志文件的指定位置（或者从最开始的日志）之后的日志内容。
- 主节点接收到来自从节点的 I/O 请求后，通过负责复制的 I/O 进程（log Dump Thread, 主节点会为自己的每一个从节点创建一个 log dump 线程。）根据请求信息读取指定日志指定位置之后的日志信息，返回给从节点。返回信息中除了日志所包含的信息之外，还包括本次返回的信息的 Binlog file 以及 Binlog position（Binlog 下一个数据读取位置）。
- 从节点的 I/O 进程接收到主节点发送过来的Binlog 日志内容、日志文件及位置点后，解析到各类 Events 之后记录到 relay log （中继日志，充当缓冲区，这样 master 就不必等待 slave 执行完成才发送下一个事件）文件（Mysql-relay-bin.xxx）的最末端，并将读取到的 Binlog文件名和位置保存到master-info 文件中，以便在下一次读取的时候能够清楚的告诉 Master ：“ 我需要从哪个 Binlog 的哪个位置开始往后的日志内容，请发给我”。
- 从节点的 SQL 线程检测到relay log 中新增加了内容后，会将 relay log 的内容解析成在能够执行 SQL 语句，然后在本数据库中按照解析出来的顺序执行，并在 relay log.info 中记录当前应用中继日志的文件名和位置点。之后从库的 SQL 线程把 relay log 应用为 binlog 日志，直到主库与从库的 binlog 日志文件完全数据一致，达到主从同步。

### 主从复制的模式
- 异步模式 (async-mode)
这种模式下，主节点不会主动推送数据到从节点，主库在执行完客户端提交的事务后会立即将结果返给给客户端，并不关心从库是否已经接收并处理，这样就会有一个问题，主节点如果崩溃掉了，此时主节点上已经提交的事务可能并没有传到从节点上，如果此时，强行将从提升为主，可能导致新主节点上的数据不完整。
![image.png](https://upload-images.jianshu.io/upload_images/24337324-d9feb0481d80c23d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 半同步模式(semi-sync)
介于异步复制和全同步复制之间，主库在执行完客户端提交的事务后不是立刻返回给客户端，而是等待至少一个从库接收到并写到 relay log 中才返回成功信息给客户端（只能保证主库的 Binlog 至少传输到了一个从节点上），否则需要等待直到超时时间然后切换成异步模式再提交（主从之间网络延迟恢复正常的时候，会自动从异步复制又转为半同步复制，还是相当智能的）。
相对于异步复制，半同步复制提高了数据的安全性，一定程度的保证了数据能成功备份到从库，同时它也造成了一定程度的延迟，但是比全同步模式延迟要低，这个延迟最少是一个 TCP/IP 往返的时间。所以，半同步复制最好在低延时的网络中使用。
半同步模式不是 MySQL 内置的，从 MySQL 5.5 开始集成，需要 master 和 slave 安装插件开启半同步模式。
![image.png](https://upload-images.jianshu.io/upload_images/24337324-7a172922207d4985.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 半同步复制的潜在问题
客户端事务在存储引擎层提交后，在得到从库确认的过程中，主库宕机了，此时，可能的情况有两种：
①事务还没发送到从库上
此时，客户端会收到事务提交失败的信息，客户端会重新提交该事务到新的主上，当宕机的主库重新启动后，以从库的身份重新加入到该主从结构中，会发现，该事务在从库中被提交了两次，一次是之前作为主的时候，一次是被新主同步过来的。
②事务已经发送到从库上
此时，从库已经收到并应用了该事务，但是客户端仍然会收到事务提交失败的信息，重新提交该事务到新的主上。

- Loss-Less半同步复制（无损复制）：针对半同步复制的问题，在5.7.2引入了Loss-less Semi-Synchronous，在调用binlog sync之后，engine层commit之前等待Slave ACK。这样只有在确认Slave收到事务events后，事务才会提交。在commit之前等待Slave ACK，同时可以堆积事务，利于group commit，有利于提升性能。但也会有个问题，假设主库在存储引擎提交之前挂了，那么很明显这个事务是不成功的，但由于对应的Binlog已经做了Sync操作，从库已经收到了这些Binlog，并且执行成功，相当于在从库上多了数据，也算是有问题的，但多了数据，问题一般不算严重。这个问题可以这样理解，作为MySQL，在没办法解决分布式数据一致性问题的情况下，它能保证的是不丢数据，多了数据总比丢数据要好。
无损复制其实就是对semi sync增加了`rpl_semi_sync_master_wait_point`参数，来控制半同步模式下主库在返回给会话事务成功之前提交事务的方式。rpl_semi_sync_master_wait_point该参数有两个值：AFTER_COMMIT（5.6默认）和AFTER_SYNC（5.7新增）；
![image.png](https://upload-images.jianshu.io/upload_images/24337324-fa3b991c25bf8530.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
master将每个事务写入binlog , 传递到slave刷新到磁盘(relay log，如果sync_relay_log!= 1，可能会有数据丢失的风险，例如从库挂了，导致relay log没有来的及刷盘，重启后，过段时间主库挂了，从库升级为主库，之前没有来的及刷盘的relay log相关的数据就丢失了)。master等待slave反馈接收到relay log的ack之后，再提交事务并且返回commit OK结果给客户端。 即使主库crash，所有在主库上已经提交的事务都能保证已经同步到slave的relay log中。


- 全同步模式
指当主库执行完一个事务，然后所有的从库都复制了该事务并成功执行完才返回成功信息给客户端。因为需要等待所有从库执行完该事务才能返回成功信息，所以全同步复制的性能必然会收到严重的影响。

### 半同步复制与无损复制的对比
1) ACK的时间点不同
-  半同步复制在InnoDB层的Commit Log后等待ACK。
-  无损复制在MySQL Server层的Write binlog后等待ACK。

2) 主从数据一致性
-  半同步复制在主从切换会有`幻读`、数据`丢失`风险。
-  无损复制主从切换会有数据`变多`风险。

### 基于GTID复制模式
https://www.huaweicloud.com/articles/de38e0849972a8cc1bc4a58d17704d7d.html
注意：mysql重启会创建一个新的binlog，所以有可能会导致主从binlog文件名不一致。https://blog.csdn.net/king_kgh/article/details/74833539

GTID即全局事务ID (global transaction identifier), 其保证为每一个在主上提交的事务在复制集群中可以生成一个唯一的ID。GTID最初由google实现，官方MySQL在5.6才加入该功能。mysql主从结构在一主一从情况下对于GTID来说就没有优势了，而对于2台主以上的结构优势异常明显，可以在数据不丢失的情况下切换新主。
GTID实际上是由UUID+TID (即transactionId)组成的。其中UUID(即server_uuid) 产生于auto.conf文件(cat /data/mysql/data/auto.cnf)，是一个MySQL实例的唯一标识。TID代表了该实例上已经提交的事务数量，并且随着事务提交单调递增，所以GTID能够保证每个MySQL实例事务的执行（不会重复执行同一个事务，并且会补全没有执行的事务）。GTID在一组复制中，全局唯一。
-  GTID在binlog中的结构

![image](https://upload-images.jianshu.io/upload_images/24337324-f3d44ba6f78cbb4c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 

-  GTID event 结构

![image](https://upload-images.jianshu.io/upload_images/24337324-55ce3c2bdea5e7c2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

-  Previous_gtid_log_event
Previous_gtid_log_event 在每个binlog 头部都会有每次binlog rotate的时候存储在binlog头部Previous-GTIDs在binlog中只会存储在这台机器上执行过的所有binlog，不包括手动设置gtid_purged值。换句话说，如果你手动set global gtid_purged=xx； 那么xx是不会记录在Previous_gtid_log_event中的。

### GTID的工作原理

从服务器连接到主服务器之后，把自己执行过的GTID (Executed_Gtid_Set: 即已经执行的事务编码)<SQL线程> 、获取到的GTID (Retrieved_Gtid_Set: 即从库已经接收到主库的事务编号) <IO线程>发给主服务器，主服务器把从服务器缺少的GTID及对应的transactions发过去补全即可。当主服务器挂掉的时候，找出同步最成功的那台从服务器，直接把它提升为主即可。如果硬要指定某一台不是最新的从服务器提升为主， 先change到同步最成功的那台从服务器， 等把GTID全部补全了，就可以把它提升为主了。
- 传统复制
![image.png](https://upload-images.jianshu.io/upload_images/24337324-048bc32d1555bea0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- GTID复制
![image.png](https://upload-images.jianshu.io/upload_images/24337324-2831b240ec0e9403.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

GTID是MySQL 5.6的新特性，可简化MySQL的主从切换以及Failover。GTID用于在binlog中唯一标识一个事务。当事务提交时，MySQL Server在写binlog的时候，会先写一个特殊的Binlog Event，类型为GTID_Event，指定下一个事务的GTID，然后再写事务的Binlog。主从同步时GTID_Event和事务的Binlog都会传递到从库，从库在执行的时候也是用同样的GTID写binlog，这样主从同步以后，就可通过GTID确定从库同步到的位置了。也就是说，无论是级联情况，还是一主多从情况，都可以通过GTID自动找点儿，而无需像之前那样通过File_name和File_position找点儿了。

简而言之，GTID的工作流程为：
-  master更新数据时，会在事务前产生GTID，一同记录到binlog日志中。
-  slave端的i/o 线程将变更的binlog，写入到本地的relay log中。
-  sql线程从relay log中获取GTID，然后对比slave端的binlog是否有记录。
-  如果有记录，说明该GTID的事务已经执行，slave会忽略。
-  如果没有记录，slave就会从relay log中执行该GTID的事务，并记录到binlog。
-  在解析过程中会判断是否有主键，如果没有就用二级索引，如果没有就用全部扫描。

### GTID的优缺点

GTID的优点
-  一个事务对应一个唯一ID，一个GTID在一个服务器上只会执行一次;
-  GTID是用来代替传统复制的方法，GTID复制与普通复制模式的最大不同就是不需要指定二进制文件名和位置;
-  减少手工干预和降低服务故障时间，当主机挂了之后通过软件从众多的备机中提升一台备机为主机;

GTID复制是怎么实现自动同步，自动对应位置的呢？
比如这样一个主从架构：ServerC <-----ServerA ----> ServerB
即一个主数据库ServerA，两个从数据库ServerB和ServerC

当主机ServerA 挂了之后 ，此时ServerB执行完了所有从ServerA 传过来的事务，ServerC 延时一点。这个时候需要把 ServerB 提升为主机 ，Server C 继续为备机；当ServerC 链接ServerB 之后,首先在自己的二进制文件中找到从ServerA 传过来的最新的GTID，然后将这个GTID 发送到ServerB ,ServerB 获得这个GTID之后,就开始从这个GTID的下一个GTID开始发送事务给ServerC。这种自我寻找复制位置的模式减少事务丢失的可能性以及故障恢复的时间。

GTID的缺点(限制)
-  不支持非事务引擎;
-  不支持create table ... select 语句复制(主库直接报错);(原理: 会生成两个sql, 一个是DDL创建表SQL, 一个是insert into 插入数据的sql; 由于DDL会导致自动提交, 所以这个sql至少需要两个GTID, 但是GTID模式下, 只能给这个sql生成一个GTID)
-  不允许一个SQL同时更新一个事务引擎表和非事务引擎表;
-  在一个复制组中，必须要求统一开启GTID或者是关闭GTID;
-  开启GTID需要重启 (mysql5.7除外);
-  开启GTID后，就不再使用原来的传统复制方式;
-  对于create temporary table 和 drop temporary table语句不支持;
-  不支持sql_slave_skip_counter;


### 5.7 半同步复制性能优化
- 支持发送binlog和接受ack的异步化：
旧版本的semi sync受限于dump thread ，原因是dump thread承担了两份不同且又十分频繁的任务：传送binlog给slave ，还需要等待slave反馈信息，而且这两个任务是串行的，dump thread必须等待slave返回之后才会传送下一个events事务。dump thread已然成为整个半同步提高性能的瓶颈。在高并发业务场景下，这样的机制会影响数据库整体TPS 
> 为了解决上述问题，在5.7版本的semi sync框架中，独立出一个Ack Receiver线程 ，专门用于接收slave返回的ack请求，这将之前dump线程的发送和接受工作分为了两个线程来处理。这样master上有两个线程独立工作，可以同时发送binlog到slave，和接收slave的ack信息。因此半同步复制得到了极大的性能提升。这也是MySQL 5.7发布时号称的Faster semi-sync replication
- 控制主库接收slave写事务成功反馈数量：
MySQL 5.7新增了rpl_semi_sync_master_wait_slave_count参数，可以用来控制主库接受多少个slave写事务成功反馈，给高可用架构切换提供了灵活性。例如：当count值为2时，master需等待两个slave的ack。

- Binlog互斥锁改进：
旧版本半同步复制在主提交binlog的写会话和dump thread读binlog的操作都会对binlog添加互斥锁，导致binlog文件的读写是串行化的，存在并发度的问题。
移除了dump thread对binlog的互斥锁，加入了安全边际保证binlog的读安全。


### 问题
- Mysql主从有什么优点？为什么要选择主从？
> 负载均衡和高可用
- 若是主从复制，达到了写性能的瓶颈，你是怎么解决的呢？
> 设计上进行解决采用分库、分表
- 主从复制的过程有数据延迟怎么办？
> 对于数据的实时性要求很高，要求强一致性，可以采用同步复制策略

### MGR
https://database.51cto.com/art/202004/615706.htm
https://www.cnblogs.com/kevingrace/p/10260685.html
https://cloud.tencent.com/developer/article/1548269
### MHA
https://juejin.cn/post/6844904099075325960
#### MHA + vip漂移
https://cloud.tencent.com/developer/article/1183644
#### 配置多套
https://blog.csdn.net/weixin_31487917/article/details/113404188
### mysql高可用架构 MHA与MGR
https://www.modb.pro/db/24305
https://cloud.tencent.com/developer/article/1555359
#### MHA + ProxySQL
http://blog.itpub.net/26736162/viewspace-2762781/
#### MGR + ProxySQL
https://juejin.cn/post/6844904036718608392
#### xcom
https://zhuanlan.zhihu.com/p/149991963
#### 网易改进mgr
https://zhuanlan.zhihu.com/p/88841537
### 冲突检测
https://www.modb.pro/db/25554

## 备份与恢复
https://juejin.cn/post/6844903511201677325
### 为什么要备份
- 灾难恢复，数据库在运行过程中，终会遇到各种各样的问题: 硬件故障、Bug 导致数据损坏、由于服务器宕机或者其他原因造成的数据库不可用。除此以外还有人为操作：DELETE 语句忘加条件、ALTER TABLE 执行错表、DROP TABLE 执行错表、黑客攻击，即使这些问题你都还没遇到，但是根据墨菲定律，总会有遇上的时候。
- 回滚，由于某种Bug或系统被黑造成大量的损失，这个时候就需要回滚到某个状态。常见的有区块链交易所被黑然后回滚，游戏漏洞被利用然后整体回滚。
- 审计，有时候有这样的需求：需要知道某一个时间点的数据是怎么样的，可能是年末审计，也可能是因为官司。
- 测试，一个基本的测试需求是，定时拉取线上数据到测试环境，如果有备份，就可以非常方便地拉取数据。

### 备份方式
#### 逻辑备份
逻辑备份是最常见的方式，在数据量比较少的时候很常用。

- 逻辑备份的优势：
   - 备份恢复比较简单，例如 mysqldump 就是 MySQL 自带的备份工作，无需额外安装。恢复的时候可以直接使用 mysql 命令进行恢复。
   - 可以远程备份和恢复，也就是说，可以在其他机器执行备份命令。
   - 备份出来的数据非常直观，备份出来后，可以使用 sed grep 等工具进行数据提取或者修改。
   - 与存储引擎无关，因为备份文件是直接从 MySQL 里面提取出来的数据，所以在直观上，备份数据数据不对引擎做区分，可以很方便地从 MyISAM 引擎改到 InnoDB 引擎。
   - 避免受到文件损坏的影响，如果直接复制原始文件，可能会受到某个文件损坏的影响而得到一个损坏的备份。使用逻辑备份，只要 MySQL 还能执行 SELECT 语句，就可以得到一份可以信赖的逻辑备份，在文件损坏的时候很有用。

- 逻辑备份缺点：
   - 因为必须使用 MySQL 服务进行数据操作，所以备份的时候会占用更多 CPU，且备份时间可能会更长。
   - 逻辑备份在某些场景下比数据库文件更大，文本存储的数据不总是比存储引擎更高效。当然，使用压缩的话会得到一个更小的备份，但是要占用 CPU 资源。（如果索引较多，逻辑备份会比物理备份小。）
   - 恢复时间更长，使用逻辑备份的数据恢复，需要占用更多资源去进行锁分配、索引构建、冲突检查、日志刷新。

- 逻辑备份常用方法:
   - mysqldump 是 MySQL自带的备份工具，通用性强，非常常见。使用的使用通常要加上一些参数，后面继续介绍。
   - select into outfile，以符号分割数据创建逻辑备份，对于要导入到 CSV 等表格会比较实用。
   - mydumper，允许使用多线程进行备份，备份文件会进行表结构和数据分离，在恢复某些表或数据的时候会非常有效

#### 物理备份
物理备份在数据量较大的时候非常常见。
- 物理备份的优势：
   - 备份速度快，因为物理备份是基于复制进行备份，意味者复制有多快，备份就能有多快。
   - 恢复速度快，只需要把文件复制到数据库目录就可以完成恢复，不需要检查锁、构建索引。
   - 恢复简单，对于 MySIAM 引擎的表，不需要停库，只需要简单地复制进数据目录就可以。对于 InnoDB，如果是每个表一个表空间，也可以不停库操作，使用卸载加载表空间的方式便可导入（不太安全）。

- 物理备份缺点：
   - 没有官方物理热备份工具的支持。没有官方工具的支持，意味着出问题的概率较大，使用的时候就要谨慎了
   - InnoDB 的原始文件通常比逻辑备份要大。InnoDB 表空间往往包含很多未被使用的空间，InnoDB 表在删除数据后不会释放空间，所以即使数据量不大，文件有可能很大。除此以外，文件中除了数据还包含了索引、日志等信息。
   - 物理备份不总可以跨平台跨版本。MySQL 文件和操作系统、MySQL 版本息息相关，如果环境与原来不一致，很有可能会出现问题。

- 物理备份常用方法:
   - xtrabackup 是最常用的物理备份工具，由 percona 开源，能够实现对 InnoDB 存储引擎和 XtraDB 存储引擎非阻塞地备份（对于 MyISAM 还是要加锁），得到一份一致性备份。
   - 直接复制文件/文件系统快照，这种方式对于 MySIAM 引擎是非常高效的，只需要执行 FLUSH TABLE WITH READ LOCK 就可以复制得到一份备份文件。但是对于 InnoDB 引擎就比较困难，因为 InnoDB 引擎使用了大量的异步技术，即使执行了 FLUSH TABLE WITH READ LOCK，它还是会继续合并日志、缓存数据。所以要用这种方法备份 InnoDB，需要确保 checkpoint 已经最新。

### 为什么要备份 binlog
如果有 DBA 告诉你，这个数据库能够恢复到两个个月内任何状态，这说明了，这个数据库的 binlog 日志至少保留了两个月。备份 binlog 的好处:
- 可以实现基于任意时间点的恢复
- 可以用于误操作数据闪回
- 可以用于审计

### 增量备份
当数据了变得庞大时，一个常见策略就是做定期的增量备份。例如：周一做了一次全量备份，然后周二到日做增量备份。
增量备份只包含变化的数据集，一般情况不会比原始数据量大，所以可以减少服务器的开销、备份时间、备份空间。
当然，使用增量备份也会有风险，增量备份每一次迭代都是基于上一次的备份实现，意味着只要其中一份备份出现问题，那么就有可能导致所有备份不可用。

### 延迟同步
延迟同步是常见的使用主从复制使用模式，在遇到误操作的时候，无论是用于恢复数据，还是使用跳过的方式跳过错误都是非常有用。

例如在主库做了 drop 的误操作，在主库找到命令所在 binlog 日志和 pos 位置，Delay库停止同步，然后使用 start slave until master_log_file='<对应file>',master_log_pos=<误操作命令前一个SQL的pos>; 等待同步到这个位置，执行跳过一条 SQL 的命令再开启同步。

### 数据校验
除了备份，非常重要的一件事情就是验证备份数据的可用性。想象一下，当你需要进行数据恢复的时候，忽然发现过去的备份数据都是无效的，那得有多难受。很多朋友在写好备份脚本加到定时任务后，只要检查到定时任务有执行，备份目录有文件就不再关注了，往往到了需要使用备份文件的时候才发现备份数据有问题。

目前对于备份文件的数据校验没有非常方便的办法，用的比较多的还是定时把备份文件拉出来做备份恢复演练，例如一个月做一次备份恢复演练就可以有效提高备份文件可用性，心里也踏实。

数据校验部分，如果是逻辑备份，往往会抽查某个表的数据，检查是否符合预期。如果是物理备份，首先要使用 mysqlcheck 等命令检查是否有表损坏，没有损坏再抽查表数据。

### 总结
- 逻辑备份和物理备份可以一起使用，不同的备份周期使用不同的工具，全备周期不应该太长，至少一周一次全备
- 如果数据量较大，可以使用增量备份的方式减少数据量，要注意的是，增量备份风险更大
- binlog功能要开启，且最好定时备份 binlog
- 有条件的话可以增加一个 Delay 库，在做紧急恢复的时候有奇效
- 数据校验要定时去做，否则当需要备份恢复的时候而备份文件又失效，后悔都来不及

## 事务
在MySQL5.7中事务默认以只读事务开启，当随后判定为读写事务时，则转换成读写模式，并为其分配事务ID和回滚段
### mvcc
https://www.cnblogs.com/bytesfly/p/mysql-transaction-innodb-mvcc.html
#### 二级索引与mvcc
https://blog.51cto.com/u_15127645/2778788
### undo log
https://www.cnblogs.com/ZhuChangwu/p/14060916.html
https://www.cnblogs.com/zzyhogwarts/p/14961054.html
https://blog.csdn.net/qq_39459385/article/details/84644005
https://juejin.cn/post/6844903686393561096
配置：
https://www.linuxidc.com/Linux/2017-05/143400.htm
回收（自动）：
https://www.cnblogs.com/tmdba/p/6385504.html
https://blog.csdn.net/weixin_34347959/article/details/113435653

##redo log、undo log、binlog


## 索引
https://xie.infoq.cn/article/01441fa0cc16819f13a8085c9
https://www.infoq.cn/article/ojkwyykjoyc2ygb0sj2c
https://tech.meituan.com/2014/06/30/mysql-index.html
https://www.huaweicloud.com/articles/2a80018c95d27b7c26e6840301b7bd18.html
### 索引结构与磁盘空间顺序
https://www.cnblogs.com/lqlqlq/p/13715450.html
### innodb与myisam索引
https://cloud.tencent.com/developer/article/1615563
innodb聚族索引：
- 优点
①节省一次IO；
②主键按顺序排列，方便主键范围查找
③对于二级索引想要找主键是很容易的
- 缺点：
①插入依赖于插入顺序，随机插入容易造成页分裂
②行移动代价比较大
②二级索引查找可能需要进行回表操作

myisam一级索引：
- 优点
①插入、行移动代价小，不需要动到数据
②不需要进行回表操作
- 缺点
①按照一级索引检索，需要多一次io
②数据按照插入顺序排列，范围查找，会造成多次的io开销
③二级索引，没法直接获取主键，需要多一次io开销


### 联合索引
https://juejin.cn/post/6844904073955639304
### 索引合并
https://blog.csdn.net/itas109/article/details/79151299
### ICP
https://www.cnblogs.com/zhoujinyi/archive/2013/04/16/3016223.html
https://www.cnblogs.com/Terry-Wu/p/9273177.html
https://blog.csdn.net/wanghao112956/article/details/90714806
复合索引(a, b, c)，最左前缀，where也包含其余索引列但是没法使用索引（例如模糊查询c like ‘%XXX%’）
ICP能减少引擎层回表的次数和MySQL Server 访问存储引擎的次数。
### ICP、MRR、BKA
https://blog.csdn.net/caomiao2006/article/details/52205177
- mrr保证同一个页只需一次读取，没有mrr可能同一个页被读取多次。
### NULL在索引存储方式
https://juejin.cn/post/6844903921450745863

## group by  
https://blog.csdn.net/xtdhqdhq/article/details/18408905
- 松散型(Using index for group-by)补充：
通过group-by去使用索引；
使用覆盖索引；
最左前缀匹配，但是不能包含完整索引列；
聚合函数只能使用max(), min()；
> 只需找到min，上一条就是上一个分组的max了。相当于转化成 如何找上一个值的问题了。从页目录找到上一个组长，发起单链表扫描。直到下一个值==min，那么这个值就是上一组的max。

- 紧凑型(Using index for group-by)补充：
通过where + group-by去使用索引；
或者需要回表查；

### sql_mode问题
`which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by`

https://www.imooc.com/article/294753

### 临时表
https://www.jb51.net/article/85341.htm

### group by 和 order by同时使用
https://www.cnblogs.com/matengfei123/p/10158639.html

1 order by 非分组字段正例(每个班级，按总分数排名)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-161bd4d274e83d48.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


2. group by 内部排序问题(每个班级，按总分数排名，取每个班级第一名)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-4c17b09594eabd67.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
limit是防止派生表被优化。

## order by 
### 文件排序问题 
https://blog.csdn.net/xianyun1992/article/details/107651184
https://blog.csdn.net/lijingkuan/article/details/70341176

## explain使用说明
https://zhuanlan.zhihu.com/p/260236204
Filtered：表示返回结果的行数占需读取行数的百分比。
在表连接的时候很有用，可以看到驱动表的扫了多少次被驱动表：Rows * Filtered
### Extra
https://www.cnblogs.com/wy123/p/7366486.html
查找数据方式：
1. 索引查找：利用B+tree特性
2. 索引扫描：顺序扫描
3. 表扫描：顺序扫描

- Using Index（分组场景下有Using Index for group-by）：索引查询、不用回表
查询字段：索引列；
查询条件：索引第一个键（等值，`区分度差的时候，可能会变成Using where; Using index`）；

- NULL：索引查询、回表
查询字段：非索引列（回表）；
查询条件：索引第一个键（等值，`区分度差的时候，可能会变成Using Index condition`）；

- Using Index condition：索引扫描、回表
查询字段：非索引列（回表）；
查询条件：（索引第一个键 + 其他索引键）或者 （索引第一个键是范围查询）；

- Using where ：全表扫描

- Using where; Using index：索引扫描、不用回表
   查询字段：索引列；
   查询条件：（索引第一个键 + 其他索引键）或者 （索引第一个键是范围查询）或者（其他索引键）；

- Using Index condition; Using where：索引扫描、表扫描、回表
   查询字段：任意；
   查询条件：（索引第一个键 + 其他索引键）或者 （索引第一个键是范围查询）或者（其他索引键）；

- Using MRR：回表之前将主键排序，从而减少随机IO

## SQL查询语句执行顺序详解（引起怀疑）
https://juejin.cn/post/6864555988873707527
个人认为应该是外表先通过where过滤，再进行表连接；
扫描内表时on、where一起进行过滤；
另外子查询where exists是先整个外层查完（包括limit），再查子查询

## 表连接原理
https://www.cnblogs.com/zuochanzi/p/10409752.html
https://www.modb.pro/db/49439
### 连接例子
![image.png](https://upload-images.jianshu.io/upload_images/24337324-511b31cf223f3479.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 子查询原理
https://juejin.cn/post/6844904201357623304#heading-48
### 子查询优化（半连接）
https://www.cnblogs.com/micrari/p/6921806.html
5.7 对in、any进行优化，not in会自动转not exists；
8.0 对in、any、exists进行优化
注意：子查询与内连接的区别：子查询不能让外表出现重复数据
- 内连接-TablePullOut
  子查询关联字段是主键或唯一索引
- 内连接-FirstMatch
  子查询关联字段是非唯一索引，或者关联字段不是索引
- 内连接-LooseScan
  子查询关联字段是联合索引
- 物化（物化表不会重复）
  关联字段是非索引，先物化，然后建立唯一索引，再表连接
- 重复消除
  用不了索引，创建临时表消除重复
### in 与 exists
https://juejin.cn/post/6844903876252925960
### 派生表
https://blog.csdn.net/Smallc0de/article/details/111552824
https://www.cnblogs.com/onlysun/p/4512076.html

## 完整性约束
https://zhuanlan.zhihu.com/p/82319482
### 外键
https://blog.csdn.net/lan12334321234/article/details/70049370
https://www.cnblogs.com/wasayezi/p/7412049.html

## 分区表
https://www.cnblogs.com/dw3306/p/12620042.html
http://mysql.taobao.org/monthly/2017/11/09/
https://blog.csdn.net/javahuangfang/article/details/8541420
### 分区列必须包含主键（唯一索引）问题
原因：为了确保主键的效率。否则同一主键区的东西一个在Ａ分区，一个在Ｂ分区，显然会比较麻烦。主键属于区里面私有的，保证各个区之间不冲突。
解决方案：可以使用复合主键（id, 分区列）
例如：update XXX where id = 1;这个就得做全分区扫描，所以相当于mysql想告诉用户，主键必须要跟分区字段绑定。

## 分库分表
https://www.cnblogs.com/f-ck-need-u/p/9388407.html
### 分布式全局id问题
https://segmentfault.com/a/1190000020993874
https://juejin.cn/post/6844904016141369352
https://tech.meituan.com/2017/04/21/mt-leaf.html
https://cloud.tencent.com/developer/news/678423 赞赞赞
趋势递增的含义：
由于“没有一个全局时钟”，每台服务器分配的ID是绝对递增的，但从全局看，生成的ID只是趋势递增的（有些服务器的时间早，有些服务器的时间晚）
### sharding-jdbc
https://jishuin.proginn.com/p/763bfbd600c2
## 读写分离
SDK与中间件
https://juejin.cn/post/6844904013272449037
https://www.modb.pro/db/66316
https://www.dazhuanlan.com/endlessid/topics/1002107
### proxySQL（选中）
https://www.cnblogs.com/keme/p/12290977.html
https://blog.csdn.net/weixin_45697293/article/details/111300068
https://javamana.com/2021/04/20210413150128325u.html
https://cloud.tencent.com/developer/article/1429052
写：默认路由到主库；
读：需要区分select与select ... for update
多个分组（多个数据库）：在规则里面配置不同账号或者不同数据库名
### mycat 
https://www.cnblogs.com/yanl55555/p/13607577.html

## 锁机制
https://learnku.com/articles/39212?order_by=vote_count&
https://juejin.cn/post/6878884451162521613
### 意向锁
https://juejin.cn/post/6844903666332368909
### 自增锁
https://juejin.cn/post/6968420054287253540
### 插入意向锁
注意：insert分为三种场景
①如果当前被加了间隙锁，则insert需要申请插入意向锁
②如果当前没有被加间隙锁，此时可以直接插入，使用隐式锁。
③插入如果存在主键或者是唯一索引冲突，则加入next-key。具体看书本442.
https://juejin.cn/post/6844903666856493064
### next key lock加锁规则
https://www.cnblogs.com/a-phper/p/10313940.html
### 死锁
https://www.aneasystone.com/archives/2018/04/solving-dead-locks-four.html

## 性能优化
https://bbs.huaweicloud.com/blogs/detail/180470
https://blog.csdn.net/weixin_41588751/article/details/105640955
https://cloud.tencent.com/developer/article/1406085
https://zhuanlan.zhihu.com/p/60098742
### count相关
https://segmentfault.com/a/1190000019930046
### limit相关
#### 先limit再回表（利用子查询）
https://blog.csdn.net/weixin_43944305/article/details/105977378

## 性能监控
https://www.cnblogs.com/davygeek/p/9272415.html
### performance_schema
https://blog.csdn.net/n88Lpo/article/details/80331752

## 线上修改配置
https://www.huaweicloud.com/articles/12622915.html

## 我的看法
### count问题
mysql可以内置一张表例如取名sys_count表，把需要计数的业务表加入到sys_count表里面。
好处：O(1)时间复杂度；
缺点：维护成本高，增删会被锁住，用户学习成本（创建业务表的时候需要通过标识表示加入到sys_count中）。

## 问题解答
### 加锁
- 无主键的加锁过程
无PK时，会创建一个隐式聚簇索引。加锁在这个隐式聚簇索引会有什么不同？

- 复合索引加锁过程

- 多条件(where condition)加锁过程
- 隐式锁与显式锁，隐式锁什么情况下会转换成显式锁

- 如果插入意向锁不阻止任何锁，这个锁还有必要存在吗？
目前看到的作用是，通过加锁的方式来唤醒等待线程。
但这并不意味着，被唤醒后可以直接做插入操作了。需要再次判断是否有锁冲突。
### 死锁相关
- 死锁发生的几个案例
https://www.toutiao.com/i6911991148575834628/?tt_from=weixin&utm_campaign=client_share&wxshare_count=1&timestamp=1634813906&app=news_article&utm_source=weixin&utm_medium=toutiao_android&use_new_style=1&req_id=20211021185826010212142017041CB6D7&share_token=f65385c7-6bfc-4e8c-a2f9-456f944bf71d&group_id=6911991148575834628&wid=1634815247594
- 为什么RR隔离机制比RC隔离机制更容易出现死锁？
https://blog.csdn.net/cri5768/article/details/100199527
https://www.jianshu.com/p/3e57a428d2a2
RR隔离机制有间隙锁，另外正常来说唯一索引是没有间隙锁（只有该记录被标记删除了，或者记录找不到的时候、或者查询条件没有覆盖全、或者是null查询 =》会用到间隙锁）；
RC隔离机制一般来说是没有间隙锁，除了一些特殊场景：https://www.jianshu.com/p/db97a65294c0

- 造成死锁的原因
   - 两次访问锁模式变了，没法快速加锁，进入锁等待队列，如果两次访问中间有其他事务加了锁，就会造成死锁。
   - 插入意向锁与间隙锁冲突导致死锁：https://www.jianshu.com/p/db97a65294c0
https://www.jianshu.com/p/bbdf5d3884d6
   
- insert中对唯一索引的加锁逻辑
   ①先做UK冲突检测，如果存在目标行，先对目标行加S Next Key Lock（该记录被标记删除或者该记录是未被事务提交，其他事务对记录加X + 行锁）
   ②如果1成功，对对应行加X + 插入意向锁
   ③如果2成功，插入记录，并对记录加X + 行锁

