+++
title = "Lambda表达式，方法引用和StreamApi"
date = "2023-06-02T13:38:47+08:00"
tags = []
slug = ""
+++

# 简介

>在Java世界里面，面向对象还是主流思想，对于习惯了面向对象编程的开发者来说，抽象的概念并不陌生。面向对象编程是对数据进行抽象，而函数式编程是对行为进行抽象。现实世界中，数据和行为并存，程序也是如此，因此这两种编程方式我们都得学。
>
>这种新的抽象方式还有其他好处。很多人不总是在编写性能优先的代码，对于这些人来说，函数式编程带来的好处尤为明显。程序员能编写出更容易阅读的代码——这种代码更多地表达了业务逻辑，而不是从机制上如何实现。易读的代码也易于维护、更可靠、更不容易出错。

面向对象编程是==对数据进行抽象==；函数式编程是==对行为进行抽象==。
核心思想: 使用不可变值和函数，函数对一个值进行处理，映射成另一个值。
对核心类库的改进主要包括集合类的API和新引入的流Stream。流使程序员可以站在更高的抽象层次上对集合进行操作。

# lambda表达式

 lambda表达式仅能放入如下代码: 预定义使用了 `@Functional` 注释的==函数式接口==，==自带一个抽象函数的方法==，或者==SAM(Single Abstract Method 单个抽象方法)类型==。这些称为lambda表达式的目标类型，可以用作返回类型，或lambda目标代码的参数。例如，若一个方法接收Runnable、Comparable或者 Callable 接口，都有单个抽象方法，可以传入lambda表达式。类似的，如果一个方法接受声明于 java.util.function 包内的接口，例如 Predicate、Function、Consumer 或 Supplier，那么可以向其传lambda表达式。

- **函数式接口**:
    来自Core Java：
    对于只有一个抽象方法的接口，需要这种接口的对象时，就可以提供一个lambda表达式。这种接口称之为`函数式接口`。
    函数式接口在java中是指:**有且仅有一个抽象方法的接口**
    函数式接口，即适用于函数式编程场景的接口。而java中的函数式编程体现就是Lambda，所以函数式接口就是可以适用于Lambda使用的接口。只有确保接口中有且仅有一个抽象方法，Java中的Lambda才能顺利地进行推导。

    >**@FunctionalInterface注解**

```java
@FunctionalInterface // 标明为函数式接口
public abstract MyFunctionInterface{
    void mrthod(); //抽象方法
}
```

一旦使用该注解来定义接口，编译器将会强制检查该接口是否确实有且仅有一个抽象方法，否则将会报错。需要注意的是，即使不使用该注解，只要满足函数式接口的定义，这仍然是一个函数式接口，使用起来都一样。(该接口是一个标记接口)

> Lambda 表达式是一个==匿名函数==，Lambda表达式基于数学中的λ演算得名，直接对应于其中的lambda抽象，是一个匿名函数，即没有函数名的函数。Lambda表达式可以表示闭包。


简写的依据：

1. **能够使用Lambda的依据是必须有相应的函数接口**
2. **Lambda表达式另一个依据是类型推断机制**
3. 在 Java 中，Lambda 表达式的格式是像下面这样

```java
// 无参数，无返回值
() -> log.info("Lambda")
 // 有参数，有返回值
(int a, int b) -> { a+b }
```

其等价于

```java
log.info("Lambda");
private int plus(int a, int b){
      return a+b;
}
```

如果用匿名内部类的形式写：

```java
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("快速新建并启动一个线程");
    }
}).run();
```

1.8之后可以用lambda表达式，进一步简化。

```java
new Thread(()->{
    System.out.println("快速新建并启动一个线程");
}).run();
```

Lambda 表达式简化了匿名内部类的形式，可以达到同样的效果，但是 Lambda 要优雅的多。虽然最终达到的目的是一样的，但其实内部的实现原理却不相同。

匿名内部类在编译之后会创建一个新的匿名内部类出来，而 Lambda 是调用 JVM `invokedynamic`指令实现的，==并不会产生新类==。

## this引用的意义

既然Lambda表达式不是内部类的简写，那么Lambda内部的`this`引用也就跟内部类对象没什么关系了。在Lambda表达式中`this`的意义跟在表达式外部完全一样。因此下列代码将输出两遍`Hello Hoolee`，而不是两个引用地址。

```java
public class Hello {
	Runnable r1 = () -> { System.out.println(this); };
	Runnable r2 = () -> { System.out.println(toString()); };
	public static void main(String[] args) {
		new Hello().r1.run();
		new Hello().r2.run();
	}
	public String toString() { return "Hello Hoolee"; }
}
```

# 方法引用

>什么是方法引用？
>简单地说，就是一个 Lambda 表达式。在 Java 8 中，我们会使用 Lambda 表达式创建匿名方法，但是有时候，我们的 Lambda 表达式可能仅仅调用一个已存在的方法，而不做任何其它事，对于这种情况，通过一个方法名字来引用这个已存在的方法会更加清晰，Java 8 的方法引用允许我们这样做。方法引用是一个更加紧凑，易读的 Lambda 表达式，注意方法引用是一个 Lambda 表达式，其中方法引用的操作符是双冒号 "::"。

方法引用的出现，使得我们可以将一个方法赋给一个变量或者作为参数传递给另外一个方法。`::`双冒号作为方法引用的符号，比如下面这两行语句，引用 `Integer`类的 `parseInt`方法。

```java
Function<String, Integer> s = Integer::parseInt;
Integer i = s.apply("10");
```

- 方法引用引用的方法是已经存在的方法。
- 所有的方法基本都可以被引用。
    **Q：返回值到底是什么类型？**
    A：返回的类型是 Java 8 专门定义的函数式接口，这类接口用 `@FunctionalInterface` 注解。
    有 `Function`、`Comparator`、 `IntBinaryOperator`等等。
    比如 `Function`这个函数式接口的定义如下：

```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}
```

所以这就引出了下面的一个关键：
你的引用方法的参数个数、类型，返回值类型要和函数式接口中的方法声明一一对应才行。

比如 `Integer.parseInt`方法定义如下：

```java
public static int parseInt(String s) throws NumberFormatException {
	return parseInt(s,10);
}
```

首先`parseInt`方法的参数个数是 1 个，而 `Function`中的 `apply`方法参数个数也是 1 个，参数个数对应上了，再来，`apply`方法的参数类型和返回类型是泛型类型，所以肯定能和 `parseInt`方法对应上。
这样一来，就可以正确的接收`Integer::parseInt`的方法引用，并可以调用`Funciton`的`apply`方法，这时候，调用到的其实就是对应的 `Integer.parseInt`方法了。

#### 什么场景适合使用方法引用:

当一个 Lambda 表达式调用了一个已存在的方法

#### 什么场景不适合使用方法引用:

需要往引用的方法传参数的时候不适合：

# Collection中的新方法

## forEach()

该方法的签名为`void forEach(Consumer<? super E> action)`，作用是对容器中的每个元素执行`action`指定的动作，其中`Consumer`是个函数接口，里面只有一个待实现方法`void accept(T t)`（后面我们会看到，这个方法叫什么根本不重要，你甚至不需要记忆它的名字）。

需求：_假设有一个字符串列表，需要打印出其中所有长度大于3的字符串._
Java7及以前我们可以用增强的for循环实现：

```java
// 使用增强for循环迭代
ArrayList<String> list = new ArrayList<>(Arrays.asList("I", "love", "you", "too"));
for(String str : list){
    if(str.length()>3)
        System.out.println(str);
}
```

现在使用`forEach()`方法结合匿名内部类，可以这样实现：

```java
// 使用forEach()结合匿名内部类迭代
ArrayList<String> list = new ArrayList<>(Arrays.asList("I", "love", "you", "too"));
list.forEach(new Consumer<String>(){
    @Override
    public void accept(String str){
        if(str.length()>3)
            System.out.println(str);
    }
});
```

上述代码调用`forEach()`方法，并使用匿名内部类实现`Comsumer`接口。到目前为止我们没看到这种设计有什么好处，但是不要忘记Lambda表达式，使用Lambda表达式实现如下：

```java
// 使用forEach()结合Lambda表达式迭代
ArrayList<String> list = new ArrayList<>(Arrays.asList("I", "love", "you", "too"));
list.forEach( str -> {
        if(str.length()>3)
            System.out.println(str);
    });
```

上述代码给`forEach()`方法传入一个Lambda表达式，我们不需要知道`accept()`方法，也不需要知道`Consumer`接口，类型推导帮我们做了一切。

# Stream

## 1. 什么是流？

　　Stream不是集合元素，它不是数据结构并不保存数据，它是有关算法和计算的，它更像一个高级版本的Iterator。原始版本的Iterator，用户只能显式地一个一个遍历元素并对其执行某些操作；高级版本的Stream，用户只要给出需要对其包含的元素执行什么操作，比如，“过滤掉长度大于 10 的字符串”、“获取每个字符串的首字母”等，Stream会隐式地在内部进行遍历，做出相应的数据转换。Stream就如同一个迭代器（Iterator），单向，不可往复，数据只能遍历一次，遍历过一次后即用尽了，就好比流水从面前流过，一去不复返。

　　而和迭代器又不同的是，Stream可以并行化操作，迭代器只能命令式地、串行化操作。顾名思义，当使用串行方式去遍历时，每个item读完后再读下一个item。**而使用并行去遍历时，数据会被分成多个段，其中每一个都在不同的线程中处理，然后将结果一起输出。** Stream的并行操作依赖于Java7中引入的Fork/Join框架（JSR166y）来拆分任务和加速处理过程。

　　Stream 的另外一大特点是，数据源本身可以是无限的。

## 2. 流的构成

当我们使用一个流的时候，通常包括三个基本步骤：获取一个数据源（source）→ 数据转换 → 执行操作获取想要的结果。**每次转换原有Stream对象不改变，返回一个新的Stream对象（可以有多次转换）**，这就允许对其操作可以像链条一样排列，变成一个管道，如下图所示:
![image-20240312135118402](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/image-20240312135118402.png)

## 3. Stream生成方式

（1）从Collection和数组获得

- Collection.stream()
- Collection.parallelStream()
- Arrays.stream(T array) or Stream.of()

---

（2）从BufferedReader获得

- java.io.BufferedReader.lines()

---

（3）静态工厂

- java.util.stream.IntStream.range()
- java.nio.file.Files.walk()

---

（4）自己构建

- java.util.Spliterator

---

（5）其他

- Random.ints()
- BitSet.stream()
- Pattern.splitAsStream(java.lang.CharSequence)
- JarFile.stream()

## 4. 流的操作类型

流的操作类型分为两种：

- ==Intermediate==：一个流可以后面跟随零个或多个intermediate操作。其目的主要是打开流，做出某种程度的数据映射/过滤，然后返回一个新的流，交给下一个操作使用。这类操作都是惰性化的（lazy），就是说，仅仅调用到这类方法，并没有真正开始流的遍历。

- ==Terminal==：一个流只能有一个terminal操作，当这个操作执行后，流就被使用“光”了，无法再被操作。所以,这必定是流的最后一个操作。Terminal操作的执行，才会真正开始流的遍历，并且会生成一个结果，或者一个side effect。

 　　在对一个Stream进行多次转换操作(Intermediate 操作)，每次都对Stream的每个元素进行转换，而且是执行多次，这样时间复杂度就是N（转换次数）个for循环里把所有操作都做掉的总和吗？其实不是这样的，**转换操作都是lazy的，多个转换操作只会在Terminal操作的时候融合起来，一次循环完成。我们可以这样简单的理解，Stream里有个操作函数的集合，每次转换操作就是把转换函数放入这个集合中，在Terminal 操作的时候循环Stream对应的集合，然后对每个元素执行所有的函数。**

　　还有一种操作被称为**short-circuiting**。用以指：对于一个intermediate操作，如果它接受的是一个无限大（infinite/unbounded）的Stream，但返回一个有限的新Stream；对于一个terminal操作，如果它接受的是一个无限大的Stream，但能在有限的时间计算出结果。
当操作一个无限大的 Stream，而又希望在有限时间内完成操作，则在管道内拥有一个short-circuiting操作是必要非充分条件。

## 5. 流的使用

>简单说，**对Stream的使用就是实现一个filter-map-reduce过程，产生一个最终结果，或者导致一个副作用（side effect）。**

### 1. 流的构造与转换

　下面提供最常见的几种构造Stream的例子:

```java
// 1. Individual values
Stream stream = Stream.of("a", "b", "c");
// 2. Arrays
String [] strArray = new String[] {"a", "b", "c"};
stream = Stream.of(strArray);
stream = Arrays.stream(strArray);
// 3. Collections
List<String> list = Arrays.asList(strArray);
stream = list.stream();
```

---

　需要注意的是，对于基本数值型，目前有三种对应的包装类型Stream：IntStream、LongStream、DoubleStream。当然我们也可以用` Stream<Integer>`、`Stream<Long>`和`Stream<Double>`，但是boxing/unboxing会很耗时，所以特别为这三种基本数值型提供了对应的Stream。

　　Java8中还没有提供其它数值型Stream，因为这将导致扩增的内容较多。而常规的数值型聚合运算可以通过上面三种Stream进行。

```java
IntStream.of(new int[]{1, 2, 3}).forEach(System.out::println);
IntStream.range(1, 3).forEach(System.out::println);
IntStream.rangeClosed(1, 3).forEach(System.out::println);
```

　　流也可以转换为其它数据结构，例如：

```java
// 1. Array
String[] strArray1 = stream.toArray(String[]::new);
// 2. Collection
List<String> list1 = stream.collect(Collectors.toList());
List<String> list2 = stream.collect(Collectors.toCollection(ArrayList::new));
Set set1 = stream.collect(Collectors.toSet());
Stack stack1 = stream.collect(Collectors.toCollection(Stack::new));
// 3. String
String str = stream.collect(Collectors.joining()).toString();
```

### 2. 流的操作

　　接下来，当把一个数据结构包装成Stream后，就要开始对里面的元素进行各类操作了。常见的操作可以归类如下：

- Intermediate 操作

    map (mapToInt, flatMap 等)、 filter、 distinct、 sorted、 peek、 limit、 skip、 parallel、 sequential、 unordered

- Terminal 操作

    forEach、 forEachOrdered、 toArray、 reduce、 collect、 min、 max、 count、 anyMatch、 allMatch、 noneMatch、 findFirst、 findAny、 iterator

- Short-circuiting 操作

    anyMatch、 allMatch、 noneMatch、 findFirst、 findAny、 limit

        　我们下面看一下Stream的比较典型用法。

---

#### 1. Intermediate 操作

- map/flatMap
    我们先来看map，它的作用就是把inputStream的每个元素映射成outputStream的另外一个元素，例如：

```java
List<Integer> nums = Arrays.asList(1, 2, 3, 4);
List<Integer> squareNums = nums.stream().map(n -> n * n)
.collect(Collectors.toList());
```

　从上面例子可以看出，map生成的是个1:1映射，每个输入元素都按照规则转换成为另外一个元素。还有一些场景，是一对多映射关系的，这时需要flatMap，例如：

```java
Stream<List<Integer>> inputStream = Stream.of(
 Arrays.asList(1),
 Arrays.asList(2, 3),
 Arrays.asList(4, 5, 6)
 );
Stream<Integer> outputStream = inputStream.
flatMap((childList) -> childList.stream());
```

　flatMap把inputStream中的层级结构 扁平化，就是将最底层元素抽出来放到一起，最终output的新Stream里面已经没有List了，都是直接的数字。
　

---

- filter
    filter对原始Stream进行某项测试，通过测试的元素被留下来生成一个新Stream。

```java
// 留下偶数
Integer[] sixNums = {1, 2, 3, 4, 5, 6};
Integer[] evens =
Stream.of(sixNums).filter(n -> n%2 == 0).toArray(Integer[]::new);
```

---

- forEach
    forEach方法接收一个Lambda表达式，然后在Stream的每一个元素上执行该表达式。

```java
// 对一个人员集合遍历，找出男性并打印姓名。
roster.stream().filter(p -> p.getGender() == Person.Sex.MALE)
.forEach(p -> System.out.println(p.getName()));
```

　可以看出来，forEach是为Lambda而设计的，保持了最紧凑的风格。当需要为多核系统优化时，可以parallelStream().forEach()，只是此时原有元素的次序没法保证，并行的情况下将改变串行时操作的行为，此时forEach本身的实现不需要调整，而Java8以前的for循环代码可能需要加入额外的多线程逻辑。但一般认为，forEach和常规for循环的差异不涉及到性能，它们仅仅是函数式风格与传统 Java 风格的差别。

　另外一点需要注意，forEach是terminal操作。因此，它执行后，Stream 的元素就被“消费”掉了，你无法对一个Stream进行两次terminal运算。下面的代码是错误的：

```java
     stream.forEach(element -> doOneThing(element));
     stream.forEach(element -> doAnotherThing(element));
```

　相反，具有相似功能的intermediate操作peek可以达到上述目的。如下是出现在Stream api javadoc上的一个示例:

```java
// peek 对每个元素执行操作并返回一个新的 Stream
Stream.of("one", "two", "three", "four").filter(e -> e.length() > 3)
 .peek(e -> System.out.println("Filtered value: " + e)).map(String::toUpperCase)
 .peek(e -> System.out.println("Mapped value: " + e)).collect(Collectors.toList());
```

　**forEach 不能修改自己包含的本地变量值，也不能用break/return之类的关键字提前结束循环。**
　

---

- limit/skip

    limit返回Stream的前面n个元素；skip则是扔掉前n个元素（它是由一个叫 subStream的方法改名而来）。


```java
//limit 和 skip 对运行次数的影响
public void testLimitAndSkip() {
 List<Person> persons = new ArrayList();
 for (int i = 1; i <= 10000; i++) {
 Person person = new Person(i, "name" + i);
 persons.add(person);
 }
List<String> personList2 = persons.stream().
map(Person::getName).limit(10).skip(3).collect(Collectors.toList());
 System.out.println(personList2);
}
private class Person {
 public int no;
 private String name;
 public Person (int no, String name) {
 this.no = no;
 this.name = name;
 }
 public String getName() {
 System.out.println(name);
 return name;
 }
}

输出结果为：
name1
name2
name3
name4
name5
name6
name7
name8
name9
name10
[name4, name5, name6, name7, name8, name9, name10]
```

　这是一个有10，000个元素的Stream，但在short-circuiting操作limit和skip的作用下，管道中map操作指定的getName()方法的执行次数为 limit 所限定的10次，而最终返回结果在跳过前3个元素后只有后面7个返回。

---

- sorted

    对Stream的排序通过sorted进行，它比数组的排序更强之处在于你可以首先对Stream进行各类map、filter、limit、skip甚至distinct来减少元素数量后再排序，这能帮助程序明显缩短执行时间。例如：

```java
// 优化：排序前进行 limit 和 skip
List<Person> persons = new ArrayList();
 for (int i = 1; i <= 5; i++) {
 Person person = new Person(i, "name" + i);
 persons.add(person);
 }

List<Person> personList2 = persons.stream().limit(2).sorted((p1, p2) -> p1.getName().compareTo(p2.getName())).collect(Collectors.toList());
System.out.println(personList2);
```

结果会简单很多：

```java
name2
name1
[stream.StreamDW$Person@6ce253f1,stream.StreamDW$Person@53d8d10a]
```

当然，这种优化是有business logic上的局限性的：即不要求排序后再取值。

---

- `Stream` 的 `collect` 操作是将流中的元素收集到一个可变容器或聚合操作中的结果。它是一个终端操作，用于将流中的元素进行聚合、转换或分组，并将结果收集到一个集合中，如列表、集合、映射等。

`collect` 操作的语法如下：

```java
<R> R collect(Collector<? super T, A, R> collector)
```

其中，`Collector` 是一个用于描述收集操作的接口，它定义了将元素收集到容器中所需的操作。`collect` 方法接受一个 `Collector` 参数，根据 `Collector` 的定义，将流中的元素进行收集，并返回最终的结果。
`Collector` 接口中定义了一些用于收集操作的静态方法，例如 `toList()`、`toSet()`、`toMap()` 等，它们提供了常见的收集操作。

## 小结

总之，Stream 的特性可以归纳为：

- 不是数据结构;

- 它没有内部存储，它只是用操作管道从source（数据结构、数组、generator function、IO channel）抓取数据;

- 它也绝不修改自己所封装的底层数据结构的数据。例如Stream的filter操作会产生一个不包含被过滤元素的新Stream，而不是从source删除那些元素;

- 所有Stream的操作必须以lambda表达式为参数;

- 不支持索引访问;

- 你可以请求第一个元素，但无法请求第二个，第三个，或最后一个;

- 很容易生成数组或者List;

- 惰性化;

- 很多Stream操作是向后延迟的，一直到它弄清楚了最后需要多少数据才会开始;

- Intermediate操作永远是惰性化的;

- 并行能力;

- 当一个 Stream 是并行化的，就不需要再写多线程代码，所有对它的操作会自动并行进行的;

- 可以是无限的。集合有固定大小，Stream 则不必。limit(n)和findFirst()这类的short-circuiting操作可以对无限的Stream进行运算并很快完成。




参考文档：
https://blog.csdn.net/justloveyou_/article/details/79562574
https://www.jianshu.com/p/4a3da6a11b58
https://www.cnblogs.com/jimoer/p/10995574.html
https://objcoding.com/2019/03/04/lambda/
https://pdai.tech/md/java/java8/java8-stream.html