---
layout: shoka
title: jvm笔记-上篇
date: 2021-11-20 17:39:49
tags:

---

# JVM笔记-上篇

## Java的几种OOM情况

### 一、堆溢出

Java堆用于存储对象实例，只要不断的创建对象，并保证GC Roots到对象之间有可达路径来避免垃圾回收机制清除这些对象，那么在对象数量达到最大堆容量后就会产生内存溢出异常。下面代码限制Java堆的大小为10M，通过参数-XX:HeapDumpOnOutOfMemoryError可以实现虚拟机在出现内存溢出的情况下将当前内存堆栈快照dump出来以供后续分析需要。

#### 动手验证

```java
public class OomTest {
  public static void main(String[] args) {
    testHeapSpaceOom();
  }

  /*jvm options：-Xms10M -Xmx10M -XX:+HeapDumpOnOutOfMemoryError*/
  public static void testHeapSpaceOom() {
    List<OomTest> list = new ArrayList<>();
    while (true) {
      list.add(new OomTest());
    }
  }
}
```

#### 现象

![](oom异常1.png)

### 二、虚拟机栈和本地方法栈溢出

由于在hotspot虚拟机中不区分虚拟机栈和本地方法栈，因此栈容量只由-Xss参数设定。关于虚拟机栈和本地方法栈，在Java虚拟机规范中描述了两种异常：

1. 如果线程请求的栈深度超过了虚拟机所允许的最大深度，就抛出了大家喜闻乐见的`StackOverFlowError`异常。

2. 如果虚拟机在拓展栈时无法申请到足够的空间，则抛出`OutOfMemoryError`异常。

#### 1、StackOverFlowError

下面例子思路就是无限递归调用方法，指定-Xss参数减少栈内存容量，导致栈帧溢出。

#### 动手验证

```java
  /*jvm options：-Xss160k*/
  public static void testStackOverFlowError() {
    try {
      throwStackOverFlowErrorMethod();
    } catch (Throwable t) {
      System.out.println("stack length:" + stackLength);
      throw t;
    }
  }

  public static void throwStackOverFlowErrorMethod() {
    stackLength++;
    throwStackOverFlowErrorMethod();
  }
```

#### 效果

![](oom异常5.png)

#### 2、OutOfMemoryError

下面例子说明了在创建线程过多的情况下，虚拟机无法申请更多的空间创建线程了，则会抛出此异常。PS：这里动手验证可能会导致操作系统假死，就不验证了。

#### 动手验证

```java
  /*jvm options：-Xss2M*/
  public static void testCreateThreadOutOfMemoryError() {
    while (true) {
      Thread t =
          new Thread(
              new Runnable() {
                @Override
                public void run() {
                  while (true) {}
                }
              });
      t.start();
    }
  }
```

#### 效果

```java
java.lang.OutOfMemoryError: unable to create new native thread
```

### 三、方法区和运行时常量池溢出

运行时常量池是方法区的一部分，因此这两个区域的溢出测试放在一起进行。

#### 1、字符串常量池小考点

`String.intern()`是一个Native方法，作用是如果字符串常量池中已经存在这个等于`String对象`的字符串，则直接返回池中该字符串对应的对象；否则，将此`String对象`包含的字符串添加到常量池中，并返回此对象的引用。

下面一个例子在jdk1.6中打印false和false，在jdk1.8中打印出true和false，引申出一次探究：`String.intern()`在jdk1.6中会把首次遇到的字符串实例复制到永久代中，返回的是永久代中这个对象的引用，而`StringBuilder`对象创建的字符串对象在Java堆中，必然不是同一个实例，所以打印出来false和false。在JDK1.8中，intern()方法不再复制实例，而是在字符串常量池中保存首次出现的实例引用，所以`intern()`返回的对象和`StringBuilder`创建的字符串对象是同一个实例，所以第一个返回true。第二个返回false是因为在虚拟机启动的过程中，`rt.jar`包中的`sun.msic.Version`类初始化了一个"java"字符串对象，导致此时字符串常量池中已经存在对这个字符串的引用了，下次`StringBuilder`在Java堆上创建字符串对象后，堆上的"java"对象和常量池中引用的初始化"java"对象不是同一个实例，导致返回false。

```java
  public static void testStringIntern() {
    String str1 = new StringBuilder("计算机").append("软件").toString();
    System.out.println(str1 == str1.intern());//true
    String str2 = new StringBuilder("ja").append("va").toString();
    System.out.println(str2 == str2.intern());//false
  }
```

#### 2、字符串常量池溢出的基本思路就是不停的向容器中添加新的字符串对象。

#### 动手验证

| 参数                       | 解释                                                                                                                                                                |
|:------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| -Xmx50m                  | 堆上限设置为50m                                                                                                                                                         |
| -XX:MaxMetaspaceSize=10m | 元空间上限10m                                                                                                                                                          |
| -XX:-UseGCOverheadLimit  | GC支出超标限制(超过98%的时间用来做GC并且回收了不到2%的堆内存) ，`-XX:-UseGCOverheadLimit`表示不限制也就是当用了98%的时间来进行垃圾回收却收集了不到2%的堆内存时不抛出`java.lang.OutOfMemoryError: GC overhead limit exceeded`异常 |

```java
  /*jvm options：-Xmx20m -XX:MaxMetaspaceSize=10m -XX:-UseGCOverheadLimit*/
  public static void testStringConstantPoolOom() {
    List<String> list = new ArrayList<String>();
    int strCount = 0;
    while (true) {
      list.add(String.valueOf(strCount++).intern());
    }
  }
```

#### 现象

![](oom异常2.png)

#### 3、方法区包含了Class的相关信息，例如类名、修饰符、常量池、字段描述、方法描述等。需要让方法区溢出的基本思路是产生大量的类去填满方法区。下面例子采用CGLib去生成类的方式让方法区溢出。

#### 动手验证

```java
  /*jvm options：-XX:MaxMetaspaceSize=10M*/
  public static void testMethodAreaOom() {
    while (true) {
      Enhancer enhancer = new Enhancer();
      enhancer.setSuperclass(TestOom.class);
      enhancer.setUseCache(false);
      enhancer.setCallback(
          new MethodInterceptor() {
            @Override
            public Object intercept(
                Object o, Method method, Object[] objects, MethodProxy methodProxy)
                throws Throwable {
              return methodProxy.invokeSuper(o, objects);
            }
          });
      enhancer.create();
    }
  }

  static class TestOom {}
```

#### 现象

![](oom异常3.png)

### 四、本机直接内存溢出

DirectMemory容量可以通过`-XX:MaxDirectMemorySize`指定，如果不指定，则默认为Java堆的最大值。下面例子越过了`DirectByteBuffer`类，直接通过反射获取`Unsafe`实例进行内存分配（`Unsafe`类的`getUnsafe()`方法限制了只有引导类加载器才会返回实例，也就是设计者希望只有`rt.jar`的类才能使用`Unsafe`的功能）。虽然使用`DirectByteBuffer`分配内存也能抛出内存溢出异常，但是它抛出异常时并没有真正向操作系统申请分配内存，只是通过计算得知内存无法分配，于是手动抛出异常，真正申请分配内存的方法是`Unsafe.allocateMemory()`方法。

#### 动手验证

```java
  static final int _1GB = 1024 * 1024 * 1024;
   /*jvm options：-Xmx20M -Xms20M -XX:MaxDirectMemorySize=10M*/
  public static void testDirectMemoryAreaOom() throws IllegalAccessException {
    int i = 0;
    try {
      Field field = Unsafe.class.getDeclaredFields()[0];
      field.setAccessible(true);
      Unsafe unsafe = (Unsafe) field.get(null);
      while (true) {
        unsafe.allocateMemory(_1GB);
        i++;
      }
    } catch (Exception e) {
      e.printStackTrace();
    } finally {
      System.out.println("分配次数：" + i);
    }
  }
```

#### 现象

![](oom异常4.png)

这里的MaxDirectMemorySize参数没有起作用。


