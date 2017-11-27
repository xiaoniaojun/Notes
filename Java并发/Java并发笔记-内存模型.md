# 第三章 Java内存模型

## 3.1 Java内存模型基础

### 3.1.1 并发编程模型的两个关键问题

线程之间如何通信以及线程之间如何同步。

Java采用共享内存的并发模型，特点是线程之间的通信是隐式的，但同步则是非隐式的，需要程序员显示处理。

### 3.1.2 Java内存模型的抽象结构

共享变量指堆内存中的变量：实例域、静态域和数组对象。

Java中的线程具有抽象的本地内存，对共享内存的修改首先会写入本地内存，由本地内存刷新到共享内存中，再由其他线程从共享内存中读取。这个过程实质是线程A向线程B发送消息，而这个消息必须通过共享内存来完成。

JMM的作用就是控制主内存和每个线程的本地内存的交互，来提供内存可见性保证。

### 3.1.3 指令重排序

从Java源码到最终执行的执行序列，指令会经历3种重排序：1.编译器优化 2. 执行级并行重排序 3.内存系统重排序，但Java会确保在不同平台上提供一致的内存可见性。

### 3.1.4并发编程模型的分类

由于写缓存区的存在，以及它只对它所在的处理器可见。因此这个特性会对内存操作的执行顺序产生影响：
处理器对内存的读/写操作，不一定与实际发生的读/写操作顺序一致。

因此Java编译器为了保证内存可减小，会在生成的指令序列中的适当位置处插入内存屏障指令来禁止特性的处理器重排序。

### 3.1.5 happens-before

JDK1.5之后，Java开始使用心得JSR-133内存模型。

如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须要存在happens-before关系。

我们需要了解的4种happens-before的存在规则有：
 程序顺序规则：一个线程中的每个操作，happens- before 于该线程中的任意后续操作。
 监视器锁规则：对一个锁的解锁，happens- before 于随后对这个锁的加锁。
 volatile 变量规则：对一个 volatile 域的写，happens- before 于任意后续对这个 volatile域的读。
 传递性：如果 A happens- before B，且 B happens- before C，那么 A happens- before C。

实际上happens-before在底层对应了JMM规定的重排序规则和CPU自己的重排序规则，可以说它是上层抽象。
并且要注意的是，happens-before只是可见性顺序，不是执行顺序。

## 3.2 重排序

### 3.2.1 数据依赖性

两个操作，只要有一个是写操作，它们就会产生数据依赖性。单线程中会保证重排序不会影响指令结果，而不同处理器和不同线程之间的数据依赖性就没有这个保证。

### 3.2.2 as-if-serial语义

指不管怎么重排序，单线程程序的执行结果不能被改变。

### 3.2.4 重排序对多线程的影响

```Java
class RecordExample {
    int a = 0;
    boolean flag = false;
    
    public void writer() {
        a = 1;
        flag = true;
    }
    
    public void reader() {
        if (flag) {
            int i = a * a;
            ...
        }
    }
}
```

如果有两个线程，A首先执行writer()方法，随后B接着执行reader()方法。 B在执行69行时，不一定能看到A对a的写入。这样的重排序还可能不止这一种。

因此，多线程程序中，重排序可能会导致程序执行结果的改变。

## 3.3 顺序一致性

顺序一致性内存模型是一个理论参考模型，处理器和编程语言的内存模型设计都会以它作为参照。

### 3.3.1 数据竞争与顺序一致性

数据竞争，就是类似竞态条件。如果一个多线程程序没有做正确的同步，就会产生这种情况。

顺序一致性保证是指，如果程序是正确同步的，程序的执行结果与改程序在顺序一致性内存模型中的执行结果相同。

### 3.3.2 顺序一致性内存模型

两大特性：
1) 一个线程中的所有操作必须按照程序的顺序来执行。
2) 所有线程都只能看到一个单一的操作执行顺序。也就是说指令的顺序对所有线程都是可见的，并且看到的一致。

图3-10 顺序一致性内存模型对程序员的视图

JMM不保证这点！

### 3.3.3 同步程序的顺序一致性效果

Java中正确同步的代码会保证顺序一致性，同步区域称为临界区。JMM的实现会尽可能在不改变程序执行结果下，为编译器和处理器的优化提供便利。

### 3.3.4 未同步程序的执行特性

简单说就是保证不会读出莫名其妙的值。

总线在一个时刻只允许一个访问，并发的总线访问会经过“总线仲裁”决定谁上位。

注意64位变量的写操作，也是不会保证一致性的。

## 3.4 volatile的内存语义

### 3.4.1 volatile的特性

理解volatile可以把volatile变量的单个读/写，看成是使用同一个锁对这些单个读/写操作做了同步。

volatile变量特性：
1.可见性。
2.原子性。

### 3.4.2 volatile写-读建立的happens-before关系

volatile变量可以实现线程之间的通信。

### 3.4.3 volatile写-读的内存语义

volatile写的内存语义： 当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存。

总之，volatile的内存语义，实质上是线程A向线程B发送消息。

### 3.4.4 volatile内存语义的实现

通过禁用某些重排序和插入屏障实现。

### 3.4.5 JSR-1333为什么要增强volatile的内存语义

旧内存模型中，volatile的写-读没有锁的释放-获取锁具有的内存语义。

JSR-133让它具有了这种语义，*从而提供一种比锁更轻量级的线程之间的通信机制*。

## 3.5 锁的内存语义

### 3.5.1 锁的释放 - 获取建立的happens-before关系

锁除了让临界区互斥执行以外，还可以让释放锁的线程向获取同一个锁的线程发送消息。

根据监视器锁规则建立的happens-before关系会同步两个临界区。

### 3.5.2 锁的释放和获取的内存语义

释放锁时，本地内存会刷新到主内存中。获取锁时，本地内存被置为无效。

可以看出，释放和获取锁，与volatile的写和读有相同的内存语义。

### 3.5.3 锁内存语义的实现

```java
class ReentrantLockExample {
    int a = 0;
    ReentrantLock lock = new ReentrantLock();
    
    public void writer() {
        lock.lock();
        try {
            a++;
        } finally {
            lock.unlock();
        }
    }
    
    public void reader() {
        lock.lock();
        try {
            int i = a;
            ...
        } finally {
            lock.unlock();
        }
    }
}
```
ReentrantLock的实现依赖于Java同步器框架AbstractQueuedSynchronizer。AQS使用一个volatile变量来维护同步状态。

根据volatile的内存语义，编译器不能对volatile读以及读后面的任意内存操作重排序；也不会volatile写以及写之前任意的操作重排序。

如果对volatile变量使用CAS，则不能对CAS以及它前后面任意内存操作重排序。

x86架构下对多核处理器上运行的程序，会为cmpxchg指令加上lock前缀。这个lock前缀会使得对内存的读 - 改 - 写操作原子执行。

公平锁获取的时候会先读volatile变量；非公平锁获取时，会使用CAS更新volatile变量，也就是对volatile同时具有读和写的内存语义。

### 3.5.4 concurrent包的实现

其实就是利用volatile和CAS的组合操作来实现的。

通用实现模式：
声明共享变量为volatile，
使用CAS的原子条件更新来实现线程间的同步。
同时配合volatile变量的读/写和CAS所具有的volatile的读和写语义来实现。

concurrent包中的高层，依赖于AQS、非阻塞数据结构和原子变量类，而它们又依赖于volition变量的读/写和CAS。

## 3.6 final域的内存语义

 对它的重排序规则：
 1) 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。
 2）初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。
 
 这些语义保证了一点：只要对象是正确构造的，那么不需要使用同步就可以保证任意线程都能看到这个final域在构造函数中被初始化之后的值。
 
 ## 3.7 happens-before
 
 
 happens-before可以看做是Java暴露给程序员的内存可见性保障，它屏蔽了Java对底层实现在保障足够内存可见性和尽量让编译器、处理器做更多的优化之间的平衡点。
 
 ### 3.7.3 happens-before规则
 
 除了之前介绍过的4条之外，还有：
 
 start规则：从线程B执行ThreadA.start()，那这个ThreadA.start() happens-before A中的任意操作。
 join规则：线程A执行ThreadB.join()并成功返回，则B中任意操作happens-before于线程A从ThreadB.join()操作成功返回。
 
 ## 3.8 双重检查锁定与延迟初始化
 
 Java多线程程序中，有时候需要采用延迟初始化来降低初始化类和创建对象的开销。
 
 双重检查锁定是一个错误的用法。
 
 ### 3.8.1 双重检查锁定的由来
 
 DCL代码：
 
 ```java
 public class DoubleCheckedLocking {
    private static Instance instance;
    
    public static Instance getInstance() {
        if (instance == null) {
            synchronized (DoubleCheckedLocking.class) {
                if (instance == null)
                    instance = new Instance(); // 问题的根源
            }
        }
    }
 }
 ```
问题根源在于，第一次检查的时候，读取到instance不为null时，insatnce引用的对象可能还没有完成初始化。

### 3.8.2 问题的根源

instance = new Singleton() 可以分解为一下3行伪代码：
memory = allocate();        // 1:分配对象的内存空间
ctorInstance(memory);       // 2:初始化对象
instance = memory();        // 3:设置instance指向刚分配的内存地址
其中2、3可能会被重排序。

### 3.8.3 基于volatile的解决方案

在instance域前加上volatile声明。
实质是禁止重排序。

### 3.8.4 基于类初始化的解决方案

JVM在类初始化期间会去获取一个锁，这个锁可以同步多个线程对同一个类的初始化。

```java
public class InstanceFactory {
    private static class InstanceHolder {
        public static Instance instance = new Instance();
    }
    
    public static Instance getInstance() {
        return InstanceHolder.instance;         // 这里将导致InstanceHolder类被初始化
    }
}
```

实质是让其他线程看不到重排序。

volatile的双重检查锁定方案的优点在于，除了可以对静态字段实现延迟初始化之外，还可以对实力字段实现。

## 3.9 Java内存模型综述

### 3.9.1 处理器的内存模型

JMM为我们屏蔽了不同处理器上的实现差异。

### 3.9.2 各种内存模型之间的关系

### 3.9.3 JMM的内存可见性保证

单线程程序，
正确同步的多线程程序： 顺序一致性

未同步/未正确同步的多线程程序：最小安全性保障

### 3.9.4 JSR-133 对就内存模型的修补

1. 增强volatile的内存语义

2. 增强final的内存语义