## ArrayList
	
### 概述
因为 ArrayList 是基于数组实现的，所以支持快速随机访问。RandomAccess 接口标识着该类支持快速随机访问。
```java
public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```
数组的默认大小为 10。
```java
private static final int DEFAULT_CAPACITY = 10;
		
transient Object[] elementData;//non-private to simplify nested class access

private int size;

private static final Object[] EMPTY_ELEMENTDATA={};
		
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA={};
```
如果新建实例时传入0则是前者，没传入参数则是后者，定义了两个空数组，区别在于第一次向list中加入元素的操作。如果是``DEFAULTCAPACITY_EMPTY_ELEMENTDATA``，则数组大小更新为10，如果是``EMPTY_ELEMENTDATA``，则为1。
```java
public ArrayList (int initialCapacity){
	if(initialCapacity>0){
		this.elementData=newObject[initialCapacity];
	}else if(initialCapacity==0){
		this.elementData=EMPTY_ELEMENTDATA;
	}else{
		throw new IllegalArgumentException("IllegalCapacity:"+
		initialCapacity);
	}
}
public ArrayList(){
	this.elementData=DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```
### 添加元素与扩容
添加元素时使用 ``ensureCapacityInternal()``方法来保证容量足够，如果不够时，需要使用 ``grow()`` 方法进行扩容，新容量的大小为 ``oldCapacity + (oldCapacity >> 1)``，也就是旧容量的 1.5 倍。

扩容操作需要调用`` Arrays.copyOf() ``把原数组整个复制到新数组中，这个操作代价很高，因此最好在创建 ``ArrayList`` 对象时就指定大概的容量大小，减少扩容操作的次数。
<p align="center">
<img src="https://github.com/Qirui0805/Personal-Blog/blob/master/ArrayList%E6%B7%BB%E5%8A%A0%E6%89%A9%E5%AE%B9.png?raw=true" width="300" />
</p>

添加元素
```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```
添加collection
```java
public boolean addAll(Collection<?extendsE> c){
	Object[] a=c.toArray();
	int numNew=a.length;
	ensureCapacityInternal(size+numNew);//IncrementsmodCount
	System.arraycopy(a,0,elementData,size,numNew);
	size+=numNew;
	return numNew!=0;
}
```
计算所需容量
```java
private static int calculateCapacity(Object[] elementData,int minCapacity){
	if(elementData==DEFAULTCAPACITY_EMPTY_ELEMENTDATA){
		return Math.max(DEFAULT_CAPACITY,minCapacity);  //有可能加一整个list进来
	}
	return minCapacity;
}
```
确定capacity
```java
private void ensureCapacityInternal(int minCapacity){
	ensureExplicitCapacity(calculateCapacity(elementData,minCapacity));
}
```
执行扩容操作
```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
    grow(minCapacity);
}
```
```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
    newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
    newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```
		
### 删除元素
需要调用 ``System.arraycopy()`` 将 ``index+1`` 后面的元素都复制到 ``index`` 位置上，该操作的时间复杂度为 O(N)，可以看出 ``ArrayList`` 删除元素的代价是非常高的。如果是同一个数组，``System.arraycopy()``会创建一个临时数组存储数据，再将数据复制回数组
```java
public E remove(int index) {
    rangeCheck(index);
    modCount++;
    E oldValue = elementData(index);
    int numMoved = size - index - 1;
    if (numMoved > 0)
    System.arraycopy(elementData, index+1, elementData, index, numMoved);
    elementData[--size] = null; // clear to let GC do its work
    return oldValue;
}
```
		
### 迭代时删除
直接调用remove会导致``concurrentexception``，因为remove后modcount会加1，必须调用``iterator.remove()``
```java
public void remove(){
	if (lastRet<0)
		throw new IllegalStateException();
		checkForComodification();
	try {
		ArrayList.this.remove(lastRet);
		cursor=lastRet;
		lastRet=-1;
		expectedModCount=modCount;
		} catch (IndexOutOfBoundsExceptionex){
			throw new ConcurrentModificationException();
	}
}
```
### java list循环中删除元素的坑 
		
#### Fail-Fast
``modCount`` 用来记录 ``ArrayList`` 结构发生变化的次数。结构发生变化是指添加或者删除至少一个元素的所有操作，或者是调整内部数组的大小，仅仅只是设置元素的值不算结构发生变化。
``ArrayList``, ``HashMap``等都不是线程安全的，在进行序列化或者迭代等操作时，需要比较操作前后 modCount 是否改变，如果改变了需要抛出 ``ConcurrentModificationException``。
```java
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException{
    // Write out element count, and any hidden stuff
    int expectedModCount = modCount;
    s.defaultWriteObject();
		// Write out size as capacity for behavioural compatibility with clone()
    s.writeInt(size);
		// Write out all elements in the proper order.
    for (int i=0; i<size; i++) {
        s.writeObject(elementData[i]);
    }
		if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}
```
```java
private class Itr implementsIterator<E>{
	int cursor;//indexofnextelementtoreturn
	int lastRet=-1;//indexoflastelementreturned;-1ifnosuch
	int expectedModCount=modCount;
	
	public boolean hasNext(){
		return cursor!=size;
	}
	@SuppressWarnings("unchecked")
	public E next(){
		checkForComodification();
		int i=cursor;
		if(i>=size)
			throw new NoSuchElementException();
		Object[] elementData=ArrayList.this.elementData;
		if(i>=elementData.length)
			throw new ConcurrentModificationException();
		cursor=i+1;
		return (E)elementData[lastRet=i];
	}
	
	final void checkForComodification(){
		if(modCount!=expectedModCount)
			throw new ConcurrentModificationException();
		}
	}
```
## Vector
``ArrayList``和``Vector``干的事差不多，实现思路不一样，绝对是两个人写的。
与``ArrayList``相似与区别：
			1. Vector是线程安全的集合类，ArrayList并不是线程安全的类。Vector类对集合的元素操作时都加了synchronized，保证线程安全。因此开销就比 ArrayList 要大，访问速度更慢。最好使用 ArrayList 而不是 Vector，因为同步操作完全可以由程序员自己来控制；
			2. Vector与ArrayList本质上都是一个Object[] 数组，ArrayList提供了size属性，Vector提供了elementCount属性，他们的作用是记录集合内有效元素的个数。与我们平常调用的arrayList.size()和vector.size()一样返回的集合内有效元素的个数。
			3. Vector与ArrayList的扩容并不一样，Vector默认扩容是增长一倍的容量，Arraylist是增长50%的容量。
			4. Vector与ArrayList的remove,add(index,obj)方法都会导致内部数组进行数据拷贝的操作，这样在大数据量时，可能会影响效率。
			5. Vector与ArrayList的add(obj)方法，如果新增的有效元素个数超过数组本身的长度，都会导致数组进行扩容。
```java
		public class Vector<E> extendsAbstractList<E> implements List<E>,RandomAccess,Cloneable,java.io.Serializable
		
		protected Object[] elementData;
		
		protected int elementCount;
		//每次扩充的值
		protected int capacityIncrement;
```
### 构造函数
如果没有传入``capacityIncrement``参数，则每次扩充时默认两倍。``ArrayList``弄了final字段比如``DEFAULT_CAPACITY``, ``EMPTY_ELEMENT``，而且在第一次添加元素时才初始化``DEFAULT_EMPTY_ELEMENT``, 但vector就没这么复杂，达到的目的都一样。不过``ArrayList``延后初始化有一个好处就是如果新的list没有用到则不会占用空间。
```java
public Vector(intinitialCapacity,intcapacityIncrement){
	super();
	if(initialCapacity<0)
		throw new IllegalArgumentException("IllegalCapacity:"+initialCapacity);
	this.elementData=new Object[initialCapacity];
	this.capacityIncrement=capacityIncrement;
}
public Vector(int initialCapacity){
	this(initialCapacity,0);
}
public Vector(){
	this(10);
}
public Vector(Collection<?extendsE>c){
	elementData=c.toArray();
	elementCount=elementData.length;
	//c.toArraymight(incorrectly)notreturnObject[](see6260652)
	if(elementData.getClass()!=Object[].class)
		elementData=Arrays.copyOf(elementData,elementCount,Object[].class);
}
```
### 添加
		
它的实现与 ``ArrayList`` 类似，但是使用了 synchronized 进行同步。
```java
public synchronized boolean add(E e) {
    modCount++;
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = e;
    return true;
    }
public synchronized E get(int index) {
    if (index >= elementCount)
    throw new ArrayIndexOutOfBoundsException(index);
    return elementData(index);
}
```
		
### 扩容
		
Vector 的构造函数可以传入 capacityIncrement 参数，它的作用是在扩容时使容量 capacity 增长 capacityIncrement。如果这个参数的值小于等于 0，扩容时每次都令 capacity 为原来的两倍。
```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                     capacityIncrement : oldCapacity);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```
替代方案
		
可以使用 ``Collections.synchronizedList();`` 得到一个线程安全的 ``ArrayList``。
```java
List<String> list = new ArrayList<>();
List<String> synList = Collections.synchronizedList(list);
```
也可以使用 concurrent 并发包下的 ``CopyOnWriteArrayList`` 类。
```java
List<String> list = new CopyOnWriteArrayList<>();
```
	
## CopyOnWriteArrayList
又是另一个人写的，没有size字段，也就不能在构造时设定大小，只有默认构造函数和以collector为参数的构造函数，
```java
private transient volatile Object[] array;

public CopyOnWriteArrayList(Collection<?extendsE> c){
	Object[] elements;
	if(c.getClass()==CopyOnWriteArrayList.class)
		elements=((CopyOnWriteArrayList<?>)c).getArray();
	else{
		elements=c.toArray();
		//c.toArray might(incorrectly)not return Object[](see6260652)
		if(elements.getClass()!=Object[].class)
		elements=Arrays.copyOf(elements,elements.length,Object[].class);
	}
	setArray(elements);
}
```
### 读写分离
		
- 写操作在一个复制的数组上进行，读操作还是在原始数组中进行，读写分离，互不影响。
- 写操作需要加锁，防止并发写入时导致写入数据丢失。
- 写操作结束之后需要把原始数组指向新的复制数组。因此也没有扩容操作。
```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);//复制数组
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```
```java
final void setArray(Object[] a) {
    array = a;
}
		@SuppressWarnings("unchecked")
private E get(Object[] a, int index) {
    return (E) a[index];
}
```		
### 适用场景
		
``CopyOnWriteArrayList`` 在写操作的同时允许读操作，大大提高了读操作的性能，因此很适合读多写少的应用场景。
		但是 CopyOnWriteArrayList 有其缺陷：
			• 内存占用：在写操作时需要复制一个新的数组，使得内存占用为原来的两倍左右；
			• 数据不一致：读操作不能读取实时性的数据，因为部分写操作的数据还未同步到读数组中。
		所以 CopyOnWriteArrayList 不适合内存敏感以及对实时性要求很高的场景。
		
## LinkedList
	
基于双向链表实现，使用 Node 存储链表节点信息。同样线程不安全
```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
}
```
每个链表存储了 first 和 last 指针：
```java
transient Node<E> first;
transient Node<E> last;
```
新Node的pre为空，next同first, first指向新Node，若list一开始不为空，则原首Node指向新Node
```java
		privatevoidlinkFirst(E e){
			final Node<E> f=first; 
			final Node<E> newNode=newNode<>(null,e,f);
			first=newNode;
			if(f==null)
				last=newNode;
			else
				f.prev=newNode;
			size++;
			modCount++;
		}
```
		默认添加到最后一个
```java
		public boolean add(Ee){
			linkLast(e);
			return true;
		}
```
### 与 ArrayList 的比较
• ArrayList 基于动态数组实现，LinkedList 基于双向链表实现；
• ArrayList 支持随机访问，LinkedList 不支持；
• LinkedList 在任意位置添加删除元素更快。
