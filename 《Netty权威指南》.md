# 第1章 Java的I/O演进之路

## 1.1 I/O基础入门

### 1.1.1 Linux网络I/O模型简介

UNIX提供了5种I/O模型：

（1）阻塞I/O模型

（2）非阻塞I/O模型

（3）I/O复用模型

（4）信号驱动I/O模型

（5）异步I/O

### 1.1.2 I/O多路复用技术

把多个I/O阻塞复用到同一个select的阻塞上，从而是的系统在单线程的情况下可以同时处理多个客户端请求。比多线程有性能优势，节约资源。

支持I/O多路复用的系统调用select/pselect/poll/epoll。

**epoll的优点：**

1. 支持一个进程打开的socket描述符（FD）不受限制（仅受限于操作系统的最大文件句柄数（内存））。
2. I/O效率不会随着FD数目的增加而线性下降。
3. 使用mmap加速内核与用户空间的消息传递。
4. epoll的API更加简单。

## 1.2 Java的I/O演进

历史题，略。

# 第2章 NIO入门

## 2.1 传统的BIO编程

C/S模型，客户端发起连接请求，三次握手，通过Socket进行通信。

### 2.1.1 BIO通信模型图

一请求一应答模型：每次接收到连接请求都创建一个新的线程进行链路处理。处理完成后通过输出流返回应答给客户端，线程销毁。

该模型最大的问题就是：缺乏弹性伸缩能力，当客户端并发访问量增加后，服务端的线程个数和客户端并发访问数呈1：1的正比关系。

### 2.1.2 同步阻塞式I/O创建的TimeServer源码分析

```Java
import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;

/**
 * <p>Description:  同步阻塞式I/O创建的TimeServer</p>
 *
 * @author 李宏博
 * @version xxx
 * @create 2019/8/14 17:58
 */


public class TimeServer {
    /**
     *
     * @param args
     */
    public static void main(String[] args) throws IOException {
        int port = 8080;
        if (args != null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                e.printStackTrace();
            }
        }

        ServerSocket server = null;
        try {
            server = new ServerSocket(port);
            System.out.println("服务端端口开启：" + port);
            Socket socket = null;
            while (true) {
                socket = server.accept();
                new Thread(new TimeServerHandle(socket)).start();
            }
        } finally {
            if (server != null) {
                System.out.println("服务端关闭");
                server.close();
                server = null;
            }
        }

    }

}

```

```Java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.Socket;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version xxx
 * @create 2019/8/14 18:05
 */


public class TimeServerHandle implements Runnable {

    private Socket socket;

    public TimeServerHandle(Socket socket) {
        this.socket = socket;
    }


    @Override
    public void run() {
        BufferedReader in = null;
        PrintWriter out = null;
        try {
            in = new BufferedReader(new InputStreamReader(this.socket.getInputStream()));
            out = new PrintWriter(this.socket.getOutputStream(),true);
            String currentTime = null;
            String body = null;
            while (true) {
                body = in.readLine();
                if (body == null) {
                    break;
                }
                System.out.println("接收到命令：" + body);
                currentTime = "time".equalsIgnoreCase(body) ? new java.util.Date(System.currentTimeMillis()).toString() : "命令错误";
                out.println(currentTime);
            }
        } catch (Exception e) {
            if (in != null) {
                try {
                    in.close();
                } catch (IOException ex) {
                    ex.printStackTrace();
                }
            }

            if (out != null) {
                out.close();
                out = null;
            }

            if (this.socket != null) {
                try {
                    this.socket.close();
                } catch (IOException ex) {
                    ex.printStackTrace();
                } finally {
                    this.socket = null;
                }
            }
        }
    }
}

```

### 2.1.3 同步阻塞式I/O创建的TimeClient源码分析

```Java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.Socket;
import java.net.UnknownHostException;

/**
 * <p>Description:  同步阻塞式I/O创建的TimeClient</p>
 *
 * @author 李宏博
 * @version xxx
 * @create 2019/8/14 18:21
 */


public class TimeClient {
    /**
     *
     * @param args
     */
    public static void main(String[] args) {
        int port = 8080;
        if (args != null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                e.printStackTrace();
            }
        }
        Socket socket = null;
        BufferedReader in = null;
        PrintWriter out = null;
        try {
            socket = new Socket("127.0.0.1",port);
            in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            out = new PrintWriter(socket.getOutputStream(),true);
            out.println("time");
            System.out.println("命令发送成功");
            String resp = in.readLine();
            System.out.println("现在时间：" + resp);
        } catch (UnknownHostException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (out != null) {
                out.close();
                out = null;
            }

            if (in != null) {
                try {
                    in.close();
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    in = null;
                }
            }

            if (socket != null) {
                try {
                    socket.close();
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    socket = null;
                }
            }
        }
    }
}

```

## 2.2 伪异步I/O编程

引入线程池。

### 2.2.1 伪异步I/O模型图

由于线程池可以设置消息队列的大小和最大线程数，因此，它的资源占用式可控的，无论多少个客户端并发访问，都不会导致资源的耗尽和宕机。

### 2.2.2 伪异步I/O创建的TimeServer源码分析

```Java
import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version xxx
 * @create 2019/8/14 18:37
 */


public class TimeServer2 {
    /**
     *
     * @param args
     */
    public static void main(String[] args) throws IOException {
        int port = 8080;
        if (args != null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                e.printStackTrace();
            }
        }

        ServerSocket server = null;
        try {
            server = new ServerSocket(port);
            System.out.println("服务端端口开启：" + port);
            Socket socket = null;
            TimeServerHandlerExecutePool singleExecutePool = new TimeServerHandlerExecutePool(50,10000);
            while (true) {
                socket = server.accept();
                singleExecutePool.execute(new TimeServerHandle(socket));
            }
        } finally {
            if (server != null) {
                System.out.println("服务端关闭");
                server.close();
                server = null;
            }
        }

    }

}

```

```Java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * <p>Description:  线程池</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/14 18:40
 */


public class TimeServerHandlerExecutePool {

    private ExecutorService executor;

    public TimeServerHandlerExecutePool(int maxPoolSize,int queueSize) {
        executor = new ThreadPoolExecutor(Runtime.getRuntime().availableProcessors(),maxPoolSize,120L,
                TimeUnit.SECONDS,new ArrayBlockingQueue<Runnable>(queueSize));
    }

    public void execute(Runnable task) {
        executor.execute(task);
    }
}

```

由于底层的通信依然采用同步阻塞模型。因此无法从根本上解决问题。

### 2.2.3 伪异步I/O弊端分析

操作Socket的输入流时，会一直阻塞，知道出现以下情况：

- 有数据可读；
- 可用数据已经读取完毕；
- 发生空指针或者I/O异常。

输出流也会阻塞，TCP发送窗口跟接收窗口大小相关，流量控制。如果采用同步I/O会，wirte操作会被无限期阻塞。

阻塞的时间取决于对方I/O线程的处理速度和网络I/O的传输速度。所以不能依赖对方的处理速度进行I/O。

通信对方返回应答时间过程会引起的级联故障：

（1）服务端处理缓慢，返回应答消息耗费60s，平时只需要10ms；

（2）采用伪异步I/O的线程正在读取故障服务节点的响应，由于读取输入流时阻塞的，它将会被同步阻塞60s；

（3）假如所有可用线程都被故障服务器阻塞，那后续所有的I/O消息都将在队列中排队。

（4）由于线程池采用阻塞队列实现，当队列积满之后，后续入队列的操作将被阻塞。

（5）由于前端只有一个Accptor线程接收客户端接入，它被阻塞在线程池的同步阻塞队列之后，新的客户端请求消息将被拒绝，客户端会发生大量的连接超时。

（6）由于所有的连接都超时，调用者会认为系统已经崩溃，无法接收新的请求消息。

## 2.3 NIO编程

本书使用的NIO都指的是非阻塞I/O。

一般来说，低负载、低并发的应用程序可以选择同步I/O以降低编程复杂度；对于高负载、高并发的网络应用，需要使用NIO的非阻塞模式进行开发。

### 2.3.1 NIO类库简介

1. **缓冲区 Buffer**

Buffer是一个对象，包含一些要写入或者要读出的数据。在面向流的I/O种，可以将数据直接写入或者将数据直接读到Stream对象中。

所有读写操作都通过缓冲区进行。

实际上是一个数组。通常为ByteBuffer。

2. **通道 Channel**

双向。流Stream是单向的。通道可以用于读、写或者二者同时进行。

3. **多路复用器 Selector**

Selector会不断地轮询注册在其上的Channel，如果某个Channel上面发生读或者写事件，这个Channe了就处于就绪状态，会被Selector轮询出来，然后通过SelectionKey可以获取就绪Channel的集合，进行后续的I/O操作。

### 2.3.2 NIO服务端序列图

步骤一：打开ServerSocketChannel，用于监听客户端的连接，它是所有客户端连接的父管道；

```Java
ServerSocketChannel acceptorSvr = ServerSocketChannel.open();
```

步骤二：绑定监听端口，设置连接为非阻塞模式；

```Java
acceptorSvr.socket().bind(new InetSocketAddress(InetAddress.getByName("IP"),port));
acceptorSvr.configureBlocking(false);
```

步骤三：创建Reactor线程，创建多路复用器并启动线程；

```Java
Selector selector = Selector.open();
New Thread(new ReactorTask()).start();
```

步骤四：将ServerSocketChannel注册到Reactor线程的多路复用器Selector上，监听ACCEPT事件；

```Java
SelectionKey key = acceptorSvr.register(selector, SelectionKey.OP_ACCEPT, ioHandler);
```

步骤五：多路复用器在线程run方法的无限循环体内轮询准备就绪的Key；

```Java
int num = Selector.select();
Set selectedKeys = selector.selectedKeys();
Iterator it = selectedKeys.iterator();
while (it.hasNext()) {
    SelectionKey key = (SelectionKey) it.next();
}
```

步骤六：多路复用监听器听到有新的客户端接入，处理新的接入请求，完成TCP三次握手，建立物理链路；

```Java
SocketChannel channel = svrChannel.accept();
```

步骤七：设置客户端链路为非阻塞模式；

```Java
channel.configureBlocking(false);
channel.socket().setReuseAddress(true);
```

步骤八：将新接入的客户端连接注册到Reactor线程的多路复用器上，监听读操作，读取客户端发送的网络消息，示例代码如下；

```java
SelectionKey key = socketChannel.register(selector, SelectionKey.OP_READ,ioHandler);
```

步骤九：异步读取客户端请求消息到缓冲区；

```Java
int readNumber = channel.read(receivedBuffer);
```

步骤十：对ByteBuffer进行编解码，如果有半包消息指针reset，继续读取后续的报文，将节码成功的消息封装成Task，投递到业务线程池中，进行业务逻辑编排；

```Java
Object message = null;
while(buffer.hasRemain()) {
    byteBuffer.mark();
    Object message = decode(byteBuffer);
    if(message == null) {
        byteBuffer.reset();
        break;
    }
    messageList.add(message);
}
if(!byteBuffer.hasRemain()) {
    byteBuffer.clear();
} else {
    byteBuffer.compact();
} ..........................
```

步骤十一：将POJO对象encode成Bytebuffer，调用SocketChannel的异步write接口，将消息异步发送到客户端；

```Java
socketChannel.write(buffer);
```

### 2.3.3 NIO创建的TimeServer源码分析

```Java
/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/15 11:40
 */


public class TimeServer3 {
    public static void main(String[] args) {
        int port = 8080;
        if (args != null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                e.printStackTrace();
            }
        }

        MultiplexerTimeServer timeServer = new MultiplexerTimeServer(port);

        new Thread(timeServer, "NIO-MultiplexerTimeServer-001").start();
    }
}
```

```Java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/15 11:42
 */


public class MultiplexerTimeServer implements Runnable{

    private Selector selector;

    private ServerSocketChannel serverSocketChannel;

    private volatile boolean stop;

    public MultiplexerTimeServer(int port) {
        try {
            selector = Selector.open();
            serverSocketChannel = ServerSocketChannel.open();
            serverSocketChannel.configureBlocking(false);
            serverSocketChannel.socket().bind(new InetSocketAddress(port),1024);
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
            System.out.println("服务器启动，端口号："+port);
        } catch (IOException e) {
            e.printStackTrace();
            System.exit(1);
        }
    }

    public void stop() {
        this.stop = true;
    }

    public void run() {
        while (!stop) {
            try {
                selector.select(1000);
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> it = selectionKeys.iterator();
                SelectionKey key = null;
                while (it.hasNext()) {
                    key = it.next();
                    it.remove();
                    try {
                        handleInput(key);
                    } catch (Exception e) {
                        if (key != null) {
                            key.cancel();
                            if (key.channel() != null) {
                                key.channel().close();
                            }
                        }
                    }

                }
            } catch (Throwable t) {
                t.printStackTrace();
            }
        }

        if (selector != null) {
            try {
                selector.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private void handleInput(SelectionKey key) throws IOException {
        if (key.isValid()) {
            if (key.isAcceptable()) {
                ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
                SocketChannel sc = ssc.accept();
                sc.configureBlocking(false);
                sc.register(selector,SelectionKey.OP_READ);
            }

            if (key.isReadable()) {
                SocketChannel sc = (SocketChannel) key.channel();
                ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                int readBytes = sc.read(readBuffer);//非阻塞
                if (readBytes > 0) {
                    readBuffer.flip();
                    byte[] bytes = new byte[readBuffer.remaining()];
                    readBuffer.get(bytes);
                    String body = new String(bytes,"UTF-8");
                    System.out.println("服务器接收到命令："+body);
                    String currentTime = "time".equalsIgnoreCase(body) ? new java.util.Date(System.currentTimeMillis()).toString() : "命令错误";
                    doWrite(sc,currentTime);
                } else if (readBytes < 0) {//返回值等于-1，链路已经关闭
                    key.cancel();
                    sc.close();
                } else {//返回值等于0
                }
            }
        }
    }

    private void doWrite(SocketChannel channel, String response) throws IOException {
        if (response != null && response.trim().length() > 0) {//由于SocketChannel是异步非阻塞的，会出现“写半包”问题
            byte[] bytes = response.getBytes();
            ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
            writeBuffer.put(bytes);
            writeBuffer.flip();
            channel.write(writeBuffer);
        }
    }

}
```

### 2.3.4 NIO客户端序列图

步骤一：打开SocketChannel，绑定客户端本地地址（可选，默认随机分配）；

```Java
SocketChannel clientChannel = SocketChannel.open();
```

步骤二：设置SocketChannel为非阻塞模式，同时设置客户端连接的TCP参数；

```java
clientChannel.configureBlocking(false);
socket.setReuseAddress(true);
socket.setReceiveBufferSize(BUFFER_SIZE);
socket.setSendBufferSize(BUFFER_SIZE);
```

步骤三：异步连接服务端；

```java
boolean connected = clientChannel.connect(new InetSocketAddress("ip",port));
```

步骤四：判断是否连接成功，如果连接成功，则直接注册读状态位到多路复用器中，如果当前没有连接成功（异步连接，返回false，说明客户端已经发送sync包，服务端没有返回ack包，物理链路还没有建立）；

```java
if(connectied) {
    clientChannel.register(selector,Selection.OP_READ,ioHandler);
} else {
    clientChannel.register(selector,Selection.OP_CONNECT,ioHandler);
}
```

步骤五：向Reactor线程的多路复用器注册OP_CONNECT状态位，监听服务端的TCPACK应答；

```JAVA
clientChannel.register(selector, Selection.OP_CONNECT, ioHandler);
```

步骤六：创建Reactor线程，创建多路复用器并启动线程；

```JAVA
Selector seletor = Selector.open();
new Thread(new ReactorTask()).start();
```

步骤七：多路复用器在线程run方法的无限循环体内轮询准备就绪的Key；

```JAVA
int num = selector.select();
Set selectedKeys = selector,selectedKeys();
Irerator it = selectedKeys.iterator();
while (it.hasNext()) {
    SelectionKey key = (SelectionKey) it.next();
}
```

步骤八：接收connect事件进行处理；

```Java
if(key.isConnectable()) {
    ..
}
```

步骤九：判断连接结果，如果连接成功，注册读事件到多路复用器；

```java
if(channel.finishConnection()) {
    registerRead();
}
```

步骤十：注册读事件到多路复用器；

```Java
clientChannel.register(selector, SelectionKey.OP_READ, ioHandler);
```

步骤十一：异步读客户端请求消息到缓冲区；

```Java
int readNumber = channel.read(receiveBuffer);
```

步骤十二：对ByteBuffer进行编解码，如果有半包消息接收缓冲区Reset，继续读取后续的报文，将解码成功的消息封装成Task，投递到业务线程池中，进行业务逻辑编排。

略

步骤十三：将POKO对象encode成ByteBuffer，调用SocketChannel的异步write接口，将消息异步发送给客户端。

```Java
socketChannel.write(buffer);
```

### 2.3.5 NIO创建的TimeClient源码分析

```java
package NIO;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/15 14:13
 */


public class TimeClient {
    public static void main(String[] args) {
        int port = 8080;
        if (args != null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                e.printStackTrace();
            }
        }

        new Thread(new TimeClientHandle("127.0.0.1",port),"TimeClient-001").start();
    }
}
```

```java
package NIO;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/15 14:16
 */


public class TimeClientHandle implements Runnable {

    private String host;
    private int port;
    private Selector selector;
    private SocketChannel socketChannel;
    private volatile boolean stop;

    public TimeClientHandle(String host, int port) {
        this.host = host == null ? "127.0.0.1" : host;
        this.port = port;
        try {
            selector = Selector.open();
            socketChannel = SocketChannel.open();
            socketChannel.configureBlocking(false);
        } catch (IOException e) {
            e.printStackTrace();
            System.exit(1);
        }
    }


    @Override
    public void run() {
        try {
            doConnect();
        } catch (IOException e) {
            e.printStackTrace();
            System.exit(1);
        }

        while (!stop) {
            try {
                selector.select(1000);
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> it = selectionKeys.iterator();
                SelectionKey key = null;
                while (it.hasNext()) {
                    key = it.next();
                    it.remove();
                    try {
                        handleInput(key);
                    } catch (IOException e) {
                        if (key != null) {
                            key.cancel();
                            if (key.channel() != null) {
                                key.channel().close();
                            }
                        }
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
                System.exit(1);
            }
        }

        if (selector != null) {
            try {
                selector.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }



    private void handleInput(SelectionKey key) throws IOException {
        if (key.isValid()) {
            SocketChannel sc = (SocketChannel) key.channel();
            if (key.isConnectable()) {
                if (sc.finishConnect()) {
                    sc.register(selector, SelectionKey.OP_READ);
                    doWrite(sc);
                } else {
                    System.exit(1);
                }
            }

            if (key.isReadable()) {
                ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                int readBytes = sc.read(readBuffer);
                if (readBytes > 0) {
                    readBuffer.flip();
                    byte[] bytes = new byte[readBuffer.remaining()];
                    readBuffer.get(bytes);
                    String body = new String(bytes,"UTF-8");
                    System.out.println("现在时间：" + body);
                    this.stop = true;
                } else if (readBytes < 0) {
                    key.cancel();
                    sc.close();
                } else {}
            }
        }
    }

    private void doConnect() throws IOException {
        if (socketChannel.connect(new InetSocketAddress(host,port))) {
            socketChannel.register(selector, SelectionKey.OP_READ);
            doWrite(socketChannel);
        } else {
            socketChannel.register(selector,SelectionKey.OP_CONNECT);
        }
    }

    private void doWrite(SocketChannel sc) throws IOException {
        byte[] req = "Time".getBytes();
        ByteBuffer writeBuffer = ByteBuffer.allocate(req.length);
        writeBuffer.put(req);
        writeBuffer.flip();
        sc.write(writeBuffer);
        if (!writeBuffer.hasRemaining()) {
            System.out.println("命令发送成功");
        }
    }

}
```

NIO编程的难度大，而且还没有考虑”半包读“和”半包写“，加上会更复杂。

NIO的优点：

（1）客户端发起的连接操作是异步的，可以通过在多路复用器注册OP_CONNECT等待后续结果，不需要像之前的客户端那样被同步阻塞。

（2）SocketChannel的读写操作都是异步的，如果没有可读写的数据它不会同步等待，直接返回，这样I/O通信线程就可以处理其他的链路，不需要同步等待这个链路可用。

（3）线程模型的优化：在linux上的底层是epoll。性能不随客户端的增加先行下降。

因此适合做高性能，高负载的网络服务器。

## 2.4 AIO编程

异步通道通过以下两种方式获取操作结果：

- java.util.concurrent.Future类表示异步操作的结果。
- 在执行异步操作的时候传入一个java.nio.channels。

CompletionHandle接口的实现类作为操作完成的回调。

不需要多路复用器（Selector），简化NIO的编程模型。

### 2.4.1 AIO创建的TimeServer源码分析

```java
package AIO;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/16 9:38
 */


public class TimeServer {

    public static void main(String[] args) {
        int port = 8080;
        if (args != null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                e.printStackTrace();
            }
        }

        AsyncTimeServerHandler timeserver = new AsyncTimeServerHandler(port);

        new Thread(timeserver,"AIO-" +
                "AsyncTimeServerHandler-001").start();
    }

}
```

```Java
package AIO;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.util.concurrent.CountDownLatch;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/16 9:41
 */


public class AsyncTimeServerHandler implements Runnable{

    private int port;

    CountDownLatch latch;
    AsynchronousServerSocketChannel asynchronousServerSocketChannel;


    public AsyncTimeServerHandler(int port) {
        this.port = port;
        try {
            asynchronousServerSocketChannel = AsynchronousServerSocketChannel.open();
            asynchronousServerSocketChannel.bind(new InetSocketAddress(port));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {

        latch = new CountDownLatch(1);
        doAccept();
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }

    private void doAccept() {
        asynchronousServerSocketChannel.accept(this, new AcceptCompletionHandler());
    }
}

```

```Java
package AIO;

import java.nio.ByteBuffer;
import java.nio.channels.CompletionHandler;
import java.nio.channels.AsynchronousSocketChannel;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/16 9:54
 */


public class AcceptCompletionHandler implements CompletionHandler<AsynchronousSocketChannel,AsyncTimeServerHandler> {
    @Override
    public void completed(AsynchronousSocketChannel result, AsyncTimeServerHandler attachment) {
        attachment.asynchronousServerSocketChannel.accept(attachment,this);
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        result.read(buffer, buffer, new ReadCompletionHandler(result));

    }

    @Override
    public void failed(Throwable exc, AsyncTimeServerHandler attachment) {
        exc.printStackTrace();
        attachment.latch.countDown();
    }

}

```

```java
package AIO;

import java.io.IOException;
import java.io.UnsupportedEncodingException;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/16 10:14
 */


public class ReadCompletionHandler implements CompletionHandler<Integer, ByteBuffer> {

    private AsynchronousSocketChannel channel;

    public ReadCompletionHandler(AsynchronousSocketChannel channel) {
        if (this.channel == null) {
            this.channel = channel;
        }
    }

    @Override
    public void completed(Integer result, ByteBuffer attachment) {

        attachment.flip();
        byte[] body = new byte[attachment.remaining()];
        attachment.get(body);
        try {
            String req = new String(body, "UTF-8");
            System.out.println("服务器接收到命令：" + req);
            String currentTime = "time".equalsIgnoreCase(req) ? new java.util.Date(System.currentTimeMillis()).toString() : "命令错误";
            doWrite(currentTime);
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }

    private void doWrite(String currentTime) {
        if (currentTime != null && currentTime.trim().length() > 0) {
            byte[] bytes = (currentTime).getBytes();
            ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
            writeBuffer.put(bytes);
            writeBuffer.flip();
            channel.write(writeBuffer, writeBuffer, new CompletionHandler<Integer, ByteBuffer>() {
                @Override
                public void completed(Integer result, ByteBuffer buffer) {
                    if (buffer.hasRemaining())
                        channel.write(buffer, buffer, this);
                }

                @Override
                public void failed(Throwable exc, ByteBuffer buffer) {
                    try {
                        channel.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    }

    @Override
    public void failed(Throwable exc, ByteBuffer attachment) {
        try {
            this.channel.close();
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
}

```

### 2.4.2 AIO创建的TimeClient源码分析

```Java
package AIO;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/16 10:32
 */


public class TimeClient {

    public static void main(String[] args) {
        int port = 8080;
        if (args != null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                e.printStackTrace();
            }
        }

        new Thread(new AsyncTimeClientHandler("127.0.0.1",port),"AIO-AsyncTimeClient-001").start();
    }
}

```

```Java
package AIO;

import java.io.IOException;
import java.io.UnsupportedEncodingException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.util.concurrent.CountDownLatch;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/16 10:35
 */


public class AsyncTimeClientHandler implements CompletionHandler<Void,AsyncTimeClientHandler>,Runnable {

    private AsynchronousSocketChannel client;
    private String host;
    private int port;
    private CountDownLatch latch;

    public AsyncTimeClientHandler(String host, int port) {

        this.host = host;
        this.port = port;
        try {
            client = AsynchronousSocketChannel.open();
        } catch (IOException e) {
            e.printStackTrace();
        }

    }

    @Override
    public void run() {

        latch = new CountDownLatch(1);
        client.connect(new InetSocketAddress(host, port), this, this);
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        try {
            client.close();
        } catch (IOException e) {
            e.printStackTrace();
        }

    }

    @Override
    public void completed(Void result, AsyncTimeClientHandler attachment) {

        byte[] req = "time".getBytes();
        ByteBuffer writeBuffer = ByteBuffer.allocate(req.length);
        writeBuffer.put(req);
        writeBuffer.flip();
        client.write(writeBuffer, writeBuffer, new CompletionHandler<Integer, ByteBuffer>() {
            @Override
            public void completed(Integer result, ByteBuffer buffer) {
                if (buffer.hasRemaining()) {
                    client.write(buffer, buffer, this);
                } else {
                    ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                    client.read(readBuffer, readBuffer, new CompletionHandler<Integer, ByteBuffer>() {
                        @Override
                        public void completed(Integer result, ByteBuffer buffer) {
                            buffer.flip();
                            byte[] bytes = new byte[buffer.remaining()];
                            buffer.get(bytes);
                            String body;
                            try {
                                body = new String(bytes,"UTF-8");
                                System.out.println("现在时间：" + body);
                                latch.countDown();
                            } catch (UnsupportedEncodingException e) {
                                e.printStackTrace();
                            }
                        }

                        @Override
                        public void failed(Throwable exc, ByteBuffer buffer) {
                            try {
                                client.close();
                            } catch (IOException e) {
                                e.printStackTrace();
                            }
                        }
                    });
                }
            }

            @Override
            public void failed(Throwable exc, ByteBuffer buffer) {
                try {
                    client.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        });

    }

    @Override
    public void failed(Throwable exc, AsyncTimeClientHandler attachment) {
        exc.printStackTrace();

        try {
            client.close();
            latch.countDown();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

```

## 2.5 4种I/O的对比

### 2.5.1 概念澄清

1. 异步非阻塞I/O

JDK1.4开始提供的NIO严格按照网络编程模型和JDK实现来分类，实际上只能被称为非阻塞I/O不能叫异步非阻塞I/O。1.5用epoll替换了select/poll，只是底层性能的优化，并没有替换I/O模型。

JDK1.7提供的NIO2.0是真正的异步I/O。

但习惯上NIO相对于IO还是被称为异步非阻塞IO或非阻塞I/O。

2. 多路复用器Selector

3. 伪异步I/O

完全来源于实践，加入一个缓冲区（线程池）。不是官方概念。

### 2.5.2 不同I/O模型的对比

|                     | 同步阻塞I/O（BIO） | 伪异步I/O             | 非阻塞I/O（NIO）                     | 异步I/O（AIO）                            |
| ------------------- | ------------------ | --------------------- | ------------------------------------ | ----------------------------------------- |
| 客户端个数：I/O线程 | 1：1               | M:N（其中M可以大于N） | M：1（1个I/O线程除了多个客户端连接） | M：0（不需要启动额外的I/O线程，被动回调） |
| I/O类型（阻塞）     | 阻塞I/O            | 阻塞I/O               | 非阻塞I/O                            | 非阻塞I/O                                 |
| I/O类型（同步）     | 同步I/O            | 同步I/O               | 同步I/O（I/O多路复用）               | 异步I/O                                   |
| API使用难度         | 简单               | 简单                  | 非常复杂                             | 复杂                                      |
| 可靠性              | 非常差             | 差                    | 高                                   | 高                                        |
| 吞吐量              | 低                 | 中                    | 高                                   | 高                                        |

## 2.6 选择Netty的理由

### 2.6.1 不选择Java原生NIO的原因

（1）NIO的类库和API繁杂，使用麻烦，需要熟练掌握Selector、ServerSocketChannel、SocketChannel、ByteBuffer等；

（2）涉及到Reactor模式，必须非常熟悉多线程和网络编程；

（3）可靠性补齐的工作量和难度大；

（4）BUG，epoll bug。

### 2.6.2 为什么选择Netty

优点如下：

- API使用简单，开发门槛低。
- 功能强大，预置了多种编解码功能，支持多种主流协议。
- 。。。

## 2.7 总结

# 第3章 Netty入门应用

### 3.1 Netty开发环境的搭建

注意Netty版本号为 5.0.0.Alpha1。

# 3.2 Netty 服务端开发

NIO开发步骤回顾：

（1）创建ServerSocketChannel，并配置为非阻塞；

（2）绑定监听，配置TCP参数，例如backlog大小；

（3）创建一个独立的I/O线程，用于轮询多路复用器Selector；

（4）创建Selector，将之前的ServerSocketChannel注册在Selector上，监听SelectionKey.ACCEPT；

（5）启动I/O线程，在循环体中执行Selector.select()方法，轮询就绪的Channel；

（6）轮循处于就绪状态的Channel，对其判断，如果是OP_ACCEPT，说明是新的客户端接入，则调用ServerSocketChannel.accept()方法接受新的客户端；

（7）设置新接入的客户端链路SocketChannel为非阻塞模式，配置其他的一些TCP参数；

（8）将SocketChannel注册到Selector，监听OP_READ操作位；

（9）监听到后读取；

（10）如果是OP_WRITE，则继续发送。

```Java
package Time;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/16 15:05
 */


public class TimeServer {

    public void bind(int port) throws Exception {

        //创建两个线程组的原因：一个用于服务端接受客户端的连接，另一个用于进行SocketChannel的网络读写
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            //Netty用于启动NIO服务端的辅助启动类，目的是降低服务端的开发复杂度
            ServerBootstrap b = new ServerBootstrap();

            b.group(bossGroup,workerGroup)//传入线程组
                    .channel(NioServerSocketChannel.class)//设置Channel，对应NIO中的ServerSocketChannel
                    .option(ChannelOption.SO_BACKLOG, 1024)//配置TCP参数，设置backlog为1024
                    .childHandler(new ChildChannelHandler());

            ChannelFuture f = b.bind(port).sync();//绑定并阻塞，完成后取消阻塞，返回一个ChannelFuture，类似于JDK的Future
            f.channel().closeFuture().sync();//阻塞，等待服务端链路关闭之后才退出main函数
        } finally {
            //优雅退出
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }


    private class ChildChannelHandler extends ChannelInitializer<SocketChannel> {

        @Override
        protected void initChannel(SocketChannel arg0) throws Exception {
            arg0.pipeline().addLast(new TimeServerHandle());
        }
    }

    public static void main(String[] args) throws Exception {
        int port = 8080;
        if (args != null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                e.printStackTrace();
            }
        }

        new TimeServer().bind(port);
    }
}

```

```java
package Time;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;

import java.io.UnsupportedEncodingException;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/16 15:28
 */


public class TimeServerHandler extends ChannelHandlerAdapter {

    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

        ByteBuf buf = (ByteBuf) msg;//类似ByteBuffer，更强大灵活
        byte[] req = new byte[buf.readableBytes()];
        buf.readBytes(req);
        String body = new String(req, "UTF-8");
        String currentTime = "time".equalsIgnoreCase(body) ? new java.util.Date(System.currentTimeMillis()).toString() : "命令错误";
        ByteBuf resp = Unpooled.copiedBuffer(currentTime.getBytes());
        ctx.write(resp);
    }

    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.flush();//从性能角度考虑，为避免频繁唤醒Selector进行消息发送，Netty不直接写，而是先放到缓冲数组，再调用flush写入。
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();//发生异常时释放。
    }

}

```

## 3.3 Netty客户端开发

```java
package Time;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/16 15:47
 */


public class TimeClient {

    public void connect(int port, String host) throws Exception {

        EventLoopGroup group = new NioEventLoopGroup();

        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
                    .channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY,true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new TimeClientHandler());//将ClientHandler设置到ChannelPipeline中，用于处理网络I/O事件
                        }
                    });

            ChannelFuture f = b.connect(host, port).sync();//connect异步连接，sync同步等待连接

            f.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception {

        int port = 8080;
        if (args != null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                e.printStackTrace();
            }
        }

        new TimeClient().connect(port, "127.0.0.1");
    }
}
```

```java
package Time;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;
import io.netty.util.concurrent.EventExecutorGroup;

import java.io.UnsupportedEncodingException;
import java.util.logging.Logger;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/16 15:59
 */


public class TimeClientHandler extends ChannelHandlerAdapter {

    private static final Logger logger = Logger.getLogger(TimeClientHandler.class.getName());

    private final ByteBuf firstMessage;

    public TimeClientHandler() {
        byte[] req = "time".getBytes();
        firstMessage = Unpooled.buffer(req.length);
        firstMessage.writeBytes(req);
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        ctx.writeAndFlush(firstMessage);
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf buf = (ByteBuf) msg;
        byte[] req = new byte[buf.readableBytes()];
        buf.readBytes(req);
        String body = new String(req, "UTF-8");
        System.out.println("现在时间：" + body);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        logger.warning("出错：" + cause.getMessage());
        ctx.close();
    }

}
```

## 3.4 运行和调试

略

## 3.5 总结

# 第4章 TCP粘包/拆包问题的解决之道

## 4.1 TCP粘包/拆包

TCP不了解上层业务数据的具体含义，它会根据TCP缓冲区的实际情况进行包的划分。

### 4.1.1 TCP粘包/拆包问题说明

4种情况（假设客户端发包）：

（1）服务端收到两个独立数据包，无粘包/拆包；

（2）服务端收到两个粘合在一起的包，粘包；

（3）服务端分两次读取到一个包，拆包；

（4）同3；

（5）接收窗口过小，出现多次拆包；

### 4.1.2 TCP粘包/拆包发生的原因

3个原因：

（1）应用程序write写入的字节大小大于套接口发送大小；

（2）进行MSS（最大报文长度）大小的TCP分段；

（3）以太网帧的payload大于MTU进行IP分片；

### 4.1.3 粘包问题的解决策略

由于底层的TCP不能理解业务数据，只能通过上层的应用协议栈设计来解决，4个策略：

（1）消息定长（例如固定每个报文段的大小为固定长度，不够则补空格）；

（2）在包尾增加回车换行符进行分割，例如FTP协议；

（3）消息头消息体，消息头中设立一个总长度字段；

（4）更复杂的应用层协议。

## 4.2 未考虑TCP粘包导致功能异常案例

### 4.2.1 TimeServer的改造

```java
package _4_;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/16 15:28
 */


public class TimeServerHandler extends ChannelHandlerAdapter {

    private int counter;

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

        ByteBuf buf = (ByteBuf) msg;//类似ByteBuffer，更强大灵活
        byte[] req = new byte[buf.readableBytes()];
        buf.readBytes(req);
        String body = new String(req, "UTF-8").substring(0, req.length - System.getProperty("line.separator").length());
        System.out.println("服务端接收到命令：" + body + "; 计数：" + ++counter);
        String currentTime = "time".equalsIgnoreCase(body) ? new java.util.Date(System.currentTimeMillis()).toString() : "命令错误";
        ByteBuf resp = Unpooled.copiedBuffer(currentTime.getBytes());
        ctx.writeAndFlush(resp);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();//发生异常时释放。
    }

}
```

### 4.2.2 TimeClient的改造
```java
package _4_;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;

import java.util.logging.Logger;


/**
 *
 */
public class TimeClientHandler extends ChannelHandlerAdapter {

    private static final Logger logger = Logger.getLogger(TimeClientHandler.class.getName());

    private int counter;

    private byte[] req;

    public TimeClientHandler() {
        req = ("time" + System.getProperty("line.separator")).getBytes();
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ByteBuf message = null;
        for (int i = 0; i < 100; i++) {//建立连接后循环发送100条消息，保证每条都被写入到Channel
            message = Unpooled.buffer(req.length);
            message.writeBytes(req);
            ctx.writeAndFlush(message);
        }
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf buf = (ByteBuf) msg;
        byte[] req = new byte[buf.readableBytes()];
        buf.readBytes(req);
        String body = new String(req, "UTF-8");
        System.out.println("现在时间：" + body + "; 计数：" + ++counter);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        logger.warning("出错：" + cause.getMessage());
        ctx.close();
    }
}
```

### 4.2.3 运行结果

出现TCP粘包

## 4.3 利用LineBasedFrameDecoder解决TCP粘包问题

代码不贴了，就是将自己在ByteBuf中的操作省去，然后加入两个解码器LineBasedFrameDecoder和StringDecoder。

### 4.3.3 运行TCP粘包的时间服务器程序

运行成功，符合预期。成功解决了TCP粘包导致的读半包问题。

### 4.3.4 LineBasedFrameDecoder和StringDecoder的原理分析

LineBasedFrameDecoder的工作原理：依次遍历，有“\n”或者“\r\n”则结束。以换行符为结束标志的解码器，支持携带结束符或者不携带结束符两种解码方式，同时支持配置单行的最大长度。如果连续读取到最大长度后仍然没有发现换行符，就会抛出异常，同时忽略掉之前督导的异常码流。

StringDecoder的功能非常简单，就是将接收到的对象转换成字符串，然后继续调用后面的Handler。

LineBasedFrameDecoder+StringDecoder组合就是按行切换的文本解码器，被设计用来支持TCP的粘包和拆包。

## 4.4 总结

# 第5章 分隔符和定长解码器的应用

区分消息的四种方式：

（1）消息定长

（2）回车换行符作为消息结束符

（3）特殊符号作为结束标志，回车为一种特殊实现

（4）通常在消息头中定义长度字段来标识消息的总长度

## 5.1 DelimiterBasedFrameDecoder应用开发

以分隔符作为码流结束标识的消息的解码。以“$_”作为分隔符。

### 5.1.3 DelimiterBasedFrameDecoder服务端开发

```java
package _5_1_;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.DelimiterBasedFrameDecoder;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.logging.LogLevel;
import io.netty.handler.logging.LoggingHandler;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/19 11:03
 */


public class EchoServer {
    public void bind(int port) throws Exception {

        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup,workGroup)
                    .channel(NioServerSocketChannel.class)
                    .handler(new LoggingHandler(LogLevel.INFO))
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ByteBuf delimiter = Unpooled.copiedBuffer("$_".getBytes());//创建分隔符缓冲对象，以$_为分隔符。
                            ch.pipeline().addLast(new DelimiterBasedFrameDecoder(1024,delimiter));//设置单条消息最大长度，加入分隔符对象
                            ch.pipeline().addLast(new StringDecoder());
                            ch.pipeline().addLast(new EchoServerHandler());
                        }
                    });

            ChannelFuture f = b.bind(port).sync();

            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception {
        int port = 8080;
        if (args != null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                e.printStackTrace();
            }
        }

        new EchoServer().bind(port);
    }
}

```

```java
package _5_1_;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelHandlerInvoker;
import io.netty.util.concurrent.EventExecutorGroup;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/19 11:12
 */


public class EchoServerHandler extends ChannelHandlerAdapter {

    int counter = 0;

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

        String body = (String) msg;
        System.out.println("第" + ++counter +"次接收来自客户端的消息：["+body+"]");//打印接收到的消息
        body += "$_";
        ByteBuf echo = Unpooled.copiedBuffer(body.getBytes());
        ctx.writeAndFlush(echo);

    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

```

### 5.1.2 DelimiterBasedFrameDecoder客户端开发

```java
package _5_1_;

import io.netty.bootstrap.Bootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.DelimiterBasedFrameDecoder;
import io.netty.handler.codec.string.StringDecoder;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/19 11:18
 */


public class EchoClient {

    public void connect(int port, String host) throws Exception {

        EventLoopGroup group = new NioEventLoopGroup();

        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
                    .channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY,true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ByteBuf delimiter = Unpooled.copiedBuffer("$_".getBytes());
                            ch.pipeline().addLast(new DelimiterBasedFrameDecoder(1024,delimiter));
                            ch.pipeline().addLast(new StringDecoder());
                            ch.pipeline().addLast(new EchoClientHandler());
                        }
                    });

            ChannelFuture f = b.connect(host, port).sync();

            f.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception {
        int port = 8080;
        if (args != null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                e.printStackTrace();
            }
        }

        new EchoClient().connect(port, "127.0.0.1");
    }
}

```

```java
package _5_1_;

import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/19 11:27
 */


public class EchoClientHandler extends ChannelHandlerAdapter {

    private int counter;

    static final String ECHO_REQ = "hello,world.$_";

    public EchoClientHandler() {
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        for (int i = 0; i < 10; i++) {
            ctx.writeAndFlush(Unpooled.copiedBuffer(ECHO_REQ.getBytes()));
        }
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("第" + ++counter +"次接收来自服务端的返回：["+msg+"]");
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }

}

```

## 5.2 FixedLengthFrameDecoder应用开发

FixedLengthFrameDecoder固定长度解码器。

### 5.2.1 FixedLengthFrameDecoder服务端开发

利用FixedLengthFrameDecoder解码器，无论一次接收到多少数据报，它都会按照构造函数中设置的固定长度进行解码，如果是半包消息，FixedLengthFrameDecoder会缓存半包消息并等待下个包到达后进行拼包，知道读取到一个完整的包。

```java
package _5_2_;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.DelimiterBasedFrameDecoder;
import io.netty.handler.codec.FixedLengthFrameDecoder;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.logging.LogLevel;
import io.netty.handler.logging.LoggingHandler;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/19 11:03
 */


public class EchoServer {
    public void bind(int port) throws Exception {

        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup,workGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 100)//加入一个属性
                    .handler(new LoggingHandler(LogLevel.INFO))
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new FixedLengthFrameDecoder(20));//设置单条消息最大长度，加入分隔符对象
                            ch.pipeline().addLast(new StringDecoder());
                            ch.pipeline().addLast(new EchoServerHandler());
                        }
                    });

            ChannelFuture f = b.bind(port).sync();

            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception {
        int port = 8080;
        if (args != null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                e.printStackTrace();
            }
        }

        new EchoServer().bind(port);
    }
}

```

```java
package _5_2_;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/19 11:12
 */


public class EchoServerHandler extends ChannelHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

        System.out.println("接收到来自客户端的消息："+"["+msg+"]");

    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

```



### 5.2.2 利用telnet命令行测试EchoServer服务端

telnet命令相关操作此处不赘述。

以20字节长度进行截取。

## 5.3 总结

- DelimiterBasedFrameDecoder用于对使用分隔符结尾的消息进行自动解码。
- FixLengthFrameDecoder用于对固定长度的消息进行自动解码。

# 第6章 编解码技术

Java序列化的目的：

- 网络传输
- 对象持久化

Java对象编解码技术：在进行远程跨进程服务调用时，需要把被传输的Java对象编码为字节数组或者ByteBuffer对象。而当远程服务读取到ByteBuffer对象或者字节数组时，需要将其解码为发送时的Java对象。

Java序列化仅仅是Java编解码技术的一种，有缺陷。

## 6.1 Java序列化的缺点

RPC很少使用Java序列化进行消息的编解码和传输。原因如下：

### 6.1.1 无法跨语言

Java序列化技术时Java语言内部的私有协议，其他语言并不支持，对于用户来说完全是黑盒。对于Java序列化后的字节数组，别的语言无法进行反序列化。

### 6.1.2 序列化后的码流太大

序列化测试：

```java
package _6_1_2_;

import java.io.Serializable;
import java.nio.ByteBuffer;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/19 12:52
 */


public class UserInfo implements Serializable {

    private static final long serialVersionID = 1L;

    private String userName;
    private int userID;

    public UserInfo(String userName, int userID) {
        this.userName = userName;
        this.userID = userID;
    }

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public int getUserID() {
        return userID;
    }

    public void setUserID(int userID) {
        this.userID = userID;
    }

    public byte[] codeC() {
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        byte[] value = this.userName.getBytes();
        buffer.putInt(value.length);
        buffer.put(value);
        buffer.putInt(this.userID);
        buffer.flip();

        value = null;
        byte[] result = new byte[buffer.remaining()];
        buffer.get(result);
        return result;
    }
}

```

```java
package _6_1_2_;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.ObjectOutputStream;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/19 12:59
 */


public class TestUserInfo {
    public static void main(String[] args) throws IOException {
        UserInfo info = new UserInfo("Welcome to Netty", 100);
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutputStream os = new ObjectOutputStream(bos);
        os.writeObject(info);
        os.flush();
        os.close();
        byte[] b = bos.toByteArray();
        System.out.println("JDK序列化长度：" + b.length);
        bos.close();
        System.out.println("-------------------------");
        System.out.println("byte数组序列化长度：" + info.codeC().length);
    }
}

```

测试结果：JDK序列化机制编码后的二进制数组大小是二进制编码的5倍多。

评判编解码框架的优劣时，考虑如下因素：

- 是否支持跨语言，支持的语言种类是否丰富；
- 编码后的码流大小；
- 编解码的性能；
- 类库是否小巧，API使用是否方便；
- 使用者需要手工开发的工作量和难度。

编码后的字节数组大，存储时占用空间大，硬件成本大，网络传输时更占用带宽，导致系统的吞吐量降低。

### 6.1.3 序列化性能太低

性能测试：

```java
package _6_1_3_;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.ObjectOutputStream;
import java.nio.ByteBuffer;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/19 13:43
 */


public class PerformTestUserInfo {

    public static void main(String[] args) throws IOException {
        UserInfo info = new UserInfo("Welcome to Netty", 100);
        int loop = 1000000;

        ByteArrayOutputStream bos = null;
        ObjectOutputStream os = null;

        long startTime = System.currentTimeMillis();
        for (int i = 0; i < loop; i++) {
            bos = new ByteArrayOutputStream();
            os = new ObjectOutputStream(bos);
            os.flush();
            os.close();
            byte[] b = bos.toByteArray();
            bos.close();
        }

        long endTime = System.currentTimeMillis();
        System.out.println("JDK序列化用时："+(endTime - startTime)+" ms");

        System.out.println("=========================================");
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        startTime = System.currentTimeMillis();
        for (int i = 0; i < loop; i++) {
            byte[] b = info.codeC(buffer);
        }
        endTime = System.currentTimeMillis();
        System.out.println("byte数组序列化用时："+(endTime - startTime)+" ms");
    }
}

```

Java序列化性能只有二进制编码的6%左右，性能很低。

## 6.2 业界主流的编解码框架

略

## 6.3 总结

# 第7章 MessagePack编解码

MessagePack是一个高效的二进制序列化框架。像JSON一样支持不同语言间的数据交换，但是性能更快，序列化之后的码流也更小。

## 7.1 MessagePack介绍

特点如下：

- 编解码高效，性能高；
- 序列化之后的码流小；
- 支持跨语言。

### 7.1.1 MessagePack多语言支持

几乎都支持，不例举。

### 7.1.2 MessagePack Java API介绍

略

## 7.2 MessagePack编码器和解码器开发

Netty的编解码框架可以非常方便的继承第三方序列化框架。

### 7.2.1 MessagePack编码器开发

```java
package _7_2_1_;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.MessageToByteEncoder;
import org.msgpack.MessagePack;

import java.util.List;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/19 14:05
 */


public class MsgpackEncoder extends MessageToByteEncoder<Object> {
    @Override
    protected void encode(ChannelHandlerContext atg0, Object arg1, ByteBuf arg2) throws Exception {
        MessagePack msgpack = new MessagePack();
        byte[] raw = msgpack.write(arg1);
        arg2.writeBytes(raw);
    }
}

```

MsgpackEncoder继承MessageToByteEncoder，负责将Object类型的POJO对象编码为byte数组，然后写入到ByteBuf中。

### 7.2.2 MessagePack解码器开发

```java
package _7_2_1_;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.MessageToMessageDecoder;
import org.msgpack.MessagePack;

import java.util.List;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/19 14:39
 */


public class MsgpackDecoder extends MessageToMessageDecoder<ByteBuf> {

    @Override
    protected void decode(ChannelHandlerContext arg0, ByteBuf arg1, List<Object> arg2) throws Exception {
        final byte[] array;
        final int length = arg1.readableBytes();
        array = new byte[length];
        arg1.getBytes(arg1.readerIndex(), array, 0, length);
        MessagePack msgpack = new MessagePack();
        arg2.add(msgpack.read(array));//解码
    }
}

```

### 7.2.3 功能测试

暂未通过；

# 第8章 Google Protobuf编解码

略

# 第9章 JBoss Marshalling编解码

# 第10章 HTTP协议开发应用

HTTP是一个属于应用层的面向对象的协议。Netty的HTTP协议也是异步非阻塞的。

## 10.1 HTTP协议介绍

主要特点：

- 支持C/S模式；
- 简单——客户端向服务端请求服务，秩序指定服务URL，携带必要的请求参数或消息体；
- 灵活——HTTP允许传输任意类型的数据对象，传输的内容类型由HTTP消息头中的Content-Type加以标记；
- 无状态——对事务处理没有记忆能力。

### 10.1.1 HTTP协议的URL

```url
http://host[":"port][abs_path]
```

详细介绍不赘述。

### 10.1.2 HTTP请求消息（HttpRequest）

三部分：

- HTTP请求行；
- HTTP消息头；
- HTTP请求正文。

GET和POST的区别：

（1）根据HTTP规范，GET用于信息获取，应该是安全的和幂等的；POST则表示可能改变服务器上的资源的请求。

（2）GET提交，请求的数据会附在URL之后，就是把数据放置在请求行中，以“?”分隔URL和传输数据，多个参数用“&”连接；而POST提交会把提交的数据放置再HTTP消息的包体中，数据不会在地址栏中显示出来。

（3）传输数据的大小不同，GET受具体浏览器长度的限制。POST理论上长度不受限制。

（4）安全性。

### 10.1.3 HTTP响应消息（HttpResponse）

三部分：

- 状态行；
- 消息报头；
- 响应正文。

## 10.2 Netty HTTP服务端入门开发

相比于传统的Tomcat、Jetty等Web容器，它更加轻量和小巧，灵活性和定制性也更好。

### 10.2.1 HTTP服务端例程场景描述

例程场景如下：文件服务器使用HTTP协议对外提供服务，当客户端通过浏览器访问文件服务器时，对访问路径进行检查，检查失败时返回HTTP403错误，该页无法访问；如果校验通过，以连接的方式打开当前文件目录，每个目录或者文件都是个超链接，可以递归访问。

如果是目录，可以继续递归访问它下面的子目录或者文件，如果时文件且可读，则可以在浏览器端直接打开，或者通过【目标另存为】下载该文件。

### 10.2.2 HTTP服务端开发

```java
package HTTP;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpRequestDecoder;
import io.netty.handler.codec.http.HttpResponseEncoder;
import io.netty.handler.stream.ChunkedWriteHandler;

import java.net.InetAddress;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/21 10:44
 */


public class HttpFileServer {

    private static final String DEFAULT_URL = "/src";

    public void run(final int port, final String url) throws Exception {

        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast("http-decoder", new HttpRequestDecoder());//向ChannelPipeLine中添加HTTP请求消息解码器
                            ch.pipeline().addLast("http-aggregator", new HttpObjectAggregator(65536));//添加HttpObjectAggregator解码器
                            ch.pipeline().addLast("http-encoder", new HttpResponseEncoder());//HTTP响应编码器
                            ch.pipeline().addLast("http-chunked", new ChunkedWriteHandler());//支持异步发送大的码流（例如大文件传输，但不占用过多内存，防止Java内存溢出错误）
                            ch.pipeline().addLast("fileServerHandler", new HttpFileServerHandler(url));//
                        }
                    });

            ChannelFuture f = b.bind(InetAddress.getLocalHost().getHostAddress(), port).sync();
            System.out.println("Http文件目录服务器启动，网址是：" + InetAddress.getLocalHost().getHostAddress() +":" + port + url);
            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception{
        int port = 8080;
        if (args != null && args.length > 0) {
            try {
                port = Integer.valueOf(args[0]);
            } catch (NumberFormatException e) {
                e.printStackTrace();
            }
        }

        String url = DEFAULT_URL;
        if (args.length > 1)
            url = args[1];

        new HttpFileServer().run(port, url);
    }

}

```

```java
package HTTP;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.*;
import io.netty.handler.codec.http.*;
import io.netty.handler.stream.ChunkedFile;

import javax.activation.MimetypesFileTypeMap;
import java.io.File;
import java.io.FileNotFoundException;
import java.io.RandomAccessFile;
import java.io.UnsupportedEncodingException;
import java.net.URLDecoder;
import java.util.regex.Pattern;

import static io.netty.handler.codec.http.HttpHeaders.Names.*;
import static io.netty.handler.codec.http.HttpHeaders.isKeepAlive;
import static io.netty.handler.codec.http.HttpHeaders.setContentLength;
import static io.netty.handler.codec.http.HttpMethod.GET;
import static io.netty.handler.codec.http.HttpResponseStatus.*;
import static io.netty.handler.codec.http.HttpVersion.HTTP_1_1;
import static io.netty.util.CharsetUtil.UTF_8;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/21 11:02
 */


public class HttpFileServerHandler extends SimpleChannelInboundHandler<FullHttpRequest> {
    private final String url;

    public HttpFileServerHandler(String url) {
        this.url = url;
    }

    @Override
    protected void messageReceived(ChannelHandlerContext ctx, FullHttpRequest request) throws Exception {
        //对Http请求消息的解码结果进行判断，如果解码失败，直接构造HTTP400错误返回。
        if (!request.getDecoderResult().isSuccess()) {
            sendError(ctx, BAD_REQUEST);
            return;
        }

        //对请求行中的方法进行判断，如果不是从浏览器或者表单设置为GET发起的请求（例如POST），则构造HTTP405错误返回。
        if (request.getMethod() != GET) {
            sendError(ctx, METHOD_NOT_ALLOWED);
            return;
        }

        final String uri = request.getUri();
        //对URL进行包装
        final String path = sanitizeUri(uri);

        //如果构造的URI不合法，则返回403错误
        if (path == null) {
            sendError(ctx, FORBIDDEN);
            return;
        }

        //使用新组装的URI路径构造File对象。
        File file = new File(path);

        //如果文件不存在或是系统隐藏文件，则构造Http404异常返回
        if (file.isHidden() || !file.exists()) {
            sendError(ctx, NOT_FOUND);
            return;
        }

        //如果文件是目录，则发送目录的链接给客户端连接
        if (file.isDirectory()) {
            if (uri.endsWith("/")) {
                sendListing(ctx, file);
            } else {
                sendRedirect(ctx, uri + "/");
            }
            return;
        }

        //点击或下载文件，校验文件合法性。
        if (!file.isFile()) {
            sendError(ctx, FORBIDDEN);
            return;
        }

        //使用随机文件读写类以制度的方式打开文件，如果文件打开失败，则返回404错误。
        RandomAccessFile randomAccessFile = null;
        try {
            randomAccessFile = new RandomAccessFile(file, "r");//以只读方式打开文件
        } catch (FileNotFoundException fnfe) {
            sendError(ctx, NOT_FOUND);
            return;
        }

        //获取文件的长度，构造成功的Http应答信息
        long fileLength = randomAccessFile.length();
        HttpResponse response = new DefaultFullHttpResponse(HTTP_1_1, OK);

        setContentLength(response, fileLength);
        setContentTypeHeader(response, file);

        //判断连接是否KeepAlive，如果是，则设置
        if (isKeepAlive(request)) {
            response.headers().set(CONNECTION, HttpHeaders.Values.KEEP_ALIVE);
        }

        ctx.write(response);
        ChannelFuture sendFileFuture;

        //通过Netty的ChunkedFile对象直接将文件写入到发送缓冲区。
        sendFileFuture = ctx.write(new ChunkedFile(randomAccessFile, 0, fileLength, 8192),
                ctx.newProgressivePromise());
        sendFileFuture.addListener(new ChannelProgressiveFutureListener() {
            @Override
            public void operationProgressed(ChannelProgressiveFuture future, long progress, long total) throws Exception {
                if (total < 0)
                    System.err.println("传输出错：" + progress);
                else
                    System.err.println("传输进度：" + progress+"/"+total);
            }

            @Override
            public void operationComplete(ChannelProgressiveFuture future) throws Exception {
                System.out.println("传输完成。");
            }
        });

        //使用chunked编码，最后需要发送一个编码结束的空消息体，将LastHttpContent.EMPTY_LAST_CONTENT发送到缓冲区，标识所有消息体都发送完成。
        ChannelFuture lastContentFuture = ctx.writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT);
        if (!isKeepAlive(request)) {
            lastContentFuture.addListener(ChannelFutureListener.CLOSE);
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        if (ctx.channel().isActive()) {
            sendError(ctx, INTERNAL_SERVER_ERROR);
        }
    }

    private  static final Pattern INSECURE_URI = Pattern.compile(".*[<>&\"].*");
    private String sanitizeUri(String uri) {
        try {
            //使用java.net.URLDecoder对URL进行解码，使用UTF-8字符集。
            uri = URLDecoder.decode(uri, "UTF-8");
        } catch (UnsupportedEncodingException e) {
            try {
                uri = URLDecoder.decode(uri, "ISO-8859-1");
            } catch (UnsupportedEncodingException ex) {
                throw new Error();
            }
        }

        //对URI进行合法性判断，如果URI与允许访问的URI一致或者是其子目录（文件），则校验通过，否则返回空。
        if (!uri.startsWith(url))
            return null;
        if (!uri.startsWith("/"))
            return null;

        //将硬编码的文件路径替换为本地操作系统的文件路径分隔符。
        uri = uri.replace('/', File.separatorChar);
        //对新的URI进行二次合法性校验。
        if(uri.contains(File.separator + '.') || uri.contains('.' + File.separator) ||
                uri.startsWith(".") || uri.endsWith(".") || INSECURE_URI.matcher(uri).matches()){
            return null;
        }

        //校验完成，使用当前运行程序所在的工作目录+URI构造绝对路径返回。
        return System.getProperty("user.dir") + File.separator + uri;
    }

    private static final Pattern ALLOWED_FILE_NAME = Pattern.compile("[A-Za-z0-9][-_A-Za-z0-9\\.]*");
    private void sendListing(ChannelHandlerContext ctx, File dir) {

        //创建成功的Http响应消息，并设置消息头
        FullHttpResponse response = new DefaultFullHttpResponse(HTTP_1_1, OK);
        response.headers().set(CONTENT_TYPE, "text/html;charset=UTF-8");

        //消息体
        String dirPath = dir.getPath();
        StringBuilder buf = new StringBuilder();

        buf.append("<!DOCTYPE html>\r\n");
        buf.append("<html><head><title>");
        buf.append(dirPath);
        buf.append("目录:");
        buf.append("</title></head><body>\r\n");

        buf.append("<h3>");
        buf.append(dirPath).append(" 目录：");
        buf.append("</h3>\r\n");
        buf.append("<ul>");
        buf.append("<li>链接：<a href=\" ../\">..</a></li>\r\n");

        //展示所有文件和文件夹，同时使用超链接来标识
        for (File f : dir.listFiles()) {
            if(f.isHidden() || !f.canRead()) {
                continue;
            }
            String name = f.getName();
            if (!ALLOWED_FILE_NAME.matcher(name).matches()) {
                continue;
            }

            buf.append("<li>链接：<a href=\"");
            buf.append(name);
            buf.append("\">");
            buf.append(name);
            buf.append("</a></li>\r\n");
        }

        buf.append("</ul></body></html>\r\n");

        //分配对应消息的缓冲对象。
        ByteBuf buffer = Unpooled.copiedBuffer(buf, UTF_8);

        //将缓冲区中的响应消息存放到Http应答消息中，然后释放缓冲区
        response.content().writeBytes(buffer);
        buffer.release();
        ctx.writeAndFlush(response).addListener(ChannelFutureListener.CLOSE);
    }

    private void sendRedirect(ChannelHandlerContext ctx, String newUri) {
        FullHttpResponse response = new DefaultFullHttpResponse(HTTP_1_1, FOUND);
        response.headers().set(LOCATION,newUri);
        ctx.writeAndFlush(response).addListener(ChannelFutureListener.CLOSE);
    }

    private void sendError(ChannelHandlerContext ctx, HttpResponseStatus status) {
        FullHttpResponse response = new DefaultFullHttpResponse(HTTP_1_1, status,
                Unpooled.copiedBuffer("Failure: " + status.toString() + "\r\n", UTF_8));
        response.headers().set(CONTENT_TYPE, "text/html;charset=UTF-8");
        ctx.writeAndFlush(response).addListener(ChannelFutureListener.CLOSE);
    }

    private void setContentTypeHeader(HttpResponse response, File file) {
        MimetypesFileTypeMap mimetypesFileTypeMap = new MimetypesFileTypeMap();
        response.headers().set(CONTENT_TYPE, mimetypesFileTypeMap.getContentType(file.getPath()));
    }
}

```

## 10.3  Netty HTTP+XML协议栈开发

由于HTTP协议的通用性，很多异构系统间的通信交互采用HTTP协议，通过HTTP协议承载业务数据进行消息交互。例如非常流行的HTTP+XML或者Restful+JSON。

很多基于HTTP的应用都是后台应用，HTTP仅仅是承载数据交换的一个通道，是一个载体而不是Web容器。所以在这种场景下，一般不需要Tomcat这样的重量行Web容器。

且Tomcat会有很多安全漏洞。所以一个轻量级的HTTP协议栈是个更好的选择。

### 10.3.1 开发场景介绍

模拟一个简单的用户订购系统。客户端填写订单，通过HTTP客户端向服务端发送订购请求，请求消息放在HTTP消息体中，以XML承载，即采用HTTP+XML的方式进行通信。

双方采用HTTP1.1协议，连接类型为CLOSE方式，即双方交互完成，由HTTP服务端主动关闭链路，随后客户端也关闭链路并退出。

订购请求消息定义如表所示：

| 字段名称 | 类型      | 备注                                                         |
| -------- | --------- | ------------------------------------------------------------ |
| 订购数量 | Int64     | 订购的商品数量                                               |
| 客户信息 | Customer  | 客户信息，负责POJO对象                                       |
| 账单地址 | Address   | 账单的地址                                                   |
| 寄送方式 | Shippling | 枚举类型如下：<br>普通快递<br>宅急送<br>国际配送<br>国内快递<br>国际快递 |
| 送货地址 | Address   |                                                              |
| 总价     | float     |                                                              |

 客户信息定义：

| 字段名称 | 类型          | 备注               |
| -------- | ------------- | ------------------ |
| 客户ID   | Int64         | 客户ID，厂整型     |
| 姓       | String        | 客户姓氏，字符串   |
| 名       | String        | 客户名字，字符串   |
| 全名     | List\<String> | 客户全程，字符列表 |

地址信息定义：

| 字段名称 | 类型   | 备注 |
| -------- | ------ | ---- |
| 街道1    | String |      |
| 街道2    | String |      |
| 城市     | String |      |
| 省份     | String |      |
| 邮政编码 | String |      |
| 国家     | String |      |

邮递方式定义：

| 字段名称 | 类型     | 备注 |
| -------- | -------- | ---- |
| 普通邮递 | 枚举类型 |      |
| 宅急送   | 枚举类型 |      |
| 国际邮递 | 枚举类型 |      |
| 国内快递 | 枚举类型 |      |
| 国际快递 | 枚举类型 |      |

### 10.3.2 HTTP+XML协议栈设计

步骤：

1. 构造订购请求消息，将请求消息编码为HTTP+XML格式；
2. HTTP客户端发起连接，通过HTTP协议栈发送HTTP请求消息；
3. HTTP服务端对HTTP+XML请求消息进行解码，解码成请求POJO；
4. 服务端构造应答消息并编码，通过HTTP+XML方式返回给客户端；
5. 客户端对HTTP+XML响应消息进行解码，解码成响应POJO。

流程分析：

步骤1，需要自定义HTTP+XML格式的请求消息编码器；

步骤2，可重用Netty的能力；

步骤3，Netty可解析HTTP请求消息，但是消息体为XML，Netty无法将其解码为POJO，需要在Netty协议栈的基础上扩展实现。

步骤4，需定制将POJO以XML方式发送

步骤5，需定制解码。

设计思路：

（1）需要一套通用、高性能的XML序列化框架，它能够灵活的实现POJO-XML的互相转换，最好能够通过工具自动生成绑定关系，或者通过XML的方式配置双方的映射关系；

（2）作为通用的HTTP+XML协议栈，XML-POJO的映射关系应该非常灵活，支持命名空间和自定义标签。

（3）一系列HTTP+XML消息编解码器

（4）编解码过程应该对协议栈使用者透明，对上层业务零侵入。

### 10.3.3 高效的XML绑定框架JiBx

1. JiBX入门

专门为Java语言设计的XML数据绑定框架JiBx。

优点：转换效率高、配置绑定文件简单、不需要操作xpath文件、不需要写输行的get/set方法、对象属性名与XML文件element名可以不同等。

绑定XML与Java对象的两个步骤：第一步是绑定XML文件，也就是映射XML文件与Java对象之间的对应关系；第二步是在运行时，实现XML文件与Java势力之间的互相转换。

在运行程序之前，需要先配置绑定文件并进行绑定，在绑定过沉重它将会动态地修改程序中相应地class文件，主要是生成对应对象实例地方法和添加呗绑定标记地输行JiBX_bindingList等。它使用的技术是BCEL(Byte Code Engineering Library)，BCEL是Apache Software Foundation的Jakarta项目的一部分，也是目前Java classworking最广泛使用的一种框架，它可以让你深入JVM汇编语言进行类操作。在JiBX运行时，它使用了目前比较流行的一个技术XPP（Xml Pull Parsing），这也是JiBX如此高效的原因。

XPP：将整个文档写入内存，然后进行DOM操作，也不是使用基于事件流的SAX。XPP使用饿是不断增加的数据流处理方式，同时允许在解析XML文件时中断。

因书上的项目为Ant打包，与我使用的Maven有冲突，所以开发过程略。

# 第11章 WebSocket协议开发

WebSocket解决的问题：由于HTTP协议的开销，导致它们不合适低延迟应用。

WebSocket将网络套接字引入到了客户端和服务端，浏览器和服务器之间可以通过套接字建立持久的连接，双方随时都可以互发数据给对方，而不是之前由客户端控制的一请求一应答模式。

## 11.1 HTTP协议的弊端

主要弊端如下：

（1）HTTP协议为半双工。可以双向但不能同时。

（2）HTTP消息冗长而繁琐。HTTP消息包含消息头、消息体、换行符等。通常以文本传输，相比于其他二进制通信协议，冗长而繁琐。

（3）针对服务器推送的黑客攻击。例如长时间轮询。

## 11.2 WebSocket入门

H5提供的一种浏览器与服务器间进行全双工通信的网络技术。

在WebSocketAPI中，浏览器和服务器只需要一个握手的动作，然后浏览器和服务器直接就形成了一条快速通道，两者就可以直接互相传送数据了。基于TCP双向全双工进行消息传递。

WebSocket的特点：

- 单一的TCP连接，采用全双工模式通信；
- 对代理、防火墙和路由器透明；
- 无头部信息、Cookie和身份验证；
- 无安全开销；
- 通过“ping/pong”帧保持链路激活；
- 服务器可以主动传递消息给客户端，不再需要客户端轮询。

### 11.2.1 WebSocket背景

取代轮询和Comet技术，是客户端浏览器具备像C/S架构下桌面系统一样的实时通信能力。

在流量和负载增大的情况下，WebSocket方案相比传统的AJAX轮询方案有极大的性能优势。

### 11.2.2 WebSocket连接建立

发送一个HTTP请求，与平常的不同，包含一些附加头信息。

返回的也包含附加信息。

### 11.2.3 WebSocket生命周期

握手成功后，就能通过“messages”的方式进行通信了。一个消息由一个或多个帧组成。可以被分割成多个帧或者被合并。

### 11.2.4 WebSocket连接关闭

为关闭WebSocket连接，客户端与服务端需要通过一个安全的方法关闭底层TCP连接以及TLS会话。如果合适，丢弃任何可能已经接收的字节，必要时（比如受到攻击）可以通过任何可用的手段关闭连接。

底层的TCP连接，在正常情况下，应该首先由服务器关闭。在异常情况下（例如在一个合理的时间周期后没有接收到服务器的TCP Close），客户端可以发起TCP Close。因此，当服务器被指示关闭WebSocket连接时，它应该立即发起一个TCP Close操作；客户端应该等待服务器的TCP Close。

WebSocket的握手关闭消息带有一个状态码和一个可选的关闭原因，它必须按照协议要求发送一个Close控制帧，当对端接收到关闭控制帧指令时，需要主动关闭WebSocket连接。

## 11.3 Netty WebSocket协议开发

### 11.3.1 WebSocket服务端功能介绍

支持WebSocket的浏览器通过WebSocket协议发送请求消息给服务端，服务端对请求消息进行判断，如果时合法的WebSocket请求，则获取请求消息体（文本），并在后面追加字符串“欢迎使用Netty WebSocket服务，现在时刻：系统时间”。

客户端HTML通过内嵌的JS脚本创建WebSocket连接，如果握手成功，在文本框中打印“打开WebSocket服务正常。浏览器支持WebSocket！”。

### 11.3.2 WebSocket服务端开发

```java
package WebSocket.server;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.Channel;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;

import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpServerCodec;
import io.netty.handler.stream.ChunkedWriteHandler;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/23 13:12
 */


public class WebSocketServer {

    public void run(int port) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast("http-codec", new HttpServerCodec());//添加HttpServerCodec，将请求和应答消息编码或者解码为HTTP消息；
                            pipeline.addLast("aggregator", new HttpObjectAggregator(65536));//添加HttpObjectAggregator，将HTTP消息的多个部分组合成一条完整的HTTP消息
                            ch.pipeline().addLast("http-chunked", new ChunkedWriteHandler());//向客户端发送H5文件，主要用于支持浏览器和服务端进行WebSocket通信
                            pipeline.addLast("handler", new WebSocketServerHandler());
                        }
                    });
            Channel ch = b.bind(port).sync().channel();
            System.out.println("Websocket服务器启动，端口：" + port + '。');
            System.out.println("浏览器访问"+"http://localhost:"+ port + '/');
            ch.closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception {
        int port = 8080;
        if (args.length > 0 ) {
            try {
                port = Integer.parseInt(args[0]);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        new WebSocketServer().run(port);
    }
}

```

```java
package WebSocket.server;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.codec.http.DefaultFullHttpResponse;
import io.netty.handler.codec.http.FullHttpRequest;
import io.netty.handler.codec.http.websocketx.*;

import java.util.logging.Level;
import java.util.logging.Logger;

import static io.netty.handler.codec.http.HttpHeaders.isKeepAlive;
import static io.netty.handler.codec.http.HttpHeaders.setContentLength;
import static io.netty.handler.codec.http.HttpResponseStatus.BAD_REQUEST;
import static io.netty.handler.codec.http.HttpVersion.HTTP_1_1;
import static io.netty.util.CharsetUtil.UTF_8;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/23 13:33
 */


public class WebSocketServerHandler extends SimpleChannelInboundHandler<Object> {

    private static final Logger logger = Logger.getLogger(WebSocketServerHandler.class.getName());
    private WebSocketServerHandshaker handshaker;

    @Override
    protected void messageReceived(ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof FullHttpRequest) {//握手请求
            handleHttpRequest(ctx, (FullHttpRequest) msg);
        } else if (msg instanceof WebSocketFrame) {
            handleWebSocketFrame(ctx, (WebSocketFrame) msg);
        }
    }


    private void handleHttpRequest(ChannelHandlerContext ctx, FullHttpRequest req) {
        if (!req.getDecoderResult().isSuccess() || (!"websocket".equals(req.headers().get("Upgrade")))) {
            sendHttpResponse(ctx, req, new DefaultFullHttpResponse(HTTP_1_1, BAD_REQUEST));//如果不是握手请求就返回400；
            return;
        }

        //构造握手工厂
        WebSocketServerHandshakerFactory wsFactory = new WebSocketServerHandshakerFactory("ws://localhost:8080/socket", null, false);
        //握手处理类
        handshaker = wsFactory.newHandshaker(req);
        if (handshaker == null) {
            WebSocketServerHandshakerFactory.sendUnsupportedWebSocketVersionResponse(ctx.channel());
        } else {
            handshaker.handshake(ctx.channel(), req);
        }
    }

    private void sendHttpResponse(ChannelHandlerContext ctx, FullHttpRequest req, DefaultFullHttpResponse resp) {
        if (resp.getStatus().code() != 200) {
            ByteBuf buf = Unpooled.copiedBuffer(resp.getStatus().toString(), UTF_8);
            resp.content().writeBytes(buf);
            buf.release();
            setContentLength(resp, resp.content().readableBytes());
        }

        ChannelFuture f =ctx.channel().writeAndFlush(resp);
        if (!isKeepAlive(req) || resp.getStatus().code() != 200) {
            f.addListener(ChannelFutureListener.CLOSE);
        }
    }

    private void handleWebSocketFrame(ChannelHandlerContext ctx, WebSocketFrame frame) {
        if (frame instanceof CloseWebSocketFrame) {
            handshaker.close(ctx.channel(), ((CloseWebSocketFrame) frame).retain());
            return;
        }

        if (frame instanceof PingWebSocketFrame) {
            ctx.channel().write(new PongWebSocketFrame(frame.content().retain()));
            return;
        }

        if (!(frame instanceof TextWebSocketFrame)) {
            throw new UnsupportedOperationException(String.format("%s frame types not supported", frame.getClass().getName()));
        }

        String request = ((TextWebSocketFrame) frame).text();
        logger.info(request);
        if (logger.isLoggable(Level.FINE)) {
            logger.fine(String.format("%s received %s", ctx.channel(), request));
        }

        ctx.channel().write(
                new TextWebSocketFrame(request
                        + " , 欢迎使用Netty WebSocket服务，现在时刻："
                        + new java.util.Date().toString()));
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

```

WebSocket本身非常复杂，可以通过多种形式（文本方式，二进制方式）承载信息。

# 第12章 私有协议栈开发

私有协议灵活，在公司内部或组织内部使用。

绝大多数私有协议传输层采用TCP/IP。

## 12.1 私有协议介绍

除非授权，不然其他厂商无法使用。私有协议也称非标准协议。

传统的Java应用中，通常使用以下4中方式进行跨节点通信：

（1）通过RMI进行远程服务调用；

（2）通过Java的Socket+Java序列化的方式进行跨节点调用；

（3）利用一些开源的RPC框架进行远程服务嗲用，如Facebook的Thrift、Apache的Avro等；

（4）利用标准的公有协议进行跨节点调用，例如HTTP+XML、RESTful+JSON或者WebService。

私有协议：链路层的物理连接，对请求和响应消息进行编码，在请求和应答消息本身以外，也需要携带一些其他控制和管理类指令，例如链路建立的握手请求和响应消息、链路检测的心跳消息等。

并没有标准的定义，只要能够用于跨进程、跨主机数据交换的非标准协议，都可以称为私有协议。

## 12.2 Netty协议栈功能设计

用于内部各模块之间的通信， 它基于TCP/IP协议栈，是一个类HTTP协议的应用层协议栈。

### 12.1.1 网络拓扑图

无固定客户端，服务端。谁发起谁就是客户端，接收时服务端。

### 12.2.2 协议栈功能描述

（1）基于Netty的NIO通信框架，提供高性能饿异步通信能力；

（2）提供消息的编解码框架，可以实现POJO的序列化和反序列化；

（3）提供基于IP地址的白名单接入认证机制；

（4）链路的有效性检验机制；

（5）链路的断连重连机制。

### 12.2.3 通信模型

具体步骤：

（1）Netty协议栈客户端发送握手请求消息，携带节点ID等有效身份认证信息；

（2）Netty协议栈服务端对握手请求消息进行合法性校验，包括节点ID有效性校验、节点重复登录校验和IP地址合法性校验，校验通过后，返回登录成功的握手应答消息；

（3）链路建立成功之后，客户端发送业务；

（4）链路成功之后，服务端发送心跳信息；

（5）链路建立成功之后，客户端法统心跳信息；

（6）链路建立成功之后，服务端发送业务消息；

（7）服务端退出时，服务端关闭连接，客户端感知对方关闭连接后，被动关闭客户端连接。

### 12.2.4 消息定义

两部分：

- 消息头；
- 消息体。

协议详细描述部分不赘述，因为是私有协议，内容不重要，开发过程才是重点，之后看Dubbo在着重看协议内容。

### 12.3 Netty协议栈开发

### 12.3.1 数据结构定义

```java
package PrivateProtocol.struct;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/23 14:49
 */


public class NettyMessage {

    private Header header;//消息头
    private Object body;//消息体

    public Header getHeader() {
        return header;
    }

    public void setHeader(Header header) {
        this.header = header;
    }

    public Object getBody() {
        return body;
    }

    public void setBody(Object body) {
        this.body = body;
    }

    @Override
    public String toString() {
        return "NettyMessage{" +
                "header=" + header +
                ", body=" + body +
                '}';
    }
}

```

```java
package PrivateProtocol.struct;

import java.util.HashMap;
import java.util.Map;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/23 14:50
 */


public class Header {

    private int crcCode = 0xabef0101;
    private int length;//消息长度
    private long sessionID;//会话ID
    private byte type;//消息类型
    private byte priority;//消息优先级
    private Map<String, Object> attachment = new HashMap<String, Object>();//附件

    public int getCrcCode() {
        return crcCode;
    }

    public void setCrcCode(int crcCode) {
        this.crcCode = crcCode;
    }

    public int getLength() {
        return length;
    }

    public void setLength(int length) {
        this.length = length;
    }

    public long getSessionID() {
        return sessionID;
    }

    public void setSessionID(long sessionID) {
        this.sessionID = sessionID;
    }

    public byte getType() {
        return type;
    }

    public void setType(byte type) {
        this.type = type;
    }

    public byte getPriority() {
        return priority;
    }

    public void setPriority(byte priority) {
        this.priority = priority;
    }

    public Map<String, Object> getAttachment() {
        return attachment;
    }

    public void setAttachment(Map<String, Object> attachment) {
        this.attachment = attachment;
    }

    @Override
    public String toString() {
        return "Header{" +
                "crcCode=" + crcCode +
                ", length=" + length +
                ", sessionID=" + sessionID +
                ", type=" + type +
                ", priority=" + priority +
                ", attachment=" + attachment +
                '}';
    }
}

```

由于心跳消息、握手请求和握手应答消息都可以统一由NettyMessage承载，所以不需要为这几类控制消息做单独的数据结构定义。

### 12.3.2 消息编解码

```java
package PrivateProtocol.codec;

import PrivateProtocol.struct.NettyMessage;
import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.MessageToByteEncoder;


import java.io.IOException;
import java.util.Map;


/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/23 14:59
 */


public class NettyMessageEncoder extends MessageToByteEncoder<NettyMessage> {

    MarshallingEncoder marshallingEncoder;
    public NettyMessageEncoder() throws IOException{
        this.marshallingEncoder = new MarshallingEncoder();
    }

    @Override
    protected void encode(ChannelHandlerContext ctx, NettyMessage msg, ByteBuf sendBuf) throws Exception {
        if (msg == null || msg.getHeader() == null) {
            throw new Exception("The encode message is null");
        }
        sendBuf.writeInt(msg.getHeader().getCrcCode());
        sendBuf.writeInt(msg.getHeader().getLength());
        sendBuf.writeLong(msg.getHeader().getSessionID());
        sendBuf.writeByte(msg.getHeader().getType());
        sendBuf.writeByte(msg.getHeader().getPriority());
        sendBuf.writeInt(msg.getHeader().getAttachment().size());

        String key = null;
        byte[] keyArray = null;
        Object value = null;
        for (Map.Entry<String, Object> param : msg.getHeader().getAttachment().entrySet()) {
            key = param.getKey();
            keyArray = key.getBytes("UTF-8");
            sendBuf.writeInt(keyArray.length);
            sendBuf.writeBytes(keyArray);
            value = param.getValue();
            marshallingEncoder.encode(value, sendBuf);
        }

        key = null;
        keyArray = null;
        value = null;
        if (msg.getBody() != null) {
            marshallingEncoder.encode(msg.getBody(), sendBuf);
        } else
            sendBuf.writeInt(0);
        sendBuf.setInt(4, sendBuf.readableBytes() - 8);
    }
}

```

```java
package PrivateProtocol.codec;

import io.netty.buffer.ByteBuf;
import org.jboss.marshalling.Marshaller;

import java.io.IOException;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/23 15:06
 */


public class MarshallingEncoder {

    private static final byte[] LENGTH_PLACEHOLDER = new byte[4];

    Marshaller marshaller;

    public MarshallingEncoder() throws IOException {
        marshaller = MarshallingCodecFactory.buildMarshalling();
    }


    public void encode(Object msg, ByteBuf out) throws IOException {
        try {
            int lengthPos = out.writerIndex();
            out.writeBytes(LENGTH_PLACEHOLDER);
            ChannelBufferByteOutput output = new ChannelBufferByteOutput(out);
            marshaller.start(output);
            marshaller.writeObject(msg);
            marshaller.finish();
            out.setInt(lengthPos, out.writerIndex() - lengthPos - 4);
        } finally {
            marshaller.close();
        }
    }
}

```

```java
package PrivateProtocol.codec;


import PrivateProtocol.struct.Header;
import PrivateProtocol.struct.NettyMessage;
import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.LengthFieldBasedFrameDecoder;

import java.io.IOException;
import java.nio.ByteOrder;
import java.util.HashMap;
import java.util.Map;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/23 15:36
 */

/*
 *Netty的LengthFieldBasedFrameDecoder解码器，它支持自动的TCP粘包和半包处理，
 *只需要给出标识消息长度饿字段偏移量和消息长度自身所占的字节数，Netty就能自动实现对半包的处理。
 */
public class NettyMessageDecoder extends LengthFieldBasedFrameDecoder {
    MarshallingDecoder marshallingDecoder;


    public NettyMessageDecoder(int maxFrameLength, int lengthFieldOffset, int lengthFieldLength) throws IOException {
        super(maxFrameLength, lengthFieldOffset, lengthFieldLength);
        marshallingDecoder = new MarshallingDecoder();
    }

    @Override
    protected Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
        ByteBuf frame = (ByteBuf) super.decode(ctx, in);
        if (frame == null) {
            return null;
        }

        NettyMessage message = new NettyMessage();
        Header header = new Header();
        header.setCrcCode(frame.readInt());
        header.setLength(frame.readInt());
        header.setSessionID(frame.readLong());
        header.setType(frame.readByte());
        header.setPriority(frame.readByte());

        int size = frame.readInt();
        if (size > 0) {
            Map<String, Object> attch = new HashMap<String, Object>(size);
            int keySize = 0;
            byte[] keyArray = null;
            String key = null;
            for (int i = 0; i < size; i++) {
                keySize = frame.readInt();
                keyArray = new byte[keySize];
                frame.readBytes(keyArray);
                key = new String(keyArray, "UTF-8");
                attch.put(key, marshallingDecoder.decode(frame));
            }
            keyArray = null;
            key = null;
            header.setAttachment(attch);
        }
        if (frame.readableBytes() > 4) {
            message.setBody(marshallingDecoder.decode(frame));
        }
        message.setHeader(header);
        return message;
    }
}

```

```java
package PrivateProtocol.codec;

import io.netty.buffer.ByteBuf;
import org.jboss.marshalling.ByteInput;
import org.jboss.marshalling.Unmarshaller;

import java.io.IOException;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/23 15:47
 */


public class MarshallingDecoder {

    private final Unmarshaller unmarshaller;

    public MarshallingDecoder() throws IOException {
        unmarshaller = MarshallingCodecFactory.buildUnMarshalling();
    }

    protected Object decode(ByteBuf in) throws Exception {
        int objectSize = in.readInt();
        ByteBuf buf = in.slice(in.readerIndex(), objectSize);
        ByteInput input = new ChannelBufferByteInput(buf);
        try {
            unmarshaller.start(input);
            Object obj = unmarshaller.readObject();
            unmarshaller.finish();
            in.readerIndex(in.readerIndex() + objectSize);
            return obj;
        } finally {
            unmarshaller.close();
        }
    }
}

```

### 12.3.3 握手和安全验证

```java
package PrivateProtocol.client;

import PrivateProtocol.struct.Header;
import PrivateProtocol.struct.NettyMessage;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import java.awt.*;

import static PrivateProtocol.common.MessageType.LOGIN_REQ;
import static PrivateProtocol.common.MessageType.LOGIN_RESP;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/23 16:08
 */


public class LoginAuthReqHandler extends ChannelHandlerAdapter {

    private static final Log LOG = LogFactory.getLog(LoginAuthReqHandler.class);

    //TCP连接三次握手成功之后又客户端构造握手请求消息发送给服务端，由于采用白名单认证机制，不需要携带消息体。
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(buildLoginReq());
    }

    //对我收应答消息进行处理
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

        NettyMessage message = (NettyMessage) msg;

        if (message.getHeader() != null && message.getHeader().getType() == LOGIN_RESP.value()) {
            byte loginResult = (byte) message.getBody();
            //非0则认证失败，关闭链路
            if (loginResult != (byte) 0) {
                ctx.close();
            } else {
                LOG.info("Login is ok：" + message);
                ctx.fireChannelRead(msg);
            }
        } else {
            //不是握手应答消息，传递给后面的ChannelHandler处理
            ctx.fireChannelRead(msg);
        }

    }

    private NettyMessage buildLoginReq() {
        NettyMessage message = new NettyMessage();
        Header header = new Header();
        header.setType(LOGIN_REQ.value());
        message.setHeader(header);
        return message;
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.fireExceptionCaught(cause);
    }
}

```

```java
package PrivateProtocol.server;

import PrivateProtocol.struct.Header;
import PrivateProtocol.struct.NettyMessage;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.net.UnknownHostException;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

import static PrivateProtocol.common.MessageType.LOGIN_REQ;
import static PrivateProtocol.common.MessageType.LOGIN_RESP;


/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/23 15:56
 */


public class LoginAuthRespHandler extends ChannelHandlerAdapter {

    public final static Log LOG = LogFactory.getLog(LoginAuthRespHandler.class);

    //重复登录保护
    private Map<String, Boolean> nodeCheck = new ConcurrentHashMap<String, Boolean>();

    //白名单
    private String[] whitekList = {"127.0.0.1", InetAddress.getLocalHost().getHostAddress()};

    public LoginAuthRespHandler() throws UnknownHostException {
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        NettyMessage message = (NettyMessage) msg;

        if (message.getHeader() != null && message.getHeader().getType() == LOGIN_REQ.value()) {
            String nodeIndex = ctx.channel().remoteAddress().toString();
            NettyMessage loginResp = null;

            if (nodeCheck.containsKey(nodeIndex)) {
                loginResp = buildResponse((byte)-1);
            } else {
                InetSocketAddress address = (InetSocketAddress) ctx.channel().remoteAddress();
                String ip = address.getAddress().getHostAddress();
                boolean isOK = false;
                for (String WIP : whitekList) {
                    if (WIP.equals(ip)) {
                        isOK = true;
                        break;
                    }
                }
                loginResp = isOK ? buildResponse((byte)0) : buildResponse((byte)-1);
                if (isOK) {
                    nodeCheck.put(nodeIndex, true);
                }
                LOG.info("The login response is : " + loginResp
                        + " body [" + loginResp.getBody() + "]");
                ctx.writeAndFlush(loginResp);
            }
        } else {
            ctx.fireChannelRead(msg);
        }
    }

    private NettyMessage buildResponse(byte result) {
        NettyMessage message = new NettyMessage();
        Header header = new Header();
        header.setType(LOGIN_RESP.value());
        message.setHeader(header);
        message.setBody(result);
        return message;
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        //出现异常时需要去注册，保证后续客户端可以重连成功
        nodeCheck.remove(ctx.channel().remoteAddress().toString());
        ctx.close();
        ctx.fireExceptionCaught(cause);
    }
}

```

### 12.3.4 心跳检测机制

```java
package PrivateProtocol.client;

import PrivateProtocol.struct.Header;
import PrivateProtocol.struct.NettyMessage;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import java.util.concurrent.ScheduledFuture;
import java.util.concurrent.TimeUnit;

import static PrivateProtocol.common.MessageType.*;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/26 10:36
 */


public class HeartBeatReqHandler extends ChannelHandlerAdapter {

    private static final Log LOG = LogFactory.getLog(HeartBeatReqHandler.class);

    private volatile ScheduledFuture<?> heartBeat;

    //握手成功后启动无限循环定时器用于定期发送心跳消息
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        NettyMessage message = (NettyMessage) msg;
        if (message.getHeader() != null && message.getHeader().getType() == LOGIN_RESP.value()) {
            //每5秒发送一条心跳消息
            heartBeat = ctx.executor().scheduleAtFixedRate(
                    new HeartBeatTask(ctx), 0, 5000, TimeUnit.MILLISECONDS
            );
            //处理服务端发送的心跳应答消息
        } else if (message.getHeader() != null && message.getHeader().getType() == HEARTBEAT_RESP.value()) {
            LOG.info("Client receive server heart beat message : ---> " + message);
        } else {
            ctx.fireChannelRead(msg);
        }
    }

    private class HeartBeatTask implements Runnable {

        private final ChannelHandlerContext ctx;

        private HeartBeatTask(ChannelHandlerContext ctx) {
            this.ctx = ctx;
        }

        @Override
        public void run() {
            NettyMessage heartBeat = buildHeartBeat();
            LOG.info("Client send heart beat messsage to server : ---> " + heartBeat);
            ctx.writeAndFlush(heartBeat);
        }

        private NettyMessage buildHeartBeat() {
            NettyMessage message = new NettyMessage();
            Header header = new Header();
            header.setType(HEARTBEAT_REQ.value());
            message.setHeader(header);
            return message;
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        if (heartBeat != null) {
            heartBeat.cancel(true);
            heartBeat = null;
        }

        ctx.fireExceptionCaught(cause);
    }
}

```

```java
package PrivateProtocol.server;

import PrivateProtocol.struct.Header;
import PrivateProtocol.struct.NettyMessage;
import io.netty.channel.ChannelHandlerAppender;
import io.netty.channel.ChannelHandlerContext;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import static PrivateProtocol.common.MessageType.HEARTBEAT_REQ;
import static PrivateProtocol.common.MessageType.HEARTBEAT_RESP;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/26 10:55
 */


public class HeartBeatRespHandler extends ChannelHandlerAppender {
    private static final Log LOG = LogFactory.getLog(HeartBeatRespHandler.class);

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        NettyMessage message = (NettyMessage) msg;
        if (message.getHeader() != null && message.getHeader().getType() == HEARTBEAT_REQ.value()) {
            LOG.info("Receive client heart beat message : ---> " + message);
            NettyMessage heartBeat = buildHeatBeat();
            LOG.info("Send heart beat response message to client : ---> " + heartBeat);
            ctx.writeAndFlush(heartBeat);
        } else {
            ctx.fireChannelRead(msg);
        }
    }

    private NettyMessage buildHeatBeat() {
        NettyMessage message = new NettyMessage();
        Header header = new Header();
        header.setType(HEARTBEAT_RESP.value());
        message.setHeader(header);
        return message;
    }
}

```

### 12.3.5 断线重连

```java
package PrivateProtocol.client;

import PrivateProtocol.codec.NettyMessageDecoder;
import PrivateProtocol.codec.NettyMessageEncoder;
import PrivateProtocol.common.NettyConstant;
import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;

import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.timeout.ReadTimeoutHandler;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import java.net.InetSocketAddress;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

import static PrivateProtocol.common.NettyConstant.*;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/26 11:04
 */


public class NettyClient {

    private static final Log LOG = LogFactory.getLog(NettyClient.class);

    private ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);

    EventLoopGroup group = new NioEventLoopGroup();

    public void connect(int port, String host) throws Exception {

        try {
            Bootstrap b = new Bootstrap();

            b.group(group)
                    .channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            //消息解码，并设置了最大长度上限
                            ch.pipeline().addLast(new NettyMessageDecoder(1024 * 1024, 4, 4));
                            //消息编码
                            ch.pipeline().addLast("MessageEncoder", new NettyMessageEncoder());
                            //读超时
                            ch.pipeline().addLast("readTimeoutHandler", new ReadTimeoutHandler(50));
                            //握手请求
                            ch.pipeline().addLast("LoginAuthHandler", new LoginAuthReqHandler());
                            //心跳消息
                            ch.pipeline().addLast("HeartBeatHandler", new HeartBeatReqHandler());

                            //类似于AOP但比AOP性能更高。
                        }
                    });


            //ChannelFuture f = b.connect(new InetSocketAddress(host, port),
                    //new InetSocketAddress(LOCALIP, LOCAL_PORT)).sync();
            ChannelFuture f = b.connect("127.0.0.1", 8080).sync();
            f.channel().closeFuture().sync();
        } finally {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        TimeUnit.SECONDS.sleep(1);
                        try {
                            connect(PORT, REMOTEIP);//发起重连操作
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    }

    public static void main(String[] args) throws Exception {
        new NettyClient().connect(PORT, REMOTEIP);
    }
}

```

```java
package PrivateProtocol.server;

import PrivateProtocol.codec.NettyMessageDecoder;
import PrivateProtocol.codec.NettyMessageEncoder;
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.logging.LogLevel;
import io.netty.handler.logging.LoggingHandler;
import io.netty.handler.timeout.ReadTimeoutHandler;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

import static PrivateProtocol.common.NettyConstant.PORT;
import static PrivateProtocol.common.NettyConstant.REMOTEIP;

/**
 * <p>Description:  xx</p>
 *
 * @author 李宏博
 * @version 1.0
 * @create 2019/8/26 11:23
 */


public class NettyServer {

    private static final Log LOG = LogFactory.getLog(NettyServer.class);

    public void bind() throws Exception {

        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        ServerBootstrap b = new ServerBootstrap();

        b.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, 100)
                .handler(new LoggingHandler(LogLevel.INFO))
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new NettyMessageDecoder(1024 * 1024, 4, 4));
                        ch.pipeline().addLast(new NettyMessageEncoder());
                        ch.pipeline().addLast(new ReadTimeoutHandler(50));
                        ch.pipeline().addLast(new LoginAuthRespHandler());
                        ch.pipeline().addLast(new HeartBeatRespHandler());
                    }
                });
        b.bind(REMOTEIP, PORT).sync();
        LOG.info("Netty server start ok : "
                + (REMOTEIP + " : " + PORT));
    }

    public static void main(String[] args) throws Exception {
        new NettyServer().bind();
    }

}
```

## 12.5 总结

协议栈还有很多缺陷，比如断线后，发送的消息保存在哪里？

# 之后的章节

源码就不记录了，详细看这个https://www.javadoop.com/post/netty-part-1