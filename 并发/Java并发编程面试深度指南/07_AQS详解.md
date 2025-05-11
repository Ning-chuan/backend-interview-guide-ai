# 第7章：AQS详解

## 章节概述
本章深入剖析AbstractQueuedSynchronizer（AQS）框架，它是Java并发包中锁和同步器的基础。几乎所有Java并发包中的锁和同步工具（如ReentrantLock、Semaphore、CountDownLatch等）都是基于AQS实现的。掌握AQS对理解Java并发机制至关重要。

## 主要内容
1. **AQS基本概念**
   - AQS的设计思想
   - 框架结构
   - 同步状态管理
   - 队列模型

2. **AQS核心实现原理**
   - 独占模式与共享模式
   - 同步队列的实现（CLH队列）
   - 节点状态与等待机制
   - 阻塞与唤醒机制

3. **AQS源码分析**
   - acquire与release
   - acquireShared与releaseShared
   - tryAcquire与tryRelease
   - Condition实现
   - 超时机制

4. **基于AQS的同步器实现分析**
   - ReentrantLock
   - ReentrantReadWriteLock
   - Semaphore
   - CountDownLatch
   - 内部实现对比

5. **自定义同步器**
   - 独占锁实现示例
   - 共享锁实现示例
   - 条件变量使用
   - 最佳实践

6. **AQS在并发框架中的应用**
   - 线程池中的应用
   - 并发容器中的应用
   - 源码级案例分析 

## 1. AQS基本概念

### 1.1 什么是AQS
AbstractQueuedSynchronizer（简称AQS）是Java并发包（java.util.concurrent）中的核心框架，它提供了一个用于实现阻塞锁和相关同步器的基础框架。AQS的设计基于模板方法模式，通过继承AQS并实现其抽象方法，可以轻松构建出各种同步器。

### 1.2 设计思想
AQS的核心设计思想包括：
- **状态管理**：使用一个volatile整型变量（state）表示同步状态
- **FIFO等待队列**：基于CLH锁队列变体实现的等待线程队列
- **模板方法设计模式**：提供框架，子类通过实现特定方法定制行为
- **两种模式**：独占模式（如ReentrantLock）和共享模式（如Semaphore、CountDownLatch）

### 1.3 框架结构
AQS的主要组成部分：
- **同步状态（state）**：volatile修饰的int变量
- **等待队列**：CLH变体的双向链表队列
- **Node节点**：队列中的节点，包含线程引用和等待状态等信息
- **ConditionObject**：条件变量实现，提供await/signal机制

```java
public abstract class AbstractQueuedSynchronizer 
    extends AbstractOwnableSynchronizer implements java.io.Serializable {
    
    // 同步状态
    private volatile int state;
    
    // 队列头节点
    private transient volatile Node head;
    
    // 队列尾节点
    private transient volatile Node tail;
    
    // 内部类Node定义
    static final class Node {
        volatile int waitStatus; // 等待状态
        volatile Node prev;      // 前驱节点
        volatile Node next;      // 后继节点
        volatile Thread thread;  // 关联的线程
        Node nextWaiter;         // 条件队列中的后继节点
    }
}
```

### 1.4 同步状态管理
- **state变量**：通过volatile修饰确保可见性
- **getState(), setState(), compareAndSetState()**：原子性操作方法
- **子类通过修改state值实现锁的获取与释放**

## 2. AQS核心实现原理

### 2.1 独占模式与共享模式
AQS支持两种同步模式：

**独占模式（Exclusive）**：
- 同一时刻只允许一个线程获取同步状态
- 代表实现：ReentrantLock
- 核心方法：acquire()、release()

**共享模式（Shared）**：
- 同一时刻可以允许多个线程获取同步状态
- 代表实现：Semaphore、CountDownLatch
- 核心方法：acquireShared()、releaseShared()

### 2.2 同步队列的实现（CLH队列）
AQS使用一个改进的CLH队列来管理等待获取同步状态的线程：
- **双向链表结构**：便于寻找前驱和后继节点
- **FIFO原则**：先进先出的公平性保证
- **节点状态变化**：通过waitStatus标记节点状态
- **基于CAS操作**：保证入队和出队操作的线程安全

CLH队列节点状态（waitStatus）：
- **CANCELLED(1)**：线程已取消获取锁
- **SIGNAL(-1)**：后继节点需要被唤醒
- **CONDITION(-2)**：节点在条件队列中等待
- **PROPAGATE(-3)**：用于共享模式，表示状态需要向后传播
- **0**：初始状态

### 2.3 独占模式的获取与释放
独占模式获取流程（acquire方法）：
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && 
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

处理步骤：
1. 尝试获取锁（tryAcquire）- 由子类实现
2. 获取失败，创建节点并加入队列（addWaiter）
3. 在队列中等待获取锁（acquireQueued）
4. 如有需要，恢复线程的中断状态（selfInterrupt）

释放流程（release方法）：
```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

### 2.4 共享模式的获取与释放
共享模式获取流程（acquireShared方法）：
```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

共享模式释放（releaseShared方法）：
```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

### 2.5 超时获取机制
AQS支持带超时的获取操作（tryAcquireNanos）：
```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout) {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) || 
           doAcquireNanos(arg, nanosTimeout);
}
```

### 2.6 条件变量实现
AQS内部类ConditionObject实现了Condition接口：
- **条件队列**：与同步队列分离的另一个等待队列
- **await**：将当前线程加入条件队列等待
- **signal/signalAll**：将条件队列中的线程转移到同步队列

## 3. AQS源码分析

### 3.1 acquire与release详解
**acquire方法**的详细流程：
1. 调用子类实现的tryAcquire尝试获取锁
2. 获取失败，通过addWaiter将当前线程封装成Node加入队列
3. acquireQueued方法中线程通过自旋方式获取锁
4. 获取失败则park当前线程，等待unpark唤醒

典型源码执行路径：
```java
// 入口方法
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

// 加入等待队列
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // 快速尝试在队尾添加
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 快速添加失败，进入完整的入队流程
    enq(node);
    return node;
}

// 在队列中等待获取锁
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 前驱是头节点时尝试获取锁
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // 帮助GC
                failed = false;
                return interrupted;
            }
            // 判断是否需要park当前线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### 3.2 Condition实现原理
Condition的实现位于AQS的内部类ConditionObject中：
- **await**：将线程从同步队列转移到条件队列
- **signal**：将线程从条件队列转移到同步队列

流程图示：
1. 线程持有锁 -> 调用await -> 释放锁 -> 进入条件队列 -> 等待signal
2. 其他线程持有锁 -> 调用signal -> 条件队列头节点转移到同步队列 -> 等待获取锁

## 4. 基于AQS的同步器实现分析

### 4.1 ReentrantLock
ReentrantLock是可重入独占锁，内部通过继承AQS实现：
- **公平锁（FairSync）**和**非公平锁（NonfairSync）**两种实现
- 核心是对tryAcquire和tryRelease的实现

非公平锁tryAcquire实现：
```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    // 状态为0表示锁未被持有
    if (c == 0) {
        // CAS尝试获取锁
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 当前线程已持有锁，重入
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // 溢出检查
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

### 4.2 ReentrantReadWriteLock
ReentrantReadWriteLock实现了读写锁：
- **读锁（共享锁）**：允许多个线程同时读取
- **写锁（独占锁）**：只允许一个线程写入
- **状态分解**：state的高16位表示读锁次数，低16位表示写锁次数

### 4.3 Semaphore
Semaphore实现了信号量机制：
- 基于AQS的共享模式实现
- state表示许可证的数量
- tryAcquireShared尝试获取许可
- tryReleaseShared尝试释放许可

### 4.4 CountDownLatch
CountDownLatch用于线程同步：
- 基于AQS的共享模式实现
- state表示计数器值
- countDown方法递减计数器
- await等待计数器归零

## 5. 自定义同步器

### 5.1 独占锁实现示例
自定义不可重入独占锁（互斥锁）：

```java
public class Mutex {
    private static class Sync extends AbstractQueuedSynchronizer {
        // 是否处于占用状态
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        // 尝试获取锁
        public boolean tryAcquire(int acquires) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        // 尝试释放锁
        protected boolean tryRelease(int releases) {
            if (getState() == 0) 
                throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
        
        // 提供条件变量
        Condition newCondition() { 
            return new ConditionObject(); 
        }
    }
    
    private final Sync sync = new Sync();
    
    public void lock() { sync.acquire(1); }
    public boolean tryLock() { return sync.tryAcquire(1); }
    public void unlock() { sync.release(1); }
    public Condition newCondition() { return sync.newCondition(); }
    public boolean isLocked() { return sync.isHeldExclusively(); }
}
```

### 5.2 共享锁实现示例
简单的二元信号量BinarySemaphore实现：

```java
public class BinarySemaphore {
    private static class Sync extends AbstractQueuedSynchronizer {
        // 初始可用信号量为1
        Sync() { setState(1); }
        
        // 获取共享锁
        protected int tryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 || 
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
        
        // 释放共享锁
        protected boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next > 1) // 最多只能有1个许可
                    throw new Error("Maximum permits exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
    }
    
    private final Sync sync = new Sync();
    
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    
    public void release() {
        sync.releaseShared(1);
    }
}
```

## 6. AQS在并发框架中的应用

### 6.1 线程池中的应用
线程池ExecutorService中的使用：
- **ThreadPoolExecutor**使用AQS实现的Worker内部锁
- **FutureTask**使用AQS管理任务状态和等待/通知机制

### 6.2 并发容器中的应用
并发容器中的使用：
- **LinkedBlockingQueue**等阻塞队列使用AQS实现阻塞和唤醒
- **CyclicBarrier**内部使用ReentrantLock和Condition

### 6.3 面试常见问题与解答

#### Q1: AQS的核心思想是什么？
A: AQS核心思想是基于volatile状态变量和FIFO队列实现的等待-通知机制。通过继承AQS并实现其模板方法，可以轻松构建各种同步器。

#### Q2: ReentrantLock和synchronized的区别？
A: ReentrantLock基于AQS实现，提供更高级的特性：
- 可选的公平锁机制
- 可中断的锁获取
- 超时获取锁
- 可实现多条件变量
- 非阻塞地获取锁的能力
- 可实现更灵活的锁操作

#### Q3: 什么是AQS的独占模式和共享模式？
A: 独占模式同一时刻只允许一个线程获取同步状态（如ReentrantLock）；共享模式允许多个线程同时获取同步状态（如Semaphore、CountDownLatch）。

#### Q4: 如何基于AQS自定义一个同步器？
A: 继承AbstractQueuedSynchronizer并实现以下方法：
- 独占模式：tryAcquire()、tryRelease()
- 共享模式：tryAcquireShared()、tryReleaseShared()
- 按需实现：isHeldExclusively()

## 7. 总结与实践建议

### 7.1 AQS的设计精髓
- **模板方法模式**：固定整体算法框架，子类实现特定步骤
- **状态管理与CAS**：通过volatile状态和CAS操作保证线程安全
- **阻塞与唤醒机制**：基于LockSupport的park/unpark
- **等待队列**：高效的双向队列实现公平性

### 7.2 实际应用建议
- 深入理解AQS原理后再使用第三方库提供的同步器
- 设计自定义同步器时，参考已有实现避免常见陷阱
- 使用CountDownLatch处理一次性等待/通知
- 使用CyclicBarrier处理可重用的同步点
- 使用Semaphore控制并发访问量

### 7.3 性能优化方向
- 减少锁竞争：分段锁、减小锁粒度
- 适当使用非公平锁提高吞吐量
- 避免长时间持有锁
- 使用读写锁分离读写操作
- 考虑无锁数据结构替代基于AQS的锁 