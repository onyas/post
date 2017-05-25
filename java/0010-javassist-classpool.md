---
title: Javassist入门之ClassPool简介
tags:
	- javassist
	- 翻译

	
---

ClassPool对象是CtClass对象的容器。 一旦创建了一个CtClass对象，它将被记录在ClassPool中，这是因为编译器当编译由该ClassPool对象引用的源代码时，可能稍后需要访问该ClassPool对象。

例如，假设一个新方法getter()被添加到表示Point类的CtClass对象中。之后，该程序尝试编译源代码，包括在Point中的getter()方法调用，并使用编译的代码作为方法的主体，这将被添加到另一个类Line。如果表示Point的CtClass对象丢失，则编译器无法编译getter()方法调用。请注意，原始类定义不包括getter()。因此，为了正确编译这种方法调用，ClassPool必须在程序执行时包含CtClass的所有实例。

### 避免内存溢出

如果CtClass对象的数量变得非常大，那么ClassPool的这个规范可能会导致巨大的内存消耗（这很少发生，因为Javassist尝试以各种方式减少内存消耗）。为了避免此问题，您可以从ClassPool中显式删除不必要的CtClass对象。如果在CtClass对象上调用detach()，那么该CtClass对象将从ClassPool中删除。例如，

```
	CtClass cc = ... ;
	cc.writeFile();
	cc.detach();
```

在调用detach()后，不能调用该CtClass对象上的任何方法。但是，您可以在ClassPool上调用get()来创建表示同一个类的CtClass的新实例。如果调用get()，ClassPool再次读取一个类文件，并且新创建一个由get()返回的CtClass对象。

另一个想法是偶尔用一个新的ClassPool替换并丢弃旧的。如果旧的ClassPool被垃圾回收，则该ClassPool中包含的CtClass对象也将被垃圾回收。要创建一个新的ClassPool实例，请执行以下代码片段：

```
	ClassPool cp = new ClassPool(true);
	// if needed, append an extra search path by appendClassPath()
```

这将创建一个ClassPool对象，它与ClassPool.getDefault()返回的默认ClassPool一样。请注意，ClassPool.getDefault()是为了方便提供的单例工厂方法。它与上述创建ClassPool对象一样，尽管它保留了ClassPool的单个实例并重用它。getDefault()返回的ClassPool对象并没有特殊的角色，它是一个方便的方法。

请注意，new ClassPool(true)是一个方便的构造函数，它构造一个ClassPool对象，并将系统搜索路径附加到它。调用该构造函数等效于以下代码：

```
	ClassPool cp = new ClassPool();
	cp.appendSystemPath();  // or append another path by appendClassPath()
```

### 级联ClassPool

<font color="red">如果程序在Web应用程序服务器上运行，</font>则可能需要创建多个ClassPool实例; 应为每个类加载器（即容器）创建一个ClassPool的实例。 程序应该通过调用ClassPool的构造函数而不是调用getDefault()来创建一个ClassPool对象。

多个ClassPool对象可以像java.lang.ClassLoader一样级联。例如，

```
	ClassPool parent = ClassPool.getDefault();
	ClassPool child = new ClassPool(parent);
	child.insertClassPath("./classes");
```

如果调用child.get()，则子类ClassPool首先委托给父ClassPool。 如果父类ClassPool无法找到类文件，则子类ClassPool将尝试在./classes目录下找类文件。

如果child.childFirstLookup为true，则子类ClassPool将尝试在委派给父ClassPool之前找到一个类文件。例如，

```
	ClassPool parent = ClassPool.getDefault();
	ClassPool child = new ClassPool(parent);
	child.appendSystemPath();         // the same class path as the default one.
	child.childFirstLookup = true;    // changes the behavior of the child.
```

### 更改类名用于定义新的类

新类可以被定义为现有类的副本。下面的程序是：

```
	ClassPool pool = ClassPool.getDefault();
	CtClass cc = pool.get("Point");
	cc.setName("Pair");
```

该程序首先获取类Point的CtClass对象。然后它调用setName()来给该CtClass对象添加一个新的名字。在此调用之后，所有由该CtClass对象表示的类定义中的改变都将从Point改为Pair。类定义的其他部分不会改变。

请注意，CtClass中的setName()会更改ClassPool对象中的记录。从实现的角度来看，ClassPool对象是CtClass对象的哈希表。setName()更改与哈希表中的CtClass对象关联的key。key从原始类名更改为新类名。

因此，如果再次在ClassPool对象上调用get(“Point”)，则它不会返回变量cc引用的CtClass对象。 ClassPool对象再次读取一个类文件Point.class，并为类Point构造一个新的CtClass对象。这是因为与名称Point相关联的CtClass对象不再存在。参见以下内容：

```
	ClassPool pool = ClassPool.getDefault();
	CtClass cc = pool.get("Point");
	CtClass cc1 = pool.get("Point");   // cc1 is identical to cc.
	cc.setName("Pair");
	CtClass cc2 = pool.get("Pair");    // cc2 is identical to cc.
	CtClass cc3 = pool.get("Point");   // cc3 is not identical to cc.
```

cc1和cc2指的是CtClass的相同实例cc，而cc3不是。请注意，执行cc.setName("Pair")后，cc和cc1引用的CtClass对象表示Pair类。

ClassPool对象用于在类和CtClass对象之间维护一对一的映射。Javassist从不允许两个不同的CtClass对象来表示相同的类，除非创建了两个独立的ClassPool。这是程序转换一致性的重要特征。

要创建由ClassPool.getDefault()返回的ClassPool的默认实例的另一个副本，请执行以下代码片段（此代码已在上面显示）：

```
	ClassPool cp = new ClassPool(true);
```

如果您有两个ClassPool对象，则可以从每个ClassPool获取表示相同类文件的不同CtClass对象。 您可以不同地修改这些CtClass对象以生成不同版本的类。

### 重命名一个冻结的类来定义一个新类

一旦CtClass对象通过writeFile()或toBytecode()转换为类文件，Javassist将拒绝对该CtClass对象的进一步修改。因此，在表示Point类的CtClass对象被转换为类文件之后，由于在Point上执行setName()被拒绝，所以不能将Pair类定义为Point的副本。以下代码段错误：

```
	ClassPool pool = ClassPool.getDefault();
	CtClass cc = pool.get("Point");
	cc.writeFile();
	cc.setName("Pair");    // wrong since writeFile() has been called.
```

为了避免这种限制，你应该在ClassPool中调用getAndRename()。例如，

```
	ClassPool pool = ClassPool.getDefault();
	CtClass cc = pool.get("Point");
	cc.writeFile();
	CtClass cc2 = pool.getAndRename("Point", "Pair");
```

如果调用getAndRename()，ClassPool首先读取Point.class来创建一个表示Point类的新CtClass对象。然而，在散列表中记录该CtClass对象之前，它将CtClass对象从Point重新命名为Pair。因此，在表示Point类的CtClass对象上调用writeFile()或toBytecode()后，可以执行getAndRename()。

