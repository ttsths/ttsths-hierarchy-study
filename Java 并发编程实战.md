# Java 并发编程实战

[TOC]

# 并发基础

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



## 管程--Monitor

**管程**：指的是管理共享变量以及对共享变量的操作过程，让他们支持并发

**MESA模型**

![屏幕快照 2019-10-20 下午8.36.54.png](http://ww1.sinaimg.cn/large/006zHS9Ngy1g84y4msm73j30ji0n00y9.jpg)

- 条件变量，每个条件变量都对应有一个等待阻塞队列，用于解决线程同步问题
- 入口等待队列：用于进入管程，



> 阻塞队列(里面存放**共享数据**)，入队、出队
>
> 入队：队列满则等待
>
> 出队：队列为空则需要等待出列不为空
>
> 入队成功，需要通知条件变量。队列不空对应的等待队列
>
> 出队成功，需要通知条件变量。队列不满对应的等待队列

```java
public class BlockedQueue<T> {
		final Lock lock = new ReentrantLock();
		// 条件变量：队列不满
		final Condition notFull = lock.newCondition();
		// 条件变量：队列不空
		final COndition notEmpty = lock.newCondition();
		
		// 入队
		void enq(T x){
				lock.lock();
				try{
					while(队列不满){
						// 等待队列不满
						notFull.await();
					}
					// 省略入队操作
					// 入队后，通知可出队
					notEmpty.signal();
				}finally{
					lock.unlock();
				}
		}
		
		// 出队
		void deq(){
				lock.lock();
				try{
					while(队列已空){
						// 等待队列不空
						notEmpty.await();
						// 省略出队操作
						// 出队后通知可入队
						notFull.singal();
					}
				}finally{
					lock.unlock();
				}
		}
		
}
```



**wait()的正确姿势，需要在while里面调用wait()**

```java
while(条件不满足){
	wait()
}
```

**MESA**管程模型中，T2通知完T1后，T2还是会接着执行，T1并不会立即执行，仅仅是从条件变量的等待队列进到入口等待队列里面。

**notify()何时可以使用**

尽量使用notifyAll()方法，



### 线程的生命周期

- (New)初始状态，在Java内被创建，操作系统未真实创建线程
- (Runnable)可运行状态，分配CPU执行，线程已经创建，
- (Runnable)运行状态，有空闲CPU，被分配到CPU的线程转换成运行态
- (Blocked、Waiting、Time_waiting)休眠状态，等待某个事件，进入休眠状态会释放CPU使用权
- (Terminated)终止状态

> Synchronized:Runnable—>Blocked

**JVM层面并不关心操作系统调度相关的状态**，Java调用阻塞API时，指的是线程在操作系统层面的线程状态

> Runnable—>Waiting

Object.wait()

Thread.join():A.join()主线程会等A线程执行完，此时主线程会进入Waiting状态，当A线程执行完时，切换为Runnable

LockSupport.park():所有JUC下的锁，都是基于LockSupport对象实现。

> Runnable—>Timed_Waiting

Thread.sleep(long millis);

Object.wait(long timeout);

LockSupport.parkNanos(Object blocker,long deadline);

LockSupport.parkUnit(long deadline);

> New—>Runnable

继承Thread对象，重写run()

实现Runnable接口，重写run()

> Runnable—>Terminated

Run()方法结束或者中途遇到异常

Interrupt()

### CPU密集型与IO密集型

CPU:最佳线程数=CPU核数+1

IO:最佳线程数=CPU 核数 * [ 1 +（I/O 耗时 / CPU 耗时）]

每个线程有自己的调用栈，一个方法一个栈针，栈针包括：局部变量表，操作数栈，返回地址.

线程封闭：方法里的局部变量，不会有并发问题

## 如何解决并发问题

- 封装共享变量
  - 将共享变量封装在内部，对所有公共方法指定并发访问策略
  - 对于不会发生变化的共享变量，建议使用final修饰
- 识别共享变量间的约束条件
- 制定并发访问策略
  - 避免共享：ThreadLocal
  - 不变模式：Java中较少
  - 同步工具：管程、JUC、并发容器等等

编写健壮的并发程序的原则：

- 优先使用成熟的工具类
- 迫不得已时才使用低级别的同步原语：syschronized、Lock、Semaphore等
- 避免过早优化

# 并发工具

**Java SDK并发包通过Lock 和Condition两个接口实现管程，其中Lock解决互斥，Condition解决同步问题**

Lock的特点：可中断、支持超时、非阻塞获取

```java
// 支持中断的API
void lockInterruptibly() throws InterruptedException;
// 支持超时的API
boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
// 支持非阻塞获取锁的API
boolean tryLock();
```

### 如何保证可见性

Java程序里的可见性主要依靠**Happens-Before**规则

ReentrantLock，主要依靠内部volatile成员变量state。

###可重入锁

### 公平锁和非公平锁

## Semaphore

Dijkstra(迪杰斯特拉)于1965年提出信号量

### 信号量模型

一个计数器，一个等待队列，三个方法

**Semaphore 可以允许多个线程访问一个临界区，例如线程池、连接池**

![Semaphore.png](http://ww1.sinaimg.cn/large/006zHS9Ngy1g8cqcmshj2j30ry0kawht.jpg)

- 一个计数器
- 一个等待队列
- 三个方法—原子性方法
  - init():初始化计数器
  - Down()：计数器减1，计数器的值小于0，则当前线程被阻塞
  - Up()：计数器加1，如果计数器的值小于或者等于0，则唤醒等待队列中的一个线程，并且将其从等待队列中移除

### 限流器

Semaphore与管程的区别：

## 读多写少-ReadWriteLock

ReentrantReadWriteLock:允许多读，单写，写操作时禁止读共享变量

支持公平与非公平模式（AbstractQueuedSynchronizer）

支持Lock接口

写锁支持条件变量，**读锁不支持条件变量**

## StampedLock

java8里面提供了性能更好，但是功能仅仅是ReadWriteLock的子集.

- 提供三种锁模式：写锁、悲观读锁、乐观读锁(stamp)

- 不支持重入

- StampedLock的悲观读锁、写锁不支持条件变量
- 当线程阻塞在readLock()或者writeLock()上时，调用次线程的interrupt()方法会导致CPU飙升。SO:**调用StampedLock一定不要调用中断操作，可以使用readLockInterruptibly()和写锁的writeLockInterruptibly()**



## CountDownLatch和CyclicBarrier：让线程步调一致

### CoubtDownLatch实现线程等待，解决一个线程等待多个线程的场景

```java

// 创建2个线程的线程池
Executor executor = 
  Executors.newFixedThreadPool(2);
while(存在未对账订单){
  // 计数器初始化为2
  CountDownLatch latch = 
    new CountDownLatch(2);
  // 查询未对账订单
  executor.execute(()-> {
    pos = getPOrders();
    latch.countDown();
  });
  // 查询派送单
  executor.execute(()-> {
    dos = getDOrders();
    latch.countDown();
  });
  
  // 等待两个查询操作结束
  latch.await();
  
  // 执行对账操作
  diff = check(pos, dos);
  // 差异写入差异库
  save(diff);
}
```

### CyclicBarrier实现线程同步,CycliBarrier是一组线程之间相互等待，计数器可循环利用

```java

// 订单队列
Vector<P> pos;
// 派送单队列
Vector<D> dos;
// 执行回调的线程池 
Executor executor = 
  Executors.newFixedThreadPool(1);
final CyclicBarrier barrier =
  new CyclicBarrier(2, ()->{
    executor.execute(()->check());
  });
  
void check(){
  P p = pos.remove(0);
  D d = dos.remove(0);
  // 执行对账操作
  diff = check(p, d);
  // 差异写入差异库
  save(diff);
}
  
void checkAll(){
  // 循环查询订单库
  Thread T1 = new Thread(()->{
    while(存在未对账订单){
      // 查询订单库
      pos.add(getPOrders());
      // 等待
      barrier.await();
    }
  });
  T1.start();  
  // 循环查询运单库
  Thread T2 = new Thread(()->{
    while(存在未对账订单){
      // 查询运单库
      dos.add(getDOrders());
      // 等待
      barrier.await();
    }
  });
  T2.start();
}
```

## 并发容器

- List

  - CopyOnWriteArrayList
    - 原理：内部维护一个array，当有写入操作时，copy一份数组（类似于快照）
    - 数据结构：数组
    - 应用场景：
      - 读多写少
      - 可以接受短暂的不一致

- Set

  - CopyOnWriteSet
  - ConcurrentSkipListSet

- Map

  - ConcurrentHashMap（Key无序）

    - 数据结构
    - 原理
    - 应用场景

  - ConcurrentSkipListMap（Key有序）

    - 数据结构：跳表
    - 原理
    - 应用场景
      - 插入、删除、查询操作的平均时间复制读O(log n)，在对ConcurrentHashMap的性能不满意是，可以尝试ConcurrentSkipListMap

    | 集合类                | Key          | Value        | 是否线程安全 |
    | --------------------- | ------------ | ------------ | ------------ |
    | HashMap               | 允许为NULL   | 允许为NULL   | 否           |
    | TreeMap               | 不允许为NULL | 允许为NULL   | 否           |
    | Hashtable             | 不允许为NULL | 不允许为NULL | 是           |
    | ConcurrentHashMap     | 不允许为NULL | 不允许为NULL | 是           |
    | ConcurrentSkipListMap | 不允许为NULL | 不允许为NULL | 是           |

  

- Queue

  **阻塞队列Blocking、单端队列Queue标识、双端队列Deque标识**

  - 单端阻塞队列：ArrayBlockingQueue(有界)、LinkedBlockingQueue(有界)、SynchronizedQueue、LinkedTransferQueue(性能>LinkedBlockingQueue)、PriorityBlockingQueue(优先级) 和 DelayQueue(延迟)

    - 数据结构：数组(ArrayBlockingQueue)|链表(LinkedBlockingQueue)
    - 原理：队尾入队，队首出队

  - 双端阻塞队列：LinkedBlockingDeque

    - 数据结构
    - 实现原理：队首队尾皆可入队出队

  - 单端非阻塞队列：其实现是 ConcurrentLinkedQueue。

  - 双端非阻塞队列：其实现是 ConcurrentLinkedDeque

    

  

![屏幕快照 2019-10-27 下午3.56.23.png](http://ww1.sinaimg.cn/large/006zHS9Ngy1g8ctbm7sxtj316w0hmdkm.jpg)



> Java 容器的快速失败机制（Fail-Fast）
>
> 在使用迭代器对集合对象进行遍历的时候，如果 A 线程正在对集合进行遍历，此时 B 线程对集合进行修改（增加、删除、修改），或者 A 线程在遍历过程中对集合进行修改，都会导致 A 线程抛出 ConcurrentModificationException 异常。
>
>
> 安全失败（fail-safe）

## 原子类-CAS

CAS：Compare And Swap.

包含三个参数：共享变量的内存地址A、用于比较的值B、和共享变量的新值C

### 原子类概览

原子类：基本数据类型、对象引用类型、数组、累加器、对象属性更新器

![原子类.png](http://ww1.sinaimg.cn/large/006zHS9Ngy1g8cultzytqj311u0ia0xu.jpg)

