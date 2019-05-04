# Session与Cookie的区别：

## 存储方式：

Cookie只能存储字符串，如果要存储非ASCII字符串还要对其编码；

Session可以存储任何类型的数据，可以把Session看成是一个容器。

## 隐私安全：

Cookie存储在浏览器中，对客户端时可见的。信息容易泄露出去。如果使用Cookie，最好将Cookie加密。

Session村粗在服务器上，对客户端时透明的。不存在敏感信息泄露问题。

## 有效期：

Cookie保存在硬盘中，只需要设置maxAge属性为比较大的正整数，计时关闭浏览器，Cookie还是存在的；

Session保存在服务器中，设置maxInactiveInterval属性值来确定Session的有效期。并且Session依赖于名为JSESSIONID的Cookie，该Cookie默认的maxAge属性为-1.如果关闭了浏览器，该Session虽然没有从服务器中消亡，但也就失效了。

## 对服务器的负担：

Session是保存在服务器的，每个用户都回产生一个Session，如果是并访问的用户非常多，是不能用Session的，Session回消耗大量的内存；

Cookie是保存在客户端的。不占用服务器的资源。所以大型网站一般使用Cookie进行会话跟踪。

## 浏览器的支持：

如果浏览器禁用了Cookie那么Cookie是无用的；

如果禁用了Cookie，Session可以通过URL地址重写来进行会话跟踪。

## 跨域名：

Cookie可以设置domain属性来实现跨域名；

Session只在当前的域名内有效，不可以跨域名。

# Servlet安全性问题

由于Servlet是单例得，当多个用户访问Servlet的时候，服务器回为每个用户创建一个线程。当多个用户并发访问Servlet共享资源的时候就会出现线程安全问题。

原则：

1. 如果一个变量需要多个用户共享，则应当在访问该变量的时候，加同步机制synchronized（对象）{ }
2. 如果一个变量不需要共享，则直接在doGet()或者doPost()定义。这样就不会存在线程安全问题。

