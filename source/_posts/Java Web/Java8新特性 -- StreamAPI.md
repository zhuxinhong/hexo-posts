---
title: Java8 -- StreamAPI
date: 2016-12-19
categories: Java Web
tags: 
- Java8
---

#	Java8新特性 -- StreamAPI

Stream 是 Java 8 中处理集合的关键抽象概念，它可以指定你希望对集合进行的操作，但是将执行操作的时间交给具体实现来决定。

*	迭代器遍历策略特定，不能并发执行。
*	集合、数组、生成器、迭代器均和创建 Stream。
* 	过滤器 filter 选择元素，map 改变元素。
*  	改变 Stream 的操作包括 limit、distinct 和 sorted。
*	从Stream中获得结果，使用 reduction 操作符，例如 count、max、min、findFirst 或 findAny。其中一些方法返回 Optional 值。
* 	Optional 类型为了安全地替代使用 null 值，需要借助 ifPresent 和 orElse 方法。
*  	Collectors 类的 groupingBy 和 partitioningBy 方法允许你对 Stream 中内容分组。
*	Java 8 对原始类型 int、long 和 double 提供专门的 Stream。
* 	并行 Stream，放弃排序约束。

##	从迭代器到 Stream 操作

处理集合时，通常会迭代所有元素并对其中的每一个进行处理。

```Java
public class Sample1 {

    public static void main(String[] args) {
        List<String> list = letters();
        list.forEach(System.out::println);

        countWord(list);
        filterCountWord(list);
        parallelFilterCountWord(list);

    }

    private static List<String> letters() {
        List<String> list = new ArrayList<>();
        list.add("Tommy");
        list.add("Focus");
        list.add("Tim");
        list.add("Jack");
        return list;
    }

    private static void countWord(List<String> list) {
        int count = 0;
        for (String str : list) {
            if (str.length() > 4)
                count++;
        }
        System.out.println("countWord: " + count);
    }

    private static void filterCountWord(List<String> list) {
        long count = list.stream().filter(w -> w.length() > 4).count();
        System.out.println("filterCountWord: " + count);
    }

    private static void parallelFilterCountWord(List<String> list) {
        long count = list.parallelStream().filter(w -> w.length() > 4).count();
        System.out.println("parallelFilterCountWord: " + count);
    }

}
```

1.	Stream 不存储元素。元素可能被存储在底层的集合中，或根据需要产生出来。
2. 	Stream 操作符不改变源对象。它们返回一个持有结果的新 Stream。
3. 	Stream 操作符可能延迟执行。

##	创建 Stream

###	API

```Java
    /**
     * Returns a sequential {@code Stream} containing a single element.
     *
     * @param t the single element
     * @param <T> the type of stream elements
     * @return a singleton sequential stream
     */
    public static<T> Stream<T> of(T t) {
        return StreamSupport.stream(new Streams.StreamBuilderImpl<>(t), false);
    }
```

```Java
    /**
     * Returns a sequential ordered stream whose elements are the specified values.
     *
     * @param <T> the type of stream elements
     * @param values the elements of the new stream
     * @return the new stream
     */
    @SafeVarargs
    @SuppressWarnings("varargs") // Creating a stream from an array is safe
    public static<T> Stream<T> of(T... values) {
        return Arrays.stream(values);
    }
```

generate 和 iterate 是两个用来创建无限 Stream 的静态方法。

```Java
public class Sample2 {

    public static void main(String[] args) {
        Stream<String> echos = Stream.generate(() -> "Echo");
        Stream<Double> randoms = Stream.generate(Math::random);
        Stream<BigInteger> integers = Stream.iterate(BigInteger.ZERO, n -> n.add(BigInteger.ONE));

        Object[] powers = Stream.iterate(1.0, p -> p * 2).peek(e -> System.out.println("Fetching " + e)).limit(20).toArray();

        System.out.println("well done ...");

    }
}

```

###	filter、map 和 flatMap

流转换指从一个流中读取数据，并将转换后的数据写入另一个流。

```Java
public class Sample3 {

    public static void main(String[] args) {
        List<String> list = letters();
        Stream<String> lowerString = list.stream().map(String::toLowerCase);
        lowerString.forEach(System.out::println);

        Stream<Character> characterStream = list.stream().flatMap(Sample3::characterStream);
        characterStream.forEach(System.out::println);
    }

    private static List<String> letters() {
        List<String> list = new ArrayList<>();
        list.add("Tommy");
        list.add("Focus");
        list.add("Tim");
        list.add("Jack");
        return list;
    }

    public static Stream<Character> characterStream(String s) {
        List<Character> result = new ArrayList<>();
        for (char c : s.toCharArray()) {
            result.add(c);
        }
        return result.stream();
    }
}
```

1.	map 方法，会对每个元素应用一个函数，将返回值收集到一个新的流中。
2. 	flatMap 方法，完成从 T 到 G\<U> 的 f 函数和从 U 到 G\<U> 的 g 方法，这是 monads 理论的一个关键概念。

###	提取子流和组合流

Stream.limit(n) 返回一个 n 个元素的新流（如果原始流的长度小于n，则返回原始的流）。

```Java
public class Sample5 {

    public static void main(String[] args) {
        unique();

        sort();

    }

    public static void unique(){
        Stream<String> uniqueWords = Stream.of("zhuxh", "zhuxh", "zhuxh", "focus").distinct();
        uniqueWords.forEach(System.out::println);
    }

    public static void sort(){
        List<String> list = letters();
        Stream<String> stringStream = list.stream();

        Stream<String> longestFirst = stringStream.sorted(Comparator.comparing(String::length).reversed());
        longestFirst.forEach(System.out::println);

    }

    private static List<String> letters() {
        List<String> list = new ArrayList<>();
        list.add("Tommy");
        list.add("Focus");
        list.add("Tim");
        list.add("Jack");
        return list;
    }
}
```

###	简单的聚合方法

聚合方法都是*终止操作*，当一个流应用了终止操作后，不能再应用其他操作。

聚合方法，count、max、min，返回一个 Optional<T> 值，可能封装返回值，也可能没有返回值。

```Java
public class Sample6 {

    public static void main(String[] args) {
        Stream<String> stringStream = letters().stream();

        Optional<String> largestString = stringStream.max(String::compareToIgnoreCase);

        if(largestString.isPresent()){
            System.out.println(largestString.get());
        }
    }

    private static List<String> letters() {
        List<String> list = new ArrayList<>();
        list.add("Tommy");
        list.add("Focus");
        list.add("Tim");
        list.add("Jack");
        return list;
    }

}
```

类似的API还有 findFirst(), findAny()。

还提供了 anyMatch(), allMatch 和 noneMatch()，这些方法总是会检查整个流，但仍可以通过并行执行来提高速度。

###	Optional 类型

java.util.Optional\<T\>

Optional\<T\> 对象是对一个 T 类型对象的封装，或者表示不是任何对象。它比一般指向 T 类型引用更安全，不会反悔null--前提是你正确使用。

创建 Optional 对象

```Java
    public static <T> Optional<T> of(T value) {
        return new Optional<>(value);
    }
    
    public static <T> Optional<T> ofNullable(T value) {
        return value == null ? empty() : of(value);
    }
    
    public static<T> Optional<T> empty() {
        @SuppressWarnings("unchecked")
        Optional<T> t = (Optional<T>) EMPTY;
        return t;
    }

```

获取值

```Java
   public T get() {
        if (value == null) {
            throw new NoSuchElementException("No value present");
        }
        return value;
    }
```

ifPresent 方法另一种形式可以接受一个函数，如存在可选值，它会将该值传递给函数，否则不进行任何操作。

```Java
    public boolean isPresent() {
        return value != null;
    }
    
    public void ifPresent(Consumer<? super T> consumer) {
        if (value != null)
            consumer.accept(value);
    }
```

####	flatMap

组合可选值函数

```Java
public class MyMath {

    public static Optional<Double> inverse(Double x) {
        return x == 0 ? Optional.empty() : Optional.of(1 / x);
    }

    public static Optional<Double> squareRoot(Double x) {
        return x < 0 ? Optional.empty() : Optional.of(Math.sqrt(x));
    }
}



    public static void main(String[] args) {
        Double x = 1d;

        Optional<Double> result = MyMath.inverse(x).flatMap(MyMath::squareRoot);

        Optional<Double> res2 = Optional.of(-4d).flatMap(MyMath::inverse).flatMap(MyMath::squareRoot);
    }
```

###	聚合

元素求和

```Java
public class Sample2 {

    public static void main(String[] args) {
        List<Integer> intList = new ArrayList<>();
        for (int i = 0; i < 100; i++) {
            intList.add(i);
        }
        Stream<Integer> stream = intList.stream();

        Optional<Integer> sum = stream.reduce(Integer::sum);

        System.out.println(sum.get());
    }
}
```

字符串长度求和

```Java
public class Sample7 {

    public static void main(String[] args) {
        List<String> list = letters();
        
        Stream<String> stream = list.stream();
        int res = stream.reduce(0, (total, word) -> total + word.length(), (t1, t2) -> t1 + t2);
        System.out.println(res);

        int res2 = stream.mapToInt(String::length).sum();
        System.out.println(res2);
    }

    private static List<String> letters() {
        List<String> list = new ArrayList<>();
        list.add("Tommy");
        list.add("Focus");
        list.add("Tim");
        list.add("Jack");
        return list;
    }
}
```

###	收集结果

处理完流之后，收集结果

```Java
public class Sample8 {

    public static void main(String[] args) {
        String[] words = new String[]{"Tommy", "Focus", "Tim", "Jack"};
        Stream<String> stream = Arrays.stream(words);
        HashSet<String> hashSet = stream.collect(HashSet::new, HashSet::add, HashSet::addAll);

        List<String> list = Arrays.stream(words).collect(Collectors.toList());
        Set<String> set = Arrays.stream(words).collect(Collectors.toSet());

        TreeSet<String> treeSet = Arrays.stream(words).collect(Collectors.toCollection(TreeSet::new));

        String strJoin = Arrays.stream(words).collect(Collectors.joining());

        String result = Arrays.stream(words).collect(Collectors.joining(","));

        String result2 = Arrays.stream(words).map(Object::toString).collect(Collectors.joining(","));


        IntSummaryStatistics summary = Arrays.stream(words).collect(Collectors.summarizingInt(String::length));
        double averagwLength = summary.getAverage();
        double maxLength = summary.getMax();
        double min = summary.getMin();
    }
}
```

###	收集结果到 Map

```Java
public class Sample9 {

    public static void main(String[] args) {
        User user1 = new User(1, "Focus");
        User user2 = new User(2, "Tommy");
        User user3 = new User(3, "Jack");

        List<User> list = Arrays.asList(user1, user2, user3);

        s1(list);
        s2(list);
        s3(list);
    }

    /**
     * key重复会抛异常
     **/
    public static void s1(List<User> list) {
        Map<Integer, String> idToName = list.stream().collect(Collectors.toMap(User::getId, User::getName));
        Map<Integer, User> idToUser = list.stream().collect(Collectors.toMap(User::getId, Function.identity()));
    }

    public static void s2(List<User> list) {
        User user = new User(1, "Billy");

        List<User> userList = new ArrayList<>(list);
        userList.add(user);

        Map<Integer, Set<String>> idToName = userList.stream().collect(Collectors.toMap(
                u -> u.getId(),
                u -> Collections.singleton(u.getName()),
                (a, b) -> {
                    Set<String> r = new HashSet<String>(a);
                    r.addAll(b);
                    return r;
                }));
    }

    public static void s3(List<User> list) {
        Map<Integer, User> idToUser = list.stream().collect(
                Collectors.toMap(
                        User::getId,
                        Function.identity(),
                        (existingVal, newVal) -> {
                            throw new IllegalStateException();
                        },
                        TreeMap::new
                )
        );
    }
}
```

###	分组、分片

对流处理结果分组、分片。

```Java
public class User {

    public User(Integer id, String name) {
        this.id = id;
        this.name = name;
        this.email = name + "@qiyi.com";
    }

    public User(Integer id, String name, Integer sex) {
        this(id, name);
        this.sex = sex;
    }

    private Integer id;

    private String name;

    private String email;

    private Integer sex;
    
    ....
}
```

```Java
public class Sample10 {

    public static void main(String[] args) {
        User user1 = new User(1, "Focus", 1);
        User user2 = new User(2, "Tommy", 1);
        User user3 = new User(3, "Jack", 1);

        User user4 = new User(4, "Rose", 2);
        User user5 = new User(5, "Jane", 2);

        List<User> list = Arrays.asList(user1, user2, user3, user4, user5);

        /** 按性别分组**/
        Map<Integer, List<User>> map = list.stream().collect(Collectors.groupingBy(User::getSex));

        /** 按ID小于等于3分组**/
        Map<Boolean, List<User>> map1 = list.stream().collect(Collectors.partitioningBy(e -> e.getId() <= 3));

        /** 按性别分组 收集为set**/
        Map<Integer, Set<User>> map2 = list.stream().collect(Collectors.groupingBy(User::getSex, Collectors.toSet()));
    }
}
```

Java8 还提供了其他收集器，用来对分组后元素进行 downstream 处理：

*	counting 返回收集元素总个数
*	summing (Int|Long|Double) 接受一个函数作为参数，将该函数应用到downstream元素中，求和
* 	maxBy、minBy 接受一个比较器，生成 downstream 元素中最大值和最小值
*  	mapping 方法将一个函数应用到 downstream 结果上，并且需要另一个收集器来处理结果
*	....

###	原始类型流

将每个整数包装成响应对象很低效，Stream API 提供了 IntStream、LongStream 和 DoubleStream 等类型，专门用来直接存储原始类型值。如果要存 short、char、byte 和 boolean 类型，使用 IntStream；如果要存储 float 类型，使用 DoubleStream。设计者认为，不需要为其他5种原始类型都添加对应的专门类型。

将原始类型流转换为一个对象流，可以使用 boxed 方法:

```Java
	Stream<Integer> integers = IntStream.range(0, 100).boxed();
```

原始类型流方法与对象流上调用类似，但有以下几点显著区别：

*	toArray 返回原始类型数组。
* 	产生 Optional 结果的方法返回一个 OptionalInt、OptionalLong 或者 OptionalDouble 类型，这些与 Optional 类似，但没有 get 方法，而是对应的 getAsInt、getAsLong 和 getAsDouble。
*	方法 sum、average、max 和 min 会返回总和、平均值、最大值和最小值。对象流中没有定义这些方法。
* 	summaryStatistics 方法产生一个 IntSummaryStatistics、LongSummaryStatistics 或者 DoubleSummaryStatistics 对象。  

###	并行流

流使得并行计算变得容易。处理过程几乎全自动。但必须遵守一些约定。

*	创建并行流， Collection.parellelStream, parallel 方法。
* 	确保传递给并行流操作的函数都是线程安全的。

有序并不妨碍并行。例如，计算 stream.map(fun) 时，流可以被分片为 n 段，每一段都被并发处理，然后按顺序组合结果。

执行一个流操作时，不会修改流底层的集合（即使修改是线程安全）。流不会收集它自己的数据----这些数据总是存在于另一个集合中。如果你想修改原有集合，那么就无法定义流操作的输出。JDK 文档称这种需求为“不干扰”。

```Java
	//正确
	List<String> wordList = ...;
	Stream<String> words = wordList.stream();
	wordList.add("END");
	long n = words.distinct().count();
```

```Java
	//错误
	Stream<String> words = wordList.stream();
	words.forEach(s -> if (s.length() < 12) wordList.remove(s));
```