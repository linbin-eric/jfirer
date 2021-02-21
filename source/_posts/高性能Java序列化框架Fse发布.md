---
title: 高性能Java序列化框架Fse发布
date: 2020-02-03 08:00:00
tags:
---

# 高性能Java序列化框架Fse发布

[TOC]

## 使用场景

将Java对象序列化为二进制数据进行保存，以及二进制数据反向序列化为Java对象，在很多场景中都有应用。比如将对象序列化后离线存储至其他介质，或者存储于Redis这样的缓存之中。

目前常见的有几种框架可以支撑，比如 Hession ，Kryo，Fst，Protobuf，JDK原生等。有一些框架需要提前编写元数据配置文件以支撑跨语言序列化能力，比如 Protobuf 。不过如果团队的技术栈是统一的 Java 体系的话，则能够开箱即用的序列化框架使用起来会更加方便一些，特别有些时候对象特别复杂，编写元数据配置文件也是很繁琐的一个事情。

Fse 框架正是应用于这样的场景，不需要编写元数据配置信息，开箱即用的 Java 序列化框架，对需要序列化的对象没有任何特殊要求。在性能基准测试中，该框架的性能表现显著优于其他框架，下面是测试对比

![](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20200204221213.png)

欢迎加入技术交流群186233599讨论交流，也欢迎关注笔者公众号：风火说。

<!--more-->

## 使用说明

首先在Pom文件中引入依赖，如下

```xml
<dependency>
    <groupId>com.jfireframework</groupId>
    <artifactId>fse</artifactId>
    <version>aegean-2.0</version>
</dependency>
```

API 使用方式如下

```java
Fse fse = new Fse();
TestData data = new TestData();
//创建一个二进制数组容器，用于容纳序列化后的输出。容器大小会在需要时自动扩大，入参仅决定初始化大小。
ByteArray buf = ByteArray.allocate(100);
//执行序列化，会将序列化对象序列化到二进制数组容器之中。
fse.serialize(data, buf);
//得到序列化后的二进制数组结果
byte[] resultBytes = buf.toArray();
//清空容器内容，可以反复使用该容器
buf.clear();
//填入数据，准备进行反序列化
buf.put(resultBytes);
TestData result = (TestData) fse.deSerialize(buf);
assertTrue(result.equals(data));
```

## 开源地址

Gitee:https://gitee.com/eric_ds/fse

Github:https://github.com/linbin-eric/Fse