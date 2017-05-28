---
title: Javassist入门之类加载器
tags:
	- java
	- javassist
	- 翻译
	
---

如果提前知道哪些类必须修改，那么修改类的最简单的方法如下：

1. 通过调用ClassPool.get()获取一个CtClass对象，
2. 修改它
3. 在该CtClass对象上调用writeFile()或toBytecode()来获取修改的类文件。

如果在加载时才能确定一个类是否被修改，用户必须使Javassist与类加载器协作。Javassist可以与类加载器一起使用，以便在加载时可以修改字节码。 Javassist的用户可以定义自己的类加载器版本，但也可以使用Javassist提供的类加载器。

### Ctclass中的toClass方法

CtClass提供了一个方便的方法toClass()，它为当前线程 加载由CtClass对象表示的类 请求上下文类加载器。要调用此方法，调用者必须具有适当的权限; 否则可能会抛出SecurityException。

下面的这段代码展示了如何使用toClass()

```
	public class Hello {
	    public void say() {
	        System.out.println("Hello");
	    }
	}

	public class Test {
	    public static void main(String[] args) throws Exception {
	        ClassPool cp = ClassPool.getDefault();
	        CtClass cc = cp.get("Hello");
	        CtMethod m = cc.getDeclaredMethod("say");
	        m.insertBefore("{ System.out.println(\"Hello.say():\"); }");
	        Class c = cc.toClass();
	        Hello h = (Hello)c.newInstance();
	        h.say();
	    }
	}
```

Test.main()在Hello的say()方法中插入了一个println()方法，然后它在修改的Hello类上构造了一个实例，并在该实例上调用say()。

请注意，上面的程序取决于在调用toClass()之前，Hello类没有被加载的事实。如果不是，JVM会在toClass请求加载修改的Hello类之前加载原来的Hello类。 因此，加载修改的Hello类将失败(引发LinkageError)。例如，如果测试中的main()类似于此：

```
	public static void main(String[] args) throws Exception {
	    Hello orig = new Hello();
	    ClassPool cp = ClassPool.getDefault();
	    CtClass cc = cp.get("Hello");
	        :
	}
```

main方法的第一行就加载了原来的Hello类，则调用toClass会抛出异常，因为类加载器不会在同一时间加载两个不同版本的Hello类

<font color="red">如果程序在某些应用程序服务器(如JBoss和Tomcat)上运行，</font>则toClass()使用的上下文类加载器可能不合适。在这种情况下，您会看到一个意外的ClassCastException。为了避免这种异常，你必须明确地给toClass()一个适当的类加载器。例如，如果bean是您的会话bean对象，那么以下代码会工作：

```
	CtClass cc = ...;
	Class c = cc.toClass(bean.getClass().getClassLoader());
```

你应该给予toClass()已经加载程序的类加载器(在上面的例子中是bean对象的类)。

toClass()是为了方便提供的，如果您需要更复杂的功能，您应该编写自己的类加载器。

### Java中的类加载

在Java中，多个类加载器可以共存，每个类加载器创建自己的命名空间。不同的类加载器可以加载具有相同类名的不同类文件。加载的两个类被认为是不同的。此功能使我们能够在单个JVM上运行多个应用程序，即使这些程序拥有相同名称但不同的类。

>**注意**：JVM不允许动态重新加载类。 一旦类加载器加载类，它就不能在运行时重新加载该类的修改版本。 因此，在JVM加载它之后，您不能更改类的定义。 然而，JPDA(Java Platform Debugger Architecture)提供了重新加载类的有限能力。 见第3.6节。

如果同一个类文件由两个不同的类加载器加载，那么JVM使两个不同的类具有相同名称和定义。两个类被视为不同的。由于两个类不相同，一个类的实例不能分配给另一个类的变量。两个类之间的转换操作失败，并抛出ClassCastException。

例如，以下代码片段抛出异常：

```
	MyClassLoader myLoader = new MyClassLoader();
	Class clazz = myLoader.loadClass("Box");
	Object obj = clazz.newInstance();
	Box b = (Box)obj;    // this always throws ClassCastException.
```

Box类由两个类加载器加载,假设类加载器CL加载包含该代码片段的类。由于此代码片段引用MyClassLoader，Class，Object和Box，所以CL也加载这些类(除非委托给另一个类加载器)。因此，变量b的类型是由CL加载的Box类。另一方面，myLoader也加载了Box类。对象obj是由myLoader加载的Box类的一个实例。因此，最后一个语句总是引发ClassCastException，因为obj的类是与使用变量b类型的Box类是不同的。

多个类加载器形成一个树结构。除了引导加载器之外，每个类加载器都有一个父类加载器，它通常加载该子类加载器的类。由于加载类的请求可以沿着类加载器的这个层次结构进行委派，所以类可能由你没有请求的类加载器加载。因此，已经请求加载类C的类加载器可能与实际加载类C的加载程序不同。为了区别，我们将前一个加载器称为<font color="red">C的启动器，</font>我们将后者加载器称为<font color="red">C的实际加载器。</font>

此外，如果类加载器CL把加载C类（C的发起者）的请求委托给父类加载器PL，那么类加载器CL从不被请求加载类C中定义引用的任何类。CL不是这些类的启动类。相反，父类加载器PL成为其启动器，并且被请求加载它们。<font color="red">类C中引用的类由C的实际加载器加载</font>

要理解这个行为，我们来考虑下面的例子。

```
	public class Point {    // loaded by PL
	    private int x, y;
	    public int getX() { return x; }
	        :
	}
	
	public class Box {      // the initiator is L but the real loader is PL
	    private Point upperLeft, size;
	    public int getBaseX() { return upperLeft.x; }
	        :
	}
	
	public class Window {    // loaded by a class loader L
	    private Box box;
	    public int getBaseX() { return box.getBaseX(); }
	}
```

假设一个类Window由一个类加载器L加载。Window的启动器和实际加载器都是L。由于Window的定义中引用了Box，JVM将要求L加载Box。这里，假设L将此任务委托给父类加载器PL。Box的启动器是L，但是真正的加载器是PL。在这种情况下，Point的启动器不是L，而是PL，因为它与Box的实际装载器相同。 因此，L从未被请求加载Point。

接下来，让我们考虑一个稍微修改的例子。

```
	public class Point {
	    private int x, y;
	    public int getX() { return x; }
	        :
	}
	
	public class Box {      // the initiator is L but the real loader is PL
	    private Point upperLeft, size;
	    public Point getSize() { return size; }
	        :
	}
	
	public class Window {    // loaded by a class loader L
	    private Box box;
	    public boolean widthIs(int w) {
	        Point p = box.getSize();
	        return w == p.getX();
	    }
	}
```

现在，Window的定义中也引用了Point。在这种情况下，如果类加载器L被请求加载Point，则它必须委托给PL。您必须避免有两个类加载器双重加载相同的类。两个装载器中的一个必须委托给另一个。

如果在加载Point时L不委托给PL，则widthIs()将抛出ClassCastException。由于Box的实际装载器是PL，所以在Box中引用到的Point也由PL加载。因此，getSize()的结果值是由PL加载的Point的实例，而widthIs()中的变量p的类型由L加载.JVM将它们视为不同的类型，因此由于类型不匹配而引发异常。

这种行为有些不方便但很有必要。如果以下声明：

```
	Point p = box.getSize();
```

没有抛出异常，那么Window的开发者可能会破坏Point对象的封装。例如，由PL加载的Point类中字段x是私有的。但是，如果L使用以下定义加载Point，则Window类可以直接访问x的值：

```
	public class Point {
	    public int x, y;    // not private
	    public int getX() { return x; }
	        :
	}
```

对于java中类加载的更多信息，下面的研究可能会更有帮助

Sheng Liang and Gilad Bracha, "Dynamic Class Loading in the Java Virtual Machine", 
ACM OOPSLA'98, pp.36-44, 1998.

Javassist提供了一个叫javassist.Loader的类加载器，这个类加载器使用javassist.ClassPool来读取类文件。

例如，javassist.Loader可以被用来加载由Javassist修改的特殊类

```
	import javassist.*;
	import test.Rectangle;
	
	public class Main {
	  public static void main(String[] args) throws Throwable {
	     ClassPool pool = ClassPool.getDefault();
	     Loader cl = new Loader(pool);
	
	     CtClass ct = pool.get("test.Rectangle");
	     ct.setSuperclass(pool.get("test.Point"));
	
	     Class c = cl.loadClass("test.Rectangle");
	     Object rect = c.newInstance();
	         :
	  }
	}
```

这段程序修改了test.Rectangle类，把它的父类设置成test.Point。然后加载了这个修改的类，并创建了test.Rectangle的一个实例。

如果用户希望在加载类时按需修改它，则用户可以向javassist.Loader添加事件侦听器。添加的事件侦听器在类加载器加载类时收到通知。event-listener类必须实现以下接口：

```
	public interface Translator {
	    public void start(ClassPool pool)
	        throws NotFoundException, CannotCompileException;
	    public void onLoad(ClassPool pool, String classname)
	        throws NotFoundException, CannotCompileException;
	}
```

当javaxist.Loader中的addTranslator()将此事件侦听器添加到javassist.Loader对象时，将调用start()方法。在javassist.Loader加载类之前调用onLoad()方法。onLoad()可以修改加载类的定义。

例如，以下事件监听器在加载之前将所有类更改为公共类。

```
	public class MyTranslator implements Translator {
	    void start(ClassPool pool)
	        throws NotFoundException, CannotCompileException {}
	    void onLoad(ClassPool pool, String classname)
	        throws NotFoundException, CannotCompileException
	    {
	        CtClass cc = pool.get(classname);
	        cc.setModifiers(Modifier.PUBLIC);
	    }
	}
```

请注意，onLoad()不必调用toBytecode()或writeFile()，因为javassist.Loader调用这些方法来获取一个类文件。

要使用MyTranslator对象运行应用程序类MyApp，请编写如下主要类：

```
	import javassist.*;
	
	public class Main2 {
	  public static void main(String[] args) throws Throwable {
	     Translator t = new MyTranslator();
	     ClassPool pool = ClassPool.getDefault();
	     Loader cl = new Loader();
	     cl.addTranslator(pool, t);
	     cl.run("MyApp", args);
	  }
	}
```

要运行这段程序，执行

```
	% java Main2 arg1 arg2...
```

MyApp类和其他应用程序类由MyTranslator转译

请注意，像MyApp这样的应用程序类无法访问诸如Main2，MyTranslator和ClassPool这些被加载的类，因为它们由不同的加载器加载。应用程序类由javassist.Loader加载，而被加载的类（如Main2）则由默认的Java类加载器加载。

javassist.Loader以与java.lang.ClassLoader不同的顺序搜索类。 ClassLoader首先将加载操作委托给父类加载器，然后仅在父类加载器找不到它们时尝试加载类。 另一方面，javassist.Loader尝试在委派给父类加载器之前加载这些类。 它仅在以下情况下进行委托：

1. 通过在ClassPool对象上调用get()找不到类时
2. 或者这些类通过使用delegateLoadingOf()来指定父类加载器加载。

此搜索顺序允许通过Javassist加载修改的类。但是，如果由于某种原因找不到修改的类，它将委托给父类加载器。一旦类由父类加载器加载，则该类中引用的其他类也将由父类加载器加载，因此它们不会被修改。回想一下，C类中引用的所有类都由C的实际加载器加载。如果您的程序无法加载修改的类，则应确保所有使用该类的类是否已由javassist.Loader加载。

### 编写一个类加载器

一个简单的用Javassist编写的加载器如下所示

```
	import javassist.*;
	
	public class SampleLoader extends ClassLoader {
	    /* Call MyApp.main().
	     */
	    public static void main(String[] args) throws Throwable {
	        SampleLoader s = new SampleLoader();
	        Class c = s.loadClass("MyApp");
	        c.getDeclaredMethod("main", new Class[] { String[].class })
	         .invoke(null, new Object[] { args });
	    }
	
	    private ClassPool pool;
	
	    public SampleLoader() throws NotFoundException {
	        pool = new ClassPool();
	        pool.insertClassPath("./class"); // MyApp.class must be there.
	    }
	
	    /* Finds a specified class.
	     * The bytecode for that class can be modified.
	     */
	    protected Class findClass(String name) throws ClassNotFoundException {
	        try {
	            CtClass cc = pool.get(name);
	            // modify the CtClass object here
	            byte[] b = cc.toBytecode();
	            return defineClass(name, b, 0, b.length);
	        } catch (NotFoundException e) {
	            throw new ClassNotFoundException();
	        } catch (IOException e) {
	            throw new ClassNotFoundException();
	        } catch (CannotCompileException e) {
	            throw new ClassNotFoundException();
	        }
	    }
	}
```

MyApp类是一个应用程序。要执行此程序，首先将类文件放在./class目录下，不能包含在类搜索路径中。否则，MyApp.class将被默认的系统类加载器加载，该加载器是SampleLoader的父加载器。目录名./class由构造函数中的insertClassPath()指定。如果需要，您可以选择不同的名称，而不是./class。然后做如下：

```
	% java SampleLoader
```

类加载器加载MyApp类(./class/MyApp.class)，并使用命令行参数调用MyApp.main()。

这是使用Javassist最简单的方法。但是，如果编写更复杂的类加载器，则可能需要详细了解Java类的加载机制。例如，上述程序将MyApp类置于与SampleLoader类所属的名称空间分开的名称空间中，因为这两个类由不同的类加载器加载。因此，MyApp类不能直接访问SampleLoader类。

### 修改系统类

像java.lang.String这样的系统类不能由系统类加载器以外的类加载器加载。 因此，上面显示的SampleLoader或javassist.Loader无法在加载时修改系统类。

如果您的应用程序需要这样做，系统类必须被<font color="red">静态修改。</font>例如，以下程序向java.lang.String添加一个新的字段hiddenValue：

```
	ClassPool pool = ClassPool.getDefault();
	CtClass cc = pool.get("java.lang.String");
	CtField f = new CtField(CtClass.intType, "hiddenValue", cc);
	f.setModifiers(Modifier.PUBLIC);
	cc.addField(f);
	cc.writeFile(".");
```

该程序生成一个文件“./java/lang/String.class”。

要使用此修改的String类运行您的程序MyApp，请执行以下操作：

```
	% java -Xbootclasspath/p:. MyApp arg1 arg2...
```

假设MyApp的定义如下

```
	public class MyApp {
	    public static void main(String[] args) throws Exception {
	        System.out.println(String.class.getField("hiddenValue").getName());
	    }
	}
```

如果被修改的String类被正确加载，MyApp将打印出hiddenValue

*注意：为了覆盖rt.jar中的系统类而使用此技术的应用程序不应该被部署，因为这将违反Java 2 Runtime Environment二进制代码许可证。*

### 运行时重加载类

如果JVM启动时启用了JPDA(Java Platform Debugger Architecture)，则一个类是动态可重新加载的。在JVM加载一个类之后，可以卸载类定义的旧版本，并重新加载一个新的类定义。也就是说，该类的定义可以在运行时动态修改。但是，新的类定义必须与旧的定义有些兼容。<font color="red">JVM不允许在两个版本之间进行模式更改。</font>他们有一套相同的方法和字段。

Javassist提供了一个方便的类，用于在运行时重新加载类。有关更多信息，请参阅javassist.tools.HotSwapper的API文档。


