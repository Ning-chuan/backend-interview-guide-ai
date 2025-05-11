# 第2章：Java锁机制

## 章节概述
本章深入讲解Java中的各种锁机制，包括底层实现原理、适用场景以及性能特性。锁是Java并发编程中最基础也是最重要的同步机制，理解锁的工作原理对于编写高效且线程安全的程序至关重要。

## 什么是锁？
在多线程环境下，当多个线程同时访问共享资源时，可能会导致数据不一致的问题。锁的主要作用是保护共享资源，确保在同一时刻只有一个线程可以访问该资源，从而避免并发问题。

## 主要内容

### 1. **synchronized关键字**

#### 基本使用方法
`synchronized`是Java中最基本的同步机制，它可以应用于方法或代码块。

**修饰实例方法**：锁定当前对象实例
```java
public synchronized void method() {
    // 线程安全的代码
}
```

**修饰静态方法**：锁定当前类的Class对象
```java
public static synchronized void staticMethod() {
    // 线程安全的代码
}
```

**修饰代码块**：可以指定锁对象
```java
public void method() {
    synchronized(this) {  // 锁定当前对象
        // 线程安全的代码
    }
    
    synchronized(ClassName.class) {  // 锁定类对象
        // 线程安全的代码
    }
    
    Object lock = new Object();
    synchronized(lock) {  // 锁定指定对象
        // 线程安全的代码
    }
}
```

#### 锁对象选择（对象锁、类锁）
- **对象锁**：锁定特定的对象实例，不同对象实例之间互不影响
- **类锁**：锁定类的Class对象，影响所有实例

#### 底层实现原理
`synchronized`在JVM中通过监视器（Monitor）实现，具体涉及到对象头（Object Header）中的Mark Word部分：

1. **Monitor**：每个对象都与一个监视器关联
2. **对象头**：包含Mark Word（存储锁状态、哈希码等）和Klass Pointer（指向类元数据）
3. **字节码**：使用monitorenter和monitorexit指令实现同步

#### 锁优化
随着JDK的发展，synchronized经历了多次优化：

- **偏向锁**：同一线程多次获取同一把锁，避免了不必要的CAS操作
- **轻量级锁**：多个线程交替执行同步块时，通过CAS操作避免重量级锁
- **重量级锁**：多个线程同时竞争时，通过操作系统的互斥量（Mutex）实现

**锁升级过程**：无锁 → 偏向锁 → 轻量级锁 → 重量级锁（不可逆过程）

**示例：多线程访问计数器**
```java
public class Counter {
    private int count = 0;
    
    public synchronized void increment() {
        count++;
    }
    
    public synchronized int getCount() {
        return count;
    }
    
    public static void main(String[] args) throws InterruptedException {
        Counter counter = new Counter();
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                counter.increment();
            }
        });
        
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                counter.increment();
            }
        });
        
        t1.start();
        t2.start();
        
        t1.join();
        t2.join();
        
        System.out.println("最终计数: " + counter.getCount());  // 输出: 20000
    }
}
```

### 2. **Lock接口与ReentrantLock**

Java 5引入了`java.util.concurrent.locks`包，提供了更灵活的锁实现。

#### ReentrantLock的基本使用
ReentrantLock是Lock接口最常用的实现类，它提供了与synchronized相似的独占锁功能，但具有更多的灵活性。

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class ReentrantLockExample {
    private final Lock lock = new ReentrantLock();
    private int count = 0;
    
    public void increment() {
        lock.lock();  // 获取锁
        try {
            count++;
        } finally {
            lock.unlock();  // 释放锁（必须在finally块中）
        }
    }
    
    public int getCount() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

**注意**：使用ReentrantLock必须手动释放锁，通常在finally块中进行，避免因异常导致锁未释放

#### 公平锁与非公平锁
- **非公平锁**（默认）：不保证等待线程的顺序，可能导致某些线程饥饿，但吞吐量更高
  ```java
  Lock unfairLock = new ReentrantLock(); // 默认非公平锁
  ```

- **公平锁**：按照线程请求的顺序获取锁，等待时间最长的线程优先获取锁
  ```java
  Lock fairLock = new ReentrantLock(true); // 公平锁
  ```

#### 可中断锁
ReentrantLock支持中断响应，可以避免死锁问题：

```java
Lock lock = new ReentrantLock();
try {
    // 可中断地获取锁
    lock.lockInterruptibly();
    // 执行临界区代码
} catch (InterruptedException e) {
    // 响应中断
    System.out.println("线程被中断");
} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

#### 尝试获取锁
```java
Lock lock = new ReentrantLock();
if (lock.tryLock()) {  // 尝试获取锁，立即返回结果
    try {
        // 获取到锁后的操作
    } finally {
        lock.unlock();
    }
} else {
    // 获取锁失败的处理逻辑
}

// 带超时的尝试获取锁
if (lock.tryLock(1, TimeUnit.SECONDS)) {  // 尝试在1秒内获取锁
    try {
        // 获取到锁后的操作
    } finally {
        lock.unlock();
    }
} else {
    // 超时未获取到锁的处理逻辑
}
```

#### 与synchronized的对比
| 特性 | synchronized | ReentrantLock |
|------|--------------|---------------|
| 灵活性 | 低（隐式获取和释放） | 高（显式控制） |
| 公平性 | 非公平 | 可选（公平/非公平） |
| 可中断 | 不支持 | 支持 |
| 超时获取 | 不支持 | 支持 |
| 用法 | 关键字，简单 | 类，较复杂 |
| 性能 | JDK 6后性能已优化 | 与synchronized相当 |

### 3. **读写锁（ReentrantReadWriteLock）**

读写锁分离了读和写的锁定机制，允许多个线程同时读取共享资源，但写操作需要独占访问。

#### 读写锁原理
- **读锁**（共享锁）：多个线程可以同时持有读锁
- **写锁**（排他锁）：只允许一个线程持有，且与读锁互斥

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ReadWriteLockExample {
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private int data = 0;
    
    public int readData() {
        rwLock.readLock().lock();  // 获取读锁
        try {
            return data;
        } finally {
            rwLock.readLock().unlock();  // 释放读锁
        }
    }
    
    public void writeData(int newData) {
        rwLock.writeLock().lock();  // 获取写锁
        try {
            data = newData;
        } finally {
            rwLock.writeLock().unlock();  // 释放写锁
        }
    }
}
```

#### 读写锁的应用场景
- 读多写少的场景（如缓存）
- 大量并发读取，但写入较少的数据结构

#### 写锁降级
ReentrantReadWriteLock支持写锁降级为读锁，但不支持读锁升级为写锁：

```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();
rwLock.writeLock().lock();  // 获取写锁

try {
    // 修改共享数据
    data = newValue;
    
    // 获取读锁（写锁降级）
    rwLock.readLock().lock();
} finally {
    rwLock.writeLock().unlock();  // 释放写锁，但保持读锁
}

try {
    // 使用共享数据（此时仍持有读锁）
    useData(data);
} finally {
    rwLock.readLock().unlock();  // 释放读锁
}
```

### 4. **StampedLock**

Java 8引入的StampedLock提供了比ReadWriteLock更高级的功能，包括乐观读模式：

#### 基本使用

```java
import java.util.concurrent.locks.StampedLock;

public class StampedLockExample {
    private double x, y;
    private final StampedLock sl = new StampedLock();
    
    // 写入数据
    public void move(double deltaX, double deltaY) {
        long stamp = sl.writeLock();  // 获取写锁
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            sl.unlockWrite(stamp);  // 释放写锁
        }
    }
    
    // 读取数据（悲观读）
    public double distanceFromOrigin() {
        long stamp = sl.readLock();  // 获取读锁
        try {
            return Math.hypot(x, y);
        } finally {
            sl.unlockRead(stamp);  // 释放读锁
        }
    }
    
    // 乐观读
    public double distanceFromOriginOptimistic() {
        // 尝试乐观读
        long stamp = sl.tryOptimisticRead();
        double currentX = x, currentY = y;
        
        // 检查在读取过程中数据是否被修改
        if (!sl.validate(stamp)) {
            // 数据已被修改，退化为悲观读
            stamp = sl.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        
        return Math.hypot(currentX, currentY);
    }
}
```

#### 与ReadWriteLock的对比
- StampedLock不支持重入
- StampedLock有乐观读模式，性能更好
- StampedLock不是以继承AQS方式实现的
- StampedLock提供了三种模式：写、读、乐观读

### 5. **锁的底层实现**

#### CAS操作
比较并交换（Compare And Swap）是一种原子操作，是Java锁实现的基础：

```java
// 伪代码表示CAS操作
boolean compareAndSet(V expected, V newValue) {
    if (当前值 == expected) {
        当前值 = newValue;
        return true;
    } else {
        return false;
    }
}
```

Java中的CAS操作通过Unsafe类的native方法实现，底层依赖于CPU的原子指令（如x86架构的CMPXCHG指令）。

#### AQS（AbstractQueuedSynchronizer）
AQS是Java中大多数同步器的基础框架，包括ReentrantLock、Semaphore、CountDownLatch等：

- **状态管理**：通过一个int类型的state变量表示同步状态
- **队列管理**：使用CLH队列管理等待的线程
- **独占/共享模式**：支持独占模式和共享模式

自定义同步器需要重写以下方法：
- `tryAcquire()`：尝试获取独占锁
- `tryRelease()`：尝试释放独占锁
- `tryAcquireShared()`：尝试获取共享锁
- `tryReleaseShared()`：尝试释放共享锁
- `isHeldExclusively()`：当前同步器是否被独占

#### CLH队列锁
CLH（Craig, Landin, and Hagersten）队列是一种高效的自旋锁，用于实现线程等待队列：

- 基于链表实现的FIFO自旋锁
- 每个线程通过自旋等待前驱节点的状态变化
- AQS中的等待队列基于CLH队列的变种实现

### 6. **死锁问题**

#### 死锁的四个必要条件
1. **互斥条件**：资源不能被共享，一次只能一个线程使用
2. **请求与保持条件**：线程已获得资源，但又提出新的资源请求
3. **不剥夺条件**：线程已获得的资源，在未使用完之前，不能被强行剥夺
4. **循环等待条件**：线程之间形成环形的资源等待关系

#### 死锁示例
```java
public class DeadlockExample {
    private static final Object RESOURCE_A = new Object();
    private static final Object RESOURCE_B = new Object();
    
    public static void main(String[] args) {
        Thread threadA = new Thread(() -> {
            synchronized (RESOURCE_A) {
                System.out.println("线程A获取资源A");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (RESOURCE_B) {
                    System.out.println("线程A获取资源B");
                }
            }
        });
        
        Thread threadB = new Thread(() -> {
            synchronized (RESOURCE_B) {
                System.out.println("线程B获取资源B");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (RESOURCE_A) {
                    System.out.println("线程B获取资源A");
                }
            }
        });
        
        threadA.start();
        threadB.start();
    }
}
```

#### 死锁的检测与预防
- **检测**：
  - 使用jstack命令查看线程堆栈信息
  - 使用JConsole或VisualVM等工具检测
  
- **预防**：
  - 按固定顺序获取锁
  - 使用限时锁（tryLock(timeout)）
  - 避免嵌套锁
  - 使用线程池和并发工具类
  
#### 实际案例解决方案
1. **固定锁顺序**：确保所有线程按照相同的顺序获取锁
   ```java
   // 修改后的代码
   public void transferMoney(Account fromAccount, Account toAccount, int amount) {
       // 按照账户ID大小顺序获取锁
       Account firstLock = fromAccount.getId() < toAccount.getId() ? fromAccount : toAccount;
       Account secondLock = fromAccount.getId() < toAccount.getId() ? toAccount : fromAccount;
       
       synchronized (firstLock) {
           synchronized (secondLock) {
               // 转账逻辑
           }
       }
   }
   ```

2. **使用tryLock避免死锁**：
   ```java
   public boolean transferMoney(Account fromAccount, Account toAccount, int amount, long timeout) {
       long stopTime = System.currentTimeMillis() + timeout;
       
       while (System.currentTimeMillis() < stopTime) {
           if (fromAccount.lock.tryLock()) {
               try {
                   if (toAccount.lock.tryLock()) {
                       try {
                           // 转账逻辑
                           return true;
                       } finally {
                           toAccount.lock.unlock();
                       }
                   }
               } finally {
                   fromAccount.lock.unlock();
               }
           }
           // 短暂休眠避免CPU空转
           Thread.sleep(50);
       }
       return false; // 超时返回失败
   }
   ```

## 总结与最佳实践
1. **选择合适的锁**：
   - 简单场景：synchronized
   - 复杂需求（超时、中断等）：ReentrantLock
   - 读多写少：ReadWriteLock
   - 高并发读场景：StampedLock

2. **减小锁粒度**：锁定必要的代码段而非整个方法

3. **避免锁嵌套**：减少死锁风险

4. **考虑非阻塞算法**：在适当场景使用CAS等非阻塞算法

5. **使用并发工具类**：优先使用Java提供的线程安全集合和并发工具 