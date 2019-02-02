# 异步编程

Thread就是异步编程（通常都需要使用线程池）

想要获取线程执行结果：Future（从jdk1.5开始）
- 替代品guava：com.google.common.util.concurrent.ListenableFuture

想要多个线程分别执行，同时阻塞主线程：java.util.concurrent.CountDownLatch
- 更复杂的工具：java.util.concurrent.CyclicBarrier。可以重置计数器

# String
`intern()` 方法
- 当调用 intern 方法时，如果池已经包含一个等于此 String 对象的字符串（该对象由 equals(Object) 方法确定），则返回池中的字符串。否则，将此 String 对象添加到池中，并且返回此 String 对象的引用。
- intern()本身最大的作用就是加快String的比较（因为可以使用 == ，速度能够提高三倍）。但是这个特性看起来很强大，So What？

# JVM
类加载机制：ClassLoader（不熟悉）

# Java enum 的本质
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

# 泛型

泛型在Java1.5的时候加入

`<T extends Serializable>` 中写法是用来限定泛型的类型，必须继承接口`Serializable`
- 类似的还有`<T super Serializable>`：下界通配符（Lower Bounds Wildcards）。两者均是为了给泛型设定边界。下界通配符是用来限定为指定类和指定类的父类。
- 参考文章：[Java 泛型 <? super T> 中 super 怎么 理解？与 extends 有何不同？](https://www.zhihu.com/question/20400700)






