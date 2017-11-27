## 第4章 Java并发编程基础

### 4.1 线程简介

进程：操作系统在启动一个Java程序时，会为其创建一个Java进程。现代操作系统调度的最小单元是线程，也叫轻量级进程。一个进程里可以有多个线程，这些线程都拥有各自的计数器、堆栈和局部变量等属性，并且能够访问共享的内存变量。

一个Java程序本身就是个多线程程序。

### 4.1.2 为什么要使用多线程

1. 更多的处理器核心
2. 更快的响应时间
3. 更好的编程模型

### 4.1.3 线程的优先级

线程的优先级不能作为程序正确性的依赖。因为操作系统可以完全不用理会Java线程对于优先级的设定。

### 4.1.4 线程的状态

线程6状态。

synchronized代码块中是阻塞状态，而并发包中的Lock结构的线程状态是等待状态，因为使用了LockSupport类。

### 4.1.5 Daemon线程

Daemon线程是在非Daemon线程都结束时，它就会被销毁。而且它的finally块也不会执行。

## 4.2 启动和终止线程

随start()启动，随run()的完成结束。

### 4.2.1 构造线程

可以设置线程的相关属性。
currentThread()可以获取到当前线程。
t.group
t.daemon
t.priority
t.name
t.target
// 将父线程的InheritableThreadLocal复制过来
t.inheritableThreadLocals = ThreadLocal.createInheritedMap(parent.inheritableThreadLocals)
nextThreadID(); // 分配一个线程ID

### 4.2.2 启动线程

start()方法的含义： 当前线程（parent)同步告知Java虚拟机，只要线程规划器空闲，应立即启动调用start()方法的线程。

给线程一个名字是一个好习惯。

### 4.2.3 理解中断

其他线程调用目标线程的interrupt()对其进行中断。

中断可以理解为线程的一个标志位属性。可以通过isInterrupted()来判断是否被中断，还可以调用Thread.interrupted()对当前线程的中断标志位进行复位。
但如果线程已经处于终结状态，则调用isInterrupted()总是返回false。

跑出InterruptedException之前，Java虚拟机会先将该线程的中断标志位清除。

如果一个线程一直在执行任务，给它设置的中断标志在结束时不会被清除。


### 4.2.4 过期的suspend()、resume()和stop()

这些方法由于会导致资源占用或资源没有完全释放等问题，被标记为过期的、不安全的。

### 4.2.5 安全地终止线程

可以通过设置中断 或者 设置一个开关变量的方法来终止线程，后者更加的可控。

## 4.3 线程间通信

### 4.3.1 volatile和synchronized关键字

使用volatile关键字来告诉变量需要在共享内存中去访问。

synchronized本质上是对一个对象的监视器进行获取，而这个获取过程是排他的。
任意一个对象都拥有自己的监视器，获取到才能进入区域，否则被阻塞在同步块入口，线程进入同步队列，进入BLOCKED状态。

### 4.3.2 等待/通知机制

[1]轮询模式
```java
while (value != desire) {
    Thread.sleep(1000);
}
doSomething();
```

Sleep():睡眠稍长时间基本不消耗处理器资源。

1) 难以保证及时性。
2) 难以降低开销。

[2] 等待/通知机制

(1)使用wait()、notify()和notifyAll()时需要先对调用对象加锁。( sychronized(lockObj) )
(2)调用wait()方法后，意味着放弃了锁，线程状态由RUNNING变为WAITING，并将当前线程放置到*锁对象*的等待队列。
(3)notify()或notifyAll()方法调用后，等待线程依旧不会从wait()返回，需要等调用notify()或notifyAll()的线程释放锁后，等待线程才有机会从wait()返回。
(4)notify()和notifyAll()是将等待队列中的线程移动到同步队列中，被移动的线程状态由WAITING变为BLOCKED。
(5)从wait()方法返回的前提是获得了调用对象的锁。

### 4.3.3 等待/通知的经典范式

分等待方和通知方。

等待方：
(1)获取对象的锁。
(2)如果条件不满足，则调用wait()方法，被通知后仍然要检查条件。
(3)条件满足则执行对应的逻辑。

synchronized(对象) {
    while(条件不满足) {
        对象.wait();
    }
    对应的处理逻辑
}

通知方：
(1)获得对象的锁
(2)改变条件
(3)通知所有等待在对象上的线程。

synchronized(对象) {
    改变条件
    对象.notifyAll();
}

### 4.3.4 管道输入/输出流

用于线程之间的数据传输，媒介为内存。

PipedOutputStream、PipedInputStream、PipedReader、PipedWriter
|---          面向字节           ---|---     面向字符      ---|

Piped类型的流必须通过connect()方法绑定。

### 4.3.5 Thread.join()的使用

线程A执行了thread.join()语句的含义是：线程A等待thread线程终止之后再返回。

join系列方法还提供了join(long millis)和join(long millis, int nanos)两个具备超时特性的方法。

join方法的代码（伪）：
public final synchronized void join() throws InterruptedException {
    while (isAlive()) {
        wait(0);
    }
    // 线程非alive，返回
}

### 4.3.6 ThreadLocal的使用

相当于线程的Tag，以ThreadLocal对象为键、任意对象为值。

可以通过set(T)设置值，在当前线程下通过get()方法返回设置的值。

下面是一个可以从来测试方法执行时间的类，使用它的好处是，begin()和end()方法可以在不同的方法或者类中。（一般是在一个方法前后记录时间点计算其差）

public class Profiler {
    private static final ThreadLocal<Long> TIME_THREADLOCAL = new ThreadLocal<Long>() {
        protected Long initialValue() {
            return System.currentTimeMillis();
        }
    };
    
    public void begin() {
        TIME_THREADLOCAL.set(System.currentTimeMillis());
    }
    
    public void end() {
        return System.currentTimeMillis() - TIME_THREADLOCAL.get();
    }
    
    public static void main(String[] args) throws Exception {
        Profiler.begin();
        TimeUnit.SECONDS.sleep(1);
        System.out.println("Cost: " + Profiler.end() + " mills");
    }
}

## 4.4 线程应用实例

### 4.4.1 等待超时模式

场景： 调用一个方法等待一段时间，如果在指定时间段内得到结果，就立即返回；否则，超时则返回默认值。

// 对当前对象加锁
public synchronized Object get(long mills) throws InterruptedException {
    long future = System.currentTimeMillis() + mills;
    long remaining = mills;
    // 当超时大于0并且result返回值不满足要求
    while ((result == null) && remaining > 0) {
        wait(remaining);
        remaining = future - System.currentTimeMillis();
    }
    return result;
}

### 4.4.2 一个简单地数据库连接池示例

```java
public class ConnectionPool {
    private LinkedList<Connection> pool = new LinkedList<Connection>();
    
    /**
    * 根据给定数量，初始化连接池
    */
    public ConnectionPool(int initialSize) {
        if (initialSize > 0) {
            for (int i = 0; i < initialSize; i++) {
                pool.addLast(ConnectoinDriver.createConnection());
            }
        }
    }
    
    /**
    * 归还连接
    */
    public void releaseConnection(Connection connection) {
        if (connection != null) {
            synchronized (pool) {
                // 连接释放后需要进行通知，这样其他消费者能够感知到连接池中已经归还了一个连接
                pool.addLast(connection);
                pool.notifyAll();
            }
        }
    }
    
    /**
    * 获取连接
    */
    public Connection fetchConnection(long mills) throws InterruptedException {
        synchronized (pool) {
            // 完全超时
            if (mills <= 0) {
                while (pool.isEmpty()) {
                    pool.wait();
                }
                return pool.removeFirst();
            } else {
                long future = System.currentTimeMillis() + mills;
                long remaining = mills;
                while (pool.isEmpty() && remaining > 0) {
                    pool.wait(remaining);
                    remaining = future - System.currentTimeMillis();
                }
                Connectoin result = null;
                if (!pool.isEmpty()) {
                    result = pool.removeFirst();
                }
                return result;
            }
        }
    }
}
```

使用动态代理模拟一个ConnectionDriver:

```java
public class ConnectionDriver {
    static class ConnectionHandler implements InvocationHandler {
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (method.getName().equals("commit")) {
                TimeUnit.MILLISENCONDS.sleep(100);
            }
            return null;
        }
    }
    
    // 创建一个Connection的代理，在调用commit时休眠100毫秒
    public static final Connection createConnection() {
        return (Connection) Proxy.newProxyInstance(ConnectionDriver.class.getClassLoader(),
            new Class<?>[] {Connection.class}, new ConnectionHandler());
    }
}
```

下面是模拟客户端：

```java
public class ConnectionPoolTest {
    static Connection pool = new ConnectionPool(10);
    // 保证所有ConnectionRunner能够同时开始
    static CountDownLatch start = new CountDownLatch(1);
    // main线程将会等待所有ConnectionRunner结束后才能继续执行
    static CountDownLatch end;
    
    public static void main(String[] args) throws Exception {
        // 线程数量， 可以修改线程数量进行观察
        int threadCount = 10;
        end = new CountDownLatch(threadCount);
        int count = 20;
        AtomicInteger got = new AtomicInteger();
        AtomicInteger notGot = new AtomicInteger();
        for (int i = 0; i < threadCount; i++) {
            Thread thread = new Thread(new ConnectionRunner(count, got, notGot), "ConnectionRunnerThread");
            thread.start();
        }
        start.countDown();
        end.await();
        System.out.println("total invoke: " + (threadCount * count));
        System.out.println("got connection: " + got);
        System.out.println("not got connection: " + notGot);
    }
    
    static class ConnectionRunner implements Runnable {
        int count;
        AtomicInteger got;
        AtomicInteger notGot;
        
        public ConnectoinRunner(int count, AtomicInteger got, AtomicInteger notGot) {
            this.count = countl
            this.got = got;
            this.notGot = notGot;
        }
        
        public void run() {
            try {
                start.await();
            } catch (Exception ex) {
            
            }
            while(count > 0) {
                try {
                    // 从线程池中获取连接，如果1000ms内无法获取到，将会返回null
                    // 分别统计连接获取的数量got和未获取到的数量notGot
                    Connection connection = pool.fetchConnection(1000);
                    if (connection != null) {
                        try {
                            connection.createStatement();
                            connection.commit();
                        } finally {
                            pool.releaseConnection(connection);
                            got.incrementAndGet();
                        }
                    } else {
                        notGot.incrementAndGet();
                    }
                } catch (Exception ex) {
                
                } finally {
                    count--;
                }
            }
            end.countDown();
        }
    }
}
```

注意CountDownLatch的使用。可以发现随着线程的增加，由于存在超时，会导致获取不到的情况发生，但是这样一来同时也避免了线程一直处于工作状态而造成资源浪费。

### 4.4.3 线程池技术及其示例

好处：
1.消除了频繁创建和销毁线程的系统资源开销；
2.对过量任务的提交进行缓冲。

合理分配线程池中的线程数量，找到既能充分发挥处理器性能，又能避免造成过多系统开销的那个临界点。