
#草稿 draft


现在我们有了更多问题：  

##### 为什么state是int类型而非bool呢？
你是否考虑过锁重入的发生呢？当同一个线程调用第二次lock时，
是否应该获取到锁呢？应当。因为锁是以线程做管理对象的，  
当你第二次上锁，如果是因为当前线程已经上锁，而直接将本线程设为等待的话，  
便会造成死锁问题，因为第二次上锁会永远等待第一次上锁所对应的解锁，
而第一次上锁的解锁不会执行到。  
所以，当非公平锁的compareAndSetState(0, 1)失败时，并不会直接去加等待队列，  
而是在acquire中调用了nonfairTryAcquire，判断如果独占线程为当前线程时，  
state+1，所以，当我们解锁时，次数也要和加锁时相一致，不然就要死锁。  

#### 这些等待队列中的线程在做什么？在自旋吗？
好问题。
 

#### 为什么AQS持有队列头尾两个节点，一个不行吗？
好问题，我们来看一下入等待队列的逻辑：

## 三、加入等待队列

分了几步：
####1。addWaiter  创建节点
创建一个节点，并将其以CAS的方式加入等待队列的尾部，
之前的尾部成为上一个节点，并返回这个节点。
注意，这个时候节点的waitStatus应当为
   
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

####2。acquireQueued 处理队列状态

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


#### 3。acquire 对以上方法的调用
   
       public final void acquire(int arg) {
           if (!tryAcquire(arg) &&
               acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
               // acquireQueued 如果返回中断状态的话， 
               // 这里会补中断操作
               selfInterrupt();
       }


acquireQueued 这个方法的返回值如果为true，则会通知当前线程中断

#为什么运行时不相应中断
解锁时中断


#####当等待队列太长，是否会有性能问题呢？
问题不大，链表结构。除非压缩部分


#会不会唤醒的同时，被唤醒线程又把自己wait了
被唤醒永远是2号节点，但是二号节点不参与
见

##### 自旋？

#### 独占，共享 实现




到底什么是公平锁和非公平锁呢？

问得好，我们往下看：

## 公平锁和非公平锁
先加锁/先入队
尝试之前会利用 hasQueuedPredecessors() 方法来判断 AQS 的队列中中是否有其他线程，如果有则不会尝试获取锁(这是公平锁特有的情况)。

## .tryLock()


## .unlock()


## 什么是 ConditionObject


    

     

 
 3.等待队列的元素如何被唤醒？