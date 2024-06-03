# Stream 类

## 概述

### 作用

1. 处理集合的关键抽象概念；对集合进行操作，可以执行非常复杂的查找，过滤和映射数据等操作；
2. 对集合数据进行操作，类似于SQL的数据库查询；

#### 注

1. 自己**不会存储**元素；
2. 不会改变原对象，会返回一个新的对象；
3. 操作是延迟执行的，就意味着它们会等到需要结果时执行；

#### 区别

Stream和Collection的区别，Collection是一种静态数据结构，主要面向内存，存储在内存中；而Stream是有关计算的，面向CPU，通过CPU计算的；

## 操作

### 实例化

方式一：通过集合

```java
List list=new ArrayList();
Stream stream1=list.stream();//返回一个顺序流：按照集合的顺序读取数据
Stream stream2=list.parallelStream();//返回一个并行流
```

方式二：通过数组

`Array.stream(array,from,to)` 

```java
int[] arr = new int[]{1, 2, 3, 4, 5};
IntStream stream = Arrays.stream(arr);
```

方式三：Stream.of

`Stream.empty()` 

```java
Stream.of(1,2,3,4,5);
```

方式四：创建无限流

```java
//例子：遍历输出前十个偶数
//外部迭代
Stream.iterate(0, t -> t + 2).limit(10).forEach(System.out::println);
Stream.generate(Math::random).limit(10);
```

`Stream<T> iterate(final T seed, final UnaryOperator<T> f)` 

第一个参数：种子值；

第二个参数：函数

### 中间操作

一个中间操作链，对数据源的数据进行处理；

惰性求值：终止操作时一次性全部处理；

**筛选和切片**

1. `filter`从流中筛选出某些元素；

```java
    	int[] arr = new int[]{1, 2, 3, 4, 5};
        IntStream stream = Arrays.stream(arr);
        stream.filter(value -> value > 3).forEach(System.out::println);
```

2. `limit(int n)`截断流，使其元素不超过给定数量；
3. `skip(int n)`返回一个丢弃前n个数据的流，若数据不足n个，返回一个空流；
4. `distinct()`筛选，通过流所生成元素的`hashcode()`和`equal()`去除重复元素；
5. `concat()` 连接两个流；

映射

1. `map(Function f)`接收一个函数作为参数，将元素转化为其他信息或提取信息；

```java
public class Main {
    public static void main(String[] args) {
        List<String> list = Arrays.asList("aa", "bb", "cc");
        list.stream().map(String::toUpperCase).forEach(System.out::println);
    }
}
```

2. `flatMap(Function f)`接收一个函数作为参数，将流中的每一个值转化为另一个流，然后把所有流连接成一个流；

```java
public class Main {
    public static void main(String[] args) {
        List<String> list = Arrays.asList("aa", "bb", "cc");
        //两次forEach()
        Stream<Stream<Character>> streamStream = list.stream().map(Main::StringToStream);
        streamStream.forEach(s->s.forEach(System.out::println));
        //flatMap()
        list.stream().flatMap(Main::StringToStream).forEach(System.out::println);
    }

    public static Stream<Character> StringToStream(String str) {
        ArrayList<Character> list = new ArrayList();
        for (Character c : str.toCharArray()) {
            list.add(c);
        }
        return list.stream();
    }
}
```

#### 排序

`sorted()`自然排序；

`sorted(Comparator com)` 定制排序；

### 终止操作

一旦执行终止操作，就执行中间操作链（延时），并产生结果，之后**不会**再被使用；

#### 匹配与查找

`allMatch(Predict p)` 检查是否匹配所有元素

```Java
//验证所有数字是否大于0
public class Main {
    public static void main(String[] args) {
        List<Integer> list = List.of(1, 2, 3, 4);
        System.out.println(list.stream().allMatch(integer -> integer > 0));
    }
}
```

`anyMatch(Predict p)` 是否有一个元素匹配

`noneMatch(Predict p)` 检查是否有没有匹配的元素

`findFirst()` 返回第一个元素

`findAny()` 返回任何元素

`count()` 返回流中元素的数量

`max(Comparator c)` 

`min(Comparator c)` 

`forEach(Consumer c)` 内部迭代

#### 归约

`reduce()` 可以将流中元素反复结合起来，得到一个值；

**方法一：`Optional<T> reduce(BinaryOperator<T> accumulator);` **

**BinaryOperator**是BiFunction的一个特殊例子，接收两个参数产生一个结果；

```java
//BinaryOperator源码
public interface BinaryOperator<T> extends BiFunction<T,T,T> {
    public static <T> BinaryOperator<T> minBy(Comparator<? super T> comparator) {
        Objects.requireNonNull(comparator);
        return (a, b) -> comparator.compare(a, b) <= 0 ? a : b;
    }
    
    public static <T> BinaryOperator<T> maxBy(Comparator<? super T> comparator) {
        Objects.requireNonNull(comparator);
        return (a, b) -> comparator.compare(a, b) >= 0 ? a : b;
    }
}
```

```Java
//BiFunction源码
public interface BiFunction<T, U, R> {
    R apply(T t, U u);
	//接收两个参数，返回R
    default <V> BiFunction<T, U, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t, U u) -> after.apply(apply(t, u));
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        List<Integer> list = List.of(23, 11, 2, 4, 5);
        System.out.println(list.stream().reduce((integer, integer2) -> integer + integer2));//返回stream的和
    }
}
```

`reduce()` 含有两个参数，第一个是上一次函数计算获得的返回值（reduce的返回结果会自动作为作为下一次累加器的参数），第二个参数是stream中的元素，最后得到一个`Optional` 对象；

**方法二：`T reduce(T identity, BinaryOperator<T> accumulator)`**

**identity**参数与stream中的数据类型相同，相当于一个初始值，累加器按照这个初始值迭代计算stream中的数据

**方法三：`\<U\> U reduce(U identity,BiFunction\<U, ? super T, U\> accumulator,BinaryOperator\<U\> combiner)`**

第一个参数：初始化实例；

第二个参数：累加器`accumulator` ，遍历实例及其元素；

第三个参数：参数组合器`combiner` ，其参数为返回的数据类型，Stream是支持并发操作的，为了避免竞争，对于reduce线程都会有独立的result，combiner的作用在于合并每个线程的result得到最终结果。

```java
public class Main {
    public static void main(String[] args) {
        List<Integer> list = List.of(23, 11, 2, 4, 5);
        list.stream().reduce(new ArrayList<>(), new BiFunction<List<Integer>, Integer, List<Integer>>() {
            @Override
            public List<Integer> apply(List<Integer> integers, Integer integer) {
                integers.add(integer);
                System.out.println("BiFunction");
                System.out.println(integer);
                return integers;
            }
        }, new BinaryOperator<List<Integer>>() {
            @Override
            public List<Integer> apply(List<Integer> integers, List<Integer> integers2) {
                integers.addAll(integers2);
                System.out.println("BinaryOperator");
                System.out.println(integers2);
                return integers;
            }
        });
    }
}
//BiFunction
//23
//BiFunction
//11
//BiFunction
//2
//BiFunction
//4
//BiFunction
//5
```

#### 收集

`collect(Collector c)` 将流转化为其他形式，Collector用于给Stream中的元素做汇总，实现了如何对流执行收集的操作（如收集到`list` ,`Map` ,`Set` 中）；

**Collectors**

`Collectors.toList()` 

`Collectors.counting()` 计算流中的个数

`Collectors.toMap()` 产生一个收集器，它会产生一个映射表或者并发映射表，keyMapper和valueMapper函数会应用到每一个收集到的元素身上，从而在表中产生一个键值项，默认情况下，当两个元素键值一样时，会抛出异常，可以提供一个mergeFunction来合并具有相同键值的项，默认情况下返回HashMap或者ConcurrentHashMap；

`Collectors.groupingBy()` 将相同特性的群组聚集在一起；

```Java
public class Main {
    public static void main(String[] args) {
        Stream<Locale> availableLocales = Stream.of(Locale.getAvailableLocales());
        Map<String, List<Locale>> collect = availableLocales.collect(Collectors.groupingBy(Locale::getDisplayCountry));
        List<Locale> en = collect.get("en");
    }
}
```

`Collectors.partitioningly()` 流的元素可以区分为两个列表，该函数返回为true或false的部分；

# Optional 类

`Optional<T>类` 为容器类，可以保存类型T的值，表示这个值存在，或者仅仅保存null的值，表示这个值不存在，可以避免空指针异常；

### 创建实例

`Optional.of(T t)` 创建一个Optional实例，t必须**非空**；

`Optional.empty()` 创建一个空的Optional实例；

`Optional.ofNullable(T t)` t可以为null；

### 判断Optional实例中是否包含对象

`boolean isPresent()` 

`void ifPresent(Consumer<? super T> consumer)` 如果有值，就执行Consumer的接口实现代码，并且值会作为参数传递给函数；

### 获取Optional容器的对象

`T get()` 

`T orElse(T other)`如果有值就返回，没有则返回指定的other对象；

`T orElseGet(Supplier<? extends T> other)`  如果有值就将其返回，没有则返回实现Supplier接口的对象；

`T orElseThrow(Supplier<? extends X> exceptionSupplier)` 

`<U> Optional map(Function<? super T,? extends <U>> mapper)` 类似于stream的map方法，将可选值想象为0和1的流，结果自然也是0，1或null；