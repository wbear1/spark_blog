# Broker的设计与实现一（SendMessage）

从之前介绍的HelloWorld程序我们知道，通过DefaultMQProducer可将消息发送到broker。那接下来主要介绍：1）client端是如何将消息发送的；2）broker接收到消息是如何处理和存储的

- [1、client端消息发送](#1)
  - [NameServer Cluster](#1.1)
- [2、broker端消息处理](#2)

<a name="1"></a> 
#### 1、client端消息发送
首先简单介绍下client端的架构
![client](https://github.com/wbear1/rocketmq_blog/blob/master/img/broker/1/client.png)

以同步发送消息方法send(Message msg)为例，介绍整个流程：
![client1](https://github.com/wbear1/rocketmq_blog/blob/master/img/broker/1/client1.png)

1. 获取TopicPublishInfo，初次获取通过请求namesvr后，缓存在DefaultProducerImpl中；
2. 选择要发送的messageQueue：若未指定latencyFaultEnable，默认为false，那么会随机选择queue来发送；如果指定为true，则有一定选择策略。MQFaultStrategy负责管理，由latencyMax和notAvailableDuration来指定，比如这次选择将message发送到brokerA，latency为3000ms，那么则会标记brokerA的notAvaibleDuration为180000ms，下次选择的时候：
1）获取一个可用的broker并且brokerName=lastBrokerName（主要针对重试发送）；
2）如果第一步没有，则不考虑brokerName=lastBrokerName，选择一个相对较好的broker，根据latency排序，取一个latency相对较小的broker
3）随机选择一个broker
3. 从MQClientInstance获取broker地址，根据配置选择是否通过vip通道发送消息，这里的vip通道设计大概是为了发送消息能够快速响应，与拉取消息的通道分开。
4. 发送消息，最终通过RemotingClient同步发送；
5. 将message发送完后，记录发送延迟时间，然后去更新MQFaultStrategy，若消息发送到broker的延迟过大，则减少发送给该broker的消息数


<a name="2"></a> 
#### 2、broker端消息处理


