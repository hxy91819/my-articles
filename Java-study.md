<!-- TOC -->

- [1. 前言](#1-%E5%89%8D%E8%A8%80)
- [2. 异步编程](#2-%E5%BC%82%E6%AD%A5%E7%BC%96%E7%A8%8B)
    - [多线程](#%E5%A4%9A%E7%BA%BF%E7%A8%8B)
    - [2.1. notify() 和wait()](#21-notify-%E5%92%8Cwait)
    - [2.2. 锁](#22-%E9%94%81)
        - [2.2.1. 对象监视器monitor](#221-%E5%AF%B9%E8%B1%A1%E7%9B%91%E8%A7%86%E5%99%A8monitor)
        - [2.2.2. 死锁、锁饥饿](#222-%E6%AD%BB%E9%94%81%E3%80%81%E9%94%81%E9%A5%A5%E9%A5%BF)
        - [2.2.3. synchronized](#223-synchronized)
        - [2.2.4. ReentrantLock](#224-reentrantlock)
    - [2.3. 多线程控制（可见性）](#23-%E5%A4%9A%E7%BA%BF%E7%A8%8B%E6%8E%A7%E5%88%B6%EF%BC%88%E5%8F%AF%E8%A7%81%E6%80%A7%EF%BC%89)
        - [2.3.1. Semaphore信号量](#231-semaphore%E4%BF%A1%E5%8F%B7%E9%87%8F)
        - [2.3.2. CountDownLatch](#232-countdownlatch)
- [3. String](#3-string)
- [4. JVM](#4-jvm)
- [5. Java enum 的本质](#5-java-enum-%E7%9A%84%E6%9C%AC%E8%B4%A8)
- [6. 泛型](#6-%E6%B3%9B%E5%9E%8B)
- [7. Bean 复制 与 对象 Clone](#7-bean-%E5%A4%8D%E5%88%B6-%E4%B8%8E-%E5%AF%B9%E8%B1%A1-clone)
- [Log4j2](#log4j2)
- [fastjson](#fastjson)

<!-- /TOC -->

# 1. 前言

本文为我在学习Java过程中的笔记。

# 2. 异步编程

## 多线程

线程的特点是共享地址空间，从而可以高效地共享数据。

如果要完成的任务是CPU密集型的，那多线程没有优势，甚至因为线程切换的开销，多线程反而更慢；如果要完成的任务既有CPU计算，又有磁盘或网络IO，则使用多线程的好处是，当某个线程因为IO而阻塞时，OS可以调度其他线程执行，虽然效率确实要比任务的顺序执行效率要高，然而，这种类型的任务，可以通过单线程的”non-blocking IO+IO multiplexing”的模型（事件驱动）来提高效率，采用多线程的方式，带来的可能仅仅是编程上的简单而已。

Thread就是异步编程（通常都需要使用线程池）

想要获取线程执行结果：Future（从jdk1.5开始）
- 替代品guava：com.google.common.util.concurrent.ListenableFuture

想要多个线程分别执行，同时阻塞主线程：java.util.concurrent.CountDownLatch
- 更复杂的工具：java.util.concurrent.CyclicBarrier。可以重置计数器

## 2.1. notify() 和wait()

我自己的一篇文章：
[Java 异步编程之：notify 和 wait 用法](https://segmentfault.com/a/1190000018096174)

## 2.2. 锁

锁用于对代码做控制，以避免多个线程竞争同一个资源而出现冲突。

### 2.2.1. 对象监视器monitor

使用Synchronized进行同步，其关键就是必须要对对象的监视器monitor进行获取，当线程获取monitor后才能继续往下执行，否则就只能等待。

任意一个对象都拥有自己的监视器，当这个对象由同步块或者这个对象的同步方法调用时，执行方法的线程必须先获取该对象的监视器才能进入同步块和同步方法，如果没有获取到监视器的线程将会被阻塞在同步块和同步方法的入口处，进入到BLOCKED状态

### 2.2.2. 死锁、锁饥饿

### 2.2.3. synchronized

其内部实际上是一个可重入锁。可以对一个方法进行修饰，执行改类的方法时，将会加锁。甚至也可以对一个代码块进行加锁。

### 2.2.4. ReentrantLock

相比于synchronized，ReentrantLock在功能上更加丰富，它具有可重入、可中断、可限时、公平锁等特点。

可重入：

```java
lock.lock();
lock.lock();
try
{
    i++;
            
}           
finally
{
    lock.unlock();
    lock.unlock();
}
```

由于ReentrantLock是重入锁，所以可以反复得到相同的一把锁，它有一个与锁相关的获取计数器，如果拥有锁的某个线程再次得到锁，那么获取计数器就加1，然后锁需要被释放两次才能获得真正释放(重入锁)。

可中断：

可以在某种情况下中断锁（例如死锁的时候）。
使用`lock.lockInterruptibly();`替代`lock.lock()`即可，使得线程因此lock死锁时，可以进行`Thread.interrupt()`

可限时：

- 超时不能获得锁，就返回false，不会永久等待构成死锁
- 使用`lock.tryLock(long timeout, TimeUnit unit)`来实现可限时锁，参数为时间和单位。

公平锁：

- 一般意义上的锁是不公平的，不一定先来的线程能先得到锁，后来的线程就后得到锁。不公平的锁可能会产生饥饿现象。
- 公平锁的意思就是，这个锁能保证线程是先来的先得到锁。虽然公平锁不会产生饥饿现象，但是公平锁的性能会比非公平锁差很多。

## 2.3. 多线程控制（可见性）

### 2.3.1. Semaphore信号量

多个线程间无法直接沟通数据。所以JDK提供了一批工具方便我们在线程之上控制多个线程的执行。

信号量的使用类似 lock。设定一个池子，可以限制线程能够最大以此池子的数量进行执行。最大化利用线程。

### 2.3.2. CountDownLatch

倒计时。在线程中通过逻辑控制，在外部主线程可以在倒计时结束后，得到响应。

# 3. String
`intern()` 方法
- 当调用 intern 方法时，如果池已经包含一个等于此 String 对象的字符串（该对象由 equals(Object) 方法确定），则返回池中的字符串。否则，将此 String 对象添加到池中，并且返回此 String 对象的引用。
- intern()本身最大的作用就是加快String的比较（因为可以使用 == ，速度能够提高三倍）。但是这个特性看起来很强大，So What？

# 4. JVM
类加载机制：ClassLoader（不熟悉）

# 5. Java enum 的本质
static代码块
- 类加载的时候调用。何为类加载？
    - ”.class”文件中保存着Java代码经转换后的虚拟机指令，当需要使用某个类时，虚拟机将会加载它的”.class”文件，并创建对应的class对象，将class文件加载到虚拟机的内存，这个过程称为类加载
- Java类中类似的还有：构造代码块。一个类中各类代码执行顺序为：（静态变量、静态代码块）>（变量、构造代码块）>构造函数。

继承 Enum
- Java 是单继承，所以 enum 不能再继承别的类
- enum 实质上是用final修饰的class，所以不能被继承
- 枚举之间可以比较，比较的是其再枚举中的序号
- 枚举是单例的，是实现单例模式的一种方式
    - 单例模式，不允许使用者直接 new 对象，需要使用类的 method 获取已经创建好的对象。
    - 单例模式实现方式，最经典的利用 ClassLoader机制避免了多线程的同步问题
        - ClassLoader翻译过来就是类加载器，普通的java开发者其实用到的不多，但对于某些框架开发者来说却非常常见。理解ClassLoader的加载机制，也有利于我们编写出更高效的代码。ClassLoader的具体作用就是将class文件加载到jvm虚拟机中去，程序就可以正确运行了。但是，jvm启动的时候，并不会一次性加载所有的class文件，而是根据需要去动态加载
        - 何为线程安全？
        - 单例模式的具体应用？

# 6. 泛型

泛型在Java1.5的时候加入

`<T extends Serializable>` 中写法是用来限定泛型的类型，必须继承接口`Serializable`
- 类似的还有`<T super Serializable>`：下界通配符（Lower Bounds Wildcards）。两者均是为了给泛型设定边界。下界通配符是用来限定为指定类和指定类的父类。
- 参考文章：[Java 泛型 <? super T> 中 super 怎么 理解？与 extends 有何不同？](https://www.zhihu.com/question/20400700)

# 7. Bean 复制 与 对象 Clone

Bean 复制和对象Clone其实很类似，至少笔者看不出太大区别。Object的clone()方法就是一个用来实现对象clone的方法。

进行对象克隆有两种方式：
- 实现Cloneable接口并重写Object类中的clone()方法；
- 实现Serializable接口，通过对象的序列化和反序列化实现克隆，可以实现真正的深度克隆，代码如下。

另外，在Apache Common Utils里也提供了Bean复制的方法，比较强大，但是实测其进行复制的效率要低于对象的序列化和反序列化实现克隆。

在字段比较少，且对性能要求很高的领域，尽量不要使用Bean复制的方法。参考：[Java Bean Copy框架性能对比](https://yq.aliyun.com/articles/392185)


# Log4j2

- 异步打印日志，效率较高。
- 系统的System.out等日志会被忽略；程序抛出的异常日志会被忽略。如果需要捕获，需要手动try...catch...然后log.error打印

# fastjson

json序列化工具。很常用。

- 被序列化的类，在从json转换为对象的时候，需要有无参数的构造方法，否则报异常。

