## 简介
• AQS是JUC其他同步工具如Reentrant Lock 、 Semaphore 和CountDownLatch的基础
• AQS的并发操作都是基于CAS完成，只要是对head和tail的设置
• AQS对锁的获取方法提供了独占和共享两种模式，采用何种模式由子类决定
## 结构与成员变量
• 典型的双向链表结构
```java
private transient volatile Node head;
private transient volatile Node tail;
 * The synchronization state.
private volatile int state;

static final class Node {
    static final Node SHARED = new Node();
    static final Node EXCLUSIVE = null;
     //waitStatus. Non-negative values mean that a node doesn't need to signal.
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;
     
    volatile int waitStatus;

    volatile Node prev;
    volatile Node next;

    volatile Thread thread;

    Node nextWaiter;
```
## 独占模式
### acquire()
• 先尝试获取资源，tryAcquire方法由子类实现；如果成功，之后的代码就不用执行
• 如果tryAcquire方法失败，则以独占模式创建Node, 从队尾加入队列，并继续尝试获取。如果该方法中间发生中断，则会返回true
•  selfInterrupt()调用thread.interrupt方法，最终调用native interrupt方法，给当前线程设置中断标志
```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
                   selfInterrupt();
    }
```
### addWaiter()
• 此方法的逻辑是把当前线程封装为Node, 并加入队尾；如果队列不为空，就直接尝试快速入列；如果队列为空，且修改tail出现竞争，则尝试用enq方法
```java
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        // 如果队列不为空则CAS入列；若队列为空 或 出现竞争，则进入enq方法自旋入列；如果队列为空还要创建空节点作为头节点。
      //TODO: 为啥空节点要进enq，有什么好处，明明可以进行类似的操作
        if (pred != null) {
            node.prev = pred;// TODO:  暂时没明白为什么不把这个操作放在CAS后；2020.1.16更新：我觉得因为没必要，放外面可读性好
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```
### enq()
```java
private Node enq(final Node node) {
        for (;;) { // 自旋
            Node t = tail;  // 每一次自旋都要重新获取tail，因为CAS可能出现竞争
            if (t == null) { // Must initialize
	     // 创建空节点作为头节点，称为dummy；创建成功后继续自旋，将执行else代码；如果CAS不成功，下一次自旋也将执行else
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
### acquireQueued()
• 当前线程加入队列后自旋尝试获取资源，只有当prev节点为头节点，且头节点为空节点时获取成功，否则一直自旋或发生中断
•  interrupted变量标示是否发生中断，是则返回true，回到acquire方法调用interrupt
```java
inal boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {// 自旋
                final Node p = node.predecessor();// 不直接用prev的原因还没弄明白，predecessor会在null时抛出NullPointerException
	    /如果node的前一节点为head节点,而head节点为空节点，说明node是等待队列里排在最前面的节点
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
	     //如果acquire失败，是否要park,如果是则调用LockSupport.park
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
### shouldParkAfterFailedAcquire()
• 逻辑是如果前一节点waitstatus为SIGNAL则park；如果为CANCELED，则一直找到非CANCELED的为止，顺便将CANCELED的从队列中删除，回到acquirequeued继续尝试获取资源；如果为0或PROPAGATE，则改为SIGNAL，回到acquirequeued继续尝试获取资源
```java
/* Checks and updates status for a node that failed to acquire. Returns true if thread should block. This is the main signal control in all acquire loops.  Requires that pred == node.prev.*/
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;// 获取前一节点的waitstatus
        if (ws == Node.SIGNAL)
            //若前一节点已经设置了SIGNAL，那么将通过unpack唤醒后一节点，因此可以阻塞，返回true
            return true;
        if (ws > 0) {
             * Predecessor was cancelled. Skip over predecessors and indicate retry. 同时也将取消的点删除
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;//更新next指针，去掉中间取消状态的节点
        } else {
             * waitStatus must be 0 or PROPAGATE.  Indicate that we need a signal, but don't park yet.  Caller will need to retry to make sure it cannot acquire before parking.
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);//更新pred节点的waitStatus为SIGNAL
        }
        return false;
    }
```
### parkAndCheckInterrupt() 
将当前线程的parkBlocker变量指向this，调用unsafe.park堵塞当前线程
简单来说park是申请许可，如果存在许可，马上返回，否则一直等待获得许可;unpark是将许可数量设为1，会唤醒park返回;
LockSupport提供了unpark(Thread thread)方法，可以为指定线程颁发许可
[java如何实现多线程](https://www.jianshu.com/p/82b2d2361fb8)
```java
 private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```					

共享模式
//TODO



