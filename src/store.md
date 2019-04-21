# Store模块的设计与实现

- [1、存储目录](#1)
- [2、存储设计](#2)
- [3、存储实现](#3)
  - [内存映射文件MappedFile](#3.1)
  - [MappedFileQueue](#3.2)
  - [消息存储CommitLog](#3.3)
  - [消息元数据ConsumeQueue](#3.4)
  - [简单文件Config](#3.5)

<a name="1"></a> 
#### 1、存储目录

先来看看存储目录下具体有哪些文件，对RocketMQ的存储模块有个直观的认识。
![dir](https://github.com/wbear1/rocketmq_blog/blob/master/img/store/dir.png)

+ commitlog  目录下存储了该broker接收的mq的消息，文件名按消息偏移量命名，文件内容按一定格式编码，详细编码后文介绍。如下所示：每个文件大小为1GB，00000000222264557568文件为第207个文件，00000000223338299392文件为第208个文件，前面的0~206个文件，因满足删除策略已被删除，关于删除策略在后文介绍。
![commitlog](https://github.com/wbear1/rocketmq_blog/blob/master/img/store/commitlog.png)

+ consumequeue 目录下存储了该broker的各个queue消息的offset，文件按{topicName}/{queueId}/{offset}目录存储，文件内容按一定格式编码。如下所示：topictest下有4个queue，queueId=0下面有一个文件，该文件记录了topictest下面的第0个queue中的消息在commitLog中的offset。
![consumequeue](https://github.com/wbear1/rocketmq_blog/blob/master/img/store/consumequeue.png)

+ conifg 目录下存储各种配置信息，包括：topic的配置、consumer提交的offset、consumerFilter的配置、subscriptionGroup的配置等等，文件内容为json字符串，因文件都比较小，对文件的读写直接通过inputstream和outputstream进行操作。
![config1](https://github.com/wbear1/rocketmq_blog/blob/master/img/store/config1.png)
![config2](https://github.com/wbear1/rocketmq_blog/blob/master/img/store/config2.png)

* index 目录下存储了消息索引，用于快速查找消息
* checkpoint文件
* lock文件

<a name="2"></a>
#### 2、存储设计

核心设计如下图所示   
![arch](https://github.com/wbear1/rocketmq_blog/blob/master/img/store/arch.png)

其中messageStore为存储模块对外提供的功能接口，DefaultMessageStore为RokcetMQ的默认实现。
CommitLog、ConsumeQueue、config、index、checkpoint为内部实现的几类存储。
最下面的黑色虚框表示使用内存映射文件读写文件，MappedFileQueue表示对一个目录的读写，底层都是使用MappedFile对应一个实际物理文件，出于效率的考虑，设计了AllocateMappedFileService用于提前创建文件。

MessageStore提供的主要方法：写消息、读消息、其中MessageExtBrokerInner为单条消息，MessageExtBatch为多条封装的批量消息
```java

/**
 * Store a message into store.
 *
 * @param msg Message instance to store
 * @return result of store operation.
 */
PutMessageResult putMessage(final MessageExtBrokerInner msg);

/**
 * Store a batch of messages.
 *
 * @param messageExtBatch Message batch.
 * @return result of storing batch messages.
 */
PutMessageResult putMessages(final MessageExtBatch messageExtBatch);

/**
 * Query at most <code>maxMsgNums</code> messages belonging to <code>topic</code> at <code>queueId</code> starting
 * from given <code>offset</code>. Resulting messages will further be screened using provided message filter.
 *
 * @param group Consumer group that launches this query.
 * @param topic Topic to query.
 * @param queueId Queue ID to query.
 * @param offset Logical offset to start from.
 * @param maxMsgNums Maximum count of messages to query.
 * @param messageFilter Message filter used to screen desired messages.
 * @return Matched messages.
 */
GetMessageResult getMessage(final String group, final String topic, final int queueId,
    final long offset, final int maxMsgNums, final MessageFilter messageFilter);
```

 <a name="3"></a>
#### 3、存储实现

<a name="3.1"></a>
##### 内存映射文件MappedFile
初始化MappedFile，主要是将文件映射到MappedByteBuffer，对文件的读写操作就变成对MappedByteBuffer的操作，关于文件的nio操作相关资料比较多，此处不展开。
```java
public void init(final String fileName, final int fileSize,
    final TransientStorePool transientStorePool) throws IOException {
    init(fileName, fileSize);
    this.writeBuffer = transientStorePool.borrowBuffer(); //内容先commit到内存缓冲区，定时flush到disk
    this.transientStorePool = transientStorePool;
}

private void init(final String fileName, final int fileSize) throws IOException {
    this.fileName = fileName;
    this.fileSize = fileSize;
    this.file = new File(fileName);
    this.fileFromOffset = Long.parseLong(this.file.getName());
    boolean ok = false;

    ensureDirOK(this.file.getParent());

    try {
        this.fileChannel = new RandomAccessFile(this.file, "rw").getChannel();
        //将文件映射到MappedByteBuffer对象
        this.mappedByteBuffer = this.fileChannel.map(MapMode.READ_WRITE, 0, fileSize);
        TOTAL_MAPPED_VIRTUAL_MEMORY.addAndGet(fileSize);
        TOTAL_MAPPED_FILES.incrementAndGet();
        ok = true;
    } catch (FileNotFoundException e) {
        log.error("create file channel " + this.fileName + " Failed. ", e);
        throw e;
    } catch (IOException e) {
        log.error("map file " + this.fileName + " Failed. ", e);
        throw e;
    } finally {
        if (!ok && this.fileChannel != null) {
            this.fileChannel.close();
        }
    }
}
```
写数据操作，分为两类：其一为写MessageExt对象，也即CommitLog的消息；其二为写字节数组，即ConsumeQueue对应的数据。处理逻辑类似，下面只介绍MessageExt的写入过程。
在文件最后append，append的内容通过回调函数AppendMessageCallback.doAppend实现。对于commitlog的数据结构后文介绍。
```java
public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {
    assert messageExt != null;
    assert cb != null;

    int currentPos = this.wrotePosition.get(); //获取文件当前写到的位置

    if (currentPos < this.fileSize) {
        //如果开启内存缓冲区，先commit到缓冲区；否则先写到MappedByteBuffer。最终flush到disk的逻辑可参考flush方法
        ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() : this.mappedByteBuffer.slice();
        byteBuffer.position(currentPos);
        AppendMessageResult result = null;
        //通过调用AppendMessageCallback的doAppend方法来实现数据的写入。
        if (messageExt instanceof MessageExtBrokerInner) {
            result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBrokerInner) messageExt);
        } else if (messageExt instanceof MessageExtBatch) {
            result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBatch) messageExt);
        } else {
            return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
        }
        this.wrotePosition.addAndGet(result.getWroteBytes());
        this.storeTimestamp = result.getStoreTimestamp();
        return result;
    }
    log.error("MappedFile.appendMessage return null, wrotePosition: {} fileSize: {}", currentPos, this.fileSize);
    return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
}
```
下面是doAppend的实现，对单条消息的写入。
```java
public AppendMessageResult doAppend(final long fileFromOffset, final ByteBuffer byteBuffer, final int maxBlank,
    final MessageExtBrokerInner msgInner) {
    // STORETIMESTAMP + STOREHOSTADDRESS + OFFSET <br>

    // PHY OFFSET
    long wroteOffset = fileFromOffset + byteBuffer.position();

    this.resetByteBuffer(hostHolder, 8);
    String msgId = MessageDecoder.createMessageId(this.msgIdMemory, msgInner.getStoreHostBytes(hostHolder), wroteOffset);

    // Record ConsumeQueue information
    keyBuilder.setLength(0);
    keyBuilder.append(msgInner.getTopic());
    keyBuilder.append('-');
    keyBuilder.append(msgInner.getQueueId());
    String key = keyBuilder.toString();
    Long queueOffset = CommitLog.this.topicQueueTable.get(key);
    if (null == queueOffset) {
        queueOffset = 0L;
        CommitLog.this.topicQueueTable.put(key, queueOffset);
    }

    // Transaction messages that require special handling
    final int tranType = MessageSysFlag.getTransactionValue(msgInner.getSysFlag());
    switch (tranType) {
        // Prepared and Rollback message is not consumed, will not enter the
        // consumer queuec
        case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
        case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
            queueOffset = 0L;
            break;
        case MessageSysFlag.TRANSACTION_NOT_TYPE:
        case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
        default:
            break;
    }

    /**
     * Serialize message
     */
    final byte[] propertiesData =
        msgInner.getPropertiesString() == null ? null : msgInner.getPropertiesString().getBytes(MessageDecoder.CHARSET_UTF8);

    final int propertiesLength = propertiesData == null ? 0 : propertiesData.length;

    if (propertiesLength > Short.MAX_VALUE) {
        log.warn("putMessage message properties length too long. length={}", propertiesData.length);
        return new AppendMessageResult(AppendMessageStatus.PROPERTIES_SIZE_EXCEEDED);
    }

    final byte[] topicData = msgInner.getTopic().getBytes(MessageDecoder.CHARSET_UTF8);
    final int topicLength = topicData.length;

    final int bodyLength = msgInner.getBody() == null ? 0 : msgInner.getBody().length;

    final int msgLen = calMsgLength(bodyLength, topicLength, propertiesLength);

    // Exceeds the maximum message
    if (msgLen > this.maxMessageSize) {
        CommitLog.log.warn("message size exceeded, msg total size: " + msgLen + ", msg body size: " + bodyLength
            + ", maxMessageSize: " + this.maxMessageSize);
        return new AppendMessageResult(AppendMessageStatus.MESSAGE_SIZE_EXCEEDED);
    }

    // Determines whether there is sufficient free space
    if ((msgLen + END_FILE_MIN_BLANK_LENGTH) > maxBlank) {
        this.resetByteBuffer(this.msgStoreItemMemory, maxBlank);
        // 1 TOTALSIZE
        this.msgStoreItemMemory.putInt(maxBlank);
        // 2 MAGICCODE
        this.msgStoreItemMemory.putInt(CommitLog.BLANK_MAGIC_CODE);
        // 3 The remaining space may be any value
        // Here the length of the specially set maxBlank
        final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
        byteBuffer.put(this.msgStoreItemMemory.array(), 0, maxBlank);
        return new AppendMessageResult(AppendMessageStatus.END_OF_FILE, wroteOffset, maxBlank, msgId, msgInner.getStoreTimestamp(),
            queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);
    }

    //按消息的结构逐一写入各个字段
    // Initialization of storage space
    this.resetByteBuffer(msgStoreItemMemory, msgLen);
    // 1 TOTALSIZE
    this.msgStoreItemMemory.putInt(msgLen);
    // 2 MAGICCODE
    this.msgStoreItemMemory.putInt(CommitLog.MESSAGE_MAGIC_CODE);
    // 3 BODYCRC
    this.msgStoreItemMemory.putInt(msgInner.getBodyCRC());
    // 4 QUEUEID
    this.msgStoreItemMemory.putInt(msgInner.getQueueId());
    // 5 FLAG
    this.msgStoreItemMemory.putInt(msgInner.getFlag());
    // 6 QUEUEOFFSET
    this.msgStoreItemMemory.putLong(queueOffset);
    // 7 PHYSICALOFFSET
    this.msgStoreItemMemory.putLong(fileFromOffset + byteBuffer.position());
    // 8 SYSFLAG
    this.msgStoreItemMemory.putInt(msgInner.getSysFlag());
    // 9 BORNTIMESTAMP
    this.msgStoreItemMemory.putLong(msgInner.getBornTimestamp());
    // 10 BORNHOST
    this.resetByteBuffer(hostHolder, 8);
    this.msgStoreItemMemory.put(msgInner.getBornHostBytes(hostHolder));
    // 11 STORETIMESTAMP
    this.msgStoreItemMemory.putLong(msgInner.getStoreTimestamp());
    // 12 STOREHOSTADDRESS
    this.resetByteBuffer(hostHolder, 8);
    this.msgStoreItemMemory.put(msgInner.getStoreHostBytes(hostHolder));
    //this.msgBatchMemory.put(msgInner.getStoreHostBytes());
    // 13 RECONSUMETIMES
    this.msgStoreItemMemory.putInt(msgInner.getReconsumeTimes());
    // 14 Prepared Transaction Offset
    this.msgStoreItemMemory.putLong(msgInner.getPreparedTransactionOffset());
    // 15 BODY
    this.msgStoreItemMemory.putInt(bodyLength);
    if (bodyLength > 0)
        this.msgStoreItemMemory.put(msgInner.getBody());
    // 16 TOPIC
    this.msgStoreItemMemory.put((byte) topicLength);
    this.msgStoreItemMemory.put(topicData);
    // 17 PROPERTIES
    this.msgStoreItemMemory.putShort((short) propertiesLength);
    if (propertiesLength > 0)
        this.msgStoreItemMemory.put(propertiesData);

    final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
    // Write messages to the queue buffer
    byteBuffer.put(this.msgStoreItemMemory.array(), 0, msgLen);

    AppendMessageResult result = new AppendMessageResult(AppendMessageStatus.PUT_OK, wroteOffset, msgLen, msgId,
        msgInner.getStoreTimestamp(), queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);

    switch (tranType) {
        case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
        case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
            break;
        case MessageSysFlag.TRANSACTION_NOT_TYPE:
        case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
            // The next update ConsumeQueue information
            CommitLog.this.topicQueueTable.put(key, ++queueOffset);
            break;
        default:
            break;
    }
    return result;
}
```
flush方法是指将数据刷盘，方法实现如下，传入参数为需要满足的最小页数，默认为4页（每页为4KB大小）,传入0表示强制刷盘
```java
public int flush(final int flushLeastPages) {
    if (this.isAbleToFlush(flushLeastPages)) {
        if (this.hold()) {
            int value = getReadPosition();

            try {
                //We only append data to fileChannel or mappedByteBuffer, never both.
                if (writeBuffer != null || this.fileChannel.position() != 0) {
                    this.fileChannel.force(false);
                } else {
                    this.mappedByteBuffer.force();
                }
            } catch (Throwable e) {
                log.error("Error occurred when force data to disk.", e);
            }

            this.flushedPosition.set(value);
            this.release();
        } else {
            log.warn("in flush, hold failed, flush offset = " + this.flushedPosition.get());
            this.flushedPosition.set(getReadPosition());
        }
    }
    return this.getFlushedPosition();
}

//判断是否刷盘
private boolean isAbleToFlush(final int flushLeastPages) {
    int flush = this.flushedPosition.get();
    int write = getReadPosition();

    //文件已写完，刷盘
    if (this.isFull()) {
        return true;
    }

    //待刷盘数据大于指定页数，则刷盘
    if (flushLeastPages > 0) {
        return ((write / OS_PAGE_SIZE) - (flush / OS_PAGE_SIZE)) >= flushLeastPages;
    }

    return write > flush;
}
```

读数据操作，根据pos移动到文件的指定的位置，然后返回相应的ByteBuffer。
```java
public SelectMappedBufferResult selectMappedBuffer(int pos, int size) {
    int readPosition = getReadPosition();
    if ((pos + size) <= readPosition) {

        if (this.hold()) {
            ByteBuffer byteBuffer = this.mappedByteBuffer.slice();
            byteBuffer.position(pos);
            ByteBuffer byteBufferNew = byteBuffer.slice();
            byteBufferNew.limit(size);
            return new SelectMappedBufferResult(this.fileFromOffset + pos, byteBufferNew, size, this);
        } else {
            log.warn("matched, but hold failed, request pos: " + pos + ", fileFromOffset: "
                + this.fileFromOffset);
        }
    } else {
        log.warn("selectMappedBuffer request pos invalid, request pos: " + pos + ", size: " + size
            + ", fileFromOffset: " + this.fileFromOffset);
    }

    return null;
}
```

<a name="3.2"></a>
##### MappedFileQueue
从上面的存储目录也能看到，commitlog、consumequeue等都是目录，意味着数据会存储到多个文件中，MappedFileQueue则是应对某类数据的多文件操作，比如文件创建、从目录找到指定文件读写消息。
在store启动时，MappedFileQueue会将load指定目录下的所有文件（非读取，不占用内存）。
```java
public boolean load() {
    File dir = new File(this.storePath);
    File[] files = dir.listFiles();
    if (files != null) {
        // ascending order
        Arrays.sort(files);
        for (File file : files) {

            if (file.length() != this.mappedFileSize) {
                log.warn(file + "\t" + file.length()
                        + " length not matched message store config value, ignore it");
                return true;
            }

            try {
                MappedFile mappedFile = new MappedFile(file.getPath(), mappedFileSize);

                mappedFile.setWrotePosition(this.mappedFileSize);
                mappedFile.setFlushedPosition(this.mappedFileSize);
                mappedFile.setCommittedPosition(this.mappedFileSize);
                this.mappedFiles.add(mappedFile);
                log.info("load " + file.getPath() + " OK");
            } catch (IOException e) {
                log.error("load file " + file + " error", e);
                return false;
            }
        }
    }

    return true;
}
```

在写数据时，先根据offset找到指定文件，然后再写入文件。
```java
public boolean flush(final int flushLeastPages) {
    boolean result = true;
    MappedFile mappedFile = this.findMappedFileByOffset(this.flushedWhere, this.flushedWhere == 0);
    if (mappedFile != null) {
        long tmpTimeStamp = mappedFile.getStoreTimestamp();
        int offset = mappedFile.flush(flushLeastPages);     //
        long where = mappedFile.getFileFromOffset() + offset;
        result = where == this.flushedWhere;
        this.flushedWhere = where;
        if (0 == flushLeastPages) {
            this.storeTimestamp = tmpTimeStamp;
        }
    }

    return result;
}
```

<a name="3.3"></a>
##### 消息存储CommitLog
CommitLog表示RocketMQ broker对接收的消息的处理，为存储模块最核心的实现。首先消息的结构设计如下（从网上搜索，已忘记具体来源。。）
![commitlog1](https://github.com/wbear1/rocketmq_blog/blob/master/img/store/commitlog1.png)
CommitLog内部则是通过上面提到的MappedFile和MappedFileQueue来实现，每个消息文件默认大小为1GB
```java
public CommitLog(final DefaultMessageStore defaultMessageStore) {
    this.mappedFileQueue = new MappedFileQueue(defaultMessageStore.getMessageStoreConfig().getStorePathCommitLog(),
        defaultMessageStore.getMessageStoreConfig().getMapedFileSizeCommitLog(), defaultMessageStore.getAllocateMappedFileService());
    this.defaultMessageStore = defaultMessageStore;

    //同步刷盘和异步刷盘区别处理
    if (FlushDiskType.SYNC_FLUSH == defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
        this.flushCommitLogService = new GroupCommitService();
    } else {
        this.flushCommitLogService = new FlushRealTimeService();
    }

    this.commitLogService = new CommitRealTimeService();

    this.appendMessageCallback = new DefaultAppendMessageCallback(defaultMessageStore.getMessageStoreConfig().getMaxMessageSize());
    batchEncoderThreadLocal = new ThreadLocal<MessageExtBatchEncoder>() {
        @Override
        protected MessageExtBatchEncoder initialValue() {
            return new MessageExtBatchEncoder(defaultMessageStore.getMessageStoreConfig().getMaxMessageSize());
        }
    };
    this.putMessageLock = defaultMessageStore.getMessageStoreConfig().isUseReentrantLockWhenPutMessage() ? new PutMessageReentrantLock() : new PutMessageSpinLock();

}
```

<a name="3.4"></a>
##### 消息元数据ConsumeQueue
当消息写到CommitLog后，对Consumer而言，消息并不可见，需要将消息的元数据写入ConsumeQueue之后（也即是将消息所属的topic、queue、offset等信息写到），Consumer才能消费到这些消息。
每条ConsumeQueue记录为20字节，其结构如下所示
![commitlog1](https://github.com/wbear1/rocketmq_blog/blob/master/img/store/consumequeue1.png)
ConsumeQueue的写入操作是通过DefaultMessageStore的定时任务来完成的，从CommitLog获取当前写入的消息，然后将消息的offset写入到ConsumeQueue指定的文件中。

<a name="3.5"></a>
##### 简单文件Config
Consumer消费到消息的最后offset，topic的相关配置等信息都属于config类数据，数据量比较小，读写比较方便，因此在RocketMQ内对这些数据的存储直接采用json字符串的格式，将文件直接读取或全部写入。
```java
//将文件以字符串读出
public static String file2String(final File file) throws IOException {
    if (file.exists()) {
        byte[] data = new byte[(int) file.length()];
        boolean result;

        FileInputStream inputStream = null;
        try {
            inputStream = new FileInputStream(file);
            int len = inputStream.read(data);
            result = len == data.length;
        } finally {
            if (inputStream != null) {
                inputStream.close();
            }
        }

        if (result) {
            return new String(data);
        }
    }
    return null;
}

//将字符串写入文件
public static void string2File(final String str, final String fileName) throws IOException {

    String tmpFile = fileName + ".tmp";
    string2FileNotSafe(str, tmpFile);

    String bakFile = fileName + ".bak";
    String prevContent = file2String(fileName);
    if (prevContent != null) {
        string2FileNotSafe(prevContent, bakFile);
    }

    File file = new File(fileName);
    file.delete();

    file = new File(tmpFile);
    file.renameTo(new File(fileName));
}

//加载文件
public boolean load() {
    String fileName = null;
    try {
        fileName = this.configFilePath();
        String jsonString = MixAll.file2String(fileName);

        if (null == jsonString || jsonString.length() == 0) {
            return this.loadBak();
        } else {
            this.decode(jsonString);
            log.info("load " + fileName + " OK");
            return true;
        }
    } catch (Exception e) {
        log.error("load " + fileName + " failed, and try to load backup file", e);
        return this.loadBak();
    }
}

public void decode(String jsonString) {
    if (jsonString != null) {
        ConsumerOffsetManager obj = RemotingSerializable.fromJson(jsonString, ConsumerOffsetManager.class);
        if (obj != null) {
            this.offsetTable = obj.offsetTable;
        }
    }
}
```
