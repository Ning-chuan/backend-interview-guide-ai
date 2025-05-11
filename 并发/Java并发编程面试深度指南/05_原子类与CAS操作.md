# 第5章：原子类与CAS操作

## 章节概述
本章介绍Java中的原子类以及底层的CAS操作原理，讲解无锁编程的实现方式与应用场景。在多线程环境下，原子类为我们提供了一种无锁的线程安全实现机制，相比传统的synchronized和Lock，它们通常具有更好的性能和扩展性。

## 主要内容
1. **CAS基础**
   - Compare And Swap原理
   - 底层实现（硬件指令支持）
   - CAS的三大问题（ABA问题、循环时间长、只能保证一个共享变量的原子操作）
   - 解决方案

2. **基本原子类**
   - AtomicInteger、AtomicLong、AtomicBoolean
   - 常用方法实现原理
   - 性能特性
   - 与锁机制的对比

3. **数组类型原子类**
   - AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray
   - 实现原理
   - 应用场景

4. **引用类型原子类**
   - AtomicReference
   - AtomicStampedReference（解决ABA问题）
   - AtomicMarkableReference
   - 使用示例

5. **字段更新器**
   - AtomicIntegerFieldUpdater
   - AtomicLongFieldUpdater
   - AtomicReferenceFieldUpdater
   - 适用场景与限制

6. **Java 8增强的原子类**
   - LongAdder与LongAccumulator
   - DoubleAdder与DoubleAccumulator
   - 分段累加机制
   - 与AtomicLong的性能对比
   - 高并发场景下的应用 

## 1. CAS基础

### 1.1 Compare And Swap原理
CAS（Compare And Swap）是一种无锁算法，用于解决多线程并发安全问题。它的基本思想是：

- **比较(Compare)**: 读取内存中的预期值
- **交换(Swap)**: 如果预期值与内存中的实际值相同，则将内存值更新为新值
- **结果**: 如果更新成功返回true，否则返回false

CAS的操作过程是原子性的，确保了在多线程环境下数据一致性。

```java
// CAS的伪代码实现
boolean compareAndSwap(V* address, V expectedValue, V newValue) {
    // 原子性操作
    if (*address == expectedValue) {
        *address = newValue;
        return true;
    }
    return false;
}
```

### 1.2 底层实现（硬件指令支持）
在Java中，CAS操作是通过**Unsafe**类实现的，它提供了直接访问底层资源的能力。CAS的底层依赖于处理器提供的原子指令：

- **x86架构**: 使用`cmpxchg`指令
- **SPARC架构**: 使用`cas`指令
- **ARM架构**: 使用`ldrex`和`strex`指令

这些指令在CPU级别保证了操作的原子性，通常会锁定总线或使用内存屏障来实现。

```java
// Unsafe类中的CAS方法
public final native boolean compareAndSwapInt(Object o, long offset, int expected, int x);
public final native boolean compareAndSwapLong(Object o, long offset, long expected, long x);
public final native boolean compareAndSwapObject(Object o, long offset, Object expected, Object x);
```

### 1.3 CAS的三大问题

#### 1.3.1 ABA问题
**问题描述**: 如果一个值从A变成B，再变回A，使用CAS检查时会认为这个值没有被修改过，但实际上已经经历了A->B->A的变化过程。

**示例场景**:
1. 线程1读取值A
2. 线程1被挂起
3. 线程2将值从A修改为B，再修改回A
4. 线程1恢复执行，发现值仍为A，CAS操作成功
5. 但实际上值已经被修改过

#### 1.3.2 循环时间长开销大
**问题描述**: CAS操作如果长时间不成功，会一直进行自旋重试，消耗CPU资源。

```java
// 典型的CAS自旋实现
public void increment() {
    int oldValue;
    do {
        oldValue = value; // 读取当前值
    } while (!compareAndSet(oldValue, oldValue + 1)); // 不成功就一直重试
}
```

#### 1.3.3 只能保证一个共享变量的原子操作
**问题描述**: 单个CAS操作只能保证一个共享变量的原子更新，无法直接实现多个变量的原子性更新。

### 1.4 解决方案

#### 1.4.1 解决ABA问题
Java提供了**AtomicStampedReference**和**AtomicMarkableReference**类来解决ABA问题：

- **AtomicStampedReference**: 不仅比较值，还比较版本号（stamp）
- **AtomicMarkableReference**: 使用布尔标记表示值是否被修改过

```java
// AtomicStampedReference使用示例
AtomicStampedReference<Integer> atomicRef = new AtomicStampedReference<>(100, 0);
int stamp = atomicRef.getStamp(); // 获取当前版本号
atomicRef.compareAndSet(100, 101, stamp, stamp + 1); // 更新值的同时更新版本号
```

#### 1.4.2 解决循环时间长问题
- 限制自旋次数，超过阈值后采用其他同步机制
- 使用Java 8引入的`LongAdder`等累加器类（内部使用分段锁减少冲突）

#### 1.4.3 解决多变量原子更新
- 使用`AtomicReference`包装多个变量为一个对象
- 使用锁机制

## 2. 基本原子类

### 2.1 AtomicInteger、AtomicLong、AtomicBoolean
这些类提供了对整型、长整型和布尔类型的原子操作支持，它们都继承自`Number`类（`AtomicBoolean`除外）。

#### 2.1.1 核心方法

```java
// AtomicInteger常用方法
public final int get()                  // 获取当前值
public final void set(int newValue)     // 设置值
public final int getAndSet(int newValue) // 设置新值并返回旧值
public final boolean compareAndSet(int expect, int update) // CAS操作
public final int getAndIncrement()      // 自增并返回旧值
public final int getAndDecrement()      // 自减并返回旧值
public final int getAndAdd(int delta)   // 增加指定值并返回旧值
public final int incrementAndGet()      // 自增并返回新值
public final int decrementAndGet()      // 自减并返回新值
public final int addAndGet(int delta)   // 增加指定值并返回新值
```

#### 2.1.2 实现原理
以`AtomicInteger`的`incrementAndGet()`方法为例：

```java
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}

// Unsafe类中的实现
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);
    } while (!compareAndSwapInt(o, offset, v, v + delta));
    return v;
}
```

它通过Unsafe类的`compareAndSwapInt`方法实现，该方法是一个本地方法，底层依赖CPU的原子指令。

### 2.2 性能特性
- 在**低竞争**环境下，原子类通常比锁机制有更好的性能
- 在**高竞争**环境下，持续的自旋会导致性能下降
- 原子类适合简单的原子操作，复杂的条件控制仍需使用锁

### 2.3 与锁机制的对比

| 特性 | 原子类（CAS） | 锁（synchronized/Lock） |
|-----|------------|----------------------|
| 实现机制 | 无锁（乐观锁思想） | 互斥锁（悲观锁思想） |
| 线程状态 | 不会阻塞线程 | 可能导致线程阻塞 |
| 适用场景 | 简单的原子操作，低竞争 | 复杂共享资源访问，高竞争 |
| 粒度 | 细粒度，单个变量 | 可控制粒度，保护多个变量 |
| 死锁风险 | 无 | 有 |
| CPU消耗 | 自旋消耗CPU | 阻塞不消耗CPU |

## 3. 数组类型原子类

### 3.1 AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray
这些类提供了对整型数组、长整型数组和引用类型数组的原子操作支持。

#### 3.1.1 常用方法

```java
// AtomicIntegerArray常用方法
public final int get(int i)                 // 获取索引i的元素
public final void set(int i, int newValue)  // 设置索引i的元素为新值
public final int getAndSet(int i, int newValue) // 设置新值并返回旧值
public final boolean compareAndSet(int i, int expect, int update) // CAS操作
public final int getAndIncrement(int i)     // 自增并返回旧值
public final int getAndDecrement(int i)     // 自减并返回旧值
public final int getAndAdd(int i, int delta) // 增加指定值并返回旧值
```

#### 3.1.2 使用示例

```java
// 创建原子数组
AtomicIntegerArray array = new AtomicIntegerArray(10); // 长度为10的数组
array.set(0, 100); // 设置第一个元素为100
int old = array.getAndAdd(0, 10); // 第一个元素加10，返回原值100
System.out.println("old value: " + old + ", new value: " + array.get(0)); // 输出: old value: 100, new value: 110
```

### 3.2 实现原理
数组类型的原子类与基本类型原子类类似，也是基于Unsafe类的CAS操作。不同的是，它们操作的是数组中的元素。

```java
// AtomicIntegerArray中的getAndAdd方法实现
public final int getAndAdd(int i, int delta) {
    return unsafe.getAndAddInt(array, checkedByteOffset(i), delta);
}

// 计算数组元素的偏移量
private long checkedByteOffset(int i) {
    if (i < 0 || i >= array.length)
        throw new IndexOutOfBoundsException("index " + i);
    return byteOffset(i);
}

private static long byteOffset(int i) {
    return ((long) i << shift) + base;
}
```

### 3.3 应用场景
- 需要原子更新数组元素的场景
- 计数器数组
- 标志位数组
- 缓存状态管理

## 4. 引用类型原子类

### 4.1 AtomicReference
`AtomicReference`提供了对引用类型的原子操作支持，可以原子性地更新一个对象引用。

```java
// AtomicReference示例
AtomicReference<User> userRef = new AtomicReference<>(new User("张三", 20));
User user = userRef.get();
User newUser = new User("李四", 25);
userRef.compareAndSet(user, newUser); // CAS更新
```

### 4.2 AtomicStampedReference（解决ABA问题）
`AtomicStampedReference`在`AtomicReference`的基础上增加了版本号，可以解决ABA问题。

```java
// AtomicStampedReference示例
AtomicStampedReference<Integer> stampedRef = new AtomicStampedReference<>(100, 0);
// 获取当前版本号
int stamp = stampedRef.getStamp();
// 更新值和版本号
boolean success = stampedRef.compareAndSet(100, 101, stamp, stamp + 1);
```

### 4.3 AtomicMarkableReference
`AtomicMarkableReference`使用一个布尔标记表示引用是否被修改过，比`AtomicStampedReference`更轻量。

```java
// AtomicMarkableReference示例
AtomicMarkableReference<Integer> markableRef = new AtomicMarkableReference<>(100, false);
// 获取当前标记
boolean[] markHolder = new boolean[1];
Integer value = markableRef.get(markHolder);
boolean mark = markHolder[0];
// 更新值和标记
boolean success = markableRef.compareAndSet(value, 101, mark, true);
```

### 4.4 使用示例 - 模拟银行账户转账

```java
public class Account {
    private final String name;
    private int balance;
    
    public Account(String name, int balance) {
        this.name = name;
        this.balance = balance;
    }
    
    // getter/setter省略
}

// 使用AtomicReference实现转账
public class AtomicReferenceTransfer {
    private static AtomicReference<Account> accountRef = new AtomicReference<>(new Account("账户", 1000));
    
    public static void transfer(int amount) {
        Account oldAccount, newAccount;
        do {
            oldAccount = accountRef.get();
            if (oldAccount.getBalance() < amount) {
                throw new RuntimeException("余额不足");
            }
            newAccount = new Account(oldAccount.getName(), oldAccount.getBalance() - amount);
        } while (!accountRef.compareAndSet(oldAccount, newAccount));
    }
}
```

## 5. 字段更新器

字段更新器提供了一种更新对象中某个字段的原子方式，而不需要创建新的原子类实例。

### 5.1 AtomicIntegerFieldUpdater、AtomicLongFieldUpdater、AtomicReferenceFieldUpdater

```java
// 字段更新器使用示例
public class UserCounter {
    // 注意：字段必须是volatile修饰的
    volatile int count;
    
    // 创建更新器
    private static final AtomicIntegerFieldUpdater<UserCounter> COUNTER_UPDATER = 
        AtomicIntegerFieldUpdater.newUpdater(UserCounter.class, "count");
    
    public void increment() {
        COUNTER_UPDATER.incrementAndGet(this);
    }
    
    public int getCount() {
        return count;
    }
}
```

### 5.2 适用场景与限制
**适用场景**:
- 已有类需要部分字段原子操作
- 减少对象创建，节省内存

**限制**:
1. 字段必须是**volatile**修饰的
2. 字段不能是**static**的（除非使用静态方法创建更新器）
3. 字段不能是**final**的（无法修改final字段）
4. 必须有权限访问字段（private字段无法从外部类访问）

```java
// AtomicReferenceFieldUpdater示例
public class User {
    volatile String name;
    
    private static final AtomicReferenceFieldUpdater<User, String> NAME_UPDATER = 
        AtomicReferenceFieldUpdater.newUpdater(User.class, String.class, "name");
    
    public void updateName(String expectedName, String newName) {
        NAME_UPDATER.compareAndSet(this, expectedName, newName);
    }
}
```

## 6. Java 8增强的原子类

### 6.1 LongAdder与LongAccumulator

Java 8引入了`LongAdder`和`LongAccumulator`类，用于解决高并发环境下`AtomicLong`的性能问题。

```java
// LongAdder使用示例
LongAdder counter = new LongAdder();
counter.increment(); // 增加1
counter.add(10);     // 增加10
long sum = counter.sum(); // 获取当前值
```

### 6.2 DoubleAdder与DoubleAccumulator

`DoubleAdder`和`DoubleAccumulator`是`LongAdder`和`LongAccumulator`的浮点数版本。

```java
// DoubleAdder使用示例
DoubleAdder adder = new DoubleAdder();
adder.add(1.5);
double sum = adder.sum();

// DoubleAccumulator使用示例 - 计算最大值
DoubleAccumulator accumulator = new DoubleAccumulator(Double::max, 0.0);
accumulator.accumulate(10.5);
accumulator.accumulate(5.0);
double result = accumulator.get(); // 结果为10.5
```

### 6.3 分段累加机制
`LongAdder`和`DoubleAdder`的核心思想是**分段锁**，将热点数据分散到不同的Cell（段）上。

原理：
1. 当竞争不激烈时，直接更新base值（类似AtomicLong）
2. 当竞争激烈时，会创建多个Cell单元，每个线程更新自己对应的Cell
3. 最终结果是所有Cell值与base值的总和

这种设计巧妙地避免了高并发下的资源竞争，大大提高了性能。

### 6.4 与AtomicLong的性能对比

| 特性 | AtomicLong | LongAdder |
|-----|------------|-----------|
| 实现原理 | CAS自旋 | 分段累加+CAS |
| 内存占用 | 低 | 高（多个Cell） |
| 低并发性能 | 好 | 稍差 |
| 高并发性能 | 较差 | 极好 |
| 适用场景 | 低竞争、需要立即读取精确值 | 高竞争、统计汇总 |

```java
// 性能对比示例代码
AtomicLong atomicLong = new AtomicLong(0);
LongAdder longAdder = new LongAdder();

// 10个线程各自累加100万次
ExecutorService executorService = Executors.newFixedThreadPool(10);
long start = System.currentTimeMillis();
for (int i = 0; i < 10; i++) {
    executorService.submit(() -> {
        for (int j = 0; j < 1000000; j++) {
            atomicLong.incrementAndGet();
        }
    });
}
// 等待执行完成...
System.out.println("AtomicLong耗时: " + (System.currentTimeMillis() - start) + "ms");

// 使用LongAdder测试...
// 通常LongAdder的性能会比AtomicLong高出很多
```

### 6.5 高并发场景下的应用
- **计数器**: 网站访问量统计
- **度量指标收集**: 性能监控系统
- **并发限流器**: 限制并发请求数
- **热点数据缓存**: 减少缓存更新冲突

## 面试要点

1. **CAS原理及实现**
   - CAS操作如何保证原子性？
   - Unsafe类的作用是什么？
   - CAS在处理器层面是如何实现的？

2. **ABA问题**
   - 什么是ABA问题？如何在实际应用中避免？
   - AtomicStampedReference和AtomicMarkableReference的区别是什么？

3. **乐观锁vs悲观锁**
   - CAS是乐观锁还是悲观锁？为什么？
   - 什么场景下乐观锁性能更好？什么场景下悲观锁更合适？

4. **Java 8新增原子类**
   - LongAdder为什么比AtomicLong性能更好？
   - LongAdder内部是如何减少线程竞争的？

5. **原子类使用注意事项**
   - 原子类能否保证复合操作的原子性？
   - 多个原子变量操作如何保证原子性？

记住：原子类适用于简单的原子操作场景，对于复杂的业务逻辑和多个变量的协调更新，仍然需要使用锁机制来确保线程安全。 