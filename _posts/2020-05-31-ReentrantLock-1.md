---
layout: default
title: "ReentrantLock的设计不是偶然（一）"
tags: juc
---

   
   
# ReentrantLock的设计不是偶然（一）

本文基于JAVA 1.8.0_181

2020-04         Chunk

目录
* [一、ReentrantLock 总结构概览](#Headings)
* [二、AQS 的结构](#SAQS)
    * [0.菜狗子制作的锁](#2-1)
    * [1.为什么锁的标识 state 不能是布尔类型？](#2-2)
    * [2.为什么需要锁的等待队列？](#2-3)
    * [3.如何将元素加入等待队列?](#2-4)
* [三、代码速查和小结](#CONCLUSION)





## <a name="Headings"></a>一、ReentrantLock 总结构概览

Lock接口声明了四种加锁方式和一种解锁方式；
ReentrantLock 实现了 Lock 接口，同时持有Sync类型的成员变量；
Sync 类（ReentrantLock的内部类）继承了 AbstractQueuedSynchronizer（AQS） 

Sync派生的两个类 FairSync 和 NonfairSync，实现了公平锁和非公平锁。

也即：  
在 ReentrantLock 中，Sync 作为内部类先继承了 AQS，同时公平锁/非公平锁又继承了 Sync。 


- 为什么不直接让公平锁和非公平锁直接去实现 AbstractQueuedSynchronizer 而需要 Sync 呢？

Sync 其作用在于封装，尤其注意其中 nonfairTryAcquire 方法，会在 ReentrantLock 
的 tryLock（） 方法中进行调用，也即 不论你是公平还是非公平锁， tryLock必然是以
非公平方式处理的。
同时被架空的 ReentrantLock 也因为自己持有了 Sync 而可以在自己被实例化时，确定应当使用哪种锁。  



- ReentrantLock 锁到底是怎么实现的呢

问得好，往下看：


这里，我们要先谈一下：  

## <a name="SAQS"></a>二、AQS 的结构

**<a name="2-1"></a>0.菜狗子制作的锁**

如果你需要手工实现一个锁去让一段业务代码只能同时有一个cpu核心在进行处理，该如何设计呢？ 

"菜狗子：我来！我有思路！" 

1. 先写一个类，叫 MyLock，再不妨定义一个类变量 state，类型就定义成boolean 吧，代表是否上锁。 
2. 写一个lock() 方法，判断如果 state 是 true 则说明没获取到锁，直接 Thread.sleep，死循环直到 state 为 false。
3. 如果 state 为 false ，则说明获取到锁了，将 isLock 置为 true，然后退出方法。 
4. 当然，变量state的修改都需要 CAS 操作。 

棒棒！看起来可以用，不过会遇到什么问题呢？ 



 **<a name="2-2"></a>1.为什么锁的标识 state 不能是布尔类型？**

AQS 中存在一个核心参数，state（int），它标志了当前是否有锁的竞争在发生。  
对应的就是菜狗子上文说的 state ，但是类型并不是 boolean。 
以非公平锁的加锁方法 lock() 为例：
    
    // 可暂时理解为CAS操作 0->1，详见文末 
    if (compareAndSetState(0, 1))
        // 成功则设置当前线程为独占线程
        setExclusiveOwnerThread(Thread.currentThread());
    else
        // 注意这个方法，下文有涉及
        acquire(1);
                
在一开始，非公平锁直接使用CAS的方式尝试将state的值从0变为1，
成功的话就等于获取到了锁，从而将当前线程设为独占线程，
失败则调用acquire方法，尝试添加 等待队列。

- 综上：  
如果采用 boolean ，那么当已经获取锁的线程 第二次调用 lock（）方法进行锁的重入时，因为 state 状态是上锁状态，会造成等待获取锁，
第二次上锁会永远等待第一次上锁所对应的解锁，而第一次上锁的解锁因为第二次的上锁等待而不会执行到。  
最终结果就是死锁。

**<a name="2-3"></a>2.为什么需要锁的等待队列？**

根据刚刚菜狗子的描述，当线程A获取到锁，线程 B、C、D再去获取锁时，都会进入死循环，空耗cpu资源。 
通过延长sleep的睡眠时间解决cpu空转，但是又带了睡眠时间无法确定，造成无法及时执行的问题。
我们不能让他们占着茅坑死循环，把他们收容起来，让出资源吧！

等待队列是所有没有获得到锁的线程的收容所。  
每一个任务都作为一个节点 Node （静态内部类）
借用源码中的高清图： 

     *      +------+  prev +-----+       +-----+
     * head |      | <---- |     | <---- |     |  tail
     *      +------+       +-----+       +-----+

AQS会将暂时无法取得锁的线程放入队列。AQS 只持有队列的head 和 tail节点，每个节点对应类型为Node，
但是Node本身持有上个节点对象 prev 和下个节点对象 next，
所以整个队列也是对AQS透明的，也就是在数据结构的角度可以理解为：
AQS结构上是一个使用链表实现的队列，
同时永远有一个thread为空的节点为正在工作的线程节点。    
**[难点-1]**  
AQS在每个节点加入队列时，不会处理自己的 waitStatus 状态，而是会告诉当前节点的前驱节点，你的状态为 'SIGNAL'。  
这是因为：  
- 头节点可能是当前正在工作的线程，否则就是一个单纯的、常驻的、初始化节点，不管什么情况,头节点的thread永远为空。
- 加锁是从队尾加，解锁是从队头判断状态获取。  
所以当前加入的节点去修改自己前驱节点的模式是最科学的。



最后，针对网上的文章，我不认为此结构为双端队列，双端队列在队列的两端都进行入队和出队。


**<a name="2-4"></a>3.如何将元素加入等待队列？**  

1.addWaiter： 每个需要竞争锁的线程都会被封装为 Node，通过 CAS 挂到队列的尾节点，
2.acquireQueued： 判断当前节点的前一个节点的waitStatus状态，是否是'SIGNAL'状态，
如果上个节点线程是park状态（实际为判断上个节点的waitStatus状态为'SIGNAL'状态），则当前线程也 park 等待（使用LockSupport.park）.  
3.如果上个节点状态正常但并非'SIGNAL'，则同时改变上个节点的ws为'SIGNAL'，否则层层向前删除节点。



我们即便不再看其他代码，你也能想出一个锁的实现思路了。

## <a name="CONCLUSION"></a>三、代码速查和小结

简单说：使用 AbstractQueuedSynchronizer 中的成员变量 state（int）标识锁状态，
同时使用一个 先进先出的队列保存等待线程。（CLH锁的变体）  


- 1.addWaiter  创建节点
创建一个节点，并将其以CAS的方式加入等待队列的尾部，
之前的尾部成为上一个节点，并返回这个节点。

   
        // from aqs
        private Node addWaiter(Node mode) {
            Node node = new Node(Thread.currentThread(), mode);
            // Try the fast path of enq; backup to full enq on failure
            Node pred = tail;
            if (pred != null) {
                node.prev = pred;
                if (compareAndSetTail(pred, node)) {
                    pred.next = node;
                    return node;
                }
            }
            enq(node);
            return node;
        }

- 2.acquireQueued 处理队列状态


        //node为上一步新建立的节点
       final boolean acquireQueued(final Node node, int arg) {
           boolean failed = true;
           try {
               boolean interrupted = false;
               for (;;) {
                   final Node p = node.predecessor();
                   // 前一个节点为头节点，且当前节点能获取到锁
                   if (p == head && tryAcquire(arg)) {
                        // 设置当前节点为头节点
                       setHead(node);
                       p.next = null; // help GC
                       failed = false;
                       return interrupted;
                   }
                   // 1。shouldParkAfterFailedAcquire方法  #####
                   // 先获取当前节点的前驱节点的waitStatus（ws）属性
                   // 如果前驱节点ws为-1，则直接返回true
                   // 如果前驱节点ws>0 则跳过且再往上上个节点查找，循环往复（队列被压缩了），最后返回false
                   // 如果前驱节点ws=0 or PROPAGATE，cas操作赋值前驱节点ws为-1，返回false
                   // 2。parkAndCheckInterrupt 方法 ######
                   // 直接调用 LockSupport.park，并返回线程是否中断
                   // 也就是本方法会使线程进入waiting状态
                   // 也就是1。当前节点的前驱节点状态为-1  
                   // 2。当前线程不再wait 且 是中断状态，interrupted=true  
                   // interrupted状态在这里没卵用，  
                   // 是为了返回这个状态给本方法调用者用  
                   if (shouldParkAfterFailedAcquire(p, node) &&
                       parkAndCheckInterrupt())
                       interrupted = true;
               }
           } finally {
               if (failed)
                   cancelAcquire(node);
           }
       }


- 3.acquire 对以上方法的调用

   
       public final void acquire(int arg) {
           if (!tryAcquire(arg) &&
               acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
               // acquireQueued 如果返回中断状态的话， 
               // 这里会补中断操作
               selfInterrupt();
       }





#### 下一章我们讲解解锁和唤醒