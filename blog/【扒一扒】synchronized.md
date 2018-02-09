---
title: 【扒一扒】synchronized
date: 2017/05/03 11:12:22
toc: false
list_number: false
categories:
- java
tags:
- synchronized
---

**前言：**最近看了很多关于volatile和synchronized的博客，很多说法理解不一，故此，这里整理我对于这两个关键字的理解。


## synchronized
### 1. Lock和Synchronized区别

1. `Synchronized`同步代码块，不能保证等待线程进入同步块的顺序。
2. `Synchronized` 不接受超时，不能被中断！
3. `Lock`中`unLock()`必须放到`finally{}`中，需要收到释放锁，`Synchronized`自动释放锁。
4. 锁竞争小时的性能：`Synchronized >= atom >= Lock`，竞争激烈时： `atom > Lock > Synchronized`


### 2. Synchronized的理解

synchronized加锁的类型
1. 对象锁: `synchronized (this)`或`synchronized (personObject)` ,`synchronized`修饰的成员方法，都是对象锁；
2. 类锁: `synchronized (A.class)` 和 `synchronized`修饰的静态方法，都是类锁；
3. `synchronized`包含的代码块。`synchronized (A.class){}`是类锁, `synchronized (this){}`是对象锁.

#### synchronized特点
1. 不论锁的是什么, 并不影响`类`或`对象`中没有用`synchronized`修饰的`方法`,`变量`被其他线程`访问`和`修改`.
2. `synchronized`锁住的`方法,代码块,对象`中的`变量`不会受到 `synchronized`的控制, 可以被其他线程随意读写.
3. **当一个线程获得了对象锁(类锁)，其他线程不能访问其他需要对象锁(类锁)的方法或代码块**
4. **对象锁和类锁，二者不冲突，当一个线程获得了A的对象锁，其他线程可以访问A中需要类锁的方法或代码块**
5. `synchronized (this){}`不能用于静态方法中, 但是`synchronized (A.class)`却能用于成员方法中,加的是类锁.

### 3. 锁的实现
对象锁和类锁有两种实现方式：方法上的synchronized是基于方法区中`ACC_SYNCHRONIZED`标志位实现，synchronized修饰的方法块基于`monitorenter`,` monitorexit `语义实现！
这两种实现方式，在重量级锁上，都依赖于操作系统提供的实现（管程(monitor)底层数据结构维护着请求该锁的线程的队列，及相关信息）。
> 重量级锁依赖于操作系统实现，加锁需要从用户态转换到内核态加锁，这就需要操作系统进行一次上下文切换，另外加锁、释放锁会导致比较多的上下文切换和调度延时，等待锁的线程会被挂起直至锁释放。在上下文切换的时候，cpu之前缓存的指令和数据都将失效，对性能有很大的损失。

#### synchronized修饰的成员方法和静态方法
成员方法： 

```
public synchronized void method(int, java.lang.String);
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=4, args_size=3
        0: aload_2       
        ...
        10: return   
```
静态方法：
```
public static synchronized int method2(int);
    flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: iload_0       
         ...
         4: return
```
获取锁和释放锁是在方法调用和方法返回的时候执行的，方法执行中如果抛出异常，那么锁会在抛出异常时释放。
JVM可以从方法常量池中的方法表结构(method_info Structure) 中的 ACC_SYNCHRONIZED 访问标志区分一个方法是否同步方法。


#### synchronized修饰的成员方法代码块和静态方法代码块
```
        5: monitorenter  
        ...
        12: monitorexit   
        13: goto          21
        ...     
        17: aload_1       
        18: monitorexit   
        ...     
        20: athrow        
        21: return
```
`monitorenter`,` monitorexit `语义是基于堆中实例化对象的对象头中的锁标志位实现。

其中，`ACC_SYNCHRONIZED`和`monitorenter，monitorexit`，在JVM上的语义不同但是在底层锁实现是一样的，因为`Class`被ClassLoader加载到内存中也是一个对象，也有对象头！
实例对象由：`对象头`，`实例变量`，`填充对齐数据`组成。
对象头由：`MarkWord`（存储对象的hashCode、锁信息或分代年龄或GC标志等信息），`Class Metadata Address`（指向方法区class的指针）

#### 锁原理锁优化
jdk1.6之前synchronized属于重量级锁，效率低下，因为监视器锁（monitor）是依赖于底层的操作系统的Mutex Lock来实现的，而操作系统实现线程之间的切换时需要从用户态转换到核心态，
这个状态之间的转换需要相对比较长的时间，时间成本相对较高。
重量级锁的实现是借助操作系统实现的，多种叫法：监视器/管程/monitor。
 如果被`Synchronized`修饰的方法或代码块，JVM探测出其没有竞争，会进行锁消除，即不使用锁！
jdk1.6之后JVM对synchronized的锁实现优化，首先在没有锁竞争的情况下加**偏向锁**，当另一个线程申请该锁时升级为**轻量级锁**，当同一时刻多个线程申请同一个锁时，JVM将该锁升级为**自旋锁**，当自旋一定次数后，升级为**重量级锁**，也就是依赖操作系统的`Mutex Lock`来实现。


## 参考
1. http://blog.csdn.net/javazejian/article/details/72828483
2. http://blog.csdn.net/javazejian/article/details/72772461


