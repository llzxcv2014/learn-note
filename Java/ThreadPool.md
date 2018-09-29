# ThreadPool（线程池）

## Java中的`ThreadPoolExecutor`

### 构造方法

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    .....
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);
 
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
        BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);
```

* `corePoolSize`：核心池的大小。在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了`prestartAllCoreThreads()`或者`prestartCoreThread()`方法，从这2个方法的名字就可以看出，是预创建线程的意思，即在没有任务到来之前就创建`corePoolSize`个线程或者一个线程。默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到`corePoolSize`后，就会把到达的任务放到缓存队列当中；

* `maximumPoolSize`： 线程池最大线程数，它表示在线程池中最多能创建多少个线程

* `keepAliveTime`：表示线程没有任务执行时最多保持多久时间会终止.**默认情况下，只有当线程池中的线程数大于`corePoolSize`时，`keepAliveTime`才会起作用，直到线程池中的线程数不大于`corePoolSize`：即当线程池中的线程数大于`corePoolSize`时，如果一个线程空闲的时间达到`keepAliveTime`，则会终止，直到线程池中的线程数不超过`corePoolSize`；但是如果调用了`allowCoreThreadTimeOut(boolean)`方法，在线程池中的线程数不大于`corePoolSize`时，`keepAliveTime`参数也会起作用，直到线程池中的线程数为0；**

* `unit`：参数`keepAliveTime`的时间单位`workQueue`：一个阻塞队列，用来存储等待执行的任务，**这个参数的选择会对线程池的运行过程产生重大影响**

* `threadFactory`：线程工厂，主要用来创建线程；

* `handler`：表示当拒绝处理任务时的策略，有以下四种取值：

        1. ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。 
        2. ThreadPoolExecutor.DiscardPolicy:丢弃任务，但是不抛出异常
        3. ThreadPoolExecutor.DiscardOldestPolicy:丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
        4. ThreadPoolExecutor.CallerRunsPolicy:由调用线程处理该任务

## 线程实现原理

### 1. 线程池状态

`runState`：当前的线程池的状态，用`volatile`保证可见性.

1. 线程池创建后,线程池处于`RUNNING`状态.
2. 若调用`shutdown()`方法，则线程池处于`SHUTDOWN`状态，此时线程池不能接受新的任务，会等待所有线程执行完毕。
3. 若调用`shutdownNow()`方法，则线程池处于`STOP`状态，此时，线程池不接受新的任务，并尝试终止正在执行的任务。
4. 线程池处于SHUTDOWN或STOP状态，并且所有工作线程已经销毁，任务缓存队列已经清空或执行结束后，线程池被设置为TERMINATED状态。

### 2.任务的执行

`ThreadPoolExecutor`中的重要的成员变量

```java
// 任务缓存队列，用来存放等待执行的任务
private final BlockingQueue<Runnable> workQueue;
// 线程池的主要状态锁，对线程池状态（比如线程池大小、runState等）
// 的改变都要使用这个锁
private final ReentrantLock mainLock = new ReentrantLock();
// 用来存放工作集
private final HashSet<Worker> workers = new HashSet<Worker>();  
// 线程存货时间
private volatile long  keepAliveTime;
// 是否允许为核心线程设置存活时间
private volatile boolean allowCoreThreadTimeOut;
//核心池的大小（即线程池中的线程数目大于这个参数时，
//提交的任务会被放进任务缓存队列）
private volatile int   corePoolSize;
//线程池最大能容忍的线程数
private volatile int   maximumPoolSize;
//线程池中当前的线程数
private volatile int   poolSize;
//任务拒绝策略
private volatile RejectedExecutionHandler handler;
//线程工厂，用来创建线程
private volatile ThreadFactory threadFactory;
//用来记录线程池中曾经出现过的最大线程数
private int largestPoolSize;
 //用来记录已经执行完毕的任务个数
private long completedTaskCount;  
```

1. 池中运行的线程小于`corePoolSize`时，便会尝试创建一个新的线程，即使其他线程是空闲的.
2. 若池中运行的线程小于`maximumPoolSize`时，并且阻塞队列是满的时候便会创建一个线程处理请求。

`largestPoolSize`:用来记录线程池中曾经有过的最大线程数目，跟线程池的容量没有任何关系

#### `execute()`方法

1. 1.6版本

```java
public void execute(Runnable command) {
    // 任务不为空
    if (command == null)
        throw new NullPointerException();
    // 判断是否大于corePoolSize
    if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command)) {
        // 判断当前线程池处于RUNNING状态,将任务放入缓存队列
        if (runState == RUNNING && workQueue.offer(command)) {
            // 如果当前线程不处于RUNNING状态
            // 防止在将此任务添加进任务缓存队列的同时其他线程突然调用
            // shutdown或者shutdownNow方法关闭了线程池的一种应急措施
            if (runState != RUNNING || poolSize == 0)
                // 保证添加到任务缓存队列中的任务得到处理
                ensureQueuedTaskHandled(command);
        }
        // 不处于RUNNING状态或者放入缓存队列失败
        else if (!addIfUnderMaximumPoolSize(command))
            reject(command); // is shutdown or saturated
    }
}
```

`addIfUnderCorePoolSize`方法

```java
private boolean addIfUnderCorePoolSize(Runnable firstTask) {
    Thread t = null;
    final ReentrantLock mainLock = this.mainLock;
    // 获取锁，因涉及到线程池状态的改变
    mainLock.lock();
    try {
        // 判断线程池大小是否小于corePoolSize并且线程池状态是RUNNING
        if (poolSize < corePoolSize && runState == RUNNING)
            // 创建线程去执行firstTask任务
            t = addThread(firstTask);
        } finally {
        mainLock.unlock();
    }
    if (t == null)
        return false;
    t.start();
    return true;
}
```

`addThread`方法

```java
private Thread addThread(Runnable firstTask) {
    Worker w = new Worker(firstTask);
    // 创建一个线程，执行任务
    Thread t = threadFactory.newThread(w);  
    if (t != null) {
        // 将创建的线程的引用赋值为w的成员变量
        w.thread = t;
        workers.add(w);
        // 当前线程数加1
        int nt = ++poolSize;
        if (nt > largestPoolSize)
            largestPoolSize = nt;
    }
    return t;
}
```

在`addThread`方法中，首先用提交的任务创建了一个`Worker`对象，然后调用线程工厂`threadFactory`创建了一个新的线程`t`，然后将线程`t`的引用赋值给了`Worker`对象的成员变量`thread`，接着通过`workers.add(w)`将`Worker`对象添加到工作集当中.

2. 1.7版本

```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```

#### `Worker`

* 1.6及之前

```java
private final class Worker implements Runnable {
    private final ReentrantLock runLock = new ReentrantLock();
    private Runnable firstTask;
    volatile long completedTasks;
    Thread thread;
    Worker(Runnable firstTask) {
        this.firstTask = firstTask;
    }
    boolean isActive() {
        return runLock.isLocked();
    }
    void interruptIfIdle() {
        final ReentrantLock runLock = this.runLock;
        if (runLock.tryLock()) {
            try {
        if (thread != Thread.currentThread())
        thread.interrupt();
            } finally {
                runLock.unlock();
            }
        }
    }
    void interruptNow() {
        thread.interrupt();
    }
    private void runTask(Runnable task) {
        final ReentrantLock runLock = this.runLock;
        runLock.lock();
        try {
            if (runState < STOP &&
                Thread.interrupted() &&
                runState >= STOP)
            boolean ran = false;
             // beforeExecute方法是ThreadPoolExecutor类的一个方法，
             // 没有具体实现，用户可以根据
             // 自己需要重载这个方法和后面的afterExecute方法来进行一些统计信息，
             // 比如某个任务的执行时间等
            beforeExecute(thread, task);
            try {
                task.run();
                ran = true;
                afterExecute(task, null);
                ++completedTasks;
            } catch (RuntimeException ex) {
                if (!ran)
                    afterExecute(task, ex);
                throw ex;
            }
        } finally {
            runLock.unlock();
        }
    }

    public void run() {
        try {
            // 获取第一个任我游
            Runnable task = firstTask;
            firstTask = null;
            // 当队列中有任务时获取任务执行
            while (task != null || (task = getTask()) != null) {
                runTask(task);
                task = null;
            }
        } finally {
            // 当任务队列中没有任务时，进行清理工作
            workerDone(this);  
        }
    }
 }
```

`runTask`方法是在`ThreadPoolExecutor`中

```java
Runnable getTask() {
    for (;;) {
        try {
            int state = runState;
            if (state > SHUTDOWN)
                return null;
            Runnable r;
            // Help drain queue
            if (state == SHUTDOWN)  
                r = workQueue.poll();
            //如果线程数大于核心池大小或者允许为核心池线程设置空闲时间，
            //则通过poll取任务，若等待一定的时间取不到任务，则返回null
            else if (poolSize > corePoolSize || allowCoreThreadTimeOut)
                r = workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS);
            else
                r = workQueue.take();
            if (r != null)
                return r;
            // 如果没取到任务，即r为null，则判断当前的worker是否可以退出
            if (workerCanExit()) {
                // Wake up others
                if (runState >= SHUTDOWN)
                    // 中断处于空闲状态的worker
                    interruptIdleWorkers();
                return null;
            }
            // Else retry
        } catch (InterruptedException ie) {
            // On interruption, re-check runState
        }
    }
}
```

1. 在`getTask`中，先判断当前线程池状态，如果`runState`大于`SHUTDOWN`（即为`STOP`或者`TERMINATED`），则直接返回`null`。

2. 如果`runState`为`SHUTDOWN`或者`RUNNING`，则从任务缓存队列取任务。

3. 如果当前线程池的线程数大于核心池大小`corePoolSize`或者允许为核心池中的线程设置空闲存活时间，则调用`poll(time,timeUnit)`来取任务，这个方法会等待一定的时间，如果取不到任务就返回`null`。

4. 然后判断取到的任务`r`是否为`null`，为`null`则通过调用`workerCanExit()`方法来判断当前`worker`是否可以退出

`workerCanExit()`方法

```java
private boolean workerCanExit() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    boolean canExit;
    //如果runState大于等于STOP，或者任务缓存队列为空了
    //或者允许为核心池线程设置空闲存活时间并且线程池中的线程数目大于1
    try {
        canExit = runState >= STOP ||
            workQueue.isEmpty() ||
            (allowCoreThreadTimeOut &&
             poolSize > Math.max(1, corePoolSize));
    } finally {
        mainLock.unlock();
    }
    return canExit;
}
```

`worker`退出的条件：

1. 线程池处于STOP状态
2. 任务队列已为空
3. 允许为核心池线程设置空闲存活时间并且线程数大于1

调用`interruptIdleWorkers()`中断处于空闲状态的`worker`

`interruptIdleWorkers`方法

```java
void interruptIdleWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 实际上调用的是worker的interruptIfIdle()方法
        for (Worker w : workers)  
            w.interruptIfIdle();
    } finally {
        mainLock.unlock();
    }
}
```

实际上调用的是`worker`的`interruptIfIdle()`方法

```java
void interruptIfIdle() {
    final ReentrantLock runLock = this.runLock;
    //注意这里，是调用tryLock()来获取锁的，
    // 因为如果当前worker正在执行任务，锁已经被获取了，是无法获取到锁的
    //如果成功获取了锁，说明当前worker处于空闲状态
    if (runLock.tryLock()) {
        try {
            if (thread != Thread.currentThread())
                thread.interrupt();
        } finally {
            runLock.unlock();
        }
    }
}
```

任务提交给线程池之后到被执行的过程

1. 如果当前线程池中的线程数目小于`corePoolSize`，则每来一个任务，就会创建一个线程去执行这个任务
2. 如果当前线程池中的线程数目 **>=** `corePoolSize`，则每来一个任务，会尝试将其添加到任务缓存队列当中，若添加成功，则该任务会等待空闲线程将其取出去执行；若添加失败（一般来说是任务缓存队列已满），则会尝试创建新的线程去执行这个任务
3. 如果当前线程池中的线程数目达到`maximumPoolSize`，则会采取任务拒绝策略进行处理
4. 如果线程池中的线程数量大于`corePoolSize`时，如果某线程空闲时间超过`keepAliveTime`，线程将被终止，直至线程池中的线程数目不大于`corePoolSize`；如果允许为核心池中的线程设置存活时间，那么核心池中的线程空闲时间超过`keepAliveTime`，线程也会被终止

* 1.7版本

```java
 private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable {
        private static final long serialVersionUID = 6138294804551838833L;
        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
        public void run() {
            runWorker(this);
        }
        protected boolean isHeldExclusively() {
            return getState() != 0;
        }
        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }
        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```

### 3. 线程池中线程的初始化

默认情况下，创建线程池之后，线程池中是没有线程的，需要提交任务之后才会创建线程

在实际中如果需要线程池创建之后立即创建线程:
1. `prestartCoreThread()`：初始化一个核心线程
2. `prestartAllCoreThreads()`：初始化所有核心线程

```java
public boolean prestartCoreThread() {
    // 注意传进去的参数是null
    return addIfUnderCorePoolSize(null); 
}

public int prestartAllCoreThreads() {
    int n = 0;
    // 注意传进去的参数是null
    while (addIfUnderCorePoolSize(null))
        ++n;
    return n;
}
```

执行线程会阻塞在getTask方法中的`r = workQueue.take();`，等待任务队列中有任务.

### 4. 任务缓存队列及排队策略(`workQueue`)

`workQueue`的类型为`BlockingQueue<Runnable>`，通常可以取下面三种类型：

1. `ArrayBlockingQueue`：基于数组的先进先出队列，此队列创建时必须指定大小
2. `LinkedBlockingQueue`：基于链表的先进先出队列，如果创建时没有指定此队列大小，则默认为`Integer.MAX_VALUE`；
3. `SynchronousQueue`：这个队列比较特殊，它不会保存提交的任务，而是将直接新建一个线程来执行新来的任务

### 5. 任务拒绝策略

当线程池的任务缓存队列已满并且线程池中的线程数目达到`maximumPoolSize`，如果还有任务到来就会采取任务拒绝策略，通常有以下四种策略：

1. `ThreadPoolExecutor.AbortPolicy`:丢弃任务并抛出RejectedExecutionException异常
2. `ThreadPoolExecutor.DiscardPolicy`:丢弃任务，但是不抛出异常
3. `ThreadPoolExecutor.DiscardOldestPolicy`:丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
4. `ThreadPoolExecutor.CallerRunsPolicy`:由调用线程处理该任务

### 6. 线程池的关闭

1. `shutdown()`：不会立即终止线程池，而是要等所有任务缓存队列中的任务都执行完后才终止，但再也不会接受新的任务
2. `shutdownNow()`：立即终止线程池，并尝试打断正在执行的任务，并且清空任务缓存队列，返回尚未执行的任务

### 7. 线程池容量的动态调整

* `setCorePoolSize`：设置核心池大小
* `setMaximumPoolSize`：设置线程池最大能创建的线程数目大小

## 示例

```java
public static void main(String[] args) {
         ThreadPoolExecutor executor = new ThreadPoolExecutor(5, 10, 200, TimeUnit.MILLISECONDS,
                 new ArrayBlockingQueue<Runnable>(5));
         for (int i = 0; i < 15; i++) {
             MyTask myTask = new MyTask(i);
             executor.execute(myTask);
             System.out.println("线程池中线程数目：" + executor.getPoolSize()+"，队列中等待执行的任务数目："+
             executor.getQueue().size()+"，已执行玩别的任务数目：" + executor.getCompletedTaskCount());
         }
         executor.shutdown();
}

class MyTask implements Runnable {
    private int taskNum;
    public MyTask(int num) {
        this.taskNum = num;
    }

    @Override
    public void run() {
        System.out.println("正在执行task "+taskNum);
        try {
            Thread.currentThread().sleep(4000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("task " + taskNum + "执行完毕");
    }

```

#### 使用`Executors`中提供的方法

1. `Executors.newCachedThreadPool()`创建一个缓冲池，缓冲池容量大小为`Integer.MAX_VALUE`
2. `Executors.newSingleThreadExecutor()`创建容量为1的缓冲池
3. `Executors.newFixedThreadPool(int)`创建固定容量大小的缓冲池

### 合理配置线程池大小

* CPU密集型任务，就需要尽量压榨CPU，参考值可以设为 n + 1
* IO密集型任务，参考值可以设置为 2 * n

### 1.7的改变

#### `ctl`

它记录了当前线程池的运行状态和线程池内的线程数；一个变量是怎么记录两个值的呢？它是一个AtomicInteger 类型，有32个字节，这个32个字节中，高3位用来标识线程池的运行状态，低29位用来标识线程池内当前存在的线程数；

Jdk1.7开始使用`ctl(main pool control state)`是一个`atomic integer`包装了两个字段

1. `WorkerCount`:显示有效的线程数
2. `runState`:反应线程池是运行还是关闭

为了将他们包装到一个整型,将`workerCount`限制在 $2^{29}-1$ 大约5亿个线程而不是 $2^{31}-1$.这个参数以后可以扩展为`AtomicLong`.但int型更快和简单一点。

#### `runState`的状态：

1. `RUNNING`：接受新的任务，运行队列里的任务
2. `SHUTDOWN`：不接受新的任务，运行队列里的任务
3. `STOP`：不接受新的任务，不运行队列里的任务，中断正在执行的任务
4. `TIDYING`：所有的任务都结束，`workCount`为0，线程转为`TIDYING`状态会运行`terminated()`钩子方法。
5. `TERMINATED`：`terminated()`方法运行完成

#### 状态间的转换：

* `RUNNING -> SHUTDOWN`：调用`shutdown()`
* `(RUNNING or SHUTDOWN) -> STOP`：调用`shutdownNow()`
* `SHUTDOWN -> TIDYING`：阻塞队列和池都为空
* `STOP -> TIDYING`：池为空
* `TIDYING -> TERMINATED`：`terminated()`钩子方法执行完成。

#### 与ctl相关的三个方法

```java
//获取线程池的状态,也就是将ctl低29位都置为0后的值
private static int runStateOf(int c)    { return c & ~CAPACITY; }
//获取线程池中线程数，也就是ctl低29位的值
private static int workerCountOf(int c)  { return c & CAPACITY; }  
//设置ctl的值，rs为线程池状态，wc为线程数；
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

### 资料

[Java线程池（ThreadPool）详解](https://www.cnblogs.com/kuoAT/p/6714762.html)

[java程序员必精--从源码讲解java线程池ThreadPoolExecuter的实现原理、各种坑、如何监控](https://blog.csdn.net/zqz_zqz/article/details/69488570?locationNum=12&fps=1)