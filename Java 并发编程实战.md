# Java 并发编程实战

- 可见性：一个线程对共享变量的修改，另外一个线程可以立刻看到
  - **缓存**导致的可见性。（CPU缓存不存在与内存中，volatile修饰的变量会强制刷新到内存中。volatile会有写入屏障问题，也就是happen-before问题）
- 原子性：一个或者对个操作在CPU执行过程中不被中断
  - **线程切换**导致的原子性问题
- 有序性：
  - **编译优化**导致的有序性问题

## Java内存模型

### 使用volatile的困惑

volatile int x = 0：告诉编译器，对于这个变量的读写，不能使用CPU缓存，必须从内存中读取或者写入。

```java
// 以下代码来源于【参考 1】
class VolatileExample {
  int x = 0;
  volatile boolean v = false;
  public void writer() {
    x = 42;
    v = true;
  }
  public void reader() {
    if (v == true) {
      // 这里 x 会是多少呢？
    }
  }
}

```

### Happens-Before规则

**前面一个操作的结果对后续操作是可见的**

Happens-Before约束了编译器的优化行为

- 程序的顺序性规则

  - 逻辑顺序

- volatile变量规则

- 传递性原则

- 管程中锁的规则

  - 对一个锁的解锁Happens-Before于后续对这个锁的加锁

    ```JAVA
    synchronized (this) { // 此处自动加锁
      // x 是共享变量, 初始值 =10
      if (this.x < 12) {
        this.x = 12; 
      }  
    } // 此处自动解锁
    ```

    假设X的初始值是10，线程A执行完代码块后x的值变成12，线程B进入代码块时，能够看到线程A对X的写操作

- 线程start（）规则

  - 关于线程启动，它是指主线程A启动子线程B后，子线程B能够看到主线程在启动子线程B前的操作。

    ```java
    Thread B = new Thread(() -> {
    	// 主线程调用B
    	// 所有对共享变量的修改，在此处皆可见
    });
    var = 77;
    B.start();
    ```

- 线程join()规则

  - 线程等待，线程A等待线程B完成，线程B完成后，A能看到对共享变量的操作

    ```Java
    Thread B = new Thread(() ->{
    	// 共享变量
    	var = 66;
    });
    B.start();
    B.join();
    // 子线程所有对共享两边的修改
    // 在主线程调用B.join()之后皆可见
    
    ```

### final :修饰的变量，在JDK1.5以后内存模型限制final类型变量的重排。只要构造函数没有逸出。

```java
// 以下代码来源于【参考 1】
final int x;
// 错误的构造函数
public FinalFieldExample() { 
  x = 3;
  y = 4;
  // 此处就是讲 this 逸出，
  global.obj = this;
}

```

## 互斥锁：解决原子性问题

**原子性问题的源头是线程切换**

内存锁模型

![屏幕快照 2019-10-16 下午9.23.53.png](http://ww1.sinaimg.cn/large/006zHS9Ngy1g80d9hbx7oj30m20h2q5x.jpg)

### synchronized

- 修饰非静态方法
  - 当前实例对象this
- 修饰静态方法
  - 锁的当前类得Class对象
- 修饰代码块

加锁的本质就是在锁对象的对象头中写入当前线程ID，synchronized锁的对象monitor指针指向一个ObjectMonitor，所有线程加入它的entrylist里面，去CAS抢锁，更改state加1拿锁，执行完代码，释放锁state减1和AQS机制差不多，只是所有线程不阻塞，CAS抢锁，没有队列，属于非公平锁

```java
class X {
  // 修饰非静态方法
  synchronized void foo() {
    // 临界区
  }
  // 修饰静态方法
  synchronized static void bar() {
    // 临界区
  }
  // 修饰代码块
  Object obj = new Object()；
  void baz() {
    synchronized(obj) {
      // 临界区
    }
  }
}  

```

**锁和受保护资源的关系：N:1**

可能存在的问题：

- 多个操作锁保护的资源不同，导致的并发问题
  - Synchronized(this) synchronized(A.class)

- 死锁问题：保护有关联关系的多个资源
  - 死锁：一组相互竞争资源的线程因相互等待，导致永久阻塞的现象
  - 破坏占用且等待条件
  - 破坏不可抢占条件
  - 破坏循环等待条件

## 等待-通知机制

`wait()`、`notify()`、`notifyAll()`方法操作的等待队列是互斥锁的等待队列，所以如果`synchronized`锁定的是`this`，那么对应的一定是`this.wait()`,`this.notify()`,`this.notifyAll()`。

- **`wait()`、`notify()`、`notifyAll()` 都是在`synchronized{}`方法内部被调用的**

- 尽量使用notifyAll()

> :interrobang:wait()方法和sleep()方法都能让当前线程挂起一段时间，那么他们的区别是什么
>
> - wait会释放对象的锁标识而sleep不会释放锁资源
> - wait是Object方法，需要配合同步方法中使用，sleep是Thread方法。
> - wait无需捕获异常，sleep需要
> - 相同点是都会让出CPU执行时间
>
> 对象等待池：存放的是线程
>
> 对象锁标记等待池：notifyAll()方法调用之后，对象等待池中的所有线程会移动到对象的锁标记等待池

### 安全性问题

竞态条件**：指的是程序的执行结果依赖线程执行的顺序。

```java
if (状态变量 满足 执行条件) {
  执行操作
}
```

锁是解决安全性问题的主要手段，即**互斥**。使用锁需要关注对性能的影响

### 活跃性问题

**死锁**、活锁、饥饿时典型的活跃性问题

**活锁**：有时线程虽然没有发生阻塞，但仍然会存在执行不下去的情况。

​	解决方法：通过加随机值

**饥饿**：线程因无法访问所需资源而无法执行下去。CPU繁忙时，优先级低的线程，执行的概率较小，只有锁的线程，如果执行时间过长，就有可能导致饥饿问题

​	解决方法：

- 保证资源充足
- 公平的分配资源-公平锁
- 避免只有锁的线程长时间执行

无锁的算法和数据结构：ThreadLocal，Copy-on-Write,乐观锁等Java并发包下的原子类也是一种无锁的数据结构；Disruptor是一个无锁的内存队列

### 性能

衡量性能的三个指标：

吞吐量：单位时间内处理的请求数量

延迟：从发出请求到收到响应的时间

并发量：同事处理的请求数量，并发量增加延迟也会增加

