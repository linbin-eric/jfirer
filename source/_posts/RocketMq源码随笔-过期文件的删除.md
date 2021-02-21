---
title: RocketMq源码随笔-过期文件的删除
date: 2021-1-18 14:00:00
tags: RocketMq 源码解读
---
# RocketMq源码随笔-过期文件的删除

## 引言

RocketMQ中文件的存储是分为3个不同的部分：

+ CommitLog，提交日志。所有主题、队列的消息数据都是直接写入这一文件。
+ ConsumeQueue，消费队列。按照主题和队列的方式进行区分，消费队列中写入定长20字节的消费条目信息，消费条目中指向了该信息对应在提交日志中的偏移量。
+ IndexFile，索引文件。索引文件中写入定长20字节的索引信息，索引信息中指向了消息在提交日志中的偏移量。

RocketMQ不会无限制的将消息存储下去，而是采取一定的策略清除过期文件或者在磁盘不足的时候触发文件的删除，或者是手动的方式触发文件的删除。

在启动一个`Broker`的时候，会调用`BrokerController`的`start`方法，而这个`start`方法中也会调用到`DefaultMessageStore#start`方法。这个方法在最后会启动一个默认间隔为10秒的周期任务，这个周期任务中会调用`CleanCommitLogService#run`和`CleanConsumeQueueService#run`两个方法，实现提交日志、消费队列、索引文件的过期删除。

欢迎加入技术交流群186233599讨论交流，也欢迎关注技术公众号：风火说。
<!--more-->

## 提交日志的过期删除

提交日志的删除主要依托方法`CleanCommitLogService#run`。该方法内部调用了两个方法，分别是`deleteExpiredFiles`和`redeleteHangedFile`。前者用于删除过期的文件，后者用于再次可能存在的删除失败的文件（仅删除第一个）。

### deleteExpiredFiles

这个方法是用于删除过期文件。执行步骤如下：

1. 首先是需要判断是否需要删除文件，通过两个方法的调用`isTimeToDelete`和`isSpaceToDelete`判断是否达到定时删除时间以及是否磁盘已满需要删除，以及判断属性`DefaultMessageStore.CleanCommitLogService#manualDeleteFileSeveralTimes`是否大于0意味着需要手动删除。如果这三个条件任意为真，意味着需要执行删除，那就继续后续的流程。否则结束当前方法。
2. 如果是手动删除，则属性`DefaultMessageStore.CleanCommitLogService#manualDeleteFileSeveralTimes`减1.
3. 如果属性`MessageStoreConfig#cleanFileForciblyEnable`和`DefaultMessageStore.CleanCommitLogService#cleanImmediately`为真，声明cleanAtOnece为true，否则为false。
4. 调用方法`CommitLog#deleteExpiredFile`进行文件删除。方法需要4个入参，分别是：
   1. expiredTime：过期时间或者说文件删除前的保留时间，默认为72小时。
   2. deleteFilesInterval：文件删除间隔，这里取值为100.
   3. intervalForcibly：该参数用于强制文件强制释放时间间隔，单位是毫秒。这里取值为120*1000，
   4. cleanImmediately：是否立即执行删除，这边使用的就是步骤3中的数据。

### isSpaceToDelete

判断磁盘空间是否满足删除的条件，判断要求如下：


1. 使用提交日志的路径，检查其所在的磁盘空间的使用率。默认情况下，使用率超过90%，设置磁盘不可用标志位，并且设置属性`DefaultMessageStore.CleanCommitLogService#cleanImmediately`为true。使用率超过85%，设置属性`DefaultMessageStore.CleanCommitLogService#cleanImmediately`为true。其他情况，设置运行状态位为磁盘可用。
2. 磁盘使用率小于0或者大于属性`MessageStoreConfig#diskMaxUsedSpaceRatio`的要求，默认是75%，则返回true给调用。
3. 针对消费队列的文件路径，上述步骤重复一次。
4. 如果步骤1~3都没有返回true，则返回false给调用者。意味着此时磁盘空间有剩余，不要求删除。

### isTimeToDelete

RocketMQ会配置执行删除工作的时间，默认是早上四点。如果当前时间在04:00~04:59之间，就返回true。

## redeleteHangedFile

这个方法用于删除可能存在的应该被删除，但是没有删除成功的文件。执行逻辑如下：

1. 如果当前时间与上次执行重试删除`lastRedeleteTimestamp`的时间间隔120秒，则尝试执行重试删除。否则方法结束。
2. 为属性`lastRedeleteTimestamp`赋值为当前时间。
3. 调用方法`org.apache.rocketmq.store.CommitLog#retryDeleteFirstFile`执行重试删除。

## 消费队列的过期删除

`CLeanConsumeQueueService`的`run`方法就是直接委托这个方法来实现。这个方法的作用就是删除无效的消费队列条目内容或者文件本身。其代码逻辑如下：

1. 通过方法`CommitLog#getMinOffset`获取提交日志最小的偏移量，声明为minOffset。
2. 如果`minOffset`大于类属性`lastPhysicalMinOffset`，那么意味着当前提交日志的最小偏移量对比上一次查询的值发生了变化，也就是说必然是最少一个提交日志文件被删除，那么相应的在消费队列中的过期数据也可以被删除，就执行后面的流程。反之，则意味着不需要执行任何操作，结束方法即可。
3. 将`minOffset`赋值给`lastPhysicalMinOffset`。
4. 对属性`consumeQueueTable`进行遍历，遍历其中每一个`ConsumeQueue`对象。使用本次的`minOffset`作为入参，调用方法`ConsumeQueue#deleteExpiredFile`删除过期的消费队列文件以及更新消费队列的最小偏移量。如果有删除到文件，则休眠`MessageStoreConfig#deleteConsumeQueueFilesInterval`配置的时间，继续对下一个消费队列执行删除。
5. 当循环执行完毕，使用参数`minOffset`作为入参，调用方法`IndexService#deleteExpiredFile(long)`来删除索引文件中已经完全无效的索引文件。

## 索引文件的删除

索引文件的删除是在消费队列删除完成后，调用方法`deleteExpiredFile`完成的。

### deleteExpiredFile

该方法是用于删除索引文件中的无效文件。执行流程如下：

1. 首先需要确认，索引文件中是否存在无效文件。获取第一个索引文件，获取其`endPhyOffset`属性，判断该属性的值是否小于入参的`offset`。如果是的话，至少意味着有一个文件是无效的，则执行后续流程。否则没有无效文件，则直接结束整个方法。
2. 声明一个局部变量`fileList`，遍历索引文件`IndexFile`对象，如果其`endPhyOffset`小于入参的`offset`，说明该文件是无效的，添加到`fileList`中。
3. 使用第二步的`fileList`作为入参，调用方法`IndexService#deleteExpiredFile(List<IndexFile>)`。该方法内部调用了`IndexFile#destory`方法，内部也是委托了`MappedFile#destory`方法实现的文件销毁。并且删除成功的`IndexFile`还会从属性`indexFileList`列表中删除对应的对象。