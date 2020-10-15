# Day1程序运行时，内存到底是如何进行分配的？



下面这张图描述了一个 HelloWorld.java 文件被 JVM 加载到内存中的过程：

![1-1](https://github.com/jhbxyz/ArticleRecord/blob/master/note/Android34Lecture/images/1-1.png?raw=true)

## 1.程序计数器（Program Counter Register）

Java 程序是多线程的，CPU 可以在多个线程中分配执行时间片段。当某一个线程被 CPU 挂起时，需要记录代码已经执行到的位置，方便 CPU 重新执行此线程时，知道从哪行指令开始执行。这就是程序计数器的作用。

“程序计数器”是虚拟机中一块较小的内存空间，

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

上面这句话里的“基于栈”指的就是**虚拟机栈**。虚拟机栈的初衷是用来描述 Java 方法执行的内存模型，每个方法被执行的时候，JVM 都会在虚拟机栈中创建一个**栈帧**。

### 2.1.栈帧

栈帧（Stack Frame）是用于支持虚拟机进行方法调用和方法执行的数据结构，每一个线程在执行某个方法时，都会为这个方法创建一个栈帧。

一个线程包含多个栈帧，而每个栈帧内部包含局部变量表、操作数栈、动态连接、返回地址等

![img](/Users/mac/jhb_projects/ArticleRecord/note/Android34Lecture/images/1-2.png)

#### 2.1.1.局部变量表

局部变量表是变量值的存储空间，我们调用方法时传递的参数，以及在方法内部创建的局部变量都保存在局部变量表中

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

本地方法栈和上面介绍的虚拟栈基本相同，只不过是针对本地（native）方法。在开发中如果涉及 JNI 可能接触本地方法栈多一些，在有些虚拟机的实现中已经将两个合二为一了（比如HotSpot）。

## 4.堆

Java 堆（Heap）是 JVM 所管理的内存中最大的一块，该区域唯一目的就是存放对象实例，几乎所有对象的实例都在堆里面分配，因此它也是 Java 垃圾收集器（GC）管理的主要区域，有时候也叫作“GC 堆”，同时它也是所有线程**共享**的内存区域，因此被分配在此区域的对象如果被多个线程访问的话，需要**考虑线程安全**问题。

按照对象存储时间的不同，堆中的内存可以划分为**新生代**（Young）和**老年代**（Old），其中新生代又被划分为 Eden 和 Survivor 区。具体如下图所示：

![img](/Users/mac/jhb_projects/ArticleRecord/note/Android34Lecture/images/1-3.png)

> 图中不同的区域存放具有不同生命周期的对象。这样可以根据不同的区域使用不同的垃圾回收算法，从而更具有针对性，进而提高垃圾回收效率。



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

![img](/Users/mac/jhb_projects/ArticleRecord/note/Android34Lecture/images/1-4.png)



总结来说，JVM 的运行时内存结构中一共有两个“栈”和一个“堆”，分别是：Java 虚拟机栈和本地方法栈，以及“GC堆”和方法区。除此之外还有一个程序计数器，但是我们开发者几乎不会用到这一部分，所以并不是重点学习内容。 JVM 内存中只有堆和方法区是线程共享的数据区域，其它区域都是线程私有的。并且程序计数器是唯一一个在 Java 虚拟机规范中没有规定任何 OutOfMemoryError 情况的区域。

