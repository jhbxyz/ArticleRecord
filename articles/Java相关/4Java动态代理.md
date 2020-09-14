# Java 中的动态代理

### 1.什么是代理

在某些情况下，我们不希望或是不能直接访问对象 A，而是通过访问一个中介对象 B，由 B 去访问 A 达成目的，这种方式我们就称为代理。

这里对象 A 所属类我们称为**委托类**，也称为**被代理类**，对象 B 所属类称为**代理类**。

**代理优点**

- 隐藏委托类的实现
- 解耦，不改变委托类代码情况下做一些额外处理，比如添加初始判断及其他公共操作

> 根据程序运行前代理类是否已经存在，可以将代理分为**静态代理**和**动态代理**。

### 2.静态代理

**含义**：代理类在程序**运行前**已经存在的代理方式称为静态代理。

被代理类（委托类）ClassA

```java
public class ClassA {

    public void method1() {

    }

    public void method2() {

    }

    public void method3() {

    }
}
```

代理类 ClassB

```java
public class ClassB {
    private ClassA classA;

    public ClassB(ClassA classA) {
        this.classA = classA;
    }

    public void method1() {
        classA.method1();
    }

    public void method2() {
        classA.method2();

    }
}
```

上面`ClassA`是委托类，`ClassB`是代理类，`ClassB`中的函数都是直接调用`ClassA`相应函数，并且隐藏了`Class`的`method3()`函数。

> 静态代理中代理类和委托类也常常继承同一父类或实现同一接口。

### 3.动态代理

**含义**：代理类在程序运行前不存在、运行时由程序动态生成的代理方式称为动态代理。

**优点**:

> Java 提供了动态代理的实现方式，可以在运行时刻动态生成代理类。

* 可以方便对代理类的函数做统一或特殊处理
* 如记录所有函数执行时间
* 所有函数执行前添加验证判断
* 对某个特殊函数进行特殊操作
* 不用像静态代理方式那样需要修改每个函数。





















