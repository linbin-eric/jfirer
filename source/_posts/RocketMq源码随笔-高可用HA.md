---
title: 高可用HA
date: 2021-1-10 18:00:00
tags: RocketMq 源码解读 高可用
---
# RocketMq源码随笔-高可用HA

[TOC]

## 引言

RocketMq在部署的时候对高可用的考虑有两种模式：一种是消息数据的复制，一种是基于选择的主节点确定（PS：2021-1-10尚未确定，这部分代码未看）。

下文是对复制模式的代码随笔解读。

欢迎加入技术交流群186233599讨论交流，也欢迎关注技术公众号：风火说。
<!--more-->


## HAService

### putRequest

委托给方法HAService.GroupTransferService#putRequest去实现。

### notifyTransferSome

该方法主要是为了对GroupTransferService中的等待线程执行唤醒。通过CAS操作，将入参的offset的值尝试写入push2SlaveMaxOffset，前提是入参的值大于当前值。

一旦CAS成功，则执行方法`HAService.GroupTransferService#notifyTransferSome`唤醒在GroupTransferService上等待的线程。

如果在循环CAS尝试中，发现入参的值已经小于等于push2SlaveMaxOffset，则放弃循环，退出方法。

## GroupTransferService

在HaService初始化的时候被实例化。类本身继承了ServiceThread，也是一个后台运行的线程。在HaService执行start方法的时候被启动。

GroupTransferService中有读写两个队列。

putRequest会将请求放入写队列中，并且唤醒线程。而线程被唤醒后，会将读写队列互换，这样就得到了一个只有数据可以被读取的读队列。

遍历读队列的内容，为每一个元素都执行如下逻辑：

+ 判断属性`HAService#push2SlaveMaxOffset`是否大于等于CommitLog.GroupCommitRequest#nextOffset，将结果声明为 transferOK。显然，前者大于等于后者的时候，意味着数据都已经传输到从节点完毕了。
+ 将当前时间加上`config.MessageStoreConfig#syncFlushTimeout`生命为waitUntilWhen。
+ 如果transferOK为false且当前时间未到达waitUntilWhen，这反复调用方法`WaitNotifyObject#waitForRunning`执行等待，并且重新为transferOK赋值。赋值后继续该流程直到超时或者transferOK为true。
+ 调用方法`CommitLog.GroupCommitRequest#wakeupCustomer`唤醒在request上等待的线程。

遍历结束后，清空读队列。

从代码的内容上来看，这个类本身和数据的同步无关，仅仅起到了等待的作用。

## AcceptSocketService

这个类在HaService初始化的时候被实例化，类本身继承了ServiceThread，也是一个后台运行的线程。在HaService执行start方法的时候被启动。

这个类在初始化的时候会使用属性`config.MessageStoreConfig#haListenPort`作为监听端口，用于监听从节点接入的请求。

在HaService调用start方法启动的时候，也会调用本类的beginAccept方法。在这个方法中就会使用NIO的接口，打开一个ServerChannel，并且在对应的端口上监听接入请求。

### run

run方法的实现就是在一个循环中监听服务通道的accept事件，当有客户端接入的时候，就包装为一个HaConnection对象，并且执行其start方法。

与此同时，将该HaConnection对象加入到`HAService#connectionList`属性中。

## HaConnection

使用HaService和SocketChannel作为入参进行初始化。

初始化的时候会对属性`HAService#connectionCount`执行加1操作。

然后对SocketChannel进行一系列的基本属性设置，主要是将其设置为非阻塞模式，并且后续会使用NIO的接口对这个通道进行读写。

HaConnection内部定义了两个分别用于读写的线程对通道上的数据进行处理，分别是WriteSocketService和ReadSocketService。两个类均继承了ServiceThread，并且在HaConection的start方法中被启动。

### ReadSocketService

该类会创建一个Selector，监听给定通道上的读事件。

该类定义了几个私有属性：

+ byteBufferRead，用于在通道上读取数据。
+ processPosition，用于记录当前已经处理的数据的偏移量。
+ lastReadTimestamp，用于记录该通道上最后一次读取到数据的时间。

该类的run方法的实现逻辑很简单，在一个死循环中，在selector上执行超时等待。当`selector.select`方法返回的时候，调用方法`processReadEvent`处理可能读取到的数据。

当处理读取数据错误（通道关闭或者发生异常）或者当前时间与上次数据读取差距过大时（客户端超时），都会终止循环。

终止循环后会执行以下操作：

+ 调用自身和writeSocketService的makeStop方法，设置停止标志位。
+ 调用方法`HAService#removeConnection`将连接删除。
+ 属性`HAService#connectionCount`减1.
+ 执行选择器取消，通道关闭等操作。

#### processReadEvent

该方法用于处理在通道上读取到的数据。首先是声明了一个局部变量readSizeZeroTimes，该变量用于记录灭有读取到数据的次数。也就是说在一定次数读取不到数据的时候，该方法就会结束。这种设计用于尽最大努力来一次性读取数据。

方法的整体逻辑如下

+ 调用方法`java.nio.Buffer#hasRemaining`确认byteBufferRead是否没有剩余数据。如果没有剩余数据，则执行方法`java.nio.Buffer#flip`恢复到初始可写状态。并且将processPosition重新赋值为0.
+ 执行循环判断，判断条件为`java.nio.Buffer#hasRemaining`是否为true。循环内部逻辑如下
  + 执行方法`java.nio.channels.SocketChannel#read(java.nio.ByteBuffer)`读取数据到byteBufferRead，并且将读取到的字节数声明为readSize。
  + 如果readSize为0，则readSizeZeroTimes加1。如果readSizeZeroTimes大于等于3，则结束循环。否则继续循环。
  + 如果readSize为-1，则意味着通道关闭，返回false给方法调用者。
  + 如果readSize大于0，则意味着有数据可以读取并且处理。处理逻辑如下：
    + 将readSizeZeroTimes重新赋值为0.
    + 将lastReadTimestamp赋值为当前时间。
    + 如果byteBufferRead的position与processPosition差值小于8则开始下一次循环。否则继续后续流程。
    + 使用byteBufferRead的position求取最接近其值的8的整倍数的值，声明为pos。
    + 使用byteBufferRead，在pos-8的位置读取一个long变量，声明为readOffset。
    + 将processPosition赋值为pos。
    + 将属性`HAConnection#slaveAckOffset`赋值为readOffset。
    + 如果属性`HAConnection#slaveRequestOffset`小于0，则将其赋值为readOffset.
    + 调用方法`HAService#notifyTransferSome`将slaveAckOffset作为参数传入。这将唤醒在外部等待数据同步的线程。

### WriteSocketService

这个类用于向通道中写入当前commitlog的数据。

对象在初始化的时候会声明几个属性：

+ byteBufferHeader，固定为12字节大小。前8字节是个long整型数字，代表传输偏移量；后4个字节为int整型数字，代表本次传输的内容体长度。
+ nextTransferFromWhere，下一次传输的内容在文件上的偏移量。
+ selectMappedBufferResult，根据传输偏移量，对选中的文件传输区域的包装对象。
+ lastWriteOver，上一次socket写出数据是否完毕
+ lastWriteTimestamp，上一次数据写出时间。

这个类的处理逻辑都在其run方法中。下面来看run方法的具体内容。

run方法的一开始是一个while死循环，用于不断的等待写事件的触发。循环的处理逻辑如下

+ 在selector上执行select超时等待，超时时间为1秒。方法返回后进入后续流程。
+ 判读属性`HAConnection#slaveRequestOffset`是否等于-1，如果是的话，休眠10毫秒，回头循环开始。否则进入后续流程。
+ 判断属性`HAConnection.WriteSocketService#nextTransferFromWhere`是否等于-1。等于-1意味着之前没有传输过，需要首先确定从哪一个位置开始启动传输，也就是子流程的作用。如果不等于-1，则直接从上次的位置开始启动传输即可。
  + 判断属性`HAConnection#slaveRequestOffset`是否等于0.如果不是的话，将属性`HAConnection.WriteSocketService#nextTransferFromWhere`赋值为`HAConnection#slaveRequestOffset`。如果是的话，进入子流程。
    + 调用方法`CommitLog#getMaxOffset`获取当前文件上写入值的最大偏移量。这个偏移量写入到文件中内容的偏移量。不是刷盘的偏移量。将该偏移量生命为masterOffset。
    + 将masterOffset进行修正，将masterOffset修正到最接近`mappedFileSizeCommitLog`整倍数的数字。如果masterOffset小于0，则赋值为0。
    + 将nextTransferFromWhere重新赋值为masterOffset。
+ 判断lastWriteOver是否为true，代表着上一次数据是否写出完毕。如果写出完毕，执行子流程。
  + 获取当前时间和lastWriteTimestamp的差，声明为interval。
  + 如果interval大于配置定义的心跳发送间隔，则使用byteBufferHeader组装发送数据，偏移量是nextTransferFromWhere，内容体长度是0.调用方法transferData发送数据，并且将结果赋值给lastWriteOver。
  + 如果lastWriteOver为false，则回到循环开始处，继续下一次循环。
+ 如果判断lastWriteOver为false，则直接执行transferData发送数据，并且将结果赋值给lastWriteOver。如果该结果为false，则回到循环的开始处，继续下一次循环。
+ 方法到了这里，就准备好发送本次要传输的文件内容。执行方法`DefaultMessageStore#getCommitLogData`,以nextTransferFromWhere为入参，得到类型为SelectMappedBufferResult的返回结果。结果中包含了从偏移量为nextTransferFromWhere开始的文件可读取内容的ByteBuffer对象。将这个结果声明为临时变量selectResult。
+ 如果selectResult不为null（只有在线程关闭或者偏移量超出文件本身的时候才会是null），执行下列子流程。
  + 获取selectResult中的byteBuffer的大小，声明为size。如果size大于配置`config.MessageStoreConfig#haTransferBatchSize`定义的值，则修正到这个值。
  + 将nextTransferFromWhere赋值给临时变量thisOffset，nextTransferFromWhere自增size。
  + selectResult中的ByteBuffer的大小被重新设置为size。将selectResult赋值给selectMappedBufferResult。
  + 构建传输头数据，偏移量是thisOffset，也就是上一次的nextTransferFromWhere。传输长度是size。
  + 调用transferData方法传输数据，并且将结果赋值给lastWriteOver属性。
+ 如果selectResult为null，则获取haService的waitNotifyObject实例，调用其`allWaitForRunning`方法执行等待。

以上就是整个while循环的逻辑过程。当从循环中退出的时候，就是线程结束的时候，主要是执行一些清理的动作，比如停止自身，停止读线程，从HaService中移除当前的HaConnection，关闭通道，释放可能没有写完的SelectMappedBufferResult，从HaService的waitNotifyObject中移除自身线程。

## HaClient

该类主要是用于向主节点请求数据，本身定义了一些属性用于控制同步信息。如下：

+ currentReportedOffset。向主节点汇报的当前从节点的偏移量值。在通道建立的时候被设置为当前提交日志的写入偏移量；从节点更新提交日志完成，准备汇报主节点偏移量前对该值进行更新。
+ dispatchPosition。byteBufferRead中待处理区域的起点偏移量。会在通道关闭，或者byteBufferRead被重分配的时候设置为0；每当处理完一个消息，dispatchPosition会增加这个消息的长度。
+ lastWriteTimestamp。上一次和主节点发送消息成功的时间点。会在连接创建和向主节点汇报自身偏移量的时候被更新为当前时间值；会在连接关闭的时候被设置为0.

该类本身也是个后台线程，在HaService的start方法中被启动。来看下run方法的逻辑，如下

+ 通过方法connectMaster判断是否已经连接上主节点。如果没有的话，通过waitForRunning方法休眠5秒后继续循环。
+ 调用方法`isTimeToReportOffset`判断当前是否反馈从节点自身的偏移量。如果需要的话，调用方法`reportSlaveMaxOffset`上报当前自己的偏移量。如果上报失败，则调用closeMaster方法关闭连接。
+ 在选择器返回后，调用processReadEvent处理读取数据。
+ 如果上面处理读取数据的方法返回false，调用closeMaster方法关闭通道。
+ 调用方法`HAClient#reportSlaveMaxOffsetPlus`汇报从节点偏移量增长。如果返回false，则回到循环开始，重新下一次循环。
+ 判断当前时间和`HAClient#lastWriteTimestamp`的差距是否大于配置`MessageStoreConfig#haHousekeepingInterval`的值。如果大于，意味着长时间没有数据交换，关闭当前通道。

### connectMaster

如果属性socketChannel为null，则尝试创建连接对象。通过属性`HAService.HAClient#masterAddress`获取主节点地址，创建通道对象，并且在选择器上注册这个通道的读取事件。完成后为属性`HAService.HAClient#currentReportedOffset`赋值，其值通过方法`DefaultMessageStore#getMaxPhyOffset`来获得。将lastWriteTimestamp赋值为当前时间。

返回通道对象是否为null给调用者。

### isTimeToReportOffset

获取当前时间与lastWriteTimestamp的差距。如果这个差距大于心跳发送间隔，则返回true给调用者。说明此时需要发送心跳信息。

### reportSlaveMaxOffset

清空reportOffset，在reportOffset中写入long整型数字，也就是当前从节点的偏移量。

尝试将reportOffset的内容写入通道中，最多尝试3次。如果在写入通道过程中出现异常，则返回false。

将lastWriteTimestamp赋值为当前时间。

返回reportOffset是否已经完全写入的结果给调用者。

### closeMaster

关闭连接，取消当前通道在选择器上的注册，将属性socketChannel设置为null。将lastWriteTimestamp和dispatchPosition都重置为0.

### processReadEvent

使用byteBufferRead从通道中读取数据。如果读取到了数据，就调用方法`dispatchReadRequest`进行处理。如果没有读取到数据，则尝试再次读取并且为当前进行计数。连续三次从通道读取不到数据，则结束方法。否则尝试不断读取数据。

### dispatchReadRequest

该方法实现了对读取到的数据进行处理的逻辑。总体上来说，是在一个循环中不断读取数据并且实现处理。下面我们看下一个循环中的处理逻辑，如下：

+ 判断byteBufferRead的位置与属性`dispatchPosition`的差值是否大于等于12，因为12是主节点一次发送消息的最小长度。如果没有的话，就需要进行数据整理，然后可以退出循环了。下面来看下满足12的情况下，会执行的后续逻辑。
+ 在byteBufferRead的dispatchPosition位置上读取主节点推送偏移量和本次传输内容体长度。分别声明为masterPhyOffset和bodySize。
+ 调用方法org.apache.rocketmq.store.DefaultMessageStore#getMaxPhyOffset获取当前文件的物理偏移量，声明为slavePhyOffset。
+ 如果salvephyOffset不为0，则判断其是否与masterPhyOffset相等。如果不等的话，就说明主节点要推送的数据与从节点需要接收的数据有偏差，终止流程，并且返回false给调用者。
+ 如果第一步求取的差值大于等于消息头部和消息体的长度，意味着本次读取中获得了一个完整的消息。此时就可以将读取到的消息体写入到从节点的本地文件中。
  + 从byteBufferRead中读取消息内容体，声明为bodyData。
  + 调用方法org.apache.rocketmq.store.DefaultMessageStore#appendToCommitLog，以slavePhyOffset和bodyData作为入参，将数据写入到提交文件。
  + dispatchPosition增加本次消息的总体长度。
  + 调用方法reportSlaveMaxOffsetPlus上报从节点偏移量新增。如果方法返回false，则结束整个方法，并且返回false。否则继续流程。
  + 回到循环最开始，执行下一轮循环。
+ 如果第一步的判断中，可以读取的内容长度小于12或者本次没有读取到完整的消息体，都会走到这一步。判断byteBufferRead是否有剩余，如果没有的话，执行方法reallocateByteBuffer对byteReadBuffer进行整理。
+ 退出当前循环。
+ 返回true到方法调用者。

### reportSlaveMaxOffsetPlus

该方法用于汇报从节点的偏移量增长。

+ 通过方法`org.apache.rocketmq.store.DefaultMessageStore#getMaxPhyOffset`获取当前的偏移量。声明为currentPhyOffset。
+ 如果currentPhyOffset大于属性currentReportedOffset，则执行子流程逻辑。
  + 为currentReportedOffset赋值为currentPhyOffset。
  + 调用方法HAClient#reportSlaveMaxOffset汇报从节点偏移量。
  + 如果汇报失败，则调用closeMaster方法关闭连接。
+ 如果汇报失败返回false，其余情况均返回true。

### reallocateByteBuffer

RocketMq这里代码中对ByteBuffer的使用感觉比较奇怪，不太适应。其大致的思路是不断向ByteBuffer中写入数据，然后处理数据的时候要么是从绝对偏移量开始，要么是临时设置postion，读取完毕后在恢复。

当不断从通道中读取数据后，byteBufferRead的position最终会到达limit的位置。此时就会触发这个方法。

这个方法会将未处理区域的开始偏移量dispatchPosition到capacity之间的部分写入到byteBufferReadBackup中。然后让byteBufferReadBackUp和byteBufferRead互相交换。那么此时的byteBufferRead又是一个可以继续写入的Buffer了。

## 总结

通过对高可用中涉及到的类的代码逻辑分析，可以大致梳理出主从角色的设计思路：

+ 主节点：通过HaConnection的ReadSocketService读取从节点汇报上送偏移量信息。并且更新到`org.apache.rocketmq.store.ha.HAConnection#slaveRequestOffset`属性。WriteSocketService会检查这个属性，当发现属性变化的时候就意味着可以从这个地址开始开始进行传输数据给从节点。那么WriteSocketService就会这个偏移量开始不断传输当前提交日志的内容直到所有内容都传输完毕。然后在不断的循环中一旦发现提交日志有新的内容就再次传输给从节点。
+ 从节点：通过HaClient在死循环中不断上报自己的偏移量来通知到主节点当前需要同步，然后就可以不断的接受到主节点的同步信息。





