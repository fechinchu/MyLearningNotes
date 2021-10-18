# 网络编程01-BIO与NIO

# 1.TCP/UDP协议

## 1.1.OSI网络七层模型

为使不同计算机厂家的计算机能够互相通信,以便在更大的范围内建立计算机网络,有必要建立一个国际范围的网络体系结构标准;

![image-20201204163446933](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20201204163446933.png)

各层的主要功能

* 低三层
  * 物理层:使原始的数据比特流能在物理介质上传输;
  * 数据链路层:通过检验,确认和反馈重发等手段,形成稳定的数据链路;
  * 网络层:进行路由选择和流量控制.(IP协议);
* * 传输层:提供可靠的端口到端口的数据传输服务(TCP/UDP协议)
* 高三层
  * 会话层:负责建立,管理和终止进程之间的会话和数据交换;
  * 表示层:负责数据格式转化,数据加密与解密,压缩与解压缩等;
  * 应用层:为用户的应用进程提供网络服务;

## 1.2.传输控制协议TCP

传输控制协议(TCP)是Internet一个重要的传输层协议.TCP提供面向连接,可靠,有序,字节流传输服务.应用程序在使用TCP之前,必须先建立TCP连接.

![image-20201204165407421](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20201204165407421.png)

## 1.3.TCP握手机制

![image-20201204170026520](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20201204170026520.png)

## 1.4.用户数据报协议UDP

用户数据报协议UDP是Internet传输层协议.提供无连接,不可靠,数据报尽力传输服务.

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20201204171924600.png" alt="image-20201204171924600" style="zoom: 25%;" />

开发应用人员在UDP上构建应用,需要关注以下几点:

1. 应用进程更容易控制发送什么数据以及何时发送;
2. 无需建立连接;
3. 无连接状态;
4. 首部开销小;

## 1.5.TCP和UDP对比

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20201204172734423.png" alt="image-20201204172734423" style="zoom:33%;" />

## 1.6.Socket编程

Internet中应用最广泛的网络应用编程接口,实现与3种底层协议接口:

* 数据报类型套接字SOCK_DGRAM(面向UDP接口);

* 流式套接字SOCK_STREAM(面向TCP接口);
* 原始套接字SOCK_RAW(面向网络层协议接口IP,ICMP等);

主要Socket API及其调用过程

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20201204173513721.png" alt="image-20201204173513721" style="zoom:33%;" />

Socket API函数定义

* `listen()`,`accept()`函数只能用于服务器端;
* `connect()`函数只能用于客户端;
* `socket()`,`bind()`,`send()`,`recv()`,`sendto()`,`recvfrom()`,`close()`;

## 1.7.三种I/O模式

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

**阻塞(blocking) IO**:资源不可用时,IO请求一直阻塞,直到反馈结果(有数据或超时);

**非阻塞(non-blocking) IO**:资源不可用时,IO请求离开返回,返回数据标识资源不可用;

**同步(synchronous) IO**:应用阻塞在发送或接收数据的状态,直到数据成功传输或返回失败;

**异步(asynchronous) IO**:应用发送或接收数据后立刻返回,实际处理是异步执行的;

阻塞和非阻塞式获取资源的方式,同步/异步是程序如何处理资源的逻辑设计.

BIO网络编程中使用的API:`ServerSocket#accept`,`InputStream#read`都是阻塞的API.操作系统底层API中,默认Socket操作都是Blocking型,`send/recv`等接口都是阻塞的.阻塞导致在处理网络I/O时,一个线程只能处理一个网络连接.

# 2.BIO网络编程

1. BIO Server

~~~java
public class BIOServer {

    public static void main(String[] args) throws Exception {
        ServerSocket serverSocket = new ServerSocket(8080);
        System.out.println("服务器启动成功");
        while (!serverSocket.isClosed()) {
            Socket request = serverSocket.accept();// 阻塞
            System.out.println("收到新连接 : " + request.toString());
            try {
                // 接收数据、打印
                InputStream inputStream = request.getInputStream(); // net + i/o
                BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream, "utf-8"));
                String msg;
                while ((msg = reader.readLine()) != null) { // 没有数据，阻塞
                    if (msg.length() == 0) {
                        break;
                    }
                    System.out.println(msg);
                }
                System.out.println("收到数据,来自："+ request.toString());
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                try {
                    request.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        serverSocket.close();
    }
}
~~~

2. BIO Client

~~~java
public class BIOClient {
	private static Charset charset = Charset.forName("UTF-8");

	public static void main(String[] args) throws Exception {
		Socket s = new Socket("localhost", 8080);
		OutputStream out = s.getOutputStream();

		Scanner scanner = new Scanner(System.in);
		System.out.println("请输入：");
		String msg = scanner.nextLine();
		out.write(msg.getBytes(charset)); // 阻塞，写完成
		scanner.close();
		s.close();
	}
}
~~~

3. BIO Server(多线程)

~~~java
public class BIOServer1 {
    private static ExecutorService threadPool = Executors.newCachedThreadPool();

    public static void main(String[] args) throws Exception {
        ServerSocket serverSocket = new ServerSocket(8080);
        System.out.println("tomcat 服务器启动成功");
        while (!serverSocket.isClosed()) {
            Socket request = serverSocket.accept();
            System.out.println("收到新连接 : " + request.toString());
            threadPool.execute(() -> {
                try {
                    // 接收数据、打印
                    InputStream inputStream = request.getInputStream();
                    BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream, "utf-8"));
                    String msg;
                    while ((msg = reader.readLine()) != null) { // 阻塞
                        if (msg.length() == 0) {
                            break;
                        }
                        System.out.println(msg);
                    }
                    System.out.println("收到数据,来自："+ request.toString());
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    try {
                        request.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
        serverSocket.close();
    }
}
~~~

4. BIO Server(支持Http协议)

~~~java
public class BIOServer2 {

    private static ExecutorService threadPool = Executors.newCachedThreadPool();

    public static void main(String[] args) throws Exception {
        ServerSocket serverSocket = new ServerSocket(8080);
        System.out.println("服务器启动成功");
        while (!serverSocket.isClosed()) {
            Socket request = serverSocket.accept();
            System.out.println("收到新连接 : " + request.toString());
            threadPool.execute(() -> {
                try {
                    // 接收数据、打印
                    InputStream inputStream = request.getInputStream();
                    BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream, "utf-8"));
                    String msg;
                    while ((msg = reader.readLine()) != null) {
                        if (msg.length() == 0) {
                            break;
                        }
                        System.out.println(msg);
                    }

                    System.out.println("收到数据,来自："+ request.toString());
                    // 响应结果 200
                    OutputStream outputStream = request.getOutputStream();
                    outputStream.write("HTTP/1.1 200 OK\r\n".getBytes());
                    outputStream.write("Content-Length: 11\r\n\r\n".getBytes());
                    outputStream.write("Hello World".getBytes());
                    outputStream.flush();
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    try {
                        request.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
        serverSocket.close();
    }
}
~~~

# 3.NIO网络编程

NIO中有三个核心组件

* Buffer缓冲区;
* Channel通道;
* Selector选择器;

## 3.1.Buffer缓冲区

缓冲区本质上是一个可以写入数据的内存块(类似数组),然后可以再次读取.此内存块包含在NIO Buffer对象中,该对象提供了一组方法,可以更轻松的使用内存块.相比较直接对数据的操作,Buffer API更加容易操作和管理;

使用Buffer进行数据写入与读取,需要进行如下四个步骤:

1. 将数据写入缓冲区;
2. 调用`buffer.flip()`,转化为读取模式;
3. 缓冲区读取数据;
4. 调用`buffer.clear()`或`buffer.compact()`清除缓冲区;

### 3.1.1.Buffer工作原理

Buffer三个重要属性:

1. capacity容量:作为一个内存块,Buffer具有一定的固定大小,也称为"容量";
2. position位置:写入模式时代表写入数据的位置,读取模式时代表读取数据的位置;
3. limit限制:写入模式,限制等于buffer的容量.读取模式下,limit等于写入的数据量;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20201205155937398.png" alt="image-20201205155937398" style="zoom:33%;" />

~~~java
public class BufferDemo {
    public static void main(String[] args) {
        // 构建一个byte字节缓冲区，容量是4
        //堆内
        //ByteBuffer allocate = ByteBuffer.allocate(4);
        //堆外
        ByteBuffer byteBuffer = ByteBuffer.allocateDirect(4);
        // 默认写入模式，查看三个重要的指标
        //初始化：capacity容量：4, position位置：0, limit限制：4
        System.out.println(String.format("初始化：capacity容量：%s, position位置：%s, limit限制：%s", byteBuffer.capacity(),
                byteBuffer.position(), byteBuffer.limit()));
        // 写入3字节的数据
        byteBuffer.put((byte) 1);
        byteBuffer.put((byte) 2);
        byteBuffer.put((byte) 3);
        // 再看数据
        //写入3字节后，capacity容量：4, position位置：3, limit限制：4
        System.out.println(String.format("写入3字节后，capacity容量：%s, position位置：%s, limit限制：%s", byteBuffer.capacity(),
                byteBuffer.position(), byteBuffer.limit()));

        // 转换为读取模式(不调用flip方法，也是可以读取数据的，但是position记录读取的位置不对)
        System.out.println("#######开始读取");
        byteBuffer.flip();
        byte a = byteBuffer.get();
        System.out.println(a);//print:1
        byte b = byteBuffer.get();
        System.out.println(b);//print:2
        //读取2字节数据后，capacity容量：4, position位置：2, limit限制：3
        System.out.println(String.format("读取2字节数据后，capacity容量：%s, position位置：%s, limit限制：%s", byteBuffer.capacity(),
                byteBuffer.position(), byteBuffer.limit()));

        // 继续写入3字节，此时读模式下，limit=3，position=2.继续写入只能覆盖写入一条数据
        // clear()方法清除整个缓冲区。compact()方法仅清除已阅读的数据。转为写入模式
        // 如果写入超出Buffer大小,就会报错"java.nio.BufferOverflowException";
        byteBuffer.compact();
        byteBuffer.put((byte) 3);
        byteBuffer.put((byte) 4);
        byteBuffer.put((byte) 5);
        //最终的情况，capacity容量：4, position位置：4, limit限制：4
        System.out.println(String.format("最终的情况，capacity容量：%s, position位置：%s, limit限制：%s", byteBuffer.capacity(),
                byteBuffer.position(), byteBuffer.limit()));

        // rewind() 重置position为0
        // mark() 标记position的位置
        // reset() 重置position为上次mark()标记的位置

    }
}
~~~

### 3.1.2.ByteBuffer内存类型

ByteBuffer为性能关键型代码提供了直接内存(direct堆外)和非直接内存(heap堆)两种实现.

堆外内存获取的方式:`ByteBuffer directByteBuffer = ByteBuffer.allocateDirect(noBytes)`;

如下是源码:

![image-20210623144812367](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210623144812367.png)

![image-20210623145245846](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210623145245846.png)

堆外内存的好处:

1. 进行网络IO或者文件IO时比heapBuffer少一次拷贝.(file/socket --> OS memory --> jvm heap)GC会移动对象内存,在写file或socket的过程中,JVM的实现中,会先把数据复制到堆外,再进行写入.
2. GC范围之外,降低GC压力,但实现了自动管理.DirectByteBuffer中有一个Cleaner对象(PhantomReference),Cleaner被GC前会执行clean方法,触发DirectByteBuffer中定义的Deallocator;

建议:

1. 性能确实可观才会使用;分配给大型,长寿命;(网络传输,文件读写场景);
2. 通过虚拟机参数MaxDirectMemorySize限制大小,防止耗尽整个机器的内存;

## 3.2.Channel通道

![image-20201207161641778](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20201207161641778.png)

### 3.2.1.SocketChannel

SocketChannel用于建立TCP网络连接,类似java.net.Socket.

有两种创建socketChannel形式:

1. 客户端主动发起和服务器的连接;
2. 服务端获取的新连接;

```java
//客户端主动发起连接的方式
SocketChannel socketChannel = SocketChannel.open();
socketChannel.configureBlocking(false);//设置为非阻塞模式;
socketChannel.connect(new InetSocketAddress("IP",port));

channel.write(byteBuffer);//发送请求数据-向通道写入数据;

int bytesRead = socketChannel.read(byteBuffer);//读取服务端返回-读取缓冲区的数据;

socketChannel.close();//关闭连接
```

注意:

* write写:`write()`在尚未写入任何内容时就可能返回了.需要在循环中调用`write()`;
* read读:`read()`方法可能直接返回而根本不读取任何数据,根据返回的int值判断读取了多少字节;

### 3.2.2. ServerSocketChannel

ServerSocketChannel可以监听新建的TCP连接通道,类似ServerSocket

```java
//创建网络服务端
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
serverSocketChannel.configureBlocking(false);//设置非阻塞模式;
serverSocketChannel.socket().bind(new InetSocketAddress(port));//绑定端口;
while(true){
	SocketChannel socketChannel = serverSocketChannel.accept();//获取新tcp连接通道;
  if(socketChannel!=null){
    //tcp请求 读取/响应;
  }
}
```

`sercerSocketChannel.accept()`:如果该通道处于非阻塞模式,那么如果没有挂起的连接,该方法将立即返回null.必须检查返回的socketChannel是否为null;

### 3.2.3.代码示例

#### 3.2.3.1.NIO Client

~~~java
public class NIOClient {

    public static void main(String[] args) throws Exception {
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.configureBlocking(false);
        socketChannel.connect(new InetSocketAddress("127.0.0.1", 8080));
        while (!socketChannel.finishConnect()) {
            // 没连接上,则一直等待
            Thread.yield();
        }
        Scanner scanner = new Scanner(System.in);
        System.out.println("请输入：");
        // 发送内容
        String msg = scanner.nextLine();
        ByteBuffer buffer = ByteBuffer.wrap(msg.getBytes());
        while (buffer.hasRemaining()) {
            socketChannel.write(buffer);
        }
        // 读取响应
        System.out.println("收到服务端响应:");
        ByteBuffer requestBuffer = ByteBuffer.allocate(1024);

        while (socketChannel.isOpen() && socketChannel.read(requestBuffer) != -1) {
            // 长连接情况下,需要手动判断数据有没有读取结束 (此处做一个简单的判断: 超过0字节就认为请求结束了)
            if (requestBuffer.position() > 0) break;
        }
        requestBuffer.flip();
        byte[] content = new byte[requestBuffer.limit()];
        requestBuffer.get(content);
        System.out.println(new String(content));
        scanner.close();
        socketChannel.close();
    }
}
~~~

#### 3.2.3.2.NIO Server V0版本

~~~java
public class NIOServer {

    public static void main(String[] args) throws Exception {
        // 创建网络服务端
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false); // 设置为非阻塞模式
        serverSocketChannel.socket().bind(new InetSocketAddress(8080)); // 绑定端口
        System.out.println("启动成功");
        while (true) {
            SocketChannel socketChannel = serverSocketChannel.accept(); // 获取新tcp连接通道
            // tcp请求 读取/响应
            if (socketChannel != null) {
                System.out.println("收到新连接 : " + socketChannel.getRemoteAddress());
                socketChannel.configureBlocking(false); // 默认是阻塞的,一定要设置为非阻塞
                try {
                    ByteBuffer requestBuffer = ByteBuffer.allocate(1024);
                    //判断是否能读到数据
                    while (socketChannel.isOpen() && socketChannel.read(requestBuffer) != -1) {
                        //长连接情况下,需要手动判断数据有没有读取结束 (此处做一个简单的判断: 超过0字节就认为请求结束了)
                        //如果读到数据,并且position>0执行下面的代码
                        if (requestBuffer.position() > 0) break;
                    }
                    // 如果没数据了, 则不继续后面的处理
                    if(requestBuffer.position() == 0) continue;
                    requestBuffer.flip();
                    byte[] content = new byte[requestBuffer.limit()];
                    requestBuffer.get(content);
                    System.out.println(new String(content));
                    System.out.println("收到数据,来自："+ socketChannel.getRemoteAddress());

                    // 响应结果 200
                    String response = "HTTP/1.1 200 OK\r\n" +
                            "Content-Length: 11\r\n\r\n" +
                            "Hello World";
                    ByteBuffer buffer = ByteBuffer.wrap(response.getBytes());
                    while (buffer.hasRemaining()) {
                        socketChannel.write(buffer);// 非阻塞
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        // 用到了非阻塞的API, 在设计上,和BIO可以有很大的不同.继续改进
    }
}
~~~

#### 3.2.3.3.NIO Server V1版本

~~~java
/**
 * 直接基于非阻塞的写法,一个线程处理轮询所有请求
 */
public class NIOServer1 {
    /**
     * 已经建立连接的集合
     */
    private static ArrayList<SocketChannel> channels = new ArrayList<>();

    public static void main(String[] args) throws Exception {
        // 创建网络服务端
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false); // 设置为非阻塞模式
        serverSocketChannel.socket().bind(new InetSocketAddress(8080)); // 绑定端口
        System.out.println("启动成功");
        while (true) {
            SocketChannel socketChannel = serverSocketChannel.accept(); // 获取新tcp连接通道
                // tcp请求 读取/响应
                if (socketChannel != null) {
                System.out.println("收到新连接 : " + socketChannel.getRemoteAddress());
                socketChannel.configureBlocking(false); // 默认是阻塞的,一定要设置为非阻塞
                channels.add(socketChannel);
            } else {
                // 没有新连接的情况下,就去处理现有连接的数据,处理完的就删除掉
                Iterator<SocketChannel> iterator = channels.iterator();
                while (iterator.hasNext()) {
                    SocketChannel ch = iterator.next();
                    try {
                        ByteBuffer requestBuffer = ByteBuffer.allocate(1024);

                        if (ch.read(requestBuffer) == 0) {
                            // 等于0,代表这个通道没有数据需要处理,那就待会再处理
                            continue;
                        }
                        while (ch.isOpen() && ch.read(requestBuffer) != -1) {
                            // 长连接情况下,需要手动判断数据有没有读取结束 (此处做一个简单的判断: 超过0字节就认为请求结束了)
                            if (requestBuffer.position() > 0) break;
                        }
                        if(requestBuffer.position() == 0) continue; // 如果没数据了, 则不继续后面的处理
                        requestBuffer.flip();
                        byte[] content = new byte[requestBuffer.limit()];
                        requestBuffer.get(content);
                        System.out.println(new String(content));
                        System.out.println("收到数据,来自：" + ch.getRemoteAddress());

                        // 响应结果 200
                        String response = "HTTP/1.1 200 OK\r\n" +
                                "Content-Length: 11\r\n\r\n" +
                                "Hello World";
                        ByteBuffer buffer = ByteBuffer.wrap(response.getBytes());
                        while (buffer.hasRemaining()) {
                            ch.write(buffer);
                        }
                        iterator.remove();
                    } catch (IOException e) {
                        e.printStackTrace();
                        iterator.remove();
                    }
                }
            }
        }
        // 用到了非阻塞的API, 再设计上,和BIO可以有很大的不同
        // 问题: 轮询通道的方式,低效,浪费CPU
    }
}
~~~

## 3.3.Selector选择器

Selector是一个Java NIO组件,可以检查一个或多个NIO通道,并确定哪些通道已准备好进行读取或写入.实现单个线程可以管理多个通道,从而管理多个网络连接.

一个线程使用Selector监听多个channel的不同事件,这四个事件分别对应SelectionKey四个常量:

1. Connect连接(SelectionKey.OP_CONNECT);
2. Accept准备就绪(OP_ACCEPT);
3. Read读取(OP_READ);
4. Write写入(OP_WRITE);

<img src="https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20201208094324579.png" alt="image-20201208094324579" style="zoom:33%;" />

Selector选择器实现一个线程处理多个通道的核心概念理解:事件驱动机制;

非阻塞的网络通道下,开发者通过Selector注册对于通道感兴趣的事件类型,线程通过监听事件来触发相应的代码执行(更底层是操作系统的多路复用机制);

```java
Selector selector = Selector.open();
channel.configureBlocking(false);
SelectionKey key = channel.register(selector,SelectionKey.OP_READ);//注册感兴趣的事件;
while(true){
  //有accept轮询,编程了事件通知的方式;
  int readyChannels = Selector.select();//select收到新的事件,方法才会返回;
  if(readyChannels==0) continue;
  Set<SelectionKey> selectedKeys = selector.selectedKeys();
  Iterator<SelectionKey> keyIterator = selectedKeys.iterator();
  while(keyIterator.hasNext()){
    SelectionKey key = keyIterator.next();
    //判断不同的事件类型,执行对应的逻辑处理;
    //key.isAcceptable()/key.isConnectable()/key.isReadable()/key.isWriteable();
  }
}
```

### 3.3.1.代码示例

#### 3.3.1.1.NIO Server V2版本

~~~java
/**
 * 结合Selector实现的非阻塞服务端(放弃对channel的轮询,借助消息通知机制)
 */
public class NIOServerV2 {

    public static void main(String[] args) throws Exception {
        // 1. 创建网络服务端ServerSocketChannel
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false); // 设置为非阻塞模式

        // 2. 构建一个Selector选择器,并且将channel注册上去
        Selector selector = Selector.open();
        SelectionKey selectionKey = serverSocketChannel.register(selector, 0, serverSocketChannel);// 将serverSocketChannel注册到selector
        selectionKey.interestOps(SelectionKey.OP_ACCEPT); // 对serverSocketChannel上面的accept事件感兴趣(serverSocketChannel只能支持accept操作)

        // 3. 绑定端口
        serverSocketChannel.socket().bind(new InetSocketAddress(8080));

        System.out.println("启动成功");

        while (true) {
            // 不再轮询通道,改用下面轮询事件的方式.select方法有阻塞效果,直到有事件通知才会有返回
            selector.select();
            // 获取事件
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            // 遍历查询结果e
            Iterator<SelectionKey> iter = selectionKeys.iterator();
            while (iter.hasNext()) {
                // 被封装的查询结果
                SelectionKey key = iter.next();
                iter.remove();
                // 关注 Read 和 Accept两个事件
                if (key.isAcceptable()) {
                    ServerSocketChannel server = (ServerSocketChannel) key.attachment();
                    // 将拿到的客户端连接通道,注册到selector上面
                    SocketChannel clientSocketChannel = server.accept(); // mainReactor 轮询accept
                    clientSocketChannel.configureBlocking(false);
                  	//READ事件进行注册
                    clientSocketChannel.register(selector, SelectionKey.OP_READ, clientSocketChannel);
                    System.out.println("收到新连接 : " + clientSocketChannel.getRemoteAddress());
                }

                if (key.isReadable()) {
                    SocketChannel socketChannel = (SocketChannel) key.attachment();
                    try {
                        ByteBuffer requestBuffer = ByteBuffer.allocate(1024);
                        while (socketChannel.isOpen() && socketChannel.read(requestBuffer) != -1) {
                            // 长连接情况下,需要手动判断数据有没有读取结束 (此处做一个简单的判断: 超过0字节就认为请求结束了)
                            if (requestBuffer.position() > 0) break;
                        }
                        if(requestBuffer.position() == 0) continue; // 如果没数据了, 则不继续后面的处理
                        requestBuffer.flip();
                        byte[] content = new byte[requestBuffer.limit()];
                        requestBuffer.get(content);
                        System.out.println(new String(content));
                        System.out.println("收到数据,来自：" + socketChannel.getRemoteAddress());
                        // TODO 业务操作 数据库 接口调用等等

                        // 响应结果 200
                        String response = "HTTP/1.1 200 OK\r\n" +
                                "Content-Length: 11\r\n\r\n" +
                                "Hello World";
                        ByteBuffer buffer = ByteBuffer.wrap(response.getBytes());
                        while (buffer.hasRemaining()) {
                            socketChannel.write(buffer);
                        }
                    } catch (IOException e) {
                        // e.printStackTrace();
                        key.cancel(); // 取消事件订阅
                    }
                }
            }
            selector.selectNow();
        }
        // 问题: 此处一个selector监听所有事件,一个线程处理所有请求事件. 会成为瓶颈! 要有多线程的运用
    }
}
~~~

#### 3.3.1.2.NIO Server V3版本(终极版-Reactor模型)

NIO与多线程结合的改进方案:

![image-20210623171810208](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210623171810208.png)

~~~java
/**
 * NIO selector 多路复用reactor线程模型
 */
public class NIOServerV3 {
    /** 处理业务操作的线程 */
    private static ExecutorService workPool = Executors.newCachedThreadPool();

    /**
     * 封装了selector.select()等事件轮询的代码
     */
    abstract class ReactorThread extends Thread {

        Selector selector;
        LinkedBlockingQueue<Runnable> taskQueue = new LinkedBlockingQueue<>();

        /**
         * Selector监听到有事件后,调用这个方法
         */
        public abstract void handler(SelectableChannel channel) throws Exception;

        private ReactorThread() throws IOException {
            selector = Selector.open();
        }

        volatile boolean running = false;

        @Override
        public void run() {
            // 轮询Selector事件
            while (running) {
                try {
                    // 执行队列中的任务
                    Runnable task;
                    while ((task = taskQueue.poll()) != null) {
                        task.run();
                    }
                    selector.select(1000);

                    // 获取查询结果
                    Set<SelectionKey> selected = selector.selectedKeys();
                    // 遍历查询结果
                    Iterator<SelectionKey> iter = selected.iterator();
                    while (iter.hasNext()) {
                        // 被封装的查询结果
                        SelectionKey key = iter.next();
                        iter.remove();
                        int readyOps = key.readyOps();
                        // 关注 Read 和 Accept两个事件
                        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                            try {
                                SelectableChannel channel = (SelectableChannel) key.attachment();
                                channel.configureBlocking(false);
                                handler(channel);
                                if (!channel.isOpen()) {
                                    key.cancel(); // 如果关闭了,就取消这个KEY的订阅
                                }
                            } catch (Exception ex) {
                                key.cancel(); // 如果有异常,就取消这个KEY的订阅
                            }
                        }
                    }
                    selector.selectNow();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }

        private SelectionKey register(SelectableChannel channel) throws Exception {
            // 为什么register要以任务提交的形式，让reactor线程去处理？
            // 因为线程在执行channel注册到selector的过程中，会和调用selector.select()方法的线程争用同一把锁
            // 而select()方法实在eventLoop中通过while循环调用的，争抢的可能性很高，为了让register能更快的执行，就放到同一个线程来处理
            FutureTask<SelectionKey> futureTask = new FutureTask<>(() -> channel.register(selector, 0, channel));
            taskQueue.add(futureTask);
            return futureTask.get();
        }

        private void doStart() {
            if (!running) {
                running = true;
                start();
            }
        }
    }

    private ServerSocketChannel serverSocketChannel;
    // 1、创建多个线程 - accept处理reactor线程 (accept线程)
    private ReactorThread[] mainReactorThreads = new ReactorThread[1];
    // 2、创建多个线程 - io处理reactor线程  (I/O线程)
    private ReactorThread[] subReactorThreads = new ReactorThread[8];

    /**
     * 初始化线程组
     */
    private void newGroup() throws IOException {
        // 创建IO线程,负责处理客户端连接以后socketChannel的IO读写
        for (int i = 0; i < subReactorThreads.length; i++) {
            subReactorThreads[i] = new ReactorThread() {
                @Override
                public void handler(SelectableChannel channel) throws IOException {
                    // work线程只负责处理IO处理，不处理accept事件
                    SocketChannel ch = (SocketChannel) channel;
                    ByteBuffer requestBuffer = ByteBuffer.allocate(1024);
                    while (ch.isOpen() && ch.read(requestBuffer) != -1) {
                        // 长连接情况下,需要手动判断数据有没有读取结束 (此处做一个简单的判断: 超过0字节就认为请求结束了)
                        if (requestBuffer.position() > 0) break;
                    }
                    if (requestBuffer.position() == 0) return; // 如果没数据了, 则不继续后面的处理
                    requestBuffer.flip();
                    byte[] content = new byte[requestBuffer.limit()];
                    requestBuffer.get(content);
                    System.out.println(new String(content));
                    System.out.println(Thread.currentThread().getName() + "收到数据,来自：" + ch.getRemoteAddress());

                    // TODO 业务操作 数据库、接口...
                    workPool.submit(() -> {
                    });

                    // 响应结果 200
                    String response = "HTTP/1.1 200 OK\r\n" +
                            "Content-Length: 11\r\n\r\n" +
                            "Hello World";
                    ByteBuffer buffer = ByteBuffer.wrap(response.getBytes());
                    while (buffer.hasRemaining()) {
                        ch.write(buffer);
                    }
                }
            };
        }

               // 创建mainReactor线程, 只负责处理serverSocketChannel
        for (int i = 0; i < mainReactorThreads.length; i++) {
            mainReactorThreads[i] = new ReactorThread() {
                AtomicInteger incr = new AtomicInteger(0);

                @Override
                public void handler(SelectableChannel channel) throws Exception {
                    // 只做请求分发，不做具体的数据读取
                    ServerSocketChannel ch = (ServerSocketChannel) channel;
                    SocketChannel socketChannel = ch.accept();
                    socketChannel.configureBlocking(false);
                    // 收到连接建立的通知之后，分发给I/O线程继续去读取数据
                    int index = incr.getAndIncrement() % subReactorThreads.length;
                    ReactorThread workEventLoop = subReactorThreads[index];
                    workEventLoop.doStart();
                    SelectionKey selectionKey = workEventLoop.register(socketChannel);
                    selectionKey.interestOps(SelectionKey.OP_READ);
                    System.out.println(Thread.currentThread().getName() + "收到新连接 : " + socketChannel.getRemoteAddress());
                }
            };
        }


    }

    /**
     * 初始化channel,并且绑定一个eventLoop线程
     *
     * @throws IOException IO异常
     */
    private void initAndRegister() throws Exception {
        // 1、 创建ServerSocketChannel
        serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false);
        // 2、 将serverSocketChannel注册到selector
        int index = new Random().nextInt(mainReactorThreads.length);
        mainReactorThreads[index].doStart();
        SelectionKey selectionKey = mainReactorThreads[index].register(serverSocketChannel);
        selectionKey.interestOps(SelectionKey.OP_ACCEPT);
    }

    /**
     * 绑定端口
     *
     * @throws IOException IO异常
     */
    private void bind() throws IOException {
        //  1、 正式绑定端口，对外服务
        serverSocketChannel.bind(new InetSocketAddress(8080));
        System.out.println("启动完成，端口8080");
    }

    public static void main(String[] args) throws Exception {
        NIOServerV3 nioServerV3 = new NIOServerV3();
        nioServerV3.newGroup(); // 1、 创建main和sub两组线程
        nioServerV3.initAndRegister(); // 2、 创建serverSocketChannel，注册到mainReactor线程上的selector上
        nioServerV3.bind(); // 3、 为serverSocketChannel绑定端口
    }
}

~~~

## 3.4.NIO对比BIO

![image-20201208152809713](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20201208152809713.png)

### 