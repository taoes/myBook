
 ## 1. JDK8的Stream

 JDK 8新增了Stream处理的方式,结合函数式编程，可以很方便的解决我们以前的那又臭又长的代码，虽然说在性能上，Stream操作相比于原生的操作性能略低，但是应该是瑕不掩瑜的，下面记录着个人开发中常用的一些Stream使用案例。

### 使用foreach遍历

```java
  public static void main(String[] args) {
    List<String> stringList = Arrays.asList("A", "B", "C", "D", "E", "F", "G");
    stringList.forEach(x -> System.out.println(x));KT
  }
```

当然这里也可以传入一个Consumer<T>或者指定一个静态方法对象,其本质是一样的，均属于函数式编程，如下:

```java
package com.seven.java.stream;

import java.util.Arrays;
import java.util.List;
import java.util.function.Consumer;

public class Foreach {

  public static void main(String[] args) {
    List<String> stringList = Arrays.asList("A", "B", "C", "D", "E", "F", "G");
    // 传入方法
    stringList.forEach(Foreach::printText);
    // 传入Function
    Consumer<String> printText = text -> System.out.println(text.toLowerCase());
    stringList.forEach(printText);
  }

  public static void printText(String text) {
    System.out.println(text.toLowerCase());
  }
}

```

### 使用Map转换

在上面的实例中，我们将大写字母专为小写字母输出，那么我们假设这种需求，遍历一个整数集合，将其平方后输出其值得1/2,根据上面的示例，我们可以写一个方法或者Consumer接收一个int类型的数据，然后在该代码中将int平方后在除以2,这样也可以实现我们的需求，但其转换和输出的代码逻辑放在一起，从可阅读性的角度来说并不合适。幸运的是JDK8提供的`map方法`来供我们转换数据,map方法要求传递一个Function类型，该类型传入一个泛型对象T，返回一个泛型对象K，定义如：`Function<Integer, Double> convertIntToDouble`

```java
package com.seven.java.stream;

import java.util.Arrays;
import java.util.List;
import java.util.function.Function;

public class Map {
  public static void main(String[] args) {
    List<Integer> integerList = Arrays.asList(1, 2, 3, 4, 5, 6, 7);
    Function<Integer, Double> convertIntToDouble = i -> Math.pow(i, 2) / 2.0;
    integerList.stream().map(convertIntToDouble).forEach(i -> System.out.println(i));
  }
}
```



### 使用Filter过滤

同样的是上面的例子，我们添加一个条件，在给定的集合中过滤出奇数，然后平方再除以2，然后输出,这个是否我们就需要添加过滤的方式了,filter方法的参数要求传递一个PreDicate，该类型传递一泛型数据，要求返回一个Boolean类型的数据。

```java
package com.seven.java.stream;

import java.util.Arrays;
import java.util.List;
import java.util.function.Function;
import java.util.function.Predicate;

public class Filter {

  public static void main(String[] args) {
    List<Integer> integerList = Arrays.asList(1, 2, 3, 4, 5, 6, 7);
    Predicate<Integer> filterOddNumber = integer -> integer % 2 != 0;
    Function<Integer, Double> convertIntToDouble = i -> Math.pow(i, 2) / 2.0;
    integerList
        .stream()
        .filter(filterOddNumber)
        .map(convertIntToDouble)
        .forEach(i -> System.out.println(i));
  }
}
```

### 使用Group分组

同样式上面的需求，这是我们有个新的需求，将证书按照奇数和偶数分开.

```java
package com.seven.java.stream;

import java.util.Arrays;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class Group {

  public static void main(String[] args) {
    final String[] numberType = {"偶数", "奇数"};
    List<Integer> integerList = Arrays.asList(1, 2, 3, 4, 5, 6, 7);
    Map<String, List<Integer>> collect =
        integerList.stream().collect(Collectors.groupingBy(x -> numberType[(x % 2)]));
    System.out.println(collect);
  }
}
```



### 使用Collect收集

在上面的案例中均是直接打印处理啊，这样不利于后续数据的处理，我们需要将处理好的数据进行收集，这时候就需要用到collect方法，该方法需要传入Collector对象，一般的话，我们只要使用常用的Collectors下的定义好的就足够了.

常用的有如下:

```java
  // 转换为集合，其实质类型为ArrayList
  Collectors.toList()
  // 转换为Set集合(自动去重)其实质类型为HashSet
  Collectors.toSet() 
  // 转换为集合(Collect类型)
  Collectors.toCollect()
  // 转换为集合,需要指定K，V(KEY 重复的时候回抛出异常)
  Collectors.toMap(T,K)
  // 转换为线程安全的Map
  Collectors.toConcurrentMap();
```

具体的示例：

```java
package com.seven.java.stream;

import java.util.Arrays;
import java.util.List;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

public class Collect {

  public static void main(String[] args) {
    List<Integer> integerList = Arrays.asList(1, 2, 3);
    // [1,4,9]
    integerList.stream().map(integer -> integer * integer).collect(Collectors.toList());

    // 转换为Map KEY 是学生的ID，VALUE对应Student对象
    List<Student> studentList =
        Arrays.asList(
      new Student(1, "张三"),
      new Student(2, "李四"),
      new Student(3, "王二"));
    Map<Integer, Student> studentMap =
        studentList
      .stream()
      .collect(Collectors.toMap(Student::getId, Function.identity()));
  }

  /** 定义一个对象 */
  static class Student {
    private int id;
    private String name;

    public Student(int id, String name) {
      this.id = id;
      this.name = name;
    }

    public int getId() {return id;}

    public void setId(int id) {this.id = id;}

    public String getName() {return name;}

    public void setName(String name) {this.name = name;}
  }
}

```

