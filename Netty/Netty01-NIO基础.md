# Netty01-NIO基础-三大组件

# 1.NIO三大组件

## 1.1.Channel

channel 有一点类似于 stream，它就是读写数据的**双向通道**，可以从 channel 将数据读入 buffer，也可以将 buffer 的数据写入 channel，而之前的 stream 要么是输入，要么是输出，channel 比 stream 更为底层;

常见的 Channel 有

* FileChannel
* DatagramChannel
* SocketChannel
* ServerSocketChannel

## 1.2.Buffer

Buffer用来缓冲读写数据,常见的Buffer有:

* ByteBuffer
  * MappedByteBuffer
  * DirectByteBuffer
  * HeapByteBuffer
* ShortBuffer
* IntBuffer
* LongBuffer
* FloatBuffer
* DoubleBuffer
* CharBuffer

## 1.3.Selector

Selector需要结合服务器的设计演化来理解它的用途

### 1.3.1.多线程版设计

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211118214047210.png" alt="image-20211118214047210" style="zoom:50%;" />

多线程的缺点:

* 内存占用高;
* 线程上下文切换成本高;
* 只适合连接数少的场景;

### 1.3.2.线程池版设计

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211118214251983.png" alt="image-20211118214251983" style="zoom:50%;" />

线程池版缺点:

* 阻塞模式下,线程仅能处理一个socket连接;
* 仅适合短连接场景;

### 1.3.3.selector版设计

* selector的作用就是配合一个线程来管理多个channel,获取这些channel上发生的事件,这些channel工作在非阻塞模式下,不会让线程吊死在一个channel上.适合连接数特别多,但流量低的场景;
* 调用selector的`select()`会阻塞直到channel发生了读写就绪事件,这些事件发生,`select()`就会返回这些事件交给thread来进行处理;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211118214812573.png" alt="image-20211118214812573" style="zoom:50%;" />

# 2.ByteBuffer

## 2.1.ByteBuffer的基本使用

### 2.1.1.向Buffer写数据

* 调用 channel 的 read 方法`int readBytes = channel.read(buf);`
* 调用 buffer 自己的 put 方法`buf.put((byte)127);`

### 2.1.2.向Buffer读数据

* 调用 channel 的 write 方法`int writeBytes = channel.write(buf);`
* 调用 buffer 自己的 get 方法`byte b = buf.get();`

get 方法会让 position 读指针向后走，如果想重复读取数据

* 可以调用 `rewind()` 方法将 position 重新置为 0
* 或者调用` get(int i) `方法获取索引 i 的内容，它不会移动读指针

### 2.1.3.mark和reset

* mark 是在读取时，做一个标记，即使 position 改变，只要调用 `reset()` 就能回到 mark 的位置;
* rewind 和 flip 都会清除 mark 位置;

### 2.1.4.ByteBuffer的标准使用

~~~java
@Slf4j
public class TestByteBuffer {

    public static void main(String[] args) {
        // FileChannel
        // FileChannel的获取方式: 1. 输入输出流， 2. RandomAccessFile
        // data的内容:1234567890abcdefg;
        try (FileChannel channel = new FileInputStream("data.txt").getChannel()) {
            // 准备缓冲区
            ByteBuffer buffer = ByteBuffer.allocate(10);
            while(true) {
                // 从 channel 读取数据，向 buffer 写入
                int len = channel.read(buffer);
                log.debug("读取到的字节数 {}", len);
                if(len == -1) { // 没有内容了
                    break;
                }
                // 打印 buffer 的内容
                buffer.flip(); // 切换至读模式
                while(buffer.hasRemaining()) { // 是否还有剩余未读数据
                    byte b = buffer.get();
                    log.debug("实际字节 {}", (char) b);
                }
                buffer.clear(); // 切换为写模式
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
~~~

打印结果:

```shell
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 读取到的字节数 10
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 1
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 2
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 3
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 4
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 5
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 6
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 7
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 8
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 9
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 0
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 读取到的字节数 7
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 a
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 b
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 c
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 d
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 e
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 f
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 实际字节 g
17:41:34 [DEBUG] [main] c.i.n.c.TestByteBuffer - 读取到的字节数 -1
```

* ByteBuffer正确使用姿势:
  * 向buffer写入数据,例如调用`channel.read(buffer)`;
  * 调用`flip()`切换至读模式;
  * 从buffer中读取数据,例如调用`buffer.get()`;
  * 调用`clear()`或`compact()`切换至写模式;
  * 重复1-4步骤;

### 2.1.5.ByteBuffer与String的转化

```java
public static void main(String[] args) {
        // 1. 字符串转为 ByteBuffer
        ByteBuffer buffer1 = ByteBuffer.allocate(16);
        buffer1.put("hello".getBytes());
        debugAll(buffer1);

        // 2. Charset
        ByteBuffer buffer2 = StandardCharsets.UTF_8.encode("hello");
        debugAll(buffer2);

        // 3. wrap
        ByteBuffer buffer3 = ByteBuffer.wrap("hello".getBytes());
        debugAll(buffer3);

        // 4. 转为字符串
        String str1 = StandardCharsets.UTF_8.decode(buffer2).toString();
        System.out.println(str1);

        buffer1.flip();
        String str2 = StandardCharsets.UTF_8.decode(buffer1).toString();
        System.out.println(str2);

    }
```

## 2.2.ByteBuffer结构

ByteBuffer有以下重要属性:

* capacity;
* position;
* limit;

`ByteBuffer.allocate(xxx)`:ByteBuffer是分配在Java堆内存.读写效率较低,受到GC的影响;

`ByteBuffer.allocateDirect(xxx):`分配在直接内存,读写效率高(少一次拷贝),不会收到GC影响,分配的效率低;

初始状态:

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211120181338592.png" alt="image-20211120181338592" style="zoom: 67%;" />

写模式下,position是写入位置,limit等于容量,下图表示写入了4个字节后的状态:

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211120181759545.png" alt="image-20211120181759545" style="zoom:67%;" />

flip动作发生后,position切换为读取位置,limit切换为读取限制;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211120181835181.png" alt="image-20211120181835181" style="zoom: 67%;" />

读取4个字节后,状态;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211120182048598.png" style="zoom:67%;" />

clear动作:不管有没有读完,清空并转化为写模式,发生后,状态; 

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211120182333965.png" alt="image-20211120182333965" style="zoom:67%;" />

compact方法是把未读完的部分向前压缩,然后切换至写模式

![image-20211120182459466](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211120182459466.png)

## 2.3.代码解决粘包和半包

~~~java
public class TestByteBufferExam {
    public static void main(String[] args) {
         /*
         网络上有多条数据发送给服务端，数据之间使用 \n 进行分隔
         但由于某种原因这些数据在接收时，被进行了重新组合，例如原始数据有3条为
             Hello,world\n
             I'm zhangsan\n
             How are you?\n
         变成了下面的两个 byteBuffer (黏包，半包)
             Hello,world\nI'm zhangsan\nHo
             w are you?\n
         现在要求你编写程序，将错乱的数据恢复成原始的按 \n 分隔的数据
         */
        ByteBuffer source = ByteBuffer.allocate(32);
        source.put("Hello,world\nI'm zhangsan\nHo".getBytes());
        split(source);
        source.put("w are you?\n".getBytes());
        split(source);
    }

    private static void split(ByteBuffer source) {
        source.flip();
        for (int i = 0; i < source.limit(); i++) {
            // 找到一条完整消息
            if (source.get(i) == '\n') {
                int length = i + 1 - source.position();
                // 把这条完整消息存入新的 ByteBuffer
                ByteBuffer target = ByteBuffer.allocate(length);
                // 从 source 读，向 target 写
                for (int j = 0; j < length; j++) {
                    target.put(source.get());
                }
                debugAll(target);
            }
        }
        source.compact();
    }
}
~~~

# 3.Channel

## 3.1.阻塞的Channel

* 阻塞模式下，相关方法都会导致线程暂停
  * ServerSocketChannel.accept 会在没有连接建立时让线程暂停
  * SocketChannel.read 会在没有数据可读时让线程暂停
  * 阻塞的表现其实就是线程暂停了，暂停期间不会占用 cpu，但线程相当于闲置
* 单线程下，阻塞方法之间相互影响，几乎不能正常工作，需要多线程支持
* 但多线程下，有新的问题，体现在以下方面
  * 32 位 jvm 一个线程 320k，64 位 jvm 一个线程 1024k，如果连接数过多，必然导致 OOM，并且线程太多，反而会因为频繁上下文切换导致性能降低
  * 可以采用线程池技术来减少线程数和线程上下文切换，但治标不治本，如果有很多连接建立，但长时间 inactive，会阻塞线程池中所有线程，因此不适合长连接，只适合短连接

服务器端

```java
// 使用 nio 来理解阻塞模式, 单线程
// 0. ByteBuffer
ByteBuffer buffer = ByteBuffer.allocate(16);
// 1. 创建了服务器
ServerSocketChannel ssc = ServerSocketChannel.open();

// 2. 绑定监听端口
ssc.bind(new InetSocketAddress(8080));

// 3. 连接集合
List<SocketChannel> channels = new ArrayList<>();
while (true) {
    // 4. accept 建立与客户端连接， SocketChannel 用来与客户端之间通信
    log.debug("connecting...");
    SocketChannel sc = ssc.accept(); // 阻塞方法，线程停止运行
    log.debug("connected... {}", sc);
    channels.add(sc);
    for (SocketChannel channel : channels) {
        // 5. 接收客户端发送的数据
        log.debug("before read... {}", channel);
        channel.read(buffer); // 阻塞方法，线程停止运行
        buffer.flip();
        debugRead(buffer);
        buffer.clear();
        log.debug("after read...{}", channel);
    }
}
```

客户端

```java
SocketChannel sc = SocketChannel.open();
sc.connect(new InetSocketAddress("localhost", 8080));
System.out.println("waiting...");
```

## 3.2.非阻塞的Channel

服务器端，客户端代码不变

```java
// 使用 nio 来理解非阻塞模式, 单线程
// 0. ByteBuffer
ByteBuffer buffer = ByteBuffer.allocate(16);
// 1. 创建了服务器
ServerSocketChannel ssc = ServerSocketChannel.open();
ssc.configureBlocking(false); // 非阻塞模式
// 2. 绑定监听端口
ssc.bind(new InetSocketAddress(8080));
// 3. 连接集合
List<SocketChannel> channels = new ArrayList<>();
while (true) {
    // 4. accept 建立与客户端连接， SocketChannel 用来与客户端之间通信
    SocketChannel sc = ssc.accept(); // 非阻塞，线程还会继续运行，如果没有连接建立，但sc是null
    if (sc != null) {
        log.debug("connected... {}", sc);
        sc.configureBlocking(false); // 非阻塞模式
        channels.add(sc);
    }
    for (SocketChannel channel : channels) {
        // 5. 接收客户端发送的数据
        int read = channel.read(buffer);// 非阻塞，线程仍然会继续运行，如果没有读到数据，read 返回 0
        if (read > 0) {
            buffer.flip();
            debugRead(buffer);
            buffer.clear();
            log.debug("after read...{}", channel);
        }
    }
}
```

# 4.Selector

## 4.1.多路复用

单线程可以配合 Selector 完成对多个 Channel 可读写事件的监控，这称之为多路复用

* 多路复用仅针对网络 IO、普通文件 IO 没法利用多路复用
* 如果不用 Selector 的非阻塞模式，线程大部分时间都在做无用功，而 Selector 能够保证
  * 有可连接事件时才去连接
  * 有可读事件才去读取
  * 有可写事件才去写入
    * 限于网络传输能力，Channel 未必时时可写，一旦 Channel 可写，会触发 Selector 的可写事件

## 4.2.Selector的使用步骤

 ### 4.2.1.创建

```java
Selector selector = Selector.open();
```

### 4.2.2.绑定Channel事件

也称之为注册事件，绑定的事件 selector 才会关心 

```java
channel.configureBlocking(false);
SelectionKey key = channel.register(selector, 绑定事件);
```

* channel 必须工作在非阻塞模式
* FileChannel 没有非阻塞模式，因此不能配合 selector 一起使用
* 绑定的事件类型可以有
  * connect - 客户端连接成功时触发
  * accept - 服务器端成功接受连接时触发
  * read - 数据可读入时触发，有因为接收能力弱，数据暂不能读入的情况
  * write - 数据可写出时触发，有因为发送能力弱，数据暂不能写出的情况

### 4.2.3.监听Channel事件

可以通过下面三种方法来监听是否有事件发生，方法的返回值代表有多少 channel 发生了事件

* 方法1，阻塞直到绑定事件发生

```java
int count = selector.select();
```

* 方法2，阻塞直到绑定事件发生，或是超时（时间单位为 ms）

```java
int count = selector.select(long timeout);
```

* 方法3，不会阻塞，也就是不管有没有事件，立刻返回，自己根据返回值检查是否有事件

```java
int count = selector.selectNow();
```

#### 4.2.3.1.select何时不阻塞

* 事件发生时
  * 客户端发起连接请求，会触发 accept 事件;
  * 客户端发送数据过来，客户端正常、异常关闭时，都会触发 read 事件，另外如果发送的数据大于 buffer 缓冲区，会触发多次读取事件;
  * channel 可写，会触发 write 事件;
  * 在 linux 下 nio bug 发生时;
* 调用 `selector.wakeup()`;
* 调用 `selector.close()`;
* selector 所在线程 interrupt;

### 4.2.4.处理accept事件

客户端代码为

```java
public class Client {
    public static void main(String[] args) {
        try (Socket socket = new Socket("localhost", 8080)) {
            System.out.println(socket);
            socket.getOutputStream().write("world".getBytes());
            System.in.read();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

服务器端代码为

```java
@Slf4j
public class ChannelDemo6 {
    public static void main(String[] args) {
        try (ServerSocketChannel channel = ServerSocketChannel.open()) {
            channel.bind(new InetSocketAddress(8080));
            System.out.println(channel);
            Selector selector = Selector.open();
            channel.configureBlocking(false);
            channel.register(selector, SelectionKey.OP_ACCEPT);

            while (true) {
                int count = selector.select();
//                int count = selector.selectNow();
                log.debug("select count: {}", count);
//                if(count <= 0) {
//                    continue;
//                }

                // 获取所有事件
                Set<SelectionKey> keys = selector.selectedKeys();

                // 遍历所有事件，逐一处理
                Iterator<SelectionKey> iter = keys.iterator();
                while (iter.hasNext()) {
                    SelectionKey key = iter.next();
                    // 判断事件类型
                    if (key.isAcceptable()) {
                        ServerSocketChannel c = (ServerSocketChannel) key.channel();
                        // 必须处理
                        SocketChannel sc = c.accept();
                        log.debug("{}", sc);
                    }
                    // 处理完毕，必须将事件移除
                    iter.remove();
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

**事件发生后,要么处理,要么取消(cancel),不能什么都不做,否则下次该事件仍会触发,这是因为nio底层使用的是水平触发**;

### 4.2.5.处理read事件

```java
@Slf4j
public class Server {

    public static void main(String[] args) throws IOException {
        // 1. 创建 selector, 管理多个 channel
        Selector selector = Selector.open();
        ServerSocketChannel ssc = ServerSocketChannel.open();
        ssc.configureBlocking(false);
        // 2. 建立 selector 和 channel 的联系（注册）
        // SelectionKey 就是将来事件发生后，通过它可以知道事件和哪个channel的事件
        SelectionKey sscKey = ssc.register(selector, 0, null);
        // key 只关注 accept 事件
        sscKey.interestOps(SelectionKey.OP_ACCEPT);
        log.debug("sscKey:{}", sscKey);
        ssc.bind(new InetSocketAddress(8080));
        while (true) {
            // 3. select 方法, 没有事件发生，线程阻塞，有事件，线程才会恢复运行
            // select 在事件未处理时，它不会阻塞, 事件发生后要么处理，要么取消，不能置之不理
            selector.select();
            // 4. 处理事件, selectedKeys 内部包含了所有发生的事件
            Iterator<SelectionKey> iter = selector.selectedKeys().iterator(); // accept, read
            while (iter.hasNext()) {
                SelectionKey key = iter.next();
                // 处理key 时，要从 selectedKeys 集合中删除，否则下次处理就会有问题
                iter.remove();
                log.debug("key: {}", key);
                // 5. 区分事件类型
                if (key.isAcceptable()) { // 如果是 accept
                    ServerSocketChannel channel = (ServerSocketChannel) key.channel();
                    SocketChannel sc = channel.accept();
                    sc.configureBlocking(false);

                    SelectionKey scKey = sc.register(selector, 0, null);
                    scKey.interestOps(SelectionKey.OP_READ);
                    log.debug("{}", sc);
                    log.debug("scKey:{}", scKey);
                } else if (key.isReadable()) { // 如果是 read
                    try {
                        SocketChannel channel = (SocketChannel) key.channel(); // 拿到触发事件的channel
                        ByteBuffer buffer = ByteBuffer.allocate(4);
                        int read = channel.read(buffer); // 如果是客户端正常断开，read 的方法的返回值是 -1
                        if(read == -1) {
                            key.cancel();
                        } else {
                            buffer.flip();
//                            debugAll(buffer);
                            System.out.println(Charset.defaultCharset().decode(buffer));
                        }
                    } catch (IOException e) {
                        e.printStackTrace();
                        key.cancel();  // 因为客户端断开了,因此需要将 key 取消（从 selector 的 keys 集合中真正删除 key）
                    }
                }
            }
        }
    }
}
```

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211121110317475.png" alt="image-20211121110317475" style="zoom:50%;" />

* `iter.remove()`;
  * 因为 select 在事件发生后，就会将相关的 key 放入 selectedKeys 集合，但不会在处理完后从 selectedKeys 集合中移除，需要我们自己编码删除。例如
    * 第一次触发了 ssckey 上的 accept 事件，没有移除 ssckey 
    * 第二次触发了 sckey 上的 read 事件，但这时 selectedKeys 中还有上次的 ssckey ，在处理时因为没有真正的 serverSocket 连上了，就会导致空指针异常;
* `cancel()`会取消注册在selector上的channel,并从keys集合中删除key后续不会再监听事件;

### 4.2.6.处理客户端断开

客户端的断开分为正常退出断开,还要强制断开;

分两种情况进行处理:

1. 客户端强制断开:会报错IOException,捕捉错误,并将key进行`cancel()`;
2. 客户端正常断开:触发read事件,buffer读取为-1,如果读取到-1,将key进行`cancel()`;

![image-20211121135947123](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211121135947123.png)

### 4.2.7.处理消息的边界

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211121112557091.png" alt="image-20211121112557091" style="zoom:50%;" />

* 一种思路是固定消息长度，数据包大小一样，服务器按预定长度读取，缺点是浪费带宽
* 另一种思路是按分隔符拆分，缺点是效率低
* TLV 格式，即 Type 类型、Length 长度、Value 数据，类型和长度已知的情况下，就可以方便获取消息大小，分配合适的 buffer，缺点是 buffer 需要提前分配，如果内容过大，则影响 server 吞吐量
  * Http 1.1 是 TLV 格式
  * Http 2.0 是 LTV 格式

#### 4.2.7.1.代码处理消息的边界

思路如下:

1. 将ByteBuffer与channel进行绑定,一个channel有一个ByteBuffer保证线程安全;
2. 当ByteBuffer大小不够的时候进行扩容;

![image-20211121135415544](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211121135415544.png)

服务器:

```java
private static void split(ByteBuffer source) {
    source.flip();
    for (int i = 0; i < source.limit(); i++) {
        // 找到一条完整消息
        if (source.get(i) == '\n') {
            int length = i + 1 - source.position();
            // 把这条完整消息存入新的 ByteBuffer
            ByteBuffer target = ByteBuffer.allocate(length);
            // 从 source 读，向 target 写
            for (int j = 0; j < length; j++) {
                target.put(source.get());
            }
            debugAll(target);
        }
    }
    source.compact(); // 0123456789abcdef  position 16 limit 16
}

public static void main(String[] args) throws IOException {
    // 1. 创建 selector, 管理多个 channel
    Selector selector = Selector.open();
    ServerSocketChannel ssc = ServerSocketChannel.open();
    ssc.configureBlocking(false);
    // 2. 建立 selector 和 channel 的联系（注册）
    // SelectionKey 就是将来事件发生后，通过它可以知道事件和哪个channel的事件
    SelectionKey sscKey = ssc.register(selector, 0, null);
    // key 只关注 accept 事件
    sscKey.interestOps(SelectionKey.OP_ACCEPT);
    log.debug("sscKey:{}", sscKey);
    ssc.bind(new InetSocketAddress(8080));
    while (true) {
        // 3. select 方法, 没有事件发生，线程阻塞，有事件，线程才会恢复运行
        // select 在事件未处理时，它不会阻塞, 事件发生后要么处理，要么取消，不能置之不理
        selector.select();
        // 4. 处理事件, selectedKeys 内部包含了所有发生的事件
        Iterator<SelectionKey> iter = selector.selectedKeys().iterator(); // accept, read
        while (iter.hasNext()) {
            SelectionKey key = iter.next();
            // 处理key 时，要从 selectedKeys 集合中删除，否则下次处理就会有问题
            iter.remove();
            log.debug("key: {}", key);
            // 5. 区分事件类型
            if (key.isAcceptable()) { // 如果是 accept
                ServerSocketChannel channel = (ServerSocketChannel) key.channel();
                SocketChannel sc = channel.accept();
                sc.configureBlocking(false);
                ByteBuffer buffer = ByteBuffer.allocate(16); // attachment
                // 将一个 byteBuffer 作为附件关联到 selectionKey 上****************
                SelectionKey scKey = sc.register(selector, 0, buffer);
                scKey.interestOps(SelectionKey.OP_READ);
                log.debug("{}", sc);
                log.debug("scKey:{}", scKey);
            } else if (key.isReadable()) { // 如果是 read
                try {
                    SocketChannel channel = (SocketChannel) key.channel(); // 拿到触发事件的channel
                    // 获取 selectionKey 上关联的附件
                    ByteBuffer buffer = (ByteBuffer) key.attachment();
                    int read = channel.read(buffer); // 如果是正常断开，read 的方法的返回值是 -1
                    if(read == -1) {
                        key.cancel();
                    } else {
                        split(buffer);
                        // 需要扩容******************
                        if (buffer.position() == buffer.limit()) {
                            ByteBuffer newBuffer = ByteBuffer.allocate(buffer.capacity() * 2);
                            buffer.flip();
                            newBuffer.put(buffer); // 0123456789abcdef3333\n
                            key.attach(newBuffer);//重新绑定新的ByteBuffer
                        }
                    }

                } catch (IOException e) {
                    e.printStackTrace();
                    key.cancel();  // 因为客户端断开了,因此需要将 key 取消（从 selector 的 keys 集合中真正删除 key）
                }
            }
        }
    }
}
```

客户端:

```java
SocketChannel sc = SocketChannel.open();
sc.connect(new InetSocketAddress("localhost", 8080));
SocketAddress address = sc.getLocalAddress();
// sc.write(Charset.defaultCharset().encode("hello\nworld\n"));
sc.write(Charset.defaultCharset().encode("0123\n456789abcdef"));
sc.write(Charset.defaultCharset().encode("0123456789abcdef3333\n"));
System.in.read();
```

### 4.2.8.处理write事件

服务端:

```java
public class WriteServer {
    public static void main(String[] args) throws IOException {
        ServerSocketChannel ssc = ServerSocketChannel.open();
        ssc.configureBlocking(false);
        Selector selector = Selector.open();
        ssc.register(selector, SelectionKey.OP_ACCEPT);
        ssc.bind(new InetSocketAddress(8080));
        while (true) {
            selector.select();
            Iterator<SelectionKey> iter = selector.selectedKeys().iterator();
            while (iter.hasNext()) {
                SelectionKey key = iter.next();
                iter.remove();
                if (key.isAcceptable()) {
                    SocketChannel sc = ssc.accept();
                    sc.configureBlocking(false);
                    SelectionKey sckey = sc.register(selector, 0, null);
                    sckey.interestOps(SelectionKey.OP_READ);
                    // 1. 向客户端发送大量数据
                    StringBuilder sb = new StringBuilder();
                    for (int i = 0; i < 5000000; i++) {
                        sb.append("a");
                    }
                    ByteBuffer buffer = Charset.defaultCharset().encode(sb.toString());

                    // 2. 返回值代表实际写入的字节数
                    int write = sc.write(buffer);
                    System.out.println(write);

                    // 3. 判断是否有剩余内容
                    if (buffer.hasRemaining()) {
                        // 4. 关注可写事件   1                     4
                        sckey.interestOps(sckey.interestOps() + SelectionKey.OP_WRITE);
//                        sckey.interestOps(sckey.interestOps() | SelectionKey.OP_WRITE);
                        // 5. 把未写完的数据挂到 sckey 上
                        sckey.attach(buffer);
                    }
                } else if (key.isWritable()) {
                    ByteBuffer buffer = (ByteBuffer) key.attachment();
                    SocketChannel sc = (SocketChannel) key.channel();
                    int write = sc.write(buffer);
                    System.out.println(write);
                    // 6. 清理操作
                    if (!buffer.hasRemaining()) {
                        key.attach(null); // 需要清除buffer
                        key.interestOps(key.interestOps() - SelectionKey.OP_WRITE);//不需关注可写事件
                    }
                }
            }
        }
    }
}
```

客户端:

```java
public class WriteClient {
    public static void main(String[] args) throws IOException {
        SocketChannel sc = SocketChannel.open();
        sc.connect(new InetSocketAddress("localhost", 8080));

        // 3. 接收数据
        int count = 0;
        while (true) {
            ByteBuffer buffer = ByteBuffer.allocate(1024 * 1024);
            count += sc.read(buffer);
            System.out.println(count);
            buffer.clear();
        }
    }
}
```
