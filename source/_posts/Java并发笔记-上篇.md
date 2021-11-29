---
layout: shoka
title: Java并发笔记-上篇
date: 2021-11-16 08:56:30
tags:
---

# Java并发笔记

## 可重入锁

### 可重入锁是什么？

顾名思义，就是支持重入的锁，它表示该锁能够支持一个线程对资源的重复加锁。除此之外，该锁还支持获取锁的公平与非公平性选择。

### 可重入锁的代码示例-synchronized

```java
/** 证明synchronized是可重入锁 */
public class SynchronizedDemo {
  static Object objectLockA = new Object();

  static void m1() {
    synchronized (objectLockA) {
      System.out.println(Thread.currentThread().getName() + "\t-----外层调用");
      synchronized (objectLockA) {
        System.out.println(Thread.currentThread().getName() + "\t-----中层调用");
        synchronized (objectLockA) {
          System.out.println(Thread.currentThread().getName() + "\t-----内层调用");
        }
      }
    }
  }

  public static void main(String[] args) {
    m1();
  }
```

### 可重入锁的代码示例-ReenterantLock

```java
/** ReenterantLock也是可重入锁，加锁几次就要释放锁几次 */
public class ReenterantLockDemo {
  public static Lock lockA = new ReentrantLock();

  public static void main(String[] args) {
    new Thread(
            () -> {
              lockA.lock();
              try {
                System.out.println(Thread.currentThread().getName() + "\t ----外层");
                lockA.lock();
                try {
                  System.out.println(Thread.currentThread().getName() + "\t ----内层");
                } finally {
                  lockA.unlock();
                }
              } finally {
                lockA.unlock();
              }
            },
            "t1")
        .start();
    new Thread(
            () -> {
              lockA.lock();
              try {
                System.out.println(Thread.currentThread().getName() + "\t ----第二个线程获取锁");
              } finally {
                lockA.unlock();
              }
            },
            "t2")
        .start();
  }
```

### 公平与非公平获取锁的区别是什么？

如果一个锁是公平的，那么锁的获取顺序就应该符合请求的绝对时间顺序，也就是**FIFO**。

对于非公平的锁，只要CAS设置同步状态成功，则表示当前线程获取了锁。

#### 产生结果

公平的锁每次都是同步队列中的第一个节点获取到锁，非公平的锁会出现一个线程连续获取锁的现象。

为什么会出现线程连续获取锁的情况呢？当一个线程请求锁时，只要获取了同步状态即成功获取锁。在这个前提下，刚释放锁的线程再次获取同步状态的概率非常大，使得其他线程只能在同步队列中等待。

非公平的锁可能导致 **线程饥饿** ，为什么它又被设置成默认的实现呢？如果把每次不同的线程获取到锁视为一次切换，公平性的锁每次都进行切换，非公平的锁出现切换的概率大大降低，这说明非公平锁的开销非常小。引用自 **Java并发编程艺术** 一书的表5-7的例子显示，在ubuntu server 14.04 i5-3470 8GB，10个线程，每个线程获取100000次锁的场景中，公平锁与非公平锁相比，总耗时是其93.4倍，总切换次数是其133倍。这可以说明：公平性的锁保证了锁的获取按照FIFO原则，而代价是进行大量的线程切换。非公平性锁虽然可能造成 **线程饥饿**，但极少的线程切换，保证了更大的吞吐量。

## LockSupport

### LockSupport 是什么？

[官方中文文档](https://www.apiref.com/java11-zh/java.base/java/util/concurrent/locks/LockSupport.html)

#### api介绍

用于创建锁和其他同步类的基本线程阻塞原语。

**该类与使用它的每个线程关联一个Permit（许可证）（在[`Semaphore`](https://www.apiref.com/java11-zh/java.base/java/util/concurrent/Semaphore.html)类的意义上）。** 如果许可证可用，将立即返回`park` ，并在此过程中消费; 否则*可能会*阻止。 如果尚未提供许可，则致电`unpark`获得许可。 **（与Semaphores不同，许可证不会累积。最多只有一个。）**可靠的使用需要使用volatile（或原子）变量来控制何时停放或取消停放。 对于易失性变量访问保持对这些方法的调用的顺序，但不一定是非易失性变量访问。

方法`park`和`unpark`提供了阻止和解除阻塞线程的有效方法，这些线程没有遇到导致不推荐使用的方法`Thread.suspend`和`Thread.resume`无法用于此类目的的问题：一个线程调用`park`和另一个线程尝试`unpark`将保留活跃性，由于许可证。 此外，如果调用者的线程被中断，则会返回`park` ，并且支持超时版本。 `park`方法也可以在任何其他时间返回，“无理由”，因此通常必须在返回时重新检查条件的循环内调用。 在这个意义上， `park`可以作为“忙碌等待”的优化，不会浪费太多时间旋转，但必须与`unpark`配对才能生效。

三种形式的`park`每个也支持`blocker`对象参数。 在线程被阻塞时记录此对象，以允许监视和诊断工具识别线程被阻止的原因。 （此类工具可以使用方法[`getBlocker(Thread)`](https://www.apiref.com/java11-zh/java.base/java/util/concurrent/locks/LockSupport.html#getBlocker(java.lang.Thread))访问[阻止程序](https://www.apiref.com/java11-zh/java.base/java/util/concurrent/locks/LockSupport.html#getBlocker(java.lang.Thread)) 。）强烈建议使用这些表单而不是没有此参数的原始表单。 在锁实现中作为`blocker`提供的正常参数是`this` 。

这些方法旨在用作创建更高级别同步实用程序的工具，并且对于大多数并发控制应用程序本身并不有用。 `park`方法仅用于以下形式的构造：

```
   while (!canProceed()) { // ensure request to unpark is visible to other threads ... LockSupport.park(this); } 
```

在调用`park`之前，线程没有发布请求`park`需要锁定或阻塞。 因为每个线程只有一个许可证，所以任何中间使用`park` ，包括隐式地通过类加载，都可能导致无响应的线程（“丢失unpark”）。

#### 总结

LockSupport中的`park()` 和`unpark()`的作用分别是阻塞线程和解除阻塞线程，可以将其看作是线程等待唤醒机制 (wait/notify) 的改进版。

LockSupport使用Permit（许可证）的概念来达到阻塞和唤醒线程的功能，每个线程都有一个许可(Permit)，Permit只有两个值1和0，默认是0。

Permit可以看做成一个Semaphore，但与Semaphore不同，Permit累加上限只有1。

### 三种实现线程等待唤醒的方法

1. 使用Object中的`wait()`方法让线程等待，使用Object中的`notify()`方法唤醒线程
   
   注意条件
   
   1. wait和notify方法必须要在synchronized同步块或者方法里面且成对出现使用
   2. 先wait后notify才能成功唤醒线程
   
   ```java
     static Object objectLock = new Object();
     private static void synchronizedWaitNotify() {
       new Thread(
               () -> {
                 synchronized (objectLock) {
                   System.out.println(Thread.currentThread().getName() + "\t" + "------come in");
                   try {
                     objectLock.wait(); // 等待
                   } catch (InterruptedException e) {
                     e.printStackTrace();
                   }
                   System.out.println(Thread.currentThread().getName() + "\t" + "------被唤醒");
                 }
               },
               "A")
           .start();
   
       new Thread(
               () -> {
                 synchronized (objectLock) {
                   objectLock.notify(); // 唤醒
                   System.out.println(Thread.currentThread().getName() + "\t" + "------通知");
                 }
               },
               "B")
           .start();
     }
   ```
   
   ![img](locksupport代码结果1.png)

2. 使用JUC包中的Condition的`await()`方法让线程等待，使用Condition的`signal()`方法唤醒线程
   
   注意条件
   
   1. 线程先要获得并持有锁，必须在锁块（synchronized或lock）中
   2. 必须要先等待后唤醒，线程才能够被唤醒
   
   ```java
     static Lock lock = new ReentrantLock();
     static Condition condition = lock.newCondition();
     private static void lockAwaitSignal() {
       new Thread(
               () -> {
                 lock.lock();
                 try {
                   System.out.println(Thread.currentThread().getName() + "\t" + "------come in");
                   try {
                     condition.await();
                   } catch (InterruptedException e) {
                     e.printStackTrace();
                   }
                   System.out.println(Thread.currentThread().getName() + "\t" + "------被唤醒");
                 } finally {
                   lock.unlock();
                 }
               },
               "A")
           .start();
   
       new Thread(
               () -> {
                 lock.lock();
                 try {
                   condition.signal();
                   System.out.println(Thread.currentThread().getName() + "\t" + "------通知");
                 } finally {
                   lock.unlock();
                 }
               },
               "B")
           .start();
     }
   ```
   
   ![img](locksupport代码结果2.png)

3. 使用LockSupport类阻塞当前线程以及唤醒指定的被阻塞线程

### LockSupport 的主要方法

#### 阻塞

`park()/park(Object blocker)`

作用：阻塞当前线程/阻塞传入的具体线程

permit 默认是 0，所以一开始调用`park()`方法，当前线程就会阻塞，直到别的线程将当前线程的 permit 设置为 1 时，`park() `方法会被唤醒，然后会将 permit 再次设置为 0 并返回。

##### park() 方法通过 Unsafe 类实现

```java
// Disables the current thread for thread scheduling purposes unless the permit is available.
public static void park() {
    UNSAFE.park(false, 0L);
}
```

#### 唤醒

`unpark(Thread thread)`

作用：唤醒处于阻断状态的指定线程

调用`unpark(thread)`方法后，就会将 thread 线程的许可 permit 设置成 1（注意多次调用 `unpark()`方法，不会累加，permit 值还是 1），这会自动唤醒 thread 线程，即之前阻塞中的`LockSupport.park()`方法会立即返回。

##### unpark() 方法通过 Unsafe 类实现

```java
// Makes available the permit for the given thread
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```

### LockSupport代码示例

#### 正常使用 LockSupport

```java
private static void lockSupportParkUnpark() {
    Thread a =
        new Thread(
            () -> {
              System.out.println(Thread.currentThread().getName() + "\t" + "------come in");
              LockSupport.park(); // 线程 A 阻塞
              System.out.println(Thread.currentThread().getName() + "\t" + "------被唤醒");
            },
            "A");
    a.start();

    new Thread(
            () -> {
              LockSupport.unpark(a); // B 线程唤醒线程 A
              System.out.println(Thread.currentThread().getName() + "\t" + "------通知");
            },
            "B")
        .start();
  }
```

程序运行结果：A 线程先执行`LockSupport.park()`方法将通行证（permit）设置为 0， permit 初始值本来就为 0，然后 B 线程执行 `LockSupport.unpark(a) `方法将 permit 设置为 1，此时 A 线程可以通行

![img](locksupport代码结果3.png)

#### 先 unpark() 后 park()

```java
  private static void lockSupportUnparkPark() {
    Thread a =
        new Thread(
            () -> {
              try {
                TimeUnit.SECONDS.sleep(3L);
              } catch (InterruptedException e) {
                e.printStackTrace();
              }
              System.out.println(
                  Thread.currentThread().getName()
                      + "\t"
                      + "------come in"
                      + System.currentTimeMillis());
              LockSupport.park();
              System.out.println(
                  Thread.currentThread().getName()
                      + "\t"
                      + "------被唤醒"
                      + System.currentTimeMillis());
            },
            "A");
    a.start();

    new Thread(
            () -> {
              LockSupport.unpark(a);
              System.out.println(Thread.currentThread().getName() + "\t" + "------通知");
            },
            "B")
        .start();
  }
```

程序运行结果：因为引入了通行证的概念，所以先执行`unpark()`其实并不会有什么影响，从程序运行时间戳可以看出，B 线程先放行通行证之后，A 线程执行 `LockSupport.park()` 时并没有被阻塞，相当于没有执行。

![img](locksupport代码结果4.png)

#### 异常情况：没有考虑到 permit 上限值为 1

```java
  private static void lockSupport2Park1Unpark() {
    Thread a =
        new Thread(
            () -> {
              try {
                TimeUnit.SECONDS.sleep(3L);
              } catch (InterruptedException e) {
                e.printStackTrace();
              }
              System.out.println(
                  Thread.currentThread().getName()
                      + "\t"
                      + "------come in"
                      + System.currentTimeMillis());
              LockSupport.park();
              LockSupport.park();
              System.out.println(
                  Thread.currentThread().getName()
                      + "\t"
                      + "------被唤醒"
                      + System.currentTimeMillis());
            },
            "A");
    a.start();

    new Thread(
            () -> {
              LockSupport.unpark(a);
              LockSupport.unpark(a);
              System.out.println(Thread.currentThread().getName() + "\t" + "------通知");
            },
            "B")
        .start();
  }
```

程序运行结果：由于 permit 的上限值为 1，所以执行两次 `LockSupport.park()` 操作将导致 A 线程阻塞

![img](locksupport代码结果5.png)

### LockSupport 重点说明

> **LockSupport是用来创建锁和其他同步类的基本线程阻塞原语**

LockSupport是一个线程阻塞工具类，所有的方法都是静态方法，可以让线程在任意位置阻塞，阻塞之后也有对应的唤醒方法。归根结底，LockSupport调用的Unsafe中的native代码。

> **LockSupport提供park()和unpark()方法实现阻塞线程和解除线程阻塞的过程**

LockSupport和每个使用它的线程都有一个许可(permit)关联。permit相当于1，0的开关，默认是0，调用一次unpark就加1变成1，调用一次park会消费permit，也就是将1变成0，同时park立即返回。

如再次调用park会变成阻塞(因为permit为零了会阻塞在这里，一直到permit变为1)，这时调用unpark会把permit置为1。

每个线程都有一个相关的permit，permit最多只有一个，重复调用unpark也不会积累凭证。

> **形象的理解**

线程阻塞需要消耗凭证(permit)，这个凭证最多只有1个。

1. 当调用park方法时
   1. 如果有凭证，则会直接消耗掉这个凭证然后正常退出;
   2. 如果无凭证，就必须阻塞等待凭证可用;
2. 而unpark则相反，它会增加一个凭证，但凭证最多只能有1个，累加无效。

### LockSupport 面试题

> **为什么可以先唤醒线程后阻塞线程?**

因为unpark获得了一个凭证，之后再调用park方法，就可以名正言顺的凭证消费，故不会阻塞。

> **为什么唤醒两次后阻塞两次，但最终结果还会阻塞线程?**

因为凭证的数量最多为1，连续调用两次unpark和调用一次unpark效果一样，只会增加一个凭证；而调用两次park却需要消费两个凭证，证不够，不能放行。

## AQS初探

### AQS 是什么？

AQS（AbstractQueuedSynchronizer）：抽象的队列同步器

一般我们说的 AQS 指的是` java.util.concurrent.locks`包下的`AbstractQueuedSynchronizer`，但其实还有另外三种抽象队列同步器：`AbstractOwnableSynchronizer`、`AbstractQueuedLongSynchronizer `和 `AbstractQueuedSynchronizer`。

### AQS 是 JUC 的基石

![](aqs-1.png)

**举几个常见的例子**

1. ReentrantLock
   
   ![](aqs-2.png)

2. CountDownLatch
   
   ![](aqs-3.png)

3. ReentrantReadWriteLock
   
   ![](aqs-4.png)

4. Semaphore
   
   ![](aqs-5.png)

> **进一步理解锁和同步器的关系**

锁 -> 面向锁的使用者。定义了程序员和锁交互的使用层API，隐藏了实现细节，你调用即可，可以理解为用户层面的 API。

同步器 -> 面向锁的实现者。比如Java并发大神Douglee，提出统一规范并简化了锁的实现，屏蔽了同步状态管理、阻塞线程排队和通知、唤醒机制等，Java 中有那么多的锁，就能简化锁的实现啦。

### AQS 能干嘛

> **AQS：加锁会导致阻塞**

有阻塞就需要排队，实现排队必然需要有某种形式的队列来进行管理。

抢到资源的线程直接使用办理业务，抢占不到资源的线程的必然涉及一种排队等候机制，抢占资源失败的线程继续去等待（类似办理窗口都满了，暂时没有受理窗口的顾客只能去候客区排队等候），仍然保留获取锁的可能且获取锁流程仍在继续（候客区的顾客也在等着叫号，轮到了再去受理窗口办理业务）。

既然说到了排队等候机制，那么就一定 会有某种队列形成，这样的队列是什么数据结构呢？如果共享资源被占用，就需要一定的阻塞等待唤醒机制来保证锁分配。这个机制主要用的是CLH队列的变体实现的，将暂时获取不到锁的线程加入到队列中，这个队列就是AQS的抽象表现。它将请求共享资源的线程封装成队列的结点（Node） ，通过CAS、自旋以及LockSuport.park()的方式，维护state变量的状态，使并发达到同步的效果。

### **AQS术语理解**

![](aqs-6.png)

有阻塞就需要排队，实现排队必然需要队列

AQS使用一个volatile的int类型的成员变量来表示同步状态，通过内置的 FIFO队列来完成资源获取的排队工作将每条要去抢占资源的线程封装成 一个Node节点来实现锁的分配，通过CAS完成对State值的修改。
Node 节点是啥？答：你有见过 HashMap 的 Node 节点吗？JDK 用 static class Node<K,V> implements Map.Entry<K,V> { 来封装我们传入的 KV 键值对。这里也是一样的道理，JDK 使用 Node 来封装（管理）Thread
可以将 Node 和 Thread 类比于候客区的椅子和等待用餐的顾客

![](aqs-7.png)

### **AQS内部体系架构**

1. AQS的int变量
   
   AQS的同步状态State成员变量，类似于银行办理业务的受理窗口状态：零就是没人，自由状态可以办理；大于等于1，有人占用窗口，等着去。
   
   ```java
   /**
    * The synchronization state.
    */
   private volatile int state;
   ```

2. AQS的CLH队列
   
   CLH队列（三个大牛的名字组成），为一个双向队列，类似于银行侯客区的等待顾客。
   
   ![](aqs-8.png)

3. 内部类Node（Node类在AQS类内部）
   
   Node的等待状态waitState成员变量，类似于等候区其它顾客(其它线程)的等待状态，队列中每个排队的个体就是一个Node。
   
   ```java
   /**
    * Status field, taking on only the values:
    *   SIGNAL:     The successor of this node is (or will soon be)
    *               blocked (via park), so the current node must
    *               unpark its successor when it releases or
    *               cancels. To avoid races, acquire methods must
    *               first indicate they need a signal,
    *               then retry the atomic acquire, and then,
    *               on failure, block.
    *   CANCELLED:  This node is cancelled due to timeout or interrupt.
    *               Nodes never leave this state. In particular,
    *               a thread with cancelled node never again blocks.
    *   CONDITION:  This node is currently on a condition queue.
    *               It will not be used as a sync queue node
    *               until transferred, at which time the status
    *               will be set to 0. (Use of this value here has
    *               nothing to do with the other uses of the
    *               field, but simplifies mechanics.)
    *   PROPAGATE:  A releaseShared should be propagated to other
    *               nodes. This is set (for head node only) in
    *               doReleaseShared to ensure propagation
    *               continues, even if other operations have
    *               since intervened.
    *   0:          None of the above
    *
    * The values are arranged numerically to simplify use.
    * Non-negative values mean that a node doesn't need to
    * signal. So, most code doesn't need to check for particular
    * values, just for sign.
    *
    * The field is initialized to 0 for normal sync nodes, and
    * CONDITION for condition nodes.  It is modified using CAS
    * (or when possible, unconditional volatile writes).
    */
   volatile int waitStatus;
   ```
   
   Node类的内部结构
   
   ```java
   static final class Node{
       //共享
       static final Node SHARED = new Node();
   
       //独占
       static final Node EXCLUSIVE = null;
   
       //线程被取消了
       static final int CANCELLED = 1;
   
       //后继线程需要唤醒
       static final int SIGNAL = -1;
   
       //等待condition唤醒
       static final int CONDITION = -2;
   
       //共享式同步状态获取将会无条件地传播下去
       static final int PROPAGATE = -3;
   
       // 初始为e，状态是上面的几种
       volatile int waitStatus;
   
       // 前置节点
       volatile Node prev;
   
       // 后继节点
       volatile Node next;
   
       // ...
   ```

4. 总结
   
   有阻塞就需要排队，实现排队必然需要队列，通过state 变量 + CLH双端 Node 队列实现。

### **AQS同步队列的基本结构**

![](aqs-9.png)

### **AQS底层是怎么排队的？**

通过调用 `LockSupport.park()` 来进行排队。

### 从 ReentrantLock 进入 AQS

#### ReentrantLock 锁

`ReentrantLock` 类是 `Lock` 接口的实现类，基本上是通过【聚合】了一个【队列同步器】的子类完成线程访问控制的。

**ReentrantLock 的原理**

![](rel-1.png)

#### 公平锁 & 非公平锁

在 `ReentrantLock` 内定义了静态内部类，分别为 `NoFairSync`（非公平锁）和 `FairSync`（公平锁）

![](rel-2.png)

`ReentrantLock` 的构造函数：不传参数表示创建非公平锁；参数为 true 表示创建公平锁；参数为 false 表示创建非公平锁。

看一下`lock()` 方法的执行流程：以 `NonfairSync` 为例

![](rel-3.png)

在 `ReentrantLock` 中，`NoFairSync` 和 `FairSync` 中 `tryAcquire()` 方法的区别，可以明显看出公平锁与非公平锁的`lock()`方法唯一的区别就在于公平锁在获取同步状态时多了一个限制条件:  

`hasQueuedPredecessors()`

![](rel-4.png)

`hasQueuedPredecessors()` 方法是公平锁加锁时判断等待队列中是否存在有效节点的方法

![](rel-5.png)

### 公平锁与非公平锁的总结

对比公平锁和非公平锁的`tryAcqure()`方法的实现代码， 其实差别就在于非公平锁获取锁时比公平锁中少了一个判断`!hasQueuedPredecessors()`，`hasQueuedPredecessors()`中判断了是否需要排队，导致公平锁和非公平锁的差异如下:

公平锁：公平锁讲究先来先到，线程在获取锁时，如果这个锁的等待队列中已经有线程在等待，那么当前线程就会进入等待队列中；

非公平锁：不管是否有等待队列，如果可以获取锁，则立刻占有锁对象。也就是说队列的第一 个排队线程在unpark()，之后还是需要竞争锁(存在线程竞争的情况下)

![](rel-6.png)

而 `acquire()` 方法最终都会调用 `tryAcquire()` 方法

![](rel-7.png)

在 `NonfairSync` 和 `FairSync` 中均重写了其父类 `AbstractQueuedSynchronizer` 中的 `tryAcquire()` 方法。

![](rel-8.png)

#### 从非公平锁的 lock() 入手

我们这里举个栗子，假设 A、B、C 三个人都要去银行窗口办理业务，但是银行窗口只有一个个，我们使用 `lock.lock()` 模拟这种情况。

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

/** @ClassName AQSDemo @Description TODO @Author sunq @Date 2021/11/29 11:08 */
public class AQSDemo {
  public static void main(String[] args) {

    ReentrantLock lock = new ReentrantLock();

    // 带入一个银行办理业务的案例来模拟我们的AQS如何进行线程的管理和通知唤醒机制
    // 3个线程模拟3个来银行网点，受理窗口办理业务的顾客
    // A顾客就是第一个顾客，此时受理窗口没有任何人，A可以直接去办理
    new Thread(
            () -> {
              lock.lock();
              try {
                System.out.println("-----A thread come in");
                try {
                  TimeUnit.MINUTES.sleep(20);
                } catch (Exception e) {
                  e.printStackTrace();
                }
              } finally {
                lock.unlock();
              }
            },
            "A")
        .start();

    // 第二个顾客，第二个线程---》由于受理业务的窗口只有一个(只能一个线程持有锁)，此时B只能等待，
    // 进入候客区
    new Thread(
            () -> {
              lock.lock();
              try {
                System.out.println("-----B thread come in");
              } finally {
                lock.unlock();
              }
            },
            "B")
        .start();

    // 第三个顾客，第三个线程---》由于受理业务的窗口只有一个(只能一个线程持有锁)，此时C只能等待，
    // 进入候客区
    new Thread(
            () -> {
              lock.lock();
              try {
                System.out.println("-----C thread come in");
              } finally {
                lock.unlock();
              }
            },
            "C")
        .start();
  }
}
```

> **先来看看线程 A（客户 A）的执行流程**

之前已经讲到过，`new ReentrantLock()`不传参默认是非公平锁，调用`lock.lock()`方法最终都会执行`NonfairSync`重写后的`lock()`方法

第一次执行    lock()    方法

由于第一次执行`lock()`方法，`state`变量的值等于 0，表示 lock 锁没有被占用，此时执行`compareAndSetState(0, 1)`CAS 判断，可得`state == expected == 0`，因此 CAS 成功，将 state 的值修改为 1。

![](rel-9.png)

再来复习下 CAS：通过 Unsafe 提供的 compareAndSwapXxx() 方法保证修改操作的原子性（通过 CPU 原语保证），如果变量的值等于期望值，则修改变量的值为 update，并返回 true；若不等，则返回 false。this 代表当前对象，stateOffset 表示 state 变量在该对象中的偏移量。

![](rel-10.png)

再来看看 `setExclusiveOwnerThread()` 方法做了啥：将拥有 lock 锁的线程修改为线程 A。

![](rel-11.png)

> **再来看看线程 B（客户 B）的执行流程**

第二次执行 lock() 方法

由于第二次执行`lock()`方法，`state`变量的值等于 1，表示 lock 锁没有被占用，此时执行 `compareAndSetState(0, 1)`CAS 判断，可得`state != expected`，因此 CAS 失败，进入`acquire()`方法。

![](rel-9.png)

`acquire()` 方法主要包含如下几个方法，下面我们一个一个来查看。

![](rel-12.png)

#### **`tryAcquire(arg)` 方法的执行流程**

先来看看 `tryAcquire()` 方法，诶，怎么抛了个异常？别着急，仔细一看是 `AbstractQueuedSynchronizer` 抽象队列同步器中定义的方法，既然抛出了异常，就证明父类强制要求子类去实现。

这里以非公平锁 `NonfairSync` 为例，在 `tryAcquire()` 方法中调用了 `nonfairTryAcquire()` 方法，注意，这里传入的参数都是 1。

![](rel-13.png)

`nonfairTryAcquire(acquires)`正常的执行流程：

在`nonfairTryAcquire()`方法中，大多数情况都是如下的执行流程：线程 B 执行`int c = getState()`时，获取到 state 变量的值为 1，表示 lock 锁正在被占用；于是执行`if (c == 0) {`发现条件不成立，接着执行下一个判断条件`else if (current ==getExclusiveOwnerThread()) {`，current 线程为线程 B，而 `getExclusiveOwnerThread()`方法返回正在占用 lock 锁的线程，为线程 A，因此 `tryAcquire()`方法最后会`return false`，表示并没有抢占到 lock 锁。

![](rel-14.png)

**补充**：`getExclusiveOwnerThread()` 方法返回正在占用 lock 锁的线程（排他锁，exclusive）。

![](rel-15.png)

> **nonfairTryAcquire(acquires) 比较特殊的执行流程：**

第一种情况是，走到`int c = getState()`语句时，此时线程 A 恰好执行完成，让出了 lock 锁，那么`state`变量的值为 0，当然发生这种情况的概率很小，那么线程 B 执行 CAS 操作成功后，将占用 lock 锁的线程修改为自己，然后返回 true，表示抢占锁成功。其实这里还有一种情况，需要留到`unlock()`方法才能说清楚

第二种情况为可重入锁的表现，假设 A 线程又再次抢占 lock 锁（当然示例代码里面并没有体现出来），这时`current == getExclusiveOwnerThread()`条件成立，将`state`变量的值加上`acquire`，这种情况下也应该`return true`，表示线程 A 正在占用 lock 锁。因此，`state`变量的值是可以大于 1 的。
![](rel-14.png)

> **继续往下走，执行 `addWaiter(Node.EXCLUSIVE)` 方法**

在 `tryAcquire()` 方法返回 `false` 之后，进行 `!` 操作后为 `true`，那么会继续执行 `addWaiter()` 方法。

![](rel-12.png)

来看看 addWaiter() 方法做了些啥？

之前讲过`Node`节点用于封装用户线程，这里将当前正在执行的线程通过`Node`封装起来（当前线程正是抢占 lock 锁没有抢占到的线程）。

判断`tail`尾指针是否为空，双端队列此时还没有元素呢~肯定为空呀，那么执行 `enq(node)`方法，将封装了线程 B 的`Node`节点入队。

![](rel-16.png)

**enq(node) 方法：构建双端同步队列**

也许看到这里的代码有点蒙，需要有些前置知识，在双端同步队列中，第一个节点为虚节点（也叫哨兵节点），其实并不存储任何信息，只是占位。 真正的第一个有数据的节点，是从第二个节点开始的。

![](rel-17.png)

**第一次执行 for 循环**：现在解释起来就不费劲了，当线程 B 进来时，双端同步队列为空，此时肯定要先构建一个哨兵节点。此时`tail == null`，因此进入`if(t == null) {`的分支，头指针指向哨兵节点，此时队列中只有一个节点，尾节点即是头结点，因此尾指针也指向该哨兵节点。
![](rel-18.png)

**第二次执行 for 循环**：现在该将装着线程 B 的节点放入双端同步队列中，此时`tail`指向了哨兵节点，并不等于 null，因此`if (t == null)`不成立，进入 else 分支。以尾插法的方式，先将`node`（装着线程 B 的节点）的`prev`指向之前的`tail`，再将`node`设置为尾节点（执行`compareAndSetTail(t, node)`），最后将`t.next`指向`node`，最后执行`return t`结束 for 循环。

**补充**：`compareAndSetTail(t, node)` 方法的实现

![](rel-19.png)

**注意**：哨兵节点和 `nodeB` 节点的 `waitStatus` 均为 0，表示在等待队列中。

**`acquireQueued()` 方法的执行**

执行完 `addWaiter()` 方法之后，就该执行 `acquireQueued()` 方法了，这个方法有点东西，我们放到后面再去讲它

![](rel-12.png)

> **最后来看看线程 C（客户 C）的执行流程**

线程 C 和线程 B 的执行流程很类似，都是执行 `acquire()` 中的方法。

![](rel-12.png)

但是在 `addWaiter()` 方法中，执行流程有些区别。此时 `tail != null`，因此在 `addWaiter()` 方法中就已经将 `nodeC` 添加至队尾了。

![](rel-16.png)

执行完 `addWaiter()` 方法后，就已经将 nodeC 挂在了双端同步队列的队尾，不需要再执行 `enq(node)` 方法。

![](rel-20.png)

> **补前面的坑：`acquireQueued()` 方法的执行逻辑**

先来看看看看 `acquireQueued()` 方法的源代码，其实这样直接看代码有点懵逼，我们接下来举例来理解。注意看：两个 `if` 判断中的代码都放在 `for( ; ; )` 中执行，这样可以实现自旋的操作。

![](rel-21.png)

**线程 B 的执行流程**

线程 B 执行 addWaiter() 方法之后，就进入了 acquireQueued() 方法中，此时传入的参数为封装了线程 B 的 nodeB 节点，nodeB 的前驱结点为哨兵节点，因此 final Node p = node.predecessor() 执行完后，p 将指向哨兵节点。哨兵节点满足 p == head，但是线程 B 执行 tryAcquire(arg) 方法尝试抢占 lock 锁时还是会失败，因此会执行下面 if 判断中的 shouldParkAfterFailedAcquire(p, node) 方法，该方法的代码如下：

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

哨兵节点的 `waitStatus == 0`，因此执行 CAS 操作将哨兵节点的 `waitStatus` 改为 `Node.SIGNAL(-1)`。

![](rel-22.png)

注意：`compareAndSetWaitStatus(pred, ws, Node.SIGNAL)`调用 `unsafe.compareAndSwapInt(node, waitStatusOffset, expect, update);`实现，虽然`compareAndSwapInt()`方法内无自旋，但是在`acquireQueued()`方法中的`for( ; ; )`能保证此自选操作成功（另一种情况就是线程 B 抢占到 lock 锁）。

![](rel-23.png)

执行完上述操作，将哨兵节点的 `waitStatus` 设置为了 -1。

![](rel-24.png)

执行完毕将退出 `if` 判断，又会重新进入 `for( ; ; )` 循环，此时执行 `shouldParkAfterFailedAcquire(p, node)` 方法时会返回 `true`，因此此时会接着执行 `parkAndCheckInterrupt()` 方法。

![](rel-21.png)

线程 B 调用 `park()` 方法后被挂起，程序不会然续向下执行，程序就在这儿排队等待。

![](rel-25.png)

**线程 C 的执行流程**

线程 C 最终也会执行到 `LockSupport.park(this);` 处，然后被挂起，进入等待区

**总结：**

如果前驱节点的`waitstatus`是`SIGNAL 状态（-1）`，即 `shouldParkAfterFailedAcquire() `方法会返回 `true`，程序会继续向下执行 `parkAndCheckInterrupt()`方法，用于将当前线程挂起。

根据`park()`方法 API 描述，程序在下面三种情况会继续向下执行：

1. 被`unpark`

2. 被中断（interrupt）

3. 其他不合逻辑的返回才会然续向下执行
   
   因上述三种情况程序执行至此，返回当前线程的中断状态，并清空中断状态。如果程序由于被中断，该方法会返回 true。

#### 可总算要 unlock() 了

> **线程 A 执行 `unlock()` 方法**

A 线程终于要 `unlock()` 了吗？真不容易啊！

![](rel-26.png)

`unlock()` 方法调用了 `sync.release(1)` 方法。

![](rel-27.png)

**release() 方法的执行流程**

其实主要就是看看`tryRelease(arg)`方法和`unparkSuccessor(h)`方法的执行流程，这里先大概说一下，能有个印象：线程 A 即将让出 lock 锁，因此`tryRelease()`执行后将返回 true，表示礼让成功，head 指针指向哨兵节点，并且 if 条件满足，可执行 `unparkSuccessor(h)`方法。

![](rel-28.png)

**`tryRelease(arg)` 方法的执行逻辑**

线程 A 只加锁过一次，因此 `state` 的值为 1，参数 `release` 的值也为 1，因此 `c == 0`。将 `free` 设置为 `true`，表示当前 lock 锁已被释放，将排他锁占有的线程设置为 `null`，表示没有任何线程占用 lock 锁。

![](rel-29.png)

**unparkSuccessor(h) 方法的执行逻辑**

在 release() 方法中获取到的头结点 h 为哨兵节点，h.waitStatus == -1，因此执行 CAS操作将哨兵节点的 waitStatus 设置为 0，并将哨兵节点的下一个节点（s = node.next = nodeB）获取出来，并唤醒 nodeB 中封装的线程（if (s == null || s.waitStatus > 0) 不成立，只有 if (s != null) 成立）。

![](rel-30.png)

执行完上述操作后，当前占用 lock 锁的线程为 `null`，哨兵节点的 `waitStatus` 设置为 0，`state` 的值为 0（表示当前没有任何线程占用 lock 锁）。

![](rel-31.png)

**杀个回马枪：继续来看 B 线程被唤醒之后的执行逻辑**

再次回到 lock() 方法的执行流程中来，线程 B 被 unpark() 之后将不再阻塞，继续执行下面的程序，线程 B 正常被唤醒，因此`Thread.interrupted()`的值为 false，表示线程 B 未被中断。

![](rel-25.png)

回到上一层方法中，此时 lock 锁未被占用，线程 B 执行 `tryAcquire(arg)` 方法能够抢到 lock 锁，并且将 `state` 变量的值设置为 1，表示该 lock 锁已经被占用。

![](rel-21.png)

接着来研究下 `setHead(node)` 方法：传入的节点为 `nodeB`，头指针指向 `nodeB` 节点；将 `nodeB` 中封装的线程置为 `null`（因为已经获得锁了）；`nodeB` 不再指向其前驱节点（哨兵节点）。这一切都是为了将 `nodeB` 作为新的哨兵节点。

![](rel-32.png)

执行完 `setHead(node)` 方法的状态如下图所示。

![](rel-33.png)

将 `p.next` 设置为 `null`，这是原来的哨兵节点就是完全孤立的一个节点，此时 `nodeB` 作为新的哨兵节点

![](rel-34.png)

线程 C 也是类似的执行流程。

### AQS 总结

#### AQS 的考点

第一个考点：我相信你应该看过源码了，那么AQS里面有个变量叫State，它的值有几种？

答：3个状态：没占用是0，占用了是1，大于1是可重入锁

第二个考点：如果锁正在被占用，AB两个线程进来了以后，请问这个总共有多少个Node节点？

答：答案是3个，分别是哨兵节点、nodeA、nodeB

#### AQS 源码解读案例图示

![](rel-35.png)
