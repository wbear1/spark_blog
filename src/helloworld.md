# 初步介绍

简单介绍spark: 一种大数据的处理框架。快速了解一个框架的方法是，使用该框架实现一个HelloWorld。大数据的HelloWorld习惯性使用wordcount，即统计单词数量。代码如下：

```java
public class WordCount {
 
    public static void main(String[] args) {
 
        SparkConf sparkConf = new SparkConf().setAppName("WordCount").setMaster("local[4]");
 
        JavaSparkContext jsc = new JavaSparkContext(sparkConf);
 
 
        //读取目录下所有文件到RDD
        JavaRDD<String> fileRDD = jsc.textFile("/workspace/Energy-OS/trunk/test/ss-test/test-data/wordcount");
 
 
        //将每行按空格拆分到成单词
        JavaRDD<String> wordRDD = fileRDD.flatMap(new FlatMapFunction<String, String>() {
 
            @Override
            public Iterator<String> call(String line) throws Exception {
 
                return Lists.newArrayList(line.split(" ")).iterator();
 
            }
 
        });
 
 
        //将单词映射成（word，1）
        JavaPairRDD<String, Integer> wordCountRDD = wordRDD.mapToPair(new PairFunction<String, String, Integer>() {
 
            @Override
            public Tuple2<String, Integer> call(String word) throws Exception {
 
                return new Tuple2<>(word, 1);
 
            }
 
        });
 
 
        //将相同单词聚合相加，生成结果RDD
        JavaPairRDD<String, Integer> resultRDD = wordCountRDD.reduceByKey(new Function2<Integer, Integer, Integer>() {
 
            @Override
            public Integer call(Integer wordCount1, Integer wordCount2) throws Exception {
 
                return wordCount1 + wordCount2;
 
            }
 
        });
 
 
        //打印结果RDD
        resultRDD.foreach(new VoidFunction<Tuple2<String, Integer>>() {
 
            @Override
            public void call(Tuple2<String, Integer> wordCount) throws Exception {
 
                System.out.println(wordCount._1() + " => " + wordCount._2());
 
            }
 
        });
 
 
    }
 
}
```


待统计目录下的文件如下所示：
![dir](https://github.com/wbear1/spark_blog/blob/master/img/helloworld/dir.png)

执行流程如下图所示：
![flow](https://github.com/wbear1/spark_blog/blob/master/img/helloworld/flow.png)

运行结果如下：
![result](https://github.com/wbear1/spark_blog/blob/master/img/helloworld/result.png)


首先spark是使用集群中多台机器来执行该任务，那接下来就逐一分析：该程序是如何使用spark的各个模块，怎么把spark调起来的，spark怎么拆分任务的，spark各个执行节点是如何执行用户代码的，rdd到底是什么等问题。