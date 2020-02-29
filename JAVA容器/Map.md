## HashMap

### 存储结构
	
内部包含了一个 Entry 类型的数组 table。
```java
transient Entry[] table;
```
Entry 存储着键值对。它包含了四个字段，从 next 字段我们可以看出 Entry 是一个链表。即数组中的每个位置被当成一个桶，一个桶存放一个链表。HashMap 使用拉链法来解决冲突，同一个链表中存放哈希值和散列桶取模运算结果相同的 Entry。
```java
	static class Node<K,V> implements Map.Entry<K,V>//jdk1.8改名称为Node
	static class Entry<K,V> implements Map.Entry<K,V> {
    final K key;
    V value;
    Entry<K,V> next;
    int hash;
		Entry(int h, K k, V v, Entry<K,V> n) {
        value = v;
        next = n;
        key = k;
        hash = h;
    }
		public final K getKey() {
        return key;
    }
		public final V getValue() {
        return value;
    }
		public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }
		public final boolean equals(Object o) {//Object类型
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry e = (Map.Entry)o;
        Object k1 = getKey();
        Object k2 = e.getKey();
        if (k1 == k2 || (k1 != null && k1.equals(k2))) {
            Object v1 = getValue();
            Object v2 = e.getValue();
            if (v1 == v2 || (v1 != null && v1.equals(v2)))
                return true;
        }
        return false;
    }
```
- jdk1.8
对比jdk1.7的实现，jdk1.8增加了是否为同一对象的判断（o==this), 对自身的key和value也是直接引用，在jdk1.7中使用get, 判断内容相等也用objects的实现，这个类在1.7就有了，但是jdk1.7没有使用。对比Java基础中equals的实现思路一样，只是这里不要求属于同一类
```java
public final boolean equals(Object o){
	if(o==this)
		return true;
	if(o instanceof Map.Entry){ //并不严格要求属于同一类，可以是父类和子类的关系
		Map.Entry<?,?> e=(Map.Entry<?,?>)o;
	if(Objects.equals(key,e.getKey())&&
		Objects.equals(value,e.getValue()))
		return true;
	}
	return false;
}
```
判断两对象是否等价应该用e.quals，但如果a为null, 则会NullPointerException, 所以要保证a!=null。 同时如果a和b都为null是也是等价的
```java
public static boolean equals(Object a,Object b){
	return (a==b)||(a!=null&&a.equals(b));
}

public final int hashCode() {
    return Objects.hashCode(getKey()) ^ Objects.hashCode(getValue());//位异或运算
}
public final String toString() {
    return getKey() + "=" + getValue();
}
```
### 拉链法的工作原理
```java
HashMap<String, String> map = new HashMap<>();
map.put("K1", "V1");
map.put("K2", "V2");
map.put("K3", "V3");
```
§ 新建一个 HashMap，默认大小为 16；
§ 插入 <K1,V1> 键值对，先计算 K1 的 hashCode 为 115，使用除留余数法得到所在的桶下标 115%16=3。
§ 插入 <K2,V2> 键值对，先计算 K2 的 hashCode 为 118，使用除留余数法得到所在的桶下标 118%16=6。
§ 插入 <K3,V3> 键值对，先计算 K3 的 hashCode 为 118，使用除留余数法得到所在的桶下标 118%16=6，插在 <K2,V2> 前面。
应该注意到链表的插入是以头插法方式进行的，例如上面的 <K3,V3> 不是插在 <K2,V2> 后面，而是插入在链表头部。
查找需要分成两步进行：
§ 计算键值对所在的桶；
§ 在链表上顺序查找，时间复杂度显然和链表的长度成正比。
		
### 计算数组容量
HashMap 构造函数允许用户传入的容量不是 2 的 n 次方，因为它可以自动地将传入的容量转换为 2 的 n 次方。
先考虑如何求一个数的掩码，对于 10010000，它的掩码为 11111111，可以使用以下方法得到：
		mask |= mask >> 1    11011000
mask |= mask >> 2    11111110
mask |= mask >> 4    11111111
		mask+1 是大于原始数字的最小的 2 的 n 次方。
		num     10010000
mask+1 100000000
		以下是 HashMap 中计算数组容量的代码：
```java
static final int tableSizeFor(int cap) {
    int n = cap - 1; //减1是针对cap本身就是2的倍数的情况
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
### put 操作（jdk1.7)
与jdk1.8思路基本是一致的，主要的不同之处在于jdk1.8中链表长度>8则转为红黑树。
```java
public V put(K key, V value) {
    if (table == EMPTY_TABLE) {
        inflateTable(threshold); //jdk1.8取消了这个函数，选择直接在操作中判断
    }
    // 键为 null 单独处理，jdk1.8中在hash()函数中判断，若为null则hash为0，取模后得下标为0
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key);
    // 确定桶下标
    int i = indexFor(hash, table.length);         //jdk1.8中取消了该函数，选择直接在函数中判断
    // 先找出是否已经存在键为 key 的键值对，如果存在的话就更新这个键值对的值为 value
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
		    modCount++;
    // 插入新键值对
    addEntry(hash, key, value, i);
    return null;
}
```
HashMap 允许插入键为 null 的键值对。但是因为无法调用 null 的 hashCode() 方法，也就无法确定该键值对的桶下标，只能通过强制指定一个桶下标来存放。HashMap 使用第 0 个桶存放键为 null 的键值对。
```java
		private V putForNullKey(V value) {
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    addEntry(0, null, value, 0);
    return null;
}
```
使用链表的头插法，也就是新的键值对插在链表的头部，而不是链表的尾部。
```java
		void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }
		    createEntry(hash, key, value, bucketIndex);
}
		void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    // 头插法，链表头部指向新的键值对
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
		Entry(int h, K k, V v, Entry<K,V> n) {
    value = v;
    next = n;
    key = k;
    hash = h;
}
```
### 确定桶下标
	
很多操作都需要先确定一个键值对所在的桶下标。总的来说分为三步，获取Entry的hashCode -> 计算hash -> 取模计算下标
		int hash = hash(key);
int i = indexFor(hash, table.length);
		
### 计算 hash 值
[参考资料解释](https://blog.csdn.net/caimengyuan/article/details/61204542)
对扰动函数的解释见jdk1.8, jdk1.7中为了对散列值进行优化进行了四次扰动，jdk1.8中只进行了一次。计算hash时jdk1.7中使用了key 和value的hashcode，jdk1.8中只用了key的。
```java
final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }
		    h ^= k.hashCode();
		    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}

public final int hashCode() {
    return Objects.hashCode(key) ^ Objects.hashCode(value);
}
```
		
### 取模
[详细解释见这篇博客](https://blog.csdn.net/u010386612/article/details/80302777)
		位运算的代价比求模运算小的多，因此在进行这种计算时用位运算的话能带来更高的性能。
		确定桶下标的最后一步是将 key 的 hash 值对桶个数取模：hash%capacity，如果能保证 capacity 为 2 的 n 次方，那么就可以将这个操作转换为位运算。
		这也正好解释了为什么HashMap的数组长度要取2的整次幂。因为这样（数组长度-1）正好相当于一个“低位掩码”。“与”操作的结果就是散列值的高位全部归零，只保留低位值，用来做数组下标访问。以初始长度16为例，16-1=15。2进制表示是00000000 00000000 00001111。
```java
		static int indexFor(int h, int length) {
    return h & (length-1);
}
```
### put 操作（jdk1.8)
参考：
		[jdk1.8-HashMap源码解析-csdn](https://blog.csdn.net/u010386612/article/details/80302777)
		[jdk1.8-美团技术团队HashMap解析-知乎](https://zhuanlan.zhihu.com/p/21673805)
```java
		public V put(K key,V value){
			// 对key的hashCode()做hash
			return putVal(hash(key),key,value,false,true);
		}
		final V putVal(int hash,K key,V value,boolean onlyIfAbsent, boolean evict){
			Node<K,V>[]tab; Node<K,V>p; int n,i;
			if((tab=table)==null||(n=tab.length)==0) // 步骤①：tab为空则创建
				n=(tab=resize()).length;
			if((p=tab[i=(n-1)&hash])==null)  //步骤②：计算index，并对null做处理
				tab[i]=newNode(hash,key,value,null);
			else{
				Node<K,V>e; K k;
				//步骤③：节点key存在，定位改节点，直接覆盖value（覆盖操作在最后）
				if(p.hash==hash&&
					((k=p.key)==key||(key!=null&&key.equals(k))))
					e=p;
				// 步骤④：判断该链为红黑树
				else if(p instanceof TreeNode)
					e=((TreeNode<K,V>)p).putTreeVal(this,tab,hash,key,value);
				// 步骤⑤：该链为链表
				else{
					for(int binCount=0; ;++binCount){
						if((e=p.next)==null){
							p.next=newNode(hash,key,value,null); //末尾插入
							//若大于8转红黑树
							if(binCount>=TREEIFY_THRESHOLD-1)//-1for1st
								treeifyBin(tab,hash);
							break;
						}
						// 如果存在key则覆盖
						if(e.hash==hash && ((k=e.key)==key||(key!=null&&key.equals(k))))
							break;
						p=e;
					}
				}
				// 覆盖value
				if(e!=null){//existing mapping for key
					V oldValue=e.value;
					if(!onlyIfAbsent||oldValue==null)
						e.value=value;
					afterNodeAccess(e);
					return oldValue;
				}
			}
			++modCount;
			// 步骤⑥：超过最大容量 就扩容
			if(++size>threshold)
				resize();
			afterNodeInsertion(evict);
			return null;
		}
```
### 计算hash
计算下标的思路和jdk1.7是一致的，分为三步：获取Entry的hashCode -> 计算hash -> 取模计算下标。
		不同的是：jdk1.7中为了对散列值进行优化进行了四次扰动，jdk1.8中只进行了一次。计算hash时jdk1.7中使用了key 和value的hashcode，jdk1.8中只用了key的。
		jdk1.8中在hash()函数中判断，若为null则hash为0，取模后得下标为0
```java
		static final int hash(Object key){
			int h;
			return (key==null)?0:(h=key.hashCode())^(h>>>16);
		}
```
key.hashCode()函数调用的是key键值类型自带的哈希函数，返回int型散列值。理论上散列值是一个int型，2进制32位带符号的int表值范围从-2147483648到2147483648。前后加起来大概40亿的映射空间。一个40亿长度的数组，内存是放不下的。用之前还要先做对数组的长度取模运算，得到的余数才能用来访问数组下标。
		
但这时候问题就来了，这样就算我的散列值分布再松散，要是只取最后几位的话，碰撞也会很严重。更要命的是如果散列本身做得不好，分布上成等差数列的漏洞，恰好使最后几个低位呈现规律性重复，就无比蛋疼。
		
		key.hashCode())^(h>>>16）称为扰动函数。右位移16位，正好是32bit的一半，自己的高半区和低半区做异或，就是为了混合原始哈希码的高位和低位，以此来加大低位的随机性。而且混合后的低位掺杂了高位的部分特征，这样高位的信息也被变相保留下来。用“异或”运算的理由我猜想可能是：因为hashcode的比较大，因此高位基本都会是1，用“与”（&）运算的话则就会变成全都是1，取模后得到的下标都是同一个；用“或”的话因为高位都是1，则留下的仍然只是地位的信息，没有达到目的。只有”异或“能把高位的地位的信息都保留下来
		
### 扩容
	
基本原理
	
先以jdk1.7为例。

设 HashMap 的 table 长度为 M，需要存储的键值对数量为 N，如果哈希函数满足均匀性的要求，那么每条链表的长度大约为 N/M，因此平均查找次数的复杂度为 O(N/M)。

为了让查找的成本降低，应该尽可能使得 N/M 尽可能小，因此需要保证 M 尽可能大，也就是说 table 要尽可能大。HashMap 采用动态扩容来根据当前的 N 值来调整 M 值，使得空间效率和时间效率都能得到保证。
和扩容相关的参数主要有：capacity、size、threshold 和 load_factor。
参数	    含义
capacity	table 的容量大小，默认为 16。需要注意的是 capacity 必须保证为 2 的 n 次方。
size	    键值对数量。
```java
threshold	size 的临界值，当 size 大于等于 threshold 就必须进行扩容操作。
loadFactor	装载因子，table 能够使用的比例，threshold = (int)(newCapacity * loadFactor)。
static final int DEFAULT_INITIAL_CAPACITY = 16;
static final int MAXIMUM_CAPACITY = 1 << 30;
static final float DEFAULT_LOAD_FACTOR = 0.75f;
transient Entry[] table;
transient int size;
int threshold;
final float loadFactor;
transient int modCount;
从下面的添加元素代码中可以看出，当需要扩容时，令 capacity 为原来的两倍。
void addEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    if (size++ >= threshold)
        resize(2 * table.length);
}
```
扩容使用 resize() 实现，需要注意的是，扩容操作同样需要把 oldTable 的所有键值对重新插入 newTable 中，因此这一步是很费时的。
		``newTable[i]``的引用赋给了``e.next``，也就是使用了单链表的头插入方式，同一位置上新元素总会被放在链表的头部位置；这样先放在一个索引上的元素终会被放到Entry链的尾部(如果发生了hash冲突的话），这一点和Jdk1.8有区别，下文详解。在旧数组中同一条Entry链上的元素，通过重新计算索引位置后，有可能被放到了新数组的不同位置上。
		
举例及图解
```java		
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) { //超过最大值就不管了
        threshold = Integer.MAX_VALUE;
        return;
    }
    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable);
    table = newTable;
    threshold = (int)(newCapacity * loadFactor);
}

void transfer(Entry[] newTable) {
    Entry[] src = table;
    int newCapacity = newTable.length;
    for (int j = 0; j < src.length; j++) {
        Entry<K,V> e = src[j];
        if (e != null) {
            src[j] = null;
            do {
				//头插入链表
		                Entry<K,V> next = e.next; //定位原链表中下一个entry
                int i = indexFor(e.hash, newCapacity); //计算新数组中下标
                e.next = newTable[i]; //将该节点置于新数组该下标位置的首位
                newTable[i] = e;
                e = next;
            } while (e != null);
        }
    }
}
```	
### JDK1.8对重新计算桶下标的优化
		
经过观测可以发现，我们使用的是2次幂的扩展(指长度扩为原来2倍)，所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。看下图可以明白这句话的意思，n为table的长度，图（a）表示扩容前的key1和key2两种key确定索引位置的示例，图（b）表示扩容后key1和key2两种key确定索引位置的示例，其中hash1是key1对应的哈希与高位运算结果。
		
元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：

因此，我们在扩充HashMap的时候，不需要像JDK1.7的实现那样重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”，可以看看下图为16扩充为32的resize示意图：

这个设计确实非常的巧妙，既省去了重新计算hash值的时间，而且同时，由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的bucket了。这一块就是JDK1.8新增的优化点。有一点注意区别，JDK1.7中rehash的时候，旧链表迁移新链表的时候，如果在新表的数组索引位置相同，则链表元素会倒置，但是从上图可以看出，JDK1.8不会倒置

源码分析略过，可见参考博客....
		
与 HashTable 的比较

- HashTable 使用 synchronized 来进行同步。
- HashMap 可以插入键为 null 的 Entry。
- HashMap 的迭代器是 fail-fast 迭代器。
- HashMap 不能保证随着时间的推移 Map 中的元素次序是不变的。
		
### 线程安全性-环形链表
		
在多线程使用场景中，应该尽量避免使用线程不安全的HashMap，而使用线程安全的``ConcurrentHashMap``。那么为什么说HashMap是线程不安全的，下面举例子说明在并发的多线程使用场景中使用HashMap可能造成死循环。代码例子如下(便于理解，仍然使用JDK1.7的环境)：
```java
	public class HashMapInfiniteLoop {  
		private static HashMap<Integer,String> map = new HashMap<Integer,String>(2，0.75f);  
    public static void main(String[] args) {  
        map.put(5， "C");  
		new Thread("Thread1") {  
            public void run() {  
                map.put(7, "B");  
                System.out.println(map);  
            };  
        }.start();  
        new Thread("Thread2") {  
            public void run() {  
                map.put(3, "A);  
                System.out.println(map);  
            };  
        }.start();        
    }  
}
```
		其中，map初始化为一个长度为2的数组，loadFactor=0.75，threshold=2*0.75=1，也就是说当put第二个key的时候，map就需要进行resize。
		通过设置断点让线程1和线程2同时debug到transfer方法(3.3小节代码块)的首行。注意此时两个线程已经成功添加数据。放开thread1的断点至transfer方法的“Entry next = e.next;” 这一行；然后放开线程2的的断点，让线程2进行resize。结果如下图。
		
		注意，Thread1的 e 指向了key(3)，而next指向了key(7)，其在线程二rehash后，指向了线程二重组后的链表。
		线程一被调度回来执行，先是执行 newTalbe[i] = e， 然后是e = next，导致了e指向了key(7)，而下一次循环的next = e.next导致了next指向了key(3)。
		
		e.next = newTable[i] 导致 key(3).next 指向了 key(7)。注意：此时的key(7).next 已经指向了key(3)， 环形链表就这样出现了。
		
		于是，当我们用线程一调用map.get(11)时，悲剧就出现了——Infinite Loop。
	
## ConcurrentHashMap
	
先以jdk1.7为例

### 存储结构
```java
		static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;
}
```
``ConcurrentHashMap`` 和 ``HashMap`` 实现上类似，最主要的差别是 ``ConcurrentHashMap`` 采用了分段锁（Segment），每个分段锁维护着几个桶（HashEntry），多个线程可以同时访问不同分段锁上的桶，从而使其并发度更高（并发度就是 Segment 的个数）。
Segment 继承自 ReentrantLock。
```java		
		static final class Segment<K,V> extends ReentrantLock implements Serializable {
		private static final long serialVersionUID = 2249069246763182397L;
		static final int MAX_SCAN_RETRIES =
        Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;
		transient volatile HashEntry<K,V>[] table;
		transient int count;
		transient int modCount;
		transient int threshold;
		final float loadFactor;
}
		final Segment<K,V>[] segments;
		//默认的并发级别为 16，也就是说默认创建 16 个 Segment。
		static final int DEFAULT_CONCURRENCY_LEVEL = 16;
```
### 并发操作
		
IBM-developer-ConcurrentMap解析jdk1.7

原理：分段锁，volatile规则，对结构性修改操作的设计

在 ``ConcurrentHashMap`` 中，线程对映射表做读操作时，一般情况下不需要加锁就可以完成，对容器做结构性修改的操作才需要加锁。下面以 put 操作为例说明对 ConcurrentHashMap 做结构性修改的过程。
```java
		public V put(K key, V value) { 
		       if (value == null)          //ConcurrentHashMap 中不允许用 null 作为映射值
		           throw new NullPointerException(); 
		       int hash = hash(key.hashCode());        // 计算键对应的散列码
		       // 根据散列码找到对应的 Segment 
		       return segmentFor(hash).put(key, hash, value, false); 
		}
		
		final Segment<K,V> segmentFor(int hash) { 
		   // 将散列值右移 segmentShift 个位，并在高位填充 0 
		   // 然后把得到的值与 segmentMask 相“与”
		// 从而得到 hash 值对应的 segments 数组的下标值
		// 最后根据下标值返回散列码对应的 Segment 对象
		       return segments[(hash >>> segmentShift) & segmentMask]; 
		}
		
		V put(K key, int hash, V value, boolean onlyIfAbsent) { 
           lock();  // 加锁，这里是锁定某个 Segment 对象而非整个 ConcurrentHashMap 
           try { 
               int c = count; 
 
               if (c++ > threshold)     // 如果超过再散列的阈值
                   rehash();              // 执行再散列，table 数组的长度将扩充一倍
 
               HashEntry<K,V>[] tab = table; 
               // 把散列码值与 table 数组的长度减 1 的值相“与”
               // 得到该散列码对应的 table 数组的下标值
               int index = hash & (tab.length - 1); 
               // 找到散列码对应的具体的那个桶
               HashEntry<K,V> first = tab[index]; 
 
               HashEntry<K,V> e = first; 
               while (e != null && (e.hash != hash || !key.equals(e.key))) 
                   e = e.next; 
 
               V oldValue; 
               if (e != null) {            // 如果键 / 值对以经存在
                   oldValue = e.value; 
                   if (!onlyIfAbsent) 
                       e.value = value;    // 设置 value 值
               } 
               else {                        // 键 / 值对不存在 
                   oldValue = null; 
                   ++modCount;         // 要添加新节点到链表中，所以 modCont 要加 1  
                   // 创建新节点，并添加到链表的头部 
                   tab[index] = new HashEntry<K,V>(key, hash, first, value); 
                   count = c;               // 写 count 变量
               } 
               return oldValue; 
           } finally { 
               unlock();                     // 解锁
           } 
       }
```
这里的加锁操作是针对（键的 hash 值对应的）某个具体的 Segment，锁定的是该 Segment 而不是整个 ``ConcurrentHashMap``。因为插入键 / 值对操作只是在这个 Segment 包含的某个桶中完成，不需要锁定整个``ConcurrentHashMap``。此时，其他写线程对另外 15 个Segment 的加锁并不会因为当前线程对这个 Segment 的加锁而阻塞。同时，所有读线程几乎不会因本线程的加锁而阻塞（除非读线程刚好读到这个 Segment 中某个 HashEntry 的 value 域的值为 null，此时需要加锁后重新读取该值）。
		
相比较于 ``HashTable`` 和由同步包装器包装的 ``HashMap``每次只能有一个线程执行读或写操作，``ConcurrentHashMap`` 在并发访问性能上有了质的提高。在理想状态下，``ConcurrentHashMap`` 可以支持 16 个线程执行并发写操作（如果并发级别设置为 16），及任意数量线程的读操作。
		
		
### 降低读操作对加锁的需求
		
HahEntry的next被声明为volatile, 保证写线程对链表进行修改后读线程能立即“看到”。同时，value 域也被声明为 Volatile 型，在 ConcurrentHashMap 中，不允许用 null 作为键和值，当读线程读到某个 HashEntry 的 value 域的值为 null 时，便知道产生了冲突——发生了重排序现象（猜想：读发生在写前了），需要加锁后重新读入这个 value 值
		
线程写入的两种情形：对散列表做非结构性修改的操作和对散列表做结构性修改的操作。

非结构性修改操作只是更改某个 HashEntry 的 value 域的值。由于对 Volatile 变量的写入操作将与随后对这个变量的读操作进行同步。当一个写线程修改了某个 HashEntry 的 value 域后，另一个读线程读这个值域，Java 内存模型能够保证读线程读取的一定是更新后的值。如上面put方法所言，除非读到null，否则不需要加锁去读。

对 ConcurrentHashMap 做结构性修改，实质上是对某个桶指向的链表做结构性修改。如果能够确保：在读线程遍历一个链表期间，写线程对这个链表所做的结构性修改不影响读线程继续正常遍历这个链表。那么读 / 写线程之间就可以安全并发访问这个 ConcurrentHashMap。
结构性修改操作包括 put，remove，clear。分别分析这三个操作。
		
clear 操作只是把 ConcurrentHashMap 中所有的桶“置空”，每个桶之前引用的链表依然存在，只是桶不再引用到这些链表（所有链表的结构并没有被修改）。正在遍历某个链表的读线程依然可以正常执行对该链表的遍历。

从上面的代码清单“在 Segment 中执行具体的 put 操作”中，我们可以看出：put 操作如果需要插入一个新节点到链表中时 , 会在链表头部插入这个新节点。此时，链表中的原有节点的链接并没有被修改。也就是说：插入新健 / 值对到链表中的操作不会影响读线程正常遍历这个链表。
下面来分析 remove 操作，先让我们来看看 remove 操作的源代码实现。
```java
		1	V remove(Object key, int hash, Object value) { 
		2	           lock();         // 加锁
		3	           try{ 
		4	               int c = count - 1; 
		5	               HashEntry<K,V>[] tab = table; 
		6	               // 根据散列码找到 table 的下标值
		7	               int index = hash & (tab.length - 1); 
		8	               // 找到散列码对应的那个桶
		9	               HashEntry<K,V> first = tab[index]; 
		10	               HashEntry<K,V> e = first; 
		11	               while(e != null&& (e.hash != hash || !key.equals(e.key))) 
		12	                   e = e.next; 
		13	 
		14	               V oldValue = null; 
		15	               if(e != null) { 
		16	                   V v = e.value; 
		17	                   if(value == null|| value.equals(v)) { // 找到要删除的节点
		18	                       oldValue = v; 
		19	                       ++modCount; 
		20	                       // 所有处于待删除节点之后的节点原样保留在链表中
		21	                       // 所有处于待删除节点之前的节点被克隆到新链表中
		22	                       HashEntry<K,V> newFirst = e.next;// 待删节点的后继结点
		23	                       for(HashEntry<K,V> p = first; p != e; p = p.next) 
		24	                           newFirst = new HashEntry<K,V>(p.key, p.hash, 
		25	                                                         newFirst, p.value); 
		26	                       // 把桶链接到新的头结点
		27	                       // 新的头结点是原链表中，删除节点之前的那个节点
		28	                       tab[index] = newFirst; 
		29	                       count = c;      // 写 count 变量
		30	                   } 
		31	               } 
		32	               return oldValue; 
		33	           } finally{ 
		34	               unlock();               // 解锁
		35	           } 
		36	       }
```
		和 get 操作一样，首先根据散列码找到具体的链表；然后遍历这个链表找到要删除的节点；最后把待删除节点之后的所有节点原样保留在新链表中，把待删除节点之前的每个节点克隆到新链表中。下面通过图例来说明 remove 操作。假设写线程执行 remove 操作，要删除链表的 C 节点，另一个读线程同时正在遍历这个链表。
		图 4. 执行删除之前的原链表：
		
		图 5. 执行删除之后的新链表
		
		从上图可以看出，删除节点 C 之后的所有节点原样保留到新链表中；删除节点 C 之前的每个节点被克隆到新链表中，注意：它们在新链表中的链接顺序被反转了。
		在执行 remove 操作时，原始链表并没有被修改，也就是说：读线程不会受同时执行 remove 操作的并发写线程的干扰。
		综合上面的分析我们可以看出，写线程对某个链表的结构性修改不会影响其他的并发读线程对这个链表的遍历访问。
		
### size 操作
		
每个 Segment 维护了一个 count 变量来统计该 Segment 中的键值对个数。

transient int count;
		在执行 size 操作时，需要遍历所有 Segment 然后把 count 累计起来。
		ConcurrentHashMap 在执行 size 操作时先尝试不加锁，如果连续两次不加锁操作得到的结果一致，那么可以认为这个结果是正确的。
		尝试次数使用 RETRIES_BEFORE_LOCK 定义，该值为 2，retries 初始值为 -1，因此尝试次数为 3。
		如果尝试的次数超过 3 次，就需要对每个 Segment 加锁。
```java
		/**
 * Number of unsynchronized retries in size and containsValue
 * methods before resorting to locking. This is used to avoid
 * unbounded retries if tables undergo continuous modification
 * which would make it impossible to obtain an accurate result.
 */
static final int RETRIES_BEFORE_LOCK = 2;
		public int size() {
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    int retries = -1; // first iteration isn't retry
    try {
        for (;;) {
            // 超过尝试次数，则对每个 Segment 加锁
            if (retries++ == RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); // force creation
            }
            sum = 0L;
            size = 0;
            overflow = false;
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
            // 连续两次得到的结果一致，则认为这个结果是正确的
            if (sum == last)
                break;
            last = sum;
        }
    } finally {
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    return overflow ? Integer.MAX_VALUE : size;
}
```





