# Netty 核心组件

Channel
    相当于一个Socket，对底层等读写都是使用一个Channel
    是线程安全的，可以同时很多线程使用它
EventLoop
    为Channel处理IO操作，一个EventLoop可以为多个Channel服务。
EventLoopGroup
    包含多个EventLoop


ChannelFuture
    所有的io操作返回都是异步等，需要在稍后确定操作结果

ChannelHandler
    当发生一个事件的时候，回调对应等方法
    - ChannelOutboundHandlerAdapter
    - ChannelInboundHandler
ChannelPipeline
    维护channelHandler的一个责任链
ChannelHandlerContext
ChannelInitializer
    当channel建立的时候，这个类会负责处理channelPipeline， 一般在这里添加Handler

ByteBuf
netty的数据处理容器
    使用模式：
    - 堆缓冲区
        - hasArray, 然后调用array方法获取byte[]
    - 直接缓冲区
        - 可以避免每次IO操作前后，将内容复制 再中间缓冲区 和 缓冲区之间复制
        - !hasArray, 然后调用 getBytes来复制到 一个byte数组中
    - 复合缓冲区

ByteBufAllocator
 实现了ByteBuf的池化

注意点：
netty 处理耗时的任务逻辑，是不能在IO线程处理，因为这会造成堵塞，将会严重影响性能。那怎么区分IO线程呢? 
答案就是EventLoop，EventLoop用来处理IO线程，因此耗时任务的handler不要在EventLoop里面处理。

 pipeline.addLast(group, "handler", new MyBusinessLogicHandler());