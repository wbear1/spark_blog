# 通信模块的设计与实现

不论是broker与nameserver之间，还是broker与client之间，以及broker节点之间，都面临着远程通信的需要。因此RocketMQ将通信模块独立抽出来作为一个独立的子模块，用于各节点之间的通信。相对于其它业务的rpc通信框架，该通信模块的设计与实现都比较简单。主要分为服务端、客户端和消息结构。

- [1、通信消息RemotingCommand](#1)
- [2、服务端RemotingServer](#2)
- [3、客户端RemotingClient](#3)

<a name="3"></a>
#### 1、通信消息RemotingCommand

RemotingCommand的主要字段如下所示
```java
public class RemotingCommand {
    private int code; //标识该消息，主要用于业务标识
    private LanguageCode language = LanguageCode.JAVA;  //标识消息是通过何种语言发出的，在RocketMQ中默认都是java
    private int version = 0;
    private int opaque = requestId.getAndIncrement();
    private int flag = 0; //标识该消息是request还是response
    private String remark;
    private HashMap<String, String> extFields;          //扩展字段，部分功能会使用到，如标识调用的region
    private transient CommandCustomHeader customHeader; //消息头部，大部分的rpc调用只有消息头部。

    //消息header的序列化方法，支持json和自组装二进制，默认采用json
    private SerializeType serializeTypeCurrentRPC = serializeTypeConfigInThisServer;

    private transient byte[] body;                      //消息体，比如producer发送的消息就是属于body
}

//消息头部，根据具体的rpc调用实现相应的消息头部。
public interface CommandCustomHeader {
    //在接收到的二进制消息解码后，调用该方法校验字段是否正确
    void checkFields() throws RemotingCommandException;
}
```

下面罗列了RemotingCommand中request部分code如下所示，对应rpc调用的各种命令
```java
public class RequestCode {

    public static final int SEND_MESSAGE = 10;

    public static final int PULL_MESSAGE = 11;

    public static final int QUERY_MESSAGE = 12;
    public static final int QUERY_BROKER_OFFSET = 13;
    public static final int QUERY_CONSUMER_OFFSET = 14;
    public static final int UPDATE_CONSUMER_OFFSET = 15;
    public static final int UPDATE_AND_CREATE_TOPIC = 17;
    public static final int GET_ALL_TOPIC_CONFIG = 21;
    public static final int GET_TOPIC_CONFIG_LIST = 22;

    public static final int GET_TOPIC_NAME_LIST = 23;
}
```

RemotingCommand有两个核心方法，编码和解码，其实主要就是对消息头部的编码和解码操作
```java
public ByteBuffer encodeHeader() {
        return encodeHeader(this.body != null ? this.body.length : 0);
    }

public ByteBuffer encodeHeader(final int bodyLength) {
    // 1> header length size
    int length = 4;

    // 2> header data length
    byte[] headerData;
    headerData = this.headerEncode(); //其实就是把CommandCustomHeader编码成json字符串对应的字节数组

    length += headerData.length;

    // 3> body data length
    length += bodyLength;

    //这里写的有点啰嗦，主要还是为了可读性更强一些
    ByteBuffer result = ByteBuffer.allocate(4 + length - bodyLength);

    // length
    result.putInt(length);

    // header length
    result.put(markProtocolType(headerData.length, serializeTypeCurrentRPC));

    // header data
    result.put(headerData);

    result.flip();

    return result;
}

private byte[] headerEncode() {
    this.makeCustomHeaderToNet();
    if (SerializeType.ROCKETMQ == serializeTypeCurrentRPC) {
        return RocketMQSerializable.rocketMQProtocolEncode(this);
    } else {
        return RemotingSerializable.encode(this);
    }
}

public abstract class RemotingSerializable {
    private final static Charset CHARSET_UTF8 = Charset.forName("UTF-8");

    public static byte[] encode(final Object obj) {
        final String json = toJson(obj, false);
        if (json != null) {
            return json.getBytes(CHARSET_UTF8);
        }
        return null;
    }
}

public static byte[] markProtocolType(int source, SerializeType type) {
    byte[] result = new byte[4];

    result[0] = type.getCode();
    result[1] = (byte) ((source >> 16) & 0xFF);
    result[2] = (byte) ((source >> 8) & 0xFF);
    result[3] = (byte) (source & 0xFF);
    return result;
}
```
消息编码后的结果如下所示：
![command](https://github.com/wbear1/rocketmq_blog/blob/master/img/remoting/command.png)


<a name="1"></a> 
#### 2、服务端RemotingServer

RemotingServer提供服务，支持同步调用、带回调的异步调用、不带回调的异步调用，针对不同的消息通过code来区分，然后使用不同的processor对消息进行处理。

```java
public interface RemotingServer extends RemotingService {

    //注册处理器：消息中有code字段用于标识消息类型，根据code来选择processor对消息进行处理，executor为执行processor逻辑的线程池
    void registerProcessor(final int requestCode, final NettyRequestProcessor processor,
        final ExecutorService executor);

    void registerDefaultProcessor(final NettyRequestProcessor processor, final ExecutorService executor);

    int localListenPort();

    Pair<NettyRequestProcessor, ExecutorService> getProcessorPair(final int requestCode);

    //同步调用
    RemotingCommand invokeSync(final Channel channel, final RemotingCommand request,
        final long timeoutMillis) throws InterruptedException, RemotingSendRequestException,
        RemotingTimeoutException;

    //异步调用，完成后执行回调函数
    void invokeAsync(final Channel channel, final RemotingCommand request, final long timeoutMillis,
        final InvokeCallback invokeCallback) throws InterruptedException,
        RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException;

    //异步调用，不关注结果
    void invokeOneway(final Channel channel, final RemotingCommand request, final long timeoutMillis)
        throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException,
        RemotingSendRequestException;

}
```

RocketMQ的通信模块基于Netty提供了默认的NettyRemotingServer实现。主要是对消息的编解码NettyEnCoder和NettyDecoder以及消息的处理NettyServerProcessor。连接活跃的默认时间为120s，超时服务端主动关闭连接。
```java
public void start() {
    this.defaultEventExecutorGroup = new DefaultEventExecutorGroup(
        nettyServerConfig.getServerWorkerThreads(),
        new ThreadFactory() {

            private AtomicInteger threadIndex = new AtomicInteger(0);

            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r, "NettyServerCodecThread_" + this.threadIndex.incrementAndGet());
            }
        });

    ServerBootstrap childHandler =
        this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector)
            .channel(useEpoll() ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
            .option(ChannelOption.SO_BACKLOG, 1024)
            .option(ChannelOption.SO_REUSEADDR, true)
            .option(ChannelOption.SO_KEEPALIVE, false)
            .childOption(ChannelOption.TCP_NODELAY, true)
            .childOption(ChannelOption.SO_SNDBUF, nettyServerConfig.getServerSocketSndBufSize())
            .childOption(ChannelOption.SO_RCVBUF, nettyServerConfig.getServerSocketRcvBufSize())
            .localAddress(new InetSocketAddress(this.nettyServerConfig.getListenPort()))
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline()
                        .addLast(defaultEventExecutorGroup, HANDSHAKE_HANDLER_NAME,
                            new HandshakeHandler(TlsSystemConfig.tlsMode))
                        .addLast(defaultEventExecutorGroup,
                            new NettyEncoder(),
                            new NettyDecoder(),
                            new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),
                            new NettyConnectManageHandler(),
                            new NettyServerHandler()
                        );
                }
            });

    if (nettyServerConfig.isServerPooledByteBufAllocatorEnable()) {
        childHandler.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
    }

    try {
        ChannelFuture sync = this.serverBootstrap.bind().sync();
        InetSocketAddress addr = (InetSocketAddress) sync.channel().localAddress();
        this.port = addr.getPort();
    } catch (InterruptedException e1) {
        throw new RuntimeException("this.serverBootstrap.bind().sync() InterruptedException", e1);
    }

    if (this.channelEventListener != null) {
        this.nettyEventExecutor.start();
    }

    this.timer.scheduleAtFixedRate(new TimerTask() {

        @Override
        public void run() {
            try {
                //检查超时请求，若超时则返回结果或执行相应的回调方法
                NettyRemotingServer.this.scanResponseTable();
            } catch (Throwable e) {
                log.error("scanResponseTable exception", e);
            }
        }
    }, 1000 * 3, 1000);
}
```

NettyEncoder负责将消息编码成二进制，核心实现如下：
```java
public class NettyEncoder extends MessageToByteEncoder<RemotingCommand> {
    private static final InternalLogger log = InternalLoggerFactory.getLogger(RemotingHelper.ROCKETMQ_REMOTING);

    @Override
    public void encode(ChannelHandlerContext ctx, RemotingCommand remotingCommand, ByteBuf out)
        throws Exception {
        try {
            //对header编码
            ByteBuffer header = remotingCommand.encodeHeader();
            out.writeBytes(header);
            byte[] body = remotingCommand.getBody();
            if (body != null) {
                out.writeBytes(body);
            }
        } catch (Exception e) {
            log.error("encode exception, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()), e);
            if (remotingCommand != null) {
                log.error(remotingCommand.toString());
            }
            RemotingUtil.closeChannel(ctx.channel());
        }
    }
}
```

消息处理NettyServerProcessor，后文介绍nameserver和broker的实现时，其中各种业务的处理均是对该接口的实现。
```java
public interface NettyRequestProcessor {
    //对client发过来的request处理方法
    RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request)
        throws Exception;

    //接收到request之后，首先调用rejectRequest判断是否拒绝该请求。比如当前服务繁忙，则可拒绝该请求
    boolean rejectRequest();
}
```

<a name="2"></a>
#### 3、客户端RemotingClient

RemotingClient的实现与RemotingServer类似，也支持三类调用。
```java
public interface RemotingClient extends RemotingService {

    void updateNameServerAddressList(final List<String> addrs);

    List<String> getNameServerAddressList();

    RemotingCommand invokeSync(final String addr, final RemotingCommand request,
        final long timeoutMillis) throws InterruptedException, RemotingConnectException,
        RemotingSendRequestException, RemotingTimeoutException;

    void invokeAsync(final String addr, final RemotingCommand request, final long timeoutMillis,
        final InvokeCallback invokeCallback) throws InterruptedException, RemotingConnectException,
        RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException;

    void invokeOneway(final String addr, final RemotingCommand request, final long timeoutMillis)
        throws InterruptedException, RemotingConnectException, RemotingTooMuchRequestException,
        RemotingTimeoutException, RemotingSendRequestException;

    void registerProcessor(final int requestCode, final NettyRequestProcessor processor,
        final ExecutorService executor);

    void setCallbackExecutor(final ExecutorService callbackExecutor);

    ExecutorService getCallbackExecutor();

    boolean isChannelWritable(final String addr);
}
```


