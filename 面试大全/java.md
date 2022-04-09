## 知识点大全
https://pdai.tech/md/java/thread/java-thread-x-juc-AtomicInteger.html 赞赞赞

## 面试大全
https://blog.csdn.net/qq_41701956/article/details/110119625
https://link.zhihu.com/?target=https%3A//urlify.cn/JVFVVf
https://www.eet-china.com/mp/a79667.html 赞赞赞
https://xie.infoq.cn/article/ce5ad283b31a7bfe21ac5c542 赞赞赞
https://blog.csdn.net/Shockang/article/details/116724631 赞赞赞

### 并发相关
https://juejin.cn/post/6844904125755293710#heading-88
https://www.cnblogs.com/sgh1023/p/10297322.html#_lab51
https://zhuanlan.zhihu.com/p/43383072
https://blog.csdn.net/ThinkWon/article/details/104863992

## 常量池深度理解
https://www.cnblogs.com/itplay/p/11137526.html 赞赞赞
https://juejin.cn/post/6938233964725272606
https://segmentfault.com/a/1190000038807051
### 包装类常量池
https://www.cnblogs.com/YangJavaer/p/13213003.html
https://blog.csdn.net/hlhzxy/article/details/110139291

## 字符串深度理解
https://juejin.cn/post/6844903913116680199
https://blog.nowcoder.net/n/3aa7ddc109014fbcb3c0ad8f0d57a486

## 为什么要使用元空间
https://zhuanlan.zhihu.com/p/111809384
https://www.cxyzjd.com/article/q5706503/84621210

## 深拷贝与浅拷贝
https://segmentfault.com/a/1190000038523408

## 二进制与json序列化对比
https://blog.csdn.net/qq_43719932/article/details/116374272

## c与java堆栈
https://blog.csdn.net/qq_18108083/article/details/81878320

## 类加载
https://segmentfault.com/a/1190000037574626
### 类加载器
https://www.cnblogs.com/twoheads/p/10143038.html
https://cloud.tencent.com/developer/article/1674553
https://juejin.cn/post/6921691978828611591
- jar包冲突问题
https://cloud.tencent.com/developer/news/627029
### 打破双亲委派
https://www.xttblog.com/?p=5249
### 热加载
https://developer.aliyun.com/article/65023
https://www.cnblogs.com/niumoo/p/11756703.html

## SPI
https://www.jianshu.com/p/46b42f7f593c
https://zhuanlan.zhihu.com/p/28909673

## 字节码指令
https://blog.csdn.net/u012988901/article/details/99852568

## 对象创建与内存管理
https://www.modb.pro/db/53954  赞赞赞

## 为什么JVM开启指针压缩后支持的最大堆内存是32G?
https://blog.csdn.net/liujianyangbj/article/details/108049482
jvm指针压缩的意思：使用4个字节存放指针，原来是8个字节（64位机器）。
指针四个字节可以表示的数字大小为2^32，如果是寻址大小以一个字节为单位那边可以表示4G的内存大小。java对象的大小是8字节的倍数，也就是说寻址的时候是以8字节为单位。4G * 8 = 32G。因此jvm的压缩指针最大可以表示32G的内存。超过了这个大小，压缩指针就得失效。

## 为什么JVM中的新生代要有两个Survivor区
https://blog.csdn.net/towads/article/details/79784249

## 啥是Full GC
https://cloud.tencent.com/developer/article/1582661
注意：直接内存是收到fullGC管理的
https://www.cnblogs.com/duanxz/p/6089485.html

## GC Root枚举（安全点、安全区域）
https://zhuanlan.zhihu.com/p/286110609
https://www.cnblogs.com/chenchuxin/p/15259439.html

## 并发的可达性分析（三色标记）
https://www.cnblogs.com/jmcui/p/14165601.html
https://zhuanlan.zhihu.com/p/108706654 赞赞赞

## 记忆集（卡表）
https://www.cnblogs.com/krock/p/14409988.html
https://www.iteye.com/blog/dsxwjhf-2201685 （oopMap与记忆集的理解）

## 反射
https://juejin.cn/post/6844903741024387079
### 反射为什么慢
https://www.bilibili.com/read/mobile?id=13256282&share_token=cd807e01-8f32-48cc-930d-14401d1601d4
https://www.iteye.com/blog/rednaxelafx-548536

## 伪共享问题
https://blog.csdn.net/qq_27680317/article/details/78486220 TODO
https://juejin.cn/post/6934528318888738824

## XX:ParallerGCThreads参数
https://www.cnblogs.com/dengq/p/13687534.html

## CMS并发清理阶段为什么是安全的（有新对象产生）
https://blog.csdn.net/FU250/article/details/105386291
注意：CMS回收老年代需要把整个新生代加入到GC Root。（参考jvm3书本第103页最下方备注）

## G1
https://blog.csdn.net/nihui123/article/details/103752749 赞赞赞
https://xie.infoq.cn/article/1a8b9b129a603e216a5a876d7  赞赞赞
https://bbs.huaweicloud.com/blogs/detail/281945
https://www.cnblogs.com/yanl55555/p/13366387.html
https://juejin.cn/post/6844904055282597902 总结博客，写的还行
https://zhuanlan.zhihu.com/p/419393801

- Young GC触发条件
  当E区不能再分配新的对象
- Mixed GC触发条件
 整个堆占用率大于InitiatingHeapOccupancyPercent
 元空间超过MetaspaceSize发生扩容
 调用System.gc且开启ExplicitGCInvokesConcurrent
- Full GC触发条件（转换为serial old收集器）
 没有连续的可用region存放大对象
 调用System.gc且未开启ExplicitGCInvokesConcurrent
 没有足够的内存供存活对象或晋升对象使用（报错Allocation Failure）
 元空间达到MaxMetaspaceSize之后

## 内存分配策略
https://www.cnblogs.com/heyonggang/p/11597609.html

## 美团案例分析（CMS GC问题分析与解决）
https://tech.meituan.com/2020/11/12/java-9-cms-gc.html
https://tech.meituan.com/2016/09/23/g1.html

## 执行引擎
### 恰当的变量作用域控制变量的回收（该方式不建议推广）
![image.png](https://upload-images.jianshu.io/upload_images/24337324-a4715d7dc323ab11.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-919dcf690065c6c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 方法变量都需要设置初始值，否则不能使用。（类变量不太一样，类变量在准备阶段就会被初始化零值）。

## 解释器与编译器
https://www.cnblogs.com/lemon-le/p/13840014.html 赞赞赞

## 解释执行
https://tenggegege.github.io/2019/08/10/%E4%BB%A3%E7%A0%81%E6%98%AF%E5%A6%82%E4%BD%95%E5%9C%A8CPU%E4%B8%AD%E6%89%A7%E8%A1%8C%E8%B5%B7%E6%9D%A5%E7%9A%84/

## 即时编译
https://juejin.cn/post/6995362542386151431 赞赞赞
https://tech.meituan.com/2020/10/22/java-jit-practice-in-meituan.html
https://cloud.tencent.com/developer/article/1583182
https://juejin.cn/post/6844903630408155150#comment 一般


## MESI
https://juejin.cn/post/7005454086988365838
https://cloud.tencent.com/developer/article/1548942
https://blog.csdn.net/wll1228/article/details/107775976 赞赞赞（讲解了指令重排的原因）
https://www.cnblogs.com/xiaoxiongcanguan/p/13184801.html 赞赞赞（总结的很好，最终一致性）
https://1024eye.com/article/22247

### 锁缓存行与mesi的关系
https://www.zhihu.com/question/419819590/answer/1462472048

## 内存屏障与lock前缀指令的理解
https://www.hitzhangjie.pro/blog/2021-04-17-locks%E5%AE%9E%E7%8E%B0%E9%82%A3%E4%BA%9B%E4%B8%8D%E4%B8%BA%E4%BA%BA%E7%9F%A5%E7%9A%84%E6%95%85%E4%BA%8B/ 赞赞赞 （讲述了lock前缀指令被解释成屏障的关系）
https://zhuanlan.zhihu.com/p/43526907

- jvm为了兼容各个CPU平台的指令重排优化，采用最保守的方案。也就是在前端编译期生成字节码的时候对每个volatile修饰的变量进行全方位的内存屏障插入。（这边的内存屏障是JMM自己定义的语义，只有jvm自己看的懂）
写volatile：前面插入StoreStore、后面插入StoreLoad
读volatile：后面插入LoadLoad、LoadStore
- 另外经过后端编译后（具体化到某个平台 ==> 汇编），对于x86（intel系）来说，会在volatile的写操作加入一个lock前缀指令。根据intel相关文档，该指令可以起到全内存屏障的作用（StoreLoad，x86只支持StoreLoad），也就是写入变量的时候会让storeBuffer、invalidQueue数据进行冲刷，来保证MESI的强一致性。

## 内存模型

## 双重检查锁
https://blog.csdn.net/weixin_44730681/article/details/115014920

## 协程与线程
https://cloud.tencent.com/developer/article/1768030
https://www.cnblogs.com/Survivalist/p/11527949.html#%E5%8D%8F%E7%A8%8B

## 内核线程
https://www.jianshu.com/p/7c980955627e

## 锁升级
https://www.cnblogs.com/trunks2008/p/14646610.html#21-%E5%81%8F%E5%90%91%E9%94%81%E5%8E%9F%E7%90%86
https://blog.csdn.net/tongdanping/article/details/79647337

## 偏向锁
https://blog.csdn.net/sinat_41832255/article/details/89309944

## 偏向锁与hashcode能共存吗？
https://blog.csdn.net/Saintyyu/article/details/108295657

## 批量重偏向/撤销
https://zhuanlan.zhihu.com/p/127884116
https://zhuanlan.zhihu.com/p/26475023
https://blog.csdn.net/qq_40977118/article/details/108578312
- JVM在每个类维护了一个偏向锁撤销计数器和一个epoch，对象初始为偏向锁状态的时候`对象的epoch`与`类的epoch`相等。
- 每次对该类的对象发生偏向撤销操作的时候，偏向锁撤销计数器+1，当达到批量重偏向阈值（默认20）的时候，就会对`该类epoch+1`。
- 同时遍历JVM所有线程栈，找到该类的正处于活跃状态的偏向锁对象，将`对象的epoch`设置为`该类的epoch`；
- 其余非活跃状态的对象，则不被更新，之后其他线程获得锁的时候，发现`对象epoch`不等于`该类的epoch`，被当做偏向锁无效，此时不会发生撤销操作，而是将对象重新偏向新的线程，同时`更新对象的epoch`等于`该类的epoch`。
- 在这之后如果类C的计数器继续增长（比如有其他新的线程竞争，导致撤销操作），当达到偏向锁批量撤销阈值（默认40）时，JVM会认为类C不适合偏向，之后，对于该类的新实例对象，不能走偏向锁的逻辑。


## 数组元素没法修饰volatile的问题
https://www.cnblogs.com/silyvin/p/9106613.html
https://blog.csdn.net/u014674862/article/details/89168376?utm_medium=distribute.pc_aggpage_search_result.none-task-blog-2~aggregatepage~first_rank_ecpm_v1~rank_v31_ecpm-1-89168376.pc_agg_new_rank&utm_term=volatile+%E4%BF%AE%E9%A5%B0%E6%95%B0%E7%BB%84&spm=1000.2123.3001.4430 （不能真正说明问题，读取volatile变量不一定会去主存获取最新值，这个要看CPU平台，参考volatile读屏障）

## UNSAFE（LazySet：putOrderedInt）
https://blog.csdn.net/u010597819/article/details/113922471
https://www.cnblogs.com/throwable/p/9139947.html 

## aqs（CLH）
https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html 赞赞赞
### ConditionObject
```
=======ConditionObject=====
private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null; // 存储最新有效节点
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next; // 第一个有效节点
                    else
                        trail.nextWaiter = next; // 最新有效节点的nextWaiter跳过当前无效节点
                    if (next == null)
                        lastWaiter = trail; // 最后一个有效节点
                }
                else
                    trail = t;
                t = next;
            }
        }
```
### Condition之await中断问题
```
final boolean transferForSignal(Node node) {
        /*
         * cas操作失败，节点被取消
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
// 唤醒（中断）前，转移状态的判断。
final boolean transferAfterCancelledWait(Node node) {
        // 如果cas操作失败（可以认为还没调用signal()就已经中断了）
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
            enq(node);
            return true;
        }
       // 节点还没转移到同步队列，这时候需要再等一等。（如果触发signal这个时候是不完整的）
        while (!isOnSyncQueue(node))
            Thread.yield();
        return false;
    }
```

### hasQueuedPredecessors()解析（判断线程需不需要排队）
```
public final boolean hasQueuedPredecessors() {
    //读取头节点
    Node t = tail; 
   //读取尾节点
    Node h = head;
    //s是首节点h的后继节点
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```
- ①h != t
表示队列中至少有2个节点存在。（存在两种情况如下）
- ②正在初始化第2个节点：(s = h.next) == null
初始化的时候头指针指向初始化节点，尾指针指向后继节点。说明现在有另一个线程准备入队。（初始化优先生效prev指针，next指针是最后生效）
- ③完整>=2个节点：s.thread != Thread.currentThread()
首节点的后继节点不是当前线程。

### 共享模式之PROPAGATE的意义
https://segmentfault.com/a/1190000014721183
传播的两个依据：
①许可证余量；
②获取许可证之后逻辑如果存在线程并发doReleaseShared()，那这个时候是无法释放后继节点，所以将头结点状态设置为PROPAGATE，希望上一个被唤醒的线程将唤醒动作传播下去。
- ***PROPAGATE 状态用在哪里，以及怎样向后传播唤醒动作的？***
PROPAGATE 状态用在 setHeadAndPropagate。当头节点状态被设为 PROPAGATE 后，后继节点成为新的头结点后。若 propagate > 0 条件不成立，则根据条件h.waitStatus < 0成立与否，来决定是否唤醒后继节点，即向后传播唤醒动作。

- ***引入 PROPAGATE 状态是为了解决什么问题？***
余量判断的补充，尽量将唤醒动作传播下去，即便唤醒有问题也没多大关系。
引入 PROPAGATE 状态是为了解决并发释放信号量所导致部分请求信号量的线程无法被唤醒的问题。
```
private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; 
       // 情况一：假设propagate = 0，并且在这个时间点之间存在多个线程释放了同步状态doReleaseShared();
      // 此刻head状态=0，那么是无法去唤醒头结点的后继节点。先将头结点状态设置成PROPAGATE 。

        setHead(node);

       // 情况二：假设propagate = 0，并且在这个时间点之间存在多个线程释放了同步状态doReleaseShared();
      //这个时候判断新的head=SIGNAL， 只有一个线程通过CAS操作将新的head=0，然后唤醒下一个后继节点。
      // 其他线程先将新的head = PROPAGATE 。

      // 无论是情况一、还是情况二，如果只是根据propagate > 0（同步状态余量），来判断是否要往下传播是不够的。
      // 如果是情况一：h == null || h.waitStatus < 0
     // 如果是情况二：(h = head) == null || h.waitStatus < 0
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }

private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                
            }
            if (h == head)                   
                break;
        }
    }
```

## join与CountDownLatch的区别
join：每个子线程都要调用一次，有可能造成没必要的多次上下文切换，另外这种写法也不太美观。
https://juejin.cn/post/7004473544545419278

## ThreadLocal
https://zhuanlan.zhihu.com/p/238231572
https://www.cnblogs.com/yangyongjie/p/11017409.html
https://blog.csdn.net/tujisitan/article/details/106176680 赞赞
key弱引用，gc设置key为空，entry变脏。
之后get、remove、set方法就可以使用渐进式方式回收脏entry。
get、remove：清理脏entry
set：替换脏entry、清理某些插槽
### InheritableThreadLocal
https://www.jianshu.com/p/1af4f7582b80
### WeakHashMap
https://cloud.tencent.com/developer/article/1778310
https://ifeve.com/weakhashmap/

## 泛型
擦除：仅仅对code属性的字节码进行擦除（方法体），实际上元数据中还是保留了泛型信息，通过Signature属性来保存类、接口、方法的泛型信息。这也就是为什么我们可以通过反射取得泛型参数的依据。
https://blog.nowcoder.net/n/edbe746cb53d48c294ccf679a2f67a1a 不错
https://www.cnblogs.com/afraidToForget/p/10727014.html 讲得还行
### 通配符（弥补泛型不支持协变的问题）
https://www.cnblogs.com/bax-life/p/14629985.html?share_token=4851b431-7d8b-491f-98fe-cb8a680b6fc8
> 泛型与重写冲突问题---桥方法
https://pdai.tech/md/java/basic/java-basic-x-generic.html


## FutureTask
https://blog.csdn.net/m0_37836195/article/details/84102067

![image.png](https://upload-images.jianshu.io/upload_images/24337324-d9c9ed6666f41f94.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-6f569cbb45fe86bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### FutureTask中的waiters为什么这么设计？(单向队列，前插法)
FutureTask中的waiters为什么这么设计？
### 安全保证（CAS）
- state修改：通过CAS很容易实现;
- waiters节点新增，移除
新增的话很容易实现。
移除要考虑边界问题：
①移除是首节点，这个时候会跟新增冲突，所以需要通过CAS保证安全；
②并发移除相邻的两个节点，移除动作实际是 上个节点next指针 指向 当前节点的 下一个节点，所以并发移除两个相邻节点会出现指针断层的问题。目前FutureTask的做法是移除动作做完以后再次检查下前一个节点是不是被移除掉，如果是，则从头遍历一遍。
### CompletableFuture （jdk1.8）
https://blog.csdn.net/qq_31865983/article/details/106137777

## 如何减少上下文切换
https://blog.csdn.net/sinat_27143551/article/details/103033608

## 阻塞队列
https://cloud.tencent.com/developer/article/1706970
### ArrayBlockingQueue
有界队列
循环数组，putIndex 生产者生产到哪个索引，takeIndex消费者消费到哪个索引，count代表当前的数量。
ReentrantLock（生产、消费共享一把锁） + Condition（notEmpty->消费者，notFull->生产者）

### LinkedBlockingQueue
有界队列、无界队列。
capacity：表示无界队列还是有界队列
单向链表。
ReentrantLock（putLock生产锁， takeLock消费锁） + Condition（notEmpty->消费者，notFull->生产者）
> 锁分离带来额外的操作
①生产者：当前容器未满的情况下需要唤醒生产者，上一次容器是空的情况要唤醒消费者。
②消费者：当前容器未空的情况下需要唤醒消费者，上一次容器是满的情况要去唤醒生产者。
> 线程安全问题
①count计数安全：AtomicInteger
②生产者与消费者操作节点边界安全问题
正常来说当队列只有一个节点的时候生产者与消费者同时操作才会存在线程安全问题，所以LinkedBlockingQueue的做法就是默认生成一个空节点，将head指针指向该空节点，所以整个队列至少有两个节点，才能被消费。

### PriorityBlockingQueue（学习平衡二叉堆树） TODO
https://juejin.cn/post/6968795388685844488

### SynchronousQueue
https://zhuanlan.zhihu.com/p/29227508
https://segmentfault.com/a/1190000019153021
无容器、无堆积。
相同类型（isData是否一样）下的线程会被挂起。
不同类型下的线程会相互匹配，唤醒原来互补的线程，将其从队列或者栈中移除。
里面是通过cas + 自旋 保证线程并发安全性（casNext操作指针、casItem操作匹配）。
两类操作：匹配（casItem=e）、中断（casItem=this)
clean节点：头结点的后继节点（将头结点指针指向目标节点，清理掉原来头结点）、中间节点（目标节点的前节点的next指针指向目标节点的后节点）、尾结点（标记cleanMe指的目标节点的前节点，然后删除目标节点，有可能出现尾结点加入新节点，所有cleanMe的next指针要指向新节点）


> 要求生产者请求需要很快的处理掉。具体应用：Executors.newCachedThreadPool()就使用了SynchronousQueue，这个线程池根据需要（新任务到来时）创建新的线程，如果有空闲线程则会重复使用，线程默认空闲了60秒后会被回收。

## fail-fast
https://blog.csdn.net/zymx14/article/details/78394464

## 迭代器
https://blog.csdn.net/u012442401/article/details/47850501

## CopyOnWrite...
https://www.cnblogs.com/chengxiao/p/6881974.html

## 异常机制
https://www.cnblogs.com/wfhking/p/9464458.html
https://www.jianshu.com/p/d72daaa9b722

## 反射
https://www.cnblogs.com/ssl-bl/p/11032748.html

## 函数式编程
https://www.it1352.com/998792.html
![image.png](https://upload-images.jianshu.io/upload_images/24337324-a35e55e7c3e948ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## hashMap
https://blog.csdn.net/weixin_42373997/article/details/112085344 很赞的面试题
https://blog.csdn.net/qq_38685503/article/details/88430788
https://blog.csdn.net/qq_35590459/article/details/108988011
1.8使用后插法，循环链表问题，如果前插法并发的时候不就更容易丢数据？
### tableSizeFor巧妙获取最接近的2^n
https://www.cnblogs.com/xiyixiaodao/p/14483876.html
### hash右移异或
https://www.zhihu.com/question/20733617
```
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
### 循环链表问题
https://blog.csdn.net/qxhly/article/details/104738976
考虑：当前槽有a->b，两个线程1,2。
线程1迁移到新数组后 b-> a；
线程2也开始迁移了，线程2原来的状态是next=b，e=a。所以先插入a，再插入b，b.next还有值a，所以继续插入a。就造成循环了。
### 相关问题
https://www.cnblogs.com/taotaobaibai/articles/13726618.html
### LinkedHashMap 
https://segmentfault.com/a/1190000012964859

### Comparable（内）和Comparator（外）区别比较
https://www.cnblogs.com/xujian2014/p/5215082.html

## HashTable、ConcurrentHashMap为何不支持null键和null值
https://blog.csdn.net/u010979642/article/details/117333313

get方法获取到的value的结果都为null。所以当我们用get方法获取到一个value为null的时候，这里会产生分歧：
    可能没有这个key
    可能有这个key，只不过value为null
对于单线程来说解决这个分歧很容易实现：
containsKey(key)，get(key)

对于多线程来说：
containsKey(key)，get(key)两个放在一起并不是一个原子操作，中间可能有其他线程干扰。

## ConcurrentHashmap
1.7：分段锁+数组+链表
1.8：槽锁+数组+链表+红黑树；计数分片（不存在竞争则更新baseCount，存在竞争则使用计数单元数组，如果计数单元存在竞争，则调整线程随机值，或者对计数单元数组进行扩容）；多线程分片迁移（片长度取决cpu，从索引高位置递减进行分片，如果旧数组槽为空或者槽迁移完成，则插入ForwardingNode）；
自旋+cas初始化操作；
lastrun；
getObjectVolatile保证数组元素可见性；
读、迁移安全性：迁移中不会破坏原来链表的关系，迁移完成会插入一个ForwardingNode，读取的时候根据ForwardingNode转到新数组上面读取。
### sizeCtl含义
新建而未初始化时：sizeCtl==cap
初始化过程中：sizeCtl==-1
初始化完成后：sizeCtl==cap*0.75
正在扩容时：sizeCtl==-1N，N最高位=1，高16的非最高位可以表示旧数组的容量前面有几个0，低16位 - 1代表当前有多少个线程正在帮助迁移。
### addCount
https://segmentfault.com/a/1190000039667702 TODO
并发修改baseCount，使用分片法，CountCell[]，每个线程对应去修改每个countCell，如果存在冲突还会调整线程随机值（默认是0），或者对CountCell[]进行扩容。
### 面试题
https://blog.csdn.net/zzu_seu/article/details/106698150

## BufferedInputStream为什么性能高
https://blog.csdn.net/weixin_42340670/article/details/106303366
### inputstream坑
https://www.cnblogs.com/x_wukong/p/9395881.html

## nio
### buffer
https://developer.aliyun.com/article/646519
### 直接内存的优势
https://www.zhihu.com/question/60892134
### MappedByteBuffer与DirectByteBuffer
https://zthinker.com/archives/directbytebuffer%E4%B8%8Emappedbytebuffer
> MappedByteBuffer是个抽象类，DirectByteBuffer是MappedByteBuffer的子类，MappedByteBuffer是通过new DirectByteBuffer创建出来，只不过创建的构造函数，与分配直接内存的构造函数不一样。这样做的目的实际上是复用DirectByteBuffer的API。所以两个本质上不是同一个东西。
## JNDI
http://j0k3r.top/2020/08/11/java-jndi-inject/#%EF%BC%883%EF%BC%89JRPM-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96
http://blog.topsec.com.cn/java-jndi%E6%B3%A8%E5%85%A5%E7%9F%A5%E8%AF%86%E8%AF%A6%E8%A7%A3/


