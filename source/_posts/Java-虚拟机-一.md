---
title: Java 虚拟机 一
date: 2021-08-04 11:35:13
tags: 虚拟机
cover: https://sailliao.oss-cn-beijing.aliyuncs.com/img/wallhaven-72kdvo.jpg
---

# 运行时数据区

![](1.png)

我们先看线程隔离的数据区

## 程序计数器

程序计数器（ Program Counter Register） 是一块较小的内存空间， 它可以看作是当前线程所执行的字节码的**行号指示器**。

字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令， 分支、 循环、 跳转、 异常处理、 线程恢复等基础功能都需要依赖这个计数器来完成。

因为java的多线程是依赖CPU时间片的，所以每个线程都需要一个东西来指出当前安现场执行到哪一步了。所以程序计数器是**线程私有**的。

记录器记录的是 虚拟机正在执行的指令的地址，如果是 native 方法，则这个计数器是空，**这个区域是为一个没有 OOM 的地方**

## 虚拟机栈

Java虚拟机栈（ Java Virtual Machine Stacks） 也是线程私有的。
虚拟机栈描述的是Java方法执行的内存模型： 
* 每个方法在执行的同时都会创建一个栈帧（ Stack Frame[1]） 用于存储
1. 局部变量表
2. 操作数栈
3. 动态链接
4. 方法出口
每一个方法从调用直至执行完成的过程， 就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。

* 局部变量表

局部变量表存放了编译期可知的各种**基本数据类型**（boolean、 byte、 char、 short、 int、float、 long、 double）
**对象引用**（reference类型， 它不等同于对象本身， 可能是一个指向对象起始地址的引用指针， 也可能是指向一个代表对象的句柄或其他与此对象相关的位置）、returnAddress类型（指向了一条字节码指令的地址）。

局部变量表的最小单位是**变量槽（Slot）**，其中64位长度的long和double类型的数据会占用2个局部变量空间（Slot） ， 其余的数据类型只占用1个。 

局部变量表所需的内存空间在**编译期间完成分配**， 当进入一个方法时， 这个方法需要在帧中分配多大的局部变量空间是完全确定的， 在方法运行期间不会改变局部变量表的大小。

这个区域有两种情况会溢出

1. StackOverflowError

如果线程请求的栈深度大于虚拟机所允许的深度时候抛出

2. OutOfMemoryError

如果虚拟机栈可以动态扩展（ 当前大部分的Java虚拟机都可动态扩展， 只不过Java虚拟机规范中也允许固定长度的虚拟机栈） ， 如果扩展时无法申请到足够的内存

## 本地方法栈

本地方法栈则为虚拟机使用到的 **native** 方法服务, 与虚拟机栈一样， 本地方法栈区域也会抛出StackOverflowError和OutOfMemoryError异常。

## Java 堆

java 堆有这么几个特点

1. 虚拟机所管理的内存中最大的一块
2. 被所有线程共享的一块内存区域
3. 在虚拟机启动时创
4. 此内存区域的唯一目的就是**存放对象实例**， 几乎所有的对象实例都在这里分配内存[注1]
5. 堆无法再拓展的时候会抛出 OutOfMemoryError 异常

[注]1：着JIT编译器的发展与逃逸分析技术逐渐成熟， 栈上分配、 标量替换优化技术将会导致一些微妙的变化发生， 所有的对象都分配在堆上也渐渐变得不是那么“绝对”了。

### JIT编译器

JIT编译（just-in-time compilation）狭义来说是**当某段代码即将第一次被执行时进行编译，因而叫“即时编译”**。
JIT编译是动态编译的一种特例。JIT编译一词后来被泛化，时常与动态编译等价；但要注意广义与狭义的JIT编译所指的区别。

### 逃逸分析

如果方法内部定义的变量，只在方法内部使用，那么对象就可以在栈上分配，而不用跑去堆上分配。

方法栈上的对象在方法执行完之后，栈桢弹出，对象就会自动回收。这样的话就不需要等内存满时再触发内存回收。这样的好处是程序内存回收效率高，并且GC频率也会减少，程序的性能就提高了。

Java的逃逸分析是方法级别的，因为JIT的即时编译是方法级别。

另外，如果发现某个对象只能从一个线程可访问，那么在这个对象上的操作可以不需要同步。也就是 **锁消除**

### 标量替换

简单地说，就是用标量(基础数据类型)替换聚合量(引用类型，例如一个实体类，里面有很多字段，实体类就是标量的聚合体)。
这样做的好处是如果创建的对象并未用到其中的全部变量，则可以节省一定的内存。对于代码执行而言，无需去找对象的引用，也会更快一些。

1. 标量是指不可分割的量，如java中基本数据类型和reference类型，相对的一个数据可以继续分解，称为聚合量；
2. 如果把一个对象拆散，将其成员变量恢复到基本类型来访问就叫做标量替换；
3. 如果逃逸分析发现一个对象不会被外部访问，并且该对象可以被拆散，那么经过优化之后，并不直接生成该对象，而是在栈上创建若干个成员变量；

通过 **-XX:+EliminateAllocations**可以开启标量替换， **-XX:+PrintEliminateAllocations**查看标量替换情况。

```java
Point p = new Point(1, 2);
System.out.println("point x " +p.x + " y " + p.y);
```
如果后续代码P没有使用，经过编译后代码会变成这样
```java
int x = 1;
int y = 2;
System.out.println("point x " +x + " y " +y);
```
是不是一下就理解了

## 方法区

各个线程共享的内存区域。它用于存储已被虚拟机加载的**类信息**、 **常量**、 **静态变量**、 **即时编译器编译后的代码**等数据。
多人都更愿意把方法区称为“永久代”（ Permanent Generation） , 仅仅是因为 HotSpot 虚拟机的设计团队选择把GC分代收集扩展至方法区， 或者说使用永久代来实现方法区而已。其他虚拟机有不同的实现，可能不叫永久代。

会 OOM

## 运行时常量池

Class文件中有一项信息是常量池（ Constant Pool Table） ， 用于存放**编译期生成的各种字面量和符号引用**， 这部分内容将在类加载后进入方法区的运行时常量池中存放。

会 OOM

## 直接内存

直接内存（ Direct Memory） 并不是虚拟机运行时数据区的一部分， 也不是Java虚拟机规范中定义的内存区域。 但是这部分内存也被频繁地使用， 而且也可能导致OutOfMemoryError异常出现。

在JDK 1.4中新加入了NIO（ New Input/Output） 类， 引入了一种基于通道（ Channel） 与缓冲区（ Buffer） 的I/O方式， 它可以使用Native函数库直接分配堆外内存， 然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。 这样能在一些场景中显著提高性能， 因为避免了在Java堆和Native堆中来回复制数据。

动态扩展时出现 OOM

# 对象

探讨 HotSpot 虚拟机在Java堆中对象分配、 布局和访问的全过程。

## 对象的创建

写代码的时候最简单的创建对象的方式是 new，那么 new 做了什么呢

* 虚拟机遇到一条new指令时， 首先将去检查这个指令的参数是否能在常量池中定位到一个类的**符号引用**， 并且检查这个符号引用代表的类是否已被加载、 解析和初始化过。 如果没有， 那必须先执行相应的类加载过程。
* 接下来虚拟机将为新生对象分配内存。 对象所需内存的大小在类加载完成后便可完全确定。

> Q: 一个空对象 new Object() 占多少字节
> A: 32 位机器 4+16=20字节; 64 位机器 8+16=24字节

### 内存的分配方式

1. 指针碰撞

假设Java堆中内存是绝对规整的， 所有用过的内存都放在一边， 空闲的内存放在另一边， 中间放着一个指针作为分界点的指示器， 那所分配内存就仅仅是把那个指针向空闲空间那边挪动一段与对象大小相等的距离， 这种分配方式称为“指针碰撞”（ Bump the Pointer）。

2. 空闲列表

如果Java堆中的内存并不是规整的， 已使用的内存和空闲的内存相互交错， 那就没有办法简单地进行指针碰撞了， 虚拟机就必须维护一个列表， 记录上哪些内存块是可用的， 在分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录， 这种分配方式称为“空闲列表”（ Free List） 。 
选择哪种分配方式由**Java堆是否规整决定**， 而Java堆是否规整又由**所采用的垃圾收集器是否带有压缩整理功能决定**。 因此， 在使用Serial、 ParNew等带Compact过程的收集器时， 系统采用的分配算法是指针碰撞， 而使用CMS这种基于Mark-Sweep算法的收集器时， 通常采用空闲列表

### 内存分配的线程安全

对象的创建时非常频繁的，所以可能有线程安全的问题，有两重方案解决这个问题

1. 对分配内存空间的动作进行同步处理，—实际上虚拟机采用CAS配上失败重试的方式保证更新操作的原子性；
2. 按照线程划分在不同的空间之中进行， 即每个线程在Java堆中预先分配一小块内存， 称为本地线程分配缓冲（ Thread Local Allocation Buffer,TLAB）。哪个线程要分配内存， 就在哪个线程的TLAB上分配， 只有TLAB用完并分配新的TLAB时， 才需要同步锁定。虚拟机是否使用TLAB， 可以通过**-XX： +/-UseTLAB**参数来设定
3. TLAB空间的内存非常小，缺省情况下仅占有整个Eden空间的**1%**，也可以通过选项-XX:TLABWasteTargetPercent设置TLAB空间所占用Eden空间的百分比大小。
4. TLAB的本质其实是三个指针管理的区域：start，top 和 end，每个线程都会从Eden分配一块空间，例如说100KB，作为自己的TLAB，其中 start 和 end 是占位用的，标识出 eden 里被这个 TLAB 所管理的区域，卡住eden里的一块空间不让其它线程来这里分配。

* 缺点

1. TLAB通常很小，所以放不下大对象。
2. TLAB空间还剩一点点没有用到，有点舍不得。(比如100kb的TLAB，装了80KB，又来了个30KB的对象)，这个时候就要申请新的TLAB来装，就浪费了20KB
3. JVM解决手段是设置**最大浪费空间**：当剩余的空间小于最大浪费空间，那该TLAB属于的线程在重新向Eden区申请一个TLAB空间。进行对象创建，还是空间不够，那你这个对象太大了，去Eden区直接创建吧！当剩余的空间大于最大浪费空间，那这个大对象请你直接去Eden区创建，我TLAB放不下没有使用完的空间。

当然，又回造成新的病垢。Eden空间够的时候，你再次申请TLAB没问题，我不够了，Heap的Eden区要开始GC，TLAB允许浪费空间，导致Eden区空间不连续，积少成多。以后还要人帮忙打理。

说回正题，在对对象的内存分配完毕后，开始对 对象 进行设置，例如这个对象是哪个类的实例、 如何才能找到类的元数据信息、 对象的哈希码、 对象的GC分代年龄等信息。 这些信息存放在对象的对象头（ Object Header） 之中

### 对象头

每个对象都有对象头，包含了对象的基本信息，例如

* 布局
* GC状态
* 类型
* 同步状态
* (identity) hash code (不经过重写由JVM计算的hashcode)
* 数组长度 (前提当前对象是数组)

对象头由两部分组成

1. Klass Pointer

```java
Person p = new Person();
// p 在栈上
// 实际对象在 堆
// 在堆里的对象，就可以通过 Klass Pointer 找到 元数据
```

klass pointer一般占32个bit即4个字节，如果你有足够的原因关闭默认的指针压缩，即启动参数加上了-XX:-UseCompressedOops那么它就占64个bit.
不过此处还有一个细节：根据计算，堆大小超过32GB后，就算不关指针压缩并不会报错，只是指针压缩会失效。

klass pointer的存储内容是一个指针，指向了其类元数据的信息，jvm使用该指针来确定此对象是类的哪个实例.



2. Mark Word

![](3.png)

Mark Word在64位虚拟机下，也就是占用64位大小即8个字节的空间.

* unused：未使用的
* hashcode：上文提到的identity hash code，本文出现的hashcode都是指identity hash code
* thread: 偏向锁记录的线程标识
* epoch: 验证偏向锁有效性的时间戳
* age：分代年龄
* biased_lock 偏向锁标志
* lock 锁标志
* pointer_to_lock_record 轻量锁lock record指针
* pointer_to_heavyweight_monitor 重量锁monitor指针

**FQA**

1. 为什么晋升到老年代的年龄设置(XX:MaxTenuringThreshold)不能超过15 ？
因为就给了age四个bit空间，最大就是1111(二进制)也就是15，多了没地方存.

2. 为什么你的synchronized锁住的对象，没有“传说中的”偏向锁优化？
因为hashcode并不是对象实例化完就计算好的，是调用计算出来放在mark word里的。
你调用过hashcode方法(或者隐式调用：存到hashset里，map的key，调用了默认未经重写的toString()方法等等)，把“坑位”占了，偏向锁想存的线程id没地方存了，自然就直接是轻量级锁了.
"调用过hashcode再同步发现是轻量锁"


可以使用 jol 
```java

<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.9</version>
</dependency>

public static void main(String[] args) {
    // 声明一枚长度为3306的数组
    int[] intArr = new int[3306];
    // 使用jol的ClassLayout工具分析对象布局
    System.out.println(ClassLayout.parseInstance(intArr).toPrintable());
}

```
输出
```
[I object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           61 01 00 f8 (01100001 00000001 00000000 11111000) (-134217375)
     12     4        (object header)                           ea 0c 00 00 (11101010 00001100 00000000 00000000) (3306)
     16 13224    int [I.<elements>                             N/A
Instance size: 13240 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

![](4.png)

由图可得出结论，对象在刚实例化好的时候，非常“干净”，乍眼看去两排0，但是只有一个1显得十分突兀.

```
01 00 00 00
00 00 00 00
```
其实它是001，为上文提到的偏向锁标志+锁标志，所有锁的状态如下

![](5.png)

对象大小的计算，也是非常精准的, 即：13224(一个int为四个字节乘3306) + 16 = 13240个字节.

### 对象访问定位

目前主流的访问方式有**使用句柄**和**直接指针**两种

* 使用句柄

![](6.png)

Java堆中将会划分出一块内存来作为句柄池。

* 直接指针

![](7.png)


这两种对象访问方式各有优势， 使用句柄来访问的最大好处就是reference中存储的是稳定的句柄地址，在对象被移动（垃圾收集时移动对象是非常普遍的行为） 时只会改变句柄中的实例数据指针， 而reference本身不需要修改。

使用直接指针访问方式的最大好处就是速度更快， 它节省了一次指针定位的时间开销，由于对象的访问在Java中非常频繁， 因此这类开销积少成多后也是一项非常可观的执行成本。 

就本书讨论的主要虚拟机Sun HotSpot而言， 它是使用第二种方式进行对象访问的。

### 溢出

java 中除了 程序计数器 不会溢出，其他都会溢出。

* 堆溢出

堆用于存储对象实例，只要不断地创建对象， 并且保证GC Roots到对象之间有**可达路径**来避免垃圾回收机制清除这些对象， 那么在对象数量到达最大堆的容量限制后就会产生内存溢出异常。

* 虚拟机栈和本地方法栈溢出

例如递归调用方法，不断的压栈，就会栈溢出，使用迭代器。

* 方法区和运行时常量池溢出

方法区用于存放Class的相关信息， 如类名、 访问修饰符、 常量池、 字段描述、 方法描述等。只要运行时产生大量类去填满方法区，就会溢出。

* 本机直接内存溢出

由DirectMemory导致的内存溢出， 一个明显的特征是在Heap Dump文件中不会看见明显的异常， 如果读者发现OOM之后Dump文件很小， 而程序中又直接或间接使用了NIO， 那就可以考虑检查一下是不是这方面的原因

# 垃圾收集器与内存分配策略

## 怎么确定对象的生存或死亡

### 引用计数法

给对象中添加一个引用计数器， 每当有一个地方引用它时， 计数器值就加1； 当引用失效时， 计数器值就减1； 任何时刻计数器为0的对象就是不可能再被使用的。

但是， 至少主流的Java虚拟机里面没有选用引用计数算法来管理内存， 其中最主要的原因是它很难解决对象之间相互**循环引用**的问题。

```java
public class ReferenceCountingGC{
    public Object instance=null;
    private static final int_1MB=1024*1024;
    /**
    *这个成员属性的唯一意义就是占点内存， 以便能在GC日志中看清楚是否被回收过
    */
    private byte[]bigSize=new byte[2*_1MB];

    public static void testGC() {
        ReferenceCountingGC objA=new ReferenceCountingGC();
        ReferenceCountingGC objB=new ReferenceCountingGC();
        objA.instance=objB；
        objB.instance=objA；
        objA=null；
        objB=null；
        //假设在这行发生GC,objA和objB是否能被回收?
        System.gc();
    }
}
```
虚拟机不会因为他们有互相引用而不回收他们，所以虚拟机不是引用计数的

### 可达性分析

基本思路就是通过一系列的称为“GC Roots”的对象作为起始点， 从这些节点开始向下搜索， 搜索所走过的路径称为引用链（ Reference Chain） ， 当一个对象到GC Roots没有任何引用链相连（ 用图论的话来说， 就是从GC Roots到这个对象不可达） 时， 则证明此对象是不可用的。 如下图所示， 对象object 5、 object 6、 object 7虽然互相有关联， 但是它们到GC Roots是不可达的， 所以它们将会被判定为是可回收的对象。

![](8.png)

## 引用

JDK1.2 之后有4种引用

### 强引用

强引用就是指在程序代码之中普遍存在的， 类似“Object obj=new Object()”这类的引用， 只要强引用还存在， 垃圾收集器永远不会回收掉被引用的对象。

### 软引用

软引用是用来描述一些还有用但并非必需的对象。 对于软引用关联着的对象， 在系统将要发生内存溢出异常之前， 将会把这些对象列进回收范围之中进行第二次回收。 如果这次回收还没有足够的内存， 才会抛出内存溢出异常。 在JDK 1.2之后， 提供了SoftReference类来实现软引用。

例如浏览器的回退

### 弱引用

弱引用也是用来描述非必需对象的， 但是它的强度比软引用更弱一些， 被弱引用关联的对象只能生存到下一次垃圾收集发生之前。 当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。 在JDK 1.2之后， 提供了WeakReference类来实现弱引用。

### 虚引用

虚引用也称为幽灵引用或者幻影引用， 它是最弱的一种引用关系。 一个对象是否有虚引用的存在， 完全不会对其生存时间构成影响， 也无法通过虚引用来取得一个对象实例。 为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。 在JDK 1.2之后， 提供 PhantomReference类来实现虚引用

## 对象真正死亡

在可达性分析完成后，被标记为不可达的对象，也不是立马就去死了，会经过两次标记，第一次标记是在可达性分析完成后，没有 GC Roots 引用，并且进行一次筛选。筛选的条件是此对象是否有必要执行 finalize() 方法。 当对象没有覆盖 finalize() 方法，或者 finalize() 方法已经被虚拟机调用过， 虚拟机将这两种情况都视为 **“没有必要执行”**。

如果这个对象被判定为有必要执行 finalize() 方法， 那么这个对象将会放置在一个叫做 **F-Queue 的队列** 之中， 并在稍后由一个由虚拟机自动建立的、 低优先级的 **Finalizer线程** 去执行它。 这里所谓的“执行”是指虚拟机会触发这个方法， 但并不承诺会等待它运行结束， 这样做的原因是， 如果一个对象在 finalize() 方法中执行缓慢， 或者发生了死循环（ 更极端的情况） ， 将很可能会导致F-Queue队列中其他对象永久处于等待， 甚至导致整个内存回收系统崩溃。 finalize() 方法是对象逃脱死亡命运的最后一次机会， 稍后GC将对F-Queue中的对象进行第二次小规模的标记， 如果对象要在finalize() 中成功拯救自己——只要重新与引用链上的任何一个对象建立关联即可， 那在第二次标记时它将被移除出“即将回收”的集合； 如果对象这时候还没有逃脱， 那基本上它就真的被回收了。 

```java
public class FinalizeEscapeGC {

    public static FinalizeEscapeGC SAVE_HOOK = null;

    public void isAlive() {
        System.out.println("yes,i am still alive： ) ");
    }

    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        System.out.println("finalize method executed！ ");
        FinalizeEscapeGC.SAVE_HOOK = this;
    }

    public static void main(String[] args) throws Throwable {
        SAVE_HOOK = new FinalizeEscapeGC();
        //对象第一次成功拯救自己
        SAVE_HOOK = null;
        System.gc();
        //因为finalize方法优先级很低， 所以暂停0.5秒以等待它
        Thread.sleep(500);
        if (SAVE_HOOK != null) {
            SAVE_HOOK.isAlive();
        } else {
            System.out.println("no,i am dead： ( ");
        }
        //下面这段代码与上面的完全相同，但是这次自救却失败了
        SAVE_HOOK = null;
        System.gc();
        //因为finalize方法优先级很低， 所以暂停0.5秒以等待它
        Thread.sleep(500);
        if (SAVE_HOOK != null) {
            SAVE_HOOK.isAlive();
        } else {
            System.out.println("no,i am dead： ( ");
        }
    }

}


// 输出
finalize method executed！ 
yes,i am still alive： ) 
no,i am dead： ( 

```

最好不要使用 finalize(), 它的运行代价高昂， 不确定性大， 无法保证各个对象的调用顺序。

## 方法区回收

方法区（永久代）的垃圾收集效率非常低，包含 **废弃常量 、无用的类** 两类。

### 废弃常量回收

假如一个字符串“abc”已经进入了常量池中， 但是当前系统没有任何一个String对象是叫做“abc”的， 换句话说， 就是没有任何String对象引用常量池中的“abc”常量， 也没有其他地方引用了这个字面量， 如果这时发生内存回收， 而且必要的话， 这个“abc”常量就会被系统清理出常量池。 常量池中的其他类（ 接口） 、 方法、 字段的符号引用也与此类似。

### 无用的类回收

1. 该类所有的实例都已经被回收， 也就是Java堆中不存在该类的任何实例。
2. 加载该类的ClassLoader已经被回收。
3. 该类对应的java.lang.Class对象没有在任何地方被引用， 无法在任何地方通过反射访问该类的方法。

可以回收而不是必须

# 垃圾收集算法

## 标记-清除

顾名思义，先标记出所有需要回收的对象，再清除掉。
缺点：
1. 标记和清除的效率都不高
2. 会产生空间碎片

## 复制

它将可用内存按容量划分为大小相等的两块， 每次只使用其中的一块。 当这一块的内存用完了， 就将还存活着的对象复制到另外一块上面， 然后再把已使用过的内存空间一次清理掉。 这样使得每次都是对整个半区进行内存回收， 内存分配时也就不用考虑内存碎片等复杂情况， 只要移动堆顶指针， 按顺序分配内存即可， 实现简单， 运行高效。 只是这种算法的代价是将内存缩小为了原来的一半， 未免太高了一点。

HotSpot 根据分代设计，将年轻代 按照 8:1:1 分为 Eden、Survivor0、Survivor1。每次使用Eden和其中一块Survivor。当回收时， 将Eden和Survivor中**还存活着的对象**一次性地复制到另外一块Survivor空间上， 最后清理掉Eden和刚才用过的Survivor空间。 

当Survivor空间不够用时， 需要依赖其他内存（ 这里指老年代） 进行**分配担保（ Handle Promotion）**。

## 标记-整理

老年代的对象，基本都会存活很长时间，可能存在较多大对像，如果使用复制，可能效率会低。标记出存活对象，让活着的对象向一端移动，然后清理掉端边界之外的内存。

## 分代收集

一般是把Java堆分为**新生代**和**老年代**， 这样就可以根据各个年代的特点采用最适当的收集算法。 
在新生代中， 每次垃圾收集时都发现有大批对象死去， 只有少量存活， 那就选用复制算法， 只需要付出少量存活对象的复制成本就可以完成收集。 
老年代中因为对象存活率高、 没有额外空间对它进行分配担保， 就必须使用“标记—清理”或者“标记—整理”算法来进行回收。

Minor GC （年轻代GC） Major GC （老年代的GC）

# HotSpot 算法实现

## 枚举根节点

可作为GC Roots的节点主要在**全局性的引用(例如常量或类静态属性)与执行上下文(例如栈帧中的本地变量表)** 中

枚举根节点的时候是会发生 STW (stop the world) 的。

在 HotSpot 中,是使用一组称为 **OopMap** 的数据结构来达到这个目的的， 在类加载完成的时候， HotSpot就把对象内什么偏移量上是什么类型的数据计算出来， 在JIT编译过程中， 也会在特定的位置记录下栈和寄存器中哪些位置是引用。 这样， GC在扫描时就可以直接得知这些信息了。而不用去遍历全部的东西，可能方法区就几百兆。

String.hashCode() 的本地代码

```
[Verified Entry Point]
0x026eb730： mov%eax， -0x8000（ %esp）
……
； ImplicitNullCheckStub slow case
0x026eb7a9： call 0x026e83e0
； OopMap{ebx=Oop[16]=Oop off=142}
； *caload
； -java.lang.String： hashCode@48（ line 1489）
； {runtime_call}
0x026eb7ae： push$0x83c5c18
； {external_word}
0x026eb7b3： call 0x026eb7b8
0x026eb7b8： pusha
0x026eb7b9： call 0x0822bec0； {runtime_call}
0x026eb7be： hlt
```

可以看到在 0x026eb7a9 处的 call 指令有 OopMap 记录， 它指明了EBX寄存器和栈中偏移量为16的内存区域中各有一个普通对象指针(Ordinary Object Pointer)的引用， 有效范围为从 call 指令开始直到 0x026eb730(指令流的起始位置)+142(OopMap记录的偏移量)=0x026eb7be， 即hlt指令为止。

## 安全点 Safepoint

HotSpot 没有为每条指令都生成 OopMap，只有在特定的位置，也就是 安全点，才会记录。程序执行时并非在所有地方都能停顿下来开始GC， 只有在到达安全点时才能暂停。例如方法调用、 循环跳转、 异常跳转等， 所以具有这些（指令序列复用）功能的指令才会产生Safepoint。

有两种方式让 GC 发生的时候所有线程快速"跑"到最近的安全点上停顿下来

* 抢先式中断（ Preemptive Suspension）

在GC发生时， 首先把所有线程全部中断， 如果发现有线程中断的地方不在安全点上，就恢复线程， 让它“跑”到安全点上。 现在几乎没有虚拟机实现采用抢先式中断来暂停线程从而响应GC事件。

* 主动式中断（ Voluntary Suspension）

GC需要中断线程的时候， 不直接对线程操作， 仅仅简单地设置一个标志， 各个线程执行时主动去轮询这个标志， 发现中断标志为真时就自己中断挂起。
轮询标志的地方和安全点是重合的， 另外再加上创建对象需要分配内存的地方。 
下面代码中的test指令是HotSpot生成的轮询指令， 当需要暂停线程时， 虚拟机把0x160100的内存页**设置为不可读**， 线程执行到test指令时就会产生一个**自陷异常信号**， 在预先注册的异常处理器中**暂停线程实现等待**， 这样一条汇编指令便完成安全点轮询和触发线程中断。

```
0x01b6d627： call 0x01b2b210； OopMap{[60]=Oop off=460}
； *invokeinterface size
； -Client1： main@113（ line 23）
； {virtual_call}
0x01b6d62c： nop
； OopMap{[60]=Oop off=461}
； *if_icmplt
； -Client1： main@118（ line 23）
0x01b6d62d： test%eax， 0x160100； {poll}
0x01b6d633： mov 0x50（ %esp） ， %esi
0x01b6d637： cmp%eax， %esi
```

## 安全区域 Safe Region

存在 没有正在运行的线程，例如 没有被分配时间片，sleep，或者挂起了的线程，这些线程不能进入到安全点，所以有了安全区域。
在这个区域中的任意地方开始GC都是安全的。 我们也可以把Safe Region看做是被扩展了的Safepoint。

在线程执行到Safe Region中的代码时， 首先**标识自己已经进入了Safe Region**， 那样， 当在这段时间里JVM要发起GC时， 就不用管标识自己为Safe Region状态的线程了。 
在线程要离开Safe Region时， 它要检查系统是否已经完成了根节点枚举（ 或者是整个GC过程） ， 如果完成了， 那线程就继续执行， 否则它就必须等待直到收到可以安全离开Safe Region的信号为止。

# 垃圾收集器

![](9.png)

连线表示可以搭配使用

* 并行（ Parallel） ： 指多条垃圾收集线程并行工作， **但此时用户线程仍然处于等待状态**。

* 并发（ Concurrent） ： 指用户线程与垃圾收集线程同时执行（ 但不一定是并行的， 可能会交替执行） ， 用户程序在继续运行， 而垃圾收集程序运行于另一个CPU上。

## 三色标记算法

可达性分析进一步延伸出三色标记法。通常，GC 算法会维持一套对象图，图上节点表示对象，节点之间的连线表示对象间的引用关系，其中：

* 白色节点：尚未被标记的对象；

* 黑色节点：已经被标记，且其引用关系已经被处理；

* 灰色节点：已经被标记，但引用关系尚未被处理；

## CMS (Concurrent Mark Sweep)

CMS 的堆内存结构

![](19.png)

JDK 9 已经宣布废除 CMS 垃圾收集器且默认为 G1。

以获取最短回收停顿时间为目标的收集器,主要分为4个阶段

1. 初始标记

STW, 只标记和 GC Roots 相关的节点，只标记一层

2. 并发标记

并发，根据刚刚标记的节点继续标记

3. 重新标记

STW, 快速标记，刚刚并发标记阶段产生的引用之类的

4. 并发清除

并发，清除不用对象

CMS 对 CPU 比较敏感，可能会占用资源，让用户线程变慢，
CMS 无法处理 **浮动垃圾**，由于 CMS 并发清理阶段，用户线程还在运行，所以就会产生垃圾，这一部分垃圾本次不会收集到，如果预留的内存不够，就可能产生 full GC，
CMS 是标记清除，所以会留下碎片空洞，而无法创建大对象，内存整理是无法并发的，所以会 STW，默认 CMS 会在每次收集后进行整理，但是可以设置，多少次不压缩后，进行一次压缩（-XX： CMSFullGCsBeforeCompaction）

### 其他说法

CMS 处理过程有七个步骤：

1. 初始标记(CMS-initial-mark) ,会导致stw;

标记出的对象会压入标记栈；
标记出年轻代对象引用到了的老年代对象，

2. 并发标记(CMS-concurrent-mark)，与用户线程同时运行；

这个阶段用户线程是并发的，如果有对象晋升，新对象引用到老对象，大对象直接在老年代，会把这些对象所在的Card标记为Dirty，后续只需扫描这些Dirty Card的对象，避免扫描整个老年代；可能会导致concurrent mode failure。

* card table

YGC 的时候，老年代可能会引用年轻代的对象，但是扫描整个老年代代价太大，就引入了卡表。

卡表**逻辑上**把老年代内存分成一个个大小相等的卡片(card,论文中提到适合大小是**128个字节**)，然后对每个卡片准备一个与其对应的标记位,并将这些位集中起管理就好像一个表格(mark table)一样,当改写对象引用是从老年代指向新生代时，在老年代对应的卡片标记位上设置标志位即可，通常这样的卡片我们称之为dirty card。
这项操作可以通过**write barrier**来实现，这样就算对象跨多张卡片也不会有什么问题。
卡表通常是用byte数组实现的，byte的值只能取[0,1]这两种。所以btye[i] = 1 就表示第i + 1 卡片所在内存上有指向新生代引用的老年代对象，这时只要tracing这个卡片上的对象即可。
如果每个card大小的是128字节(1024位，)那卡表就只占整个老年代的1/1024之一。所以遍历卡表的时间会远比遍历整个老年代快得多！
这其中背后思想就是典型以空间换时间的思路！这种思路在G1中也有体现，只不其对应的数据是remember set而已。

由于新生代GC与老年代GC同时使用card table，所以会出现冲突的情况:
1. 并发标记还未扫描到脏卡1.
2. Young GC扫描完脏卡，并改变dirty到clean.
3. 并发标记扫描，发现卡1已不是脏卡，则不会处理，这就造成了漏标。

增加了另一个数据结构mod union table解决此问题。

![](10.png)

mod union table是一个bit位向量，一个bit表示一个card的状态。

它由新生代垃圾收集器维护，新生代GC将card设置为clean之前，把mod union table设置为dirty。
card table状态为dirty、或者mod union table标记为dirty、或者同时两种数据结构都标记为dirty的card表示并发标记阶段引用发生了变化，需要在后面的阶段进行处理。

* concurrent mode failure

CMS回收年老代的速度太慢，导致年老代在CMS完成前就被沾满，引起full gc

* write barrie 

写屏障，用户线程写对象引用的时候就触发write barrier的逻辑，将对象所处的card设置为dirty。

3. 预清理（CMS-concurrent-preclean），与用户线程同时运行；

4. 可被终止的预清理（CMS-concurrent-abortable-preclean） 与用户线程同时运行；

预处理不会一直预处理下去，重复的次数、多少量的工作、持续的时间等等都会终止预清理。

5. 重新标记(CMS-remark) ，会导致swt；

遍历新生代对象，重新标记、根据GC Roots，重新标记、遍历老年代的Dirty Card和Mod Union Table，重新标记
完成标记整个老年代的所有的存活对象。
如果老年代有对新生代对象的引用，那么这些对象也是存活的，这个阶段时间较长，可以通过设置参数 -XX:+CMSScavengeBeforeRemark，在重新标记前执行一次 YGC，清理年轻代无用对象，或者将对象放入 survive 或者晋升到 老年代，一般 survive 很小，所以速度会很快。

6. 并发清除(CMS-concurrent-sweep)，与用户线程同时运行；

回收那些不能用的对象。

7. 并发重置状态等待下次CMS的触发(CMS-concurrent-reset)，与用户线程同时运行；

重置CMS的数据结构。

## G1



## GC 日志

```
33.125 : [GC    [DefNew : 3324K-> 152K(3712K) , 0.0025925 secs]3324K-> 152K(11904K)，0.0031680 secs]
100.667: [FullGC[Tenured: 0K   -> 210K(10240K)，0.0149142 secs]4603K-> 210K(19456K), [Perm: 2999K->2999K(21248K)]， 0.0150007 secs][Times: user=0.01 sys=0.00, real=0.02 secs]
```
33.125、100.667 代表 GC 发生的时间，从 java 虚拟机启动以来经过的秒数。
3324K -> 152K(3712K) ：GC前该内存区域已使用容量 -> GC后该内存区域已使用容量(该内存区域总容量)
3324K -> 152K(11904K)：GC前Java堆已使用容量 -> GC后Java堆已使用容量(Java堆总容量)
0.0031680 secs : GC 的时间，单位是秒

## 内存分配与回收策略

1. 对象优先在 eden 分配，Eden 没有空间的时候会 Minor GC;
2. 大对象会直接进入老年代，以来是年轻代空间不够，二来是避免晋升时对象拷贝。
3. 长期存活对象会进入老年代，对象头里面4bit位来标志的，最多 15 次，没经历一次Minor GC 会加1.
4. 动态对象年龄判断。Survivor空间中**相同年龄所有对象大小的总和大于Survivor空间的一半**， 年龄大于或等于该年龄的对象就可以直接进入老年代， 无须等到MaxTenuringThreshold中要求的年龄。
5. 分配担保

### 分配担保

虚拟机会先检查老年代**最大可用的连续空间是否大于新生代所有对象总空间**， 
如果这个条件成立， 那么Minor GC可以确保是安全的。 
如果不成立， 则虚拟机会查看 HandlePromotionFailure 设置值是否允许担保失败。 
如果允许， 那么会继续检查**老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小**， 
如果大于， 将尝试着进行一次Minor GC， 尽管这次Minor GC是有风险的； 
如果小于， 或者HandlePromotionFailure设置不允许冒险， 那这时也要改为进行一次Full GC。

# 虚拟机性能监控与故障处理工具

## JPS

JVM Process Status Tool

* -q

只输出 LVMID, 省略主类名称

* -m

输出虚拟机启动的时候传递给主类的参数

* -l

输出主类名称，如果是jar包，则是 jar 路径

* -v 

输出虚拟机启动的时候主类的参数

## jstat

JVM Process Status Tool

监视类装载、卸载的情况、总空间以及类装载所耗费的时间

https://sailliao.top/2020/12/29/jstat%E4%BD%BF%E7%94%A8/

![](20.png)

## jinfo

java配置信息工具

## jmap

![](21.png)

## jhat

虚拟机堆转储快照分析工具

## jstack

Java堆栈跟踪工具

## HSDIS

JIT生成代码反汇编

# JDK 可视化工具

## jConsole

Java监视与管理控制台

jconsole.exe 在 jdk 的 bin 目录下面，双击打开就行.

![](22.png)

## VisualVM

多合一故障处理工具
