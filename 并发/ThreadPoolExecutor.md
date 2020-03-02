### 为什么要用线程池
1. 降低资源消耗。通过重复利用已创建的线程降低线程创建、销毁线程造成的消耗。
2. 提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。
3. 提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配、调优和监控

### 线程池的底层类与接口
ExecutorService 是真正的线程池接口
Executor 是线程池的顶级接口，只是一个执行线程的工具，只提供一个execute(Runnable command)的方法，真正的线程池接口是ExecutorService
Executors 是静态工厂类，生产各种类型线程池
AbstractExecutorService 实现了ExecutorService接口，实现了其中大部分的方法（有没有实现的方法，所以被声明为Abstract）
ThreadPoolExecutor，继承了AbstractExecutorService，是ExecutorService的默认实现

### 线程池种类
• CachedThreadPool：一个任务创建一个线程；
• FixedThreadPool：所有任务只能使用固定大小的线程；
• SingleThreadExecutor：相当于大小为 1 的 FixedThreadPool。

本质上都是ThreadPoolExecutor, 由Executors下的工厂方法构造：
○ newFixedThreadPool：返回固定长度的线程池，线程池中的线程数量是固定的。
○ newCacheThreadPool：该方法返回一个根据实际情况来进行调整线程数量的线程池，空余线程存活时间是60s
○ newSingleThreadExecutor：该方法返回一个只有一个线程的线程池。
○ newSingleThreadScheduledExecutor：该方法返回一个SchemeExecutorService对象，线程池大小为1，SchemeExecutorService接口在ThreadPoolExecutor类和 ExecutorService接口之上的扩展，在给定时间执行某任务。
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
<img src="https://github.com/Qirui0805/Personal-Blog/blob/master/image/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E9%80%BB%E8%BE%91.png" width="600">

	
