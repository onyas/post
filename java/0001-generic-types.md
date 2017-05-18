---
title: 泛型
tags:
   - java
   - 泛型
   - 翻译
---
## 泛型

泛型是类型参数化的泛型类或接口。下面将修改Box类来演示这个概念。

### 一个简单的Box类

首先来查看对任务对象进行操作的非泛型类。它仅需要提供两个方法:set,用来添加对象到box中，get,用来获取它：

```
public class Box{
	pirvate Object object;
	
	public void set(Object object){this.object = object;}
	public Object get(){return object;}
}
```
由于它的方法接收或者返回一个对象，鉴于它不是原始类型，无论你想输入什么都可以。在编译时期没有办法对这个类是如何使用的做校验。一部分代码中可以存一个Integer到这个box类中，并且期望从它那里得到Integer,但另一部分代码可能错误的传了一个String，导致运行期异常。

### Box类的泛型版本

一个泛型类通过如下的格式来定义：

```
class name<T1,T2,...,Tn>{ /* ... */ }
```
类型参数部分，由尖括号分隔，紧跟在类名后面。它指定了类型参数（也称为类型变更）T1,T2,...,Tn。

为了修改Box类来用泛型，你通过把代码"public class Box"改成"public class Box<T>"来创建了一个泛型的声明。这样就引入了泛型变更T,它可以在类中的任何地方使用。

有了这些改变，Box类变成了：

```
/**
 * Generic version of the Box class.
 * @param <T> the type of the value being boxed
 */
public class Box<T> {
    // T stands for "Type"
    private T t;

    public void set(T t) { this.t = t; }
    public T get() { return t; }
}

```

正如你所见，所有Obejct出现的地方全都用T替换了。类型变量可以是你指定的任何非原始变量：任何类，任何接口，任何数组，甚至另外的类型变更。

同样的技术可以用来创建泛型接口。

### 类型参数命名约定

通常，类型参数的名字都是单个大字的字母。这与你已知的变量命名约定形成了鲜明对比，一个好的原因是:如果没有这种对比，你会很识别类型变量跟普通类名或是接口名字的区别。

最常用的类型参数命名有：

- E -Element(在java集合框架中广泛使用)
- K -Key
- N -Number
- T -Type
- V -Value
- S,U,V etc. - 第二，第三，第四类型

你将会看到这些名字被用在整个java SE API中还有接下来的教程中。

### 调用和实例化一个泛型

要在你的代码中引用泛型Box类，你必须执行泛型类型调用，也就是所T替换成具体的值，比如Integer:

```
Box<Integer> integerBox;
```

你可以认为泛型调用跟普通的方法调用相似，但不是将参数(argument)传给方法，而是传递类型定义(type argument)-在这个例子中是把Integer传给Box类。

-
类型参数和类型定义(type argument)术语:许多程序员交替的使用"类型参数"和"类型定义(type argument)"，但是这两个术语并不一样。在编码时为了提供参数化类型，提供了类型参数。因此，在Foo<T>的T是类型参数，但是在Foo<String>的String是类型定义(type argument)。当用到这此术语时，本教程将遵守这个定义。

跟其他变量声明一样，这段代码并不会创建一个Box对象。他简单的声明了integerBox将会持有一个"Box of Integer"的引用。这就是Box<Integer>怎么读的。

泛型调用通常称为参数化类型。

为了实例化这个类，跟通常一样，用new这个关键字，但是在类名跟小括号之间加上<Integer>:

```
Box<Integer> integerBox = new Box<Integer>();
```

### 钻石(The Diamond)

在Java SE 7及更高版本中，只要编译器可以从上下文中确定或推断类型定义(type argument)，就可以使用空类型定义（<>）来替换调用泛型构造函数所需的类型定义 。 这对尖角括号<>，被非正式地称为钻石。 例如，您可以使用以下语句创建Box <Integer>的实例：

```
Box<Integer> integerBox = new Box<>();
```
有关钻石符号和类型推断的更多信息，请参阅[类型推断](http://docs.oracle.com/javase/tutorial/java/generics/genTypeInference.html)。

### 多种类型参数

如前所述，一个泛型类可以有多种类型参数。例如，泛型类OrderedPair实现了泛型接口Pair：

```
public interface Pair<K, V> {
    public K getKey();
    public V getValue();
}

public class OrderedPair<K, V> implements Pair<K, V> {

    private K key;
    private V value;

    public OrderedPair(K key, V value) {
	this.key = key;
	this.value = value;
    }

    public K getKey()	{ return key; }
    public V getValue() { return value; }
}
```

下面的语句创建了OrderedPair类的两个对象

```
Pair<String, Integer> p1 = new OrderedPair<String, Integer>("Even", 8);
Pair<String, String>  p2 = new OrderedPair<String, String>("hello", "world");
```

new OrderedPair<String,Integer>这段代码，将K实例化为String,V实例化为Integer。因此，OrderedPair的构造函数的参数类型分别为String和Integer。 由于自动装箱，将String和int传递给该类是有效的。

如The Diamond所述，由于Java编译器可以从声明OrderedPair <String，Integer>推断出K和V类型，所以可以使用菱形符号来缩短这些语句：

```
OrderedPair<String, Integer> p1 = new OrderedPair<>("Even", 8);
OrderedPair<String, String>  p2 = new OrderedPair<>("hello", "world");
```

要创建泛型接口，请遵循与创建泛型类相同的约定。

### 参数化类型
您也可以使用参数化类型（即List <String>）替换类型参数（即K或V）。 例如，使用OrderedPair <K，V>示例：

```
OrderedPair<String, Box<Integer>> p = new OrderedPair<>("primes", new Box<Integer>(...));
```





 
