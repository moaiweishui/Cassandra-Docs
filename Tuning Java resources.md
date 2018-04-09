# Tuning Java resources

对JVM进行调整能够提升性能表现/降低高内存消耗。

## About garbage collection

垃圾回收（GC）指的是Java把不再被需要的数据从内存中移除的过程。选择正确的垃圾回收器与合适尺寸的堆，对于实现最好的性能表现是非常重要的。

垃圾回收停顿时间（garbage collection pause，也被称为stop-the-world事件）无疑是我们想要最小化的。当内存中某个区域被填满时，JVM需要腾出空间，此时一个停顿事件就发生了。在此停顿期间，所有的操作都被挂起。这种停顿事件同样会影响网络，当某个节点发生了stop-the-world 事件后，该节点在集群中的其他节点看来是处于离线状态的。此外，所有的Select与Insert操作都进入等待，这会带来额外的读/写时间的增加。任何超过1s的单次停顿，或者是1s内的多次停顿（这会造成这1s被多次切分），都应该被避免。出现这种情况的根本原因是，将数据存储于内存中的速度超过了数据可以被移除的速度。有关特定的问题表现与原因，参阅[Garbage collection pauses](https://docs.datastax.com/en/dse-trblshoot/doc/troubleshooting/gcPauses.html?hl=garbage%2Ccollection%2Cpauses )。

## Choosing a Java garbage collector

对于Cassandra 3.0及后续版本，使用 Concurrent-Mark-Sweep (CMS) 或 G1 garbage collector取决于下述因素：

以下情况更推荐使用G1：

- Heap size在16 GB至64 GB

  在拥有更大Heap size的情况下，G1能比CMS表现出更好的性能，这是因为G1会首先对堆中存储了大量垃圾对象的区域进行扫描，与此同时执行compact操作。而CMS在执行垃圾回收的过程中，会停止所有应用程序。

- 工作负载是可变的，也就是说，集群一直在执行不同的进程

- 考虑到未来，CMS将在Java 9中被弃用

- G1更易于配置

- G1 is self tuning

以下情况更推荐CMS：

- 拥有时间及相关经验来对垃圾回收执行手动调优与测试

  需要注意的是，为堆分配更多的内存，可能会导致性能上的损失，这是因为垃圾回收工具提高了堆内存中Cassandra元数据的数量。

- Heap size不超过16 GB

- 工作负载固定

- 要求尽可能低的延迟，G1会因为profiling带来些许额外延迟

## Setting G1 as the Java garbage collector

- 打开*jvm.options*
- 注释掉 -Xmn800M 行
- 注释掉 ### CMS Settings 块内所有内容
- 将 ### G1 Settings 块中相关内容取消注释

```
## Use the Hotspot garbage-first collector.
-XX:+UseG1GC
#
## Have the JVM do less remembered set work during STW, instead
## preferring concurrent GC. Reduces p99.9 latency.
#-XX:G1RSetUpdatingPauseTimePercent=5
```

​	Note：当使用G1，只需要设置MAX_HEAP_SIZE

## Determining the heap size

部分用户可能尝试调整Java堆去占据计算机内存的绝大部分。然而，这可能会对OS page cache操作造成影响。现代操作系统为频繁访问的数据维护OS page cache，并且能够很好的保留这些数据在内存中。调整OS page cache至合适的值会比提高Cassandra row cache带来更好的性能表现。

Cassandra基于以下公式自动计算最大Heap size（MAX_HEAP_SIZE）：
$$
max(min(1/2 ram, 1024MB), min(1/4 ram, 8GB))
$$
对于生产环境下的应用，我们基于以下规则来调整Heap size：

- Heap size一般来说介于系统内存的1/2至1/4
- 不要将所有内存都分配给堆，因为需要留出给堆外缓存与文件系统缓存
- 调整GC时，始终启用GC日志
- 逐步调整设置，并被每一次增量调整都进行测试
- 启用GC并行处理，特别是当使用DSE搜索时
- Cassandra的GCInspector类记录了每一次耗时超过200 ms的垃圾回收记录信息。垃圾回收频繁执行，且每次都消耗一定时间，这表明JVM垃圾回收压力过大。除了调整垃圾回收设置外，可以采取的措施还包括增加节点数量，及降低缓存大小
- 对于一个使用G1的节点，Cassandra社区推荐MAX_HEAP_SIZE尽可能大，最大为64 GB

#### **MAX_HEAP_SIZE**

| HARDWARE SETUP                                           | Recommended MAX_HEAP_SIZE |
| :------------------------------------------------------- | ------------------------- |
| Older computers                                          | Typically 8 GB            |
| CMS for newer computers (8+ cores) with up to 256 GB RAM | No more 16 GB             |
| G1 for newer computers (8+ cores) with up to 256 GB RAM  | 16 GB to 64 GB            |

最简单的确定最优heap size的方法是：

1. 当使用G1时，挑选一个测试节点，在*jvm.options*中设置其最大heap size为最大值

   ```
   -Xms48G
   -Xmx48G
   ```

   设置heap size的最小（Xms）最大（Xmx）为同一值，能够避免resize期间的stop-the-world GC pause，并在启动时锁定内存中的堆，以防止其中的任何内容被换出。

2. 启用GC日志

3. 检查日志来查看该测试节点使用的heap的大小，并用该值作为集群最终的heap size

#### **HEAP_NEWSIZE**

对于CMS，还需要调整HEAP_NEWSIZE。该值决定了为新对象或*young generation*所分配的堆内存的大小。Cassandra使用下面两个值中的较小值作为其默认值（in MB）：

- 100 times the number of cores
- ¼ of MAX_HEAP_SIZE

一开始，可以设置HEAP_NEWSIZE大小为物理核数乘上100 MB。

更大的HEAP_NEWSIZE带来更长的GC pause时间。较小的HEAP_NEWSIZE对应较小的GC pause时间，但通常比较昂贵。

## How Cassandra uses memory

Cassandra在JVM堆上主要执行下列操作：

- 为了执行读操作，Cassandra需要在堆中维护下列组件：
  - Bloom filters
  - Partition summary
  - Partition key cache
  - Compression offsets
  - SSTable index summary
- Cassandra收集副本已进行读取或进行反熵修复（anti-entropy repair），并且比较堆中的副本
- 写入Cassandra的数据首先被存储于堆内存中的memtables，之后再被刷入磁盘中的SSTables

为了提高性能表现，Cassandra同时使用堆外内存进行下列操作：

- Page cache，当读取磁盘上的文件时，Cassandra使用额外的内存作为页缓存
- The Bloom filter and compression offset maps reside off-heap
- Cassandra可以将被缓存的行存储于原生内存中（位于堆外）。这降低了对JVM堆的需求，这有助于将堆大小保持在JVM垃圾收集性能的最佳位置

## Adjusting JVM parameters for other Cassandra services

- Solr：一些Solr用户报告称，增加堆栈大小可以提高Tomcat的性能。

  为了增加堆栈大小，在*cassandra-env.sh*文件中取消对应部分注释，并修改默认值：

  ```
  # Per-thread stack size.
  JVM_OPTS="$JVM_OPTS -Xss256k"
  ```

  另外，减小memtable大小来为Solr缓存腾出空间可以提高性能。通过修改*cassandra.yaml*文件中的 memtable_heap_space_in_mb与memtable_offheap_space_in_mb来改变memtable大小

- MapReduce：MapReduce在JVM之外运行，因此修改JVM并不会直接影响Analytics/Hadoop操作

## Other JMX options

Cassandra通过Java Management Extensions（JMX）公开了其他统计和管理操作，JConsole与nodetool组件是JMX兼容的管理工具。

通过在*cassandra-env.sh*中编辑这些属性来配置Cassandra，以进行JMX管理：

- com.sun.management.jmxremote.port: sets the port on which Cassandra listens from JMX connections
- com.sun.management.jmxremote.port: enables or disables SSL for JMX
- com.sun.management.jmxremote.port: enables or disables remote authentication for JMX
- -Djava.rmi.server.hostname: sets the interface hostname or IP that JMX should use to connect. Uncomment and set if you are having trouble connecting

