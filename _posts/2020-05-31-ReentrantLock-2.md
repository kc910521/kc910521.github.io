---
layout: default
title: "ReentrantLock的设计不是偶然（二）"
tags: juc
---

   
   
# ReentrantLock的设计不是偶然（二）

本文基于JAVA 1.8.0_181

2020-05         Chunk

目录
* [一、解锁时发生了什么](#h1)
* [二、PARK 和 UNPARK ](#h2)
* [三、结语和问题](#h3)


## <a name="h1"></a>一、解锁时发生了什么


- 1.status操作：  
见面直接减 state，此处考虑了重入锁需要减多次。  

- 2.判断当前节点的 waitStatus：  
如果waitStatus != 0，说明这个节点不是队列中的最后一个节点。（详情见第一篇中[难点-1]）
不是最后一个就说明还有线程在 AQS 中等待，调用 unparkSuccessor 方法 unpark 唤醒后继节点（其实妥妥就是队列的第二个节点）。    


    *
    
    
        public final boolean release(int arg) {
            // 1。
            if (tryRelease(arg)) {
                Node h = head;
                // 2。
                if (h != null && h.waitStatus != 0)
                    unparkSuccessor(h);
                return true;
            }
            return false;
        }
        
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        } 
    


## <a name="h2"></a>二、PARK 和 UNPARK

摘录几个别人的总结：
    
    LockSupport是不可重入的，如果一个线程连续2次调用LockSupport.park()，那么该线程一定会一直阻塞下去。  
    
    Thread在park的时候如果调用了Thread.interrupt()方法，Thread会unpark并返回，不会抛异常。
    
    interrupt 实际上会调用 interrupt0, interrpt0是一个native方法，会调用thread.cpp，  
    thread.cpp会调用 os_linux.cpp，里面代码思路大致就是  
    获取到操作系统中对应java的线程，然后设置标志位，再设置内存屏障，再去unpark线程， 
    为什么会去unpark线程呢？因为如果是park状态，你连cpu时间片都分不到，谈何执行/通知？因此要unpark 实现线程的通信。

## <a name="h3"></a>三、结语和问题


- 为什么AQS持有队列头尾两个节点，一个不行吗？  
因为加锁等待要入队判断，解锁要出队，高频操作空间换时间。

- 为什么运行时不响应中断  
解锁时中断

- 会不会唤醒的同时，被唤醒线程又把自己wait了   
被唤醒永远是2号节点，但是二号节点不参与
见

- 公平锁和非公平锁  
即 先入队/先加锁 之区分。   
尝试之前会利用 hasQueuedPredecessors() 方法来判断 AQS 的队列中中是否有其他线程，如果有则不会尝试获取锁(这是公平锁特有的情况)。


* 下次说 ConditionObject，独占，共享  
