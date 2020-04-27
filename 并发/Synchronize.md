# Synchronize
## 使用
- 锁方法
```java
public synchronize void method(){}
```
- 锁对象
```java
public void method() {
    synchronize(this){
        //do somthing
    }
}
```
- 锁类
```java
public void method() {
    synchronize(Class.class){
        //do somthing
    }
}
```
```java
public static synchronize void method(){}
```

## 如何保证同步/同步原理
synchronize是对象锁。每一个对象都有一把内置的锁，位于对象头中，因此synchronize要与一个对象关联，它要求访问临界区的线程要先获取对象上的锁，出临界区时释放锁。

### 对象头是什么
Java对象在内存中分为三部分：对象头，实例数据和对齐填充。
对象头由两部分组成：
- Mark Word: 对象运行时数据
- Class Pointer

#### Mark Word
在不同的锁的状态时存储的数据不同，但一定有的是锁的标志位：01（未占用），10（占用）
<img src="https://github.com/Qirui0805/Personal-Blog/blob/master/image/Mark%20Word.png" align="middle" width="500"/>
#### 所以锁是什么
锁的标志位和其他字段（线程ID/Lock Record指针/ObjectMoniter指针)共同扮演锁的角色

## 锁的状态
### 偏向锁
### 轻量级锁
### 重量级锁
#### 如何加锁
synchronize会被编译为monitorenter和moniterexit指令，monitorenter指令尝试获取monitor，如果获取成功，计数器加1，如果失败，则加入队列等待。
#### 什么是Monitor
Monitor是一种同步机制，也被描述为一个对象（monitor既指机制也指对象，但不是同一东西）。获取锁就是获取对象对应的monitor，获取的过程借助mutex实现。在HotSpot中用ObjectMonitor实现。Mark Word中存储指向ObjectMonitor的指针。
#### ObjectMonitor的结构
- owner: 指向线程
- entrylist: 阻塞在入口处的线程集合
- waitlist: 处于等待状态的的线程集合
- recursion: 重入次数
#### ObjectMonitor如何与上锁的过程联系起来
- 尝试获取mutex，获取成功则owner=self，recursion++，否则进入entrylist
- 重入时判断owner，recursion++
- 调用wait方法时（开发者层面），进入waitset阻塞，等待nofityall和notify唤醒后进入entryset
- 释放mutex后recursion--
### 各个状态转换

## 参考资料
[深入分析Synchronized原理](https://juejin.im/post/5b4eec7df265da0fa00a118f)
[所谓“阻塞”：Object Monitor和AQS（1）](https://blog.csdn.net/yinwenjie/article/details/84922958)
[Programming Java Threads, Part 2](http://www.nyu.edu/classes/jcf/g22.3033-007_sp01/handouts/g22_3033_h53.htm)
[关键字: synchronized详解](https://www.pdai.tech/md/java/thread/java-thread-x-key-synchronized.html)
