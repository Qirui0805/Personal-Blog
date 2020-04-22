## 如何隔离线程间的数据
答：ThreadLocal

## 什么是ThreadLocal
这个类提供线程的本地变量，每个访问这个变量的线程都保存着独立初始化的副本，实现线程间的数据隔离（来自官方文档）

## 用于什么场景
用于存储与线程关联的变量，且要求变量在方法间共享。比如连接池中线程与数据库connection关联。（不用这个，维护一个list，然后用map<connect, obj>行不行，当然可以，但是相当于自己干了threadlocal的活，不elegant）

## 测试

```java
public class ThreadLocalTest {

    private static ThreadLocal<StringBuilder> threadLocal = ThreadLocal.withInitial(StringBuilder::new);

    public static void main(String[] args) {
        for (int i = 0; i < 3; i++) {
            new Thread() {
                @Override
                public void run() {
                    StringBuilder builder = threadLocal.get();
                    threadLocal.set(builder.append(currentThread().getName()));
                    System.out.println(String.format("%s, %s", threadLocal.get().toString(), threadLocal.get().hashCode()));
                }
            }.start();
        }
    }
}

result:

Thread-0, 896298396
Thread-1, 1385166630
Thread-2, 924507082
```

## 如何实现
每个线程维护一个独立的ThreadLocalMap对象，存储ThreadLocal实例与变量的映射关系，通过ThreadLocal的set和get方法访问。
```java
public
class Thread implements Runnable {

  ThreadLocal.ThreadLocalMap threadLocals = null;
  
}
```

## 什么是ThreadLocalMap（数据结构）
是一个hash map, 是ThreadLocal的内部类，用Entry数组存储数据，ThreadLocal实例为key，变量为value。
```java
static class ThreadLocalMap {

        static class Entry extends WeakReference<ThreadLocal<?>> {
        
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

        private Entry[] table;
}

```
## 怎么处理哈希冲突
使用开放地址法，使用set方法时，计算出桶下标后，如果桶的位置已经被占有，而且key不相同，则往后一直找到第一个为空的桶。

## 内存泄漏
ThreadLocalMap对key的引用为弱引用，（为什么？为了防止当ThreadLocal实例使用结束后，map里仍有对它的强引用导致无法被回收），这本为了使ThreadLocal实例能被回收，但会导致被回收后，entry的key为null，但map里仍有对value的强引用，value无法被回收

## 如何解决
在set时，对key为null的entry做清理
```java
private void set(ThreadLocal<?> key, Object value) {

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }
 
                if (k == null) {
                    //清理key为null的
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

但这是被动解决方案，主动解决方案应该是在使用结束后remove
