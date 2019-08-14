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

