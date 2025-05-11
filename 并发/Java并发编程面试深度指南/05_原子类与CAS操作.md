# 第5章：原子类与CAS操作

## 章节概述
本章介绍Java中的原子类以及底层的CAS操作原理，讲解无锁编程的实现方式与应用场景。

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