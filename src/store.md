# spark内部存储

spark内部数据的抽象称为RDD，那RDD到底是怎么存储的呢？

我们知道，每个RDD有分片，被称为Partition，每个Partition的存储会对应为一个Block（懂设计的人看来都很会取名字！），这些Block最终存储在Memory或Disk介质上。如下图所示：
![block](https://github.com/wbear1/spark_blog/blob/master/img/store/block.png)

Task执行时可能从本地读取数据或者从远程节点读取（比如shuffle read），因此BlockManager也是一个分布式的存储系统，真正的数据读取和写入是由Store来完成。所以接下来将分析BlockManager的设计和Store的读取和写入操作。

1. BlockManager的设计

BlockManager可以认为是Spark内部的分布式存储系统，采用的是Master-Slave设计。Master运行在Driver节点，负责跟踪统计所有的Block及其存储位置，Slave运行在各个Executor节点，负责调用底层的Store层进行真正数据的存取。简易架构图如下所示：
![arch](https://github.com/wbear1/spark_blog/blob/master/img/store/arch.png)

Executor节点上的BlockManager会向运行在Driver节点的BlockManagerMaster注册，因此BlockManagerMaster拥有所有BlockManager的信息。下面分析BlockManager对外提供的读写Block关键接口：

读取Block接口：获取指定blockId的Block（先从本地的Store获取，如果获取不到的话，会从BlockManagerMaster处获取该BlockId远程存储位置，然后再通过rpc获取存储在远程的Block），如果获取不到，则使用makeIterator方法计算出该Block并更新之。
![readblock](https://github.com/wbear1/spark_blog/blob/master/img/store/readblock.png)

写入Block接口：计算出Block，存储到相应的Store，然后向BlockManagerMaster更新BlockStatus
![writeblock](https://github.com/wbear1/spark_blog/blob/master/img/store/writeblock.png)

2. Store的读取和写入操作

Store负责数据的真正存储，存储到memory或disk，因此实现上就分为MemoryStore和DiskStore。

对于MemoryStore而言，使用map来存储数据，存储又分为堆内和堆外
![memoryStore1](https://github.com/wbear1/spark_blog/blob/master/img/store/memoryStore1.png)

![memoryStore2](https://github.com/wbear1/spark_blog/blob/master/img/store/memoryStore2.png)

对DiskStore而言，每个Block对应一个文件，如下所示方法为获取block对应的文件，filename的的生成方法也见下。
![diskStore1](https://github.com/wbear1/spark_blog/blob/master/img/store/diskStore1.png)

![diskStore2](https://github.com/wbear1/spark_blog/blob/master/img/store/diskStore2.png)

RDD的Block的数据文件如下所示：
![file](https://github.com/wbear1/spark_blog/blob/master/img/store/file.png)

