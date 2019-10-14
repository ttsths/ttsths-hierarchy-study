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
- volatile变量规则



