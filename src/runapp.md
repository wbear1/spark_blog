# 运行application代码

先回到第一篇提到WordCount程序，其执行流程简要概括如下：
![simpleflow](https://github.com/wbear1/spark_blog/blob/master/img/runapp/simpleflow.png)

RDD(Resilient Distributed Datasets)，是spark的核心抽象，它表示已被分区的、只读的、并提供了丰富操作方式的数据集合。可以简单理解就是一个分布式的array，元素分布在不同的机器节点上。后面我们再回过头来理解。

spark中对RDD的有两类操作：transform算子和action算子。transform算子表示从一个RDD转换为另一个RDD，而action算子是将RDD的值输出。举个例子：上面的flatMap算子，表示w ordRDD是由fileRDD转换而为，将fileRDD的每个元素（一行），按空白字符分隔为多个单词，wordRDD即是由这些单词组成；
foreach算子是表示遍历resultRDD的每个元素，将其打印。

下面将详细介绍这3个步骤：

1. 初始化driver

driver初始化的过程中，主要包括如下核心组件（非全部）的初始化。SparkContext表示一个spark应用的上下文，DAGSchduler用于DAG的构建和将job的提交给TaskScheduler，TaskScheduler负责将job拆分成一个个的task，这些组件后面再逐一分析，有些设计还是挺有参考意义的。
![component](https://github.com/wbear1/spark_blog/blob/master/img/runapp/component.png)

2. 通过RDD的transform算子构建DAG

1）textFile算子：内部只是创建了一个HadoopRDD，所以fileRDD表示的就是一个HadoopRDD。这里注意：只是声明了rdd，并没有实际获取文件内容。
![textFile](https://github.com/wbear1/spark_blog/blob/master/img/runapp/textFile.png)

2）flatMap算子：将父RDD即fileRDD转化成MapPartitionsRDD，转化逻辑就是代码中的FlatMapFunction对象，这里也只是创建了RDD，并没有实际获取内容
![flatMap](https://github.com/wbear1/spark_blog/blob/master/img/runapp/flatMap.png)

3）mapToPair算子：同样地，mapToPair内部也是转化为MapPartitionsRDD

4）educeByKey算子：转化为ShuffledRDD

5）foreach算子：这是action算子，这将会触发SparkContext提交job。
![foreach](https://github.com/wbear1/spark_blog/blob/master/img/runapp/foreach.png)

这些RDD的转化关系如下图所示，左侧是用户变量，右侧是spark内部RDD类型
![tranform](https://github.com/wbear1/spark_blog/blob/master/img/runapp/transform.png)

3. 提交Job

每个action算子都会触发真正job，由上一步知道SparkContext会提交Job，其内部则调用DAGScheduler组件来提交。
![action](https://github.com/wbear1/spark_blog/blob/master/img/runapp/action.png)

那DAGScheduler是如何提交job的呢？下面介绍DAGScheduler提交job的流程。
- 1）DAGScheduler内部采用EventLoop的设计，将不同的Event（如：JobSubmitted、JobCancelled、TaskSetFailed、CompletionEvent等，总共有10多种Event）push到EventQueue中，然后有一个单独的线程来处理这些Event，如果遇到比较耗时的Event，还会把Event的处理交给其它线程。
![eventLoop](https://github.com/wbear1/spark_blog/blob/master/img/runapp/eventLoop.png)

- 2）DAGScheduler对JobSubmitted事件的处理包括两个步骤：a、创建ResultStage；b、提交stage
a、创建ResultStage： DAGScheduler将Job划分不同的stage，划分的依据就是上面介绍的各个RDD的转化依赖关系，因此stage之间也是有依赖关系，而每个stage由多个可以并行执行的task组成。最后根据stage之间的拓扑排序来提交。简单来说，包括两步：构建stage的DAG，提交stage给TaskScheduler。stage有两种：ResultStage和ShuffleMapStage。ResultStage表示的就是最终rdd执行action的过程，ShuffleMapStage表示的则是最终rdd所依赖的rdd的转化过程，界限是RDD之间的宽依赖。上面的例子构建的stage如下所示：
![resultStage](https://github.com/wbear1/spark_blog/blob/master/img/runapp/resultStage.png)  

划分Stage所用算法还是比较巧妙的，使用了DFS和BFS算法。简易流程如下左图如下。下右图举了一个例子，根据stage的划分算法，会生成6个stage，0-4为ShuffleMapStage，5为ResultStage，Stage5的父Stage为Stage1和Stage3。
![resultStage1](https://github.com/wbear1/spark_blog/blob/master/img/runapp/resultStage1.png)  

![resultStage2](https://github.com/wbear1/spark_blog/blob/master/img/runapp/resultStage2.png)  

这部分代码还是很意思的，一并贴上，有兴趣可看一看。
![code1](https://github.com/wbear1/spark_blog/blob/master/img/runapp/code1.png)  
![code2](https://github.com/wbear1/spark_blog/blob/master/img/runapp/code2.png)  
![code3](https://github.com/wbear1/spark_blog/blob/master/img/runapp/code3.png)  
![code4](https://github.com/wbear1/spark_blog/blob/master/img/runapp/code4.png)  
![code5](https://github.com/wbear1/spark_blog/blob/master/img/runapp/code5.png)  
![code6](https://github.com/wbear1/spark_blog/blob/master/img/runapp/code6.png)  

b、提交Stage，流程如下所示：
![stage](https://github.com/wbear1/spark_blog/blob/master/img/runapp/stage.png)

可以看看从Stage构造出来的TaskSet到底是啥玩意，TaskSet就是Task的集合，每个task对应Stage中一个partition的计算，所以每个Stage对应的并发Task数量和partition数量是相同的。下面代码段是每个task的生成。
![code7](https://github.com/wbear1/spark_blog/blob/master/img/runapp/code7.png)  

那每个TaskSet提交给TaskScheduler后，TaskScheduler是如何处理的呢？
![taskScheduler](https://github.com/wbear1/spark_blog/blob/master/img/runapp/taskScheduler.png)

spark提供了两种调度器：FIFO和FAIR，后面写文章专门分析调度器的设计。这里的申请指driver端对资源的管理，由些可见driver是能够时刻知道当前集群的资源状况的。

至此，对driver端如何将application代码解析，然后分解一个个stage(TaskSet)，然后再拆分成一个个Task的流程有大致的了解了。接下来将分析spark单独各个模块的设计。

