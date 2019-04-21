# shuffle过程

spark的shuffle，顾名思义就是类似洗牌的过程，将数据重新分配，见下图示意。主要包括：shuffle write和shuffle read
![brief](https://github.com/wbear1/spark_blog/blob/master/img/shuffle/brief.png)

###### shuffle write 
从ShuffleMapTask的执行看shuffle write流程

- 1）先介绍下AppendOnlyMap数据结构，类似于HashMap，不支持remove操作。内部采用数组来存储数据，key和value为连续的两个元素，如下图示：
![appenOnlyMap](https://github.com/wbear1/spark_blog/blob/master/img/shuffle/appendOnlyMap.png)

主要有如下几种方法：
1、插入: 如果hash(key)有值，则判断hash(key)+1，如果也有值，继续判断hash(key)+1+2，依次类推。无值则插入
2、获取：位置获取同上
3、扩容：每次插入数据后，若元素数量达到容量的70%（默认值），则容量扩大一倍
4、destructiveSortedIterator：压缩并排序。假定k3 < K1 < k2 < k2'< k2'’,执行该方法返回结果如下所示。
![result](https://github.com/wbear1/spark_blog/blob/master/img/shuffle/result.png)

- 2）shuffle write主要包括以下几个步骤，可参考：https://blog.csdn.net/raintungli/article/details/70807376
  - a、先将record插入到AppendOnlyMap，如果容量达到一定限制，则将AppendOnlyMap中的数据merge-sort-aggerate然后spill到磁盘文件（文件名：temp_shuffle_{uuid}）
![spill](https://github.com/wbear1/spark_blog/blob/master/img/shuffle/spill.png)

  - b、对AppendOnlyMap的元素和存储在磁盘上spilled文件，进行merge-sort-aggerate操作后将数据写到shuffle文件中（文件名：shuffle_{shuffleId}_{mapId}_{reduceId}.data）。
![merge](https://github.com/wbear1/spark_blog/blob/master/img/shuffle/merge.png)  
![merge2](https://github.com/wbear1/spark_blog/blob/master/img/shuffle/merge2.png)  
  
  - c、将各个block的索引写到索引文件（文件名： shuffle_{shuffleId}_{mapId}_{reduceId}.index），索引文件按顺序存储了每个partition的offset。例如：索引文件数据为 0  100  150  150  300  320  320  320, 表示有7个block，各个block字节数为：
![indexFile](https://github.com/wbear1/spark_blog/blob/master/img/shuffle/indexFile.png)  

  - d、shuffle write完成之后，会将shuffle的信息(block的存储位置：host、port、blockId等)记录到MapOutputTracker。
  
###### shuffle read  
从ShuffleRDD的compute来研究shuffle read过程

shuffle read之前executor会先从MapOutputTracker获取shuffle信息，然后创建Fetcher去对应的机器fetch数据，如果数据在本机也有的话，会优先从本机取；如果在远程的话，会通过rpc调用来获取。（这里的rpc调用是异步调用，发送rpc请求，对方机器则开始准备数据，ready之后则回调）
    
shuffle read的时候，是一边fetch数据一边做类似write的merge-sort-aggerate操作。
  