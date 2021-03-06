### 为什么要用线程池
1. 降低资源消耗。通过重复利用已创建的线程降低线程创建、销毁线程造成的消耗。
2. 提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。
3. 提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配、调优和监控

### 线程池的底层类与接口
- ExecutorService 是真正的线程池接口
- Executor 是线程池的顶级接口，只是一个执行线程的工具，只提供一个execute(Runnable command)的方法，真正的线程池接口是ExecutorService
- Executors 是静态工厂类，生产各种类型线程池
- AbstractExecutorService 实现了ExecutorService接口，实现了其中大部分的方法（有没有实现的方法，所以被声明为Abstract）
- ThreadPoolExecutor，继承了AbstractExecutorService，是ExecutorService的默认实现

### 线程池种类
• CachedThreadPool：一个任务创建一个线程；  
• FixedThreadPool：所有任务只能使用固定大小的线程；  
• SingleThreadExecutor：相当于大小为 1 的 FixedThreadPool。  

本质上都是ThreadPoolExecutor, 由Executors下的工厂方法构造：  
○ newFixedThreadPool：返回固定长度的线程池，线程池中的线程数量是固定的。  
○ newCacheThreadPool：该方法返回一个根据实际情况来进行调整线程数量的线程池，空余线程存活时间是60s  
○ newSingleThreadExecutor：该方法返回一个只有一个线程的线程池。  
○ newSingleThreadScheduledExecutor：该方法返回一个SchemeExecutorService对象，线程池大小为1，SchemeExecutorService接口在 ThreadPoolExecutor类和 ExecutorService接口之上的扩展，在给定时间执行某任务。  
○ newSchemeThreadPool：该方法返回一个SchemeExecutorService对象，可指定线程池线程数量。  
	
## ThreadPoolExecutor源码解析
### ThreadPoolExecutor线程池类参数详解
参数	说明
- corePoolSize	核心线程数量，线程池维护线程的最少数量
- maximumPoolSize	线程池维护线程的最大数量
- keepAliveTime	线程池除核心线程外的其他线程的最长空闲时间，超过该时间的空闲线程会被销毁
- unit	keepAliveTime的单位，TimeUnit中的几个静态属性：NANOSECONDS、MICROSECONDS、MILLISECONDS、SECONDS
- workQueue	线程池所使用的任务缓冲队列
- threadFactory	线程工厂，用于创建线程，一般用默认的即可
- handler	线程池对拒绝任务的处理策略

### 详细解释
- 核心线程（corePool）：
  有新任务提交时，首先检查核心线程数，如果核心线程都在工作，而且数量也已经达到最大核心线程数，则不会继续新建核心线程，而会将任务放入等待队列。
- 等待队列 (workQueue)：
  等待队列用于存储，当核心线程都在忙时，继续新增的任务，核心线程在执行完当前任务后，也会去等待队列拉取任务继续执行，这个队列一般是一个线程安全的阻塞队列，它的容量也可以由开发者根据业务来定制。
- 非核心线程：
  当等待队列满了，如果当前线程数没有超过最大线程数，则会新建线程执行任务，那么核心线程和非核心线程到底有什么区别呢？本质上它们没有什么区别，创建出来的线程也根本没有标识去区分它们是核心还是非核心的，线程池只会去判断已有的线程数（包括核心和非核心）去跟核心线程数和最大线程数比较，来决定下一步的策略。
- 线程活动保持时间 (keepAliveTime)：
  线程空闲下来之后，保持存货的持续时间，超过这个时间还没有任务执行，该工作线程结束。
- 饱和策略 (RejectedExecutionHandler)：
  当等待队列已满，线程数也达到最大线程数时，线程池会根据饱和策略来执行后续操作，默认的策略是抛弃要加入的任务。
	
		
### 成员变量ctl
AtomicInteger这个类可以通过CAS达到无锁并发，效率比较高，这个变量有双重身份，它的高三位表示线程池的状态，低29位表示线程池中现有的线程数，用最少的变量来减少锁竞争，提高并发效率。
```java
    //CAS，无锁并发
	        private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
	        //表示线程池线程数的bit数：29位
	        private static final int COUNT_BITS = Integer.SIZE - 3;
	        //最大的线程数量，数量是完全够用了
		//0001 1111 1111 1111 1111 1111 1111 1111
	        private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
	    
	        // runState is stored in the high-order bits
	        //1110 0000 0000 0000 0000 0000 0000 0000（很耿直的我）
	        private static final int RUNNING    = -1 << COUNT_BITS; //-1每一位都是1
	        //0000 0000 0000 0000 0000 0000 0000 0000（很耿直的我）
	        private static final int SHUTDOWN   =  0 << COUNT_BITS;
	        //0010 0000 0000 0000 0000 0000 0000 0000（很耿直的我）
	        private static final int STOP       =  1 << COUNT_BITS;
	        //0100 0000 0000 0000 0000 0000 0000 0000（很耿直的我）
	        private static final int TIDYING    =  2 << COUNT_BITS;
	        //0110 0000 0000 0000 0000 0000 0000 0000（很耿直的我）
	        private static final int TERMINATED =  3 << COUNT_BITS;
	    
	        // Packing and unpacking ctl
	        //获取线程池的状态
	        private static int runStateOf(int c)     { return c & ~CAPACITY; }
	        //获取线程的数量
	        private static int workerCountOf(int c)  { return c & CAPACITY; }
	        //组装状态和数量，成为ctl
	        private static int ctlOf(int rs, int wc) { return rs | wc; }
	        /*
	         * 判断状态c是否比s小，下面会给出状态流转图
	         */
	        private static boolean runStateLessThan(int c, int s) {
	            return c < s;
	        }
	        //判断状态c是否不小于状态s
	        private static boolean runStateAtLeast(int c, int s) {
	            return c >= s;
	        }
	        //判断线程是否在运行
	        private static boolean isRunning(int c) {
	            return c < SHUTDOWN;
	        }
```
### 线程池的状态
a. RUNNING, 运行状态，值也是最小的，刚创建的线程池就是此状态。  
b. SHUTDOWN，停工状态，不再接收新任务，已经接收的会继续执行  
c. STOP，停止状态，不再接收新任务，已经接收正在执行的，也会中断  
d. TYNYING/，所有任务都停止了，工作的线程也全部结束了  
e. TERMINATED，终止状态，线程池已销毁  
	它们的流转关系如下： 
  <img src="https://github.com/Qirui0805/Personal-Blog/blob/master/image/%E6%B5%81%E7%A8%8B.png" width="300">
### 任务执行流程
<img src="https://github.com/Qirui0805/Personal-Blog/blob/master/image/execute.png" width="600">     

### execute(Runnable command)
- 如果有效线程数低于``corePoolSize``, addWork()执行核心线程
- 否则，尝试加入阻塞队列，阻塞队列未满且double check线程池状态没问题则加入成功。只有当线程池里没有线程时才新建线程，否则等待现有的线程去队列里取任务
- 否则，尝试直接addworker作为非核心线程执行
```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task. 
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        /*
         * 2. If a task can be successfully queued, we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         */
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
	    //如果线程池里没有线程了，必须创建一个线程去执行任务
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        /*
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        else if (!addWorker(command, false))
            reject(command);
    }
```
### addWorker(Runnable firstTask, boolean core)
#### 判断可否执行
- 检查线程池状态
1. 线程池状态不能为(stop, tidying, terminated)
2. 线程池状态为shutdown时，传入的任务必须为空且workqueue不能为空（见shutdown的含义）

- 检查线程池线程数量
1. 如果以核心线程执行任务，``workCountOf(ctl)``必须小于``corepoolsize``
2. 如果以非核心线程执行任务，``workCountOf(ctl)``必须小于``maximumpoolsize``

- 尝试通过CAS操作改变有效线程数量
如果失败则重新回到第一步
```java
retry:
for (;;) {
    int c = ctl.get();
    int rs = runStateOf(c);

    // First
    if (rs >= SHUTDOWN &&
        ! (rs == SHUTDOWN &&
           firstTask == null &&
           ! workQueue.isEmpty()))
        return false;
    for (;;) {
        int wc = workerCountOf(c);
        //Second
        if (wc >= CAPACITY ||
            wc >= (core ? corePoolSize : maximumPoolSize))
            return false;
        //Third
        if (compareAndIncrementWorkerCount(c))
            break retry;
        c = ctl.get();  // Re-read ctl
        if (runStateOf(c) != rs)
            continue retry;
        // else CAS failed due to workerCount change; retry inner loop
    }
}
```

#### 执行任务
- 构建Worker对象
- 检查线程池状态， 
- t.start()
```java
boolean workerStarted = false;
boolean workerAdded = false;
Worker w = null;
try {
    w = new Worker(firstTask);
    final Thread t = w.thread;
    if (t != null) {
        final ReentrantLock mainLock = this.mainLock;
        //workers的数据结构为HashSet，非线程安全，操作workers需要加同步锁
        mainLock.lock();
        try {
            int rs = runStateOf(ctl.get());
	    //RUNNING状态自然不用说，如果时SHUTDOWN状态则要求传入的任务是null,因为该状态下不能再添加新的任务了，但可以新建一个线程去执行队列中的任务 
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
```
### Worker
Worker 实现Runnable接口，持有一个Thread对象，初始化时用Worker自身传给``thread``，因此在``addWorker``中``t.start()``最终跑的是Worker里的run。
```java
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}

/** Delegates main run loop to outer runWorker  */
public void run() {
    runWorker(this);
}
```
### runWorker(Worker w)
- 获取任务
- 执行任务
```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        //传入新任务或从workqueue中获取
        while (task != null || (task = getTask()) != null) {
	    //加锁为了防止其他线程调用``shutdown``方法中断正在运行的线程	
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
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}

```

### getTask()
负责获取任务，对于不同的情况有不同的获取策略
```java
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

        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }
        //如果是核心线程且允许超时（allowCoreThreadTimeOut)或者非核心线程，则超时获取task，否则阻塞获取
        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

### 线程池总结
#### 线程的创建时机
execute的三种情况
#### 线程储存在哪里
Worker对象与线程一一对应，Worker存在名为workers的HashSet中。
#### 线程池如何重复使用线程
线程在Worker初始化时被创建，并作为在Worker的field，Worker本身作为线程的初始化参数，创建成功后直接在addWorker中调用``t.start()``方法启动线程，然后运行Worker的``run()``方法, 这个方法调用``ThreadPoolExecutor``的``runWorker(Worker)``方法，这个方法会先运行``Worker``本身的task，然后去调用``getTask()``方法去``workQueue``取任务，线程的重复使用就体现在这个方法中，当前线程数小于核心线程数上限而且没有设置超时时机的话就用``take()``方法去阻塞获取任务，否则用``poll(keepAliveTime)``方法。说明线程池最多保持数量上限为``corePoolSize``的线程一直处于存活状态，其他线程经过``keepAliveTime``后如果没有取到任务就将线程销毁。
<img src="https://github.com/Qirui0805/Personal-Blog/blob/master/image/Thread%20in%20Pool.png" width = "800">

## BlockingQueue
代表一个线程安全的队列，提供向队列中添加和取出元素的方法。常用于生产者消费者模型中。
有四对加取和检查元素的方法：   

- Throw Exception
add(e), remove()，element()
- Value (没有则返回null)
offer(e), poll(), peek()
- Block
put(e), take(), none
- Time out
offer(e, time), poll(time), peek(time)
如果没有规定容量，则为Integer.MAX_VALUE
## SynchronousQueue   

- 每次添加元素的操作都要等待其他线程相应的从队列中取出元素的操作
- 不能存储元素, 因此peek()为null
- 有公平和非公平两种方式，内部分别通过``TransferQueue``和``TransferStack``实现

## LinkedBlockingQueue
- 处理元素的顺序为FIFO。添加元素在尾部，取出元素在头部
- 构造时若传入容量则有届，没有传入则无界

#### 如何保证线程安全
- putlock, takelock。添加取出操作使用不同的锁，remove操作要同时加两把锁。
## 










	
