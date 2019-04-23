# spark streaming基于kafka receiver导致的乱序问题

本文主要分析spark streaming以recevier方式读取kafka数据时所导致的乱序问题。

背景：为了用kafka存储时序数据，生产者在向kafka发送数据时按设备mdmid作为partition的key。因此，同一设备的数据都在kafka的同一个partition里面。但是在用spark streaming以recevier方式消费数据时，却出现了乱序。

首先先介绍下何谓recevier方式：在spark streaming的任务执行节点executor上启动若干个kafka consumer，这些consumer从kafka拉取数据，形成一个inputStream，后续的数据处理基于该inputStream进行相应处理。
关键业务代码如下：
![biz](https://github.com/wbear1/spark_blog/blob/master/img/receiver/biz.png)

说明：zkQuorum表示kafka的zk地址，appName表示kafka consumer的groupId，topicmap表示消费的每个topic的处理线程数目，本例中只有1个topic，线程数目为3，所以topicmap={"topicName": 3}

下面就分析在该业务代码下，如何产生的数据乱序。

1）spark streaming内部kafka recevier的数据流向示意图如下(图中标示了核心代码所在文件)：
![flow](https://github.com/wbear1/spark_blog/blob/master/img/receiver/flow.png)

- 对于1个recevier而言，会启动一个KafkaConsumer。
![kafkaconsumer](https://github.com/wbear1/spark_blog/blob/master/img/receiver/kafkaconsumer.png)

- 建立一个固定数量的线程池，对consumer接收的数据进行处理。线程池数量就是上面提到的topicmap里面的value。
![threadpool](https://github.com/wbear1/spark_blog/blob/master/img/receiver/threadpool.png)

- 线程池里面的KafkaMessageHandler负责将数据存储到缓冲池buffer中。
![handler](https://github.com/wbear1/spark_blog/blob/master/img/receiver/handler.png)
![handler2](https://github.com/wbear1/spark_blog/blob/master/img/receiver/handler2.png)

- 为了将数据聚焦，会将多条message组成一个block。这时有单独一个线程blockPushingThread负责从上面的缓冲池取数据，产生一个block。blockPushingThread每隔spark.streaming.blockInterval取一次数据，默认为200ms。
![push](https://github.com/wbear1/spark_blog/blob/master/img/receiver/push.png)

- 产生block，会有listener负责把这个block存储到该executor上，具体是memory还是disk则根据用户提交时的配置。同时会复制到其它的executor上。
![storeblock](https://github.com/wbear1/spark_blog/blob/master/img/receiver/storeblock.png)
![storeblock2](https://github.com/wbear1/spark_blog/blob/master/img/receiver/storeblock2.png)

- 而inputStream里面的RDD类型为BlockRDD，则是根据BlockManager获取相应的block。其分区数量等于block的数量。
![block](https://github.com/wbear1/spark_blog/blob/master/img/receiver/block.png)

上面是kafka receiver从拉取数据到最后创建RDD的大致流程，只记录了关键操作，细节可依此查看相应代码，上面的分析是基于spark-core_2.11、spark-streaming-kafka-0-8_2.11和spark-streaming_2.11的2.0.0版本进行的。

2）回过头来我们来看看乱序问题：

- KafkaMessageHandler的处理是通过线程池，因此进入到缓冲池buffer里面的message就可能产生乱序，但即使线程池数量为1，也会乱序，且看下一点；
- 最终产生的BlockRDD的分区数量与kafka分区数量没有任何关系的，因此后续的业务处理会产生suffle过程，乱序在所难免。举个例子：假设BlockRDD有10个分区，kafka的topic有6个分区，BlockRDD的分区是每200ms从kafka的这6个分区获取的数据，即是说BlockRDD的每个分区均存在kafka的每一个分区的数据，后续的处理过程对数据的有序是由kafka的分区来保证的，有shuffle过程必然会导致乱序。
注：BlockRDD的数量=batchTime/spark.streaming.blockInterval，比如：batch处理是2s，blockInterval的200ms，那么blockRDD的分区数量则是10。
