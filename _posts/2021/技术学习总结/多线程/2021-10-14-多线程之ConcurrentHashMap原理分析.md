---
layout: post
title: 多线程之ConcurrentHashMap原理分析
tags: 多线程
categories: 多线程
---
# 学习内容
- put的实现过程
- 不同版本区别
- 源码分析
- 为什么使用CounterCell来计算容器大小？
- 学习内容总结

# 不同版本区别
[HashMap？ConcurrentHashMap](https://blog.csdn.net/weixin_44460333/article/details/86770169)

# 基本的数据结构
**数组+链表+红黑树**
![image](http://oss.longmarch.work/clipboard.png)
# put的实现过程
**CHM的put方法实现过程中，会经过计算key的hash值、initTable(初始化节点)、存储数据、记录集合的size等流程**
- 计算key的hash值：通过计算key的hash值将数据散列的存储到各个节点
- initTable：通过sizeCtl确保在并发场景下值只进行一次节点的初始化，构建好map的节点数组列表，并记录好下次扩容的阈值。
- 存储数据：根据不同的hash值将数据存储到node节点数组的不同位置，当遇到hash值相同的数据时，构建链表存储数据。
    - 当node数组的长度到达sizeCtl阈值时，会触发数组的扩容
    - 如果链表的的长度大于8并且node数据的长度>64 此时如果再添加数据到当前链表中，会把当前链表转化为红黑树，当出现扩容的时候，如果链表的长度小于8，会把红黑树转化为链表

# 源码分析
## 源码中位运算巧妙的应用
- (h ^ (h >>> 16))：h代表hashCode，通过将hashcode与hashcode右移16位进行异或计算后得到的hash值会更加的散列。(是因为会让高16位和低十六位进行计算)
- tabAt(tab, i = (n - 1) & hash)：i代表索引值，n代表数组的length(数组容量capacity)，hash代表通过key计算的hash值 通过hash与数组长度进行&运算可以更高效的拿到数组索引下标,可以更散列的存储数据。
- sc = n - (n >>> 2): 通过位运算计算数组扩容的阈值

## 初始化tab以及sizeCtl的作用
### 初始化tab
```java
     /**
      * Initializes table, using the size recorded in sizeCtl.
      * 初始化数组，记录线程扩容的阈值
      */
    private final Node<K,V>[] initTable() {
         Node<K,V>[] tab; int sc;
         while ((tab = table)  null || tab.length  0) {
             // 默认sizeCtl为0 
             // 当有线程在执行初始化操作时为-1.线程礼让，让其他线程执行
             // 通过sizectl作为一个标记未，来保证线程安全问题
             if ((sc = sizeCtl) < 0)
                 Thread.yield(); // lost initialization race; just spin
             // CAS成功后 进行线程初始化将
             else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                 try {
                     if ((tab = table)  null || tab.length  0) {
                         int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                         @SuppressWarnings("unchecked")
                         Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                         table = tab = nt;
                         // [16 - （16 = 10000 -> 000100 = 4）]  = 12
                         // 数组扩容的阈值为12
                         sc = n - (n >>> 2);
                     }
                 } finally {
                     sizeCtl = sc;
                 }
                 break;
             }
         }
         return tab;
     }    
```
### sizeCtl的作用
- 值为-1时： 表示一个占位符，如果sizeCtl=1，表示当前已经有线程抢到了初始化的权限。
- 值为>0的数字时， sizeCtl = sc = n - (n >>> 2) 表示下一次扩容的大小
- 值为非-1的其他负数时，代表有几个线程正在扩容,如（-2）代表有一个线程正在进行扩容。


## addCount是做什么的？
    
    addCount是用来计数返回集合的size().
    
    方法中一共做了俩件重要的事情，一件事情是基于CounterCells数组并发的设置集合的baseCount（size）,第二件事是校验数组是否需要扩容

```java
   private final void addCount(long x, int check) {CounterCell[] as; long b, s;
    // 当CounterCell数组构建完成 或者尝试CAS设置Map的baseCount失败 就通过CounterCell数组分而治之设置value属性值
       if ((as = counterCells) != null || !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
           CounterCell a; long v; int m;
           boolean uncontended = true;
           if (as  null || (m = as.length - 1) < 0 ||
           // 通过ThreadLocalRandom.getProbe()生成一个线程安全的随机数
           // 通过& 运算确保结果集落在 m (as.length -1)中，然后获取数组的元素
               (a = as[ThreadLocalRandom.getProbe() & m])  null ||
           // 如果CounterCell已经存在，可以直接通过cas设置value的值
               !(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
               fullAddCount(x, uncontended);
               return;
           }
           if (check <= 1)
               return;
           // 计算所有数组的length之和 即为map集合的size
           s = sumCount();
       }
       
       // 校验数组是否需要扩容
       if (check >= 0) {
           Node<K,V>[] tab, nt; int n, sc;
           while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                  (n = tab.length) < MAXIMUM_CAPACITY) {
               int rs = resizeStamp(n);
               if (sc < 0) {
                   if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc  rs + 1 ||
                       sc  rs + MAX_RESIZERS || (nt = nextTable)  null ||
                       transferIndex <= 0)
                       break;
                   if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                       transfer(tab, nt);
               }
               else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                            (rs << RESIZE_STAMP_SHIFT) + 2))
                   transfer(tab, null);
               s = sumCount();
           }
       }
    } 
```

### CounterCell
- **CounterCell 实现流程**
![image](http://oss.longmarch.work/c1.png)
![image](http://oss.longmarch.work/c2.png)
![image](http://oss.longmarch.work/c3.png)

**多线程并发的增加容器的大小**
![image](http://oss.longmarch.work/w4.png)
- **CounterCell 实现原理**
```java
    // 通过CounterCell[]数组 分而治之，在多线程下高效率的并发的增加元素的个数
     private final void fullAddCount(long x, boolean wasUncontended) {
            int h;
            if ((h = ThreadLocalRandom.getProbe())  0) {
                ThreadLocalRandom.localInit();      // force initialization
                h = ThreadLocalRandom.getProbe();
                wasUncontended = true;
            }
            boolean collide = false;                // True if last slot nonempty
            // 通过自旋的方式初始化CounterCells数据 并设置元素属性
            for (;;) {
                CounterCell[] as; CounterCell a; int n; long v;
                // counterCells当数组构建好后，并且CounterCell元素不为空时 封装CounterCell元素，设置值后装入CounterCells数组
                if ((as = counterCells) != null && (n = as.length) > 0) {
                    if ((a = as[(n - 1) & h])  null) {
                        if (cellsBusy  0) {            // Try to attach new Cell
                            CounterCell r = new CounterCell(x); // Optimistic create
                            if (cellsBusy  0 &&
                                U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                                boolean created = false;
                                try {               // Recheck under lock
                                    CounterCell[] rs; int m, j;
                                    if ((rs = counterCells) != null &&
                                        (m = rs.length) > 0 &&
                                        rs[j = (m - 1) & h]  null) {
                                        rs[j] = r;
                                        created = true;
                                    }
                                } finally {
                                    cellsBusy = 0;
                                }
                                if (created)
                                    break;
                                continue;           // Slot is now non-empty
                            }
                        }
                        collide = false;
                    }
                    else if (!wasUncontended)       // CAS already known to fail
                        wasUncontended = true;      // Continue after rehash
                        // 设置更新指定CounterCell的value值
                    else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                        break;
                    else if (counterCells != as || n >= NCPU)
                        collide = false;            // At max size or stale
                    else if (!collide)
                        collide = true;
                    else if (cellsBusy  0 &&
                             U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        try {
                            if (counterCells  as) {// Expand table unless stale
                                CounterCell[] rs = new CounterCell[n << 1];
                                for (int i = 0; i < n; ++i)
                                    rs[i] = as[i];
                                counterCells = rs;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        collide = false;
                        continue;                   // Retry with expanded table
                    }
                    h = ThreadLocalRandom.advanceProbe(h);
                }
                // 初始化counterCells数组
                // 仅当counterCells数组为空时，并且通过CAS CELLSBUSY 标志位的方式确保只有一个线程初始化 counterCells数组
                else if (cellsBusy  0 && counterCells  as && U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                    boolean init = false;
                    try {               
                        // Initialize table
                        if (counterCells  as) {
                            // 构建长度为2的counterCells数组 
                            CounterCell[] rs = new CounterCell[2];
                            // 构建一个 CounterCell 对象 
                            // 将它设置counterCells数组的第一个元素
                            // 并且value值为1
                            rs[h & 1] = new CounterCell(x);
                            counterCells = rs;
                            init = true;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    if (init)
                        break;
                }
                else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
                    break;                          // Fall back on using base
            }
        }
```
###  为什么基于CounterCell来计算容器大小？
加锁计算容器的大小，大幅度降低性能

![image](http://oss.longmarch.work/w1.png)

通过CAS计算容器的大小，性能会有所下降

![image](http://oss.longmarch.work/w2.png)

通过CounterCell来计算容器的大小，可以在确保线程安全的前提下并发的增加容器的大小
![image](http://oss.longmarch.work/w3.png)

## 扩容

### 扩容源码分析
    /**
    * 
    * @param n 数组的长度
    * @return  resizeStamp(16) = 32795
    * 
    * 0000 0000 0000 0000 1000 0000 0001 1011
    */
     static final int resizeStamp(int n) {
         // Integer.numberOfLeadingZeros(n) 返回最高位无符号位前的0的个数
         // n = 16 = 10000 Integer.numberOfLeadingZeros(16)   
         return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
     }
     
        /**
    * 1.扩大数组的长度 2.数据迁移
    * @param tab
    * @param nextTab
    */
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
            int n = tab.length, stride;
            // 让每一段CPU去执行一段数据的扩容，每一个CPU可以处理16个长度的数组
            if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
                stride = MIN_TRANSFER_STRIDE; // subdivide range
                
             // 构建数组迁移需要的新的数据结构   
            if (nextTab  null) {            // initiating
                try {
                    @SuppressWarnings("unchecked")
                    // 构建一个当前容量二倍的数组 
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                    nextTab = nt;
                } catch (Throwable ex) {      // try to cope with OOME
                    sizeCtl = Integer.MAX_VALUE;
                    return;
                }
                nextTable = nextTab;
                // transferIndex = n 等于原本数组长度的2倍
                transferIndex = n;
            }
            int nextn = nextTab.length;
            ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
            boolean advance = true;
            boolean finishing = false; // to ensure sweep before committing nextTab
            
            for (int i = 0, bound = 0;;) {
                Node<K,V> f; int fh;
            // 设置分工的区间    
            // 划分每个线程负责的 转移数据的范围 【index ，bound】
                while (advance) {
                    int nextIndex, nextBound;
                    if (--i >= bound || finishing)
                        advance = false;
                    else if ((nextIndex = transferIndex) <= 0) {
                        i = -1;
                        advance = false;
                    }
                    else if (U.compareAndSwapInt
                             (this, TRANSFERINDEX, nextIndex,
                              nextBound = (nextIndex > stride ?
                                           nextIndex - stride : 0))) {
                        // 例： 数组length为16扩容为32 
                        // 第一个线程进来时 nextIndex = 32 stride = 16  nextBound = 16 bound = 16 i = 31
                        // 第二个线程进来时 bound = 0 i = 15
                        bound = nextBound;
                        i = nextIndex - 1;
                        advance = false;
                    }
                }
                if (i < 0 || i >= n || i + n >= nextn) {
                    int sc;
                    if (finishing) {
                        nextTable = null;
                        table = nextTab;
                        sizeCtl = (n << 1) - (n >>> 1);
                        return;
                    }
                    // 线程处理完毕后，修改sizeCtl 的值，当前处理扩容的线程数量减少一个
                    if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                        if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                            return;
                        finishing = advance = true;
                        i = n; // recheck before commit
                    }
                }
                else if ((f = tabAt(tab, i))  null)
                    advance = casTabAt(tab, i, null, fwd);
                else if ((fh = f.hash)  MOVED)
                    advance = true; // already processed
                else {
                    // 数据迁移
                    // 主要做俩个事情  
                    // 1.将链表中的数据分为ln地位链 hn高位链
                    // 2.将数据放入新的数组列表下
                    synchronized (f) {
                        if (tabAt(tab, i)  f) {
                            Node<K,V> ln, hn;
                            if (fh >= 0) {
                                // 把当前链表进行分类 ln
                                int runBit = fh & n;
                                Node<K,V> lastRun = f;
                                for (Node<K,V> p = f.next; p != null; p = p.next) {
                                    int b = p.hash & n;
                                    if (b != runBit) {
                                        runBit = b;
                                        lastRun = p;
                                    }
                                }
                                if (runBit  0) {
                                    ln = lastRun;
                                    hn = null;
                                }
                                else {
                                    hn = lastRun;
                                    ln = null;
                                }
                                // 将链表分为不同的高低链路
                                for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                    int ph = p.hash; K pk = p.key; V pv = p.val;
                                    if ((ph & n)  0)
                                        ln = new Node<K,V>(ph, pk, pv, ln);
                                    else
                                        hn = new Node<K,V>(ph, pk, pv, hn);
                                }
                                // 低位链保持位置不动
                                setTabAt(nextTab, i, ln);
                                // 高位链需要移动n长度的位置
                                setTabAt(nextTab, i + n, hn);
                                setTabAt(tab, i, fwd);
                                advance = true;
                            }
                            // 红黑树的扩容逻辑
                            else if (f instanceof TreeBin) {
                                TreeBin<K,V> t = (TreeBin<K,V>)f;
                                TreeNode<K,V> lo = null, loTail = null;
                                TreeNode<K,V> hi = null, hiTail = null;
                                int lc = 0, hc = 0;
                                for (Node<K,V> e = t.first; e != null; e = e.next) {
                                    int h = e.hash;
                                    TreeNode<K,V> p = new TreeNode<K,V>
                                        (h, e.key, e.val, null, null);
                                    if ((h & n)  0) {
                                        if ((p.prev = loTail)  null)
                                            lo = p;
                                        else
                                            loTail.next = p;
                                        loTail = p;
                                        ++lc;
                                    }
                                    else {
                                        if ((p.prev = hiTail)  null)
                                            hi = p;
                                        else
                                            hiTail.next = p;
                                        hiTail = p;
                                        ++hc;
                                    }
                                }
                                ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                    (hc != 0) ? new TreeBin<K,V>(lo) : t;
                                hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                    (lc != 0) ? new TreeBin<K,V>(hi) : t;
                                setTabAt(nextTab, i, ln);
                                setTabAt(nextTab, i + n, hn);
                                setTabAt(tab, i, fwd);
                                advance = true;
                            }
                        }
                    }
                }
            }
        }
  ### 将当前的链表分为高低链，提高扩容的效率
  ![image](http://oss.longmarch.work/k1.png)
  
  ![image](http://oss.longmarch.work/k2.png)
    
  ### 为什么可以提高扩容的效率?
  ```java
  (f = tabAt(tab, i = (n - 1) & hash)
  因为计算hash值的公式是固定的，
  当数组进行扩容后，低位链的数据位置不变,高位链的位置在原来的基础上增加了n的位置
  ```
  ![image](http://oss.longmarch.work/k3.png)
  
# 学习疑问解答
- 为什么可以通过hash值 & n来进行区分高低链，同一链表上每个元素的hash值不相同吗？

不相同，在进行存储时，并不是仅仅通过hash值计算然后存储在数组中，而是hash值与(n -1) 进行与运算然后存储，所以同一链表的元素只是在节点数组中索引相同，hash不同。
![](http://oss.longmarch.work/AA1.png)
![](http://oss.longmarch.work/AA2.png)
- 构造函数中initailCapacity 初始化容量是直接使用吗？

不是，会将他转为最接近的2的n次方的容量
![](http://oss.longmarch.work/AA3.png)
- 什么情况下会转换为红黑树？

在treeifiBin中进行判断，是进行扩容还是将链表转为红黑树
![](http://oss.longmarch.work/AA4.png)

# 红黑树图解
## 二叉树
![](http://oss.longmarch.work/AA5.png)
## 二叉查找树
![](http://oss.longmarch.work/AA6.png)
## 红黑树
### 红黑树规则
![](http://oss.longmarch.work/AA7.png)
### 红黑树图解
![](http://oss.longmarch.work/AA8.png)
### 红黑树左旋/右旋
![](http://oss.longmarch.work/AA9.png)
![](http://oss.longmarch.work/AA10.png)
# 学习内容总结
- 通过数组的方式实现并发增加元素的个数
- 并发扩容，可以通过多个线程来进行实现数据的迁移
- 采用高低链的方式来解决多次Hash计算的问题，提高了效率
- sizeCtl的设计，3种标识状态
- resizeStamp的设计，高低位的设计来实现唯一性以及多个线程的协助扩容记录
