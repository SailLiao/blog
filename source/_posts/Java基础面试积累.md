---
title: Java基础面试积累
date: 2020-12-24 17:09:58
tags: 面试
cover: https://sailliao.oss-cn-beijing.aliyuncs.com/img/wallhaven-z86v5w.jpg
---

## new一个Object对象占用多少内存？
这里指的是空的对象，例如 
```java
Object obj = new Object()
```
所以是一个引用对象，引用的长度决定了Java的寻址能力，32位的JKD是4字节，64位的JDK是8字节
因为obj对象没有任何数据（field），会在堆上为它分配空间吗？如果分配空间，里面存储了什么内容？

以面向对象的思维来分析，对象封装了数据和行为，是一个统一的整体，虽然obj对象没有数据，但是有行为（Object类定义了12个方法）。
当我们执行完new操作后，obj的值为堆内存的地址，既然obj都指向一块内存了，说明是会在堆上为其分配空间的。

JDK是64位，8字节是引用，16字节是堆内存，总共是 8+16=24字节，所以new一个Object对象占用24字节。
如果JDK是32位，则new一个Object对象占用4+16=20字节，

**结论: 32 位机器 4+16=20字节; 64 位机器 8+16=24字节**


## 方法内联

之所以出现方法内联是因为函数调用除了执行自身逻辑的开销外，还有一些不为人知的额外开销。
这部分额外的开销主要来自方法栈帧的生成、参数字段的压入、栈帧的弹出、还有指令执行地址的跳转。

java中方法调用嵌套过多或者方法过多，这种额外的开销就越多。

很可能自身执行逻辑的开销还比不上为了调用这个方法的额外开锁。
如果类似的方法被频繁的调用，则真正相对执行效率就会很低，虽然这类方法的执行时间很短。
这也是为什么jvm会在热点代码中执行方法内联的原因，这样的话就可以省去调用调用函数带来的额外开支。

```java
public int  add(int a, int b , int c, int d){
        return add(a, b) + add(c, d);
}

public int add(int a, int b){
    return a + b;
}
```

内联之后

```java
public int  add(int a, int b , int c, int d){
        return a + b + c + d;
}
```

一个方法如果满足以下条件就很可能被jvm内联。
1. 热点代码。 如果一个方法的执行频率很高就表示优化的潜在价值就越大。那代码执行多少次才能确定为热点代码？这是根据编译器的编译模式来决定的。如果是客户端编译模式则次数是1500，服务端编译模式是10000。次数的大小可以通过-XX:CompileThreshold来调整。
2. 方法体不能太大，jvm中被内联的方法会编译成机器码放在code cache中。如果方法体太大，则能缓存热点方法就少，反而会影响性能。
3. 如果希望方法被内联，尽量用private、static、final修饰，这样jvm可以直接内联。如果是public、protected修饰方法jvm则需要进行类型判断，因为这些方法可以被子类继承和覆盖，jvm需要判断内联究竟内联是父类还是其中某个子类的方法。


## Java8 新特性

### Lambda 表达式

```java
new Thread(()->{
    // do something
}).start();
```

### 方法引用
在学习lambda表达式之后，我们通常使用lambda表达式来创建匿名方法。然而，有时候我们仅仅是调用了一个已存在的方法。如下：
```java
Arrays.sort(stringsArray,(s1,s2)->s1.compareToIgnoreCase(s2));
```
在Java8中，我们可以直接通过方法引用来简写lambda表达式中已经存在的方法。
```java
Arrays.sort(stringsArray, String::compareToIgnoreCase);
```
其中方法引用的操作符是双冒号"::"。

### 默认方法 default

这是Java语言的一个新特性，现在接口类里可以包含方法体（这就是 default 方法）了。这些方法会隐式的添加到实现这个接口的每个子类中。

### 新工具
Nashorn, JavaScript 引擎 

### Stream API
### Date Time API
```java
Date.toInstant()
```
该方法返回一个Instant，表示与此时间在时间​​轴上的同一点。

### Optional 类
一种更优雅的方式处理空指针

![](1.png)

例如

```java
String get = Optional.ofNullable("a").orElse("b");
```
