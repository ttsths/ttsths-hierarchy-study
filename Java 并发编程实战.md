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

### CoubtDownLatch

实现线程等待，解决一个线程等待多个线程的场景（**共享锁**：所有共享锁的线程共享同一个资源，一旦任意一个线程拿到共享资源，那么所有线程都拥有同一份资源，通常情况下共享锁只是一个标识，所有线程等待这个标识是否满足，一旦满足所有线程都被激活）

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
    // 打开闭锁
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

/**
* 内部调用了AQS的acquireSharedInterruptibly(1)
*/
public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
public int tryAcquireShared(int acquires) {
    return getState() == 0? 1 : -1;
}
/**
*1.将当前线程节点以共享模式加入AQS的CLH队列中（相关概念参考这里和这里）。进行2。
 2.检查当前节点的前任节点，如果是头结点并且当前闭锁计数为0就将当前节点设置为头结点，唤醒继任节点，返回（结束线程阻塞）。否则进行3。
 3.检查线程是否该阻塞，如果应该就阻塞(park)，直到被唤醒（unpark）。重复2。
 4.如果2、3有异常就抛出异常（结束线程阻塞）
*/
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                break;
        }
    } catch (RuntimeException ex) {
        cancelAcquire(node);
        throw ex;
    }
    // Arrive here only if interrupted
    cancelAcquire(node);
    throw new InterruptedException();
}
```

### CyclicBarrier实现线程同步,CycliBarrier是一组线程之间相互等待，计数器可循环利用

- await()方法将挂起线程，直到同组的其它线程执行完毕才能继续
- await()方法返回线程执行完毕的索引，注意，索引时从任务数-1开始的，也就是第一个执行完成的任务索引为parties-1,最后一个为0，这个parties为总任务数，清单中是cnt+1
- CyclicBarrier 是可循环的，显然名称说明了这点。在清单1中，每一组任务执行完毕就能够执行下一组任务。
- 如果屏障操作不依赖于挂起的线程，那么任何线程都可以执行屏障操作。在清单1中可以看到并没有指定那个线程执行50%、30%、0%的操作，而是一组线程（cnt+1）个中任何一个线程只要到达了屏障点都可以执行相应的操作
- CyclicBarrier 的构造函数允许携带一个任务，这个任务将在0%屏障点执行，它将在await()==0后执行。
- CyclicBarrier 如果在await时因为中断、失败、超时等原因提前离开了屏障点，那么任务组中的其他任务将立即被中断，以InterruptedException异常离开线程。
- 所有await()之前的操作都将在屏障点之前运行，也就是CyclicBarrier 的内存一致性效果

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

包含三个参数：共享变量的内存地址A、用于比较的值B、和共享变量的新值C.

JNI(Java Native Interface)是在 Java 和 Native 层（包括但不限于C/C++）相互调用的接口规范。

```java
unSafe.compareAndSwapInt(this,valueOffset,expect,update)
```

CAS的问题：ABA

解决方法：添加Version并保证Version是自增的



锁机制存在的问题：

- 加锁、释放锁会导致上下文切换和调度延时引起性能问题
- 一个线程持有锁、其他线程会挂起



### 原子类概览

原子类：基本数据类型、对象引用类型、数组、累加器、对象属性更新器

![原子类.png](http://ww1.sinaimg.cn/large/006zHS9Ngy1g8cultzytqj311u0ia0xu.jpg)

## AbstractQueuedSynchronizer

AQS是构建锁或者其他同步组件的基础框架，Doug Lea期望它能够成为实现大部分同步需求的基础

**AQS里面有三个核心字段：**

`private volatile int state;`

`private transient volatile Node head;`

`private transient volatile Node tail;`

- Voliate 类型的int 成员变量state 表示同步状态，state > 0 表示已经获取了锁、当state = 0时释放了锁。

  getState()、setState(int newState)、compareAndSetState(int expect,int update)

  tryAcquire(int arg):独占式获取同步状态

  tryRelease(int arg):独占式释放同步状态

  tryAcquireShared(int arg):共享式获取同步状态，返回值>=0表示获取成功

  tryReleaseShared(int arg):共享式释放同步状态

  isHeldExclusively():当前同步器是否在独占式模式下被线程占用

  acquire(int arg):方法为AQS提供的模板方法,独占式获取同步状态，但是该方法对中断不敏感：线程获取同步状态失败加入到CLH同步队列中，后续对线程进行中断操作时，线程不会从同步队列中移除

  addWaiter：如果tryAcquire返回FALSE（获取同步状态失败），则调用该方法将当前线程加入到CLH同步队列尾部

  acquireQueued：当前线程会根据公平性原则来进行阻塞等待（自旋）,直到获取锁为止；并且返回当前线程在等待过程中有没有中断过

- CLH，FIFO双向同步队列，来完成资源获取线程的排队工作

  AQS依赖它完成同步状态的管理，一个Node一个线程，保存线程的引用、状态(waitStatus)、prev、next 

  - Node节点属性
    - ***volatile int waitStatus***;节点的等待状态
      - CANCELLED = 1；*节点操作因为超时或者对应的线程被interrupt。节点不应该留在此状态，一旦达到此状态将从CHL队列中踢出。*
      - SIGNAL = -1；*节点的继任节点是（或者将要成为）BLOCKED状态（例如通过LockSupport.park()操作），因此一个节点一旦被释放（解锁）或者取消就需要唤醒（LockSupport.unpack()）它的继任节点。*
      - CONDITION = -2；*表明节点对应的线程因为不满足一个条件（Condition）而被阻塞。*
      - 0；*正常状态，新生的非CONDITION节点都是此状态。*
      - *非负值标识节点不需要被通知（唤醒）。*
    - ***volatile Node pre***;*此节点的前一个节点。节点的waitStatus依赖于前一个节点的状态。*
    - ***volatile Node next;****此节点的后一个节点。后一个节点是否被唤醒（uppark()）依赖于当前节点是否被释放。*
    - ***volatile Thread thread;****节点绑定的线程。*
    - **Node nextWaiter;***下一个等待条件（Condition）的节点，由于Condition是独占模式，因此这里有一个简单的队列来描述Condition上的线程节点。*

  - enq(Node node) CAS操作，每次比较尾结点是否一致

    ```java
    do {
    
            pred = tail;
    
    }while ( !compareAndSet(pred,tail,node) );
    ```

    

  - deq(Node node) CAS

    ```java
    while (pred.status != RELEASED) ;
    
    head  = node;
    ```

    

- LockSupport

  >LockSupport是用来创建锁和其他同步类的基本线程阻塞原语

```java
/**
* 在当前线程中使用，导致线程阻塞，参数Object是挂起对象
由于park()立即返回，所以通常情况下需要在循环中去检测竞争资源来决定是否进行下一次阻塞。park()返回的原因有三：

其他某个线程调用将当前线程作为目标调用 unpark；
其他某个线程中断当前线程；
该调用不合逻辑地（即毫无理由地）返回。
其实第三条就决定了需要循环检测了，类似于通常写的while(checkCondition()){Thread.sleep(time);}类似的功能。
*/
LockSupport.park();
LockSupport.park(Object);
LockSupport.parkNanos(Object,long);
LockSupport.parkNanos(long);
LockSupport.parkUNtil(Object,long);
LockSupport.parkUntil(long);
LockSupport.unpark();
```

## Lock

```java
public void java.util.concurrent.locks.ReentrantLock.lock()
```

ReentrantLock是可重入锁，内部委托Reentrantlock.Sync.lock()实现。Sync是一个抽象类，有FairSync和NonfairSync两个实现。

***公平锁与非公平只是在入AQS的CLH队列之前有所差别，一旦入队是相同的，都是要按照队列中的先后顺序请求锁***

>1,如果tryAcquire(arg)成功则结束，失败进入2
>
>2.创建一个独占节点Node，并且此节点加入CLH队列尾部，进行3
>
>3.自旋尝试获取锁、失败根据前一个节点来决定是否挂起(park())，直到成功获取到锁，进行4
>
>4.如果当前线程已经被中断过，那么中断当前线程(清除中断位)

### tryAcquire()

AQS存在一个state来描述当先有多少线程持有锁。支持独占和共享。

```java
/**
* 公平锁的实现方式
1. 如果当前锁有其它线程持有，c!=0，进行操作2。否则，如果当前线程在AQS队列头部，则尝试将AQS状态state设为acquires（等于1），成功后将AQS独占线程设为当前线程返回true，否则进行2。这里可以看到compareAndSetState就是使用了CAS操作。
2。 判断当前线程与AQS的独占线程是否相同，如果相同，那么就将当前状态位加1（这里+1后结果为负数后面会讲，这里暂且不理它），修改状态位，返回true，否则进行3。这里之所以不是将当前状态位设置为1，而是修改为旧值+1呢？这是因为ReentrantLock是可重入锁，同一个线程每持有一次就+1。
3. 返回false。
*/
protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            //isFirst是一个很复杂的逻辑，包括踢出无用的节点等过程
            if (isFirst(current) &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

公平锁比非公平锁多了一个判断当前节点是否在队列头，这就保证了请求锁的顺序来决定获取锁的顺序

### addWaitor(mode)

tryAcquire失败意味着加入等待队列，独占锁在Node中意味着条件Condition队列为空。mode：表示的是节点的模式，EXCLUSIVE|SHARED

```java
private Node addWaiter(Node mode) {
     Node node = new Node(Thread.currentThread(), mode);
     // Try the fast path of enq; backup to full enq on failure
     Node pred = tail;
     if (pred != null) {
         node.prev = pred;
         if (compareAndSetTail(pred, node)) {
             pred.next = node;
             return node;
         }
     }
     // 去队列操作，如果为空就创建头节点，然后同事比较节点尾部是否是改变开决定CAS操作是否成功，当且仅当成功后才将不节点的下一个节点指向新节点
     enq(node);
     return node;
}
```

### acquireQueued(mode, arg)

自旋请求锁，如果可能的话挂起线程，直到得到锁，返回当前线程是否中断过(如果park()过并且中断过的话有一个interrupted中断位)

```java
final boolean acquireQueued(final Node node, int arg) {
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 如果当前节点是头结点，如果第一个节点是DUMP节点也就是傀儡节点，那么第二个节点才是头结点
            if (p == head && tryAcquire(arg)) {
                // 将头节点的前任节点清空，并将头结点的线程清空为了更好GC，防止内存泄漏
                setHead(node);
                p.next = null; // help GC
                return interrupted;
            }
            //当前节点是否应该park()
            if (shouldParkAfterFailedAcquire(p, node) &&
                // 应该park()就挂起当前线程并且返回当前线程的中断位
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } catch (RuntimeException ex) {
        cancelAcquire(node);
        throw ex;
    }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int s = pred.waitStatus;
    // 当前节点还没有获取锁，当前节点应该被park()
    if (s < 0) return true;
    // 
    if (s > 0) {
      	// 前一个节点被CANCELLED了，那么前一个节点去掉
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } 
  	// 修改前一个节点的状态位为SINGAL，表示后面节点等待你处理
    else compareAndSetWaitStatus(pred, 0, Node.SIGNAL);
    return false;
}
```

### Release/TryRelease 

unlock()操作实际上调用了AQS的release操作，释放持有的锁

```java
/**
* 尝试去释放锁，成功，则看队列头结点能否被唤醒，如果可以的唤醒下一个节点(非CANCELED)
**/
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

/**
* 判断持有锁的线程是否是当前节点，不是则抛IllegalMonitorStateExeception()
  减少AQS状态位，如果是0，清空AQS持有锁的独占线程(设NULL)
*/
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}

private void unparkSuccessor(Node node) {
        //此时node是需要释放锁的头结点

        //清空头结点的waitStatus，也就是不再需要锁了
        compareAndSetWaitStatus(node, Node.SIGNAL, 0);

        //从头结点的下一个节点开始寻找继任节点，当且仅当继任节点的waitStatus<=0才是有效继任节点，否则将这些waitStatus>0（也就是CANCELLED的节点）从AQS队列中剔除  
       //这里并没有从head->tail开始寻找，而是从tail->head寻找最后一个有效节点。
       //解释在这里 http://www.blogjava.net/xylz/archive/2010/07/08/325540.html#377512

        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }

        //如果找到一个有效的继任节点，就唤醒此节点线程
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

## Condition

条件需要与锁绑定，Lock.newCondition(),一个Lock可以有任意的Condition对象

```java
// 对应Object.wait()
// 
void await() throws InterruptedException;
void awaitUninterruptibly();
long awaitNanos(long nanosTimeout) throws InterruptedException;
boolean await(long time, TimeUnit unit) throws InterruptedException;
boolean awaitUntil(Date deadline) throws InterruptedException;
// 对用Object.nofity()
void signal();
void signalAll();

/**
* 1.将当前线程加入Condition锁队列，不同于AQS的CLH队列，这里的队列是Condition的FIFO队列
  2.释放锁
  3.自旋(while)挂起，直到被唤醒或者超时或者CANCELLED等
  4.获取锁(acquireQueued)，并将自己从Condition的FIFO队列中释放，已经获取锁
*/
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        // 通过LockSupport.park()
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}

private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
/**
* 1.将Condition.await*()中FIFO队列的第一个Node(非CANCELLED状态)唤醒(或者全部唤醒)。只有一个线程能获取到锁，其他没有拿到锁的线程自旋等待
**/
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter  = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```



## ThreadPool

```java

ThreadPoolExecutor(
  int corePoolSize,
  int maximumPoolSize,
  long keepAliveTime,
  TimeUnit unit,
  BlockingQueue<Runnable> workQueue,
  ThreadFactory threadFactory,
  RejectedExecutionHandler handler) 
```

- CorePoolSize
- MaximumPookSize
- KeepAliveTime & unit
- workQueue：**强烈建议使用有界队列**
- ThreadFactory
- Handler:拒绝策略
  - CallerRunsPolicy：提交任务的线程自己去执行该任务
  - AbortPolicy:默认的拒绝策略，throw RejectedExecutionException（**谨慎使用**）
  - DiscardPolicy: 直接丢=丢弃任务，不会抛出异常
  - DiscardOldestPolicy:丢弃最老的任务，将新任务加入队列

## Future

```java

// 取消任务
boolean cancel(
  boolean mayInterruptIfRunning);
// 判断任务是否已取消  
boolean isCancelled();
// 判断任务是否已结束
boolean isDone();
// 获得任务执行结果
get();
// 获得任务执行结果，支持超时
get(long timeout, TimeUnit unit);
```

- 提交 Runnable 任务 submit(Runnable task) ：这个方法的参数是一个 Runnable 接口，Runnable 接口的 run() 方法是没有返回值的，所以 submit(Runnable task) 这个方法返回的 Future 仅可以用来断言任务已经结束了，类似于 Thread.join()。

- 提交 Callable 任务 submit(Callable task)：这个方法的参数是一个 Callable 接口，它只有一个 call() 方法，并且这个方法是有返回值的，所以这个方法返回的 Future 对象可以通过调用其 get() 方法来获取任务的执行结果。

- 提交 Runnable 任务及结果引用 submit(Runnable task, T result)：这个方法很有意思，假设这个方法返回的 Future 对象是 f，f.get() 的返回值就是传给 submit() 方法的参数 result。

  这个方法该怎么用呢？下面这段示例代码展示了它的经典用法。需要你注意的是 Runnable 接口的实现类 Task 声明了一个有参构造函数 Task(Result r) ，创建 Task 对象的时候传入了 result 对象，这样就能在类 Task 的 run() 方法中对 result 进行各种操作了。result 相当于主线程和子线程之间的桥梁，通过它主子线程可以共享数据。