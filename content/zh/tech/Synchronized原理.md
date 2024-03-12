+++
title = "Synchronized原理"
date = "2023-11-05T14:35:47+08:00"
tags = []
slug = ""
+++

# 共享带来的问题

## 1. 临界区 Critical Section

多种情形：

- 一个程序单个线程

- 一个程序多个线程

    - 多线程仅读
    - **多线程有并发写操作** -> 线程并发问题

    临界区：存在对共享资源的**多线程读写**的代码块 

    如下：

    ```java
        private static int count = 0;
    		//临界区
        public static void increment() {
            count++;
        }
    		//临界区
        public static void decrement() {
            count--;
        }	
    ```

    ## 2. 竞态条件 race condition

    定义：多线程在**临界区**执行，由于代码的执行序列不同而导致结果无法预测，称之为发生了**竞态条件**。

    # synchronized

    ## 1. 解决方案：

    为了避免临界区的竞态条件发生，有多种手段可以达到目的。

    - 阻塞式的解决方案：synchronized，Lock
    - 非阻塞式的解决方案：原子变量，如 `AtomicInteger count = new AtomicInteger(0);`

    

    ## 2. synchronized语法

    锁对象：理论上可以是**任意的唯一对象**

    synchronized 是可重入、不公平的重量级锁

    原则上：

    - 锁对象建议使用共享资源
    - 在实例方法中使用 this 作为锁对象，锁住的 this 正好是共享资源
    - 在静态方法中使用类名 .class 字节码作为锁对象，因为静态成员属于类，被所有实例对象共享，所以需要锁住类

    ```java
    synchronized(Object){
      // Critical Section
    }
    ```

    例：

    ```java
    public class threaddemo1 {
        private static int count = 0;
        private static final int MAX = 1000000;
    
        public static void main(String[] args) throws InterruptedException {
            Thread t1 = new Thread(() -> {
              	//此时用threaddemo1这个类的.class对象作为锁对象
                synchronized (threaddemo1.class) {
                    for (int i = 0; i < MAX; i++) {
                        count++;
                    }
                }
            });
            Thread t2 = new Thread(() -> {
                synchronized (threaddemo1.class) {
                    for (int i = 0; i < MAX; i++) {
                        count--;
                    }
                }
            });
            t1.start();
            t2.start();
            t1.join();
            t2.join();
            System.out.println(count);
        }
    }
    ```

    如果加在方法上：

    - 加在成员方法成等同于用this上锁。
    - 加在静态方法上等同于用.class上锁。

    synchronized 修饰的方法的不具备继承性，所以子类是线程不安全的，如果子类的方法也被 synchronized 修饰，两个锁对象其实是一把锁，而且是**子类对象作为锁**

    ## 3. 变量的线程安全分析

    #### 成员变量和静态变量是否线程安全？

    - 如果它们没有共享，则线程安全
    - 如果它们被共享了，根据它们的状态是否能够改变，又分两种情况
        - 如果只有读操作，则线程安全
        - 如果有读写操作，则这段代码是临界区，需要考虑线程安全

    #### 局部变量是否线程安全？

    - 局部变量是线程安全的

    - 但局部变量引用的对象则未必 （要看该对象是否被共享

        且被执行了读写操作）

        - 如果该对象没有逃离方法的作用范围，它是线程安全的
        - 如果该对象逃离方法的作用范围，需要考虑线程安全

    - 局部变量是线程安全的——每个方法都在对应线程的栈中创建栈帧，不会被其他线程共享

        ![img|375](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20200608144636.png)

    - 如果调用的对象被共享，且执行了读写操作，则**线程不安全**![img|425](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20200608144649.png)

    - 如果是局部变量，则会在堆中创建对应的对象，不会存在线程安全问题。

[![img|450](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20200608144702.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20200608144702.png)

### 常见线程安全类

- String
- Integer
- StringBuﬀer
- Random
- Vector （List的线程安全实现类）
- Hashtable （Hash的线程安全实现类）
- java.util.concurrent 包下的类

这里说它们是线程安全的是指，多个线程调用它们**同一个实例的某个方法时**，是线程安全的

- 它们的每个方法是原子的（都被加上了synchronized）
- 但注意它们**多个方法的组合不是原子的**，所以可能会出现线程安全问题

[![img|500](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20200608144903.png)](https://nyimapicture.oss-cn-beijing.aliyuncs.com/img/20200608144903.png)

# Monitor

## 对象内存布局

`HotSpot`虚拟机中，对象在内存中存储的布局可以分为三块区域：对象头（`Header`）、实例数据（`Instance Data`）和对齐填充（`Padding`）。

![image-20240312143715171](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/image-20240312143715171.png)

从上面的这张图里面可以看出，对象在内存中的结构主要包含以下几个部分：

- `Mark Word`(标记字段)：对象的`Mark Word`部分占`4`个字节，其内容是一系列的标记位，比如轻量级锁的标记位，偏向锁标记位等等。
- `Klass Pointer`（`Class`对象指针）：`Class`对象指针的大小也是4个字节，其指向的位置是对象对应的`Class`对象（其对应的元数据对象）的内存地址
- 对象实际数据：这里面包括了对象的所有成员变量，其大小由各个成员变量的大小决定，比如：`byte`和`boolean`是1个字节，`short`和`char`是2个字节，`int`和`float`是4个字节，`long`和`double`是8个字节，`reference`是4个字节
- 对齐：最后一部分是对齐填充的字节，按`8`个字节填充。

### 对象头详情

对象头包括两部分：`Mark Word` 和 类型指针。

#### **标记字段（Mark Word）**

`MarkWord`用于存储对象自身的运行时数据， 如哈希码（`HashCode`）、`GC`分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等等。

这部分数据的长度在`32`位和`64`位的虚拟机（暂不考虑开启**压缩指针**的场景）中分别为`32`个和`64个bits`。

对象需要存储的运行时数据很多，其实已经超出了`32`、`64`位`Bitmap`结构所能记录的限度，但是对象头信息是与对象自身定义的数据无关的额外存储成本，考虑到虚拟机的空间效率，Mark Word被设计成一个**非固定的数据结构**以便在极小的空间内存储尽量多的信息，它会**根据对象的状态复用自己的存储空间**。

例如在`32`位的`HotSpot`虚拟机中对象未被锁定的状态下，`Mark Word`的`32`个`bits`空间中的`25bits`用于存储对象哈希码（`HashCode`），`4bits`用于存储对象分代年龄，`2bits`用于存储锁标志位，`1bit`固定为0，在其他状态（轻量级锁定、重量级锁定、`GC`标记、可偏向）下对象的存储内容如下表所示。

![image-20240312143804773](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/image-20240312143804773.png)

### Monitor

monitor在JVM中是基于C++的实现的，ObjectMonitor中有几个关键属性：

> _owner：指向持有ObjectMonitor对象的线程
> _WaitSet：存放处于wait状态的线程队列
> _EntryList：存放处于等待锁block状态的线程队列
> _recursions：锁的重入次数
> _count：用来记录该线程获取锁的次数

![image-20240312143812661](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/image-20240312143812661.png)

    当多个线程同时访问一段同步代码时，首先会进入_EntryList队列中，当某个线程获取到对象的monitor后进入_Owner区域并把monitor中的_owner变量设置为当前线程，同时monitor中的计数器_count加1。即获得锁。
    若持有monitor的线程调用wait()方法，将释放当前持有的monitor，_owner变量恢复为null，_count自减1，同时该线程进入_WaitSet集合中等待被唤醒。
    若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)

### 重量级锁：

1. **锁的获取**:
    - 当线程要执行被`synchronized`修饰的代码时（如`synchronized(obj){}`），它需要获取与该代码关联对象（obj）的内置monitor。
    - 如果monitor是空闲的（即没有其他线程拥有它），那么请求它的线程就会成为该monitor的所有者，并进入同步代码块执行操作。
    - 如果monitor已经被另一个线程占有（有一个Owner），那么请求它的线程就会被阻塞，并放入到所谓的EntryList（或者叫Blocked Set）。
2. **锁的释放与等待**:
    - 当拥有monitor的线程离开`synchronized`块，它会释放monitor。
    - 如果这时有线程在WaitSet中，它们需要被`notify()`或`notifyAll()`唤醒来尝试重新获取monitor。
    - 被唤醒的线程将移动到EntryList中，当monitor变为可用时，这些线程将尝试获取它以进入`synchronized`块。
3. **注意点**:
    - 只有获得了synchronized修饰的对象的monitor所有权，线程才能执行这个代码块。
    - 每个对象都有一个**唯一的monitor**，这个monitor是通过对象头中的一个Mark Word来实现控制的。Mark Word是对象头的一部分，用于存储对象自身以及锁的信息，包括锁的状态或者指向锁记录的指针。
4. **重入性**:
    - Java中的锁是可重入的，即如果一个线程已经拥有了某个对象的锁，那么它可以再次进入由该对象保护的另一个`synchronized`块，而不会发生死锁。

### 锁的升级

如上是重量级锁的上锁流程，但是显而易见的是，这种方式的性能耗费太大了，需要不断的进行用户态和内核态的切换。很多时候并不需要如此重量级的锁，所以我们有些更轻量的设计，而也就有了锁升级的过程。锁升级的整体过程为：`无锁->偏向锁->轻量级锁->重量级锁`

下面的描述是开启了偏向锁的情况下。

#### 偏向锁

HotSpot的作者经过研究发现，大多数情况下，锁不仅不存在多线程竞争，而且总是由同 一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。当一个线程访问同步块并 获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出 同步块时不需要进行CAS操作来加锁和解锁，只需简单地测试一下对象头的Mark Word里是否 存储着指向当前线程的偏向锁。如果测试成功，表示线程已经获得了锁。如果测试失败，则需 要再测试一下Mark Word中偏向锁的标识是否设置成1(表示当前是偏向锁):如果没有设置，则 使用CAS竞争锁;如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。



偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时， 持有偏向锁的线程才会释放锁。偏向锁的撤销，需要等待全局安全点(在这个时间点上没有正 在执行的字节码)。它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着， 如果线程不处于活动状态，则将对象头设置成无锁状态;如果线程仍然活着，拥有偏向锁的栈 会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word要么重新偏向于其他 线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。



（1）偏向锁的获取
注意：当JVM启动了偏向锁模式（Java 6和Java 7里是默认启动的），新创建对象的Mark Word中的ThreadID为0，说明此对象处于偏向锁状态（但未偏向任何线程），也叫作匿名偏向锁状态。

线程A第一次访问同步代码块时，先检查对象头Mark Word中锁标志位是否为01，依此判断此时对象是否处于无锁状态或者偏向锁状态；

若锁标志位是为01，然后判断偏向锁的标识是否为1：
    2.1 如果不是，则进入轻量级锁逻辑（使用CAS竞争锁）（注意：此时不是使用CAS尝试获取偏向锁，而是直接升级为轻量级锁；原因是：当偏向锁的标识为0时，表明偏向锁在此对象上被禁用，禁用原因可能是JVM关闭了偏向锁模式，或该类刚经历过bulk revocation，等等。所以应该入轻量级锁逻辑）；
    2.2 如果是1，表明此对象是偏向锁状态，则进行下一步流程。

判断是偏向锁时，检查对象头Mark Word中记录的ThreadID是否是当前线程A的ID：
    3.1 如果是，则表明当前线程A已经获得过该对象锁，以后线程A进入同步代码块时，不需要CAS进行加锁，只会往当前线程A的栈中添加一条Displaced Mark Word为空的Lock Record，用来统计重入的次数。如下图。
    3.2 如果不是，则进行CAS操作，尝试将当前线程A的ID替换进Mark Word；
      3.2.1 .如果当前对象锁的ThreadID为0（匿名偏向锁状态），则会替换成功（将Mark Word中的Thread id由匿名0改成当前线程A的ID，在当前线程A栈中找到内存地址最高的可用Lock Record，将线程A的ID存入），获得到锁，执行同步代码块。
      3.2.2 .如果当前对象锁的ThreadID不为0，即该对象锁已经被其他线程B占用了，则会替换失败，开始进行偏向锁撤销。这也是偏向锁的特点，一旦出现线程竞争，就会撤销偏向锁。

（2）偏向锁的撤销
偏向锁的撤销需要等待全局安全点（safe point，代表了一个状态，在该状态下所有线程都是暂停的，stop-the-world），到达全局安全点后，持有偏向锁的线程B也被暂停了。
检查持有偏向锁的线程B的状态（会遍历当前JVM的所有线程，如果能找到线程B，则说明偏向的线程B还存活着）：
    5.1 如果线程还存活，则检查线程是否还在执行同步代码块中的代码：
      5.1.1 如果是，则把该偏向锁升级为轻量级锁，且原持有偏向锁的线程B继续获得该轻量级锁。
    5.2 如果线程未存活，或线程未在执行同步代码块中的代码，则进行校验是否允许重偏向：
      5.2.1 如果不允许重偏向，则将Mark Word设置为无锁状态（未锁定不可偏向状态），然后升级为轻量级锁，进行CAS竞争锁。
      5.2.2 如果允许重偏向，设置为匿名偏向锁状态（即线程B释放偏向锁）。当唤醒线程后，进行CAS将偏向锁重新指向线程A（在对象头和线程栈帧的锁记录中存储当前线程ID）。
唤醒暂停的线程，从安全点继续执行代码。



![image-20240312143950109](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/image-20240312143950109.png)

#### 轻量级锁

1. 轻量级锁加锁

线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并 将对象头中的Mark Word复制到锁记录中，官方称为Displaced Mark Word。然后线程尝试使用 CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失 败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。

![image-20240312143902900](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/image-20240312143902900.png)

2. 轻量级锁解锁

轻量级解锁时，会使用原子的CAS操作将Displaced Mark Word替换回到对象头，如果成 功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。因为自旋会消耗CPU，为了避免无用的自旋(比如获得锁的线程被阻塞住了)，一旦锁升级 成重量级锁，就不会再恢复到轻量级锁状态。当锁处于这个状态下，其他线程试图获取锁时， 都会被阻塞住，当持有锁的线程释放锁之后会唤醒这些线程，被唤醒的线程就会进行新一轮 的夺锁之争。

![image-20240312144048947](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/image-20240312144048947.png)

>当前只有一个等待线程，则该线程通过自旋进行等待。但是当自旋超过一定的次数，或者一个线程在持有锁，一个在自旋，又有第三个来访时，轻量级锁升级为重量级锁.

#### 重量级锁

重量级锁依赖对象内部的`monitor`锁来实现，而`monitor`又依赖操作系统的`MutexLock`（互斥锁）

`Mutex`变量的值为`1`，表示互斥锁空闲，这个时候某个线程调用lock可以获得锁，而`Mutex`的值为`0`表示互斥锁已经被其他线程获得，其他线程调用`lock`只能挂起等待

获取锁时,锁对象的`Mark Word`中存储的是指向重量级锁的指针，此时等待锁的线程都会进入阻塞状态。

我们经常看见的`synchronized`就是非常典型的重量级锁，通过指令`moniter enter` 加锁，`moniter exit`解锁。

#### 为什么重量级锁开销比较大

原因是当系统检查到是重量级锁之后，会把等待想要获取锁的线程阻塞，被阻塞的线程不会消耗`CPU`，但是阻塞或者唤醒一个线程，都需要通过操作系统来实现，也就是相当于从用户态转化到内核



![image-20240312144019597](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/image-20240312144019597.png)







关于偏向锁和lock record的拓展阅读：[死磕synchronized六：偏向锁居然也会用到lock record](https://juejin.cn/post/7036284339847430151)





refer：

[黑马程序员juc笔记](https://github.com/Seazean/JavaNote/blob/main/Prog.md)

[并发编程笔记](https://nyimac.gitee.io/2020/06/08/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/)

[难搞的偏向锁终于被 Java 移除了](https://www.cnblogs.com/FraserYu/p/15743542.html)

[JAVA 对象头分析及Synchronized锁](https://www.cnblogs.com/hongdada/p/14087177.html)

[synchronized与对象的Monitor](https://dymanzy.github.io/2017/08/07/synchronized%E4%B8%8E%E5%AF%B9%E8%B1%A1%E7%9A%84Monitor/)

[Java面试常见问题：Monitor对象是什么？](https://zhuanlan.zhihu.com/p/356010805)

[Synchronized 轻量级锁会自旋？好像并不是这样的。](https://www.cnblogs.com/yescode/p/14474104.html)

[偏向锁的获取和撤销详解](https://blog.csdn.net/weixin_43882265/article/details/121318419)

[synchronized原理和偏向锁、轻量级锁、重量级锁的升级过程](https://blog.csdn.net/MariaOzawa/article/details/107665689?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522163687686616780366594305%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D&request_id=163687686616780366594305&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_v31_ecpm-1-107665689.first_rank_v2_pc_rank_v29&utm_term=%E5%81%8F%E5%90%91%E9%94%81%E7%9A%84%E6%A0%87%E8%AF%86%E4%B8%BA0%E6%97%B6%E4%B8%BA%E4%BB%80%E4%B9%88%E4%BC%9A%E5%8D%87%E7%BA%A7%E4%B8%BA%E8%BD%BB%E9%87%8F%E7%BA%A7%E9%94%81&spm=1018.2226.3001.4187)

《java并发编程的艺术》