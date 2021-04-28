[toc]
## AQS原理 ##
- **volatile** int变量 **state** 标识共享资源。
- 如果资源空闲，当前线程锁定资源。其中**tryAcquire**由子类实现。
- 如果资源被占，在等待队列中阻塞至唤醒。<br>![210329.aqs.png](https://img-blog.csdnimg.cn/20210406003513967.png)
```java
	//获取资源
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
	//释放资源
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

### acquire ###
- 尝试获取，**子类中实现**。
```java
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }	
```

- 入队列。cas入队列尾，失败自旋。
```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
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
```

- 队列中自旋，直到获取锁或[park阻塞](https://blog.csdn.net/weixin_39687783/article/details/85058686)。
  - 中断恢复后需要清除中断标记，否则[线程无法park](https://cgiirw.github.io/2018/05/27/Interrupt_Ques/)。 
```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
				//前置为虚拟头结点尝试获取锁
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
				//获取锁失败：压缩取消节点，如果前置节点为唤醒状态，则阻塞
				//如果被interrupt唤醒，此时中断标记为true，后续需要再次设置中断标记
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
				//node状态设为CANCELLED
                cancelAcquire(node);
        }
    }
    private final boolean parkAndCheckInterrupt() {
		//阻塞线程
        LockSupport.park(this);
		//返回线程中断标记并清空，否则后续无法再park
        return Thread.interrupted();
    }
```

- 设置中断标记。
  - 如果被其他线程 [interrupt](https://blog.csdn.net/canot/article/details/51087772) 唤醒过，而不是unpark唤醒,会擦除中断标记，需要再设置标记一次。
```java
    static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
```

### release ###
- 尝试释放。子类实现。
```java
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }	
```

- 唤醒队首阻塞的线程。
```java
    private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

### 线程中断 ###
- [park,interrupt,sleep的伪代码](https://blog.csdn.net/anlian523/article/details/106752414)
- [park,sleep,wait对比](https://juejin.cn/post/6844903984197533704)。

## ReentrantLock ##
- 内部类Sync继承AbstractQueuedSynchronizer，分为公平锁FairSync和非公平锁NonfairSync两种实现。
- 默认非公平锁。
- [从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)。

### lock ###
> 非公平锁为例
- 尝试抢占。
```java
    final void lock() {
		//抢占
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
			//aqs的aquire
            acquire(1);
    }
```

- 尝试获取。cas设置state，如果重入state++。
```java
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
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

### unlock ###
- 调sync释放。
```java
    public void unlock() {
        sync.release(1);
    }
```

- 尝试释放。减小state，至0释放锁。
```java
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

## ThreadPoolExecutor ##
- [内部类Worker继承AQS](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)，标识线程状态修改的权利。
```java
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        //封装的线程
        final Thread thread;
        //提交的业务逻辑
        Runnable firstTask;

        public void lock()        { acquire(1); }
        public void unlock()      { release(1); }
    }
```
- 执行业务逻辑的过程中持有锁。
```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
				//获取任务成功，加锁
                w.lock();
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
					//执行完成后解锁
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

- [回收空闲线程](https://www.jianshu.com/p/489d4015b8d2)，如线程池shutdown时，需要先获取锁，确保不影响正常业务。
```java
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(SHUTDOWN);
			//回收闲置线程
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }
    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
				//线程未中断且能获取锁
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
```