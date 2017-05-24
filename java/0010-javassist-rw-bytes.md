---
title: 读写字节码
tags: 
	- java
	- javassist
	- 翻译
	 
---
Javassist是处理Java字节码的类库，Java字节码存储在称为class文件的二进制文件中，每个class文件都包含一个Java类或接口。

类Javassist.CtClass是class文件的抽象表示，一个CtClass（编译时类）对象是处理class文件的句柄。以下程序是一个非常简单的例子：

```
	ClassPool pool = ClassPool.getDefault();
	CtClass cc = pool.get("test.Rectangle");
	cc.setSuperclass(pool.get("test.Point"));
	cc.writeFile();
```

该程序首先获得一个ClassPool对象，该对象在Javassist中控制字节码修改。ClassPool对象是表示class文件的CtClass对象的容器，它根据需要读取构建CtClass对象的class文件，并记录构造的对象以响应稍后的访问。要修改类的定义，用户必须首先从ClassPool对象获取对表示该类的CtClass对象的引用。ClassPool中的get()方法用于此目的。如上代码所示，CtClass对象代表了从ClassPool对象获取的类test.Rectangle，并分配给变量cc。getDefault()方法返回的ClassPool对象搜索默认的系统搜索路径。

从实现的角度来看，ClassPool是一个CtClass对象的哈希表，它使用类名作为键。ClassPool中的get()搜索此哈希表以查找与指定键相关联的CtClass对象。如果没有找到这样的CtClass对象，那么get()将读取一个类文件来构造一个新的CtClass对象，该对象被记录在哈希表中，然后作为get()的值返回。

从ClassPool对象获取的CtClass对象可以被修改（稍后将介绍如何修改CtClass的详细信息）。在上面的例子中，它被修改使得test.Rectangle的超类变成一个类test.Point。当CtClass()中的writeFile()方法最终调用时，这个变化反映在原始的class文件中。

writeFile()将CtClass对象转换为class文件并将其写入本地磁盘。Javassist还提供了直接获取修改后的字节码的方法，要获取字节码，请调用toBytecode()：

```
	byte[] b = cc.toBytecode();
```

您也可以直接加载CtClass：

```
	Class clazz = cc.toClass();
```

toClass()请求当前线程的上下文类加载器加载由CtClass表示的类文件，它返回一个表示加载类的java.lang.Class对象，有关详细信息，请参阅下面的这一部分。

### 定义一个新类

要从头开始定义一个新类，必须在ClassPool上调用makeClass()。

```
	ClassPool pool = ClassPool.getDefault();
	CtClass cc = pool.makeClass("Point");
```

该程序定义了一个不包含成员的类Point。Point的成员方法可以使用CtNewMethod中声明的工厂方法创建，并用CtClass中的addMethod()附加到Point类中。

makeClass()不能创建新的接口; ClassPool中的makeInterface()可以做到。可以使用CtNewMethod中的abstractMethod()创建接口中的成员方法。请注意，接口方法是抽象方法。

### 冻结类

如果CtClass对象通过writeFile()，toClass()或toBytecode()转换为class文件，则Javassist将冻结CtClass对象，不允许对该CtClass对象进行进一步修改。这是为了在开发人员尝试修改已经加载的类文件时发出警告，因为JVM不允许重新加载类。

冻结的CtClass可以除霜，以便允许修改类定义。例如，

```
	CtClasss cc = ...;
    :
	cc.writeFile();
	cc.defrost();
	cc.setSuperclass(...);    // OK since the class is not frozen.
```

在调用defrost()之后，可以再次修改CtClass对象。

如果ClassPool.doPruning设置为true，则Javassist会在Javassist冻结该对象时修剪包含在CtClass对象中的数据结构。为了减少内存消耗，修剪会丢弃该对象中不必要的属性（attribute_info结构）。例如，Code_attribute结构（方法体）被丢弃。因此，在修剪CtClass对象后，除方法名称，签名和注释之外，方法的字节码不可访问，修剪的CtClass对象不能再次除霜。ClassPool.doPruning的默认值为false。

要禁止修剪一个特定的CtClass，必须提前对该对象调用stopPruning()：

```
	CtClasss cc = ...;
	cc.stopPruning(true);
	    :
	cc.writeFile();                             // convert to a class file.
	// cc is not pruned.
```

CtClass对象cc不被修剪。因此，在调用writeFile()后可以进行除霜。


>**注意**：调试时，您可能希望临时停止修剪并冻结，并将修改后的类文件写入磁盘驱动器。 debugWriteFile()是一个方便的方法。它停止修剪，写一个类文件，除霜，并再次修剪（如果它最初打开）。

###类查找路径

由静态方法ClassPool.getDefault()返回的默认ClassPool与JVM（Java虚拟机）具有的相同的搜索路径。<font color="red">如果程序运行在诸如JBoss和Tomcat之类的Web应用程序服务器上，那么ClassPool对象可能无法找到用户类，</font>因为这样的Web应用程序服务器使用多个类加载器以及系统类加载器。 在这种情况下，必须向ClassPool注册一个附加的类路径。 假设该池指的是一个ClassPool对象：

```
	pool.insertClassPath(new ClassClassPath(this.getClass()));
```

此语句注册了用于加载该引用对象的类的类路径。您可以使用任何Class对象作为参数，而不是this.getClass()。用于加载由该类对象表示的类的类路径被注册。

您可以将一个目录注册为类搜索路径。例如，以下代码将目录/usr/local/javalib添加到搜索路径中：

```
	ClassPool pool = ClassPool.getDefault();
	pool.insertClassPath("/usr/local/javalib");
```

用户添加的搜索路径不仅可以是一个目录，而且也可以是一个url

```
	ClassPool pool = ClassPool.getDefault();
	ClassPath cp = new URLClassPath("www.javassist.org", 80, "/java/", "org.javassist.");
	pool.insertClassPath(cp);
```

该程序将“http://www.javassist.org:80/java/”添加到类搜索路径。此URL仅用于搜索属于包org.javassist的类。例如，要加载类org.javassist.test.Main，其类文件将从以下获取：

```
	http://www.javassist.org:80/java/org/javassist/test/Main.class
```

此外，您可以直接向ClassPool对象发送一个字节数组，并从该数组构造一个CtClass对象。为此，请使用ByteArrayClassPath。 例如，

```
	ClassPool cp = ClassPool.getDefault();
	byte[] b = a byte array;
	String name = class name;
	cp.insertClassPath(new ByteArrayClassPath(name, b));
	CtClass cc = cp.get(name);
```

获取的CtClass对象表示由b指定的类文件定义的类。如果调用get()，并且给定get()的类名称等于由name指定的类名，ClassPool将从给定的ByteArrayClassPath读取一个类文件。

如果您不知道该类的完全限定名称，那么可以在ClassPool中使用makeClass()：

```
	ClassPool cp = ClassPool.getDefault();
	InputStream ins = an input stream for reading a class file;
	CtClass cc = cp.makeClass(ins);
```

makeClass()返回从给定输入流构造的CtClass对象。您可以使用makeClass()热切地将类文件提供给ClassPool对象。如果搜索路径包含大型jar文件，这可能会提高性能，由于ClassPool对象根据需要读取类文件，因此可能会重复搜索整个jar文件中的每个类文件，makeClass()可用于优化此搜索。 由makeClass()构造的CtClass保存在ClassPool对象中，并且类文件永远不会再读取。

用户可以扩展类搜索路径，他们可以定义一个实现ClassPath接口的新类，并把此类的一个实例传递给ClassPool的insertClassPath()。 这允许将非标准资源包括在搜索路径中。
