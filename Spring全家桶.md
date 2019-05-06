# 七大模块

## Spring Core

Core是框架的最基础部分，提供IOC和依赖注入特性。基本概念BeanFactory，提供对Factory模式的经典实现来消除对程序性单例模式的需要，并真正地允许你从程序逻辑中分离出以来关系和配置。

## Spring Context

构建与Core封装包基础上的Context封装包，提供了一种框架是的对象访问方法，有些像JNDI注册器。

## Spring DAO

Data Access Object提供了JDBC的抽象层，它可消除冗长的JDBC编码和解析书库厂商特有的错误代码。并且，JDBC封装包封装包还提供了一种比编程性更好的声明性事务管理方法，不仅仅是实现了特定接口，而且对所有的POJOs(plain oil Java objects)都适用。

## Spring ORM

ORM封装包提供了常用的“对象/关系”映射APIs的集成层”。其中包括了JPA、JDO、Hibernate和iBatis。里ORM封装包，可以混合使用所有SPring提供的特性进行“对象/关系”映射，如前边提到的简单声明性事务管理。

## Spring AOP

Spring的AOP封装包提供了符合AOP Alliance规范的面向方面的编程实现，让你可以定义例如方法拦截器（method-intercptors）和切点（pointcuts）。

## Spring Web

提供了基础的针对Web开发的集成特性，例如多方文件上传，利用Servlet listerns进行IOC容器初始化和针对Web的ApplicationContext。

## Spring Web MVC

Spring中的MVC封装包提供了Web应用的MVC实现。Spring的MVC框架并不是仅仅提供一种传统的实现，它提供了一种清晰的分离模型，在领域模型代码和WebFrom之间。并且，还可以借助Spring框架的其他特性。

# Spring（AOP模块）

## cglib代理

**静态代理**需要实现目标对象的相同接口

**动态代理**需要目标对象一定要有接口

**cglib代理**也叫子类代理，从内存中构建出一个子类来扩展目标对象的功能。

- CGLIB是一个强大的高性能的代码生成包，它可以在运行期扩展Java类与实现Java接口。

## 编写cglib代理

- 需要引入cglib-jar文件，但是spring的核心包中已经包括了cglib功能，所以直接引入spring-core包即可；
- 代理的类不能为final（需要动态构建子类）；
- 目标对象的方法如果为final/static，那么久不会被拦截，即不会执行目标对象额外的业务方法。

```java
//需要实现MethodInterceptor接口
public class ProxyFactory implements MethodInterceptor{
    
    // 维护目标对象
    private Object target;
    public ProxyFactory(Object target){
        this.target = target;
    }
    
    // 给目标对象创建代理对象
    public Object getProxyInstance(){
        //1. 工具类
        Enhancer en = new Enhancer();
        //2. 设置父类
        en.setSuperclass(target.getClass());
        //3. 设置回调函数
        en.setCallback(this);
        //4. 创建子类(代理对象)
        return en.create();
    }
    

    @Override
    public Object intercept(Object obj, Method method, Object[] args,
            MethodProxy proxy) throws Throwable {
        
        System.out.println("开始事务.....");
        
        // 执行目标对象的方法
        Object returnValue = method.invoke(target, args);
        
        System.out.println("提交事务.....");
        
        return returnValue;
    }

}
```

# Spring事务管理

## Spring事务管理接口

- PlatformTransactionManager：平台事务管理器
- TransactionDefinition：事务定义信息（事务隔离级别、传播行为、超市、只读、回滚规则）
- TransactionStatus：事务运行状态

**事务管理：按照给定的事务规则来执行提交或者回滚操作。**

### PlatformTransactionManager接口

**String并不直接管理事务，而是提供了多种事务管理器。**Spring事务管理器的接口是： **org.springframework.transaction.PlatformTransactionManager** ，通过这个接口，Spring为各个平台如JDBC、Hibernate等都提供了对应的事务管理器，但是具体的实现就是各个平台自己的事情了。

### TransactionDefinition接口

事务管理器接口**PlatformTransactionManager**通过**getTeansaction(TransactionDefinition definition)**方法来得到一个事务，这个方法里面的参数是TransactionDefinition类，这个类定义了一些基本的事务属性。

事务属性：

>隔离级别
>
>传播行为
>
>回滚规则
>
>是否只读
>
>事务超时

#### 事务隔离级别（定义了一个事务可能受其他并发事务影响的程度）：

