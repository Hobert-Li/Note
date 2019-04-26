> 前面的被系统吞了 之后再补上

# 第8章 虚拟机字节码执行系统

## 8.1概述

## 8.2 运行时栈帧结构

### 8.2.1 局部变量表

是一组变量存储空间，用于存放方法参数和方法内部定义的局部变量。

用于存放方法参数和方法内部定义的局部变量。

容量以变量槽（Variable Slot）为最小单位，虚拟机规范中并没有明确指明一个Slot应占用的内存空间大小，知识很有导向性地说道每个Slot都应该能存放一个boolean、byte、char、short、int、float、reference或returnAddress类型的数据。

只要保证计时在64位虚拟机中使用了64位的物理内存空间去实现一个Slot，虚拟机仍要使用对齐和不败的手段让Slot在外观上看起来与32位虚拟机中的一致。

Java中占用32位以内的数据类型有boolean、byte、char、short、int、float、reference和returnAddress8种类型。（Java语言与Java虚拟机种的剧本数据类型是存在本质差别的）。reference类型表示对一个对象实例的引用。虚拟机规范没有指明长度和结构。但需要做到如下两点：

1. 