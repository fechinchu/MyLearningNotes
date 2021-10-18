# 网络编程02-Netty

# 1.Netty简介

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

# 2.Netty源码的领域知识

## 2.1.I/O模式

### 2.1.1.三种I/O模式

当我们去食堂吃饭:

* 食堂排队打饭模式:排队在窗口,打好才走;
* 点单,等待被叫模式:等待被叫,好了自己去端;
* 包厢模式:点单后菜直接被端上桌;

![image-20210326104059810](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210326104059810.png)

* 阻塞与非阻塞:
  * 菜没好,要不要等?--->数据就绪前要不要等待?
  * 阻塞:没有数据传过来时,读会阻塞直到有数据;缓冲区满时,写操作也会阻塞.非阻塞遇到这些情况,都是直接返回;
* 同步与异步:
  * 菜好了,谁端?--->数据就绪后,数据操作谁完成?
  * 数据就绪后需要自己去读是同步,数据就绪直接读好再回调给程序是异步.

### 2.1.2.Netty对三种I/O模式的支持

|      ~~BIO->OIO(Deprecated)~~      | NIO                    | NIO                      | NIO                       | ~~AIO(Removed)~~           |
| :--------------------------------: | ---------------------- | ------------------------ | ------------------------- | -------------------------- |
|                                    | COMMON                 | Linux                    | macOS/BSD                 |                            |
| ~~ThreadPerChannelEventLoopGroup~~ | NioEventLoopGroup      | EpollEventLoopGroup      | KQueueEventLoopGroup      | ~~AioEventLoopGroup~~      |
|   ~~ThreadPerChannelEventLoop~~    | NioEventLoop           | EpollEventLoop           | KQueueEventLoop           | ~~AioEventLoop~~           |
|     ~~OioServerSocketChannel~~     | NioServerSocketChannel | EpollServerSocketChannel | KQueueServerSocketChannel | ~~AioServerSocketChannel~~ |
|        ~~OioSocketChannel~~        | NioSocketChannel       | EpollSocketChannel       | KQueueSocketChannel       | ~~AioSocketChannel~~       |

* 为什么deprecate阻塞I/O?
  * 连接数高的情况下:阻塞->耗资源,效率低;
* 为什么删掉已经做好的AIO支持?
  * Windows实现成熟,但是很少用来做服务器;
  * Linux常用来做服务器,但是AIO实现不够成熟;
  * Linux下AIO相比较NIO的性能不明显;

### 2.1.3.为什么Netty有多种NIO实现?

通用的NIO实现(Common)在Linux下也是使用epoll,为什么自己单独实现?

因为实现的更好!

* Netty暴露了更多的可控参数,例如:
  * JDK的NIO默认实现是水平触发;
  * Netty是边缘触发(默认)和水平触发可切换;
  * 注意:边缘触发和水平触发的区别
    * 边缘触发:每当状态变化时，触发一个事件;
    * 水平触发:只要满足条件，就触发一个事件(只要有数据没有被获取，内核就不断通知你);
* Netty实现的垃圾回收更少,性能更好;

### 2.1.4.NIO一定优于BIO么?

* BIO代码简单;
* 特定场景:连接数少,并发度低,

## 2.2.Reactor

### 2.2.1.Reactor简介

| BIO                   | NIO     | AIO      |
| --------------------- | ------- | -------- |
| Thread-Per-Connection | Reactor | Proactor |

Reactor是一种开发模式,模式的核心流程:

> 1. 注册感兴趣的事件;
> 2. 扫描是否有感兴趣的事件发生;
> 3. 事件发生后作出相应的处理;

| client/Server | SocketChannel/ServerSocketChannel | OP_ACCEPT | OP_CONNECT | OP_WRITE | OP_READ |
| ------------- | --------------------------------- | --------- | ---------- | -------- | ------- |
| client        | SocketChannel                     |           | Y          | Y        | Y       |
| server        | ServerSocketChannel               | Y         |            |          |         |
| server        | SocketChannel                     |           |            | Y        | Y       |

### 2.2.2.Thread-Per-Connection模式

 ![image-20210326163547104](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210326163547104.png)

注意:其中的read和sean都是阻塞操作;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210326163730183.png" alt="image-20210326163730183" style="zoom: 33%;" />

### 2.2.3.Reactor模式

####2.2.3.1.单线程模式

![image-20210326163938954](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210326163938954.png)

 

#### 2.2.3.2.多线程模式

![image-20210326164144914](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210326164144914.png)

#### 2.2.3.3.主从多线程模式

![image-20210326164358272](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210326164358272.png)

#### 2.2.3.4.如何在Netty中使用Reactor模式

![image-20210326164551920](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210326164551920.png)

两组EventLoopGroup(Main&Sub)处理不同通道不同的事件:

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210624142218479.png" alt="image-20210624142218479" style="zoom: 50%;" />

## 2.3.Netty的通讯

### 2.3.1.为什么TCP应用中会出现粘包和半包现象?

粘包的主要原因:

* 发送方每次写入数据 < 套接字缓冲区大小;
* 接收方读取套接字缓冲区数据不够及时;

半包的主要原因:

* 发送方写入数据>套接字缓冲区大小;
* 发送的数据大于协议MTU(Maxmum Transmission Unit,最大传输单元),必须拆包;

根本原因:

* TCP是流式协议,消息无边界;
* 注:UDP像邮寄的包裹,虽然一次运输多个,但每个都有界限,一个一个签收,所以无粘包,半包问题;

### 2.3.2.解决问题的根本手段:找出消息的边界

![image-20210326172318599](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210326172318599.png)

 ### 2.3.3.Netty对三种常用封帧方式的支持

![image-20210326173204008](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210326173204008.png)

### 2.3.4.二次编解码

在我们的项目中,除了可选的压缩解压缩之外,还需要一层解码,因为一次解码的结果是字节,需要和项目中所使用的对象做转换,方便使用,这层解码器可以称为"二次解码器",相应的,对应的编码器是为了将Java对象转换成字节流方便存储或传输;

* 一次解码器:ByteToMessageDecoder
  * io.netty.buffer.ByteBuf(原始数据流) -> io.netty.buffer.ByteBuf(用户数据);
* 二次解码器:MessageToMessageDecoder<I>
  * io.netty.buffer.ByteBuf(用户数据)->java Object;

### 2.3.5.Google Protobuf简介与使用

* Protobuf是一个灵活的,搞笑的用于序列化数据的协议;
* 相比较XML和JSON格式,Protobuf更小,更快,更便捷;
* Protobuf是跨语言的,并且自带了一个编译器(protoc),只需要用它编译,可以自动生成Java,python,C++代码,不需要再写其他代码;

## 2.4.keepalive与idle监测

![image-20210326191731948](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210326191731948.png)

### 2.4.1.为什么需要应用层keeplive?

* 协议分层,各层关注点不同:
  * 传输层关注是否"通",应用层关注是否可服务;
* TCP层的keeplive默认关闭,且经过路由等中转设备keepalive包可能被丢弃;
* TCP层的keeplive时间太长;
  * 默认>2小时,虽然可改,但属于系统参数,改动影响所有应用;

> 提示:Http属于应用层协议,但是尝尝听到名词"HTTP Keep-Alive"指的是对长连接和短连接的选择;
>
> * Connection:Keep-Alive长连接(HTTP/1.1 默认长连接,不要带这个header);
> * Connection:Close短连接;

### 2.4.2.Idle监测是什么?

假设开了一个饭店,别人电话订餐,电话通了以后,订餐的说了一堆订单要求,说着说着,对方就不讲话了;你会立即发问吗?并不会.**一般会稍微等待一定时间,在这个时间内看看对方会不会说话(Idle检测)如果还不说,认定对方存在问题(Idle),于是开始发问:"你还在吗"(keepalive),或者问都不问直接挂机(关闭连接)**;

Idle检测,还是负责诊断,诊断后,做出不同的行为,决定Idl监测的最终用途;

* 发送keepalive:一般用来配合keepalive,减少keepalive消息;
  * keepalive设计演进:V1定时keepalve消息->V2空闲监测+判定为Idle时才发keepalive;
    * V1:keepalive消息与服务器正常消息交换完全不关联,定时就发送;
    * V2:有其他数据传输的时候,不发送keepalive,无数据传输超过一定时间,判定为Idle,再发keepalive;

* 直接关闭连接:
  * 快速释放损坏的,恶意的,很久不用的连接,让系统时刻保持最好的状态;
  * 简单粗暴,客户端可能需要重连;
* 实际应用中:结合起来使用.按需keepalive,保证不会空闲,如果空闲,关闭连接;

### 2.4.3.如何在Netty中开启TCP Keepalive和Idle检测

开启keepalive:

* Server端开启TCP keepalive

  ```java
  bootstrap.childOption(ChannelOOption.SO_KEEPLIVE,true;
  bootstrap.childOption(NioChannelOption.of(StandardSocketOptions.SO_KEEPALIVE),true);                                                           
  ```

开启不同的Idle Check:

* ```
  ch.pipeline().addLast("idleCheckHandler",new IdleStateHandler(0,20,0,TimeUnit.SECONDS));
  ```

  ![image-20210328001257958](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210328001257958.png)




