# checkpoint原理

checkpoint，无非就是将RDD保存到某个地方（比如HDFS），下次需要时不用重新计算。用法也比较简单，调用RDD.checkpoint()，表明该RDD需要做checkpoint，只是作了标记说明，此时checkpoint状态为Initialized，并没有真正开始做checkpoint操作，这个方法名起的不是很好。
![code1](https://github.com/wbear1/spark_blog/blob/master/img/checkpoint/code1.png)

那什么时候开始真正做checkpoint呢？如果对该RDD使用了action算子（且称之为finalRDD），则在action对应的job之后；如果只对该RDD使用了transform算子(且称之为tempRDD)，则在子RDD的action的job执行之后。
![code2](https://github.com/wbear1/spark_blog/blob/master/img/checkpoint/code2.png)

在rdd.doCheckpoint中，如果配置了spark.checkpoint.checkpointAllMarkedAncestors为true（默认为false），则先对祖先RDD进行checkpoint（上面提到的tempRDD就是在这时被checkpoint的），然后对自己做checkpoint。这个时候checkpoint的状态转变为CheckpointingInProgress，然后才是真正的做checkpoint，checkpoint也是触发一个job，这个job的操作就是将RDD的内容写到指定checkpoint的文件中。
![code3](https://github.com/wbear1/spark_blog/blob/master/img/checkpoint/code3.png)

![code4](https://github.com/wbear1/spark_blog/blob/master/img/checkpoint/code4.png)

当job执行完之后，把checkpoint的内容作为CheckpointRDD保存到rdd的checkpointData中，同时checkpoint的状态转变为Checkpointed，然后把rdd的依赖清除。
![code5](https://github.com/wbear1/spark_blog/blob/master/img/checkpoint/code5.png)

到这里，checkpoint算是结束了。接下来看看是如何利用checkpoint的，看看RDD的getOrCompute方法，如果该RDD的checkpoint状态为checkpointed，则从上面提到过的CheckpointRDD中读取。这里的实现需要注意下： 把CheckpointRDD放到dependencies变量中了，前面提到的依赖是deps变量。
![code6](https://github.com/wbear1/spark_blog/blob/master/img/checkpoint/code6.png)

![code7](https://github.com/wbear1/spark_blog/blob/master/img/checkpoint/code7.png)

![code8](https://github.com/wbear1/spark_blog/blob/master/img/checkpoint/code8.png)

然后看看CheckpointRDD的compute方法（读取checkpoint内容的方法），下面的代码是ReliableCheckpointRDD的实现（把Checkpoint内容存储在HDFS上使用的就是该实现），可以看到就是从指定的checkpoint文件读取。
![code9](https://github.com/wbear1/spark_blog/blob/master/img/checkpoint/code9.png)

![code10](https://github.com/wbear1/spark_blog/blob/master/img/checkpoint/code10.png)

最后总结下checkpoint操作：
1. 先触发Job，然后才做checkpoint，不管是finalRDD还是tempRDD；
2. checkpoint状态分为Initialized, CheckpointingInProgress, Checkpointed，checkpoint完成后会将原有的依赖清除掉；
3. checkpoint操作会重新触发一个job，也就是说该RDD被计算了两次，这个开销还是比较大的，相信后面的版本应该会优化。