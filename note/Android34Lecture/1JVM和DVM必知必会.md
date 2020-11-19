# Day1程序运行时，内存到底是如何进行分配的？

下面这张图描述了一个 HelloWorld.java 文件被 JVM 加载到内存中的过程：

<img src="https://raw.githubusercontent.com/jhbxyz/ArticleRecord/master/note/Android34Lecture/images/1-1.png" >

## 1.程序计数器（Program Counter Register）

Java 程序是多线程的，CPU 可以在多个线程中分配执行时间片段。当某一个线程被 CPU 挂起时，需要记录代码已经执行到的位置，**方便 CPU 重新执行此线程时，知道从哪行指令开始执行**。这就是程序计数器的作用。

“程序计数器”是虚拟机中一块较小的内存空间

* 主要用于记录当前**线程执行的位置**。
* 分支操作、循环操作、跳转、异常处理等也都需要依赖这个计数器来完成。

#### 关于程序计数器还有几点需要格外注意：

* 在 Java 虚拟机规范中，对程序计数器这一区域**没有规定**任何 OutOfMemoryError 情况（或许是感觉没有必要吧）。
* **线程私有**的，每条线程内部都有一个私有程序计数器。它的生命周期随着线程的创建而创建，随着线程的结束而死亡。
* 当一个线程正在执行一个 Java 方法的时候，这个计数器记录的是正在执行的虚拟机字节码指令的地址。如果正在执行的是 Native 方法，这个计数器值则为空（Undefined）。

## 2.虚拟机栈

虚拟机栈也是线程私有的，与线程的生命周期同步。在 Java 虚拟机规范中，对这个区域规定了两种异常状况：

* StackOverflowError：当线程请求栈深度超出虚拟机栈所允许的深度时抛出。
* OutOfMemoryError：当 Java 虚拟机动态扩展到无法申请足够内存时抛出。

JVM 是基于栈的解释器执行的，DVM 是基于寄存器解释器执行的。

上面这句话里的“基于栈”指的就是**虚拟机栈**。虚拟机栈的初衷是用来描述 Java 方法执行的内存模型，每个方法被执行的时候，JVM 都会在**虚拟机栈**中创建一个**栈帧**。

### 2.1.栈帧

栈帧（Stack Frame）是用于支持虚拟机进行**方法调用和方法执行**的数据结构，每一个线程在执行某个方法时，都会为这个**方法**创建一个栈帧。

一个线程包含多个栈帧，而每个栈帧内部包含局部变量表、操作数栈、动态连接、返回地址等

![1-2](https://github.com/jhbxyz/ArticleRecord/blob/master/note/Android34Lecture/images/1-2.png?raw=true)

#### 2.1.1.局部变量表

局部变量表是变量值的存储空间，我们调用方法时传递的参数，以及在方法内部创建的**局部变量**都**保存**在局部变量表中

```java
public static int add(int k) {
	int i = 1;
	int j = 2;
	return i + j + k;
}
```

> 变量 i 、j和 k 都是局部变量，存在局部变量表中
>
> **注意**：系统不会为局部变量赋予初始值（实例变量（成员变量）和类变量（静态变量）都会被赋予初始值），也就是说不存在类变量那样的准备阶段。

#### 2.1.2.操作数栈

操作数栈（Operand Stack）也常称为操作栈，它是一个后入先出栈（LIFO）。

当一个方法刚刚开始执行的时候，这个方法的操作数栈是空的。在方法执行的过程中，会有各种字节码指令被压入和弹出操作数栈

#### 2.1.3.动态链接

动态链接的主要目的是为了支持方法调用过程中的动态连接（Dynamic Linking）。

在一个 class 文件中，一个方法要调用其他方法，需要将这些方法的符号引用转化为其所在内存地址中的直接引用，而符号引用存在于**方法区**中。

#### 2.1.4.返回地址

当一个方法开始执行后，只有两种方式可以退出这个方法：

- **正常退出**：指方法中的代码正常完成，或者遇到任意一个方法返回的字节码指令（如return）并退出，没有抛出任何异常。
- **异常退出**：指方法执行过程中遇到异常，并且这个异常在方法体内部没有得到处理，导致方法退出。

无论当前方法采用何种方式退出，在方法退出后都需要返回到方法被调用的位置，程序才能继续执行。而虚拟机栈中的“返回地址”就是用来帮助当前方法恢复它的上层方法执行状态。

## 3.本地方法栈

本地方法栈和上面介绍的虚拟栈基本相同，只不过是针对本地（native）方法。在开发中如果涉及 JNI 可能接触本地方法栈多一些，在有些虚拟机的实现中已经将两个合二为一了（比如**HotSpot**）。

## 4.堆

Java 堆（Heap）是 JVM 所管理的内存中最大的一块，该区域唯一目的就是存放对象实例，几乎所有对象的实例都在堆里面分配，因此它也是 Java 垃圾收集器（GC）管理的主要区域，有时候也叫作“GC 堆”，同时它也是所有线程**共享**的内存区域，因此被分配在此区域的对象如果被多个线程访问的话，需要**考虑线程安全**问题。

按照对象存储时间的不同，堆中的内存可以划分为**新生代**（Young）和**老年代**（Old），其中新生代又被划分为 Eden 和 Survivor 区。具体如下图所示：

![1-1](https://github.com/jhbxyz/ArticleRecord/blob/master/note/Android34Lecture/images/1-3.png?raw=true)

> 图中不同的区域存放具有不同生命周期的对象。这样可以根据**不同的区域使用不同的垃圾回收算法**，从而更具有针对性，进而提高垃圾回收效率。

## 5.方法区

方法区（Method Area）也是 JVM 规范里规定的一块运行时数据区。方法区主要是存储已经被 JVM 加载的**类信息**（版本、字段、方法、接口）、**常量、静态变量**、即时编译器编译后的代码和数据。该区域**同堆**一样，也是被各个线程**共享的**内存区域。

## 6.异常再现

### 6.1.StackOverflowError 栈溢出异常

**递归**调用是造成StackOverflowError的一个常见场景

```java
public class StackOver {
    private int number;

    public static void main(String[] args) {
        try {
            final StackOver stackOver = new StackOver();
            stackOver.method();
        } catch (StackOverflowError e) {
            e.printStackTrace();
            System.out.println("栈内存溢出");
        }
    }

    private void method() {
        number++;
        method();
    }
}
```

原因就是每调用一次method方法时，都会在虚拟机栈中创建出一个栈帧。因为是递归调用，method方法并不会退出，也不会将栈帧销毁，所以必然会导致StackOverflowError。因此当需要使用递归时，需要格外谨慎。

### 6.2.OutOfMemoryError 内存溢出异常

理论上，虚拟机**栈、堆、方法区**都有发生OutOfMemoryError的可能。但是实际项目中，大多发生于堆当中。

```java
public class HeapError {
    public static void main(String[] args) {
        final ArrayList<HeapError> heapErrors = new ArrayList<>();
        while (true) {
            heapErrors.add(new HeapError());
        }
    }
}
```

在一个无限循环中，动态的向ArrayList中添加新的HeapError对象。这会不断的占用堆中的内存，当堆内存不够时，必然会产生OutOfMemoryError，也就是内存溢出异常。

## 7.总结

![1-1](https://github.com/jhbxyz/ArticleRecord/blob/master/note/Android34Lecture/images/1-4.png?raw=true)

总结来说，JVM 的运行时内存结构中一共有两个“栈”和一个“堆”，分别是：Java 虚拟机栈和本地方法栈，以及“GC堆”和方法区。除此之外还有一个程序计数器，但是我们开发者几乎不会用到这一部分，所以并不是重点学习内容。 **JVM 内存中只有堆和方法区是线程共享的数据区域，其它区域都是线程私有的**。并且程序计数器是唯一一个在 Java 虚拟机规范中没有规定任何 OutOfMemoryError 情况的区域。



# Day2 GC 回收机制与分代回收策略

Java 内存运行时区域的各个部分，其中**程序计数器、虚拟机栈、本地方法栈 3 个区域随线程而生，随线程而灭**；栈中的栈帧随着方法的进入和退出而有条不紊地执行着出栈和入栈操作，这几个区域内不需要过多考虑回收的问题。

**堆和方法区**则不一样，一个接口中的多个实现类需要的内存可能不一样，一个方法中的多个分支需要的内存也可能不一样，我们只有在程序处于运行期间时才能知道会创建哪些对象，这部分内存的分配和回收都是动态的，**垃圾收集器所关注**的就是这部分内存。

## 1.什么是垃圾

所谓垃圾就是内存中已经**没有用的对象**。 既然是”垃圾回收"，那就必须知道哪些对象是垃圾。Java 虚拟机中使用一种叫作"**可达性分析”**的算法来决定对象是否可以被回收。

> 没有被 GC Root引用的对象，就是垃圾

## 2.可达性分析

”GC Root"的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，最后通过判断对象的引用链是否可达来决定对象是否可以被回收

![img](https://github.com/jhbxyz/ArticleRecord/blob/master/note/Android34Lecture/images/2-1.png?raw=true)

> 上图中圆形图标虽然标记的是对象，但实际上代表的是此对象在内存中的引用。包括 GC Root 也是一组引用而并非对象。

**没有被 GC Root 引用的对象，就是垃圾**

### 2.1.GC Root 对象

在 Java 中，有以下几种对象可以作为 GC Root：

- Java 虚拟机栈（局部变量表）中的引用的对象。
- 方法区中**静态**引用指向的对象。
- 仍处于**存活**状态中的**线程**对象。
- **Native** 方法中 JNI 引用的对象。

> 全局变量（成员变量）同静态变量不同，它不会被当作 GC Root。

### 2.2.什么时候回收

不同的虚拟机实现有着不同的 GC 实现机制，但是一般情况下每一种 GC 实现都会在以下两种情况下触发垃圾回收。

- Allocation Failure：在堆内存中分配时，如果因为可用剩余空间不足导致**对象内存分配失败**，这时系统会触发一次 GC。
- System.gc()：在应用层，Java 开发工程师可以**主动调用**此 API 来请求一次 GC。

## 3.如何回收垃圾

### 3.1.标记清除算法（Mark and Sweep GC）

从”GC Roots”集合开始，将内存**整个遍历**一次，保留所有可以被 GC Roots 直接或间接引用到的对象，而剩下的对象都当作垃圾对待并回收，过程分两步。

- Mark 标记阶段：找到内存中的所有 GC Root 对象，只要是和 GC Root 对象直接或者间接相连则标记为灰色（也就是存活对象），否则标记为黑色（也就是垃圾对象）。
- Sweep 清除阶段：当遍历完所有的 GC Root 之后，则将标记为垃圾的对象直接清除。

![img](https://github.com/jhbxyz/ArticleRecord/blob/master/note/Android34Lecture/images/2-2.png?raw=true)

- 优点：实现简单，不需要将对象进行移动。
- 缺点：这个算法需要**中断进程**内**其他组件的执行**（stop the world），并且**可能产生内存碎片**，提高了垃圾回收的频率（会造成频繁回收）。

### 3.2.复制算法（Copying）

将现有的内存空间分为两快，每次只使用其中一块，在垃圾回收时将正在使用的内存中的存活对象复制到未被使用的内存块中。之后，清除正在使用的内存块中的所有对象，交换两个内存的角色，完成垃圾回收。

1.复制算法之前，内存分为 A/B 两块，并且当前只使用内存 A，内存的状况如下图所示：

![img](https://github.com/jhbxyz/ArticleRecord/blob/master/note/Android34Lecture/images/2-3.png?raw=true)

2.标记完之后，所有可达对象都被按次序复制到内存 B 中，并设置 B 为当前使用中的内存。内存状况如下图所示：

![img](https://github.com/jhbxyz/ArticleRecord/blob/master/note/Android34Lecture/images/2-4.png?raw=true)

- 优点：按顺序分配内存即可，实现简单、运行高效，不用考虑内存碎片。
- 缺点：可用的内存大小缩小为原来的一半，**对象存活率高**时会频繁进行复制。

### 3.3.标记-压缩算法 (Mark-Compact)

需要先从根节点开始对所有可达对象做一次标记，之后，它并不简单地清理未标记的对象，而是将所有的存活对象压缩到内存的一端。最后，清理边界外所有的空间。因此标记压缩也分两步完成：

1.Mark 标记阶段：找到内存中的所有 GC Root 对象，只要是和 GC Root 对象直接或者间接相连则标记为灰色（也就是存活对象），否则标记为黑色（也就是垃圾对象）。

2.Compact 压缩阶段：将剩余存活对象按顺序压缩到内存的某一端。

![img](https://github.com/jhbxyz/ArticleRecord/blob/master/note/Android34Lecture/images/2-5.png?raw=true)

- 优点：这种方法既避免了碎片的产生，又不需要两块相同的内存空间，因此，其性价比比较高。
- 缺点：所谓压缩操作，仍需要进行局部对象移动，所以一定程度上还是降低了效率。

## 4.JVM分代回收策略

Java 虚拟机根据对象存活的周期不同，把堆内存划分为几块，一般分为新生代、老年代，这就是 JVM 的内存分代策略。**注意: 在 HotSpot 中除了新生代和老年代，还有永久代**

分代回收的中心思想就是：对于新创建的对象会在新生代中分配内存，此区域的对象生命周期一般较短。如果经过**多次回收仍然存活下来**，则将它们转移到老年代中。

### 4.1.年轻代（Young Generation）

采用的 GC 回收算法是**复制算法**。

新生成的对象优先存放在新生代中，新生代对象朝生夕死，存活率很低，在新生代中，常规应用进行一次垃圾收集一般**可以回收 70%~95% 的空间**，回收效率**很高**。

新生代又可以继续细分为 3 部分：Eden、Survivor0（简称 S0）、Survivor1（简称S1）。这 3 部分按照 **8:1:1** 的比例来划分新生代。

**绝大多数刚刚被创建的对象会存放在 Eden 区。**

> **新生代的剩余空间不足**，则这个大对象会**直接被分配**到老年代上

![img](https://github.com/jhbxyz/ArticleRecord/blob/master/note/Android34Lecture/images/2-6.png?raw=true)

当 **Eden** 区第一次满的时候，会进行垃圾回收（**复制算法**）。首先将 Eden区的垃圾对象回收清除，并将存活的对象复制到 S0，此时 S1是空的

![img](https://github.com/jhbxyz/ArticleRecord/blob/master/note/Android34Lecture/images/2-7.png?raw=true)

**下一次 Eden** 区满时，再执行一次垃圾回收。此次会将 Eden和 S0区中所有垃圾对象清除，并将存活对象复制到 S1，此时 S0变为空。

![img](https://github.com/jhbxyz/ArticleRecord/blob/master/note/Android34Lecture/images/2-8.png?raw=true)

如此反复在 **S0 和 S1之间切换几次**（默认 15 次）之后，如果还有存活对象。说明这些对象的生命周期较长，则将它们转移到老年代中。

![img](https://github.com/jhbxyz/ArticleRecord/blob/master/note/Android34Lecture/images/2-9.png?raw=true)

### 4.2.年老代（Old Generation）

一般采用**标记压缩**的回收算法。

一个对象如果在新生代**存活了足够长的时间**而没有被清理掉，则会被复制到老年代。老年代的内存大小一般比新生代大，能存放更多的对象。如果对象比较大（比如长字符串或者大数组），并且**新生代的剩余空间不足**，则这个大对象会**直接被分配**到老年代上。

> 注意：对于老年代可能存在这么一种情况，老年代中的对象有时候会引用到新生代对象。这时如果要执行新生代 GC，则可能需要查询整个老年代上可能存在引用新生代的情况，这显然是低效的。所以，老年代中维护了一个 512 byte 的 card table，所有老年代对象引用新生代对象的信息都记录在这里。每当新生代发生 GC 时，只需要检查这个 card table 即可，大大提高了性能。

## 5.GC Log 分析

新生代和老年代所打印的日志是有区别的。

- 新生代 GC：这一区域的 GC 叫作 Minor GC。因为 Java 对象大多都具备朝生夕灭的特性，所以 Minor GC 非常频繁，一般回收速度也比较快。
- 老年代 GC：发生在这一区域的 GC 也叫作 Major GC 或者 Full GC。当出现了 Major GC，经常会伴随至少一次的 Minor GC。

> 注意：在有些虚拟机实现中，Major GC 和 Full GC 还是有一些区别的。Major GC 只是代表回收老年代的内存，而 Full GC 则代表回收整个堆中的内存，也就是新生代 + 老年代。

## 6.再谈引用

判断对象是否存活我们是通过GC Roots的**引用可达性**来判断的。但是JVM中的**引用关系并不止一种**，而是有四种，根据引用强度的由强到弱，他们分别是:强引用(Strong Reference)、软引用(Soft Reference)、弱引用(Weak Reference)、虚引用(Phantom Reference)。

![img](https://github.com/jhbxyz/ArticleRecord/blob/master/note/Android34Lecture/images/2-10.png?raw=true)

重点看下软引用SoftReference的使用，**不当的使用软引用有时也会导致系统异常**。



# Day3字节码层面分析 class 类文件结构



## 1.上帝视角看 class 文件

class 文件里只有两种数据结构：无符号数和表。

- 无符号数：属于基本的数据类型，以 u1、u2、u4、u8 来分别代表 1 个字节、2 个字节、4 个字节和 8 个字节的无符号数，无符号数可以用来描述数字、索引引用、数量值或者字符串（UTF-8 编码）。
- 表：表是由多个无符号数或者其他表作为数据项构成的复合数据类型，**class文件中所有的表都以“_info”结尾。**其实，整个 Class 文件本质上就是一张表。

![img](/Users/mac/jhb_projects/ArticleRecord/note/Android34Lecture/images/3-1.png)

## 2.class 文件结构

 class 文件中只存在无符号数和表这两种数据结构，这些无符号数和表就组成了 class 中的各个结构。这些结构按照预先规定好的顺序紧密的从前向后排列，相邻的项之间没有任何间隙。

![img](/Users/mac/jhb_projects/ArticleRecord/note/Android34Lecture/images/3-2.png)

### 2.1.魔数 magic number

 class 文件开头的四个字节是 class 文件的魔数，它是一个固定的值--0XCAFEBABE。魔数是 class 文件的标志。

> 证明这个文件是 class 文件，能被 JVM 识别

### 2.2.常量池（重点）

在常量池中保存了**类的各种相关信息**，比如类的名称、父类的名称、类中的方法名、参数名称、参数类型等，这些信息都是以各种表的形式保存在常量池中的。







# Day4编译插桩操纵字节码，实现不可能完成的任务



## 1.编译插桩是什么

就是在代码编译期间修改已有的代码或者生成新代码。

> Dagger、ButterKnife 甚至是 Kotlin 语言，它们都用到了编译插桩的技术。

Android 项目中 .java 文件的编译过程：

![img](/Users/mac/jhb_projects/ArticleRecord/note/Android34Lecture/images/4-1.png)

我们可以在 1、2 两处对代码进行改造。

- 在 .java 文件编译成 .class 文件时，APT、AndroidAnnotation 等就是在此处触发代码生成。

- 在 .class 文件进一步优化成 .dex 文件时，也就是直接操作字节码文件

  > 这种方式功能更加强大，应用场景也更多。但是门槛比较高，需要对字节码有一定的理解。

红色虚框，就是编译插桩的过程

![img](/Users/mac/jhb_projects/ArticleRecord/note/Android34Lecture/images/4-2.png)

一般情况下，我们经常会使用编译插桩实现如下几种功能：

- 日志埋点；
- 性能监控；
- 动态权限控制；
- 业务逻辑跳转时，校验是否已经登录；
- 甚至是代码调试等。

## 2.插桩工具介绍

目前市面上主要流行两种实现编译插桩的方式：

### AspectJ

AspectJ 是老牌 AOP（Aspect-Oriented Programming）框架，如果你做过 J2EE 开发可能对这个框架更加熟悉，经常会拿这个框架跟 Spring AOP 进行比较。其主要优势是成熟稳定，使用者也不需要对字节码文件有深入的理解。

### ASM

目前另一种编译插桩的方式 ASM 越来越受到广大工程师的喜爱。通过 ASM 可以修改现有的字节码文件，也可以动态生成字节码文件，并且它是一款完全以字节码层面来操纵字节码并分析字节码的框架（此处可以联想一下写汇编代码时的酸爽）。

使用 ASM 来实现简单的编译插桩效果，通过插桩实现课时开始讲的需求，在每一个 Activity 打开时输出相应的 log 日志。

### 2.1.实现思路

- 遍历项目中所有的 .class 文件

  > 我们可以自己定义 Transform，来获取所有 .class 文件引用。但是 Transform 的使用需要依赖 Gradle Plugin。因此我们第一步需要创建一个单独的 Gradle Plugin，并在 Gradle Plugin 中使用自定义 Transform 找出所有的 .class 文件。

- 遍历到目标 .class 文件 （Activity）之后，通过 ASM 动态注入需要被插入的字节码

  > 接下来就需要过滤出目标 Activity 文件，并在目标 Activity 文件的 onCreate 方法中，通过 ASM 插入相应的 log 日志字节码。



# Day5深入理解 ClassLoader 的加载机制



## 1.Java 中的类何时被加载器加载

在 Java 程序启动的时候，并不会一次性加载程序中所有的 .class 文件，而是在程序的运行过程中，动态地加载相应的类到内存中。

Java 程序中的 .class 文件会在以下 2 种情况下被 ClassLoader 主动加载到内存中：

- 调用类构造器
- 调用类中的静态（static）变量或者静态方法

## 2.Java 中 ClassLoader

JVM 中自带 3 个类加载器：

- 启动类加载器 BootstrapClassLoader

  > 是由 C/C++ 语言编写的，它本身属于虚拟机的一部分，我们无法在 Java 代码中直接获取它的引用
  >
  > 加载系统属性“sun.boot.class.path”配置下类文件
  >
  > Jdk1.8.0/Contents/Home/jre ，JRE 目录下的 jar 包或者 .class 文件。

- 扩展类加载器 ExtClassLoader （JDK 1.9 之后，改名为 PlatformClassLoader）

  > ExtClassLoader 加载系统属性“java.ext.dirs”配置下类文件
  >
  > Jdk1.8.0/Contents/Home/jre/lib/ext 下面的文件

- 系统加载器 APPClassLoader

  > AppClassLoader 主要加载系统属性“java.class.path”配置下类文件，也就是环境变量 CLASS_PATH 配置的路径。
  >
  > 因此 AppClassLoader 是面向用户的类加载器，
  >
  > 我们**自己编写的代码**以及使用的**第三方 jar 包**通常都是由它来加载的。

### 3.双亲委派模式

JVM 中已经有了这 3 种 ClassLoader，那么 JVM 又是**如何知道该使用哪一个类加载器去加载相应的类呢**？答案就是：双亲委派模式。

#### 定义

所谓双亲委派模式就是，当类加载器收到加载类或资源的请求时，通常都是先委托给父类加载器加载，也就是说，只有当父类加载器找不到指定类或资源时，自身才会执行实际的类加载过程。

```java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
        // First, check if the class has already been loaded
        //1
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                if (parent != null) {
                		//2
                    c = parent.loadClass(name, false);
                } else {
                		//3
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                //4
                c = findClass(name);
            }
        }
        return c;
}
```

- 判断该 Class 是否已加载，如果已加载，则直接将该 Class 返回。
- 如果该 Class 没有被加载过，则判断 parent 是否为空，如果不为空则将加载的任务委托给parent。
- 如果 parent == null，则直接调用 BootstrapClassLoader 加载该类。
- 如果 parent 或者 BootstrapClassLoader 都没有加载成功，则调用当前 ClassLoader 的 findClass 方法继续尝试加载。

那这个 parent 是什么呢?

在每一个 ClassLoader 中都有一个 CLassLoader 类型的 parent 引用，并且在构造器中传入值。如果我们继续查看源码，可以看到 AppClassLoader 传入的 parent 就是 ExtClassLoader，而 ExtClassLoader 并没有传入任何 parent，也就是 null。

#### 举例说明

比如执行以下代码：

```java
Test test = new Test();
```

默认情况下，JVM 首先使用 AppClassLoader 去加载 Test 类。

- AppClassLoader 将加载的任务委派给它的父类加载器（parent）—ExtClassLoader。
- ExtClassLoader 的 parent 为 null，所以直接将加载任务委派给 BootstrapClassLoader。
- BootstrapClassLoader 在 jdk/lib 目录下无法找到 Test 类，因此返回的 Class 为 null。
- 因为 parent 和 BootstrapClassLoader 都没有成功加载 Test 类，所以AppClassLoader会调用自身的 findClass 方法来加载 Test。

> Test 的 ClassLoader 为 AppClassLoader 类型，而 AppClassLoader 的 parent 为 ExtClassLoader 类型。ExtClassLoader 的 parent 为 null。

### 4.Android 中的 ClassLoader

在 Android 虚拟机里是无法直接运行 .class 文件的，Android 会将所有的 .class 文件转换成一个 .dex 文件，并且 Android 将加载 .dex 文件的实现封装在 **BaseDexClassLoader** 中，而我们一般只使用它的两个子类：PathClassLoader 和 DexClassLoader。

#### PathClassLoader 

PathClassLoader 用来加载系统 apk 和被安装到手机中的 apk 内的 dex 文件。

#### DexClassLoader

对比 PathClassLoader 只能加载已经安装应用的 dex 或 apk 文件，DexClassLoader 则没有此限制，可以从 SD 卡上加载包含 class.dex 的 .jar 和 .apk 文件，这也是插件化和热修复的基础，在不需要安装应用的情况下，完成需要使用的 dex 的加载。

![img](/Users/mac/jhb_projects/ArticleRecord/note/Android34Lecture/images/5-1.png)

- dexPath：包含 class.dex 的 apk、jar 文件路径 ，多个路径用文件分隔符（默认是“:”）分隔。
- optimizedDirectory：用来缓存优化的 dex 文件的路径，即从 apk 或 jar 文件中提取出来的 dex 文件。该路径不可以为空，且应该是应用私有的，有读写权限的路径。





# Day6Class 对象在执行引擎中的初始化过程

一个 class 文件被加载到内存中需要经过 3 大步：装载、链接、初始化。其中链接又可以细分为：验证、准备、解析 3 小步。

![img](/Users/mac/jhb_projects/ArticleRecord/note/Android34Lecture/images/6-1.png)

## 1.装载

装载是指 Java 虚拟机查找 .class 文件并生成字节流，然后根据字节流创建 java.lang.Class 对象的过程。

### 这一过程主要完成以下 3 件事：

1）ClassLoader 通过一个类的全限定名（包名 + 类名）来查找 .class 文件，并生成二进制字节流：其中 class 字节码文件的来源不一定是 .class 文件，也可以是 jar 包、zip 包，甚至是来源于网络的字节流。

2）把 .class 文件的各个部分分别解析（parse）为 JVM 内部特定的数据结构，并存储在**方法区**。

3）在内存中创建一个 java.lang.Class 类型的对象：接下来程序在运行过程中所有对该类的访问都通过这个类对象，也就是这个 Class 类型的类对象是提供给外界访问该类的接口。

### 加载时机

以下两种情况一般会对 class 进行装载操作。

- 隐式装载：在程序运行过程中，当碰到通过 new 等方式生成对象时，系统会隐式调用 ClassLoader 去装载对应的 class 到内存中；
- 显示装载：在编写源代码时，主动调用 Class.forName() 等方法也会进行 class 装载操作，这种方式通常称为显示装载。

## 2.链接

链接过程分为 3 步：验证、准备、解析。

### 验证：

验证是链接的第一步，目的是为了确保 .class 文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危及虚拟机本身的安全。主要包含以下几个方面的检验。

- 文件格式检验：检验字节流是否符合 class 文件格式的规范，并且能被当前版本的虚拟机处理。 
- 元数据检验：对字节码描述的信息进行语义分析，以保证其描述的内容符合 Java 语言规范的要求。 
- 字节码检验：通过数据流和控制流分析，确定程序语义是合法、符合逻辑的。 
- 符号引用检验：符号引用检验可以看作是对类自身以外（常量池中的各种符号引用）的信息进行匹配性校验。

### 准备

这一阶段的主要目的是为类中的静态变量分配内存，并为其设置“0值”

```java
public static int value = 100;
```

在准备阶段，JVM 会为 value 分配内存，并将其设置为 0。而真正的值 100 是在初始化阶段设置。

有一种情况比较特殊--静态常量，

```java
public static final int value = 100;
```

> 以上代码会在准备阶段就为 value 分配内存，并设置为 100。



Java 中基本类型的默认”0值“如下：

- 基本类型（int、long、short、char、byte、boolean、float、double）的默认值为 0；
- 引用类型默认值是 null；

### 解析

这一阶段的任务是把常量池中的符号引用转换为直接引用，也就是具体的**内存地址**。

## 3.初始化

这是 class 加载的最后一步，这一阶段是执行类构造器<clinit>方法的过程，并真正初始化类变量。比如：

```java
public static int value = 100;
```

在准备阶段 value 被分配内存并设置为 0，在初始化阶段 value 就会被设置为 100。

### 初始化的时机

对于装载阶段，JVM 并没有规范何时具体执行。但是对于初始化，JVM 规范中严格规定了 class 初始化的时机，主要有以下6种情况会触发 class 的初始化：

- 虚拟机启动时，初始化包含 main 方法的主类；
- 遇到 new 指令创建对象实例时，如果目标对象类没有被初始化则进行初始化操作；
- 当遇到访问静态方法或者静态字段的指令时，如果目标对象类没有被初始化则进行初始化操作；
- 子类的初始化过程如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化；
- 使用反射 API 进行反射调用时，如果类没有进行过初始化则需要先触发其初始化；
- 第一次调用 java.lang.invoke.MethodHandle 实例时，需要初始化 MethodHandle 指向方法所在的类。

### 初始化类变量（静态变量）

在初始化阶段，只会初始化与类相关的静态赋值语句和静态语句，也就是有 static 关键字修饰的信息，而没有 static 修饰的语句块在**实例化对象**的时候才会执行。

![img](/Users/mac/jhb_projects/ArticleRecord/note/Android34Lecture/images/6-2.png)

### 被动引用

上述的 6 种情况在 JVM 中被称为主动引用，除此 6 种情况之外所有引用类的方式都被称为被动引用。被动引用并**不会触发** class 的初始化。

最典型的就是在子类中调用父类的静态变量

![img](/Users/mac/jhb_projects/ArticleRecord/note/Android34Lecture/images/6-3.png)

可以看出 Child 继承自 Parent 类，如果直接使用 Child 来访问 Parent 中的 value 值，**则不会初始化 Child 类**

对于静态字段，只有直接定义这个字段的类才会被初始化，因此通过子类 Child 来引用父类 Parent 中定义的静态字段，只会触发父类 Parent 的初始化而不会触发子类 Child 的初始化。至于是否要触发子类的加载和验证，在虚拟机规范中并未明确规定

### class 初始化和对象的创建顺序

![img](/Users/mac/jhb_projects/ArticleRecord/note/Android34Lecture/images/6-4.png)

在 main 方法中执行了 2 次 new Child() 的操作，执行上述代码结果如下：

![img](/Users/mac/jhb_projects/ArticleRecord/note/Android34Lecture/images/6-5.png)

总结一下对象的初始化顺序如下：

静态变量/静态代码块 -> 普通代码块 -> 构造函数

1. 父类静态变量和静态代码块；
2. 子类静态变量和静态代码块；
3. 父类普通成员变量和普通代码块；
4. 父类的构造函数；
5. 子类普通成员变量和普通代码块；
6. 子类的构造函数。





# Day7Java 内存模型与线程

## 1.Java 内存模型

内存模型是一套共享内存系统中多线程读写操作行为的规范，这套规范屏蔽了底层各种硬件和操作系统的内存访问差异，解决了 CPU 多级缓存、CPU 优化、指令重排等导致的内存访问问题，从而保证 Java 程序（尤其是多线程程序）在各种平台下对内存的访问效果一致。



## 2.happens-before 先行发生原则

happens-before 用于描述两个操作的内存可见性，通过保证可见性的机制可以让应用程序免于数据竞争干扰。

## 3.总结

- Java 内存模型的来源：主要是因为 CPU 缓存和指令重排等优化会造成多线程程序结果不可控。
- Java 内存模型是什么：本质上它就是一套规范，在这套规范中有一条最重要的 happens-before 原则。
- volatile 和 synchronized 关键字来实现 happens-before 原则



# Day8既生 Synchronized，何生 ReentrantLock

# 1.synchronized

### 1.1.synchronized 可以用来修饰以下 3 个层面：

- 修饰实例方法；（成员方法）

  > 锁对象是当前实例对象

- 修饰静态类方法；（静态方法）

  > 锁对象是当前类的字节码文件对象

- 修饰代码块。

  > 锁对象就是跟在后面括号中的对象

### 1.2.实现细节

#### 使用 synchronized 作用于代码块：

编译而成的字节码中会包含 monitorenter 和 monitorexit 这两个字节码指令

上面字节码中有 1 个 monitorenter 和 2 个 monitorexit。这是因为虚拟机需要保证当异常发生时也能释放锁。因此 2 个 monitorexit 一个是代码正常执行结束后释放锁，一个是在代码执行异常时释放锁。

#### synchronized 修饰方法有哪些区别：

被 synchronized 修饰的方法在被编译为字节码后，在方法的 flags 属性中会被标记为 ACC_SYNCHRONIZED 标志。当虚拟机访问一个被标记为 ACC_SYNCHRONIZED 的方法时，**会自动**在方法的开始和结束（或异常）位置添加 monitorenter 和 monitorexit 指令。

### 1.3.关于 monitorenter 和 monitorexit

可以理解为一把具体的锁。在这个锁中保存着两个比较重要的属性：计数器和指针。

- 计数器代表当前线程一共访问了几次这把锁；
- 指针指向持有这把锁的线程。

锁计数器默认为0，当执行monitorenter指令时，如锁计数器值为0 说明这把锁并没有被其它线程持有。那么这个线程会将计数器加1，并将锁中的指针指向自己。当执行monitorexit指令时，会将计数器减1。



## 2.ReentrantLock

### 2.1.ReentrantLock 基本使用

ReentrantLock 的使用同 synchronized 有点不同，它的加锁和解锁操作都需要手动完成

ReentrantLock 也能实现同 synchronized 相同的效果。

ReentrantLock 的使用中，将 unlock 操作放在 finally 代码块中。这是因为 ReentrantLock 与 synchronized 不同，当异常发生时 synchronized 会自动释放锁，但是 ReentrantLock 并不会自动释放锁。因此好的方式是将 unlock 操作放在 finally 代码块中，保证任何时候锁都能够被正常释放掉。 

### 2.2.公平锁实现

```java
ReentrantLock lock = new ReentrantLock(true);
```

默认情况下，synchronized 和 ReentrantLock 都是**非公平锁**。但是 ReentrantLock 可以通过传入 true 来创建一个公平锁。所谓公平锁就是通过同步队列来实现多个线程按照**申请锁的顺序获取锁**。

> 排队去拿锁，挨个执行任务，第一个执行一次，第二个执行，然后第一个在执行



### 2.3.读写锁（ReentrantReadWriteLock）

读锁，在读的时候，都可以读





# Day9Java 线程优化 偏向锁，轻量级锁、重量级锁

synchronized 是**重量级锁**

## 1.synchronized 实现原理

要了解 synchronized 的原理需要先理清楚两件事情：对象头和 Monitor。

 Java 对象在内存中的布局分为 3 部分：对象头、实例数据、对齐填充

### 对象头



### Monitor

Monitor 可以把它理解为一个同步工具，也可以描述为一种同步机制。实际上，它是一个保存在对象头中的一个对象。

*  EntrySet ：等待拿锁区域
* Owner ：持有锁
* WaitSet：调用 wait() 方法后，进入这个集合



## 2.Java 虚拟机对 synchronized 的优化

 锁自旋、轻量级锁、偏向锁。

### 2.1.锁自旋

所谓自旋，就是让该线程等待一段时间，不会被立即挂起，看当前持有锁的线程是否会很快释放锁。而所谓的等待就是**执行一段无意义的循环**即可（自旋）。

> 自旋锁也存在一定的缺陷：
>
> - 自旋锁要占用 CPU，如果锁竞争的时间比较长，那么自旋通常不能获得锁，白白浪费了自旋占用的 CPU 时间。
> - 这通常发生在锁持有时间长，且竞争激烈的场景中，此时应主动禁用自旋锁。

### 2.2.轻量级锁

 CAS（Compare And Swap）

> 轻量级锁所适应的场景是线程交替执行同步块的场合，如果存在同一时间访问同一锁的场合，就会导致轻量级锁膨胀为重量级锁。



### 2.3.偏向锁

偏向锁的意思是如果一个线程获得了一个偏向锁，如果在接下来的一段时间中没有其他线程来竞争锁，那么持有偏向锁的线程再次进入或者退出同一个同步代码块，不需要再次进行抢占锁和释放锁的操作。

> 其实偏向锁并不适合所有应用场景, 因为一旦出现锁竞争，偏向锁会被撤销，并膨胀成轻量级锁，而撤销操作（revoke）是比较重的行为，只有当存在较多不会真正竞争的 synchronized 块时，才能体现出明显改善；因此实践中，还是需要考虑具体业务场景，并测试后，再决定是否开启/关闭偏向锁。



# Day10深入理解 AQS 和 CAS 原理

AQS 全称是 Abstract Queued Synchronizer，一般翻译为同步器。它是一套实现多线程同步功能的框架

AQS 在源码中被广泛使用，尤其是在 JUC（Java Util Concurrent）中，比如 **ReentrantLock**、**Semaphore**、**CountDownLatch**、**ThreadPoolExecutor**。



CAS 全称是 Compare And Swap，译为比较和替换，是一种通过硬件实现并发安全的常用技术，底层通过利用 CPU 的 CAS 指令对缓存加锁或总线加锁的方式来实现多处理器之间的原子操作。

AQS 有两种不同的实现：独占锁（ReentrantLock 等）和分享锁（CountDownLatch、读写锁等）



# Day11线程池之刨根问底

线程的创建需要**开辟虚拟机栈、本地方法栈、程序计数器**等线程私有的内存空间，在线程销毁时需要回收这些系统资源，频繁地创建销毁线程会浪费大量资源

在 阿里Java开发手册 中已经严禁使用 Executors 来创建线程池，这是为什么呢

## 1.流程解析

1.当前线程池中运行的线程数量还没有达到 corePoolSize 大小时，线程池会创建一个新线程执行提交的任务，无论之前创建的线程是否处于空闲状态。

2.当前线程池中运行的线程数量已经达到 corePoolSize 大小时，线程池会把任务加入到等待队列中，直到某一个线程空闲了，线程池会根据我们设置的等待队列规则，从队列中取出一个新的任务执行

3.如果线程数大于 corePoolSize 数量但是还没有达到最大线程数 maximumPoolSize，并且等待队列已满，则线程池会创建新的线程来执行任务。

4.最后如果提交的任务，无法被核心线程直接执行，又无法加入等待队列，又无法创建“非核心线程”直接执行，线程池将根据拒绝处理器定义的策略处理这个任务。比如在 ThreadPoolExecutor 中，如果你没有为线程池设置 RejectedExecutionHandler。这时线程池会抛出 RejectedExecutionException 异常，即线程池拒绝接受这个任务。

## 2.为何禁止使用 Executors

为何在阿里 Java 开发手册中严禁使用 Executors 工具类来创建线程池。尤其是 newFixedThreadPool 和 newCachedThreadPool 这两个方法。

- newFixedThreadPool 

  > ```java
  > new LinkedBlockingQueue<Runnable>()//相当于不限制任务的数量，一直排队添加
  > ```
  >
  > 传入的是一个无界的阻塞队列，理论上可以无限添加任务到线程池。当核心线程执行时间很长（比如 sleep10s），则新提交的任务还在不断地插入到阻塞队列中，最终造成
  > OOM。

- newCachedThreadPool 

  > 缓存线程池的最大线程数为 Integer 最大值。当核心线程耗时很久，线程池会尝试创建新的线程来执行提交的任务，当内存不足时就会报无法创建线程的错误。





# Day12 DVM 以及 ART 是如何对 JVM 进行优化的？

## 1.什么是 Dalvik

Dalvik 是 Google 公司自己设计用于 Android 平台的 Java 虚拟机，在 Android 5.0 之前叫作 **DVM**，5.0 之后改为 **ART**（Android Runtime）。



Android 系统的第一个 Dalvik 虚拟机是由 Zygote 进程创建的，而应用程序进程是由 Zygote 进程 fork 出来的。

Zygote 进程是在系统启动时产生的，它会完成虚拟机的初始化，库的加载，预置类库的加载和初始化等操作，而在系统需要一个新的虚拟机实例时，Zygote 通过复制自身，最快速的提供一个进程；

























