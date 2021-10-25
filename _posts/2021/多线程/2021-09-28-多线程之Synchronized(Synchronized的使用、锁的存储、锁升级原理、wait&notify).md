---
layout: post
title: 多线程之Synchronized(Synchronized的使用、锁的存储、锁升级原理、wait&notify)
tags: 多线程
categories: 多线程
---
# 1.学习收获
- 学习方法
- 如何保证线程安全性？
- Synchronized的基本使用
- 锁的存储
- Synchronized锁的升级原理
- wait/notify实现线程通信

# 2.学习方法
场景->需求->解决方案->应用->原理
- 场景: 多线程场景
- 需求：多线程并行产生的线程安全性
- 解决方案：加锁(Synchronized)
- 应用：synchronized的几种使用方式,对象锁、静态方法锁、代码块
- 原理：偏向锁(cas乐观锁) -> 轻量级锁(自旋锁) -> 重量级锁(mutex互斥)

# 3.如何保证线程安全性 
多线程并行环境下会产生线程安全问题，可通过管理数据状态的访问，即通过 **synchronized** 锁保证数据的安全性还要保证性能。

# 4.synchronized的基本使用
```java
/**
 * @PackageName: com.raven.multithreaded.synchronizedtheory
 * @ClassName: BasicUse
 * @Blame: raven
 * @Date: 2021-08-14 15:46
 * @Description: synchronized 基本使用
 * <p>
 * 2种表现形式
 * 2种作用范围 (对象锁or类锁)区别是 是否挂对象跨线程被保护
 * 控制锁的范围由对象的生命周期而决定！！！
 * 1.修饰实例方法
 * 2.修饰静态方法
 * 3.修饰代码块
 */
public class BasicUse {
    private final Object lock;

    public BasicUse(Object lock) {
        this.lock = lock;
    }
    /**
     * 修饰静态方法
     * 锁对象是BasicUse.class(对象锁)
     * 是类锁，不同实例访问该方法会出现锁竞争
     * 锁的范围较大(整个方法代码都会上锁)
     */
    public synchronized static void staticMethod() {
        System.out.println("synchronized修饰静态方法");
    }

    /**
     * 修饰费静态方法
     * 锁对象是BasicUse对象
     * 是对象锁，不同BasicUse实例间不会出现锁竞争
     * 锁的范围较大(整个方法代码都会上锁)
     */
    public synchronized void nonStaticMethod() {
        System.out.println("synchronized修饰非静态方法");
    }

    /**
     * 修饰代码块
     * 锁对象是lock对象，不同BasicUse实例传入同一个lock对象时会出现锁竞争
     * 锁的范围较小
     */
    public void codeBlockLock() {
        System.out.println("before sth..");
        synchronized (lock) {
            System.out.println("锁对象为 lock ");
        }
        System.out.println("after sth..");
    }

    /**
     * 修饰代码块
     * 锁对象是BasicUse的class类对象，会出现锁竞争
     * 锁的范围较小
     */
    public void codeBlockClass() {
        System.out.println("before sth..");
        synchronized (BasicUse.class) {
            System.out.println("锁对象为 BasicUse的class类对象 ");
        }
        System.out.println("after sth..");
    }
    /**
     * 修饰代码块
     * 锁对象是BasicUse对象，不同的BasicUse实例间不会出现锁竞争
     */
    public void codeBlockThis(){
        System.out.println("before sth..");
        synchronized (this) {
            System.out.println("锁对象为 BasicUse对象 ");
        }
        System.out.println("after sth..");
    }

}

/**
 * @PackageName: com.raven.multithreaded.synchronizedtheory
 * @ClassName: BasecUseTest
 * @Blame: raven
 * @Date: 2021-08-14 16:08
 * @Description: 模拟使用不同方式的锁
 */
public class BasicUseTest {

    public static void main(String[] args) {
        // 模拟使用synchronized修饰静态方法的锁
//        useStaticMethodLock();
        Object lock = new Object();
        BasicUse basicUse1 = new BasicUse(lock);
        BasicUse basicUse2 = new BasicUse(lock);

        // 模拟使用synchronized修饰非静态方法的锁
//        useNonStaticMethodLock(basicUse1, basicUse2);

        // 模拟使用synchronized修饰代码块 -> 锁对象为BasicUse对象的class对象
//        useCodeBlockMethodLock(basicUse1, basicUse2);

        // 模拟使用synchronized修饰代码块 -> 锁对象为传入的Object对象，不同实例不同线程间仍然线程安全
        useCodeLockMethodLock(basicUse1, basicUse2);


    }

    private static void useCodeLockMethodLock(BasicUse basicUse1, BasicUse basicUse2) {
        new Thread(() -> {
            basicUse1.codeBlockLock();
        }).start();
        new Thread(() -> {
            basicUse2.codeBlockLock();
        }).start();
    }

    private static void useCodeBlockMethodLock(BasicUse basicUse1, BasicUse basicUse2) {
        new Thread(() -> {
            basicUse1.codeBlockClass();
        }).start();
        new Thread(() -> {
            basicUse2.codeBlockClass();
        }).start();
    }

    private static void useNonStaticMethodLock(BasicUse basicUse1, BasicUse basicUse2) {
        new Thread(() -> {
            basicUse1.nonStaticMethod();
        }).start();
        new Thread(() -> {
            basicUse2.nonStaticMethod();
        }).start();
    }

    public static void useStaticMethodLock() {
        new Thread(() -> {
            BasicUse.staticMethod();
        }).start();
        new Thread(() -> {
            BasicUse.staticMethod();
        }).start();
    }
}

```
# 5.锁存储以及锁升级原理？


## 5.1锁的实现
**jdk1.6之前**，synchronized的实现 **是重量级锁**，即互斥锁，虽然能实现线程安全，但性能堪忧，**在1.6之后**，对内部实现做了优化，出现了锁升级的概念
**。即(偏向锁(cas乐观锁) -> 轻量级锁(自旋锁) -> 重量级锁(mutex互斥))**


### 5.1.1偏向锁
![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/6117ac106667026d5c49729c.png)

**cas[(Compare and swap(value,expect,update)]比较、
实现原子性、
乐观锁、**

### 5.1.2轻量级锁
![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/6117adb46667026d5c4972ae.png)


**什么是自旋？**

```java
boolean cas()

for(;;){// 自旋
    if(cas){
        return; // 标识成功
    }
}

```
**绝大多数的线程在获取锁以后，在非常daunt的实际内都会去释放锁**

**自旋会占用CPU资源，所以在指定的自旋次数之后，如果还没有获得轻量级锁，锁就会升级为重量级锁(可通过preBlockSpin 设置自旋次数，或使用自适应自旋)**

### 5.1.3重量级锁
当锁升级到重量级锁后，没有获取到锁的线程就会被阻塞 (->BLCOKED状态) 并且会产生系统级别的线程切换


**每一个对象都会存在有ObjectMonitor监视器,所以任何对象都可以成为锁对象，可以通过monitor()方法获取当前对象的监视器对象
ObjectMonitor就是实现重量级锁的核心**，monitor是一种MutexLock(互斥锁),不同os会有不同的实现，所以重量级锁慢的主要原因是因为会产生系统级别的线程切换(用户态<-<->->内核态),性能开销比较大

![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/6117b0e56667026d5c4972d5.png)

![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/6117b0fe6667026d5c4972d8.png)

```doc
java 同步代码块的方法会多出俩个指令  通过javap -v 类名称查看
一个是monitor(监视器)enter
一个是monitorexit

.......
    24: monitorexit
    25: goto          33
    28: astore_2
    29: aload_1
    30: monitorexit
......
```

![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/6117b03e6667026d5c4972ca.png)
重量级锁的实现: **A线程执行monitorenter指令，争抢锁对象，如果monitorenter成功，就会获取到锁对象，其他线程拿不到锁对象就会进入同步队列(每一个被阻塞的线程都会加入到该队列)，当A线程执行monitorexit指令后，随机唤醒同步队列中的一个线程，线程再次进行争抢锁对象。**







## 5.2锁在内存中的存储
![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/6117a7b26667026d5c497260.png)

![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/6117a7f76667026d5c497263.png)

## 5.3锁升级原理
### **synchronized在不同情况下使用不同的锁：**

假如有俩个线程ThreadA/ThreadB

**1.当只有ThreadA去访问锁对象(大多数情况)时，使用偏向锁，将ThreadA的ThreadID 标记到锁对象中，锁标志位修改为01。**
![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/6117a9e56667026d5c49727b.png)

**2.ThreadA和ThreadB交替访问，则将锁升级为轻量级锁，并通过自旋的方式尝试获取锁对象，将标志位改为00，轻量级锁将栈帧指向锁记录的指针。**
![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/6117aa226667026d5c49727f.png)

**3.多个线程同时访问时，锁会升级为重量级锁，并且会阻塞式访问。**
![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/6117ab966667026d5c497295.png)

# 6.wait 和 notify

wait、notify、notifyall
**线程的通信机制**

wait：**将线程阻塞并加入到同步队列中、释放当前的同步锁**

notify/notifyall：**唤醒被阻塞的线程**
```java
public class WaitThread extends Thread {

    private Object lock;

    public WaitThread(Object object) {
        this.lock = object;
    }

    @Override
    public void run() {
        synchronized (lock) {
            System.out.println("wait before:" + Thread.currentThread().getName() + " running");
            try {
                // 会做俩件事
                // 1.将当前线程阻塞加入等待队列
                // 2.释放锁
                lock.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("wait after:" + Thread.currentThread().getName() + " running");
        }
    }
}


public class NotifyThread extends Thread {

    private Object lock;

    public NotifyThread(Object object) {
        this.lock = object;
    }

    @Override
    public void run() {
        synchronized (lock) {
            System.out.println("notify before:" + Thread.currentThread().getName() + " running");
            lock.notify();
            System.out.println("notify after:" + Thread.currentThread().getName() + " running");
        }
    }
}

public class WaitAndNotifyTest {

    public static void main(String[] args) {
        Object lock = new Object();
        WaitThread waitThread = new WaitThread(lock);
        waitThread.start();
        NotifyThread notifyThread = new NotifyThread(lock);
        // notify线程和wait线程必须使用同一把锁，notify线程才能唤醒wait线程
//        Object lock2 = new Object();
//        NotifyThread notifyThread = new NotifyThread(lock2);
        notifyThread.start();
    }
}

```

![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/6117b22f6667026d5c4972e6.png)

**ThreadA线程执行monitorenter指令，获取到了锁对象，执行一些逻辑，当执行了lock.wait后，他会释放锁，并把当前线程加入到等待队列中(等待队列没有资格获得锁)**

**ThreadB线程执行monitorenter指令，获取到了锁对象，执行一些逻辑，当执行lock.notify后，会随机唤醒等待队列中的一个线程，移到同步队列， 然后执行monitorexit指令随机唤醒同步队列中的线程，再见通过锁竞争机制争取锁，执行接下来的逻辑。**
