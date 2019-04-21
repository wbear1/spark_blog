# spark准备运行环境

从提交application，到最终代码的运行，spark本身是需要准备环境的。运行环境是由执行模式来决定的，spark支持多种执行模式，有本地执行的local模式、有只依赖于spark的standalone client模式和standalone cluster模式、有资源管理系统yarn上执行的yarn client模式和yarn cluster模式、也有在mesos上执行的mesos client模式和mesos cluster模式。目前绝大部分公司都支持多种大数据计算框架，通常会选择资源管理系统来承载这些计算框架，主流的系统主要有yarn和mesos。我们公司目前使用的是yarn，所以本文主要介绍yarn cluster模式下，spark是如何准备运行环境的。

先介绍几个yarn相关的基本概念：

ResourceManager(简称：RM): yarn的master，负责管理和分配yarn集群资源 
NodeManager(简称：NM): yarn的slave，负责管理yarn集群中每个节点
ApplicationMaster(简称：AM): 管理在yarn上运行的应用程序的实例，每个应用程序对应一个AM

下图简单示意这几个概念：
![concept](https://github.com/wbear1/spark_blog/blob/master/img/env/concept.png)

spark相关的概念：

driver: 相当于master，负责管理应用的任务管理和监控
executor: 相当于slave，负责任务的执行


首先了解下上一篇的wordcount程序是如何提交的：先将程序打成jar包，假设为wordcount.jar，然后在gateway机器上通过spark提供的spark-submit脚本提交。提交命令如下：
spark-submit --name MyWordCount --master yarn --deloy-mode cluster --class WordCount wordcount.jar

这条命令执行后，就触发了spark开始准备运行环境。执行过程如下：
![flow](https://github.com/wbear1/spark_blog/blob/master/img/env/flow.png)

- 1、执行spark-submit后，client先检查yarn集群资源（cpu/memory）是否满足需求，若满足则向ResourceManager(后面简称RM)提交应用
- 2、RM则从所有NodeManager（后面简称NM）中挑出一个，作为ApplicationMaster(后面简称AM)。AM启动时主要两件事情：
  - 启动driver
  - 向RM反向注册AM，请求RM分配executor资源
- 3、RM收到请求后，分配资源后通知AM，AM则将各个NM上的executor启动
- 4、executor启动后会反向AM注册，表示启动成功

到这里，spark的运行环境就准备好了，接下来将分析driver如何将用户代码发布到各个executor执行。