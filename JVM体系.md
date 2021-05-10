# 一、jvm、 class 结构

![image-20200617165555087](.\图片\image-20200617165555087.png)

任何语言，只要符合class的规范，都可以再java虚拟机上执行

java虚拟机是一种规范，jvm和java无关，符合jvm的规范，就可以直接在jvm上运行。

jvm规范上哪找？官 网  https://docs.oracle.com/en/java/javase/13/ 

常见jvm规范

> Hotspot,
>
> 曾经的Jrokit,
>
> Taobao Vm
>
> Liquid Vm

2. Class文件中包含哪些东西-- Class File Format 

​	idea查看二进制编码类文件插件 -- BinEd  -- open as binary  ---一般使用 Hex dicimal 16进制来看

​	j class lib 也可

   Class文件中包含哪些东西：

```java
//u1 ，u2 ,u4  代表字节数，一个十六进制数是4位，两个十六进制数8位，也就是1个字节
ClassFile {
    u4             magic; 
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```



# 二、类加载-初始化

1）Loading

​		1.双亲委派

​	    2.LazyLoading的五种情况

​        3.ClassLoad的源码

​		4.自定义类加载器

​		5.混合执行，编译执行，解释执行

2)  Linking

1. Verification 校验  (检验文件是否符合JVM规定)
2. preparation 准备 (类中的静态变量赋默认值)
3. resolution  解析 (将 类，方法，属性等符号引用解析为直接引用 )

3）initializing 初始化 （给静态成员变量赋初始值）



类加载器- ClassLoader

一个类创建的过程  ： 类的二进制码被加载到内存中，类的对象被创建到堆中，然后指向被加载到内存中的二进制文件

如何查看获取ClassLoader ？  String.class.getClassLoader();

类加载器的层次问题

![image-20200901154352285](.\图片\image-20200901154352285.png)

不同的类加载器负责加载不同的类

Bootstrap的类加载器都是null （BootStrap由C++实现的模块，所以没有java对象能对应，所以是null） 。 父加载器不是继承关系

双亲委派 : 自底向上检查该类可否被加载，如果没有则再自顶向下进行实际查找和加载

为何需要双亲委派？ 为了安全，或者是加载过的类不要再次加载，只需要从顶层找

ClassLoad源码体现了双亲委派的过程，然后，这个过程体现了一个设计模式： 模板方法

​	先去缓存找是否已加载这个类 (findCache) ，如果有，直接返回，如果没有，再去父加载器中找，进行一个递归调用，然后再从父到子，如果都没有，则报 ClassNotFound

自定义类加载器：继承ClassLoader并且 重写findClass方法

如何打破双亲委派机制？tomcat都有自己指定的ClassLoader，这里面打破了双亲委派机制，直接继承ClassLoader并且重写loadClass



5.类的编译过程

java默认为混合模式 ：解释器+ 热点代码编译

热点代码：多次被调用的方法，多次被调用的循环

参数    -Xmixed 默认为混合模式，开始解释执行，启动速度较快，对热点代码进行检测和编译

-Xint 使用节食模式，启动很快，执行稍慢

-Xcomp 使用纯编译模式，执行很快，启动很慢



# 三、java内存模型

## 1.硬件层 -- 并发优化基础知识

<img src=".\图片\image-20200914155103813.png" alt="image-20200914155103813" style="zoom:50%;" />

因此，CPU从外部共享缓存中读取数据时，就遇到一个问题，多个cpu核心对应各自的L1,L2缓存，怎样保证这些核心的缓存一致性？

<img src=".\图片\image-20200914155827335.png" alt="image-20200914155827335" style="zoom:50%;" />

答：使用缓存一致性协议。InterCpu使用的是MESI协议，不同 的Cpu使用不同的协议

给数据标记状态，分为四种状态。

<img src=".\图片\image-20200914160504230.png" alt="image-20200914160504230" style="zoom:50%;" />

因此，现代Cpu的一致性实现 = 缓存锁 (MESI ..)+ 总线锁

Cache Line 缓存行 的概念 ： 读取数据时，不是需要多少读多少，而是一次性读取 64 字节 (一块缓存) 数据。

伪共享的概念：位于同一缓存行的两个不同数据，被两个不同cpu锁定，其中的一个发生改变，就通知cpu重读整个缓存行。。比如cpu1用到的是x，但被改动的是y,但x和y在同一缓存行，cpu1依然要重读整个缓存行

解决伪共享的办法：缓存行对齐

应用：disruptor 消息队列，其中的游标数据，前后增加了7个字节数据进行缓存对齐

## 2.cpu乱序执行

由于cpu与内存之间的速度差很大，读取数据花费的时间远比计算数据多得多，因此，cpu会在一条指令的执行过程中（比如去内存读取数据慢100倍），去同时执行另一条指令，但前提是，这条指令与前面的指令没有依赖关系。

WCBuffer (writer combine)合并写机制，填满buffer后执行，不用等待，执行速度更快（感觉机制类似于缓存对齐）

**硬件内存屏障：**

CPU内部的 有序性的保障：cpu内存屏障，sfence  -- save fence ;lfence--load fence ; mfence (mix fence -- save 和 load)

原子指令， 如x86上的 "lock..." 指令是一个FullBarrier



**JVM级别如何规范:**

LoadLoad屏障 ； StoreStore屏障 ；LoadStore ；StoreLoad ；

**Volatile实现细节**

1.字节码层面：ACC_VOLATILE

2.JVM层面：

  <img src=".\图片\image-20200914182632833.png" alt="image-20200914182632833" style="zoom: 80%;" />

3.OS和硬件层面： hsdis - HotSpot Dis Assembler	;	windows使用lock实现

**Synchronized实现细节**

1.字节码：monitorenter, monitorexist

2.jvm层面：C++调用了操作系统提供的同步机制，

3.OS和硬件层面：X86：lock cmpxchg (比较和交换) 实现

## 3.关于对象，内存的面试题--带出对象内存布局

**1.解释一下对象的创建过程？**

​	1.Class loading   2.Class Linking (verification:JVM校验类 ；preparation：静态变量赋默认值；resolution：解析，将类，方法，属性等符号引用解析为直接应用)  3.Class initializing (静态变量赋初始值，加载静态代码块)  4.申请对象内存   5.成员变量赋默认值  6.执行构造方法（成员变量顺序赋初始值，执行构造方法语句）

**2.对象在内存中的存储布局？**

​	普通对象：1.对象头（markword 8字节） 2.ClassPointer指针 （指向对应的类对象  默认4字节；不开启压缩参数  为8字节）				3.实例数据 （对象的具体属性，包括引用类型，如String）4.Padding对齐，8的倍数。

​    数组对象：1.对象头	2.ClassPointer 	3.数组长度 (4字节) 	4.数组数据	5.对齐

两个参数： --XX:+UseCompressedClassPointers 		-XX:+UseCompressedOops

Oops = ordinary object pointers 普通对象指针

**3.对象头具体包含什么**

markword 64位：大体包含 锁信息2位，是否偏向1位，gc标记 分带年龄 ， 无锁状态还包含独享的hashcode

<img src=".\图片\image-20200915175426054.png" alt="image-20200915175426054" style="zoom:50%;" />

<img src=".\图片\image-20200915175710021.png" alt="image-20200915175710021" style="zoom:67%;" />

**4.对象怎么定位**

https://blog.csdn.net.clover_lily/article/details/80095580

1.句柄池     对于GC的三色标记算法友好

2.直接指针	hotspot使用直接指针

5.**对象怎么分配**

​	栈，堆。。。老年代，青年代



# 四、java运行时数据区和常用指令

## 1.运行时数据区

<img src=".\图片\image-20200916172416448.png" alt="image-20200916172416448"  />

PC程序计数器：存放指令位置，虚拟机的运行类似一个循环

> while(not end){
>
>  取pc中的位置，找到对应位置的指令；
>
> 执行指令
>
> }

Direct Memory

> ​	JVM可以直接访问的内核空间的内存（OS管理的内存）
>
> ​	NIO，提高效率，实现zero copy



JVM Stack:

Frame 栈帧- 每个方法对应一个栈帧

1. Local Variable Table
2. Operand Stack 对于long的处理（store and load），多数虚拟机的实现都是原子的 jls 17.7，没必要加volatile
3. Dynamic Linking https://blog.csdn.net/qq_41813060/article/details/88379473 jvms 2.6.3
4. return address a() -> b()，方法a调用了方法b, b方法的返回值放在什么地方



Heap

Method Area

1. Perm Space (<1.8) 字符串常量位于PermSpace FGC不会清理 大小启动的时候指定，不能变
2. Meta Space (>=1.8) 字符串常量位于堆 会触发FGC清理 不设定的话，最大就是物理内存



Runtime Constant Pool

Native Method Stack

## 2.基本二进制指令

iload 压栈

istore 弹栈

InvokeStatic 运行静态方法

InvokeVirtual 运行可能会是多态的方法

InvokeSpecial 运行可直接定位的方法，例如private方法，这种不可被重写的方法

InvokeInterface 通过接口调用的方法

InvokeDynamic 运行lambda表达式或者反射等动态语言会用到的

# 五、GC Collector - 三色标记

## 1.垃圾算法

**一、什么是垃圾：**没有任何引用指向的对象

**二、如何找垃圾：**

Refrence count : 没有引用指向的对象为垃圾，但这种算法无法解决循环引用的问题。

Root Searching (根可达算法) ：从根对象开始搜索，搜到的都是非垃圾。

​	哪些是根对象？线程栈变量，静态变量，常量池，JNI指针

**三、找到垃圾后，如何处理：**

1.Mark-Sweep 标记清除：算法相对简单，存活对象较多的时候效率高，两遍扫描，容易产生碎片

2.Copy 算法：将内存一分为二，找到存活对象后，整体复制到空闲一半内存，，然后清除之前一半内存区域；使用存活对象较少的情况，只扫描一次，效率提高，没有碎片；但是容易空间浪费，移动复制对象，需要调整对象的引用

3.Mark-Compact 标记压缩：把存活对象向前填充，把内存整理到连续。不会产生碎片，方便对象分配，不会产生内存减半，扫描两次，需要移动对象，效率偏低。

## 2.堆内存逻辑分区

一、部分垃圾回收器使用的模型

> 除Epsilon ZGC Shenandoah之外的GC都是使用逻辑分代模型
>
> G1是逻辑分代，物理不分代
>
> 除此之外不仅逻辑分代，而且物理分代

新生代 + 老年代 + 永久代（1.7）Perm Generation/ 元数据区(1.8) Metaspace

1. 永久代 元数据 - Class
2. 永久代必须指定大小限制 ，元数据可以设置，也可以不设置，无上限（受限于物理内存）
3. 字符串常量 1.7 - 永久代，1.8 - 堆
4. MethodArea逻辑概念 - 永久代、元数据

新生代 = Eden + 2个suvivor区

1. YGC回收之后，大多数的对象会被回收，活着的进入s0
2. 再次YGC，活着的对象eden + s0 -> s1
3. 再次YGC，eden + s1 -> s0
4. 年龄足够 -> 老年代 （15 CMS 6）
5. s区装不下 -> 老年代

老年代

1. 顽固分子
2. 老年代满了FGC Full GC



对象何时进入老年代？

​	超过 XX:MaxTenuringThreadhold指定次数（YGC）

​		Parallel Scavenge 垃圾回收器 ：15

​		CMS ： 6

​		G1 ：15

动态年龄：	eden + s1 copy到 s2 区域中的对象超过50%的话，动态年龄最大的对象，直接进入到 old区

## 3.常见的垃圾回收器



常见垃圾回收组合 ：1 serial + serial old  ; 2 parallel scavenge + prallel old （ps+po 默认的） ;3 par new + CMS

 serial 单线程垃圾回收器:  STW 停顿，效率低程序会出现卡顿   serial old 用在老年代的单线程垃圾回收器

parallel scavenge ：STW停顿，但是是多线程的垃圾收集  parallel old  压缩算法的垃圾收集器，多线程

Par New : STW停顿 ，但是是ps 的增强  CMS： 里程碑，1.4后开启，并发标记清除

当内存大的时候，垃圾回收时，STW时间会很长, 因此诞生了CMS

CMS：a mostly concurrent ,low-pause collector

​	4 phases:  1.initial mark (标记根对象)  2.concurrent mark(根对象链接的其他对象)   3.remark (部分重新建立连接的对象)  4.concurrent sweep

CMS的问题：

​	1.内存碎片化：因为mark sweep天然的问题，一但老年代产生很多碎片内存，就会使用serial old方法进行标记压缩清除老年代的垃圾

解决：–XX:CMSInitiatingOccupancyFraction 92% 可以降低这个值，让CMS保持老年代足够的空间

ConcurrentMarkSweep 老年代 并发的， 垃圾回收和应用程序同时运行，降低STW的时间(200ms) CMS问题比较多，所以现在没有一个版本默认是CMS，只能手工指定 CMS既然是MarkSweep，就一定会有碎片化的问题，碎片到达一定程度，CMS的老年代分配对象分配不下的时候，使用SerialOld 进行老年代回收 想象一下： PS + PO -> 加内存 换垃圾回收器 -> PN + CMS + SerialOld（几个小时 - 几天的STW） 几十个G的内存，单线程回收 -> G1 + FGC 几十个G -> 上T内存的服务器 ZGC 算法：三色标记 + Incremental Update

G1(10ms) 算法：三色标记 + SATB

ZGC (1ms) PK C++ 算法：ColoredPointers + LoadBarrier

Shenandoah 算法：ColoredPointers + WriteBarrier



垃圾收集器跟内存大小的关系（随着机器内存 的大小，选择不同的垃圾回收器）

1. Serial 几十兆
2. PS 上百兆 - 几个G
3. CMS - 20G
4. G1 - 上百G
5. ZGC - 4T - 16T（JDK13）

## 4.内存分带模型

1. 部分垃圾回收器使用的模型

   > 除Epsilon ZGC Shenandoah之外的GC都是使用逻辑分代模型
   >
   > G1是逻辑分代，物理不分代
   >
   > 除此之外不仅逻辑分代，而且物理分代

2. 新生代 + 老年代 + 永久代（1.7）Perm Generation/ 元数据区(1.8) Metaspace 都在方法区 

   1. 永久代 元数据 - Class
   2. 永久代必须指定大小限制 ，元数据可以设置，也可以不设置，无上限（受限于物理内存）
   3. 字符串常量 1.7 - 永久代，1.8 - 堆
   4. MethodArea逻辑概念 - 永久代、元数据

3. 新生代 = Eden + 2个suvivor区

   1. YGC回收之后，大多数的对象会被回收，活着的进入s0
   2. 再次YGC，活着的对象eden + s0 -> s1
   3. 再次YGC，eden + s1 -> s0
   4. 年龄足够 -> 老年代 （15 CMS 6）
   5. s区装不下 -> 老年代

4. 老年代

   1. 顽固分子
   2. 老年代满了FGC Full GC

5. GC Tuning (Generation)

   1. 尽量减少FGC
   2. MinorGC = YGC
   3. MajorGC = FGC

6. 对象分配过程图 [![img](https://github.com/bjmashibing/JVM/raw/master/%E5%AF%B9%E8%B1%A1%E5%88%86%E9%85%8D%E8%BF%87%E7%A8%8B%E8%AF%A6%E8%A7%A3.png)](https://github.com/bjmashibing/JVM/blob/master/对象分配过程详解.png)

7. 动态年龄：（不重要） https://www.jianshu.com/p/989d3b06a49d

8. 分配担保：（不重要） YGC期间 survivor区空间不够了 空间担保直接进入老年代 参考：https://cloud.tencent.com/developer/article/1082730



## 5.常见垃圾回收组合参数设定(1.8)

- -XX:+UseSerialGC = Serial New (DefNew) + Serial Old
  - 小型程序。默认情况下不会是这种选项，HotSpot会根据计算及配置和JDK版本自动选择收集器
- -XX:+UseParNewGC = ParNew + SerialOld
  - 这个组合已经很少用（在某些版本中已经废弃）
  - https://stackoverflow.com/questions/34962257/why-remove-support-for-parnewserialold-anddefnewcms-in-the-future
- -XX:+UseConc(urrent)MarkSweepGC = ParNew + CMS + Serial Old
- -XX:+UseParallelGC = Parallel Scavenge + Parallel Old (1.8默认) 【PS + SerialOld】
- -XX:+UseParallelOldGC = Parallel Scavenge + Parallel Old
- -XX:+UseG1GC = G1
- Linux中没找到默认GC的查看方法，而windows中会打印UseParallelGC
  - java +XX:+PrintCommandLineFlags -version
  - 通过GC的日志来分辨
- Linux下1.8版本默认的垃圾回收器到底是什么？
  - 1.8.0_181 默认（看不出来）Copy MarkCompact
  - 1.8.0_222 默认 PS + PO

## 6.调优前的基础概念：

1. 吞吐量：用户代码时间 /（用户代码执行时间 + 垃圾回收时间） 俗话：干正经事时间越多，吞吐量就越大
2. 响应时间：STW越短，响应时间越好

所谓调优，首先确定，追求啥？吞吐量优先，还是响应时间优先？还是在满足一定的响应时间的情况下，要求达到多大的吞吐量...

问题：

科学计算，吞吐量。数据挖掘，thrput。吞吐量优先的一般：（PS + PO）

响应时间：网站 GUI API （1.8 G1）

## 7.什么是调优？

1. 根据需求进行JVM规划和预调优
2. 优化运行JVM运行环境（慢，卡顿）
3. 解决JVM运行过程中出现的各种问题(解决OOM)

## 8.调优，从规划开始

- 调优，从业务场景开始，没有业务场景的调优都是耍流氓
- 无监控（压力测试，能看到结果）不调优。根据业务场景调优。
- 步骤：
  1. 熟悉业务场景（没有最好的垃圾回收器，只有最合适的垃圾回收器）
     1. 响应时间、停顿时间 [CMS G1 ZGC] （需要给用户作响应）
     2. 吞吐量 = 用户时间 /( 用户时间 + GC时间) [PS]
  2. 选择回收器组合
  3. 计算内存需求（经验值 1.5G 16G）
  4. 选定CPU（越高越好）
  5. 设定年代大小、升级年龄
  6. 设定日志参数
     1. -Xloggc:/opt/xxx/logs/xxx-xxx-gc-%t.log -XX:+UseGCLogFileRotation  -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=20M -XX:+PrintGCDetails  -XX:+PrintGCDateStamps -XX:+PrintGCCause
     2. 或者每天产生一个日志文件
  7. 观察日志情况

## 9.如何看懂GC日志

### YongGC

> 2019-04-18T14:52:06.790 0800: 2.653: [GC  (Allocation Failure) [PSYoungGen: 33280K->5113K(38400K)]  33280K->5848K(125952K), 0.0095764 secs] [Times: user=0.00 sys=0.00,  real=0.01 secs]

含义：

> 2019-04-18T14:52:06.790 0800（当前时间戳）: 2.653（应用启动基准时间）: [GC (Allocation Failure) [PSYoungGen（表示  Young GC）: 33280K（年轻代回收前大小）->5113K（年轻代回收后大小）(38400K（年轻代总大小）)]  33280K（整个堆回收前大小）->5848K（整个堆回收后大小）(125952K（堆总大小）), 0.0095764（耗时） secs] [Times: user=0.00（用户耗时） sys=0.00（系统耗时）, real=0.01（实际耗时） secs]

### Full GC

> 2019-04-18T14:52:15.359 0800: 11.222: [Full GC (Metadata GC Threshold) [PSYoungGen:  6129K->0K(143360K)] [ParOldGen: 13088K->13236K(55808K)]  19218K->13236K(199168K), [Metaspace: 20856K->20856K(1069056K)],  0.1216713 secs] [Times: user=0.44 sys=0.02, real=0.12 secs]

含义：

> 2019-04-18T14:52:15.359 0800（当前时间戳）: 11.222（应用启动基准时间）: [Full GC (Metadata GC Threshold)  [PSYoungGen: 6129K（年轻代回收前大小）->0K（年轻代回收后大小）(143360K（年轻代总大小）)]  [ParOldGen: 13088K（老年代回收前大小）->13236K（老年代回收后大小）(55808K（老年代总大小）)]  19218K（整个堆回收前大小）->13236K（整个堆回收后大小）(199168K（堆总大小）), [Metaspace:  20856K（持久代回收前大小）->20856K（持久代回收后大小）(1069056K（持久代总大小）)], 0.1216713（耗时）  secs] [Times: user=0.44（用户耗时） sys=0.02（系统耗时）, real=0.12（实际耗时） secs]





案例1：垂直电商，最高每日百万订单，处理订单系统需要什么样的服务器配置？

> 这个问题比较业余，因为很多不同的服务器配置都能支撑(1.5G 16G)
>
> 1小时360000集中时间段， 100个订单/秒，（找一小时内的高峰期，1000订单/秒）
>
> 经验值，
>
> 非要计算：一个订单产生需要多少内存？512K * 1000 500M内存
>
> 专业一点儿问法：要求响应时间100ms
>
> 压测！

案例2：12306遭遇春节大规模抢票应该如何支撑？

> 12306应该是中国并发量最大的秒杀网站：
>
> 号称并发量100W最高
>
> CDN -> LVS -> NGINX -> 业务系统 -> 每台机器1W并发（单机10K问题-redis能承担住） 100台机器
>
> 普通电商订单 -> 下单 ->订单系统（IO）减库存 ->等待用户付款
>
> 12306的一种可能的模型： 下单 -> 减库存 和 订单(redis kafka) 同时异步进行 ->等付款
>
> 减库存最后还会把压力压到一台服务器
>
> 可以做分布式本地库存 + 单独服务器做库存均衡
>
> 大流量的处理方法：分而治之

怎么得到一个事务会消耗多少内存？

> 1. 弄台机器，看能承受多少TPS？是不是达到目标？扩容或调优，让它达到
> 2. 用压测来确定

## 9.优化环境

1. 有一个50万PV的资料类网站（从磁盘提取文档到内存）原服务器32位，1.5G 的堆，用户反馈网站比较缓慢，因此公司决定升级，新的服务器为64位，16G 的堆内存，结果用户反馈卡顿十分严重，反而比以前效率更低了
   1. 为什么原网站慢? 很多用户浏览数据，很多数据load到内存，内存不足，频繁GC，STW长，响应时间变慢
   2. 为什么会更卡顿？ 内存越大，FGC时间越长
   3. 咋办？ PS  换成  PN + CMS 或者 换成 G1
2. 系统CPU经常100%，如何调优？(面试高频) CPU100%那么一定有线程在占用系统资源，
   1. 找出哪个进程cpu高（top）
   2. 该进程中的哪个线程cpu高（top -Hp）
   3. 导出该线程的堆栈 (jstack)
   4. 查找哪个方法（栈帧）消耗时间 (jstack)
   5. 工作线程占比高 | 垃圾回收线程占比高   使用jstack查找
3. 系统内存飙高，如何查找问题？（面试高频）
   1. 导出堆内存 (jmap)
   2. 分析 (jhat jvisualvm mat jprofiler ... )
4. 如何监控JVM
   1. jstat jvisualvm jprofiler arthas top...

## 10.解决JVM运行中的问题

一个案例理解常用工具

1. 测试代码：

   ```java
   package com.mashibing.jvm.gc;
   
   import java.math.BigDecimal;
   import java.util.ArrayList;
   import java.util.Date;
   import java.util.List;
   import java.util.concurrent.ScheduledThreadPoolExecutor;
   import java.util.concurrent.ThreadPoolExecutor;
   import java.util.concurrent.TimeUnit;
   
   /**
    * 从数据库中读取信用数据，套用模型，并把结果进行记录和传输
    */
   
   public class T15_FullGC_Problem01 {
   
       private static class CardInfo {
           BigDecimal price = new BigDecimal(0.0);
           String name = "张三";
           int age = 5;
           Date birthdate = new Date();
   
           public void m() {}
       }
   
       private static ScheduledThreadPoolExecutor executor = new ScheduledThreadPoolExecutor(50,
               new ThreadPoolExecutor.DiscardOldestPolicy());
   
       public static void main(String[] args) throws Exception {
           executor.setMaximumPoolSize(50);
   
           for (;;){
               modelFit();
               Thread.sleep(100);
           }
       }
   
       private static void modelFit(){
           List<CardInfo> taskList = getAllCardInfo();
           taskList.forEach(info -> {
               // do something
               executor.scheduleWithFixedDelay(() -> {
                   //do sth with info
                   info.m();
   
               }, 2, 3, TimeUnit.SECONDS);
           });
       }
   
       private static List<CardInfo> getAllCardInfo(){
           List<CardInfo> taskList = new ArrayList<>();
   
           for (int i = 0; i < 100; i++) {
               CardInfo ci = new CardInfo();
               taskList.add(ci);
           }
   
           return taskList;
       }
   }
   ```

2. java -Xms200M -Xmx200M -XX:+PrintGC com.mashibing.jvm.gc.T15_FullGC_Problem01

3. 一般是运维团队首先受到报警信息（CPU Memory）

4. top命令观察到问题：内存不断增长 CPU占用率居高不下

5. top -Hp 观察进程中的线程，哪个线程CPU和内存占比高

6. jps定位具体java进程 jstack 定位线程状况，重点关注：WAITING BLOCKED eg. waiting on <0x0000000088ca3310> (a java.lang.Object) 假如有一个进程中100个线程，很多线程都在waiting on  ，一定要找到是哪个线程持有这把锁 怎么找？搜索jstack dump的信息，找 ，看哪个线程持有这把锁RUNNABLE 作业：1：写一个死锁程序，用jstack观察 2 ：写一个程序，一个线程持有锁不释放，其他线程等待

7. 为什么阿里规范里规定，线程的名称（尤其是线程池）都要写有意义的名称 怎么样自定义线程池里的线程名称？（自定义ThreadFactory）

8. jinfo pid

9. jstat -gc 动态观察gc情况 / 阅读GC日志发现频繁GC / arthas观察 / jconsole/ jvisualVM / Jprofiler（最好用） jstat -gc 4655 500 : 每个500个毫秒打印GC的情况 如果面试官问你是怎么定位OOM问题的？如果你回答用图形界面（错误） 1：已经上线的系统不用图形界面用什么？（cmdline arthas）

    2：图形界面到底用在什么地方？测试！测试的时候进行监控！（压测观察）

10. jmap - histo 4655 | head -20，查找有多少对象产生, 稍微有线上影响，但影响不大

11. jmap -dump:format=b,file=xxx pid ： 

    线上系统，内存特别大，jmap导出堆文件执行期间会对进程产生很大影响，甚至卡顿（电商不适合） 

    1：设定了参数HeapDump，OOM的时候会自动产生堆转储文件

    2：很多服务器备份（高可用），停掉这台服务器对其他服务器不影响 

    3：在线定位(一般小点儿公司用不到)

12. java -Xms20M -Xmx20M -XX:+UseParallelGC -XX:+HeapDumpOnOutOfMemoryError com.mashibing.jvm.gc.T15_FullGC_Problem01

13. 使用MAT / jhat /jvisualvm 进行dump文件分析 

     https://www.cnblogs.com/baihuitestsoftware/articles/6406271.html 

     jhat -J-mx512M xxx.dump 

     http://192.168.17.11:7000 

     拉到最后：找到对应链接 可以使用OQL查找特定问题对象

14. 找到代码的问题



## 11.arthas在线排查工具

- 为什么需要在线排查？ 在生产上我们经常会碰到一些不好排查的问题，例如线程安全问题，用最简单的threaddump或者heapdump不好查到问题原因。为了排查这些问题，有时我们会临时加一些日志，比如在一些关键的函数里打印出入参，然后重新打包发布，如果打了日志还是没找到问题，继续加日志，重新打包发布。对于上线流程复杂而且审核比较严的公司，从改代码到上线需要层层的流转，会大大影响问题排查的进度。
- jvm观察jvm信息
- thread定位线程问题
- dashboard 观察系统情况
- heapdump + jhat分析  （jhat -J-mx512 20200104.hprof）
- jad反编译 动态代理生成类的问题定位 第三方的类（观察代码） 版本问题（确定自己最新提交的版本是不是被使用）
- redefine 热替换 目前有些限制条件：只能改方法实现（方法已经运行完成），不能改方法名， 不能改属性 m() -> mm()
- sc  - search class
- watch  - watch method
- 没有包含的功能：jmap



## 12.案例汇总

OOM产生的原因多种多样，有些程序未必产生OOM，不断FGC(CPU飙高，但内存回收特别少) （上面案例）

1. 硬件升级系统反而卡顿的问题，因为内存变大后，FGC要花的时间更多了，所以更卡了（见上）

2. 线程池不当运用产生OOM问题（见上） 不断的往List里加对象（实在太LOW）

3. smile jira问题 

   实际系统不断重启 解决问题 

   加内存 + 更换垃圾回收器 G1

    真正问题在哪儿？不知道

4. tomcat http-header-size过大问题（Hector）

5. lambda表达式导致方法区溢出问题，因为每个lambda表达式都会生成一个Class。(MethodArea / Perm Metaspace) LambdaGC.java     -XX:MaxMetaspaceSize=9M -XX:+PrintGCDetails

   ```java
   "C:\Program Files\Java\jdk1.8.0_181\bin\java.exe" -XX:MaxMetaspaceSize=9M -XX:+PrintGCDetails "-javaagent:C:\Program Files\JetBrains\IntelliJ IDEA Community Edition 2019.1\lib\idea_rt.jar=49316:C:\Program Files\JetBrains\IntelliJ IDEA Community Edition 2019.1\bin" -Dfile.encoding=UTF-8 -classpath "C:\Program Files\Java\jdk1.8.0_181\jre\lib\charsets.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\deploy.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\access-bridge-64.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\cldrdata.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\dnsns.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\jaccess.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\jfxrt.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\localedata.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\nashorn.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\sunec.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\sunjce_provider.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\sunmscapi.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\sunpkcs11.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\zipfs.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\javaws.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\jce.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\jfr.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\jfxswt.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\jsse.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\management-agent.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\plugin.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\resources.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\rt.jar;C:\work\ijprojects\JVM\out\production\JVM;C:\work\ijprojects\ObjectSize\out\artifacts\ObjectSize_jar\ObjectSize.jar" com.mashibing.jvm.gc.LambdaGC
   [GC (Metadata GC Threshold) [PSYoungGen: 11341K->1880K(38400K)] 11341K->1888K(125952K), 0.0022190 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
   [Full GC (Metadata GC Threshold) [PSYoungGen: 1880K->0K(38400K)] [ParOldGen: 8K->1777K(35328K)] 1888K->1777K(73728K), [Metaspace: 8164K->8164K(1056768K)], 0.0100681 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
   [GC (Last ditch collection) [PSYoungGen: 0K->0K(38400K)] 1777K->1777K(73728K), 0.0005698 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
   [Full GC (Last ditch collection) [PSYoungGen: 0K->0K(38400K)] [ParOldGen: 1777K->1629K(67584K)] 1777K->1629K(105984K), [Metaspace: 8164K->8156K(1056768K)], 0.0124299 secs] [Times: user=0.06 sys=0.00, real=0.01 secs] 
   java.lang.reflect.InvocationTargetException
   	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
   	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
   	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
   	at java.lang.reflect.Method.invoke(Method.java:498)
   	at sun.instrument.InstrumentationImpl.loadClassAndStartAgent(InstrumentationImpl.java:388)
   	at sun.instrument.InstrumentationImpl.loadClassAndCallAgentmain(InstrumentationImpl.java:411)
   Caused by: java.lang.OutOfMemoryError: Compressed class space
   	at sun.misc.Unsafe.defineClass(Native Method)
   	at sun.reflect.ClassDefiner.defineClass(ClassDefiner.java:63)
   	at sun.reflect.MethodAccessorGenerator$1.run(MethodAccessorGenerator.java:399)
   	at sun.reflect.MethodAccessorGenerator$1.run(MethodAccessorGenerator.java:394)
   	at java.security.AccessController.doPrivileged(Native Method)
   	at sun.reflect.MethodAccessorGenerator.generate(MethodAccessorGenerator.java:393)
   	at sun.reflect.MethodAccessorGenerator.generateSerializationConstructor(MethodAccessorGenerator.java:112)
   	at sun.reflect.ReflectionFactory.generateConstructor(ReflectionFactory.java:398)
   	at sun.reflect.ReflectionFactory.newConstructorForSerialization(ReflectionFactory.java:360)
   	at java.io.ObjectStreamClass.getSerializableConstructor(ObjectStreamClass.java:1574)
   	at java.io.ObjectStreamClass.access$1500(ObjectStreamClass.java:79)
   	at java.io.ObjectStreamClass$3.run(ObjectStreamClass.java:519)
   	at java.io.ObjectStreamClass$3.run(ObjectStreamClass.java:494)
   	at java.security.AccessController.doPrivileged(Native Method)
   	at java.io.ObjectStreamClass.<init>(ObjectStreamClass.java:494)
   	at java.io.ObjectStreamClass.lookup(ObjectStreamClass.java:391)
   	at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1134)
   	at java.io.ObjectOutputStream.defaultWriteFields(ObjectOutputStream.java:1548)
   	at java.io.ObjectOutputStream.writeSerialData(ObjectOutputStream.java:1509)
   	at java.io.ObjectOutputStream.writeOrdinaryObject(ObjectOutputStream.java:1432)
   	at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1178)
   	at java.io.ObjectOutputStream.writeObject(ObjectOutputStream.java:348)
   	at javax.management.remote.rmi.RMIConnectorServer.encodeJRMPStub(RMIConnectorServer.java:727)
   	at javax.management.remote.rmi.RMIConnectorServer.encodeStub(RMIConnectorServer.java:719)
   	at javax.management.remote.rmi.RMIConnectorServer.encodeStubInAddress(RMIConnectorServer.java:690)
   	at javax.management.remote.rmi.RMIConnectorServer.start(RMIConnectorServer.java:439)
   	at sun.management.jmxremote.ConnectorBootstrap.startLocalConnectorServer(ConnectorBootstrap.java:550)
   	at sun.management.Agent.startLocalManagementAgent(Agent.java:137)
   ```

6. 直接内存溢出问题（少见） 《深入理解Java虚拟机》P59，使用Unsafe分配直接内存，或者使用NIO的问题

7. 栈溢出问题 -Xss设定太小。一个方法不断调用其他方法，一个方法产生一个栈帧，调用深度太高了。解决方法

8. 比较一下这两段程序的异同，分析哪一个是更优的写法：

   ```java
   Object o = null;
   for(int i=0; i<100; i++) {
       o = new Object();
       //业务处理
   }
   //每次新对象出现后，旧对象没有引用指向，就会被回收
   ```

   ```java
   for(int i=0; i<100; i++) {
       Object o = new Object();
   }
   //方法结束就，对象才会回收
   ```

9. 重写finalize引发频繁GC 小米云，HBase同步系统，系统通过nginx访问超时报警，最后排查，C++程序员重写java代码时候把finalize引发频繁GC问题 

   为什么C++程序员会重写finalize？（C++程序员以为需要手动回收内存，new   delete语句，会调用析构函数）

    finalize耗时比较长（200ms） 

10. 如果有一个系统，内存一直消耗不超过10%，但是观察GC日志，发现FGC总是频繁产生，会是什么引起的？ System.gc() (这个比较Low)

11. Distuptor有个可以设置链的长度，如果过大，然后对象大，消费完不主动释放，会溢出 (来自 死物风情)

12. 用jvm都会溢出，mycat用崩过，1.6.5某个临时版本解析sql子查询算法有问题，9个exists的联合sql就导致生成几百万的对象（来自 死物风情）

13. new 大量线程，会产生 native thread OOM，（low）应该用线程池， 解决方案：减少堆空间（太TMlow了）,预留更多内存产生native thread JVM内存占物理内存比例 50% - 80%

## 13 垃圾回收算法回顾

### 1.CMS (concurrent mark sweep) 的四个阶段

初始标记 （inital - mark） --- 并发标记 (concurrent - mark)  --  重新标记（remark）---并发清理 (concurrent - sweep)

1.在初始标记的时候，根据根可达算法，标记根对象的直接引用，此间会有STW  。2.并发标记时，并发的标记其他的对象和引用，3.重新标记，在并发表及期间，有可能会有新的垃圾产生以及被标记的垃圾被重新使用，因此重新标记 4.并发清理，此间还是会有垃圾产生，被称为“浮动辣鸡” 浮动辣鸡的清理在下一个CMS循环重新标记

CMS的问题

1. Memory Fragmentation 内存碎片化

   > -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction 默认为0 指的是经过多少次FGC才进行压缩

2. Floating Garbage 浮动辣鸡

   > Concurrent Mode Failure 产生：if the concurrent collector is unable to finish reclaiming the  unreachable objects before the tenured generation fills up, or if an  allocation cannot be satisfiedwith the available free space blocks in  the tenured generation, then theapplication is paused and the collection is completed with all the applicationthreads stopped
   >
   > 解决方案：降低触发CMS的阈值
   >
   > PromotionFailed
   >
   > 解决方案类似，保持老年代有足够的空间
   >
   > –XX:CMSInitiatingOccupancyFraction 92% 可以降低这个值，让CMS保持老年代足够的空间

### 2.G1 (gabage first)

特点：1.并发收集 2.压缩空闲空间不会延长Gc的暂停时间 3.更易预测的Gc暂停时间 4.适用不需要很高的吞吐量 的场景



### 3.三色标记算法

漏标问题

## 14.各个调优算法的参数汇总

### 1.GC常用参数

- -Xmn -Xms -Xmx -Xss 年轻代 最小堆 最大堆 栈空间
- -XX:+UseTLAB 使用TLAB，默认打开
- -XX:+PrintTLAB 打印TLAB的使用情况
- -XX:TLABSize 设置TLAB大小
- -XX:+DisableExplictGC System.gc()不管用 ，FGC
- -XX:+PrintGC
- -XX:+PrintGCDetails
- -XX:+PrintHeapAtGC
- -XX:+PrintGCTimeStamps
- -XX:+PrintGCApplicationConcurrentTime (低) 打印应用程序时间
- -XX:+PrintGCApplicationStoppedTime （低） 打印暂停时长
- -XX:+PrintReferenceGC （重要性低） 记录回收了多少种不同引用类型的引用
- -verbose:class 类加载详细过程
- -XX:+PrintVMOptions
- -XX:+PrintFlagsFinal  -XX:+PrintFlagsInitial 必须会用
- -Xloggc:opt/log/gc.log
- -XX:MaxTenuringThreshold 升代年龄，最大值15
- 锁自旋次数 -XX:PreBlockSpin 热点代码检测参数-XX:CompileThreshold 逃逸分析 标量替换 ... 这些不建议设置

### 

### 2.Parallel常用参数

- -XX:SurvivorRatio
- -XX:PreTenureSizeThreshold 大对象到底多大
- -XX:MaxTenuringThreshold
- -XX:+ParallelGCThreads 并行收集器的线程数，同样适用于CMS，一般设为和CPU核数相同
- -XX:+UseAdaptiveSizePolicy 自动选择各区大小比例

### 

### 3.CMS常用参数

- -XX:+UseConcMarkSweepGC
- -XX:ParallelCMSThreads CMS线程数量
- -XX:CMSInitiatingOccupancyFraction 使用多少比例的老年代后开始CMS收集，默认是68%(近似值)，如果频繁发生SerialOld卡顿，应该调小，（频繁CMS回收）
- -XX:+UseCMSCompactAtFullCollection 在FGC时进行压缩
- -XX:CMSFullGCsBeforeCompaction 多少次FGC之后进行压缩
- -XX:+CMSClassUnloadingEnabled
- -XX:CMSInitiatingPermOccupancyFraction 达到什么比例时进行Perm回收
- GCTimeRatio 设置GC时间占用程序运行时间的百分比
- -XX:MaxGCPauseMillis 停顿时间，是一个建议时间，GC会尝试用各种手段达到这个时间，比如减小年轻代

### 

### 4.G1常用参数

- -XX:+UseG1GC
- -XX:MaxGCPauseMillis 建议值，G1会尝试调整Young区的块数来达到这个值
- -XX:GCPauseIntervalMillis ？GC的间隔时间
- -XX:+G1HeapRegionSize 分区大小，建议逐渐增大该值，1 2 4 8 16 32。 随着size增加，垃圾的存活时间更长，GC间隔更长，但每次GC的时间也会更长 ZGC做了改进（动态区块大小）
- G1NewSizePercent 新生代最小比例，默认为5%
- G1MaxNewSizePercent 新生代最大比例，默认为60%
- GCTimeRatio GC时间建议比例，G1会根据这个值调整堆空间
- ConcGCThreads 线程数量
- InitiatingHeapOccupancyPercent 启动G1的堆空间占用比例

## 15.线程和纤程