# 第2章 Tomcat总体架构

系统设计及中间件设计时的参考：生命周期管理、可扩展的容器组件设计、类加载方式。

## 2.1 总体设计

如何设计一个应用服务器？

### 2.1.1 Server

最基本的功能：接收请求，业务处理，返回响应。

两个方法：

- start()：启动服务器，打开Socket链接，监听端口，负责接收请求，处理及返回。
- stop()：停止服务器并释放网络资源。

作为嵌入在应用系统中的远程请求处理方案，且访问量低时可行。但作为应用服务器不可行。

### 2.1.2 Connector和Container

请求监听与请求处理放到一起扩展性差。

Connector负责监听，返回。

Container负责处理请求。

均分别拥有自己的start()和stop()方法来加载和释放自己维护的资源。

明显的缺陷：如何让Connector与Container对应？可以维护一个复杂关系映射，但是并不必需。Container设计足够灵活。

引入Service，负责维护多个Connector和一个Container。

在Tomcat中，Container是一个更加通用的概念。为了与Tomcat中的组件命名一致，所以重新命名为Engine，用以表示整个Servlet引擎。

Engine表示整个Servlet引擎。Server表示整个Servlet容器。

### 2.1.3 Container设计

应用服务器是用来部署并运行Web应用的，是一个运行环境，而不是独立的业务处理系统。因此需要在Engine容器中支持管理Web应用，当接收到Connector的处理请求时，Engine容器能够找到一个合适的Web应用来处理。

使用一个Context来表示一个Web应用，并且一个Engine可以包含多个Context。

虚拟主机，加入Host。一个Host可以包含多个Context。

Tomcat的设计中Engine可以包含Host也可以包含Context，这是由具体的Engine实现确定的。Tomcat提供的默认实现StandardEngine只能包含Host。

一个Web应用可以包含多个Servlet实例。在Tomcat中，Servlet定义被称为Wrapper。

“容器”的作用都是处理请求并返回响应数据。所以引入一个Container接口：addchild()添加子容器，backgroundProcess()实现文件变更的扫描。

### 2.1.4 Lifecycle

所有组件均存在启动、停止这两个生命周期方法，可在此基础上扩展生命周期管理的方法，即对于生命周期管理进行一次接口抽象。

将Server接口替换为Lifecycle接口：

- Init()：初始化组件
- start()：启动组件
- stop()：停止组件
- destory()：销毁组件
- addLifecycleListener：添加事件监听器（用于监听组件的状态变化）
- removeLifecycleListener：删除

Tomcat核心组件的默认实现均继承自LifecycleBeanBase抽象类，该类不但负责组件各个状态的转换和事件处理，还将组件自身注册为MBean，以便通过Tomcat的管理工具进行动态维护。

### 2.1.5 Pipeline和Valve

以上设计以保证核心架构的了可伸缩性和可扩展性。但是还要考虑各个组件的灵活性，使其同样可扩展。

责任链模式是一种比较好的选择。Tomcat即采用该模式来实现客户端请求的处理。在Tomcat中每个Container组件通过执行一个责任链来完成具体的请求处理。

Pipeline（管道）用于构造责任链，Valve（阀）代表责任链上的每个处理器。Pipeline中维护了一个基础的Valve（位于末端，最后执行）。

Tomcat的每个层级的容器（Engine、Host、Context、Wrapper）均有对应的基础Valve实现，同时维护一个Pipeline实例。即任何层级的容器都可以对请求处理进行可扩展。

### 2.1.6 Connector设计

基本功能：

- 监听服务器端口，读取来自客户端的请求。
- 将请求数据按照指定协议进行解析。
- 根据请求地址匹配正确的容器进行处理。
- 将响应返回客户端。

Tomcat支持多协议，默认支持HTTP和AJP。同时支持多种I/O方式，包括BIO（8.5之后移除）、NIO、APR。而且在Tomcat8之后新增了对NIO2和HTTP/2协议的支持。因此对协议和I/O进行抽象和建模时需要关注的重点。

在Tomcat中，ProtocolHandler表示一个协议处理器，其包含一个Endpoint（无此接口，仅有AbstractEndpoint抽象类）用于启动Socket监听，还包含一个Processor用于按照指定协议读取数据，并将请求交由容器处理。

在Connector启动时，Endpoint会启动线程来监听，并在接收到请求后调用Processor进行数据读取。

当Processor读取客户端请求后，需要按照请求地址映射到具体的容器进行处理，这个过程即为请求映射。由于Tomcat各个组件采用通用的生命周期管理，而且可以通过管理工具进行状态变更，因此请求映射除考虑映射规则的实现外，还要考虑容器组件的注册与销毁。

Tomcat通过Mapper和MapperListener两个类实现上述功能。前者用于维护容器映射信息，同时按照映射规则（Servlet规范）查找容器。

