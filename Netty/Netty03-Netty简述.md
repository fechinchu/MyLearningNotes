# Netty03-Netty简述

# 1.Netty概述

Netty是由Trustin Lee(韩国,Line公司)

* 本质:网络应用程序框架;
* 实现:异步,事件驱动;
* 特效:高性能,可维护,快速开发;
* 用途:开发服务器和客户端;

![image-20210325134752696](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210325134752696.png)

Netty所用的项目:参考:https://netty.io/wiki/adopters.html

* Ali seata;
* Apache Dubbo;
* Elasticsearch;
* Netflix Ribbon;
* Spring5;
* Zookeeper;

Netty重要的四个内容:

* Reactor线程模型:一种高性能的多线程程序设计思路;
* Netty中自定义的Channel概念,
* ChannelPipeline职责链设计模式:事件处理机制;
* 内存管理:增强的ByteBuf缓冲区;

Netty的优势:

* Netty vs NIO，工作量大，bug 多
  * 需要自己构建协议
  * 解决 TCP 传输问题，如粘包、半包
  * epoll 空轮询导致 CPU 100%
  * 对 API 进行增强，使之更易用，如 FastThreadLocal替代ThreadLocal，ByteBuf替代ByteBuffer
* Netty vs 其它网络应用框架
  * Mina 由 apache 维护，将来 3.x 版本可能会有较大重构，破坏 API 向下兼容性，Netty 的开发迭代更迅速，API 更简洁、文档更优秀
  * 久经考验，16年，Netty 版本
    * 2.x 2004
    * 3.x 2008
    * 4.x 2013
    * 5.x 已废弃（没有明显的性能提升，维护成本高）

# 2.HelloWorld Demo

## 2.1.服务器端

```java
public class HelloServer {
    public static void main(String[] args) {
        // 1. 启动器，负责组装 netty 组件，启动服务器
        new ServerBootstrap()
            // 2. BossEventLoop, WorkerEventLoop(selector,thread), group 组
            .group(new NioEventLoopGroup())
            // 3. 选择 服务器的 ServerSocketChannel 实现
            .channel(NioServerSocketChannel.class)
            // 4. boss 负责处理连接 worker(child) 负责处理读写，决定了 worker(child) 能执行哪些操作（handler）
            .childHandler(
                    // 5. channel 代表和客户端进行数据读写的通道 Initializer 初始化，负责添加别的 handler
                new ChannelInitializer<NioSocketChannel>() {
                @Override
                protected void initChannel(NioSocketChannel ch) throws Exception {
                    // 6. 添加具体 handler
                    ch.pipeline().addLast(new LoggingHandler());
                    ch.pipeline().addLast(new StringDecoder()); // 将 ByteBuf 转换为字符串
                    ch.pipeline().addLast(new ChannelInboundHandlerAdapter() { // 自定义 handler
                        @Override // 读事件
                        public void channelRead(ChannelHandlerContext ctx,Object msg) throws Exception {
                            System.out.println(msg); // 打印上一步转换好的字符串
                        }
                    });
                }
            })
            // 7. 绑定监听端口
            .bind(8080);
    }
}
```

## 2.2.客户端

```java
public class HelloClient {
    public static void main(String[] args) throws InterruptedException {
        // 1. 启动类
        new Bootstrap()
            // 2. 添加 EventLoop
            .group(new NioEventLoopGroup())
            // 3. 选择客户端 channel 实现
            .channel(NioSocketChannel.class)
            // 4. 添加处理器
            .handler(new ChannelInitializer<NioSocketChannel>() {
                @Override // 在连接建立后被调用
                protected void initChannel(NioSocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new StringEncoder());
                }
            })
            // 5. 连接到服务器
            .connect(new InetSocketAddress("localhost", 8080))
            .sync() // Netty 中很多方法都是异步的，如 connect，这时需要使用 sync 方法等待 connect 建立连接完毕
            .channel() // 获取 channel 对象，它即为通道抽象，可以进行数据读写操作
            // 6. 向服务器发送数据
            .writeAndFlush("hello, world");
    }
}
```

## 2.3.执行流程

![image-20211122161939921](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211122161939921.png)

# 3.Netty组件

## 3.1.EventLoop与EventLoopGroup

* EventLoop:事件循环对象
  * EventLoop 本质是一个单线程执行器（同时维护了一个 Selector），里面有 run 方法处理 Channel 上源源不断的 io 事件;
  * 继承关系如下:
    * 一条线是继承自 j.u.c.ScheduledExecutorService 因此包含了线程池中所有的方法
    * 另一条线是继承自 netty 自己的 OrderedEventExecutor
      * 提供了 boolean inEventLoop(Thread thread) 方法判断一个线程是否属于此 EventLoop
      * 提供了 parent 方法来看看自己属于哪个 EventLoopGroup;
* EventLoopGroup
  * EventLoopGroup 是一组 EventLoop，Channel 一般会调用 EventLoopGroup 的 register 方法来绑定其中一个 EventLoop，后续这个 Channel 上的 io 事件都由此 EventLoop 来处理（保证了 io 事件处理时的线程安全）;
  * 继承自 netty 自己的 EventExecutorGroup
    * 实现了 Iterable 接口提供遍历 EventLoop 的能力
    * 另有 next 方法获取集合中下一个 EventLoop

### 3.1.1.EventLoopGroup的基础用法

```java
@Slf4j
public class TestEventLoop {
    public static void main(String[] args) {
        // 1. 创建事件循环组
        EventLoopGroup group = new NioEventLoopGroup(2); // io 事件，普通任务，定时任务
//        EventLoopGroup group = new DefaultEventLoopGroup(); // 普通任务，定时任务
        // 2. 获取下一个事件循环对象
        System.out.println(group.next());
        System.out.println(group.next());
        System.out.println(group.next());
        System.out.println(group.next());

        // 3. 执行普通任务
        group.next().execute(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.debug("ok!!!");
        });

        // 4. 执行定时任务
        group.next().scheduleAtFixedRate(() -> {
            log.debug("ok");
        }, 0, 1, TimeUnit.SECONDS);

    }
}
```

### 3.1.2.EventLoopGroup Reactor

