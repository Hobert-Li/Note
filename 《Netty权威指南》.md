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

