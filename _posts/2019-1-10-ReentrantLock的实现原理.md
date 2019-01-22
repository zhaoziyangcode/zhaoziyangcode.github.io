---
layout:     post                    # 使用的布局（不需要改）
title:      ReentrantLock的实现原理（一）               # 标题 
subtitle:   公平锁的实现原理 #副标题
date:       2019-1-10              # 时间
author:     ziye                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - JAVA
    - 多线程
---
### ReentrantLock锁的特性
- 使用synchronized来作同步处理，加锁和释放锁都是隐式的，都是底层通过不同的机器指令实现的
- ReentrantLock是一个普通的类，基于AQS(AbstractQueuedSynchroizer)来实现加锁，释放锁，中断锁等操作
- ReentrantLock与Synchronized的性能差异：Synchronized在JDK1.6之前性能较ReentrantLock差一些，在JDK1.6之后Synchronized的性能与ReentrantLock相差并不大；ReentrantLock并不是一种替代内置加锁的方式，而是一种更加高级的选择；与Synchronized相比，ReentrantLock实现的功能更加丰富，具有，可重入，可中断，可限时，公平锁等特点。
- 可重入：ReentrantLock有一个与获取锁相关的计数器，当有锁的线程再次获取锁的时候，这个计数器就加一，finally就需要释放两次才能真正释放锁
- 可中断：ReentrantLock对中断是有相应的，lockInterruptibly方法来中断锁
- 可限时：通过lock.tryLock(long timeout, TimeUnit unit)来实现限时锁，超时无法获取锁就返回false，不会永久等待造成死锁。

```
  public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
```
### 锁的类型 
- 公平锁
- 非公平锁
##### ReentrantLock默认使用非公平锁，非公平锁是每个等待的线程都会去尝试获取锁，而不用去考虑线程等待的先后顺序，公平锁是先到先得，先等的线程先获取锁，后来的线程后获取锁，非公平锁在效率和吞吐量上面都优于公平锁，而公平锁会造成线程饥饿（有可能会一直等待）公平锁需要关心队列的情况，得按照队列里的先后顺序来获取锁(会造成大量的线程上下文切换)，而非公平锁则没有这个限制，Synchronized就是一种非公平锁。
- 获取锁：

```
    //默认非公平锁
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    //公平锁
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```
- 先看获取锁的过程

```
     public void lock() {
        sync.lock();
    }
```
##### 而sync.lock()是一个抽象的方法

```
 abstract void lock();
```
##### 具体由其子类实现 而实现方式 分为FairSync 和 NoFairSync两种实现分别对应公平锁和非公平锁
####公平锁 FairSync

```
        final void lock() {
            acquire(1);
        }
        //AbstractQueuedSynchronizer
        public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
        }
        //tryAcquire 也是由其子类FairSync实现的  FairSync extends Sync， Sync extends AbstractQueuedSynchronizer
        protected final boolean tryAcquire(int acquires) {
            //获取当前线程
            final Thread current = Thread.currentThread();
            //获取AQS 里的state 判断是否=0 若是 =0 则说明当前队列没有其他线程
            int c = getState();
            if (c == 0) {
                //判断当前队列是否有线程 如果有则不会获取锁 这是 公平锁特有的情况
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //判断当前锁是不是可重入锁
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
- 如果使用tryAcquire(arg)获取锁失败则 acquireQueued(addWaiter(Node.EXCLUSIVE), arg) 尝试使用 addWaiter方法将线程写入队列

```
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        //首先判断队列是否是空
        if (pred != null) {
            node.prev = pred;
            //利用CAS写入队列
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        
        enq(node);
        return node;
    }
```
- enq 相当于使用自旋 + CAS保证一定写入队列

```
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
- 写入队列之后会根据compareAndSetTail挂起线程

```
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //获取上一个节点
                final Node p = node.predecessor();
                //判断上一个节点是否是头部节点，是就尝试加锁
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //如果获取锁失败或者上一个节点不是头部节点则会根据上一个节点的状态来进行处理
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
- 获取上一个节点，判断上一个节点是否是头部节点，是就尝试加锁，如果获取锁失败或者上一个节点不是头部节点则会根据上一个节点的状态来进行处理shouldParkAfterFailedAcquire 状态如取消，等待等

```
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
        //节点等待
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
             
            return true;
        if (ws > 0) {
        //节点取消
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
- 如果当前线程需要挂起则调用parkAndCheckInterrupt方法挂起线程 是调用LockSuppot.park()

```
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
#### 非公平锁的获取
- 公平锁与非公平锁的差异主要在获取锁：公平锁就相当于买票，后来的人需要排到队尾依次买票，不能插队。
 而非公平锁则没有这些规则，是抢占模式，每来一个人不会去管队列如何，直接尝试获取锁。

```
        final void lock() {
            //直接尝试获取锁
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
```
- 还有一个区别就是非公平锁直接尝试获取锁，不需要判断队列中是否还有线程在等待

```
 final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //没有 !hasQueuedPredecessors() 判断
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

##### 锁的释放

```
    public void unlock() {
        sync.release(1);
    }
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
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
```

- 首先会判断当前线程是否为获得锁的线程，由于是重入锁所以需要将 state 减到 0 才认为完全释放锁。释放之后需要调用 unparkSuccessor(h) 来唤醒被挂起的线程。






















