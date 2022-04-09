## AOP
### 动态代理
https://www.cnblogs.com/whirly/p/10154887.html
https://developer.aliyun.com/article/695983
### jdk
https://blog.csdn.net/CarryBest/article/details/112857330
### cglib fastclass
https://zhuanlan.zhihu.com/p/106069224
### 源码分析
https://cloud.tencent.com/developer/article/1512235 言简意赅！！！
https://www.xujun.org/note-116590.html 还行
https://developer.aliyun.com/article/761230
https://juejin.cn/post/6844904015843557389#comment
### springAOP与AspectJ
https://blog.csdn.net/flyfeifei66/article/details/82784321
### resolveBeforeInstantiation作用
给与一次返回代理对象的机会，用户自己一targetSource有关。
https://www.jianshu.com/p/bcaebced6309
### targetSource
https://blog.csdn.net/shenchaohao12321/article/details/85538163
springaop代理的是targetSource，代理类可以根据目标类生成，invokeHanlder组合传入的是targetSource，调用的时候从targetSource获取target。
### 通知执行顺序（递归实现）
方法与通知链会建立一个映射关系。

@Order 设置切面级的优先级，Order值越小before先执行，after越后面执行。

https://blog.csdn.net/qq_32331073/article/details/80596084
![image.png](https://upload-images.jianshu.io/upload_images/24337324-5b544209c86de5c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- before
![image.png](https://upload-images.jianshu.io/upload_images/24337324-24880deffe46afa2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- after
![image.png](https://upload-images.jianshu.io/upload_images/24337324-6cde3b2188496f66.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
注意：不同版本同一个切面逻辑不太一样
https://zhuanlan.zhihu.com/p/266111498

### aop问题
- 原始类的方法调用都属于本地调用
![image.png](https://upload-images.jianshu.io/upload_images/24337324-40c6af89369485cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-6920af0be1db1931.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
外部 -> 代理类 -> 原始类 -> 原始类

## bean创建
https://www.zybuluo.com/Catyee/note/1802409 赞赞赞
### 循环依赖
https://blog.csdn.net/czrbijiben/article/details/78680485 问题原因
https://www.zybuluo.com/Catyee/note/1803699 讲得还行
https://developer.aliyun.com/article/766880 
### 属性注入
https://www.cnblogs.com/lifullmoon/p/14452969.html
https://www.cnblogs.com/lifullmoon/p/14453011.html

## BeanFactory
### BeanFactory与FactoryBean
https://zhuanlan.zhihu.com/p/196688174
### FactoryBean的作用
https://blog.csdn.net/u014082714/article/details/81166648
## bean的scope
https://www.cnblogs.com/yoci/p/10642553.html
## @Import
https://zhuanlan.zhihu.com/p/147025312
## @Condition相关
https://blog.csdn.net/lizhiyuan_eagle/article/details/91583834
## @ComponentScan，@ComponentScans
https://blog.csdn.net/luojinbai/article/details/85877956
## @PropertySource
https://www.jianshu.com/p/e38613c24851
## @Async
https://cloud.tencent.com/developer/article/1647642

## 事件机制
https://mrbird.cc/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Spring%E4%BA%8B%E4%BB%B6%E5%8F%91%E5%B8%83%E4%B8%8E%E7%9B%91%E5%90%AC.html   赞赞赞赞！！
https://www.cnblogs.com/rickiyang/p/12001524.html 讲得一般

## springboot
### 项目结构
https://cloud.tencent.com/developer/article/1595466
https://www.jianshu.com/p/e6bb67c55b7a
### 入门篇
https://blog.csdn.net/qq_44850489/article/details/108666332
### 自动配置
https://juejin.cn/post/6988877227794366495
https://juejin.cn/post/6844903928958713863
### 启动tomcat容器
https://blog.csdn.net/qq_32101993/article/details/99700910
https://www.cnblogs.com/sword-successful/p/11383723.html
### 启动流程
https://juejin.cn/post/6895341123816914958
#### properties加载
https://blog.csdn.net/qq_31086797/article/details/107395108
### jar包启动
https://blog.csdn.net/kaihuishang666/article/details/108405691
https://www.cnblogs.com/ideaAI/p/13870105.html
![image.png](https://upload-images.jianshu.io/upload_images/24337324-6055558680d8cce4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- MANIFEST.MF
https://blog.csdn.net/u011479200/article/details/80097886

## 事务
https://juejin.cn/post/6887751198737170446#heading-6
### 事务常见问题
https://www.jianshu.com/p/283f044b820e
https://juejin.cn/post/6844904096747503629
### jpa + hibernate
https://www.cnblogs.com/JiHC/p/12845417.html
https://www.cnblogs.com/wang-meng/p/5701996.html
https://github.com/Homiss/Java-interview-questions/blob/master/%E6%A1%86%E6%9E%B6/Hibernate%E9%9D%A2%E8%AF%95%E9%A2%98.md
https://www.cnblogs.com/javastack/p/12957000.html
- 懒加载问题
https://blog.csdn.net/huangyaa729/article/details/84974679
- mappedBy
https://www.zhihu.com/question/64187262
https://www.cnblogs.com/redcoatjk/p/4236445.html
设置mappedBy的那一方表示放弃维护权。

## 数据库连接池
https://zhuanlan.zhihu.com/p/114800827

## springmvc
https://segmentfault.com/a/1190000021848063
https://juejin.cn/post/6844904017772937229
### 父子容器
https://cloud.tencent.com/developer/article/1816225
https://www.zhihu.com/question/488723652

## spingsecurity
https://segmentfault.com/a/1190000018616620
https://www.jianshu.com/p/0bb1e26868ba

## quartz
https://juejin.cn/post/6844903760624353293#heading-7
### QuartzSchedulerThread执行流程
![image.png](https://upload-images.jianshu.io/upload_images/24337324-41c16949cc192b7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-8dddbba92701330d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-307832f47acd7257.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-e5fe8c4fadb3b85b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-51949728a2bdc3e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-f160ea0a6ba13b63.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-533e45cc68a950e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-92a6288fa299fdff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-4a54e157dde1826a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-d864bfd39847480a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-fc76a7cf25a3b793.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-55a1074d3e86774f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-e3362a62a27d486b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/24337324-ac13e34be60e8df6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 同一个job上一次未执行完，下一次已经开始触发
https://blog.csdn.net/xdd19910505/article/details/52705479
两种方式：使用有状态job；concurrent=false;



## 面试
https://segmentfault.com/a/1190000022372094
https://www.cnblogs.com/lifullmoon/p/14422101.html
