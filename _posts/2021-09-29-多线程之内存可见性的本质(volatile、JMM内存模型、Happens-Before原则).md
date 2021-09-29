---
layout: post
title: 多线程之内存可见性的本质(volatile、JMM内存模型、Happens-Before原则)
tags: 多线程
categories: 多线程
---

# 学习内容
- 初步认识Volitale
- 从硬件层面了解可见性的本质
- 什么是JMM
- Happens-Before规则

# 初步认识Volitale
## volatile的作用
 volatile可保证内存可见性
```java
/**
 * @PackageName: com.raven.multithreaded.memoryvisibility
 * @ClassName: VolatileTest
 * @Blame: raven
 * @Date: 2021-08-16 20:53
 * @Description: 使用volatile关键字保证内存可见性
 */
public class VolatileTest {
    private volatile static boolean flag = false;
    public static void main(String[] args) throws InterruptedException {

        Thread thread = new Thread(() -> {
            // thread线程 正常情况下是会一直输出字符串，
            // 因为flag被volatile修饰，能够确保内存可见性，
            // 所以当主线程修改flag后，会读取flag的值，停止输出跳出循环，终止线程。
            while (!flag) {
                System.out.println("thread running");
            }
        });
        thread.start();
        Thread.sleep(1000);
        flag = true;
    }
}
```
volatile为什么不能解决原子性？

线程A与线程B可同时拿到主内存中i的值,并进行update，最终返回结果，整个过程不是原子操作，俩个线程都是将i+1，他们拿到的i的值也都是0，所以最终i的结果为1而不是想要的结果2.

![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/611b78846667021afa953b61.png)

## 如何保证可见性
通过hsdis工具反编译程序，可以看到汇编指令中多了一个Lock的汇编指令，从而确保内存可见性

## 可见性到底是什么？
通过不同层面了解可见性是什么。
- 硬件方面
- JMM层面

# 从硬件层面了解内存可见性的本质
```doc
计算器处理数据，主要分为CPU、内存、I/O设备，而他们处理数据的能力也是大不相同，
同样的数据CPU处理可能需要1ms，内存或许需要10ms，而I/O设备则可能需要100ms，
所以如果能最大化的利用CPU资源，就可以最高效的处理数据
```
## 最大化的利用CPU资源
- CPU增加高速缓存
- 引入线程、进程
- 指令优化 -> 重排序

## CPU的高速缓存
```doc
为最大化的提高COU资源利用率，可通过增加高速缓存达到目的(L1/L2/L3)，
即多核CPU在处理数据时，将数据进行多级缓存，从缓存中获取数据，不直接从主内存中获取数据，从而提高效率。
```
![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/611b55a86667021afa953466.png)

```doc
但是多级缓存就会面临缓存一致性问题，即有可能出现缓存数据和本地数据不一致的情况，所以必须解决缓存一致性问题。
CPU层面的解决方案主要有：
1.总线锁，即在CPU0要做 i++操作的时候，其在总线上发出一个LOCK#信号，其他处理器就不能操作缓存了该共享变量内存地址的缓存，也就是阻塞了其他CPU，使该处理器可以独享此共享内存。(锁的力度大，降低效率)
2.缓存锁(缓存一致性协议)
```

## 缓存一致性协议(MESI)
```doc
缓存一致性机制就整体来说，是当某块CPU对缓存中的数据进行操作了之后，就通知其他CPU放弃储存在它们内部的缓存，或者从主内存中重新读取
MESI表达的是缓存行的四种状态(CPU硬件层面的实现)
M(modify):被修改的.处于这一状态的数据，只在本CPU中有缓存数据，而其他CPU中没有。同时其状态相对于内存中的值来说，是已经被修改的，且没有更新到内存中。
E(exclusive):独占的.处于这一状态的数据，只有在本CPU中有缓存，且其数据没有修改，即与内存中一致。
S(shared):共享的。处于这一状态的数据在多个CPU中都有缓存，且与内存一致。
I(invalid):失效的。本CPU中的这份缓存已经无效。
```
**shard:**
![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/611b5c266667021afa95354d.png)

**S->M:**
![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/611b5c3d6667021afa953561.png)

**M->E:**
![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/611b5c4e6667021afa953565.png)

## MESI 带来的CPU阻塞问题
```doc
通过缓存一致性协议可以解决缓存一致性的问题，但同样带来了新的问题，即CPU阻塞的问题
CPU0和CPU1共享资源数据，当CPU0中共享数据发送变更后,会通信告诉CPU1该数据失效(invalidate),
然后等待CPU1发送回执消息,告知CPU0已经将该共享资源置状态修改为I，CPU0可以将状态修改E。
但是在CPU1回复的这一段时间内,CPU0是处于阻塞的，也就降低了CPU的资源利用
```
![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/611b5c596667021afa953568.png)
**通过storebuffer解决CPU阻塞问题**
```doc
为解决CPU阻塞问题，Cpu将更新操作缓存到了storebuffer,当其他CPU给了ack回执后，storebuffer再同步到主内存,
但storebuffer任然会存在内存可见性问题。
```
```java 
int value = 3;
boolean isFinish = false; 
void cpu0(){
    value = 10;  // (S->M) -> [storebuffer -> 通知其他cpu缓存行失效]
    isFinish = true; (E状态)
}

void cpu1(){
    if(isFinish){ // true
        assert value  10; // false
    }
}
```
```doc
理想状态下 isFinish  true assert value  10 也是 true
但是因为cpu的乱序执行 -> 指令重排序 -> 还是会有可见性问题.
所以在CPU层面上提供了指令 -> 内存屏障指令
```
## 内存屏障指令
**内存屏障指令用来解决可见性问题**

CPU层面提供了三种屏障:
- 写屏障(store barrier)
- 读屏障(load barrier)
- 全屏障(full barrier)

**通过内存屏障指令：**
- 确保一些特定操作执行的顺序； 
- 影响数据的可见性(可能是某些指令执行后的结果)。

# 什么是JMM？
## java内存模型

```doc
JMM最核心的价值就是解决了有序性、可见性，是语言级别抽象内存模型
通过volatile、synchronized、final、等所有满足了(happens-before)规则的方式，都可以确保内存的有序性、可见性
```

**源代码 -> 编译器的重排序 -> cpu层面的重排序（指令级/内存） -> 最终的指令**

## 语言级别的内存屏障
通过 javap -v 类名反编译java文件，可以看到被volatile修饰的成员变量在反编译后，当变量被修改时 都会执行putstatic的指令，并且被volatile修饰的属性最后都会执行OrderAccess::storeload指令。从而确保操作对其他线程内存可见。

![image](http://dipaov2.oss-cn-shanghai.aliyuncs.com//test/edipao/driver/image/611b70146667021afa953949.png)
不同平台下JMM的内存屏障的实现是不一样的

x86下 : **lock**

linux : **storeload**

# Happens-Before规则
**可见性的保障-> 除了volatile以外,还可以通过其他方式确保内存可见性，只需要满足Happens-Before规则即可**

## volatile规则
```java
/**
 * @PackageName: com.raven.multithreaded.memoryvisibility.happensbefore
 * @ClassName: VolatileRule
 * @Blame: raven
 * @Date: 2021-08-16 21:16
 * @Description: happens-before规则 - volatile规则
 * 被volatile关键字修饰的变量 在被修改的时候都会有内存屏障指令(OrderAccess::storeload)
 * 从而确保内存的可见性
 */
public class VolatileRule {

    volatile static int num = 0;
    volatile static boolean flag = false;

    public static void writer() {
        // 1
        num = 10;
        // 2
        flag = true;
        // 1 happens-before 2
    }

    public static void reader() {
        // 3
        if (flag) {
            // 4
            num = 100;
        }
        // 3 happens-before 4
    }

    public static void main(String[] args) throws InterruptedException {
        System.out.println("before:" + num);
        // thread线程修改 num 和 flag的值
        new Thread(() -> {
            writer();
        }).start();

        // 主线程读取，仍然可以读取到修改后的值
        Thread.sleep(1000);
        reader();
        System.out.println("after:" + num);
    }
}
```

## 程序的顺序规则
```java
/**
 * @PackageName: com.raven.multithreaded.memoryvisibility.happensbefore
 * @ClassName: SingleThreadSequenceRule
 * @Blame: raven
 * @Date: 2021-08-16 21:07
 * @Description: happens-before规则 - 程序的顺序规则
 * 单线程下程序一定顺序执行
 * 可以保障内存的可见性
 */
public class SingleThreadSequenceRule {

    static int num = 0;
    static boolean flag = false;

    public static void writer() {
        // 1
        num = 10;
        // 2
        flag = true;
        // 1 happens-before 2
    }

    public static void reader() {
        // 3
        if (flag) {
            // 4
            num = 100;
        }
        // 3 happens-before 4
    }

    public static void main(String[] args) {
        writer();
        reader();
        System.out.println(num);
    }
}
```
## 传递性规则

```java
/**
 * @PackageName: com.raven.multithreaded.memoryvisibility.happensbefore
 * @ClassName: TransmissibilityRule
 * @Blame: raven
 * @Date: 2021-08-16 21:24
 * @Description: happens-before规则 - 传递性规则
 * 线程间的happens-before依赖会进行传递
 * 即：1 happens-before 2
 * 2 happens-before 3
 * 则 1 happens-before 3
 */
public class TransmissibilityRule {
    static int num = 0;
    static boolean flag = false;

    public static void writer() {
        // 1
        num = 10;
        // 2
        flag = true;
        // 1 happens-before 2
    }

    public static void reader() {
        // 3
        if (flag) {
            // 4
            num = 100;
        }
        // 3 happens-before 4
    }

    public static void main(String[] args) {
        writer();
        reader();
        System.out.println(num);
    }
}
```

## 线程启动规则
```java
/**
 * @PackageName: com.raven.multithreaded.memoryvisibility.happensbefore
 * @ClassName: StartRule
 * @Blame: raven
 * @Date: 2021-08-16 21:27
 * @Description: happens-before规则 - 启动规则
 * 在线程调用start方法前，其他线程的修改是对该线程可见的
 */
public class StartRule {
    static int num = 0;

    public static void main(String[] args) {
        Thread thread = new Thread(() -> {
            System.out.println(num);
        });
        num = 100;
        thread.start();

    }
}
```

## join规则
```java
/**
 * @PackageName: com.raven.multithreaded.memoryvisibility.happensbefore
 * @ClassName: JoinRule
 * @Blame: raven
 * @Date: 2021-08-16 21:43
 * @Description: happens-before规则 - join规则
 */
public class JoinRule {
    static int num = 0;

    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(() -> {
            num = 100;
        });
        thread.start();
        // join会阻塞主线程 底层是基于wait/notify通信的
        // 等到阻塞释放，获取thread线程的执行结果
        // join会主动简历一个 happens-before规则
        thread.join();
        System.out.println(num);
    }
}
```
如何让多个线程顺序执行？
```java
/**
 * @PackageName: com.raven.multithreaded.memoryvisibility
 * @ClassName: JoinSequenceTest
 * @Blame: raven
 * @Date: 2021-08-16 21:48
 * @Description: 通过join 确保多个线程顺序执行
 */
public class JoinSequenceTest {

    public static void main(String[] args) throws InterruptedException {
        Thread thread1 = new Thread(() -> {
            System.out.println("thread 1 sth");
        });
        Thread thread2 = new Thread(() -> {
            System.out.println("thread 2 sth");
        });
        Thread thread3 = new Thread(() -> {
            System.out.println("thread 3 sth");
        });

        // 通过join确保多个线程间顺序执行的原理就是建立了 happens- before原则
        // 从而使程序 顺序输出 thread 1  thread 2  thread 3
        thread1.start();
        thread1.join();
        thread2.start();
        thread2.join();
        thread3.start();
    }
}

```
## sync 监视器锁
```java
/**
 * @PackageName: com.raven.multithreaded.memoryvisibility.happensbefore
 * @ClassName: SyncRule
 * @Blame: raven
 * @Date: 2021-08-16 21:49
 * @Description: happens-before规则 - synchronized锁
 * 加锁可以满足 happens-before规则 ，确保数据的内存可见性
 */
public class SyncRule {
    public void demo(){
        synchronized (this){
            // do sth
        }
    }
}

```
