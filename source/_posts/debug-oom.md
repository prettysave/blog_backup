---
title: 使用spring-integration-redis出现的内存泄漏
date: 2021-03-12 17:33:55
categories: 踩坑记录
tags:
- spring
- redis
- 分布式锁
---

早上上班不久，被运维告知昨天晚上生产环境出现了OOM，并产生了一个2.5G的.hprof文件。

通过控制台发现在 昨天晚上服务器CPU使用率较高，达到了百分之60-70，但在程序抛出OOM，重启之后CPU负载马上降到了10以下。当时的gc_time是这样的：

![](http://tvax4.sinaimg.cn/large/005NahBely1goh9u4qjuhj30gg050wf9.jpg)

很明显可以看出当前程序的old gc时间远远超过了young gc,说明老年代存在大量的无法回收的对象，导致频繁的old gc。并且CPU的负载率波动于old gc 的时长基本保持一致，初步估计是因为频繁的old gc导致服务器CPU负载飙升。

而在项目重启之后，CPU的负载、old gc time都有明显的降低，故怀疑是程序中出现了内存泄漏所致。
从运维那里得到当时的.hprof文件后，使用IDEA中的插件jprofiler进行分析：

![](http://tvax4.sinaimg.cn/large/005NahBely1goh68o9pifj32do0n2e81.jpg)

发现快照中存在大量的RedisLock对象，看了它就是内存泄漏的元凶。
这是程序中使用RedisLock的地方:

![](http://tvax4.sinaimg.cn/large/005NahBely1goh6dr0gllj31g20deqky.jpg)

看来好像没有什么问题，每次都会对锁进行释放，那么为什么会产生这么多的对象不回收呢？
接下来继续对快照进行分析，看一下快照中的大对象：

![](http://tvax4.sinaimg.cn/large/005NahBely1goh6gbfgjcj30rs04u4bt.jpg)

这一看了不得，RedisLockRegistry这个对象竟然占了2339502196个字节，转换一下就是 2G多 ？？
接下来把对象展开：

![](http://tvax4.sinaimg.cn/large/005NahBely1goh7wcw0hsj31ua0x6kjl.jpg)

发现对象的大小主要来源于对象中ConcurrentHashMap结构的locks变量的大小，而locks中存放的值就是本次内存泄漏的元凶，RedisLock对象。
那么这个locks是干什么的呢？接下来对源码进行分析:

![](http://tvax4.sinaimg.cn/large/005NahBely1goh7wwhlivj310q09cwio.jpg)

这是利用RedisLockRegistry中得到一个RedisLock的代码，其中会根据传进来的lockKey来判断locks中是否有这样的一个RedisLock对象。有则返回，无则创建，放入lock中并返回。
this.locks：

![](http://tvax4.sinaimg.cn/large/005NahBely1goh6s5yroqj312m03076g.jpg)

而这个lock，又恰好满足前面分析中内存泄漏元凶的结构，接下来对locks进行分析：

![](http://tvax4.sinaimg.cn/large/005NahBely1goh6vl7mpjj31oe04yju8.jpg)

代码中使用到locks的只有两个地方，一个是刚刚的obtain方法，一个则是expireUnusedOlderThan(),看方法名，这应该是把对象从locks中移出的方法：

![](http://tvax4.sinaimg.cn/large/005NahBely1goh7xz21fhj319i0f4n8f.jpg)

果然，expireUnusedOlderThan()方法，遍历locks中所有的RedisLock对象，将满足条件的lock从map中移出。

而判断应该移出的对象应同时满足：
1.当前时间戳 －上次获取锁的时间戳 > 传递进来的时间参数
2.当前对象并没有获取redis锁

那么接下来看一下，在什么地方使用了这个方法来释放对象呢：

![](http://tvax4.sinaimg.cn/large/005NahBely1goh78vbtncj317s0h0qlm.jpg)

好吧，其实只是提供了这个方法，但是并没有对locks进行释放。

那么问题已经很明显了，在使用RedisLock的过程中，虽然都有进行unlock释放锁，但是由于对象本身一直在map里面，没有显示释放，而在这个项目中所使用的lockKey是伴随时间不断更新的，这就会导致越来越多的RedisLock对象一直遗留在map中无法回收，最终导致OOM。

解决方案：
1.加一个定时任务移出locks中的无效对象
2.在每次unlock时，移出locks中的无效对象

总结：
1.内存快照属实有用，在出现OOM时可以帮助我们很快就定位到问题所在。启动参数时加上-XX:+HeapDumpOnOutOfMemoryError即可启用
2.在前期决定使用spring-integration做分布式锁时，在网上找的资料并没有对locks做什么介绍，在看源码时也忽略了这一点，关注点全在redis值相关的内容上，以后看源码还是要全面一点，不能人云亦云。没看到什么问题不代表没有问题，可能只是问题出现不频繁。