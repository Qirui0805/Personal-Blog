### 问题
- 公平锁和非公平锁是如何实现的
- 如何获取锁，如何重入
### 简介
• 称为可重入锁，功能基于CAS和AQS的独占模式实现
• 有公平锁和非公平锁两种策略
		
### 成员变量
				

• Sync类抽象成员类，内部类，继承于AQS, 同步方法都通过这个类实现
• Sync类有FairSync和NonfairSync两种实现方式，默认情况下为非公平锁
```java
public class ReentrantLock implements Lock, java.io.Serializable {
    /** Synchronizer providing all implementation mechanics */
    private final Sync sync;
    * Subclassed into fair and nonfair versions below. Uses AQS state to represent the number of holds on the lock.
public ReentrantLock() { sync = new NonfairSync(); }
public ReentrantLock(boolean fair) { sync = fair ? new FairSync() : new NonfairSync(); }
      abstract static class Sync extends AbstractQueuedSynchronizer {}
```
### 非公平锁：
#### lock()
• 调用syn的lock方法
```java
	public void lock(){
		sync.lock();
	}
```
• 直接尝试抢占锁——把state设为1，state变量定义在AQS中, 设置exclusiveOwnerThread为当前线程，该变量定义在AbstractOwnableSynchronizer中，AQS继承了该类；若失败则通过acquire加锁，实现逻辑已在AQS中实现，NonFair, Fair的tryacquire方法不一样。
• volatile int state; 用volatile修饰对其他线程保持可见性。当为0时表示锁是空闲，可以获取锁，当大于0时表示获得锁。 独占锁时大于0表示锁的重入次数，共享锁时，state共当前共享线程个数。
```java
	final void lock() {
	            if (compareAndSetState(0, 1))
	                setExclusiveOwnerThread(Thread.currentThread());
	            else
	                acquire(1);
	        }
```
CAS:
```java
	protected final boolean compareAndSetState(int expect, int update) {
	        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
	    }
```	    
#### acquire()
• 逻辑在AQS中实现，不再赘述，只关注NonFair和Fair的tryAcquire方法的不同。
• 如果tryAcquire失败，则公平锁和非公平锁逻辑就一样了，都进入队列等待
```java
	public final void acquire(int arg) {
	        if (!tryAcquire(arg) &&
	            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
	            selfInterrupt();
	    }
```
#### tryAcquire()
```java
protected final boolean tryAcquire(int acquires) {
           return nonfairTryAcquire(acquires);
       }
   }
```
#### nonfairTryAcquire()
• 该方法定义在Sync中，判断锁的状态，如果为0表示空闲，则不管队列情况直接尝试获取锁；若当前线程占有锁，则重入，状态+1
```java
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {//state==0，锁空闲
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {// 锁重入
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
### 公平锁：
#### lock()
• 不去抢锁，直接进入tryaquire方法
```java
final void lock() {
           acquire(1);
       }
```
#### tryAcquire()
• 与非公平锁的逻辑不同之处为，nonfailTryAcquire不同的是，当state为0时，要先判断hasQueuedPredecessors(), hasQueuedPredecessors判断head和tail不相等（说明有等待线程）并且（head.next为null=>说明有线程正在入队列的中间状态，肯定不是当前线程, 因为一个线程一个时间只能做一件事 或者 head.next.thread不是当前线程)
```java
protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
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
### 回答问题
#### 非公平锁和公平锁如何实现
- 非公平锁：无论同步队列中是否有线程在等待，会直接尝试获取锁
- 公平锁：如果同步队列中有线程等待，则直接加入队列等待
#### 如何获取锁，如何重入
- 如何获取锁：锁用state变量表示，0表示锁为空闲状态，大于0表示被占有。获取锁通过
