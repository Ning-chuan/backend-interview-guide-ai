# 第2章：Java锁机制

## 章节概述
本章深入讲解Java中的各种锁机制，包括底层实现原理、适用场景以及性能特性。

## 主要内容
1. **synchronized关键字**
   - 基本使用方法
   - 锁对象选择（对象锁、类锁）
   - 底层实现原理（monitor、对象头、锁升级）
   - 锁优化（偏向锁、轻量级锁、重量级锁）

2. **Lock接口与ReentrantLock**
   - ReentrantLock的基本使用
   - 公平锁与非公平锁
   - 可中断锁
   - 与synchronized的对比

3. **读写锁（ReentrantReadWriteLock）**
   - 读写锁原理
   - 读写锁的应用场景
   - 写锁降级

4. **StampedLock**
   - 乐观读（tryOptimisticRead）
   - 悲观读（readLock）
   - 写锁（writeLock）
   - 与ReadWriteLock的对比

5. **锁的底层实现**
   - CAS操作
   - AQS（AbstractQueuedSynchronizer）
   - CLH队列锁

6. **死锁问题**
   - 死锁的四个必要条件
   - 死锁的检测与预防
   - 实际案例分析 