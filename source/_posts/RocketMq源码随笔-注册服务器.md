---
title: RocketMQ源码随笔-注册服务器
date: 2021-1-25 18:00:00
tags: RocketMq 源码解读
---
# RocketMQ源码随笔-注册服务器

[TOC]

## NamesrvStartup

该类用于启动注册服务器。其`main`方法委托了`main0`方法，该方法的执行逻辑如下：

1. 调用方法`NamesrvStartup#createNamesrvController`创建一个`NamesrvController`实例，声明为`controller`。
2. 调用方法`NamesrvStartup#start`将这个controller启动。

那么下面就分别来看下两个方法的具体内容。

欢迎加入技术交流群186233599讨论交流，也欢迎关注技术公众号：风火说。
<!--more-->

### createNamesrvController

这个方法最重要的就是用构造方法创建了`NamesrvController`对象。而在调用构造方法之前有较多的代码是用于解析命令行对象，以及可能的情况下读取文件中的配置信息、打印当前的整体配置信息。

这些额外配置不存在的时候，默认配置下，注册服务器是监听于9876端口。

### start

该方法的作用是启动入参的`NamesrvController`实例。具体来说，流程如下：

1. 执行方法`NamesrvController#initialize`进行初始化。
2. 为运行时添加一个hook，在JVM关闭的时候，执行方法`NamesrvController#shutdown`对注册服务器执行优雅关闭。
3. 执行方法`NamesrvController#start`启动注册服务器。

## NamesrvController

这个类用于控制注册服务器。

### 构造方法

构造方法中主要是为了几个重要属性进行赋值操作。比如初始化`kvConfigManager`和`routeInfoManager`这两个重要的属性。

### initialize

该方法用于初始化注册服务器，执行逻辑如下：

1. 执行方法`kvconfig.KVConfigManager#load`加载配置信息。默认情况下，加载 ${user.home}/namesrv/kvConfig.json 文件的内容到属性`kvconfig.KVConfigManager#configTable`中。
2. 新建一个NettyRemotingServer对象，为属性`NamesrvController#remotingServer`赋值。这个新建的对象，使用了`BrokerHousekeepingService`作为入参。该`BrokerHousekeepingService`的作用就是在发生通道关闭、异常、空闲等情况时，将该通道从路由信息里删除。
3. 创建一个线程池，赋值给属性`NamesrvController#remotingExecutor`，用于注册服务器在Netty中的业务执行。
4. 调用方法`NamesrvController#registerProcessor`将业务处理器注册到`RemotingServer`中。使用的线程池就是步骤3创建的线程池。
5. 创建一个间隔时间为10秒的周期性任务，任务内容是调用方法`RouteInfoManager#scanNotActiveBroker`扫描非激活模式的`Broker`。

### start

该方法没有更多内容，只是简单了启动了`RemotingServer`。在这个方法之后，就可以开始监听`Broker`上送的注册请求。

## KVConfigManager

该类是注册服务器的配置存储类。会将配置信息存储在文件 ${user.home}/namesrv/kvConfig.json 。内部用来存储配置信息的是一个`HashMap<String, HashMap<String, String>> `结构，也就是两级结构。

第一级是命名空间，第二集是KV对，都是字符串形式。

该类的`load`方法可以从文件中加载数据到内存里，`persist`方法可以将内存中的数据再写入到文件中。

## DefaultRequestProcessor

这个类是 rocketmq-namesrv 这个包下面，代码量最多的类了。因为业务处理都实现在了这个类上面。

按照`NettyRequestProcessor`接口的实现套路，业务请求的分流都是在`processRequest`方法中，这里也是，接下来就一个个看这个类支持的命令。

### PUT_KV_CONFIG

该命令没有请求体，请求头中有`namespace`、`key`、`value`字段，调用方法`kvconfig.KVConfigManager#putKVConfig`将配置项放入到配置管理器中即可。

### GET_KV_CONFIG

该命令没有请求体，请求头中有`namespace`、`key`字段，调用方法`kvconfig.KVConfigManager#getKVConfig`获取对应配置项。

如果配置项存在，返回成功响应。如果配置信息不存在，返回失败响应，响应码为*QUERY_NOT_FOUND*。

### DELETE_KV_CONFIG

该命令没有请求体，请求头中有`namespace`、`key`字段，调用方法`kvconfig.KVConfigManager#deleteKVConfig`删除对应配置项。

### QUERY_DATA_VERSION

该命令用于查询注册服务器上`Broker`的数据版本号。具体执行逻辑如下：

1. 从命令的内容体解析出`DataVersion`对象，从请求头中解析出`BrokerAddr`数据。使用这两个作为入参，调用方法`RouteInfoManager#isBrokerTopicConfigChanged`判断与服务器上该`BrokerAddr`的版本号是否一致，将结果声明为`changed`。
2. 如果`changed`为`false`，表明版本号没有变化，那么服务器上的数据在当前时间还是有效的，调用方法`RouteInfoManager#updateBrokerInfoUpdateTimestamp`更新这个数据的有效时间。
3. 调用方法`RouteInfoManager#queryBrokerTopicConfig`查询服务器上`BrokerAddr`对应的版本号，声明为`nameSeverDataVersion`。
4. 构建命令响应对象，如果`nameSeverDataVersion`不为null，则编码后设置到内容体。在响应头中设置`changed`属性，值为步骤1产生的声明对象。

### REGISTER_BROKER

该命令用于`Broker`信息的注册。首先获取请求头中MQ的版本号，如果版本号大于等于3.0.11，则调用方法`processor.DefaultRequestProcessor#registerBrokerWithFilterServer`进行信息注册；否则调用方法`processor.DefaultRequestProcessor#registerBroker`进行信息注册。

#### registerBrokerWithFilterServer

方法的执行逻辑如下：

1. 对请求命令进行解码工作，创建出`RegisterBrokerRequestHeader`对象。使用该对象对象和请求中的body字段执行crc校验，如果校验失败，返回系统错误响应。否则，继续后续流程。
2. 如果命令请求对象中包含内容体，则解码出`RegisterBrokerBody`对象，声明为`registerBrokerBody`。如果命令请求对象不包含内容体，则手动创建`RegisterBrokerBody`对象，并且将其`DataVersion`的版本号设置为0，时间戳设置为0.
3. 调用方法`RouteInfoManager#registerBroker`注册路由信息，将结果声明为`result`。
4. 创建类型为`RegisterBrokerResponseHeader`的响应头对象，声明为`responseHeader`。将`result`的`masterAddr`和`HaServerAddr`属性设置到响应头对象中。
5. 从配置管理器中以*ORDER_TOPIC_CONFIG*作为命名空间，取出该命名空间下面的配置数据对象，编码后将二进制设置为响应的内容体。
6. 返回响应对象。

#### registerBroker

与`registerBrokerWithFilterServer`方法的流程基本一致，只不过在调用方法`RouteInfoManager#registerBroker`的时候，入参的`filterServerList`为null。

### UNREGISTER_BROKER

该命令用于注销 Broker 的注册。调用方法`org.apache.rocketmq.namesrv.processor.DefaultRequestProcessor#unregisterBroker`完成，而该方法内部则是委托给了方法`org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#unregisterBroker`。

### GET_ROUTEINFO_BY_TOPIC

该命名用于查询主题的路由信息，调用了方法`org.apache.rocketmq.namesrv.processor.DefaultRequestProcessor#getRouteInfoByTopic`。

该方法用于在路由管理器中根据主题名称获取全量的路由信息，具体流程如下：

1. 使用方法`org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#pickupTopicRouteData`根据请求的主题名称得到类型为`TopicRouteData`的结果，声明为`topicRouteData`。
2. 如果`topicRouteData`不为null，则执行如下子流程。
   1. 如果配置`org.apache.rocketmq.common.namesrv.NamesrvConfig#orderMessageEnable`开启，则从命名空间*ORDER_TOPIC_CONFIG*下面，获取入参主题名称的配置信息，声明为`orderTopicConf`。将`orderTopicConf`设置到属性`org.apache.rocketmq.common.protocol.route.TopicRouteData#orderTopicConf`。
   2. 将`topicRouteData`进行编码，设置为响应的内容体，返回响应对象。
3. 如果`topicRouteData`为null，则返回*TOPIC_NOT_EXIST*响应。

### GET_BROKER_CLUSTER_INFO

该命令调用了方法`org.apache.rocketmq.namesrv.processor.DefaultRequestProcessor#getBrokerClusterInfo`。该方法的逻辑就是调用方法`org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#getAllClusterInfo`得到一个编码后的内容体，将这个内容体设置为响应的内容体，返回响应对象即可。

编码的内容体数据结构类是`ClusterInfo`，其属性如下

```java
HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
```

### WIPE_WRITE_PERM_OF_BROKER

该命令用于擦除Broker的写权限，也就说所有在该`Broker`上的主题都没有写入权限了。调用了方法`org.apache.rocketmq.namesrv.processor.DefaultRequestProcessor#wipeWritePermOfBroker`实现，该方法的逻辑如下：

1. 调用方法`org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#wipeWritePermOfBrokerByLock`擦除入参`Broker`的写权限，方法的返回值为擦除的队列信息个数。将结果声明为`wipeTopicCnt`。
2. 将`wipeTopicCnt`设置到响应头的对应属性，返回响应。

### GET_ALL_TOPIC_LIST_FROM_NAMESERVER

该命令用于获取注册服务器上全量的主题信息，调用方法`org.apache.rocketmq.namesrv.processor.DefaultRequestProcessor#getAllTopicListFromNameserver`实现。

该方法内部调用方法`org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#getAllTopicList`获取所有的主题名称形成的列表，并且编码为二进制数组，设置为响应的内容体，将响应返回。

### DELETE_TOPIC_IN_NAMESRV

该命令用于删除服务器上的主题信息，通过方法`org.apache.rocketmq.namesrv.processor.DefaultRequestProcessor#deleteTopicInNamesrv`实现。方法实现也简单，直接从`topicQueueTable`中删除对应的主题名称即可。

### GET_KVLIST_BY_NAMESPACE

该命令用于获取服务器上特定命名空间下的配置信息。通过方法`org.apache.rocketmq.namesrv.kvconfig.KVConfigManager#getKVListByNamespace`获取到对应的配置信息，并且编码为二进制数组。

如果数组存在，则设置到响应的内容体中，返回成功响应。

如果数组不存在，则返回*QUERY_NOT_FOUND*响应。

### GET_TOPICS_BY_CLUSTER

该命令用于获取集群下所有的主题名称，调用方法`org.apache.rocketmq.namesrv.processor.DefaultRequestProcessor#getTopicsByCluster`完成。该方法内部调用`org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#getTopicsByCluster`获取集群下的所有主题名称的编码结果，将编码结果的二进制数组设置到响应的内容体中，返回成功响应。

### GET_SYSTEM_TOPIC_LIST_FROM_NS

这个命令有点奇怪，看命令名称是获取系统主题列表。但是从方法实现上，内部的内容整体是混乱的。这个命令暂且放下，等看到相关联的请求查询的时候在处理。

### GET_UNIT_TOPIC_LIST

该命令用于获取集群下，有*unit*标识的主题名称集合。通过方法`org.apache.rocketmq.namesrv.processor.DefaultRequestProcessor#getUnitTopicList`实现，该方法内部调用了方法`org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#getUnitTopics`来返回具备*unit*标识的主题名称集合的编码后二进制数组。将这个数组设置为响应的内容体，并且返回。

### GET_HAS_UNIT_SUB_TOPIC_LIST

该命令用于获取集群下，有*unit_sub*标识的主题名称集合。做法上与*GET_UNIT_TOPIC_LIST*命令是相同的，只不过用的标识不同。

### GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST

该命令用于获取集群下，同时有*unit*和*unit_sub*标识的主题名称集合。做法上与上述的一致，只不过用的标识不同。

### UPDATE_NAMESRV_CONFIG

这个命令是用于管理端直接发送配置的文本到注册服务，用于更新注册服务自身的配置，而后将配置信息持久化到磁盘文件。

### GET_NAMESRV_CONFIG

这个命令用于获取注册服务的配置信息，将配置信息设置到响应的内容体中。

## RouteInfoManager

该类是路由信息的管理器，其中使用了多个类来抽象各种路由信息。下面先看下这些定义类。

**QueueData**

该类保存了Broker中的队列信息。有如下属性：

+ brokerName，Broker的名称，默认情况下是Broker所在机器的域名，可以由配置定义。
+ readQueueNums，用于读取的队列数量。
+ writeQueueNums，用于写入的队列数量。
+ perm，该Broker的权限信息，权限指的是是否可读、是否可写。
+ topicSynFlag，主题同步标识。

**BrokerData**

该类保存了Broker集群的地址信息，有如下属性：

+ cluster，集群标识。
+ brokerName，Broker名称。
+ brokerAddrs，brokerId和BrokerAddr的映射表。该属性存储了同一个Broker名称下id和地址的映射关系。

**BrokerLiveInfo**

该类保存了具体某个Broker的存活信息，有如下属性

+ lastUpdateTimestamp，最近一次数据更新时间。
+ dataVersion，该Broker的主题配置信息的版本号。
+ channel，Netty的Channel对象，该对象即是Broker与服务器之间的链接对象。
+ haServerAddr，高可用主节点地址。格式为${ip}:\${port} 。

### 存储属性

RouteInfoManager内部管理着5个Map结构，用于存储路由相关信息，这些信息用代码来看会更清晰一些，如下：

```java
HashMap<String/* topic */, List<QueueData>> topicQueueTable;
HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
```

### registerBroker

该方法用于实现Broker信息注册到路由管理器上，具体方法流程如下：

1. 从`clusterAddrTable`中以入参的`clusterName`获取集群下所有`Broker`的名称，声明为`brokerNames`。
2. 如果`brokerNames`为null，则为其赋值一个空的`HashSet<String>`。并且在`clusterAddrTable`放入这个`clusterName`和`brokerNames`两个值。
3. 在`brokerNames`中添加本次注册上来的Broker的名称。
4. 从`brokerAddrTable`以`brokerName`获取`BrokerData`对象，如果不存在则新建一个并且放入到`brokerAddrTable`中。
5. 取出步骤4中`brokerData`中的`brokerAddrs`映射，遍历其中的元素，如果值与入参的`brokerAddr`相等，键与入参的`brokerId`不等，则删除这个这一键值对。这种情况说明此时该IP对应的Broker信息已经发生了变化。
6. 将入参的`brokerId`和`brokerAddr`放入到`brokerAddrs`中。
7. 如果brokerId为0也就是主节点，并且入参的`topicConfigWrapper`不为null，也就是说Broker发送的注册命令是包含了请求体，那么执行子流程。否则继续后续流程。
   1. 从`brokerLiveTable`查询该`broker`的版本号，与`topicConfigWrapper`的版本号对比，确认是否有变化。如果有变化，或者该`Broker`是新注册的（`brokerName`第一次注册或者`brokerId`第一次注册），那么就很有可能本次携带了新的主题配置信息。则需要更更新注册服务器上主题配置信息。也就执行后续流程。否则结束子流程，继续执行步骤8.
   2. 遍历属性`TopicConfigSerializeWrapper#topicConfigTable`，对集合中每一个元素调用方法`RouteInfoManager#createAndUpdateQueueData`，更新主题对应的队列信息。
8. 构建`BrokerLiveInfo`对象，放入`brokerLiveTable`中。
9. 如果入参的`filterServerList`不为null，则放入`filterServerTable`。
10. 如果brokerId不为0，也就是当前是从节点在注册自己，则从`brokerAddrs`获取主节点的地址。如果主节点地址存在，则进一步获取其 HaServer 地址。将这两个数据设置到返回的结果对象result中。
11. 返回结果对象result。从代码可以看出，如果当前注册不是从节点，或者对应的主节点不存在，则result是一个空对象。

### createAndUpdateQueueData

该方法是用于创建或更新 topicQueueTable 中的`QueueData`对象的。具体流程如下：

1. 构建一个QueuData对象，里面的属性来自brokerName、topicConfig 对象。
2. 从 topicQueueTable 中获取topicConfig 主题对应的 queueDataList 对象。
3. 如果 `queueDataList` 不存在，意味着该主题是第一次出现在注册服务器中。构建一个新的`linkedList`对象，添加`queueData`对象到其中，并且将`queueDataList`放入到`topicQueueTable`中。流程结束。
4. 如果`queueDataList`存在，则对其元素遍历，执行如下子操作。
   1. 元素的`brokerName`属性与入参的`brokerName`值相同，则继续执行后续流程，否则进入下一次循环迭代。
   2. 判断元素与步骤1构建的对象是否相同，如果相同，不做操作；如果不同，意味着数据有变化，将元素从集合中删除。
5. 如果步骤4中有元素被删除，则将步骤1的对象，添加到`queueDataList`中。

### unregisterBroker

该方法用于在删除路由管理器中某一个`Broker`的信息。具体流程如下：

1. 在`brokerLiveTable`中删除该`Broker`信息。
2. 在`filterServerTable`删除该`Broker`的信息。
3. 声明一个局部变量`removeBrokerName`。从`brokerAddrTable`获取该`BrokerName`对应的`brokerData`。如果其不为空，则执行子流程。
   1. 从`brokerData`的`brokerAddrs`删除该`brokerId`对应的映射。
   2. 如果`brokerAddrs`集合为空，则从`brokerAddrTable`删除该`brokerName`对应的映射。为`removeBrokerName`赋值`true`。
4. 如果`removeBrokerName`为真，则执行子流程，否则流程结束。
   1. 从`clusterAddrTable`获取该`clusterName`对应的`brokerName`的集合，声明为`nameSet`。
   2. `nameSet`不为null的情况下，从`nameSet`删除本次的`brokerName`。如果删除后`nameSet`为空，则从`clusterAddrTable`删除该`brokerName`的映射。
   3. 调用方法`removeTopicByBrokerName`删除`brokerName`对应的主题信息。

### removeTopicByBrokerName

该方法用于删除`brokerName`对应的主题配置信息，具体执行逻辑如下：

1. 遍历`topicQueueTable`，为每一个元素执行后续逻辑。
2. 针对每一个元素，取出其`QueueData`列表，遍历该对象。执行子流程。
   1. 遍历`QueueData`列表，如果元素`QueueData`的`brokerName`与入参`brokerName`相同，则从列表中删除该元素。
   2. 遍历完毕后，如果列表为空，则从`topicQueueTable`中删除该映射。

### pickupTopicRouteData

首先来看下数据结构对象`TopicRouteData`的定义，其属性如下

```java
String orderTopicConf;
List<QueueData> queueDatas;
List<BrokerData> brokerDatas;
HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable
```

从数据结构对象也可以简单的推测出`pickupTopicRouteData`方法的实现逻辑。大致来说分为几个步骤：

1. 从`topicQueueTable`按照主题名称查询`queueDatas`。
2. 根据`queueDatas`中每一个元素`QueueData`的`brokerName`属性从`brokerAddrTable`取得`brokerData`对象，组成成一个List，也就是`brokerDatas`。
3. 根据步骤2的`brokerDatas`从`filterServerTable`查询到对应的`filterServer`列表，组装为映射。
4. 将步骤1到3的值组装为`TopicRouteData`对象返回给调用者。

### wipeWritePermOfBrokerByLock

该方法会以可中断的方式获取写锁，获取成功后调用方法`wipeWritePermOfBroker`。如果获取失败则返回0，获取成功则执行方法`wipeWritePermOfBroker`执行擦除工作。

`wipeWritePermOfBroker`方法的内容也很简单，遍历`topicQueueTable`，针对每一个元素，在遍历其`QueueData`，如果`brokerName`与入参的`brokerName`相同就意味着找到对应的`QueueData`。将这个里面的`perm`属性重新设置值，去掉代表写权限的标志位即可。

### getUnitTopics

该方法用于获取具备*unit*标识的主题名称集合。具体流程如下：

1. 以可中断的方式获取读锁。遍历`topicQueueTable`元素。
2. 如果键值对中的`QueueData`列表的首个元素的`topicSynFlag`属性值包含了*unit*标识，将这个键值对的`key`，也即是主题名称加入到临时集合中。
3. 遍历完后后，返回临时集合编码的二进制数组。

### onChannelDestroy

当一个Broker的通道关闭的时候，会触发到这个方法。这个方法的代码虽然比较多，但是方法思路很简单，首先通过Channel在`brokerLiveTable`中找到对应的BrokerLiveInfo对象。并且依靠这个对象的信息，在路由管理器中删除所有相关的信息接口。

## 总结

从`rocketmq-namesrv`的代码来看，注册服务还是比较简单的。注册服务彼此之间不互相通信，仅仅单纯做一个`Broker`信息的保留。这样的设计，使得注册服务的实现就变得非常简单，不过本身注册服务并不是一个实时性要求很强的业务，因此这种设计也可以认为是恰到好处。