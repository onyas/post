---
title: javassist入门之其他特性
tags:
	- java
	- javassist
	- 翻译
	
---

### 泛型

Javassist 的低级别 API 完全支持 Java 5 引入的泛型。但是，高级别的API（如CtClass）不直接支持泛型。

Java 的泛型是通过擦除技术实现。 编译后，所有类型参数都将被删除。 例如，假设您的源代码声明一个参数化类型 Vector<String>：

```
	Vector<String> v = new Vector<String>();
	  :
	String s = v.get(0);
```

编译后的字节码等价于以下代码：

```
	Vector v = new Vector();
	  :
	String s = (String)v.get(0);
```

因此，在编写字节码变换器时，您可以删除所有类型参数，因为 Javassist 的编译器不支持泛型。如果源代码使用 Javassist 编译，例如通过 CtMethod.make()，源代码必须显式类型转换。如果源代码由常规 Java 编译器（如javac）编译，则不需要做类型转换。

例如，如果你有一个类：

```
	public class Wrapper<T> {
	  T value;
	  public Wrapper(T t) { value = t; }
	}
```

并想添加一个接口 Getter<T> 到类 Wrapper<T>：

```
	public interface Getter<T> {
	  T get();
	}
```

那么你真正要添加的接口其实是Getter（将类型参数<T>掉落），最后你添加到 Wrapper 类的方法是这样的：

```
	public Object get() { return value; }
```

注意，不需要类型参数。由于 get 返回一个 Object，如果源代码是由 Javassist 编译的，那么在调用方需要进行显式类型转换。例如，如果类型参数T是 String，则必须插入(String)，如下所示：

```
	Wrapper w = ...
	String s = (String)w.get();
```

### 可变参数

目前，Javassist 不直接支持可变参数。 因此，要使用 varargs 创建方法，必须显式设置方法修饰符。假设要定义下面这个方法：

```
	public int length(int... args) { return args.length; }
```

使用 Javassist 应该是这样的：

```
	CtClass cc = /* target class */;
	CtMethod m = CtMethod.make("public int length(int[] args) { return args.length; }", cc);
	m.setModifiers(m.getModifiers() | Modifier.VARARGS);
	cc.addMethod(m);
```

参数类型int ...被更改为int []，Modifier.VARARGS被添加到方法修饰符中。

要在由 Javassist 的编译器编译的源代码中调用此方法，需要这样写：

```
	length(new int[] { 1, 2, 3 });
```

而不是这样：

```
	length(1, 2, 3);
```

### J2ME

如果要修改 J2ME 执行环境的类文件，则必须先执行预验证。预验证基本上是生成堆栈映射，这类似于在 JDK 1.6 中引入 J2SE 的堆栈映射表。当javassist.bytecode.MethodInfo.doPreverify 为 true 时，Javassist 才会维护 J2ME 的堆栈映射。

对于指定的 CtMethod 对象，你可以调用以下方法，手动生成堆栈映射：

```
	m.getMethodInfo().rebuildStackMapForME(cpool);
```

这里，cpool 是一个 ClassPool 对象，通过在 CtClass 对象上调用 getClassPool() 可以获得。 ClassPool 对象负责从给定类路径中查找类文件。要获得所有的 CtMethod 对象，需要在 CtClass 对象上调用 getDeclaredMethods() 方法。

### 装箱/拆箱

Java 中的装箱和拆箱是语法糖。没有用于装箱或拆箱的字节码。所以 Javassist 的编译器不支持它们。 例如，以下语句在 Java 中有效：

```
	Integer i = 3;
```

因为隐式地执行了装箱。 但是，对于 Javassist，必须将值类型从 int 显式地转换为 Integer：

```
	Integer i = new Integer(3);
```

### 调试

将 CtClass.debugDump 设为本地目录。 然后 Javassist 修改和生成的所有类文件都保存在该目录中。要停止此操作，将 CtClass.debugDump 设置为 null 即可。其默认值为 null。

例如，

```
	CtClass.debugDump =“./dump”;
```

所有修改的类文件都保存在 ./dump 中。