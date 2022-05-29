

@[toc]
## AQS原理 ##
### 核心数据结构
![210329.aqs.png](https://img-blog.csdnimg.cn/20210406003513967.png)
- **volatile** int变量 **state** 标识共享资源。
- 如果资源空闲，当前线程锁定资源。
- 如果资源被占，在等待队列中阻塞至唤醒。
- 源码定义中，线程抽象为Node，CLH队列为链表。
```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    private transient volatile Node head; //CLH队列头节点
    private volatile int state;
}

static final class Node {
    /** 共享 */
    static final Node SHARED = new Node();

    /** 独占 */
    static final Node EXCLUSIVE = null;

    /**
     * 因为超时或者中断，节点会被设置为取消状态，被取消的节点时不会参与到竞争中的，他会一直保持取消状态不会转变为其他状态；
     */
    static final int CANCELLED =  1;

    /**
     * 后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，使后继节点的线程得以运行
     */
    static final int SIGNAL    = -1;

    /**
     * 节点在等待队列中，节点线程等待在Condition上，当其他线程对Condition调用了signal()后，改节点将会从等待队列中转移到同步队列中，加入到同步状态的获取中
     */
    static final int CONDITION = -2;

    /**
     * 表示下一次共享式同步状态获取将会无条件地传播下去
     */
    static final int PROPAGATE = -3;

    /** 等待状态 */
    volatile int waitStatus;

    /** 前驱节点 */
    volatile Node prev;

    /** 后继节点 */
    volatile Node next;

    /** 获取同步状态的线程 */
    volatile Thread thread;

    Node nextWaiter;
}
```

### 独占式 ###
> 同一时刻仅有**一个线程持有同步状态**。

#### 同步获取：acquire
- 入口：acquire(int arg)
```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
	//同步获取资源
    public final void acquire(int arg) {
		// tryAcquire尝试获取
		// 获取同步状态失败时，addWaiter加入同步队列尾部，
		// 阻塞直到获取锁
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
			//线程被中断过，由于标记在底层被清除，selfInterrupt重新设置中断标记
            selfInterrupt();
    }
}
```

- tryAcquire：子类中实现，父类默认不支持。
```java
   protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
   }	
```

- addWaiter：入队列。依赖unsafe的cas。
```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
			//自旋设置队列尾的节点为新增的节点
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
		//如果队列为空，尝试使用当前节点初始化队列
        enq(node);
        return node;
    }
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
				//队列为空，CAS初始化队列头尾
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
				//队列非空，cas加入队尾
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
    private final boolean compareAndSetTail(Node expect, Node update) {
        return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
    }
    private final boolean compareAndSetHead(Node update) {
        return unsafe.compareAndSwapObject(this, headOffset, null, update);
    }
```

- acquireQueued：队列中自旋，直到获取锁或[park等待](https://blog.csdn.net/weixin_39687783/article/details/85058686)。
  - 中断恢复后需要清除中断标记，否则[线程无法park](https://cgiirw.github.io/2018/05/27/Interrupt_Ques/)。 
```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
				//前置为虚拟头结点尝试获取状态
                if (p == head && tryAcquire(arg)) {
					//获取状态成功，设置为头节点
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
				//获取状态失败，只有前置节点是signal才会park
				//park，并在唤醒之后返回中断标记
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
					//如果被interrupt唤醒，此时中断标记为true，消除之后需要再次设置中断标记
                    interrupted = true;
            }
        } finally {
            if (failed)
				//异常退出，当前节点状态设为CANCELLED
                cancelAcquire(node);
        }
    }
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
				//压缩取消状态的前置节点
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
			//前置节点非singal和cancelled，将前置节点设置为signal，依赖外层重试
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
    private final boolean parkAndCheckInterrupt() {
		//挂起线程
        LockSupport.park(this);
		//返回线程中断标记并清空，否则后续无法再park
        return Thread.interrupted();
    }
```

- selfInterrupt：设置中断标记。
  - 如果被其他线程 [interrupt](https://blog.csdn.net/canot/article/details/51087772) 唤醒过，而不是unpark唤醒,会擦除中断标记，需要再设置标记一次。
```java
    static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
```

#### 同步中断获取：acquireInterruptibly
- 线程中断后，aquire方法中的线程仍然位于同步队列，不对中断响应。
- acquireInterruptibly如果在获取同步状态时，线程中断，直接异常。
```java
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
		//类似acquire中加入队列和队列中自旋，不同之处在于如果线程中断会抛异常
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }
   private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
		//加入队列
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
					//如果线程中断过、直接抛异常
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

```

#### 同步超时获取：tryAcquireNanos
- 对同步响应中断获取+超时控制
```java
    public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
		//类似acquire，获取状态失败后，加入队列和队列中自旋，不同之处在于如果线程中断会抛异常，超时返回false
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }
    private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
		//入队列
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L)
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
					//parkNanos完成超时控制
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
					//线程中断抛异常
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

#### 释放：release ####
- 入口：release
```java
    public final boolean release(int arg) {
		//尝试释放。子类实现。
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
				//唤醒后续节点
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
	private void unparkSuccessor(Node node) {
	    int ws = node.waitStatus;
	    if (ws < 0)
			//状态设置为0，准备释放状态
	        compareAndSetWaitStatus(node, ws, 0);
	    Node s = node.next;
		//后续节点不可用
	    if (s == null || s.waitStatus > 0) {
	        s = null;
			//从tail找可用节点，因为此时s.next仍然可能不可用
	        for (Node t = tail; t != null && t != node; t = t.prev)
	            if (t.waitStatus <= 0)
	                s = t;
	    }
	    if (s != null)
			//唤醒目标节点
	        LockSupport.unpark(s.thread);
	}	
```

### 共享式 ###
#### 同步获取：acquireShared
- tryAcquireShared的返回值>=0，表示获取状态成功，其他类似独占式
```java
    public final void acquireShared(int arg) {
		//tryAcquireShared子类实现
        if (tryAcquireShared(arg) < 0)
			//获取失败，入队列自旋
            doAcquireShared(arg);
    }
    private void doAcquireShared(int arg) {
		//入队列
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
					//子类实现获取状态
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
						//当前节点设置为头节点
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
				//park判定
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
- 

#### 同步中断获取：acquireSharedInterruptibly
- 同步获取状态+感知中断
```java
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

#### 同步超时获取：tryAcquireSharedNanos
- 同步获取状态+感知中断+超时限制
```java
    public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquireShared(arg) >= 0 ||
            doAcquireSharedNanos(arg, nanosTimeout);
    }

```

#### 释放：releaseShared ####
```java
    public final boolean releaseShared(int arg) {
		//子类实现尝试释放
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
    private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
					//cas设置头节点状态为0，准备释放状态
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
					//唤醒后续节点
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

## 线程中断原理 ###
- native实现中两个变量标识线程中断相关的状态：
  - 中断状态：boolean，初始false
  - 免挂起许可：int(0,1)，初始0
- [park,interrupt,sleep的伪代码](https://blog.csdn.net/anlian523/article/details/106752414)
  - park：许可=1或中断状态=true，不可挂起。挂起恢复后消耗许可。
  - unpark：线程挂起才唤醒，增加许可。
  - interrupt：中断状态改为true，unpark。
  - sleep：挂起前后，如果中断状态=true，改成false后抛异常。
- [park,sleep,wait对比](https://juejin.cn/post/6844903984197533704)。

## ReentrantLock ##
- 内部类Sync继承AQS。
- 分为公平锁FairSync和非公平锁NonfairSync两种实现。默认非公平锁。
![在这里插入图片描述](https://img-blog.csdnimg.cn/a3188ba2539642a0b43942b2a88756aa.png)

### 加锁：lock ###
- 非公平锁
```java
static final class NonfairSync extends Sync {

    final void lock() {
		//尝试抢占，抢占成功则获取锁，OWNER设置为当前线程
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
			//AQS的同步获取
            acquire(1);
    }
    protected final boolean tryAcquire(int acquires) {
		//重写AQS的tryAcquire
        return nonfairTryAcquire(acquires);
    }
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
        	//已经获取锁，cas设置状态
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
        	//重入，state自增
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}

public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
        protected final boolean compareAndSetState(int expect, int update) {
		// CAS设置状态
		return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
}
```

- 公平锁
```java
   static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
			//直接执行aqs的同步获取方法，相比非公平锁少了抢占
            acquire(1);
        }

        /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
				//相比非公平锁多了判断当前线程是否是同步队列的第一个节点
				//是第一个节点则获取状态
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
				//重入
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
    public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```

### 解锁：unlock ###
- 调sync释放。
```java
public class ReentrantLock implements Lock, java.io.Serializable {
    public void unlock() {
		//aqs的同步释放方法
        sync.release(1);
    }
}
```

## ThreadPoolExecutor ##、
- 维护线程集合是使用reentrantlock保证线程安全。
- 内部类Worker继承AQS，使用state标识线程的执行状态。
	- state:-1，新建的worker
	- state:0，等待获取任务的worker
	- state:1，正在执行任务的worker


### 提交任务：execute
```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    public void execute(Runnable command) {
        if (command == null)
        int c = ctl.get();
		//小于核心线程数，addWorker新增核心工作线程
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
		//大于等于核心线程数，加入阻塞队列
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
		//线程池非running或者加入队列失败，创建新非核心工作线程
        else if (!addWorker(command, false))
			//失败执行拒绝策略
            reject(command);
    }

    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
			//获取线程池当前状态
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

			//自旋增加线程数量
            for (;;) {
                int wc = workerCountOf(c);
				//判断当前线程数量是否超过限制
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
				//cas设置ctl
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
			//新建业务逻辑的封装类Worker
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
				//通过ReentrantLocl保证加入worker集合的操作线程安全
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
				//启动线程
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }

}

```

- Worker继承AQS
```java
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        public void run() {
			//执行线程
            runWorker(this);
        }
        //封装的线程
        final Thread thread;
        //提交的业务逻辑
        Runnable firstTask;

        protected boolean tryAcquire(int unused) {
			//尝试获取状态
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
			//释放状态
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
		
		//使用aqs的同步获取和释放
        public void lock()        { acquire(1); }
        public void unlock()      { release(1); }
    }

    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
		//准备获取任务，线程设置成可中断
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
			//获取待执行任务
            while (task != null || (task = getTask()) != null) {
				//获取任务成功，锁定aqs资源
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
   private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // 判断是否需要超时：核心线程可超时或者线程数多于核心池大小
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
				//如果超时，则减小线程数，返回空，外层结束循环
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
				//支持超时用poll+时限，无超时用take
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
				//获取任务超时
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

#### 任务终止：shutdown
- shutdown()：按过去执行已提交任务的顺序发起一个有序的关闭，不接受新任务。
	- 方法执行完成时可能业务线程还未结束。
- 中断线程的时候需要获取aqs的状态
```java
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
			//设置线程池的状态
            advanceRunState(SHUTDOWN);
			//中断闲置线程
            interruptIdleWorkers();
			//空方法，子类实现
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }

    private void interruptIdleWorkers() {
        interruptIdleWorkers(false);
    }

    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
				//线程未中断且能释放worker的状态，即线程空闲，不在处理业务逻辑
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

#### 任务终止：shutdownNow
- shutdownNow() :尝试停止所有的活动执行任务、暂停等待任务的处理，并返回等待执行的任务列表。
	- 方法执行完成后活动的线程会被中断，
```java
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
			//终止所有活动线程
            interruptWorkers();
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
    private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
				//强行中断
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }
        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
					//线程强行中断
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
```

## CountDownLatch ##
- 内部Sync类继承AQS，使用共享模式。主线程等待所有分支线程完成时，继续执行。

```java
 private static final class Sync extends AbstractQueuedSynchronizer {
        Sync(int count) {
			//初始化的时候设置非0的state
            setState(count);
        }
        protected int tryAcquireShared(int acquires) {
			//自定义获取方法，state为0获取成功
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
			//每次释放state--，到0释放成功
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
```

### 等待：await
- await获取共享资源，直到state为0的时候获取状态成功。
- 主线程使用
```java
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```

### 到数：countDown。
- countDown释放一个资源。
- 分支线程使用
```java
    public void countDown() {
        sync.releaseShared(1);
    }
```

## 参考资料
- [死磕java并发](https://www.cmsblogs.com/article/1391297829913366528)
- [从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)
- [Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)
- [满满干货的线程池](https://www.jianshu.com/p/489d4015b8d2)