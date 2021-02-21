---
title: RocketMq源码随笔-Broker的初始化
date: 2021-1-13 18:00:00
tags: RocketMq 源码解读
---
# RocketMq源码随笔-Broker的初始化

[TOC]

## 引言

Broker的初始化是Broker启动的第一个步骤。初始化的过程中会涉及到许多信息、配置的加载。日志、索引、消费队列信息的加载和恢复。

欢迎加入技术交流群186233599讨论交流，也欢迎关注技术公众号：风火说。
<!--more-->


## BrokerStartup

`Broker`的启动是依靠`BrokerStartup`方法。

首先是通过`BrokerStartup#createBrokerController`来创建一个`BrokerController`对象。

`createBrokerController`方法首先是一大串配置确认，转化、打印等。接下来是三个真正的创建步骤：

+ 创建`BrokerController`实例，并且加载读取到的外部配置信息。
+ 调用方法`BrokerController#initialize`执行初始化。
+ 添加 hook，在收到外部关闭指令的时候关闭 controller 实例。

## BrokerController

### 构造方法

这个类十分复杂，在构造方法中初始化了一大堆的管理类。这些管理类就随着后续使用到的时候一个个来说明好了。

### initialize

这个方法是初始化来看。只能一个个硬看了。

第一步，执行4个不同子类的load方法实现配置信息的加载，代码如下：

```java
boolean result = this.topicConfigManager.load();
result = result && this.consumerOffsetManager.load();
result = result && this.subscriptionGroupManager.load();
result = result && this.consumerFilterManager.load();
```

load方法很简单，就是读取配置文件的内容，然后将配置文件转化为字符串，并且传入子类抽象方法decode来实现各个子类自己的功能。

rocketmq 中这四个类都继承了这个基类，对于每一个具体的实现类而言，其decode的思路都是一致的。

首先是将字符串通过json方式转化为自身类的一个对象，然后将其中的属性赋值给自己的属性。

对于 TopicConfigManager 而言，这个属性就是类型为 ConcurrentMap<String, TopicConfig> 的 topicConfigTable。

对于 ConsumerOffsetManager 而言，这个属性就是类型为 `ConcurrentMap<String/* topic@group */, ConcurrentMap<Integer, Long>>`的offsetTable。

对于 SubscriptionGroupManager 而言，这个属性就是类型为ConcurrentMap<String, SubscriptionGroupConfig> 的 subscriptionGroupTable。

对于 ConsumerFilterManager 而言，这个属性就是类型为 ConcurrentMap<String/*Topic*/, FilterDataMapByTopic> 的 filterDataByTopic。

**第二步**，在第一步成功的情况下，为属性 `messageStore` 赋值。`MessageStore` 的赋值首先是创建 `DefaultMessageStore`对象，然后再加载可能存在的插件，通过插件对`MessageStore`进行更新，然后为其添加一个 `CommitLogDispatcher` 到自身的列表中（这部分代码待后续看）。同时也调用其`load`方法为自身加载信息。

**第三步**，第一步和第二步的加载流程都正确的情况下，开始进行启动工作。这部分的代码比较多，需要细分来看。

+ 启动一个NettyRemotingServer，使用 ClientHousekeepingService 实现类作为通道事件监听器。其实现就是在通道出现关闭、异常、空闲的情况下，关闭生产者、消费者、过滤器服务的对应通道。并且本身会创建一个后台周期任务，扫描生产者、消费者、过滤器服务的非活动链接。这个后台任务会在BrokerController执行start方法的时候启动。
+ 为属性 fastRemotingServer 复制，所有配置参考上面的RemotingServer，但是端口号比上面的小两位。从名字上去猜测，可能会更加快速的响应某一些请求。
+ 按照各自的配置信息，创建几个线程池，分别是：sendMessageExecutor、pullMessageExecutor、replyMessageExecutor、queryMessageExecutor、adminBrokerExecutor、clientManageExecutor、heartbeatExecutor、endTransactionExecutor、consumerManageExecutor。
+ 注册处理器。注册处理器的流程都是相似的，都是通过方法`NettyRemotingServer#registerProcessor`将处理器和对应的线程池绑定为一个Pair对象，并且将这个pair对象放入processorTable中，其值就是pair对象，key就是对应的请求编码。一共注册了有SendMessageProcessor、pullMessageExecutor、ReplyMessageProcessor、NettyRequestProcessor、ClientManageProcessor、ConsumerManageProcessor、EndTransactionProcessor、AdminBrokerProcessor。所有的处理器除了pullMessageProcessor外，都会在remotingServer和FastRemotingServer同时注册一遍。
+ 创建周期重复任务，周期性执行方法`BrokerStats#record`、`consumerOffsetManager#persist`、`consumerFilterManager#persist`、`BrokerController#protectBroker`、`BrokerController#printWaterMark`，此外还有一个定时任务是用来打印信息。
+ 为broker设置nameserver的地址。这里有两种形式。第一种是配置中直接指定了nameserver的地址。第二种是不指明nameserver的地址，但是给出一个地址服务器的url。那么会创建一个定时任务，周期性请求该地址，返回响应得到nameserver的地址，然后更新自身。nameserver的地址信息，是保存在brokerOuterAPI这个实例中。后续请求nameserver也是由这个实例来发起。后面再看具体的代码。
+ 在未启用多副本协议的情况下，根据当前broker的角色采取不同的高可用方式。如果是从角色，检查是否配置了HA地址，如果配置的话，更新到MessageStore中，并且设置属性`updateMasterHAServerAddrPeriodically`为false。否则的话，则设置属性`updateMasterHAServerAddrPeriodically`为true。如果非从角色，则周期性的调用方法`BrokerController#printMasterAndSlaveDiff`打印主从差距。
+ 判断TLS模式，执行对应逻辑，略过。
+ 执行方法`BrokerController#initialTransaction`。这个方法从实现来看，是为两个属性赋值，分别是`transactionalMessageService`和 transactionalMessageCheckListener。做法都是相似的，通过SPI机制实现读取，如果没有读取到的话，则使用各自框架中提供的默认实现。分别是TransactionalMessageServiceImpl 和 DefaultTransactionalMessageCheckListener 。这些类应该在讲到事务消息的时候会使用到。
+ 执行方法`BrokerController#initialAcl`，如果配置中开启了ACL（权限接入），通过SPI形式加载AccessValidator对象，将这些加载注册到RPCHook中，在请求之前执行校验。
+ 执行方法`BrokerController#initialRpcHooks`,通过SPI方式加载RPCHook对象，并且注册。同样会在请求执行的时候被调用。

到这里，初始化就完成了。

## DefaultMessageStore

### 初始化\构造方法

为该类中所有到的所有属性进行赋值。

并且执行了两个类的start方法，分别是AllocateMappedFileService和IndexService。后者的start方法是空的，前者的start方法是启动了这个线程。

### load

检查临时文件是否存在，并且将结果声明为lastExitOK。如果临时文件存在的话，则lastExitOK值为false。因为正常退出的情况下，这个临时文件应该是会被删除的。

接着是一系列的load过程。主要有：

+ 如果scheduleMessageService不为空，则执行其load方法。方法的作用是加载了配置文件，此外初始化好了延迟级别对应的毫秒数信息
+ 执行commitLog的load方法，将提交日志文件的信息加载到对象属性中。
+ 执行loadConsumeQueue方法，将消费队列文件的信息加载对象属性中，并且使用这些信息重建`consumeQueueTable`属性。

上述执行过程都成功的情况下，继续后续的流程。否则关闭allocateMappedFileService并且返回false代表失败。

后续的流程有：

+ 创建storeCheckpoint对象，从磁盘文件恢复检查点信息或者不存在的情况下创建文件和信息。
+ 执行方法`index.IndexService#load`，将索引文件信息载入内存，并且销毁掉最后写入时间大于检查时间的索引文件。
+ 执行方法`recover`.

最后返回本次load的结果情况。如果有一个方法返回失败或者执行异常，都返回false；都成功的情况下返回true。

### loadConsumeQueue

1. 获取consumequeue的文件夹路径，并且准备遍历文件夹下的所有元素。
2. 第一步文件夹下的所有元素都是以topic为名的文件夹，这个文件夹下的则是对应的ConsumerQueue文件夹。遍历这些文件夹，对每个文件夹执行下面的操作
   1. 将文件夹名转化为4字节整型数字，声明为queueId。
   2. 以topic，queueId，文件大小等信息创建ConsumerQueue对象。
   3. 将ConsumerQueue对象放入topic对应的queueId的键值对中。
   4. 执行ConsumerQueue的load方法进行加载。

### recover

该方法用于执行启动阶段的数据恢复工作。执行流程如下：

1. 执行方法`DefaultMessageStore#recoverConsumeQueue`对消费队列进行恢复。确认消费队列有效区域的最大偏移量，同时确认有效的消费队列中消费条目对应的提交日志的最大偏移量。声明为`maxPhyOffsetOfConsumeQueue`
2. 根据入参的`lastExitOk`，选择不同的提交日志恢复逻辑。如果为true，则执行方法`CommitLog#recoverNormally`;如果为false，则执行方法`CommitLog#recoverAbnormally`.这两个方法的入参都是第一步产生的`maxPhyOffsetOfConsumeQueue`。这个入参用于判断消费队列的指向是否超过了提交日志，如果是的话，则需要根据提交日志的最大偏移量删除掉消费队列中超出的这部分消费条目。
3. 执行方法`DefaultMessageStore#recoverTopicQueueTable`完成主题队列的恢复。

这个方法执行完毕后，提交日志，消费队列都恢复完毕。简单来说，就是用消费队列指向的提交日志的最大偏移量来确定提交日志的当前的最大偏移量。用提交日志的最小偏移量来确定消费队列本次的最小消费条目。

### recoverConsumeQueue

该方法会对属性`consumeQueueTable`进行遍历，遍历其中所有的`ConsumeQueue`对象，执行其`recover`方法。

之后，在所有的`ConsumeQueue`对象中，返回其中最大的`ConsumeQueue#maxPhysicOffset`属性给方法调用者。

### recoverTopicQueueTable

从方法名可以看出，这个方法还是用于恢复属性`org.apache.rocketmq.store.CommitLog#topicQueueTable`。其实现逻辑如下：

1. 执行方法`org.apache.rocketmq.store.CommitLog#getMinOffset`，获取当前提交日志的最小偏移量。将返回值声明为minPhyOffset。
2. 遍历所有的ConsumeQueue对象，将每一个ConsumeQueue的topic和queueId整合成key，调用方法`org.apache.rocketmq.store.ConsumeQueue#getMaxOffsetInQueue`（该方法使用文件的最大偏移量除以消费条目大小得到），获得返回值作为value，放入键值对中。同时执行方法`org.apache.rocketmq.store.ConsumeQueue#correctMinOffset`，入参为`minPhyOffset`。
3. 将步骤2产生的map设置为属性`CommitLog#topicQueueTable`.

### truncateDirtyLogicFiles

该方法用于从消费队列中删除超出有效范围的数据。做法是从属性`consumeQueueTable`获取所有的`ConsumeQueue`实例。使用入参的`phyOffset`调用方法`org.apache.rocketmq.store.ConsumeQueue#truncateDirtyLogicFiles`删除超出有效范围的无效数据。

## CommitLog

### load

该方法是委托方法`MappedFileQueue#load`进行实现，而方法`MappedFileQueue#load`的作用就是将commitLog路径下的文件进行全部的加载。

### recoverNormally

这个方法用于对提交日志的数据在重启后进行恢复和判断。整个执行逻辑如下：

1. 选择倒数第三个文件开始，进行数据检查和恢复。
2. 在循环中执行对MappedFile的检查。概括来说，就是选择MappedFile，获取其对应的ByteBuffer，不断的执行`checkMessageAndReturnSize`方法生成`DispatchRequest`对象。通过对`DispatchRequest`不断创建来检查`MappedFile`数据的有效性。一个`MappedFile`检测完毕就检测下一个文件，直到文件全部检查完或者发现无效数据区域。
3. 步骤2最终完成对文件有效区域的检查，并且得到最大有效区域的偏移量，也就是processOffset。将两个属性flushedWhere和CommitedWhere设置为processOffset。并且调用方法`org.apache.rocketmq.store.MappedFileQueue#truncateDirtyFiles`将超出有效区域的文件内容销毁。
4. 如果入参的`maxPhyOffsetOfConsumeQueue`大于等于processOffset，那就意味着消费队列存在冗余数据，调用方法`org.apache.rocketmq.store.DefaultMessageStore#truncateDirtyLogicFiles`删除超出processOffset的冗余数据。

上述的逻辑是在存在提交日志文件的情况下执行的。如果提交日志本身都不存在，则将flushedWhere、commitedWhere设置为0，调用方法`org.apache.rocketmq.store.DefaultMessageStore#destroyLogics`销毁消费队列数据。

### recoverAbnormally

这个方法用于在系统异常关闭的情况下，对提交日志进行有效区域的恢复。方法流程如下：

1. 从MappedFileQueue的最后一个文件开始向前确认，寻找开始恢复开始的文件。通过方法`isMappedFileMatchedRecover`来对`MappedFile`进行判断。
2. 读取这个文件的ByteBuffer，不断的生成DispatchRequest对象。并且针对有数据的`DispatchRequest`，为其执行方法`org.apache.rocketmq.store.DefaultMessageStore#doDispatch`重新分发数据，创建索引和消费条目信息。
3. 当一个文件读取完毕读取下一个文件直到没有文件可以读取或者文件上的有效区域都处理完毕。
4. 文件的最大有效区域的偏移量processOffset被设置到MappedFileQueue的flushedWhere和committedWhere。并且清除这之后的数据区域。
5. 如果入参的`maxPhyOffsetOfConsumeQueue`大于等于processOffset，意味着消费队列存在冗余条目，执行方法`org.apache.rocketmq.store.DefaultMessageStore#truncateDirtyLogicFiles`进行清除。

对比正常流程的恢复，异常流程下的恢复多了文件选择的不同，多了继续分发的这个操作。

## IndexService

### load

**第一步**，在路径`rootDir + File.separator + "index"`上加载所有的文件。

**第二步**，按照文件名进行升序排列。

**第三步**，遍历文件，使用每一个文件的信息，创建IndexFile对象。

**3.1**，执行IndexFile 的 load方法。

**3.2**，如果入参的 lstExitOk 为false，检查该IndexFile的最后写入时间，也就是endTimeStamp的值是否大于属性`StoreCheckpoint#indexMsgTimestamp`，如果是的话，则对该文件进行释放动作，并且忽略该文件。

**3.3**，上面步骤没有忽略该文件的话，则将IndexFile对象加入到属性`IndexService#indexFileList`中。

## IndexFile

索引文件，是用于存储消息的key在Commitlog中的偏移地址，方便快速定位某一个消息的物理位置的。索引文件自身分为三块区域：头部、哈希槽区域、索引区域。该文件的默认大小为头部40字节，哈希槽区域4\*5\*10<sup>6</sup>字节，索引区域20\*4\*5\*10<sup>6</sup>,约为400.5M。

### 格式

文件格式如下

| 序号 | 内容         | 长度                            |
| ---- | ------------ | ------------------------------- |
| 1    | 头部         | 40                              |
| 2    | 哈希槽位区域 | 4X槽位个数（默认为5000000）     |
| 3    | 索引区域     | 20X索引个数（默认为5000000\*4） |

头部格式如下

| 序号 | 内容                   | 长度 |
| ---- | ---------------------- | ---- |
| 1    | 第一消息的写入时间     | 8    |
| 2    | 最后一条消息的写入时间 | 8    |
| 3    | 第一条消息的物理偏移   | 8    |
| 4    | 最后一条消息的物理偏移 | 8    |
| 5    | 已用哈希槽位数         | 4    |
| 6    | 当前存储索引个数       | 4    |

哈希槽位是一个4字节整型变量，哈希槽位的值是属于该槽位的索引下标值。

索引区域存储的是索引数组，每一个索引的长度是20字节，一共存储了4个值，其格式如下。

索引的格式如下

| 序号 | 内容                                                         | 长度 |
| ---- | ------------------------------------------------------------ | ---- |
| 1    | key的哈希值                                                  | 4    |
| 2    | 该消息的物理偏移量                                           | 8    |
| 3    | 该消息的写入时间与第一个索引记录的写入时间的差值（单位是秒） | 4    |
| 4    | 使用同一个哈希槽位的上一个索引的索引下标                     | 4    |

序号4存储着同一个哈希槽位上的上一个索引的下标值，通过该值，就可以形成同一个槽位上的链表关系，也就能找到同一个槽位上的所有索引。而且由于存储的是下标值，因此需要一个值来代表非法下标，结束链表。在这里，rocketmq使用0来代表非法。这就同时要求索引的下标是从1开始。

从IndexFile的格式已经可以看出其使用方式。IndexFile是用于存储哈希数据的，其写入思路大致如下：

+ 首先计算出哈希槽位，读取哈希槽位上的值。
+ 将数据按照索引格式写入索引区域，并且将第一步读取的值当做同一个哈希槽上的前向元素的索引下标，一并写入索引区域。
+ 将本次使用的索引下标写入到第一步计算出的哈希槽位。
+ 系统的索引下标+1。

### load

该load方法是委托方法`IndexHeader#load`来执行。IndexHeader的load方法就是读取索引文件的头部区域，将文件内容读取到内存之中。