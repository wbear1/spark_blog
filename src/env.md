1、三类rpc调用的处理
2、hold
3、serverconfig/clientconfig/messagestoreconfig


=================

serverconfig:
ASYNC_MASTER/SYNC_MASTER  =>   brokerId: 0

SLAVE => brokerId: >0


listenPort: 默认10911
fastListenPort: 默认10909


messagestoreconfig:
haListenPort: 默认10912


sever/fastServer:
    sendMessageProcessor:
        send/sendV2/sendBatch/consumeSendMsgBack
   
1) 返回ConsumeConcurrentlyStatus.RECONSUME_LATER时
   client会调用consumerSendBack
   
   这里可以修改ConsumeConcurrentlyContext的delayLevel（-1 直接进DLQ，0 由broker来控制延迟，>0 由client来指定延迟等级）
   
2) sendMsgProcessor 
    a、如果level为-1,topic=> %DLQ%_{group}
    b、如果level为>=0,将topic修改为%RETRY%_{group}, level=0的话，level=3+reconsumeTimes
    然后调用store存储消息
3) commitLog在处理时，level>0，则修改topic为SCHEDULE_TOPIC_XXXX，queudId=level-1，同时记录原始的topic和queueId
   消息被写到SCHEDULE_TOPIC_XXXX了
4) 然后由ScheduleMessageService来负责写延迟的消息（搜索topic=SCHEDULE_TOPIC_XXXX），将消息修改回原来的topic(%RETRY%_{group})和queueId
5) pushConsumer在启动时，若是clustering模式，则会订阅%RETRY%_{group}这个topic    
        
    queryMessageProcessor:
        queryMessage/viewMsgByID/

    clientManageProcessor:
        hearbeat/unregisterClient/checkClientConfig
        
    consumerManagementProcessor:
        getConsumeListByGroup/updateConsumeOffset/queryConsumeOffset
        
    endTransactionProcessor:
        endTrans
        
    amdinBrokerProcessor:
        defalut
    
    
server:
    pullMessageProcessor
    

