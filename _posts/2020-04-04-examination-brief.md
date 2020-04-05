---
layout: default
title: "JAVA期末复习大纲"
tags: document
---
#  JAVA期末复习大纲 v1     


javap -v xxxxx
monitorenter 互斥入口
exit异常出口
ACC_SYNCHRONIZED
减少用户线程-》核心线程 耗时处理-》自旋锁，竞争失败马上转换，二十执行几次空循环
重量级锁 内核态
long和double的非原子性协定
如果在本线程内部观察，所有操作都是有序的
JOL看对象头 mark word(64bit 8byte) + class point

延迟偏向
偏向锁延迟jvm参数
释放锁后头不一定改回来，偏向锁五秒后不上锁也是偏向，sync之后 退出也是偏向
轻量锁退出sync置为无锁


偏向出现锁竞争
偏向于某个线程，放入对象头，0000的话没有偏心任何线程（可偏向）

轻量自旋时间内获取锁失败 10ms？-》升级
wait -》升级
偏向锁无法重新偏向，之后都是轻量级
批量撤销：
正常应该是：偏向-》撤销-cas-》偏向
偏向升级轻量CAS加锁: 拿对象头markword和预期值无锁状态markword，写入线程id，失败则膨胀为轻量级
可偏向则使用偏向锁
old new
交替执行没有轻量锁；
重量级才会调用os函数


但是epoch（每次撤销+1）>20 之后，就不撤销了，而是再偏向/20之前还是无锁+不可偏向
epoch的增加是找class的所有正在同步块的对象的最新值（cas）
1、批量重偏向和批量撤销是针对类的优化，和对象无关。
2、偏向锁重偏向一次之后不可再次重偏向。
3、当某个类已经触发批量撤销机制后，JVM会默认当前类产生了严重的问题，剥夺了该类的新实例对象使用偏向锁的权利



callable runable 区别。runnable无返回值，不抛出异常

futureTask(new callable)
Future = service.submit(new callable)
forkjoin
workstealingpool

notify notifyAll
wait 队列->锁队列

lock:aqs

semaphore acquire tryacquire

自旋 sleep ，yield ，park unpark
为什么aqs用链表存储
aqs cas改c=1，否则new node虚拟空节点，并一直存在
公平锁加锁过程：
try 设置state=1
获取队列 判断队头=队尾
加锁-失败-入队-判断上个节点在park，自己park
不是park（doing）--判断自己在不在头部-则竞争-
公平锁看有没有人排队，有人直接入q，非公平尝试去获取锁，不行入q
下一个来了，如果上一个0，改成-1，然后自己睡
目的：
#公平锁-进来先看自己是否是队首（2）




yield wait
yield不需要唤醒，线程变为就绪，不释放锁， Thread
wait需要notify唤醒，线程变为block，释放锁，object；
notify后不会立即执行
aqs头为空因为已经加到锁了
tcc 2pc
 
reenterlock
读锁   写锁
map存了信息（一个count重入次数，一个线程id）
lock加锁解锁次数不一致死锁
JMM 屏蔽os差异；保证可见性，不重排序；
mesi 保证可见性
volatile 防止指令重排，并非可见性
happen-before：是 jmm和hb对dev的承诺;volatile属性不都加；
包含 lock；hb传递性；这玩意和原子性没卵关系；

编译优化，级别乱序：
指令优化，内存级别乱序执行是因为不同cpu拿到不同指令；

DCL:

netty->future task
线程之间通信:
1. 共享内存-》锁
  通过内存
2.actor模型 akka
 通过消息 rxjava springflux

同步 异步
调用方式 异步回调

阻塞 非阻塞
开不开线程

runnbale
bitmap 查找和排序

buffer = byte[]
java io 面向流；要么输入要么输出；
发byte出去不会回来了

但是nio面向块；发buffer（堆外，操作系统内存；堆内，底层还是byte数组）可以双向通信；
buffer有capacity，position（下一个要写或者读的位置），limit（标记位置的结尾）
flip翻转（可以理解为调整状态）



clear也不清数据，只是改变位置
flip（）就是postion的位置给limit，然后自己=0

对象是哪个，就是从哪里读（channel.read（bytebuffer）就是从channel里面读到bytebuffer里）
.write就是从buffer读写入channel
）
buffer不clear会引起问题：源于write方法会移动指针，造成limit==position 而只有limit==capacity才能退出；
顺序
buffer.slice（）复制一个共享的buffer；有啥用？

buffer.allocateDirect 堆外内存 有个address引用堆外内存；性能好点，因为少一步copy；
allocate 堆内内存 底层byte[] jvm内；也需要把内存copy到操作系统；
warp（byte）堆方式创建buffer：注意！byte是引用！

selector：轮训的作用；




* CountDownLatch 是一次性的，CyclicBarrier 是可循环利用的
* CountDownLatch 参与的线程的职责是不一样的，有的在倒计时，有的在等待倒计时结束。CyclicBarrier 参与的线程职责是一样的



