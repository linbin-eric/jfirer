---
title: RocketMq源码随笔-刷盘
date: 2021-1-10 18:00:00
tags: RocketMq 源码解读 高可用
---
# RocketMq源码随笔-刷盘

[TOC]

## 引言

在rocketmq中有两种刷盘模式：同步刷盘和异步刷盘。

![](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20201229133946.png)

从类图上来看，有三个不同的实现思路。那下面逐一来看过。

适用情况如下

+ 同步刷盘使用GroupCommitService。
+ 异步刷盘且未开启TransientStorePool，使用FlushRealTimeService。
+ 异步刷盘且开启TransientStorePool，使用CommitRealService。

欢迎加入技术交流群186233599讨论交流，也欢迎关注技术公众号：风火说。
<!--more-->

## GroupCommitService

GroupCommitService是在同步刷盘的时候会使用到的。

该类有两个属性：requestsWrite和requestsRead。都是使用volatile关键字修饰的List对象。

### putRequest

该方法用于将刷盘请求放入。首先是对requestWrite对象进行加锁处理，然后将请求放入其中后，解锁。解锁换成后，调用方法wakeup唤醒可能在休眠中的线程。

### doCommit

这个方法是在run方法中被调用。真正实现了数据的磁盘刷写。下面来看下逻辑。

+ 首先是在requestRead上进行加锁。
+ 如果requestRead队列不为空，则进行子流程刷盘操作。
  + 遍历requestRead列表，为每一个元素执行对应的逻辑，如下：
    + 调用方法`MappedFileQueue#getFlushedWhere`判断当前MappedFileQueue的刷盘偏移量是否大于等于该消息写入后的结束偏移量，将结果声明为flushOk。
    + 如果flushOK为false，就意味着还存在没有刷写到磁盘的区域，需要执行刷盘。调用方法`MappedFileQueue#flush`进行刷盘操作。
    + 如果flushOk为true，继续后续流程。如果flushOk为false，则再一次执行该子流程。这是因为在写消息的时候，可能会出现END_OF_FILE的情况。这种情况下就会将消息写入到新的文件中。因此第一次调用`MappedFileQueue#flush`会将倒数第二个文件刷写到磁盘。第二次调用该方法的时候才会选择到最新的MappedFile，执行刷写。在这种情况下就需要执行2次来保证刷写操作会执行到最新的文件上。
    + 两次或者一次刷写结束后，执行方法`CommitLog.GroupCommitRequest#wakeupCustomer`唤醒在该请求上等待的外部线程。至此，元素遍历的逻辑完成。
  + 获取属性`MappedFileQueue#storeTimestamp`，如果大于0，执行方法`StoreCheckpoint#setPhysicMsgTimestamp`设置检查点的写入时间。
  + 清空requestRead队列。
+ 如果requestRead队列是空的，执行方法`MappedFileQueue#flush`。
+ 释放锁。

## CommitRealTimeService

该类在异步刷盘且开启TransientStorePool被使用到。这个类的思路简单说是将写入在内存区域WriteBuffer中的内容提交到文件上。因此其主要逻辑就只是在run方法中。下面来看具体的细节。

+ 获取将执行提交逻辑的间隔时间，也就是属性`MessageStoreConfig#commitIntervalCommitLog`，声明为interval。
+ 获取WriteBuffer最少写入到文件上的页数，也就是属性`MessageStoreConfig#commitCommitLogLeastPages`，声明为commitDataLeastPages。默认值是4，按照4K一页的大小，最少一次写入到文件的数据是16K。
+ 获取将WriteBuffer写入到文件的最大间隔时间，也即是属性`MessageStoreConfig#commitCommitLogThoroughInterval`，声明为commitDataThoroughInterval。
+ 如果当前时间距离上一次提交时间超出了commitDataThoroughInterval，则为commitDataLeastPages赋值为0.也就是说，超出时间间隔的情况下，一定要执行一次写入，忽略写入数据大小的限制。
+ 调用方法`MappedFileQueue#commit`执行数据提交到磁盘工作。
+ 如果本次提交有写入磁盘数据，则为属性`lastCommitTimestamp`赋值当前时间戳。并且执行方法`FlushRealTimeService.wakeup`。因为MappedFIleQueue的commit方法只是内存数据提交到文件，没有执行force操作，没有强制刷到磁盘上。
+ 调用waitForRunning方法执行一定等待，等待时间为interval。

上述流程都是在run方法的while循环中。如果线程结束，还会尝试执行一定次数的`MappedFileQueue#commit`方法，尽可能将数据提交到文件。

## FlushRealTimeService

从名字可以看的出来，这个类是用于配合CommitRealTimeService，当然也可以自行独立使用。

类本身有两个属性：lastFlushTimestamp和printTimes。前者是存储最后一次刷盘的时间，后者是用于判断是否要在日志上输出刷盘信息。但是rocketmq将日志输出的内容去掉了，所以这个属性实际上是没有用了。该类的业务实现就在run方法中，下面来看下run方法的具体内容。

+ 获取属性`MessageStoreConfig#flushCommitLogTimed`，声明为flushCommitLogTimed。该属性的含义是指是否固定时间周期进行刷盘。默认为false。
+ 获取属性`MessageStoreConfig#flushIntervalCommitLog`，声明为interval。该属性意味着两次刷盘操作之间的最大休眠间隔。
+ 获取属性`MessageStoreConfig#flushCommitLogLeastPages`,声明为flushPhysicQueueLeastPages。该属性的含义是一次刷盘最少页数。
+ 获取属性`MessageStoreConfig#flushCommitLogThoroughInterval`，声明为flushPhysicQueueThoroughInterval。该值为最大刷盘间隔，也就是说超过这个时间，一定要执行一次刷盘，哪怕没有足够的数据。
+ 判断当前时间距离上次刷盘时间lastFlushTimestamp是否超过了flushPhysicQueueThoroughInterval。如果是的话，则为flushPhysicQueueLeastPages重新赋值为0.但是需要注意，lastFlushTimestamp的赋值和实际是否实际刷盘并没有关系。每次lastFlushTimestamp距离当前时间超过flushPhysicQueueThoroughInterval，才会被设置值，与此同时将flushPhysicQueueLeastPages设置为0.而flushPhysicQueueLeastPages设置为0，确保了后续调用方法`MappedFileQueue#flush`的时候，会更新`MappedFileQueue#storeTimestamp`属性。这其实意味着MappedFileQueue的存储时间和文件上的存储时间并一定相等。
+ 如果flushCommitLogTimed为true的，调用Thread.sleep实现休眠；否则调用方法waitForRunning实现休眠。区别在于前者在代码中没有提供打断的功能，那么刷盘就是一个周期性的定时任务。后者则可以被wakeup方法打断休眠，实时按照需要进行刷盘。
+ 休眠完成后，调用方法`MappedFileQueue#flush`进行刷盘。
+ 将属性`org.apache.rocketmq.store.MappedFileQueue#storeTimestamp`赋值给`org.apache.rocketmq.store.StoreCheckpoint#physicMsgTimestamp`。

上述流程是在一个while循环中被不断的执行，直到线程被停止。但线程被停止的时候，仍然会执行一定次数的flush操作，尽最大可能去将数据落盘。

## 总结

下面来做个简单的总结。

RocketMQ总体上存在两种刷盘模式：同步刷盘和异步刷盘。两种模式的区别在于，同步模式下，消息写入后，需要等待消息刷入硬盘，调用者才会返回，继续后续的流程；异步模式下，消息写入后，调用者即刻返回。后台线程会根据配置的时间周期间隔，执行文件刷盘动作。

而异步刷盘有两种细微的区分：优先写入内存区域的加速模式和无加速的默认模式。加速模式下，消息是写入到内存中，比在磁盘文件的映射上写入性能要更好。后台会根据配置的时间周期策略等将内存数据写入到文件，并且执行刷盘逻辑。

对于MappedFileQueue而言，其记录了两个属性：flushedWhere和committedWhere。前者代表着目前已经落盘的偏移量。后者代表已经提交到文件的偏移量。显然，后者这个属性只有在异步加速模式下才会使用到。

对于MappedFile而言，有三个属性：wrotePosition、committedPosition、flushedPosition。代表含义如下

+ wrotePosition：已经写入的内容的偏移量。这个偏移量可能是写入到文件也可能是写入到内存。
+ committedPosition：内存区域提交到文件的偏移量。该属性只有在异步加速模式下才会有用。
+ flushedPosition：已经刷入磁盘的偏移量。

