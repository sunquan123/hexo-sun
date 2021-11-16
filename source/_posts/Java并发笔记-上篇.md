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









