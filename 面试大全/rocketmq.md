## 原理梳理
https://blog.csdn.net/csdnnews/article/details/109505979
## 主流技术对比
https://cloud.tencent.com/developer/article/1683304
https://www.jianshu.com/p/4ec578455316
https://zhuanlan.zhihu.com/p/70123414   赞赞赞
https://developer.aliyun.com/article/669688
### rabbitmq
https://rdc.hundsun.com/portal/article/937.html
没有注册中心的概念，路由元数据都放在broker上面

### rocketmq与kafka
https://m.toutiaocdn.com/i7057420294008439328/?app=news_article&timestamp=1643252289&use_new_style=1&req_id=202201271058090102120420322564CDE5&group_id=7057420294008439328&tt_from=mobile_qq&utm_source=mobile_qq&utm_medium=toutiao_android&utm_campaign=client_share&share_token=10948a43-2edf-4737-b2d3-57880deb341f

https://www.jianshu.com/p/35fcd6e0bc93

## 推模式与拉模式
https://blog.csdn.net/FateRuler/article/details/85764080
rabbitmq：推、拉（支持手动确认、自动确认）
kafka：拉
rocketmq：拉

## 可靠消息最终一致性
https://cloud.tencent.com/developer/article/1824295

## 最大努力通知
https://developer.aliyun.com/article/766685 赞赞赞

## 官方建议
https://github.com/apache/rocketmq/blob/master/docs/cn/best_practice.md

## 我的部署架构
机房A：一个namesrv，一个broker（master-slave）
机房B：一个namesrv，一个broker（master-slave）
两个机房同时提供服务。
故障处理方案：如果slave挂问题不大。如果master挂了，改master无法写入，客户端会通过负载均衡机制绕过该故障的master。写入到另外一个机房的master。所以影响也不会很大。接下来可以手动重启故障的master。

## dleger理解
https://zhuanlan.zhihu.com/p/77166786
性能问题：tps下降将近10倍
https://blog.csdn.net/qq23ue/article/details/112764507

## 源码改造
### 消费组和主题建立强关联
①在rocketmq-console建立消费组和主题
①客户端消费者在启动的时候会发送心跳给所有broker
![image.png](https://upload-images.jianshu.io/upload_images/24337324-e0df5fea718f7ce1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-b89bfd860f4468f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

②broker接收到心跳会做一层校验，订阅信息不合法那就返回失败
![image.png](https://upload-images.jianshu.io/upload_images/24337324-1503867414b10a1a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
③客户端消费者抛出异常并打印错误日志信息。

### rocketmq-console
参考阿里云控制台：https://help.aliyun.com/document_detail/43357.html
- 整个前端干掉（默认augularjs），换成angular7
- 消息堆积告警（租户自行配置，默认每10分钟内，超过100条数据）
语音、企业微信、短信、邮件，什么时间段采用哪种通信方式。
- 死信队列告警（租户自行配置，默认每10分钟内，默认超过5条数据）
- 新增acl管理界面
①全局白名单设置。global_white_ips
②初次分配：在线分配accessKey，secrectKey，租户白名单，topic权限，组权限（PUB|SUB）。
access_manage
③创建topic：客户端（生成者）想要创建某个topic，在rocketmq-console数据库生成一条申请记录，然后在rocketmq真正创建。
access_manage（创建主题、acl添加一条主题权限pub|sub）
④订阅审批：topic与消费者组授权，客户端（消费者）想要订阅某个topic，先在rocketmq-console界面去申请订阅，在rocketmq-console数据库生成一条申请记录，发邮件通知topic的提供方负责人，进行审批，审批通过后直接调用broker的UPDATE_AND_CREATE_TOPIC接口（MQClientAPIImpl已内置该接口调用）。
（acl添加一条主题权限sub，添加一个消费组）。
另外，保持消费组与主题强关联（多对多）。
access_manage
```
- accessKey: RocketMQ
  secretKey: 12345678
  whiteRemoteAddress:
  admin: false
  defaultTopicPerm: DENY
  defaultGroupPerm: SUB
  topicPerms:
  - topicA=DENY
  - topicB=PUB|SUB
  - topicC=SUB
  groupPerms:
  # the group should convert to retry topic
  - groupA=DENY
  - groupB=PUB|SUB
  - groupC=SUB
```
- 统计每个topic的（昨日生产总数	昨日消费总数	今天生产总数	今天消费总数）
![image.png](https://upload-images.jianshu.io/upload_images/24337324-25967eb3043fdf13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/24337324-adac503a01877f86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
默认都是使用topic去记录，累加所有的topic的计数=broker的总数。
![image.png](https://upload-images.jianshu.io/upload_images/24337324-517ff5d29c4cb523.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 每天零点定时统计的记录持久化到数据库

- 租户隔离（每个租户只能看到各自发布的topic的详细信息，主题名称：
租户名_主题，组名称：租户名_组名称）。

## 源码分析
### 客户端SDK主要类
![image.png](https://upload-images.jianshu.io/upload_images/24337324-3f9bdb0cf8875c4b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### nameserver
管理路由元数据。
每个broker在启动时，会向当前所有的nameserver发送路由元数据，并且之后的每隔30s一次进行路由元数据的更新。
nameserver之间不会互相通信。
nameserver每隔一段时间会去检测哪些broker超过120s没有过来续约，如果超过则简单判断为下线，并剔除路由表与该broker相关的信息。
客户端实例定期30s拉取路由元数据信息。

### 生产者启动
![image.png](https://upload-images.jianshu.io/upload_images/24337324-b38d307f169a955d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-50d56dfe32d45352.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 生产者发送消息
- 校验消息
![image.png](https://upload-images.jianshu.io/upload_images/24337324-514804aeaa4409d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 查询主题的路由信息
![image.png](https://upload-images.jianshu.io/upload_images/24337324-3fb59a695a40fc2c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果最终都没有找到则抛出异常。
![image.png](https://upload-images.jianshu.io/upload_images/24337324-eba5df84b19130ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-89cfcae4f51939fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-562c60fbe1cd0234.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-a851860784a7b318.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

消息路由方式有两种：
①轮询（默认）：根据broker、队列序号两个维度进行排序，轮询该排序列表，如果发生故障，则记住上一次请求的broker，下一次请求过规避掉该broker。
②故障延迟机制：根据每次的请求的延迟时间，从映射表获取对应的不可用时间，将broker及下一次可用时间存入到故障延迟表，通过轮询的方式判断当前broker是否可用（下一次可用时间<now），如果可用则直接调用。如果不可用则将故障延迟表的broker按照下一次可用时候进行排序，优先选中可用性较好的broker。
- 消息发送
同步发送
异步发送
单向发送

### 消息存储
#### 文件
- commitLog
存储消息内容。固定文件大小，默认1G；文件名=起始偏移量。
条目：消息长度、消息内容。
偏移量定位，调整mappedFile指针即可。
- ConsumeQueue
提高消费者检索速度。固定文件大小，默认30w * 20字节；三级目录，consumequeue -> 主题名称 -> 队列id。
条目：commitLog offset、size、tag hashcode
文件名=逻辑偏移量。
时间查找：二分法。
- IndexFile
根据key属性，快速从commitLog检索消息。
hash索引。提供500w个hash槽，2000w个条目。hash冲突链表。前插法。
条目：hashcode、phyoffset、timedif（已经有记录开始时间，这个时候用差量就够了，节省空间）、pre index no
- checkpoint
记录commitLog、ConsumeQueue、IndexFile最后一次刷盘时间
- abort
用于判断是否异常重启
- config目录
配置信息，消费进度、订阅信息、主题信息
#### 内存映射
https://blog.csdn.net/weixin_51709400/article/details/122648396
- MappedFile：映射commitLog、ConsumeQueue、IndexFile等文件
wrotePosition：写指针；
committedPosition：提交指针，专门给暂存池用的；
flushPosition：刷盘指针。
- MappedFileQueue：对存储目录的封装，管理多个MappedFile
> mappedFile预分配与预热
预分配：服务启动的时候分配两个mappedFile（当前mappedFile、下一个mappedFile）,后续当前mappedFile写满的时候可以直接切换到原先已经预分配好的下一个mappedFile，并且又会让后台线程AllocateMappedFileService继续分配下下一个mappedFile。所有当mappedFile写满了以后，是不存在mappedFile创建所带来的损耗。另外一点是预热：所谓预热是当分配mappedFile的时候，会预先写入0。这样做的好处有两点：①做内存锁定的时候，并不会完全锁定该内存区域。linux是根据写时复制，按需分配的。②分配内存的时候，并不会立即分配相应的页，所以在操作的过程会发生多次的缺页中断。所以mappedFile预热说白了，就是一次性锁定内存，一次性分配完所需要的页。通过这种方式防止性能抖动问题。
##### 暂存池
在内存资源紧张的情况下，内存映射内核缓冲区，可能会发生频繁的swap操作，必然会影响到读写性能。开辟一块堆外内存（用户空间），并锁定该内存区域（单独占用物理内存），避免操作系统发生swap内存交换。
另外，暂存池会按照一定周期，一定数量的页提交到内核缓冲区上，类似是读写分离，从而提高存储性能。
缺点就是消息会有一定的滞后。
#### 刷盘机制
- 同步刷盘
组提交，GroupCommitService后台线程：读容器、写容器。写容器接收用户消息，当触发刷盘的时候，将读写容器互相转化（加锁），使用读容器的消息去刷盘。
- 异步刷盘
FlushRealTimeService线程每500ms将FileChannel新追加的内存刷入到磁盘中。（wrotePosition - flushPosition）
另外如果开启了暂存池，额外会开启一个CommitRealTimeService线程每隔200ms将ByteBuffer新追加的内容提交到FileChannel。（wrotePosition - commitedPosition）
#### 过期文件删除
默认每个文件的过期时间为72小时，后台线程10s调度一次，判断文件是否过期。
- 指定文件删除时间点：例如凌晨4点；
- 磁盘空间不足，则立即删除。
### 消息消费
#### 消息队列负载与重分配
在同一主题下，一个队列只能同时被一个消费者消费，一个消费可以消费多个队列。
- 消费者定时向每个broker发送心跳包，注册自己的消费者信息。
- RebalanceService，消费者会定时20 s去拉取同一主题下的消费者id列表，消费队列列表。并将其进行排序按照平均分配策略（有5种，平均分配，hash取模，一致性hash，手动配置，机房配置）进行分配。
- 剔除掉过时的消费队列（ProcessQueue，dropped=true），添加新调整的消费队列，并将新添加的消费队列生成一个PullRequest加入到PullMessageQueue。
#### 消息推还是拉
rocketmq支持拉模式（手动去调用api）、推模式，不过推模式底层是基于拉模式去实现的。
- 后台线程PullMessageService从PullMessageQueue中获取PullRequest，每个PullRequest代表这一个队列，每次拉取最多不超过32条消息，拉取完会将PullRequest重新放入到PullMessageQueue上，等待下一轮被拉取。
- 拉取的msgList按偏移量顺序被放入到ProcessQueue（可以认为是消费队列的内存状态，PullRequest的字段）中，然后提交给线程池去消费。
- 拉取的时候可能会触发流程，ProcessQueue消息条数 > 1000或者ProcessQueue消息的跨度 > 1000。消费者消费能力跟不上，减缓拉取速度，或者消费者某条消息发生堵塞，导致消费进度无法推进。
- broker主节点会根据自己能力，建议下次应该从主节点拉取，还是从从节点拉取。
- 如果拉取的offset不对，broker会调整offset，并建议下次从该offset开始拉取。
- 长轮询，如果当时没有可消费的消息，broker会挂起当前消费者请求。①接着会有后台线程PullRequestHoldService 5s一次去轮询这些请求的消费队列是否有消息进来，如果有则唤醒消费者请求。②如果当前节点时主节点，当对应的消息被ReputMessageService线程存储到消费队列时，会唤醒对应的消费者请求。这种方式有较高的实时性。
#### 消息重试
集群模式一开始会订阅当前主题、重试消息主题"%RETRY%"+消费组名。
#### 并发消费
- 如果是重试消息，则需要进行原主题的恢复
- 执行具体监听器逻辑
- 返回CONSUME_SUCCESS或CONSUM_LATER。
- ProcessQueue移除该消息，不管消费成功还是重试消费都会更新消费进度（取ProcessQueue最小的偏移量），重试消费会重新生成一条新的消息存储到commitLog上。
- 如果是CONSUM_LATER， 创建重试消息主题"%RETRY%"+消费组名
- CONSUM_LATER重试消息会被当做延迟消息进行存储（将重试消息的主题、队列id保存到属性字段上），默认情况下第一次重试延迟级别=3，每重试一次延迟级别+1
```
if (0 == delayLevel) {
                delayLevel = 3 + msgExt.getReconsumeTimes();
            }
```
- 超过一定重试次数，默认16，就会被加入到死信队列。
#### 延迟消息
不支持任意时间精度的定时任务。只提供特定级别的延迟消息。原因：任意时间精度意味着要进行排序处理，为了更好的性能rocketmq只支持特定级别的延迟消息。
1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
SCHEDULE_TOPIC：统一定时任务主题
每个延迟级别对应一个消费队列。
每个消费队列都会有一个后台线程去获取定时消息，判断定时消息是否可以被处理，如果可以，就会将其还原为原来的消息并插入到commitLog中。
#### 顺序消息
为了保证顺序，只能有一个消费队列。另外消费者需要保证消费的时候是单线程在处理。
- 消费队列负载
每个消费者可以尝试向broker对应的消费队列加锁，只有一个消费者才可以获得
- 消息拉取
只有获得锁的消费者才可以去拉取消息
- 消息消费
同一时刻只能有一个线程在消费。
消息重试，只能在本地重试，消息重试过程中进度无法推进，重试达到一定的次数会进入死信队列中。
#### 消息过滤
tag、sql、类过滤。
注意：tag过滤，broker只会过滤tag的hashcode，还需要在本地进行一次过滤。消费队列条目存储了tag的hashcode，为了固定长度而考虑。

#### 消费进度
- 进度持久化
广播模式持久化在本地，集群模式持久化在broker。
- 每消费一条消息都会更新消费进度（取ProcessQueue最小的偏移量）
- 重分配时，剔除对应消费队列，并更新消费进度。
> 消费进度读取
CONSUM_FROM_FIRST_OFFSET：从头开始
CONSUM_FROM_TIMESTAMP：从某个时间点开始
CONSUM_FROM_LAST_OFFSET：从最大偏移量开始

1、主，从服务器都在运行过程中，消息消费者是从主拉取消息还是从从拉取？
答：默认情况下，RocketMQ消息消费者从主服务器拉取，当主服务器积压的消息超过了物理内存的40%，则建议从从服务器拉取。但如果slaveReadEnable为false，表示从服务器不可读，从服务器也不会接管消息拉取。

2、当消息消费者向从服务器拉取消息后，会一直从从服务器拉取？
答：不是的。分如下情况：
1）如果从服务器的slaveReadEnable设置为false，则下次拉取，从主服务器拉取。
2）如果从服务器允许读取并且从服务器积压的消息未超过其物理内存的40%，下次拉取使用的Broker为订阅组的brokerId指定的Broker服务器，该值默认为0，代表主服务器。
3）如果从服务器允许读取并且从服务器积压的消息超过了其物理内存的40%，下次拉取使用的Broker为订阅组的whichBrokerWhenConsumeSlowly指定的Broker服务器，该值默认为1，代表从服务器。

3、主从服务消息消费进是如何同步的？
答：消息消费进度的同步时单向的，从服务器开启一个定时任务，定时从主服务器同步消息消费进度；无论消息消费者是从主服务器拉的消息还是从从服务器拉取的消息，在向Broker反馈消息消费进度时，优先向主服务器汇报；消息消费者向主服务器拉取消息时，如果消息消费者内存中存在消息消费进度时，主会尝试跟新消息消费进度。

读写分离的正确使用姿势：
1、主从Broker服务器的slaveReadEnable设置为true。
2、通过updateSubGroup命令更新消息组whichBrokerWhenConsumeSlowly、brokerId，特别是其brokerId不要设置为0，不然从从服务器拉取一次后，下一次拉取就会从主去拉取。

### 事务消息
#### 执行流程
- 发送消息
- 存储到RMQ_SYS_TRANS_HALF_TOPIC，队列为0
- 事务提交/回滚
- 不会真的删除，存储到RMQ_SYS_TRANS_OP_HALF_TOPIC，代表已经处理过的半消息
#### 回查流程
通过TransactionalMessageCheckService线程去定期处理RMQ_SYS_TRANS_HALF_TOPIC消息，默认是1分钟一次。
- 通过OP过滤已经处理过的半消息（需要检验Op列表的有效性，保证Op列表最后一条数据的创建时间 > startTime + transationTimeOut，若不符合则多次分页查询，一次32条）
- 如果超过最大回查次数或者文件到达过期时间，则丢弃该消息。进度推进
- 异步发送回查动作，发送之前会copy原来的并创建一条新的prepare消息。进度推进。（原因：异步处理，没法立即知道结果。另外一点是回查需要修改检查次数，rocketMq不支持修改动作。几乎所有的操作都是追加，也就是顺序IO。）
- 最后会更新half、op的进度
```
public void check(long transactionTimeout, int transactionCheckMax,
        AbstractTransactionalMessageCheckListener listener) {
        try {
            String topic = TopicValidator.RMQ_SYS_TRANS_HALF_TOPIC;
            Set<MessageQueue> msgQueues = transactionalMessageBridge.fetchMessageQueues(topic);
            if (msgQueues == null || msgQueues.size() == 0) {
                log.warn("The queue of topic is empty :" + topic);
                return;
            }
            log.debug("Check topic={}, queues={}", topic, msgQueues);
            for (MessageQueue messageQueue : msgQueues) {
                long startTime = System.currentTimeMillis();
                MessageQueue opQueue = getOpQueue(messageQueue);
                long halfOffset = transactionalMessageBridge.fetchConsumeOffset(messageQueue);
                long opOffset = transactionalMessageBridge.fetchConsumeOffset(opQueue);
                log.info("Before check, the queue={} msgOffset={} opOffset={}", messageQueue, halfOffset, opOffset);
                if (halfOffset < 0 || opOffset < 0) {
                    log.error("MessageQueue: {} illegal offset read: {}, op offset: {},skip this queue", messageQueue,
                        halfOffset, opOffset);
                    continue;
                }

                List<Long> doneOpOffset = new ArrayList<>();
                HashMap<Long, Long> removeMap = new HashMap<>();
                // 拉取32条op消息。（分页）
                PullResult pullResult = fillOpRemoveMap(removeMap, opQueue, opOffset, halfOffset, doneOpOffset);
                if (null == pullResult) {
                    log.error("The queue={} check msgOffset={} with opOffset={} failed, pullResult is null",
                        messageQueue, halfOffset, opOffset);
                    continue;
                }
                // single thread
                int getMessageNullCount = 1;
                long newOffset = halfOffset;
                long i = halfOffset;
                while (true) {
                    if (System.currentTimeMillis() - startTime > MAX_PROCESS_TIME_LIMIT) {
                        log.info("Queue={} process time reach max={}", messageQueue, MAX_PROCESS_TIME_LIMIT);
                        break;
                    }
                    // 如果过是处理过的就跳过该消息，进度推进
                    if (removeMap.containsKey(i)) {
                        log.debug("Half offset {} has been committed/rolled back", i);
                        Long removedOpOffset = removeMap.remove(i);
                        doneOpOffset.add(removedOpOffset);
                    } else {
                        GetResult getResult = getHalfMsg(messageQueue, i);
                        MessageExt msgExt = getResult.getMsg();
                        // 如果获取到prepare消息为空，可重试查询一次
                        if (msgExt == null) {
                            if (getMessageNullCount++ > MAX_RETRY_COUNT_WHEN_HALF_NULL) {
                                break;
                            }
                            if (getResult.getPullResult().getPullStatus() == PullStatus.NO_NEW_MSG) {
                                log.debug("No new msg, the miss offset={} in={}, continue check={}, pull result={}", i,
                                    messageQueue, getMessageNullCount, getResult.getPullResult());
                                break;
                            } else {
                                log.info("Illegal offset, the miss offset={} in={}, continue check={}, pull result={}",
                                    i, messageQueue, getMessageNullCount, getResult.getPullResult());
                                i = getResult.getPullResult().getNextBeginOffset();
                                newOffset = i;
                                continue;
                            }
                        }

                        // 如果超过最大回查次数或者文件到达过期时间，则丢弃该消息。进度推进
                        if (needDiscard(msgExt, transactionCheckMax) || needSkip(msgExt)) {
                        // 放入TRANS_CHECK_MAX_TIME_TOPIC主题里面，消费进度推进
                            listener.resolveDiscardMsg(msgExt);
                            newOffset = i + 1;
                            i++;
                            continue;
                        }
                        // 本次只处理，消息创建时间小于当前开始时间的消息
                        if (msgExt.getStoreTimestamp() >= startTime) {
                            log.debug("Fresh stored. the miss offset={}, check it later, store={}", i,
                                new Date(msgExt.getStoreTimestamp()));
                            break;
                        }
                        // 消息到现在的存储了多久
                        long valueOfCurrentMinusBorn = System.currentTimeMillis() - msgExt.getBornTimestamp();
                        // 立即回查时间：默认使用事务超时时间，如果用户有指定那就以用户指定的为主
                        long checkImmunityTime = transactionTimeout;
                        String checkImmunityTimeStr = msgExt.getUserProperty(MessageConst.PROPERTY_CHECK_IMMUNITY_TIME_IN_SECONDS);
                        if (null != checkImmunityTimeStr) {
                            checkImmunityTime = getImmunityTime(checkImmunityTimeStr, transactionTimeout);
                            // 如果还没到达用户指定的回查时间，则跳过该消息（新增一条追加到commitLog后面），进度推进
                            if (valueOfCurrentMinusBorn < checkImmunityTime) { 
                                if (checkPrepareQueueOffset(removeMap, doneOpOffset, msgExt)) {
                                    newOffset = i + 1;
                                    i++;
                                    continue;
                                }
                            }
                        } else {
                            // 用户没有指定，那就判断是否超过事务超时时间，如果没有说明更后面的也不符合，结束本次循环。
                            if ((0 <= valueOfCurrentMinusBorn) && (valueOfCurrentMinusBorn < checkImmunityTime)) {
                                log.debug("New arrived, the miss offset={}, check it later checkImmunity={}, born={}", i,
                                    checkImmunityTime, new Date(msgExt.getBornTimestamp()));
                                break;
                            }
                        }
                        List<MessageExt> opMsg = pullResult.getMsgFoundList();
                        // 需要检查opMsg的时效性，opMsg列表中的最后一条数据的创建时间 > 开始时间 + 事务超时时间
                        // （本次只处理prepare创建时间 < 开始时间，op创建时间 <= prepare创建时间 + 事务超时时间）
                        boolean isNeedCheck = (opMsg == null && valueOfCurrentMinusBorn > checkImmunityTime)
                            || (opMsg != null && (opMsg.get(opMsg.size() - 1).getBornTimestamp() - startTime > transactionTimeout))
                            || (valueOfCurrentMinusBorn <= -1);

                        // 只有opMsg有效，才能说明原先过滤已处理消息的动作才是有效的。否则，就得往下一页拉取opMsg列表。
                        if (isNeedCheck) {
                            if (!putBackHalfMsgQueue(msgExt, i)) {
                                continue;
                            }
                            // 异步发送回查动作，发送之前会copy原来的并创建一条新的prepare消息。进度推进。
                            // 原因：异步处理，没法立即知道结果。
                            //      另外一点是回查需要修改检查次数，rocketMq不支持修改动作。几乎所有的操作都是追加，也就是顺序IO。
                            listener.resolveHalfMsg(msgExt);
                        } else {
                            pullResult = fillOpRemoveMap(removeMap, opQueue, pullResult.getNextBeginOffset(), halfOffset, doneOpOffset);
                            log.debug("The miss offset:{} in messageQueue:{} need to get more opMsg, result is:{}", i,
                                messageQueue, pullResult);
                            continue;
                        }
                    }
                    newOffset = i + 1;
                    i++;
                }
                // 最后会更新prepare、op的进度
                if (newOffset != halfOffset) {
                    transactionalMessageBridge.updateConsumeOffset(messageQueue, newOffset);
                }
                long newOpOffset = calculateOpOffset(doneOpOffset, opOffset);
                if (newOpOffset != opOffset) {
                    transactionalMessageBridge.updateConsumeOffset(opQueue, newOpOffset);
                }
            }
        } catch (Throwable e) {
            log.error("Check error", e);
        }

    }
```

## 相关问题
https://www.cnblogs.com/guoyu1/p/11677766.html
- 一个组消费多个topic（实例1消费topic1，实例2消费topic2）TODO
https://blog.csdn.net/qq_34216875/article/details/81207998
主题中的部分队列无法被消费；
重试主题是一样的（%RETRY% + 消费组名），那就意味着重试主题包含多种业务意义（一般一个主题代表一种业务功能），但是组内所有的消费者都可以消费多种业务意义的。
- 扩容问题
https://cloud.tencent.com/developer/article/1501781
https://blog.csdn.net/NeverQuitMan/article/details/102837626
界面上提供了一个功能（暂时没有、但可以做）：扩容时，主题、订阅配置全量修改。
- 旧版本docker clientId一致造成消息堆积
https://juejin.cn/post/6844904199667318798

## 相关原理
### autoCreateTopicEnable机制（默认主题的作用）
https://blog.csdn.net/u013087026/article/details/109821582
### namesrv动态配置
![image.png](https://upload-images.jianshu.io/upload_images/24337324-8714305d5e23e58f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### nameserver原理
https://www.jianshu.com/p/19d18b67a097
### 消息发送、消息存储
https://cloud.tencent.com/developer/article/1717385
### 关机恢复
https://www.cnblogs.com/zuoyang/p/14467949.html
- 正常恢复
```
 /**
     * When the normal exit, data recovery, all memory data have been flush
     */
    public void recoverNormally(long maxPhyOffsetOfConsumeQueue) {
        // 是否进行消息校验和判断 true
        boolean checkCRCOnRecover = this.defaultMessageStore.getMessageStoreConfig().isCheckCRCOnRecover();
        final List<MappedFile> mappedFiles = this.mappedFileQueue.getMappedFiles();
        if (!mappedFiles.isEmpty()) {
            // Began to recover from the last third file
            int index = mappedFiles.size() - 3;
            if (index < 0)
                index = 0;

            // 从第三个文件开始
            MappedFile mappedFile = mappedFiles.get(index);
            ByteBuffer byteBuffer = mappedFile.sliceByteBuffer();
            long processOffset = mappedFile.getFileFromOffset();
            long mappedFileOffset = 0;
            while (true) {
                // 校验消息是否格式正确
                DispatchRequest dispatchRequest = this.checkMessageAndReturnSize(byteBuffer, checkCRCOnRecover);
                int size = dispatchRequest.getMsgSize();
                // Normal data
                if (dispatchRequest.isSuccess() && size > 0) {
                    mappedFileOffset += size;
                }
                // Come the end of the file, switch to the next file Since the
                // return 0 representatives met last hole,
                // this can not be included in truncate offset
                else if (dispatchRequest.isSuccess() && size == 0) {
                    index++;
                    if (index >= mappedFiles.size()) {
                        // Current branch can not happen
                        log.info("recover last 3 physics file over, last mapped file " + mappedFile.getFileName());
                        break;
                    } else {
                        mappedFile = mappedFiles.get(index);
                        byteBuffer = mappedFile.sliceByteBuffer();
                        processOffset = mappedFile.getFileFromOffset();
                        mappedFileOffset = 0;
                        log.info("recover next physics file, " + mappedFile.getFileName());
                    }
                }
                // 消息格式不正确，则从当前位置开始往后的消息及文件都要删掉
                // Intermediate file read error
                else if (!dispatchRequest.isSuccess()) {
                    log.info("recover physics file end, " + mappedFile.getFileName());
                    break;
                }
            }

            processOffset += mappedFileOffset;
            this.mappedFileQueue.setFlushedWhere(processOffset);
            this.mappedFileQueue.setCommittedWhere(processOffset);
            this.mappedFileQueue.truncateDirtyFiles(processOffset);

            // Clear ConsumeQueue redundant data
            if (maxPhyOffsetOfConsumeQueue >= processOffset) {
                log.warn("maxPhyOffsetOfConsumeQueue({}) >= processOffset({}), truncate dirty logic files", maxPhyOffsetOfConsumeQueue, processOffset);
                this.defaultMessageStore.truncateDirtyLogicFiles(processOffset);
            }
        } else {
            // Commitlog case files are deleted
            log.warn("The commitlog files are deleted, and delete the consume queue files");
            this.mappedFileQueue.setFlushedWhere(0);
            this.mappedFileQueue.setCommittedWhere(0);
            this.defaultMessageStore.destroyLogics();
        }
    }
```
- 异常恢复
```
public void recoverAbnormally(long maxPhyOffsetOfConsumeQueue) {
        // 是否进行消息校验和判断 true
        // recover by the minimum time stamp
        boolean checkCRCOnRecover = this.defaultMessageStore.getMessageStoreConfig().isCheckCRCOnRecover();
        final List<MappedFile> mappedFiles = this.mappedFileQueue.getMappedFiles();
        if (!mappedFiles.isEmpty()) {
            // 从最后一个文件开始往前找，找到第一个可靠的文件（该文件的第一条消息的存储时间 < commitLog检查点时间）
            // Looking beginning to recover from which file
            int index = mappedFiles.size() - 1;
            MappedFile mappedFile = null;
            for (; index >= 0; index--) {
                mappedFile = mappedFiles.get(index);
                if (this.isMappedFileMatchedRecover(mappedFile)) {
                    log.info("recover from this mapped file " + mappedFile.getFileName());
                    break;
                }
            }

            if (index < 0) {
                index = 0;
                mappedFile = mappedFiles.get(index);
            }

            ByteBuffer byteBuffer = mappedFile.sliceByteBuffer();
            long processOffset = mappedFile.getFileFromOffset();
            long mappedFileOffset = 0;
            while (true) {
                // 校验消息是否格式正确
                DispatchRequest dispatchRequest = this.checkMessageAndReturnSize(byteBuffer, checkCRCOnRecover);
                int size = dispatchRequest.getMsgSize();

                if (dispatchRequest.isSuccess()) {
                    // Normal data
                    if (size > 0) {
                        mappedFileOffset += size;

                        // 从最后一个可靠文件开始，将消息转发到消费队列、索引文件。所以可能会导致消息重复消费
                        // rocketMq保证消息不丢失，但不保证消息会重复消费。
                        if (this.defaultMessageStore.getMessageStoreConfig().isDuplicationEnable()) {
                            if (dispatchRequest.getCommitLogOffset() < this.defaultMessageStore.getConfirmOffset()) {
                                this.defaultMessageStore.doDispatch(dispatchRequest);
                            }
                        } else {
                            this.defaultMessageStore.doDispatch(dispatchRequest);
                        }
                    }
                    // Come the end of the file, switch to the next file
                    // Since the return 0 representatives met last hole, this can
                    // not be included in truncate offset
                    else if (size == 0) {
                        index++;
                        if (index >= mappedFiles.size()) {
                            // The current branch under normal circumstances should
                            // not happen
                            log.info("recover physics file over, last mapped file " + mappedFile.getFileName());
                            break;
                        } else {
                            mappedFile = mappedFiles.get(index);
                            byteBuffer = mappedFile.sliceByteBuffer();
                            processOffset = mappedFile.getFileFromOffset();
                            mappedFileOffset = 0;
                            log.info("recover next physics file, " + mappedFile.getFileName());
                        }
                    }
                } else {
                    log.info("recover physics file end, " + mappedFile.getFileName() + " pos=" + byteBuffer.position());
                    break;
                }
            }

            processOffset += mappedFileOffset;
            this.mappedFileQueue.setFlushedWhere(processOffset);
            this.mappedFileQueue.setCommittedWhere(processOffset);
            this.mappedFileQueue.truncateDirtyFiles(processOffset);

            // Clear ConsumeQueue redundant data
            if (maxPhyOffsetOfConsumeQueue >= processOffset) {
                log.warn("maxPhyOffsetOfConsumeQueue({}) >= processOffset({}), truncate dirty logic files", maxPhyOffsetOfConsumeQueue, processOffset);
                this.defaultMessageStore.truncateDirtyLogicFiles(processOffset);
            }
        }
        // Commitlog case files are deleted
        else {
            log.warn("The commitlog files are deleted, and delete the consume queue files");
            this.mappedFileQueue.setFlushedWhere(0);
            this.mappedFileQueue.setCommittedWhere(0);
            this.defaultMessageStore.destroyLogics();
        }
    }
```
### 消费原理
http://mstacks.com/133/1406.html#content1406 还可以
- 消费队列重平衡
https://www.jianshu.com/p/0637495c9243
https://www.jianshu.com/p/debce110b967
https://jaskey.github.io/blog/2020/11/26/rocketmq-consumer-allocate/ 赞赞赞
![image.png](https://upload-images.jianshu.io/upload_images/24337324-13a95e7ce725ce89.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 推还是拉模式
https://segmentfault.com/a/1190000023854950
https://blog.csdn.net/wanderlustLee/article/details/89294355
推模式流控。
- 消息拉取（参考书上会更细）
https://juejin.cn/post/6991101454303903781
http://wuwenliang.net/2019/08/20/%E8%B7%9F%E6%88%91%E5%AD%A6RocketMQ%E4%B9%8B%E6%B6%88%E6%81%AF%E6%8B%89%E5%8F%96%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/
注意：
- DefaultLitePullConsumer 
http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/RocketMQ%20%E5%AE%9E%E6%88%98%E4%B8%8E%E8%BF%9B%E9%98%B6%EF%BC%88%E5%AE%8C%EF%BC%89/11%20DefaultLitePullConsumer%20%E6%A0%B8%E5%BF%83%E5%8F%82%E6%95%B0%E4%B8%8E%E5%AE%9E%E6%88%98.md

- 顺序消费
https://blog.csdn.net/prestigeding/article/details/79422514
broker消费队列加锁，才可以拉取，从processQueue取出数据，线程池加锁消费。
重试，在本地挂起一会儿重试，超过一定次数推送到broker成为死信队列消息。
https://segmentfault.com/a/1190000040391600

- 事务消息
https://juejin.cn/post/6844904193526857742

- 主从复制
https://blog.csdn.net/prestigeding/article/details/93672079

## 博客总结
https://no.linkedin.com/pulse/rocketmq%E5%90%90%E8%A1%80%E6%80%BB%E7%BB%93-%E7%BA%A2%E5%96%9C-%E6%B2%88

https://www.daimajiaoliu.com/daima/4858ba3dc1003e8 赞赞赞（疑难解答）

https://www.163.com/dy/article/FQEPUF000511FQO9.html#post_comment_area 写的一般

## 部署架构
https://segmentfault.com/a/1190000038318572
### dledger部署
https://github.com/apache/rocketmq/blob/master/docs/cn/dledger/deploy_guide.md
https://cloud.tencent.com/developer/article/1755906

## 源码分析博客
https://www.136.la/jingpin/show-199906.html

## 面试刷题
https://www.cnblogs.com/javazhiyin/p/13327925.html
https://blog.csdn.net/lupengfei1009/article/details/114525762

