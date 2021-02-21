---
title: RocketMq源码随笔-索引文件与消费队列的创建
date: 2021-1-10 18:00:00
tags: RocketMq 源码解读
---
# RocketMq源码随笔-索引文件与消费队列的创建

[TOC]

## 引言

Broker在将消息写入到提交日志后，写入线程的动作就结束了。而Broker后台会运行一个`ReputMessageService`线程。该线程会不断的检查提交日志的内容，如果发现了新增的消息数据，则读取消息的数据内容，组装为DispatchRequest对象，通过接口方法`CommitLogDispatcher#dispatch`实现该消息的分发处理。

消息的分发处理，通过对该接口的不同实现来进行。不同的实现对应创建不同的内容。

欢迎加入技术交流群186233599讨论交流，也欢迎关注技术公众号：风火说。
<!--more-->

## ReputMessageService

该后台线程，不断的检查好提交日志，当发现新的数据内容的时候，就会构造用于分发消息的`DispatchRequest`对象。

这个类有一个唯一属性`reputFromOffset`，用于记录下一次从提交日志的什么位置进行数据处理。这个属性会在`DefaultMessageStore`的`start`方法中被设置。

### doPut

该类的run方法就是在一个死循环中调用本方法。

方法的一开始调用方法`CommitLog#getMinOffset`获取当前提交日志的最小偏移量。如果该值大于属性`reputFromOffset`，则将该属性赋值为日志的最小偏移量。

接下来就开始启动一个循环。循环声明了一个变量doNext，初始值为true。循环判断条件是doNext以及属性`reputFromOffset`小于提交日志的最大偏移量。循环逻辑如下：

1. 如果属性`config.MessageStoreConfig#duplicationEnable`为真，且`reputFromOffset`大于等于`CommitLog#confirmOffset`时，终止循环。否则继续后续流程。
2. 使用方法`CommitLog#getData(long)`，用reputFromOffset作为入参，获取对应的SelectMappedBufferResult，声明为result。
3. 为`reputFromOffset`赋值，值为属性`SelectMappedBufferResult#startOffset`。这步含义不明确，因为上面的代码操作下来，此时这两个属性的值应该是相等的。
4. 针对这个`result`对象，执行一个循环，读取其中的所有内容。

对于第四步中的循环，展开细说一下。

循环声明了一个整型变量`readSize`，判断条件是`readSize`小于属性`SelectMappedBufferResult#size`也就是result的内容大小，并且`doNext`为true，没有迭代条件，迭代条件是在循环体中完成。这个循环会持续到所有的提交日志内容都被处理完毕。下面来看循环的内容。

1. 使用方法`CommitLog#checkMessageAndReturnSize(java.nio.ByteBuffer, checkCRC, readBody)`创建`DispatchRequest`。在正常情况下，这将会读取到一个消息的数据。
2. 判断`DispatchRequest`中的`bufferSize`是否为-1，如果是的话，声明属性`size`，值为`msgSize`；否则声明属性`size`，值为`bufferSize`。
3. 如果`DispatchRequest`的`success`属性为`true`（正常读取消息该属性都是为`true`）则执行如下子流程。
   1. 如果`DispatchRequest`的`size`属性大于0，则执行如下子流程。
      1. 调用方法`DefaultMessageStore#doDispatch`对DispatchRequest对象进行分发。
      2. 如果broker的角色不是从节点，并且配置`org.apache.rocketmq.common.BrokerConfig#longPollingEnable`为真，则调用`org.apache.rocketmq.broker.longpolling.NotifyMessageArrivingListener#arriving`方法触发对应的监听器。该属性，如果是项目启动的话，从brokerController的init方法来看，应该是`NotifyMessageArrivingListener`类型。
      3. 令`reputFromOffset`增加`size`的值；`readSize`也是同理。
      4. 如果`broker`的角色是从节点，
   2. 如果`DispatchRequest`的`size`属性为0，意味着当前处理的提交日志文件已经到了末尾，需要处理下一个。执行如下的子流程。
      1. 令`readSize`等于循环外的`result`的大小。
      2. 调用方法`CommitLog#rollNextFile`计算当前偏移量的下一个提交日志的其实偏移量，赋值给reputFromOffset。
4. 如果`DispatchRequest`的`success`属性为false，意味着数据读取存在问题。执行如下逻辑。但是下面的逻辑意味着其实此时代码出现了bug，并不能实际的解决问题。因此可以认为下面的代码实际上是不会被触发到的。
   1. 如果`DispatchRequest`的`size`大于0，说明出现了bug，应该要读取到的数据长度和记录在消息中的数据长度不一致。令`reputFromOffset`加上size的值。
   2. 如果DispatchRequest的size等于0，令doNext为false。

**总结**

ReputMessageService的职责就是在循环中不断尝试读取到新写入到提交日志中的信息，并且执行分发逻辑。该类会从上次记录或者提交日志的最小偏移量开始执行读取并且分发。

## 索引文件的创建

索引文件的创建是依靠实现类`CommitLogDispatcherBuildIndex`来对`DispatchRequest`进行分发处理进而实现的。该类的实现内容也非常的简单，直接调用了`IndexService`的`buildIndex`方法。这个`IndexService`实例是在`DefaultMessageStore`初始化的也被初始化好。

### IndexService


#### load

**第一步**，在路径`rootDir + File.separator + "index"`上加载所有的文件。

**第二步**，按照文件名进行升序排列。

**第三步**，遍历文件，使用每一个文件的信息，创建IndexFile对象。

**3.1**，执行IndexFile 的 load方法。

**3.2**，如果入参的 lstExitOk 为false，检查该IndexFile的最后写入时间，也就是endTimeStamp的值是否大于属性`StoreCheckpoint#indexMsgTimestamp`，如果是的话，则对该文件进行释放动作，并且忽略该文件。

**3.3**，上面步骤没有忽略该文件的话，则将IndexFile对象加入到属性`IndexService#indexFileList`中。

#### buildIndex

该方法用于创建索引信息。下面是方法的执行流程。

1. 执行方法`retryGetAndCreateIndexFile`获取对应的IndexFile对象用于稍后的索引创建。如果方法返回的`indexFile`为null，就意味着获取最新的索引文件或者创建新的索引文件出现了异常，结束当前方法。否则执行后续流程。
2. 获取`IndexFile`对象的`endPhyOffset`值，声明为`endPhyOffset`。如果该值小于属性`DispatchRequest#commitLogOffset`，则说明该文件的信息已经建立索引，流程结束，方法返回。如果不是的话，继续后续流程。
3. 检查属性`DispatchRequest#sysFlag`，如果其拥有标志*TRANSACTION_ROLLBACK_TYPE*，则结束方法，直接返回。否则继续后续流程。
4. 如果属性`DispatchRequest#uniqKey`不为null，调用方法`IndexService#putKey`进行索引添加。如果方法返回null，意味着所以没有创建成功，方法结束并且返回null。
5. 如果属性`DispatchRequest#keys`不为null，则使用分隔符*KEY_SEPARATOR*对该属性进行分割，分割后遍历每一个元素，为每一个元素执行方法`IndexService#putKey`。与第四步相同，如果方法执行的之后返回null，则直接结束整个方法比并且返回。

概括来说，整个思路就是在索引文件中放入索引信息。如果索引文件满了，则创建新的文件，然后放入。

#### putKey

这个方法的实现很简单，首先是调用入参的`indexFile`的`putKey`方法。如果放入失败，意味着索引文件已经满了。则通过`retryGetAndCreateIndexFile`方法创建新的`IndexFile`。如果创建失败就返回null。如果创建成功，则使用新的`IndexFile`执行`putKey`方法。

放入成功后，将刚才放入成功的`IndexFile`对象返回。可能是入参对象，也可能是方法中新创建的。

#### retryGetAndCreateIndexFile

这个方法就是调用`getAndCreateLastIndexFile`方法来获得合适的IndexFile。如果方法调用的时候返回null，则最多尝试*MAX_TRY_IDX_CREATE*限定的次数。

如果超过尝试次数仍然无法获取到`IndexFile`实例，则获取`DefaultMessageStore`的`runningFlags`实例，设置其*WRITE_INDEX_FILE_ERROR_BIT*标志位。表明当前运行状态出现了创建索引文件异常。

最后返回获取到的`IndexFile`或者是`null`，在没有获取到的情况下。

#### getAndCreateLastIndexFile

从类名可以看到，这个方法用于获取或者创建最新的`IndexFile`，下面来看下流程。

第一步，尝试获取当前最新的`IndexFile`。执行流程如下：

1. 获取`readWriteLock`的读锁。如果`indexFileList`不为空，执行后续流程。否则释放锁，结束整个步骤。
2. 获取最后一个`IndexFile`。如果该`IndexFile`还有剩余空间供写入，将其值赋值给临时变量`indexFile`。否则的话，将该值赋值给临时变量`prevIndexFile`，并且用这个文件的值分别为`lastUpdateEndPhyOffset`、`lastUpdateIndexTimestamp`进行赋值。
3. 对`readWriteLock`进行解锁。

第二步，就是对第一步的结果进行处理。如果`indexFile`变量不为null，则直接返回该变量给调用者。如果`indexFile`为null，则意味着需要创建新的索引文件。创建逻辑如下：

1. 准备创建索引文件。创建路径为\${user.home}/store/index/${currentTimeMillis}，也就是说索引文件是以创建时间为文件名命名的。
2. 创建`IndexFile`对象实例，使用`lastUpdateEndPhyOffset`和`lastUpdateIndexTimestamp`两个临时变量作为入参。
3. 对`readWriteLock`加写锁，往`indexFileList`队列中加入本次创建的`IndexFile`对象实例。
4. 如果`IndexFile`对象实例有创建成功，启动一个线程，使用第一步中赋值的`prevIndexFile`变量作为入参，调用方法`IndexService#flush`进行索引文件的磁盘刷写。该线程仅会执行一次该方法就结束。
5. 将创建的`IndexFile`对象返回给调用者。

#### flush

对索引文件进行刷盘动作，执行流程如下：

1. 声明变量`indexMsgTimestamp`，初始值为0.如果入参的`IndexFile`已经没有剩余空间，则将入参的`endTimestamp`值赋值给该变量。
2. 执行`IndexFile#flush`方法，将索引文件进行刷盘。
3. 如果`indexMsgTimestamp`大于0，则用该值设置属性`StoreCheckpoint#indexMsgTimestamp`。并且调用方法`StoreCheckpoint#flush`将检查点的数据刷盘。

### IndexFile

索引文件，是用于存储消息的key在Commitlog中的偏移地址，方便快速定位某一个消息的物理位置的。索引文件自身分为三块区域：头部、哈希槽区域、索引区域。该文件的默认大小为头部40字节，哈希槽区域4\*5\*10<sup>6</sup>字节，索引区域20\*4\*5\*10<sup>6</sup>,约为400.5M。

#### 格式

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

#### load

该load方法是委托方法`IndexHeader#load`来执行。IndexHeader的load方法就是读取索引文件的头部区域，将文件内容读取到内存之中。

#### putKey

该方法是将key值写入索引文件，流程如下

+ 判断属性`IndexHeader#indexCount`是否索引个数限制。如果不小于的话，流程失败，方法直接返回false。
+ 对key求取哈希值，使用哈希值计算出槽位下标，使用槽位下标计算出该槽位在该磁盘文件上的偏移量，声明为absSlotPos。
+ 在索引文件的absSlotPos位置上读取该哈希槽位的值，声明为slotValue.如果该值小于等于0或者大于indexCount,则重新赋值为0.
+ 计算该消息写入磁盘时间与索引文件开始时间的差额,声明为timeDiff,单位是秒。如果索引文件的beginTimeStamp小于等于0，将timeDiff重新赋值为0；如果timeDiff大于Integer.MAX_VALUE，则重新赋值为Integer.MAX_VALUE;如果timeDiff小于0，则重新赋值为0。
+ 使用indexCount计算当前待写入索引在索引文件上的偏移量，声明为absIndexPos 。
+ 对这个索引块写入值，分别是key的哈希值，该消息的物理偏移量，timeDiff，slotValue。
+ 在absSlotPos位置，也就是哈希槽，写入本次索引的下标indexCount。
+ 如果indexCount小于等于1，则设置indexHeader的beginPhyOffset和BeginTimeStamp。
+ 如果slotValue等于0，则indexHeader中的槽位使用总数，hashSlotCount加1 。
+ indexHeader的indexCount加1，设置其endPhyOffset和endTimestamp的值 。
+ 返回成功。

#### selectPhyOffset

该方法是通过索引文件查询消息的物理位置。知道了索引文件的内容布局，查找消息的位置信息就很简单了。

思路上来说先通过key的哈希值找到哈希槽位，读取槽位上的值找到索引块，访问索引块形成的列表找到合适的索引块就可以了。

下面来看下具体的代码逻辑：

+ 对消息的key求取哈希值，根据哈希值确定哈希槽位，根据槽位计算出该槽位在索引文件上的偏移地址，声明为absSlotPos。读取该槽位的值，声明为slotValue。
+ 如果slotValue的值非法（小于等于0，大于indexCount或indexCount本身小于等于1），则流程结束，本次未找到对应的消息。
+ 声明变量nextIndexToRead，将slotValue赋值给该变量。
+ 根据nextIndexToRead求取该索引在索引文件上的偏移地址。读取该索引块的数据内容。
+ 根据timeDiff和indexHeader的beginTimeStamp计算出该消息的写入时间，声明为timeRead。如果timeRead在入参限制的时间范围内并且入参key的哈希值与该索引存储的哈希值相等，则意味着该索引满足要求，将该索引存储的消息偏移地址放入入参的phyOffsets列表中。
+ 如果该索引存储的前向索引下标不是非法的话，则将该索引下标再次赋值给nextIndexToRead，重复该流程。

#### flush

将索引文件进行刷盘操作。执行流程如下

1. 执行方法`IndexHeader#updateByteBuffer`将`indexHeader`的数据再一次写入到索引文件的MappedByteBuffer上。不过这一步只是为了保险，实际上在操作的过程中，每一次都会执行写入。
2. 调用`ByteBuffer`的`force`方法进行刷盘。

## 消费队列的创建

消费队列的创建是通过实现`CommitLogDispatcherBuildConsumeQueue`来进行的。该类的实现很简单，首先是检查标志属性`DispatchRequest#sysFlag`，如果标志位是*TRANSACTION_NOT_TYPE*或*TRANSACTION_COMMIT_TYPE*，调用方法`DefaultMessageStore#putMessagePositionInfo`。其余情况不处理。 

### DefaultMessageStore

#### putMessagePositionInfo

首先执行方法`DefaultMessageStore#findConsumeQueue`来获取对应主题对应queueId的ConsumerQueue对象实例。

然后在获得的`ConsumerQueue`对象上执行方法`putMessagePositionInfoWrapper`，来处理`DispatchRequest`对象。

#### findConsumeQueue

该方法用于获取对应主题对应queueId的ConsumerQueue对象实例。整体方法流程如下

1. 使用主题从属性`org.apache.rocketmq.store.DefaultMessageStore#consumeQueueTable`获取对应的queueId和ConsumerQueue的映射Map对象。如果这个对象不存在，则创建该对象，并且放入`consumerQueueTable`。中。
2. 从第一步获得的映射中，使用queueId来查询对应的`ConsumerQueue`对象。如果不存在，则创建新的`ConsumerQueue`对象，并且放入映射Map中。返回查询到或者创建的`ConsumerQueue`对象。

### ConsumerQueue

#### 初始化\构造方法

将入参的topic和queueId赋值给自身的私有化参数。

使用这两个参数构成的文件夹路径初始化一个MappedFileQueue。

一个索引文件默认配置下包含300000个索引条目，每个条目为20个字节，默认索引文件大小约为5.7M。索引文件的文件名为该文件的起始偏移量。

#### load

load方法调用初始化中构建的MappedFileQueue的load方法。

#### putMessagePositionInfoWrapper

该方法用于处理`DispatchRequest`对象。在检查系统状态后，确认可以对`ConsumerQueue`文件进行写入的情况下，调用方法`putMessagePositionInfo`来将数据写入。

如果写入失败，则休眠后继续尝试，默认最多可以尝试30次。30次之后还失败，则标记系统运行状态为消费队列异常，方法返回。

如果写入成功，为属性`StoreCheckpoint#logicsMsgTimestamp`赋值为`DispatchRequest#storeTimestamp`。与此同时，如果broker角色为从节点，则为属性`StoreCheckpoint#physicMsgTimestamp`也设置相同的值。属性值设置完毕后，方法返回。

#### putMessagePositionInfo

该方法用于在消费队列中写入消费条目信息。流程如下

1. 将入参的offset和size相加后判断是否小于等于属性`ConsumeQueue#maxPhysicOffset`。如果是的话，意味着也许正在重建消费队列。返回true，结束方法。否则继续下面流程。
2. 在属性`byteBufferIndex`中准备好本次消费条目写入的内容。为消息的偏移量、消息的大小、消息的`tagsCode`值（消息tags的hashcode）。
3. 使用入参的`cqOffset`计算预期的偏移量，声明为`expectLogicOffset`。
4. 使用这个`expectLogicOffset`这个偏移量获取对应的`MappedFile`对象。如果没有获取到，返回false，结束方法。否则继续执行下面的流程。
5. 如果该`MappedFile`是`MappedFileQueue`中的第一个文件，`cqOffset`不为0切该`MappedFile`的`wrotePosition`的值为0，则执行子流程的内容。否则继续后续流程。
   1. 使用`expectLogicOffset`为属性`ConsumeQueue#minLogicOffset`赋值。
   2. 使用`expectLogicOffset`为属性`MappedFileQueue#flushedWhere`赋值。
   3. 使用`expectLogicOffset`为属性`MappedFileQueue#flushedWhere`赋值。
   4. 调用方法`fillPreBlank`来为MappedFile填充前向的区域，填充到`expectLogicOffset`偏移量。
6. 如果cqOffset不为0，则执行子流程；否则继续后续流程。
   1. 将当前`MappedFile`的写入偏移量加上文件的起始偏移量，声明为`currentLogicOffset`。
   2. 如果上面步骤的`expectLogicOffset`小于`currentLogicOffset`。说明当前重复创建消费条目。忽略本次创建。返回true给调用者，结束本次流程。
   3. 如果如果上面步骤的`expectLogicOffset`不等于`currentLogicOffset`，说明当前存在异常。可能是文件被意外删除。不过RocketMq这里仅仅是输出了错误日志，没有其他处理。
7. 将该消息的`offset`和`size`相加，赋值给属性`ConsumeQueue#maxPhysicOffset`。
8. 调用方法`MappedFile#appendMessage(byte[])`将消费条目的内容写入到文件中，并且将写入的结果返回。如果文件剩余空间足够，是会返回true。返回false，意味着当前文件已经满了。不过正常而言，上面的方法中选择`MappedFile`的时候，就应该已经是选择到了一个有空间的`MappedFIle`，因此不会出现没有空间可以写入的情况。

#### fillPreBlank

这个方法构建了一个特殊的索引数据块。也就是偏移量0，消息大小*Integer.MAX_VALUE*，消息tagsCode为0.

然后用这个索引块不断写入入参的`MappedFile`，直到到达入参要求的偏移量。



