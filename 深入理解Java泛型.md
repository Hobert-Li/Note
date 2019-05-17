## 什么是泛型？有什么好处？

> 泛型是JavaSE1.5的新特性，泛型的本质是**参数化类型**，也就是所说的数据类型被指定为一个参数。

好处：
> 1. 类型安全，提供编译期间的类型检测
> 2. 前后兼容
> 3. 泛化代码，代码可以更多的重复利用
> 4. 性能高，用泛型Java编写的代码可以为Java编译器和虚拟机带来更多的类型信息，这些信息对Java程序做进一步优化提供条件
>
> 

## 为什么引入泛型

在引入泛型之前，要想实现一个通用的、可处理不同类型的方法，你需要使用Object作为属性和方法参数，比如这样：

```java
public class Generic {
    private Object[] mData;
    
    public Generic(int capacity) {
        mData = new Object[capaity];
    }
    
    public Object getData(int index) {
        return mData[index];
    }
    
    public void add(int index,Object item) {
        mData[index] = item;
    }
}
```

它使用一个Object数组来保存数据，这样在使用时可以添加不同类型的对象：

```java
Genaric generic = new Generic(10);
generic.add(0,"shixin");
generic.add(1,23);
```

然而由于Object是所有类的父类，所有的类都可以作为成员被添加到上述类种；当需要使用的时候，必须进行强制转换，而且这个强转很有可能出现转换异常：

```java
String item1 = (String)generic.getData(0);
String item2 = (String)generic.getData(1);
```

上面第二行代码将一个Integer强转成String，运行时会报错。

可得，用Object实现通用、不同类型的处理，有这么两个缺点：

1. 每次使用时需要强制转换成想要的类型
2. 在编译时编译器并不知道类型转换是否成功，运行时才知道，不安全

根据《Java编程思想》中的描述，泛型出现的最引人注意的一个原因，就是为了创建容器类。

**引入泛型的主要目标：**

- 类型安全
    - 泛型的主要目标是提高Java程序的类型安全
    - 编译时期就可以检查出因Java类型不正确导致的ClassCastException
    - 符合越早出错代价越小原则

- 消除强制类型转换
    - 泛型的一个附带好处是，使用时直接得到目标类型，消除很多强制类型转换
    - 所得即所需，使代码更加可读，并且减少了出错机会

- 潜在的性能收益
    - 由于泛型的实现方式，支持泛型（几乎）不需要JVM或类文件更改
    - 所有工作都在编译器中完成
    - 编译器生成的代码跟不适用泛型（和强制类型转换）时所写的代码几乎一直，只是更能确保类型安全而已







