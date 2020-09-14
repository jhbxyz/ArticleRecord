# Hilt

* 什么是依赖？

* 什么是依赖注入？
* Java 中的依赖注入



### 1.什么是依赖？

先看下面代码

#### 1.1.首先声明一个 Student 类

```java
public class Student {
    String name;
    int age;

    public void study() {
        System.out.println("学习...");
    }
}
```

> 有两个属性（成员变量）：name 和 age
>
> 一个动作（方法）：study 方法

#### 1.1.2.声明一个 People，在构造方法中，实例化一个 Student 的对象

```java
public class People {

    private final Student student;

    public People() {
        student = new Student();
    }
}
```

看 People 类，**其中 student 的就是 People 的依赖**。

> 嗯？这个 Student 就是 People 的依赖了？
>
> 对，通俗来讲，就是我需要用到这个对象的属性方法，就是我依赖这个对象。

#### 1.1.3.总结依赖的含义

如果在 Class A 中，有 Class B 的实例，则称 Class A 对 Class B 有一个依赖。

> 在例子中类 People 中用到一个 Student 对象，我们就说类 People 对类 Student 有一个依赖。

### 2.什么是依赖注入？

还是上面例子，不过 People 的代码变了

```java
public class People {

    private final Student student;

    public People(Student student) {
        this.student = student;
    }
}
```

Student 的对象，不是由 People 这个类直接初始化的，而是由外部初始化好，传给People 的，这个就是依赖注入了。

> People 还是依赖于 Student 这个没有变
>
> 但是 People 并没有初始化 Student，而是外部传给 People 的，这个就是依赖注入了

**总结：**非自己主动初始化依赖，而通过外部来传入依赖的方式，我们就称为依赖注入。

那依赖注入有什么好处呢？

**依赖注入的好处**：解耦。

> 由于是外部实例化好 Student 对象，而非在 People 内部初始化，People 只是拿一个 Student 的引用

### 3.Java 中的依赖注入

```java
public class People {
    @Inject
    Student student2;

}
```

如果你只是写了一个 @Inject 注解，Student 并不会被自动注入。你还需要使用一个依赖注入框架，并进行简单的配置。





















