---
title: ConcurrentHashMap源码走读
date: 2019-11-04 08:00:00
tags:
---
# ConcurrentHashMap源码走读
[TOC]

## 简介

在从JDK8开始，为了提高并发度，`ConcurrentHashMap`的源码进行了很大的调整。在JDK7中，采用的是分段锁的思路。简单的说，就是`ConcurrentHashMap`是由多个`HashMap`构成。当需要进行写入操作的时候，会寻找到对应的`HashMap`，使用`synchronized`对对应的`hashmap`加锁，然后执行写入操作。显然，并发程度就取决于`HashMap`个数的多少。而在JDK8中换了一种完全不同的思路。

首先，仍然是使用`Entry[]`作为数据的基本存储。但是锁的粒度被缩小到了数组中的每一个槽位上，数据读取的可见性依靠`volatile`来保证。而在尝试写入的时候，会将对应的槽位上的元素作为加锁对象，使用`synchronized`进行加锁，来保证并发写入的安全性。

除此之外，如果多个Key的`hashcode`在取模后落在了相同的槽位上，在一定数量内（默认是8），采用链表的方式连接节点；超过之后，为了提高查询效率，会将槽位上的节点转为使用红黑树结构进行存储。

还有一个比较大的改变在于当进行扩容的时候，除了扩容线程本身，如果其他线程识别到了扩容进行中，则会尝试协助扩容。

下面来看下来针对几个重点方法进行源码分析。

欢迎加入技术交流群186233599讨论交流，也欢迎关注技术公众号：风火说。
<!--more-->

## 放入数据

添加数据的方法为`java.util.concurrent.ConcurrentHashMap#put`，该内容实现委托给方法`java.util.concurrent.ConcurrentHashMap#putVal`。

该方法整体上可以为分为三个部分：

+ 使用`spread`方法得到key的hashcode
+ 将KV对在`Entry[]`寻找合适的位置放入
+ 容器内元素总数+1，并且在需要时执行扩容。

第一步没什么好说的，直接来看第二步的相关代码，如下

```java
int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();//初始化数组，标记1
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;   //标记2           
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f); //标记3
            else {
                V oldVal = null;
                synchronized (f) {
                    //省略相关代码，标记4
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i); //标记5
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
```

代码比较复杂，我们分成了5个标记进行说明。

首先是**标记1**，如果尝试添加元素时发现`table`属性为null，则意味着整个容器尚未初始化，此时执行初始化方法，也就是`initTable`，代码如下

```java
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

整体思路很明确，通过CAS争夺`sizeCtl`属性的控制权，成功将该值设置为-1的线程可以执行初始化工作，而其他线程通过`Thread.yield()`进行等待，直到确认容器初始化完毕，也就是`table`属性有了值。当初始化完毕时，`sizeCtl`会被设置为下一次扩容的容量阀值，该值为当前容量的3/4。

如果容器已经初始化，并且Key的hashcode对应的槽位为空，则可以考虑新建一个节点放入该槽位。也就是**标记2**。这里解释下槽位上数据的读取，都是通过方法`tabAt`，代码如下

```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }
```

该取值方法是通过计算对应槽位在数组中的便宜量的值，即`((long)i << ASHIFT) + ABASE`，也就是基础偏移量+元素间隔偏移量。并且读取的时候使用的是`getObjectVolatile`，该方法的读取和对属性使用`volatile`是一样的效果，可以保证读取到最新的值。

接着来看标记2，在槽位为null的情况下，其对值的写入采用了CAS方式，也是为了保证并发的安全性。如果CAS成功，则元素添加完毕，可以直接退出循环。如果CAS失败，则意味着有其他线程已经对相同的槽位操作成功，此时就要重新循环，确认最新的情况。

如果对应的槽位不为空，且其hashcode标识为特定负数，也就是标识容器正在扩容的负数，此时需要协助进行容器扩容，也就是**标记3**。

这里对Key的hashcode做一个说明，由于key的hashcode会经过方法`spread`处理，因此必然为正数。而负数的hashcode有三个特殊的含义，分别是:

+ -1：代表容器在扩容，并且当前节点的数据已经前移到扩容后的数组中。
+ -2：代表当前槽位上的节点采用红黑树结构存储。
+ -3：代表该节点正在进行函数式运算，节点值还未最终确定。

协助扩容的分析与容器扩容放在一起，这边先暂时略过。

如果对应槽位不为空，且hashcode不为负数，就意味着该槽位可以执行元素添加，也就是**标记4**。来看下对应的代码，如下

```java
synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            //省略相关代码，其内容为在链表上添加元素，将元素添加到队列的末尾
                        }
                        else if (f instanceof TreeBin) {
                           //省略相关代码，其内容为在红黑树结构上添加元素
                        }
                    }
                }
```

为了保证对同一个槽位上并发更新的安全性，需要对槽位上的节点执行加锁操作。

取得锁之后，首先确认当前槽位上的节点是否仍然是加锁成功的节点，一致的情况说明加锁成功的前后，槽位上数据形式没有变动，才能执行后续的操作。

加锁完毕后，判断槽位上节点的类型，如果hashcode大于等于0，是为普通节点，意味着该槽位上的数据采样链表形式存储，否则判断节点类型（必然为红黑树节点，也就是TreeBin），确认其为红黑树节点。

普通节点的添加很简单，通过对比节点中的key和Value是否和要添加的KV对一致来判断是否重复，没有重复的情况下就添加到队尾。重复的情况下则依据方法入参`onlyIfAbsent`的值判断是否要进行替换。

红黑树节点的添加则比较复杂，具体算法可以参看红黑树，这边不再赘述。

当元素添加成功后，如果当前槽位采用链表存储节点，并且链表长度超过阀值，则将链表转化为红黑树结构。也就是**标记5**。

数据放入完毕后，就是对容器内元素个数的总数进行增加操作了，也就是第三步的内容。

## 容器元素总数更新

元素总数更新是依靠方法`addCount`完成。该方法总体分为两个步骤：

+ 总数更新
+ 根据入参和当前总数，判断是否执行扩容。

首先来看总数更新的部分，代码如下

```java
if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
```

整体的更新思路实际上和JDK8新增的一个统计类是完全一致的，即`java.util.concurrent.atomic.LongAdder`。这个类用于在更高的并发竞争下，降低或维持数字计算的延迟。其性能相较传统的`AtomicLong`要更好。具体的代码分析这边就不展开了，但是说下核心思路：

+ 整个统计的数据结构包含一个基本的长整形变量baseCount和一个统计单元CounterCell构成的数组，数组的长度为2的次方幂，初始长度为2，最大长度超过CPU内核数时停止扩容。
+ 当统计数字需要变化时，优先在baseCount上执行CAS操作。如果CAS成功，则意味着更新完成。如果失败，说明此时有多线程竞争，放弃在`baseCount`上的争夺。
+ 当放弃在`baseCount`上的争夺时，通过线程上的随机数h在`CounterCell[]`数组上找到槽位，在槽位上的`CounterCell`内部的整型变量上循环执行CAS更新，直到成功。
+ 如果需要初始化`CounterCell[]`数组或者添加元素到具体槽位，或者库容，只能一个线程进行，该线程需要对`cellBusy`这个属性进行CAS争夺并且成功。

这个算法的核心思路就是避免多线程在一个变量上循环CAS直到成功。因为当多线程竞争较为激烈时，大量的线程会在不断的CAS失败中浪费很多CPU时间。通过线程变量的方法，将多线程分散到不同的`CounterCell`单元中，降低了竞争的烈度和颗粒度，因此能够提高并发效率。

由于统计数据被分散在`baseCount`和`CounterCell[]`中，执行总数计算时也需要遍历这里面所有的值相加才能得到最终值。

总数更新完毕后，就到了扩容判断环节了。

## 容器扩容

容器扩容判断是在总数更新中的部分代码实现的，具体如下

```java
Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);//标记1
                if (sc < 0) {//标记2
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))//标记3
                    transfer(tab, null);
                s = sumCount();
            }
```

可以看到，扩容的依据是`sizeCtl`这个属性，当容器元素总数超过`sizeCtl`时，执行扩容流程。

首先第一步**标记1**，是对容器内当前数组长度计算盖戳标记值，也就是`resizeStamp`，其具体代码如下

```java
static final int resizeStamp(int n) {
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    }
```

由于n是2的次方幂，`Integer.numberOfLeadingZeros(n)`是获得32位整型数字中，在第一个1的位之前有多少个0的结果，因此这个值实际上就是数字n的一种换算关系。

`RESIZE_STAMP_BITS`则意味着该结果能够占据的比特位数。由于`Integer.numberOfLeadingZeros(n)`最大值为28（n的最小值为16），因此`RESIZE_STAMP_BITS`最小也必须为6。

这个方法计算出来的结果，实际上可以看成是数组的长度的固定换算值。这个值可以在多线程扩容过程用于判断是否扩容完毕了。

这里要对`sizeCtl`这个属性做一下说明，其取值有如下规律：

+ 0：这是一个初始值，意味着此时数组尚未初始化。
+ -1：这是一个控制值，意味着有线程取得了数组的初始化权利，并且正在执行初始化中。
+ 正数：该值是容器要扩容的阀值，一旦元素总数到达该值，则应该进行扩容。除非数组长度到达上限。
+ 非-1的负数：该值意味着当前数组正在扩容，该值的左边`RESIZE_STAMP_BITS`个数的比特位用于存储数组长度n的盖戳标记，右边`32-RESIZE_STAMP_BITS`位用于存储当前参与扩容的线程数。

回到扩容的代码，标记1代码完成后，就开始判断是执行扩容还是协助扩容。如果`sizeCtl`当前值为负数，就协助扩容也就是**标记2**；如果为正数，就发起扩容，也就是**标记3**。

首先来看标记3，也就是发起扩容。需要通过CAS对`sizeCtl`的值进行置换。发起扩容时需要置换的值的含义上面也说过，左边是盖戳标记，右边是参与扩容的线程数。

来看下扩容的具体代码，也就是`transfer`方法，该方法较为复杂，具体区分为几个步骤：

+ 步骤一：计算当前线程本次前移的槽位个数
+ 步骤二：初始化扩容后的数组对象，赋值给属性`nextTable`
+ 步骤三：按照步骤一计算的结果，从数组的末尾开始，每批迁移一定槽位上的节点到新的数组直到全部迁移完毕；将新的数组的值赋值给属性`table`，将属性`nextTable`设置为null，计算新的`sizeCtl`，迁移完成。

首先来看步骤一，很简单，只有一句代码

```java
if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE;
```

默认情况下，每次迁移1/8的槽位。

步骤二一样也很简单，就是一个基本的赋值动作，就不展开了。

步骤三比较复杂，在细分为几个阶段：

+ 阶段一：计算本次迁移开始的槽位下标和数量。
+ 阶段二：判断迁移是否完成，如果完成则设置相关属性。
+ 阶段三：按照阶段一的槽位下标和数量，执行迁移。

先来看阶段一，代码如下

```java
boolean advance = true;
boolean finishing = false;
ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
        int nextIndex, nextBound;
        if (--i >= bound || finishing)
            advance = false;
        else if ((nextIndex = transferIndex) <= 0) {
            i = -1;
            advance = false;
        }
        else if (U.compareAndSwapInt(this, TRANSFERINDEX, nextIndex,nextBound = (nextIndex > stride ?nextIndex - stride : 0))) {
            bound = nextBound;
            i = nextIndex - 1;
            advance = false;
         }
         }
         //阶段二代码
         //阶段三代码
    }
```

`transferIndex`的初值为数组的长度。确定本次前移的槽位范围是第二个else if来决定的。通过CAS争夺，将`transferIndex`的值降低。CAS成功后，本次减少的`transferIndex`值对应的区域，就是本次迁移的区域。通过这种方式，每个线程都可以在自己独立的槽位范围内作业而不会互相争夺，避免竞争。

阶段二用于判断迁移是否完成，具体代码如下

```java
if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n;
                }
            }
```

当`i`小于0时意味着迁移已经结束了，此时先减少迁移线程技术，也就是CAS代码`U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)`完成的功能。通过确认是否是最后一个退出迁移的线程，也就是代码` if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)`完成的功能，来执行最后一次的检查，也就是将`i`设置为数组长度值`n`。再执行一次总体循环，检查每一个槽位都迁移完毕。

最后一次确认完毕后，就开始进行退出操作。也就是相关的赋值动作，这部分简单，不展开说明了。

阶段三用于执行迁移槽位，最为复杂，来看代码

```java
else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                advance = true;
            else {
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            //省略代码，从链表中迁移数据到新数组
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                            //省略代码，从红黑树中读取元素放入新数组
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
```

逐个槽位进行判断，这个是通过外层最大的for循环来执行的。针对每一个槽位，具体情况具体分析。

+ 如果槽位为null，则尝试通过CAS将一个标识迁移的特殊节点，`ForwardingNode`放入槽位。
+ 如果槽位上的节点已经是`ForwardingNode`，则忽略，寻找下一个槽位。
+ 不是以上两种情况，则对槽位节点加锁。成功后，执行数据迁移，迁移完毕后，将槽位节点设置为`ForwardingNode`，用以标识迁移完毕。

以链表的数据迁移为例进行分析，代码如下

```java
int                          runBit  = fh & n;
        ConcurrentHashMap.Node<K, V> lastRun = f;
        for (ConcurrentHashMap.Node<K, V> p = f.next; p != null; p = p.next)
        {
            int b = p.hash & n;
            if (b != runBit)
            {
                runBit = b;
                lastRun = p;
            }
        }
        if (runBit == 0)
        {
            ln = lastRun;
            hn = null;
        }
        else
        {
            hn = lastRun;
            ln = null;
        }
        for (ConcurrentHashMap.Node<K, V> p = f; p != lastRun; p = p.next)
        {
            int ph = p.hash;
            K   pk = p.key;
            V   pv = p.val;
            if ((ph & n) == 0)
            {
                ln = new ConcurrentHashMap.Node<K, V>(ph, pk, pv, ln);
            }
            else
            {
                hn = new ConcurrentHashMap.Node<K, V>(ph, pk, pv, hn);
            }
        }
        setTabAt(nextTab, i, ln);
        setTabAt(nextTab, i + n, hn);
```

对于数组长度为n，下标在i上的节点而言，执行2倍扩容后，其下标或者仍然为i，或者为i+n。

因此迁移之前首先遍历链表，将链表中的节点分为两个部分：迁移后下标值不一致和迁移后下标值一致，并且以一致的首节点作为分界线，也就是`lastRun`变量。`runBit`为0，意味着`lastRun`和之后的部分，迁移后下标不变；`runBit`不为0，意味着`lastRun`和之后的部分，迁移后下标变为i+n。

遍历首节点到`lastRun`节点之间的部分，计算其迁移后的下标，构建新的`node`对象，并且形成链表。而后添加到新的数组中

## 协助扩容

在执行元素更新操作时，如果槽位上的节点为`ForwardingNode`，则意味着当前容器正在扩容，则需要进行协助扩容，也就是方法`helpTransfer`的内容。代码如下

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
```

这一段代码和`addCounter`中的扩容判断部分完全一致。

首先仍然是对当前数组长度计算盖戳标记，也就是`resizeStamp`。其后在while循环中判断是否要进行协助。while条件`nextTab == nextTable && table == tab && (sc = sizeCtl) < 0`表明了当前正在进行扩容，需要协助。

来看第一个if判断：

+ `(sc >>> RESIZE_STAMP_SHIFT) != rs`意味着数组长度已经发生变化，扩容可能已经结束，不需要协助。
+  `transferIndex <= 0`意味着原始数组已经没有可以分配的扩容区域，不需要协助
+ `sc == rs + 1 ||  sc == rs + MAX_RESIZERS`这个条件永远不会达成，属于bug。具体可以看https://bugs.java.com/bugdatabase/view_bug.do?bug_id=JDK-8214427

如果确认需要协助，就来到第二个if。通过CAS的方式，增加了一个协助线程数量，然后执行迁移方法。

## 遍历

遍历的实现难度主要是在于遍历的过程中元素可能会新增或者删除，或者遇到扩容的情况。分情况分析：

+ 遍历时容器没有变化
+ 遍历时容器元素有新增或者删除
+ 遍历时容器正在扩容

遍历是通过生成迭代器方式进行,主要三个方法`keySet`,`valueSet`,`entrySet`。但是遍历的机制都是相同的，具体的实现都是依赖`java.util.concurrent.ConcurrentHashMap.Traverser`实现的迭代器。

首先来看下该类的重要属性

```java
Node<K,V>[] tab;//当前迭代器需要遍历的数组
Node<K,V> next; //迭代器next方法将要返回的值
TableStack<K,V> stack, spare; // 在遍历过程中遇到ForwardingNodes节点时，存储当前遍历信息的对象
int index; //下一个要遍历的槽位的下标
int baseIndex; //初始槽位数组的当前遍历下标
int baseLimit;  //初始槽位数组的遍历下标的终值
final int baseSize; //初始遍历数组的大小
```

从迭代器的`tab`属性可以推测出迭代的取值是从`tab`中来定位对应的槽位的。而从`baseLimit`属性则可以推测出遍历的是从下标0开始的。而`baseSize`是初始数组的大小且为final，意味着遍历的范围只针对初始数组。结合以上三点，可以得到遍历的第一个原则.

>遍历是以迭代器初始化入参的数组为依据，从下标0开始，遍历到baseLimit截止。

关于迭代器的可见性，在遍历的时候，容器元素可能添加或者是删除，对于在遍历下标之前的槽位，元素的添加或者删除是不可见的，也不关心。而在遍历下标之后的槽位上的元素新增删除，在遍历到具体的槽位时即可发现。对槽位的读取，上面介绍过，采用的是volatile的方式，因此都可以看到最新的数据。

最复杂的情况要属在遍历的时候遇到容器扩容的情况。迭代器的最基本保证就是不能遍历到重复的元素。但是容器的扩容的时候，下标i的节点会被重新分配到`i`和`i+n（原数组长度）`的位置。也就是`i+n-1`位置上会有部分原本数组上`i-1`的元素，如果遍历到这个槽位，则会导致重复的元素在遍历中出现。

这边以图的形式更容易来说明，首先见下图

![](https://markdownpic-1251577930.cos.ap-chengdu.myqcloud.com/20191029144409.png)

下标0-3均已遍历过，在遍历下标4的槽位时发现了该节点是一个`ForwardingNode`节点，这意味着该数组上剩余的槽位上的节点均已迁移到新的数组中。两个数组中相同颜色的槽位意味着存在节点的迁移关系。比如槽位4上的节点就会迁移到新数组的槽位4和槽位12中。而灰色的部分意味着存在着已经遍历过的槽位。显然，从新数组的下标4开始遍历，一旦遍历到8-11槽位，就会遍历到重复的数据，这显然是不允许的。

`ConcurrentHashMap`的做法就是仍然遍历原始数组，但是发现槽位节点是`ForwardingNode`，则遍历`ForwardingNode`节点指向的数组，并且只遍历其`i`和`i+n`槽位的数据。然后回归原始数组，继续这个流程。这样的做法，就能避免遍历到新数组中可能存在重复数据的槽位。当然，同时也忽略了这些槽位上新增的数据，但是至少保证了数据的正确性。

知道了算法思路，再来看代码就好理解多了。私以为，这段代码算是最不好理解的部分了（排除红黑树）。

```java
final Node<K,V> advance() {
            Node<K,V> e;
            if ((e = next) != null)
                e = e.next;
            for (;;) {
                Node<K,V>[] t; int i, n;  // must use locals in checks
                if (e != null)
                    return next = e;
                if (baseIndex >= baseLimit || (t = tab) == null ||
                    (n = t.length) <= (i = index) || i < 0)//标记1
                    return next = null;
                if ((e = tabAt(t, i)) != null && e.hash < 0) {//标记2
                    if (e instanceof ForwardingNode) {
                        tab = ((ForwardingNode<K,V>)e).nextTable;//标记3
                        e = null;
                        pushState(t, i, n);
                        continue;
                    }
                    else if (e instanceof TreeBin)
                        e = ((TreeBin<K,V>)e).first;
                    else
                        e = null;
                }
                if (stack != null)  //标记4
                    recoverState(n);
                else if ((index = i + baseSize) >= n)  //标记5
                    index = ++baseIndex; // visit upper slots if present
            }
        }
```

方法`advance`用于确定`next`方法可以返回的值，也就是确定`next`属性的值。通过标记1的代码，`i = index`，可以确定本次需要寻找的槽位，通过标记2的代码`e = tabAt(t, i)`获取到槽位上的节点。

如果节点是`ForwardingNode`类型，则意味该槽位和后续的槽位都已经迁移完毕了，因为迁移的时候是从数组的末尾向前开始的。此时将需要遍历的数组切换为本次扩容后的数组，也就是代码`tab =((ForwardingNode<K,V>)e).nextTable`的含义。切换完成后，保存此时的遍历状态信息，也就是方法`pushState`的内容，来看下具体的代码

```java
private void pushState(Node<K,V>[] t, int i, int n) {
            TableStack<K,V> s = spare;  // reuse if possible
            if (s != null)
                spare = s.next;
            else
                s = new TableStack<K,V>();
            s.tab = t;
            s.length = n;
            s.index = i;
            s.next = stack;
            stack = s;
        }
```

这个方法的内容是通过`TableStack`形成一个堆栈的数据结构。每次保存遍历状态信息都是一次压栈操作。为了减低GC，提升效率，会将不再使用的`TableStack`对象以反向的形式连接起来，链表头存储在`spare`属性。当需要压栈时，可以先尝试从`spare`获取对象进行复用，而不是马上新建对象。

遍历状态信息保存完毕后，就从扩容后的数组开始遍历。通过标记1和2的代码获取了槽位i上的新的节点。此时就可以针对该槽位进行遍历，不过在遍历之前，需要先确定下一次遍历的下标。也即是标记4的代码内容。来看下方法`recoverState`。在标记4的调用中，其入参n是传入的扩容后的数组大小。方法代码如下

```java
private void recoverState(int n) {
            TableStack<K,V> s; int len;
            while ((s = stack) != null && (index += (len = s.length)) >= n) {
                n = len;
                index = s.index;
                tab = s.tab;
                s.tab = null;
                TableStack<K,V> next = s.next;
                s.next = spare; // save for reuse
                stack = next;
                spare = s;
            }
            if (s == null && (index += baseSize) >= n)
                index = ++baseIndex;
        }
```

在扩容后的数组第一次进入该方法，实际的作用就是将`index`的值从`i`增加到`i+n`。也就是代码`index += (len = s.length))>= n`的作用。第一次进入的时候，这个表达式为false。第二次进入的时候则为true。那就意味着上次压栈的`TableStack`保存的旧的数组和遍历下标在新的数组中对应的两个下标位置`i`和`i+n`都遍历完毕了。此时进行一个弹栈操作，并且将需要遍历的数组还原为旧的数组，下标和长度信息也还原为压栈时的情况。

一直执行弹栈操作，直到栈空或者再次在某一个扩容数组上index处于有效值,也就是`(index += (len = s.length)) < n`为真。index是有效值，则遍历该数组该下标的槽位上的节点。如果栈空，则意味着遍历回到了初始数组上，也就是`s == null`条件成立，此时将index的值加1，也就是`index = ++baseIndex`，然后继续遍历。

而下一个槽位上的节点，也会是`ForwardingNode`类型，重复这个流程，直到初始数组遍历完毕。