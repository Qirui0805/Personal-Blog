### Default 关键字
JAVA8 引进，用于修饰Interface的方法, 对于经此修饰的方法可以在接口中实现该方法，接口的实现类可无需实现该方法。

### 函数式接口

函数式接口在Java中是指：有且仅有一个抽象方法的接口。

### @Functional Interface注解
与@Override 注解的作用类似，该注解可用于一个接口的定义上：一旦使用该注解来定义接口，编译器将会强制检查该接口是否确实有且仅有一个抽象方法，否则将会报错。需要注意的是，即使不使用该注解，只要满足函数式接口的定义，这仍然是一个函数式接口，使用起来都一样。

### Lambda表达式：
（Type parameter）-> do something
或：（parameter）-> do something（只有一个参数时省略类型）
 相当于：
	new functionalInterface() {
		@Override
		public void function(Type parameter){
			do something;
		}
	}

如Runnable为函数是接口，run()

### 使用Lambda作为参数和返回值
- 作为参数 略
- 作为返回值：函数的返回值是一个函数式接口 
```java
	public class Demo06Comparator {
		private static Comparator<String> newComparator() {
			return (a, b) ‐> b.length() ‐ a.length();
		}
		public static void main(String[] args) {
			String[] array = { "abc", "ab", "abcd" };
			System.out.println(Arrays.toString(array));
			Arrays.sort(array, newComparator());
			System.out.println(Arrays.toString(array));
		}
	}
```
### Consumer接口
Consumer 接口中包含抽象方法void accept(T t) ，意为消费一个指定泛型的数据。
```java
public class Demo09Consumer {
	private static void consumeString(Consumer<String> function) {
		function.accept("Hello");
	}
	public static void main(String[] args) {
		consumeString(s ‐> System.out.println(s));
	}
}
```
默认方法：andThen
```java
	default Consumer<T> andThen(Consumer<? super T> after) {
		Objects.requireNonNull(after);
		return (T t) ‐> { accept(t); after.accept(t); };
	}
	
	public class Demo10ConsumerAndThen {
		private static void consumeString(Consumer<String> one, Consumer<String> two) {
		one.andThen(two).accept("Hello");
	}
	public static void main(String[] args) {
		consumeString(
			s ‐> System.out.println(s.toUpperCase()),
			s ‐> System.out.println(s.toLowerCase()));
		}
	}
```
### Predicate接口
有时候我们需要对某种类型的数据进行判断，从而得到一个boolean值结果。这时可以使用``java.util.function.Predicate<T>`` 接口。

抽象方法：test
```java
	boolean test(T t)
	private static void method(Predicate<String> predicate) {
		boolean veryLong = predicate.test("HelloWorld");
		System.out.println("字符串很长吗：" + veryLong);
	}
	public static void main(String[] args) {
		method(s ‐> s.length() > 5);
	}
```
默认方法： and, or, negate
	将连个Predicate的判断连起来
	 
- and
```java
	default Predicate<T> and(Predicate<? super T> other) {
		Objects.requireNonNull(other);
		return (t) ‐> test(t) && other.test(t);
	}
	
	public class Demo16PredicateAnd {
		private static void method(Predicate<String> one, Predicate<String> two) {
			boolean isValid = one.and(two).test("Helloworld");
			System.out.println("字符串符合要求吗：" + isValid);
		}
		public static void main(String[] args) {
			method(s ‐> s.contains("H"), s ‐> s.contains("W"));
		}
	}
```
- or
```java
	default Predicate<T> or(Predicate<? super T> other) {
		Objects.requireNonNull(other);
		return (t) ‐> test(t) || other.test(t);
	}
	
	public class Demo16PredicateAnd {
		private static void method(Predicate<String> one, Predicate<String> two) {
			boolean isValid = one.or(two).test("Helloworld");
			System.out.println("字符串符合要求吗：" + isValid);
		}
		public static void main(String[] args) {
			method(s ‐> s.contains("H"), s ‐> s.contains("W"));
		}
	}
```	
- negate
```java
	default Predicate<T> negate() {
		return (t) ‐> !test(t);
	}
	
	public class DemoPredicate {
		public static void main(String[] args) {
			String[] array = { "迪丽热巴,女", "古力娜扎,女", "马尔扎哈,男", "赵丽颖,女" };
			List<String> list = filter(array,
							s ‐> "女".equals(s.split(",")[1]),
							s ‐> s.split(",")[0].length() == 4);
			System.out.println(list);
		}
		private static List<String> filter(String[] array, Predicate<String> one,
			Predicate<String> two) {
			List<String> list = new ArrayList<>();
			for (String info : array) {
				if (one.and(two).test(info)) {
					list.add(info);
				}
			}
			return list;
		}
	}
```	
### Function接口
``java.util.function.Function<T,R>`` 接口用来根据一个类型的数据得到另一个类型的数据，前者称为前置条件，后者称为后置条件。

抽象方法：apply
```java
	R apply(T t)
	private static void method(Function<String, Integer> function) {
		int num = function.apply("10");
		System.out.println(num + 20);
	}
	public static void main(String[] args) {
		method(s ‐> Integer.parseInt(s));
	}
```
默认方法：andThen
```java
	default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
		Objects.requireNonNull(after);
		return (T t) ‐> after.apply(apply(t));
	}
	
	public class Demo12FunctionAndThen {
		private static void method(Function<String, Integer> one, Function<Integer, Integer> two) {
			int num = one.andThen(two).apply("10");
			System.out.println(num + 20);
		}
		public static void main(String[] args) {
			method(str‐>Integer.parseInt(str)+10, i ‐> i *= 10);
		}
	}
```
	
### Stream流
Stream（流）是一个来自数据源的元素队列
○ 元素是特定类型的对象，形成一个队列。 Java中的Stream并不会存储元素，而是按需计算。
○ 数据源 	流的来源。 可以是集合，数组 等。

和以前的Collection操作不同， Stream操作还有两个基础的特征：
○ Pipelining: 中间操作都会返回流对象本身。 这样多个操作可以串联成一个管道， 如同流式风格（fluentstyle）。 这样做可以对操作进行优化， 比如延迟执行(laziness)和短路( shortcircuiting)。
○ 内部迭代： 以前对集合遍历都是通过Iterator或者增强for的方式, 显式的在集合外部进行迭代， 这叫做外部迭代。 Stream提供了内部迭代的方式，流可以直接调用遍历方法。

当使用一个流的时候，通常包括三个基本步骤：获取一个数据源（source）→ 数据转换→执行操作获取想要的结果，每次转换原有 Stream 对象不改变，返回一个新的 Stream 对象（可以有多次转换），这就允许对其操作可以像链条一样排列，变成一个管道。

#### 获取流
java.util.stream.Stream<T> 是Java 8新加入的最常用的流接口。（这并不是一个函数式接口。）
获取一个流有以下几种常用的方式：
○ 所有的Collection 集合都可以通过stream 默认方法获取流；
```java
	List<String> list = new ArrayList<>();
	// ...
	Stream<String> stream1 = list.stream();
	Set<String> set = new HashSet<>();
	// ...
	Stream<String> stream2 = set.stream();
	Vector<String> vector = new Vector<>();
```
○ java.util.Map 接口不是Collection 的子接口，且其K-V数据结构不符合流元素的单一特征，所以获取对应的流需要分key、value或entry等情况：
```java
		Map<String, String> map = new HashMap<>();
		// ...
		Stream<String> keyStream = map.keySet().stream();
		Stream<String> valueStream = map.values().stream();
		Stream<Map.Entry<String, String>> entryStream = map.entrySet().stream();
```	
○ Stream 接口的静态方法of 可以获取数组对应的流。
```java
		String[] array = { "张无忌", "张翠山", "张三丰", "张一元" };
		Stream<String> stream = Stream.of(array);
```
#### 常用方法
流模型的操作很丰富，这里介绍一些常用的API。这些方法可以被分成两种：
- 延迟方法：返回值类型仍然是Stream 接口自身类型的方法，因此支持链式调用。（除了终结方法外，其余方法均为延迟方法。）
终结方法：返回值类型不再是Stream 接口自身类型的方法，因此不再支持类似StringBuilder 那样的链式调用。本小节中，终结方法包括count 和forEach 方法。

- 逐一处理：forEach
虽然方法名字叫forEach ，但是与for循环中的“for-each”昵称不同。
``void forEach(Consumer<? super T> action);``
该方法接收一个Consumer 接口函数，会将每一个流元素交给该函数进行处理。
```java
public static void main(String[] args) {
	Stream<String> stream = Stream.of("张无忌", "张三丰", "周芷若");
	stream.forEach(name‐> System.out.println(name));
	}
```
- 过滤：filter
可以通过filter 方法将一个流转换成另一个子集流。方法签名：
``Stream<T> filter(Predicate<? super T> predicate);``
该接口接收一个Predicate 函数式接口参数（可以是一个Lambda或方法引用）作为筛选条件。
```java
public class Demo07StreamFilter {
	public static void main(String[] args) {
		Stream<String> original = Stream.of("张无忌", "张三丰", "周芷若");
		Stream<String> result = original.filter(s ‐> s.startsWith("张"));
	}
}
```
- 映射：map
如果需要将流中的元素映射到另一个流中，可以使用map 方法。方法签名：
``<R> Stream<R> map(Function<? super T, ? extends R> mapper);``
该接口需要一个Function 函数式接口参数，可以将当前流中的T类型数据转换为另一种R类型的流
```java
public class Demo08StreamMap {
	public static void main(String[] args) {
		Stream<String> original = Stream.of("10", "12", "18");
		Stream<Integer> result = original.map(str‐>Integer.parseInt(str));
	}
}
```
- 个数：count
	正如旧集合Collection 当中的size 方法一样，流提供count 方法来数一数其中的元素个数：
		long count();
```java
	public class Demo09StreamCount {
		public static void main(String[] args) {
			Stream<String> original = Stream.of("张无忌", "张三丰", "周芷若");
			Stream<String> result = original.filter(s ‐> s.startsWith("张"));
			System.out.println(result.count()); // 2
		}
	}
```
- 取用前几个：limit
limit 方法可以对流进行截取，只取用前n个。方法签名：
		``Stream<T> limit(long maxSize);``
```java	
	public class Demo10StreamLimit {
		public static void main(String[] args) {
			Stream<String> original = Stream.of("张无忌", "张三丰", "周芷若");
			Stream<String> result = original.limit(2);
			System.out.println(result.count()); // 2
		}
	}
```
- 跳过前几个：skip
如果希望跳过前几个元素，可以使用skip 方法获取一个截取之后的新流：
	``Stream<T> skip(long n);``
如果流的当前长度大于n，则跳过前n个；否则将会得到一个长度为0的空流。基本使用：
```java
	public class Demo11StreamSkip {
		public static void main(String[] args) {
			Stream<String> original = Stream.of("张无忌", "张三丰", "周芷若");
			Stream<String> result = original.skip(2);
			System.out.println(result.count()); // 1
		}
	}
```
- 组合：concat
如果有两个流，希望合并成为一个流，那么可以使用Stream 接口的静态方法concat ：
○ 这是一个静态方法，与java.lang.String 当中的concat 方法是不同的。
```java
public class Demo12StreamConcat {
	public static void main(String[] args) {
		Stream<String> streamA = Stream.of("张无忌");
		Stream<String> streamB = Stream.of("张翠山");
		Stream<String> result = Stream.concat(streamA, streamB);
	}
}
```
- 规约操作
reduce 略

- collect
	<R,A> R collect(Collector<? super T,A,R> collector)
	
	List<String> asList = stringStream.collect(Collectors.toList());
- 方法引用
```java
	@FunctionalInterface
	public interface Printable {
		void print(String str);
	}
	
	public class Demo01PrintSimple {
		private static void printString(Printable data) {
			data.print("Hello, World!");
		}
		public static void main(String[] args) {
			printString(s ‐> System.out.println(s));
		}
		
		public static void main(String[] args) {
			printString(System.out::println);
		}
	}
```
- 双冒号:: 为引用运算符
	
语义分析
例如上例中， System.out 对象中有一个重载的println(String) 方法恰好就是我们所需要的。那么对于printString 方法的函数式接口参数，对比下面两种写法，完全等效：

- Lambda表达式写法： s -> System.out.println(s);
- 方法引用写法： System.out::println
- 
第一种语义是指：拿到参数之后经Lambda之手，继而传递给System.out.println 方法去处理。
第二种等效写法的语义是指：直接让System.out 中的println

方法来取代Lambda。两种写法的执行效果完全一样，而第二种方法引用的写法复用了已有方案，更加简洁。
注:Lambda 中 传递的参数 一定是方法引用中 的那个方法可以接收的类型,否则会抛出异常

- 通过对象名引用成员方法
```java
public static void main(String[] args) {
	MethodRefObject obj = new MethodRefObject();
	printString(obj::printUpperCase);
}
```
- 通过类名称引用静态方法
```java
public static void main(String[] args) {
	method(‐10, Math::abs);
}
- 通过super引用成员方法
```java
public void show(){
    method(super::sayHello);
}
```

		
	


		

