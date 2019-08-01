---
title: Note -- Java 并发编程的艺术
date: 2019-07-23 00:04:13
tags: Java
categories: Tech
---

# [Note] Java 并发编程的艺术

原子性

32位处理器：
通过总线锁或对缓存加锁的方式

总线锁：处理器在总线上输出 #LOCK 信号
缓存锁：“缓存锁定”

Java：
通过锁和循环CAS的方式

<!-- more -->

循环CAS：循环进行CAS直到成功为止
问题解决：
ABA问题：加入版本号 AutomicStampedReference
循环时间开销大：JVM支持CPU提供的pause指令
只能保证一个共享变量的原子操作：使用锁；将多个共享变量合为一个变量；AutomicReference

锁机制：偏向锁、轻量级锁、互斥所


## 内存模型

线程之间的通信机制：共享内存、消息传递

Java的并发采用共享内存模型

JMM (Java内存模型) 控制线程之间的通信，定义了线程与主内存之间的通信
线程之间的共享变量存储在主内存中
每个线程都有一个私有的本地内存，存储了该线程以读写共享变量的副本

线程A更新本地内存 --> 线程A刷主内存 --> 线程B读取主内存 --> 线程B更新本地内存


重排序的3种类型
1. 编译器优化的重排序
2. 指令级并行的重排序
3. 内存系统的重排序


**happens-before**

前一个操作执行的结果对后一个操作可见，且前一个操作按顺序排在第二个操作之前
不一位置前一个操作必须在后一个操作之前执行

- 程序顺序规则
- 监视器锁规则
- volatile变量规则
- 传递性

**数据依赖性**

- 写后读
- 写后写
- 读后写

**as-if-serial**

不管怎么重排序，单线程程序的执行结果不能被改变

**猜测执行**

在多线程中，对存在控制依赖的操作重排序，可能会改变程序的运行结果


### 顺序一致性

**数据竞争**

**顺序一致性内存模型**

理想的一致性内存模型

1. 一个线程中的所有操作必须按照程序的顺序来执行
2. 所有线程都只能看到一个单一的操作执行顺序。每个操作都必须原子执行且立刻对所有线程可见

JMM中没有这个保证

在JMM中，临界区内的代码可以重排序

在不改变（正确同步的）程序执行结果的前提下，尽可能的为编译器和处理器的优化打开方便之门

**未同步程序的执行特性**

- 单线程内的操作不保证按程序的顺序执行
- 不保证所有线程能看到一致的操作执行顺序
- 不保证对64位的long型和double型变量的写操作具有原子性
  - 性能考虑，JVM可能拆分为两个32位的写操作来进行

总线会同步试图并发使用总线的事务

### volatile的内存语义

对volatile变量的单个读/写，看成是使用同一个锁对这些单个读/写操作做了同步
volatile++不具有原子性

volatile变量的特性
- 可见性
- 原子性

volatile的写-读可以实现线程间的通信

**volatile写的内存语义**

当写一个volatile变量时，JMM会把该线程对应的本地内存中对应的共享变量值刷新到主内存

**volatile读的内存语义**

当读一个volatile变量时，JMM会把该线程对应的本地内存置位无效。线程接下来将从主内存中读取共享变量

**volatile内存语义的实现**

JMM会分别限制编译器重排序和处理器重排序

针对编译器制定的重排序规则：
- 当第二个操作是volatile写，不管第一个操作是什么，都不能重排序。确保volatile写之前的操作不会被重排序到volatile写之后
- 当第一个操作是volatile读，不管第二个操作是什么，都不能重排序。确保volatile读之后的操作不会被重排序到volatile读之前
- 当第一个操作是volatile写，第二个操作是volatile读时，不能重排序

生成字节码时，编译器会插入内存屏障来禁止特定类型的处理器重排序：
- 在每个volatile写操作的前面插入一个StoreStore屏障
- 在每个volatile写操作的后面插入一个StoreLoad屏障
- 在每个volatile读操作的后面插入一个LoadLoad屏障
- 在每个volatile读操作的后面插入一个LoadStore屏障

编译器可以根据具体情况省略不必要的屏障

JSR-133增强了volatile的内存语义
旧的Java内存模型允许volatile变量与普通变量重排序，volatile的写-读没有锁的释放-获取的内存语义

JSR-133严格限制了volatile变量与普通变量的重排序，确保volatile的写-读和锁的释放-获取具有相同的内存语义

### 锁的内存语义

**锁的释放和获取的内存语义**

线程释放锁时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存中

线程获取锁时，JMM会把该线程对应的本地内存置为无效

**ReentrantLock**

利用volatile变量

公平锁：
加锁方法首先读volatile变量state；
解锁方法在释放锁的最后写volatile变量state

非公平锁：
释放：和公平锁相同
获取：以原子操作的方式更新state变量。CAS，具有volatile读和写的内存语义


### concurrent包的实现

**Java线程间的通信方式**

- A线程写volatile变量，B线程读
- A线程写volatile变量，B线程用CAS更新
- A线程用CAS更新volatile变量，B线程读
- A线程用CAS更新volatile变量，B线程用CAS更新

通用化实现模式：
1. 声明共享变量为volatile
2. 使用CAS的原子条件更新来实现线程之间的同步
3. 同时配合以volatile的读/写和CAS所具有的的volatile读写的内存语义来实现线程之间的通信


### final域的内存语义

**重排序规则**

1. 在构造函数内对一个final域的写入，与虽有把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序
2. 初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序

**写final域的重排序规则**

禁止把final域的写 重排序到构造函数之外
1. JMM禁止编译器把final域的写 重排序到构造函数之外
2. 编译器会在final域的写之后，构造函数的return之前，插入一个StoreStore屏障

确保在对象引用被任意线程可见之前，对象的final域已经被正确的初始化过了（普通域不具有这个保障）

**读final域的重排序规则**

在一个线程中，初次读对象引用与初次读该对象包含的final域，JMM禁止处理器重排序这两个操作（仅针对处理器）
编译器会在读final域操作的前面插入一个LoadLoad屏障

**当final域为引用类型**

在构造函数内对一个final引用的对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序

**溢出**

在构造函数内部，不能让这个被构造对象的引用被其他线程所见（“逸出”）

**final语义在处理器中的实现**

背景-内存语义：写final域的重排序规则会要求编译器在final域的写之后，构造函数return之前插入StoreStore屏障
读final域的重排序规则要求编译器在读final域前插入一个LoadLoad屏障

x86处理器：final域的读/写不会插入任何内存屏障
由于不会对写-写操作做重排序，写final域所需的StoreStore会省略掉
由于不会对存在间接依赖关系的操作做重排序，读final域需要的LoadLoad也会省略掉


### happens-before

**定义**

1. 如果一个操作happens-before另一个操作，那么第一个操作的执行结果对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前
2. 两个操作存在happens-before关系，并不意味着Java的具体实现必须按照happens-before关系指定的顺序执行。如果重排序之后的执行结果，与按happens-before关系来执行的结果一致，那么这种重排序并不非法

as-if-serial语义保证单线程内程序的执行结果不被改变，happens-before关系保证正确同步的多线程程序的执行结果不被改变
程序顺序规则可以看成是对as-if-serial的封装

JMM将happens-before要求禁止的重排序分为了两类
1. 会改变程序结果的重排序 -- 必须禁止
2. 不会改变程序结果的重排序 -- 不做要求

**happens-before规则**

1. 程序顺序规则：一个线程中的每个操作，happens-before于该线程的任一后续操作
2. 监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁
3. volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读
4. 传递性
5. start()规则：如果线程A执行操作ThreadB.start()，那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作
6. join()规则：如果线程A执行操作ThreadB.join()并返回成功，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回


### 双重检查锁定与延迟初始化

双重检查锁定的问题：第一次判断对象是否为null时，instance引用的对象可能还没有完成初始化

1. 分配对象的内存空间
2. 初始化对象
3. 设置instance指向内存空间

其中2和3可能重排序

**解决方案**

思路1：禁止2和3重排序
思路2：允许2和3重排序，但是禁止其他线程看到这个重排序

**基于volatile的解决方案**

将instance声明为volatile型
禁止2和3的重排序

**基于类初始化的解决方案**
> Initialization On Demand Holder idiom

instance holder
JVM在类的初始化阶段（即在Class被加载后，且被线程使用前），会执行线程的初始化。在执行类的期间，JVM会去获取一个锁。这个锁可以同步多个线程对同一个类的初始化
允许2和3重排序，但是禁止其他线程看到这个重排序


在首次发生下列任意一种情况时，一个类或接口类型T将被立即初始化
1. T是一个类，而且一个T类型的实例被创建
2. T是一个类，且T中声明的一个静态方法被调用
3. T中声明的一个静态字段被赋值
4. T中声明的一个静态字段被使用，而且这个字段不是一个常量字段
5. T是一个顶级类，而且一个断言语句嵌套在T内部被执行


对于每一个类或接口C，都有一个唯一的初始化锁LC与之对应

类初始化的处理过程：
1. 通过在Class对象上同步（即获取Class对象的初始化锁），来控制类或接口的初始化。这个获取锁的线程会一直等待，直到当前线程能够获取到这个初始化锁
2. 获取到锁的线程（A）执行类的初始化，同时其他线程（B）在初始化锁对应的condition上等待
3. 线程A设置state=initialized, 然后唤醒在condition中等待的所有线程
4. 线程B结束类的初始化处理



按程序类型，Java程序的内存可见性保证可以分为3类
- 单线程程序，不会出现内存可见性问题
- 正确同步的多线程程序。具有顺序一致性（程序的执行结果与该程序在顺序一致性内存模型中的执行结果相同）
- 未同步/未正确同步的多线程程序。（JMM提供最小安全性保障：线程执行时读取到的值要么是之前某个线程写入的值，要么是默认值）



## Java并发编程基础知识

### 线程

现代操作系统调度的最小单元

**为什么使用多线程**

1. 更多的处理器核心
2. 更快的响应时间（比如将一部分操作异步处理）
3. 更好的编程模型


现代操作系统基本采用时分的形式调度运行的线程。线程的优先级决定线程需要或者少分配一些处理器资源的线程属性

程序正确性不能依赖线程的优先级，有些操作系统会忽略线程优先级的设定


**线程的状态**

- NEW
- RUNNABLE
- BLOCKED
- WAITING
- TIME_WAITING
- TERMINATED

**Daemon线程**

支持型线程，用作程序中后台调度以及支持性工作
一个JVM中不存在非Daemon线程的时候，JVM将会退出

JVM退出时Daemon线程中的finally块并不一定会执行

**线程中断**

中断可以理解为线程的一个标识位属性，表示一个运行中的线程是否被其他线程进行了中断操作
线程通过检查自身是否被中断来进行相应
在线程抛出InterruptedException之前JVM会先将该线程的中断表示位清除，此时调用isInterrupted()将会返回false

**过期的 suspend(), resume(), stop()**

不建议使用
会带来副作用，不保证对资源的正确释放，可能导致程序工作在不确定状态下

**安全的终止线程**

- 使用thread.interrupt()
- 使用volatile boolean变量进行控制

**线程间的通信**

**volatile和synchronized**

synchronized
同步块的实现使用了monitorenter和monitorexit指令
同步方法依靠方法修饰符上的ACC_SYNCHRONIZED来完成的
本质上是对一个对象的monitor进行获取，如果获取失败，线程进入同步队列，线程状态变为BLOCKED。当获得了锁的线程释放了锁，则该释放操作唤醒阻塞在同步队列中的线程，使其重新尝试对监视器的获取

**等待/通知机制**

线程A调用了对象O的wait()方法进入等待状态，而另一个线程B调用了对象O的notify()或者notifyAll()方法，线程A收到通知后从对象O的wait()方法返回，进而执行后续操作

调用wait()方法后，会释放对象的锁
从wait()方法返回的前提是该线程获取到了对象的锁

- 使用wait(), notify(), notifyAll()时需要先对调用对象加锁
- 调用wait()方法后，线程状态由RUNNING变为WAITING，并将当前线程放置到对象的等待队列
- notify()或notifyAll()方法调用后，等待线程依旧不会从wait()返回，需要调用notify()或notifyAll()的线程释放锁之后，等待线程才有机会从wait()返回
- notify()方法将等待队列中的一个等待线程从等待队列中移到同步队列中，notifyAll方法将等待队列中所有的线程全部移到同步队列，被移动的线程状态从WAITTING变为BLOCKED
- 从wait()方法返回的前提是获得了调用对象的锁

**等待/通知的经典范式**

等待方遵循如下原则：
1. 获取对象的锁
2. 如果条件不满足，调用对象的wait()方法，被通知后仍要检查条件
3. 条件满足则执行对应的逻辑

```
synchronized(对象) {
    while(条件不满足) {
        对象.wait();
    }
    对应的处理逻辑;
}
```

通知方遵循如下原则：
1. 获取对象的锁
2. 改变条件
3. 通知所有等待在对象上的线程

```
synchronized(对象) {
    改变条件;
    对象.notifyAll();
}
```

**管道输入/输出流**

主要用于线程间的数据传输，传输的媒介为内存
具体实现：PipedOutputStream, PipedInputStream, PipedReader, PipedWriter. 前两种面向字节，后两种面向字符

对于Piped类型的流，必须要先调用connect方法进行绑定

**Thread.join()**

当前线程A等待thread线程终止之后才从thread.join()返回
或者thread.join(long millis)

join()方法：
```
// 加锁当前线程对象
public final synchronized void join() throws InterruptedException {
    // 条件不满足，继续等待
    while(isAlive()) {
        wait(0);
    }
    // 条件符合，方法返回
}
```

当线程终止时，会调用线程自身的notifyAll方法，通知所有等待在该线程对象上的线程

和等待/通知经典范式一致

**ThreadLocal的使用**

线程变量，以ThreadLocal对象为键，任意对象为值的存储结构
被附带在线程上，一个线程可以根据一个ThreadLocal对象查询到绑定在这个线程上的一个值

通过set(T)方法来设置一个值，在当前线程下再通过get()方法获取到原先设置的值


### 线程应用实例

**等待超时模式**

在等待/通知范式基础上增加了超时控制，使得该模式相比原有范式更具灵活性

```
public synchronized Object get(long millis) throws InterruptedException {
    long future = System.currentTimeMillis() + millis;
    long remaining = millis;
    // 当超时大于0并且result返回值不满足要求
    while ((result == null) && remaining > 0) {
        wait(remaining);
        remaining = future - System.currentTimeMillis();
    }
    return result;
}
```

**线程池技术**

- 消除了频繁创建和消亡线程的系统资源开销
- 面对过量任务的提交能够平缓的劣化

本质上使用了一个线程安全的工作队列连接工作者线程和客户端线程，客户端线程将任务放入工作者队列后便返回，工作者线程不断地从工作队列上取出工作并执行。
当工作队列为空时，所有的工作者线程均等待在工作队列上，当有客户端提交了一个任务后便会通知任意一个工作者线程

当客户端调用execute(Job)方法时，会不断地向任务列表jobs中添加Job，而每个工作者线程会不断地从jobs上取出一个Job进行执行，当jobs为空时，工作者线程进入等待状态

添加一个Job后，对工作队列jobs调用了其notify()方法（不是notifyAll，为了获得更小的开销）


### Java中的锁

#### Lock接口

显示的获取和释放锁

特性
- 尝试非阻塞的获取锁
- 能被中断的获取锁：获取到锁的线程能够响应中断，当获取到锁的线程被中断时，中断异常将会抛出，同时锁会被释放
- 超时获取锁

基本操作
- lock()
- lockInteruptibly()
- tryLock()
- tryLock(long time, TimeUnit unit) throws InterruptedException
- unlock()
- newCondition()

#### 队列同步器 AbstractQueuedSynchronizer

**AQS的接口**

构建锁或者其他同步组件的基础框架
使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作

定义了若干同步状态获取和释放的方法来供自定义同步组件使用
自定义同步组件使用同步器提供的模板方法来实现自己定义的同步语义

面向的是锁的实现者，简化了锁的实现方式

使用getState(), setState(int newState), compareAndSetState(int expect, int update)来访问或者修改同步状态

同步器可重写的方法：
- tryAcquire(int arg): 独占式获取同步状态
- tryRelease(int arg)：独占式释放同步状态
- tryAcquireShared(int arg): 共享式获取同步状态，同一时刻可以有多个线程获取到同步状态
- tryReleaseShared(int arg): 共享式释放同步状态
- isHeldExclusively(): 当前同步器是否在独占模式下被线程占用，一般表示是否被当前线程所占

同步器提供的模板方法：
基本上分为三类：独占式获取与释放同步状态，共享式获取与释放同步状态，查询同步队列中的等待线程情况


独占锁：在同一个时刻只能有一个线程获取到锁，而其他获取锁的线程只能处于同步队列中等待，只有获取锁的线程释放了锁，后继的线程才能够获取锁

**AQS的实现**

同步队列

同步器依赖内部的同步队列（一个FIFO双向队列）来完成同步状态的管理
- 当前线程获取同步状态失败时，同步器会将当前线程已经等待状态等信息构造成为一个节点（Node）并将其加入同步队列，同时会阻塞当前线程
- 当同步状态释放时，会把首节点中的线程唤醒，使其再次尝试获取同步状态

加入同步队列的过程需要保证钱程安全
同步器提供了一个基于CAS的设置尾节点的方法 compareAndSetTail(Node expect, Node update)


独占式同步状态获取与释放

首先调用自定义同步器实现的tryAcquire方法，如果同步状态获取失败，则构造同步节点（独占式Node.EXECUSIVE)，并通过addWaiter方法将该节点加入到同步队列的尾部，最后调用acquireQueued(Node node, int arg)方法，使得该节点以”死循环“的方式获取同步状态。如果获取不到则阻塞节点中的线程，而被阻塞线程的环境主要依靠前驱节点的出队或阻塞线程被中断来实现

独占式同步状态获取流程：
{% asset_img t5-5-i.jpg %}

通过调用同步器的release(int arg)方法可以释放同步状态，释放同步状态之后会唤醒其后继节点


共享式同步状态获取与释放

eg：文件的读操作，可以共享式访问

共享式访问资源时，其他共享式的访问均被允许，独占式访问被阻塞
独占式访问资源时，同一时刻其他访问均被阻塞

在acquireShared(int arg)方法中，同步器调用tryAcquireShared(int arg)方法尝试获取同步状态
在共享式获取的自旋过程中，成功获取到同步状态并退出自旋的条件就是tryAcquireShared(int arg)方法返回值大于等于0。如果当前节点的前驱为头节点时，尝试获取同步状态

通过调用releaseShared(int arg)释放同步状态
释放同步状态后唤醒后续处于等待状态的节点