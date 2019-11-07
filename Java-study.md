<!-- TOC -->autoauto- [1. 前言](#1-前言)auto- [2. 异步编程](#2-异步编程)auto    - [多线程](#多线程)auto    - [2.1. notify() 和wait()](#21-notify-和wait)auto    - [2.2. 锁](#22-锁)auto        - [2.2.1. 对象监视器monitor](#221-对象监视器monitor)auto        - [2.2.2. 死锁、锁饥饿](#222-死锁锁饥饿)auto        - [2.2.3. synchronized](#223-synchronized)auto        - [2.2.4. ReentrantLock](#224-reentrantlock)auto    - [2.3. 多线程控制（可见性）](#23-多线程控制可见性)auto        - [2.3.1. Semaphore信号量](#231-semaphore信号量)auto        - [2.3.2. CountDownLatch](#232-countdownlatch)auto- [3. String](#3-string)auto- [4. JVM](#4-jvm)auto- [5. Java enum 的本质](#5-java-enum-的本质)auto- [6. 泛型](#6-泛型)auto- [7. Bean 复制 与 对象 Clone](#7-bean-复制-与-对象-clone)auto- [Log4j2](#log4j2)auto- [fastjson](#fastjson)auto- [跑批任务注意事项](#跑批任务注意事项)auto- [@Transaction与动态代理](#transaction与动态代理)auto- [HashMap和HashSet](#hashmap和hashset)auto- [fastjson](#fastjson-1)auto- [编码经验](#编码经验)autoauto<!-- /TOC -->

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

# 跑批任务注意事项

- 一些周期较长的（比如一周、一个月）执行一次的跑批任务，其中如果有http请求这种网络IO操作，网络IO一旦超时，可能会导致跑批任务失败。
    - 为此，需要增加跑批任务补偿机制，在跑批任务失败的时候，延迟一段时间再次进行跑批。
- Spring的@Transaction是有超时时间的，如果事务执行的时间过长，可能会导致失败，笔者的一个事务里面，还有HTTP请求，导致整个任务时间过长，超时回滚。
    - 当然，对于本身执行时间很长的数据库操作，可以在@Transaction里面设置超时时间。
``` 
Error updating database.  Cause: com.mysql.jdbc.exceptions.jdbc4.MySQLTransactionRollbackException: Lock wait timeout exceeded; try restarting transactio
```
- 使用了类似dubbo的RPC框架，通常是Controller-->dubbo service的定时任务。如果service速度过慢，可能会导致dubbo触发重发逻辑
    - 关闭dubbo重发逻辑（不推荐，需要配置很多东西，人为处理容易出错）
        - <dubbo:reference interface="com.xxx.interface.MyFacade" id="myFacade" check="false" timeout="1200000"/>
        - <dubbo:reference interface="com.xxx.interface.MyFacade" id="myFacade" check="false" retries="-1"/>
    - dubbo的provider需要及时响应用户，无论是使用多线程处理任务，还是使用MQ异步处理任务。但是要小心，如果provider里面还调用了其他的dubbo服务，简单的使用异步处理仍旧不能根本解决问题，此问题的解法时，一定需要找到耗时严重的根本位置。
    - 通常我们使用facade包装dubbo开放出去的方法，一般这个方法都会使用aop去拦截，做一些统计、日志等处理。涉及到调用链时，尽量不要循环调用facade。特别是微服务内部调用，禁止调用自己的facade。


在一些跑批任务的时候，我们往往会通过dubbo查询大量的数据，但是由于dubbo自带了超时的机制，一旦数据量过大，可能会因为超时，导致流程中断。解决方案：
- 一个大而全的RPC方法，虽然可以减少远程调用，但是也要同时考虑到这个方法能否在超时时间内返回，这个方法的数据量，是否会随着业务的增长而不断变大
- 将数据分片执行
- 但是在使用guava的Lists.partition方法进行分片的时候，要注意，得到的List是没有序列化的。如果要调用dubbo这样的远程服务，需要使用new ArrayList(List)进行一次包装
- 数据进行分片之后，需要考虑数据去重问题，因为有些方法在一起调用的时候，会对整体做去重操作，特别是当方法返回Map和Set的时候。


周期性的任务，建议在Spring中写好Job。一次性的任务，可以使用linux的[at服务](https://blog.csdn.net/a1414345/article/details/74857226)

# @Transaction与动态代理

@Transaction是一个实现事务的注解。其修饰的方法可以在执行之前开始事务，在结束之后提交事务。用起来非常方便。

但是在实际使用中，这个注解有时候会遇到没有生效的情况，因此查找了资料，想知道为什么：https://vitzhou.top/20180822_transation/

@Transaction这个注解管理着事务的传递，如果一个方法A调用方法B，方法A标记了这个注解，但是方法B没有，假设方法A和方法B中均有数据库操作，那么方法A将会开启一个事务，但是方法B不会参与这个事务，并且受到事务隔离级别的影响。
即，如果方法A插入了一条数据，方法B是无法查询到这条数据的。

## 动态代理

看了之后，发现不得不去深入了解其实现的内部原理——动态代理。

Spring是通过动态代理来实现AOP，进而实现了方便地事务处理。而动态代理是设计模式的一种。在了解它之前，我们需要下了解什么是静态代理：https://juejin.im/post/5a99048a6fb9a028d5668e62

静态代理很简单，就是在**不改变被调用的方法的内部逻辑**的基础上，实现方法的前后逻辑增补，并提供一个同样的调用方式给外部。

这里提供一个**同样的调用方式**，显然是告诉我们，代理类和被代理的类需要实现同一个接口（当然这不是必须的，最好是这样）。

而静态代理，需要没每一种代理逻辑提供不同的静态代码，会导致代码非常繁琐，重复写了很多相似的逻辑。

这时候高级的程序员就开始动脑筋了：

1. 代理方法的逻辑都是类似的，我们能不能把这部分相似的代码抽象出来，封装起来，然后不同的地方让用户自己去实现呢？
2. 静态代理需要为每个代理逻辑创建不同的类，我们能不能不预先创建好，而是在调用的时候动态地创建呢？

这两个思路非常重要，正是体现了面向对象编程的高内聚、低耦合的思想。具体的操作思路可以见：https://juejin.im/post/5a99048a6fb9a028d5668e62

JDK自己提供了动态代理的方法：java.lang.reflect.Proxy#newProxyInstance，可以动态地为对象创建一个代理对象，但是它限制了对象必须实现和代理对象相同的接口。

而[CGLIB](https://github.com/cglib/cglib)甚至可以在不实现同一个接口的情况下提供代理类。

恭喜你，看到这里你已经触及了Spring AOP的实现原理了。感兴趣的话，进一步深入了解吧。本文的关注的重点还是动态代理这个设计模式的思想：

> 将相同的逻辑封装起来，剩下的不相同的逻辑，设计好接口，让调用者自己去实现。

掌握了这个思路，或许不会减少你当下的工作量，也不会一下子让你当下无法解决的问题得到解决。但是它可以让你设计出更加可复用的代码，可以让你的代码实现更加优雅，更加易于重构。或者更简单的认为，可以让你变得更加强大。

在JDK中，也有很多API设计上使用了这个思路，在JDK 8的Stream API中，大量的filter、map等方法，要求我们传入一个函数式接口，例如`java.util.function.Predicate`等，他让我们在每次调用的时候，可以临时（或者可以说：动态）地指定一个处理逻辑，同样是用到了这个思想。

说点其他地感想，之前看到过一篇文章[别再学习框架了！](https://sizovs.net/2018/12/17/stop-learning-frameworks/)

框架太多了，但是我们要找到他们的共同点深入学习，掌握好基础，才能够以不变应万变。

但是笔者认为文章有点标题党，作者的导师在文中告诉作者：

> 技术变化无定，但有很多共同点。确定好优先级。将 80% 的学习时间投入到基础知识上。剩下的 20% 用于框架、库和工具，反正你在工作中解决问题的时候会学习它们。

框架是前人智慧的精华，通过他们了解他们的设计思路，我们可以学习到他们用到了什么优秀的基础知识。在工作中，不学习框架说的并不是要你不求甚解，只要会用就行了，而是要你能够理解框架背后的通用的原理。

不过作者也没错，并不是每个人都像他一样，研读了各大主流的框架呢。

# HashMap和HashSet

在某些业务逻辑中，因集和的顺序不同可能会产生不同结果的时候，如果适用HashMap或者HashSet，可能会出现数据完全相同，但是产生的结果不同的问题。

# fastjson

解决map的key为数字的时候，生成的json非标准格式的问题（标准json的key，不能为数字，必须为字符串）

`JSON.toJSONString(autoScheduleParamDTO.getParamDTO(), SerializerFeature.WriteNonStringKeyAsString)`

使用`SerializerFeature.WriteNonStringKeyAsString`即可

# 编码经验

在编写大型项目的时候，自顶向下开发。当遇到依赖子模块的地方时，先编写子模块的接口，暂时不去实现，先把主流程开发完。

好处是：
- 主流程在开发中，可能会出现接口调整情况，可以避免实现的逻辑要修改
- 主流程与业务最接近，这样进行开发可以更加贴近业务，避免一些没有意义的方法实现。而是将编码的方向转为“我需要实现什么目标？”

# 先插后查，先插入后查询。联动操作无法正确执行的问题。

## 情况一

如下图代码。writerDao和readerDao是不同的DataSource，即使使用了`@Transactional`，也不可能共用一个事务。

所以writerDao插入的数据还没有提交，另外一个数据源对这个数据进行查询一定是查不到的。

事务只能够在同一个数据源里使用。（否则需要考虑使用“分布式事务”，这里不讨论）

这里想要查询到，必须不使用@Transactional。由此可见，@Transactional的使用应当特别注意，必须使用事务的代码，限制在小模块里。如果需要查询到之前代码插入的数据，必须确保事务已经提交。可以重构为C（不使用`@Transactional`）调用B（使用了`@Transactional`），然后B完成了再调用A。

```
@Override
@Transactional
public void insert() {
    UserMsgInfo userMsgInfo = new UserMsgInfo();
    long id = idGen.next().id;
    userMsgInfo.setId(id);
    userMsgInfo.setAppCode("test");
    userMsgInfo.setMsgCode("test");
    int i = userMsgInfoWriterDao.insertSelective(userMsgInfo);
    UserMsgInfo userMsgInfo1 = userMsgInfoReaderDao.selectByPrimaryKey(id);
    userMsgInfoTestService.select(id);
}
```

## 情况二

```
@Override
public void insert() {
    UserMsgInfo userMsgInfo = new UserMsgInfo();
    long id = idGen.next().id;
    userMsgInfo.setId(id);
    userMsgInfo.setAppCode("test");
    userMsgInfo.setMsgCode("test");
    int i = userMsgInfoWriterDao.insertSelective(userMsgInfo);
    UserMsgInfo userMsgInfo1 = userMsgInfoReaderDao.selectByPrimaryKey(id);
    userMsgInfoTestService.select(id);
}
```

还是上面的例子，不使用`@Transactional`，也可能存在无法查询到的问题。常见的是writerDao和readerDao连接的数据源分别指向了不同的数据库。通常是主库和从库。主从之间需要进行数据同步，我们对从库的查询在同步完成之前就执行了，当然无法查询到数据。

解决这个问题有两个方法：

方法一：如果必须要查到，就使用同一个数据源，并且连接到同一个数据库
方法二：writerDao在执行方法后，必须阻塞程序执行，并确保所有从库均完成了数据同步才响应。此时能够确保readerDao无论查询哪个从库都可以查到数据。我们现在使用的galera cluster就是这种方案。