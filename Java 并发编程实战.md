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

    