# 一、基础

[JVM参数设置详解](https://blog.csdn.net/weixin_37195606/article/details/82805216)

## 1. 内存

-Xms设置内存初始化大小

-Xmx设置最大能够使用内存的大小

内存泄漏：由程序申请的一块内存，如果没有任何一个指针指向它，那么这块内存就泄漏了。

## 2. 解释执行&编译执行

在Java编程语言和环境中，及时编译器（JIT compiler，just-in-time compiler）是一个把Java的字节码（.class）转换成可以直接发送给处理器的指令的程序。目前Java使用的是混合执行模式，部分函数会被解释执行，部分可能被编译执行。JVM决定函数是否需要编译执行的一句是判断该函数是否为热点代码。如果函数被调用频率高，那么就是热点，热点代码就会被编译执行。编译为机器码。

启用解释模式：Java -Xint -version

启用编译模式：Java -Xcomp -version

## 3. 对象存活判断

### 3.1 引用计数

每个对象有一个引用计数属性，新增一个引用时计数加1，引用释放时计数减1，计数为0时可以回收。

### 3.2 可达性分析

从GC Roots开始向下搜索，搜索所走过的路径称为引用链，当一个对象没有任何引用链相连时，则此对象称为不可达对象，可以回收。

### 3.3 方法区回收

方法区主要存放永久代对象，而永久代对象的回收率比新生代低很多，所以在方法区上进行回收性价比不高，主要是对常量池的回收和对类的卸载。

为避免内存溢出，大量使用反射和动态代理时都需要虚拟机具备类卸载功能。

类的卸载条件很多，需要满足以下三个条件，并且满足了条件也不一定会被卸载：

- 该类所有的实例都已被回收，此时堆中不存在该类的任何实例。
- 加载该类的 ClassLoader 已被回收。
- 该类对应的 Class 对象没有在任何地方被引用，也就无法在任何地方通过反射访问该类方法。

### 3.4 finalize()

类似 C++ 的析构函数，用于关闭外部资源。但是 try-finally 等方式可以做得更好，并且该方法运行代价很高，不确定性大，无法保证各个对象的调用顺序，因此最好不要使用。

当一个对象可被回收时，如果需要执行该对象的 finalize() 方法，那么就有可能在该方法中让对象重新被引用，从而实现自救。自救只能进行一次，如果回收的对象之前调用了 finalize() 方法自救，再回收时不会再调用该方法。

## 4. 垃圾回收

随着系统运行时间的不断增长，垃圾对象所耗内存可能持续上升，直到出现内存溢出造成应用程序崩溃。

垃圾是指在运行程序中没有任何指针指向的对象。

## 5. 碎片整理

由于创建对象和垃圾回收器释放丢弃对象所占的空间时，内存会因此出现碎片。碎片整理将所占用的堆内存移到堆的一端，以便JVM将整理出的内存分配给新的对象。

# 二、回收算法

### 1. 标记清除算法

第一次遍历，从根节点开始标记所有被引用的对象，第二次遍历，将所有未标记的对象回收。此算法需要暂停整个应用，且会产生内存碎片。

### 2. 复制算法

把内存空间划分为两个相等的区域，遍历当前使用区域，把正在使用的对象复制到另一个区域。此算法不会产生内存碎片，但需要两倍内存空间。

### 3. 标记整理算法

标记所有被引用对象，回收未标记对象并把存活对象“压缩”到堆的其中一块，按顺序排放。此算法不产生内存碎片，且不需要两倍内存空间。

### 4. 分代收集算法

基于统计结果并综合上面三个算法的优点。分为年轻代、老年代、永久代，并根据情况回收不同分区。

#### 4.1 年轻代&老年代

- 年轻代分1个Eden区和2个Survivor区

- 回收年轻代的行为被称为Minor GC。
- 回收老年代的行为被称为Major GC 或者 Full GC。
- 新创建的对象存放在年轻代（Eden区）。
- 大对象（需要分配的内存大于等于Eden区的一半）直接进入老年代。
- 因为大多数对象朝生夕死，为减少垃圾回收时的扫描范围，将生命周期短的对象存放在年轻代。
- Minor GC的发生频率比Full GC高很多，即年轻代发生垃圾回收的频率比老年代高。

#### 4.2 对象晋升

- 对象在Eden区经历一次Minor GC后仍然存活，且能被Survivor容纳则移动到Survivor区，同时年龄设为1。

- 对象在Survivor区经历一次Minor GC，年龄增加1岁，当年龄达到晋升阈值（MaxTenuringThreshold）就会晋升到老年代。

- Survivor空间中，年龄n的对象占用的内存总和大于Survivor空间的目标存活率（TargetSurvivorRatio），则年龄大于等于n的对象晋升为老年代。

#### 4.3 回收策略

每次Minor GC前，JVM会检查**老年代最大可用的连续空间**是否大于**年轻代所有对象的总空间**。

如果条件成立，则Minor GC是安全的，执行Minor GC。

如果不成立，则JVM查看HandlePromotionFailure是否允许担保失败。

如果允许担保失败，则检查老年代**最大可用的连续空间**是否大于**历次晋升到老年代对象的平均大小**。

如果大于，则进行一次Minor GC，尽管这次Minor GC是有风险的。

如果小于，或者HandlePromotionFailure设置不允许担保失败，则改为进行一次Full GC。

设置晋升阈值：-XX:MaxTenuringThreshold（默认为15）
设置Survivor目标存活率：-XX:TargetSurvivorRatio（默认为50）

#### 4.4 永久区（PermGen Space）

永久区又叫Perm区，是内存的永久保存区域，用于存放Class和Meta的信息。只存在于HotSpot JVM中，并且只存在于JDK 7和之前的版本中，JDK 8中已经彻底移除了永久区，JDK 8中引入了一个新的内存区域叫MetaSpace。

# 三、JVM内存模型

Java虚拟机在执行Java程序的过程中会把它所管理的内存划分为若干不同的数据区域。

![img](G:\资料\复习笔记\My-study-notes\assets\imgs\JVM运行时数据区域模型.png)

## 1. 程序计数器

记录正在执行的虚拟机字节码指令的地址（如果正在执行的是本地方法则为空）。

## 2. 本地方法栈

本地方法栈与 Java 虚拟机栈类似，它们之间的区别只不过是本地方法栈为本地方法服务。

本地方法一般是用其它语言（C、C++ 或汇编语言等）编写的，并且被编译为基于本机硬件和操作系统的程序，对待这些方法需要特别处理。

![img](G:\资料\复习笔记\My-study-notes\assets\imgs\Java本地接口.png)

## 3. Java 栈（虚拟机栈）

每个 Java 方法在执行的同时会创建一个栈帧用于存储局部变量表、操作数栈、常量池引用等信息。从方法调用直至执行完成的过程，就对应着一个栈帧在 Java 虚拟机栈中入栈和出栈的过程。

## 4. Java堆

所有对象都在这里分配内存，是垃圾收集的主要区域（"GC 堆"）。

现代的垃圾收集器基本都是采用分代收集算法，其主要的思想是针对不同类型的对象采取不同的垃圾回收算法。可以将堆分成两块：

- 新生代（Young Generation）
- 老年代（Old Generation）

堆不需要连续内存，并且可以动态增加其内存，增加失败会抛出 OutOfMemoryError 异常。

可以通过 -Xms（初始值） 和 -Xmx（最大值） 这两个虚拟机参数来指定一个程序的堆内存大小。

## 5. 方法区

用于存放已被加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。

和堆一样不需要连续的内存，并且可以动态扩展，动态扩展失败会抛出 OutOfMemoryError 异常。

对方法区进行垃圾回收的主要目标是对常量池的回收和对类的卸载，但是一般比较难实现。

HotSpot 虚拟机把方法区当成永久代来进行垃圾回收。但很难确定永久代的大小，因为它受到很多因素影响，并且每次 Full GC 之后永久代的大小都会改变，所以经常会抛出 OutOfMemoryError 异常。为了更容易管理方法区，从 JDK 1.8 开始，移除永久代，并把方法区移至元空间，它位于本地内存中，而不是虚拟机内存中。

## 6. 运行时常量池

运行时常量池是方法区的一部分。

Class 文件中的常量池（编译器生成的字面量和符号引用）会在类加载后被放入这个区域。

除了在编译期生成的常量，还允许动态生成，例如 String 类的 intern()。

## 7. 直接内存

在 JDK 1.4 中新引入了 NIO 类，它可以使用 Native 函数库直接分配堆外内存，然后通过 Java 堆里的 DirectByteBuffer 对象作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了在堆内存和堆外内存来回拷贝数据。