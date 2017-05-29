---
title: 自我检查和定制
tags:
	- java
	- javassist
	- 翻译
	
---

CtClass提供了自我检查的方法。Javassist的自我检查的能力与Java反射API兼容。CtClass提供了getName()，getSuperclass()，getMethods()等等。CtClass还提供了修改类定义的方法。它允许添加一个新的字段，构造函数和方法。构造方法体也是可能的。

方法由CtMethod对象表示。CtMethod提供了几种修改方法定义的方法。请注意，如果方法是从超类继承的，那么表示继承方法的相同CtMethod对象代表在该超级类中声明的方法。CtMethod对象对应于每个方法声明。

例如，如果类Point声明方法move()，并且Point的子类ColorPoint不会覆盖move()，则Point中声明并在ColorPoint中继承的两个move()方法由相同的CtMethod对象表示。 如果由此CtMethod对象表示的方法定义被修改，则修改将反映在这两个方法上。 如果要仅修改ColorPoint中的move()方法，则首先必须向ColorPoint添加一个表示Point中的move()的CtMethod对象的副本。 CtMethod对象的副本可以通过CtNewMethod.copy()获得。

***

Javassist不允许删除一个方法或字段，但它允许更改名称。因此，如果一个方法不再需要，应该通过调用CtMethod中声明的setName()和setModifiers()来重命名并更改为私有方法。

Javassist也不允许在现有方法中添加一个额外的参数。为了不那样做，你应该在同一个类中添加一个新的接收额外的参数以及其他参数的新方法。 例如，如果要在方法中添加一个额外的int参数newZ：

```
	void move(int newX, int newY) { x = newX; y = newY; }
```

在Point类中，你应该以下代码到Point类中

```
	void move(int newX, int newY, int newZ) {
	    // do what you want with newZ.
	    move(newX, newY);
	}
```

***

Javassist还提供用于直接编辑原始类文件的低级API。例如，CtClass中的getClassFile()返回一个表示原始类文件的ClassFile对象。CtMethod中的getMethodInfo()返回一个MethodInfo对象，表示类文件中包含的method_info结构。低级API使用Java虚拟机规范中的词汇表。用户必须了解有关类文件和字节码的知识。有关更多详细信息，用户应看到javassist.bytecode包。

由Javassist修改的类文件只有在使用以$开头的特殊标识符时才需要javassist.runtime包的运行时支持。这些特殊标识符如下所述。在没有这些特殊标识符的情况下修改的类文件在运行时不需要javassist.runtime包或任何其他Javassist包。有关更多详细信息，请参阅javassist.runtime软件包的API文档。

### 将源文本插入方法正文的开始/结尾

CtMethod和CtConstructor提供了insertBefore()，insertAfter()和addCatch()的方法。 它们用于将代码片段插入到现有方法的正文中。用户可以使用Java编写的源文本来指定这些代码段。Javassist包括一个用于处理源文本的简单Java编译器。它接收用Java编写的源文本，并将其编译为Java字节码，它将内联到方法体中。

在一个行号指定的位置插入一个代码片段也是可能的(如果行号表包含在类文件中)。CtMethod和CtConstructor中的insertAt()在原始类定义的源文件中获取源文本和行号。它编译源文件并将编译的代码插入指定行号。

insertBefore()，insertAfter()，addCatch()和insertAt()方法接收一个String对象表示语句或块的。一个语句是单个控件结构，如if和while，或以一个半冒号（;）结尾的表达式）。一个块是用大括号{}包围的一组语句。 因此，以下每行都是有效语句或块的示例：

```
	System.out.println("Hello");
	{ System.out.println("Hello"); }
	if (i < 0) { i = -i; }
```

语句和块可以引用字段和方法。如果该方法使用-g选项（在类文件中包含一个局部变量属性）编译，它们也可以引用它们插入到的方法的参数。 否则，他们必须通过下面描述的特殊变量$0，$1，$2，...尽管在块中声明新的局部变量是允许的，访问方法中声明的局部变量是不允许的。 但是，如果这些变量在指定的行号可用，并且目标方法使用-g选项编译，则insertAt()允许语句和块访问局部变量。

传递给insertBefore()，insertAfter()，addCatch()和insertAt()的方法的String对象由Javassist中包含的编译器编译。 由于编译器支持语言扩展，因此以$开头的几个标识符具有特殊的含义：

* $0, $1, $2, ...    	依次代表真实的参数
* $args	参数数组，它的类型是Object[]
* $$	所有的真实参数，例如m($$) 等价于 m($1,$2,...)
* $cflow(...)	cflow 变量
* $r	返回结果的类型，用于强制类型转换
* $w	包装器类型，用于强制类型转换
* $_	返回值
* $sig	代表参数类型的java.lang.Class对象数组。
* $type	一个 java.lang.Class 对象，表示返回值类型
* $class	一个 java.lang.Class 对象，表示当前正在修改的类

#### $0, $1, $2, ...

传递给目标方法的参数使用 $1，$2，... 访问，而不是原始的参数名称。$1表示第一个参数，$2 表示第二个参数，以此类推。这些变量的类型与参数类型相同。$0等价于this指针。如果方法是静态的，则$0不可用。

下面有一些使用这些特殊变量的例子。假设一个类 Point:

```
	class Point {
	    int x, y;
	    void move(int dx, int dy) { x += dx; y += dy; }
	}
```

要在调用方法 move() 时打印 dx 和 dy 的值，请执行以下程序：

```
	ClassPool pool = ClassPool.getDefault();
	CtClass cc = pool.get("Point");
	CtMethod m = cc.getDeclaredMethod("move");
	m.insertBefore("{ System.out.println($1); System.out.println($2); }");
	cc.writeFile();
```

请注意，传递给 insertBefore() 的源文本是用大括号 {} 括起来的。insertBefore() 只接受单个语句或用大括号括起来的语句块。

修改后的类 Point 的定义是这样的：

```
	class Point {
	    int x, y;
	    void move(int dx, int dy) {
	        { System.out.println(dx); System.out.println(dy); }
	        x += dx; y += dy;
	    }
	}
```

$1和$2分别替换为dx和dy。
$1,$2,$3 ...是可更新的。如果这些变量被赋予新值，则由该变量表示的参数的值也将被更新。

##### $args

变量 $args 表示所有参数的数组。该变量的类型是Object类的数组。如果参数类型是原始类型(如int)，则该参数值将转换为包装器对象（如java.lang.Integer）以存储在 $args 中。 因此，如果第一个参数的类型是不是原始类型，那么$args[0]等于$1。注意 $args[0] 不等于 $0,因为$0表示 this。

如果Object的数组被分配给 $args，那么该数组的每个元素都被分配给每个参数。如果参数类型是基本类型，则相应元素的类型必须是包装类型。在将值分配给参数之前，必须将该值从包装器类型转换为基本类型。

##### $$

变量 $$ 是所有参数列表的缩写，用逗号分隔。 例如，如果方法 move() 的有 3 个参数，则

```
	move($1, $2, $3)
```

如果 move() 不带任何参数，则 move（$$）等同于move()。
$$ 可以与其他方法一起使用。 如果你写一个表达式：

```
	exMove($$, context)
```

这个表达式等价于

```
	exMove($1, $2, $3, context)
```

注意，$$ 开启了通用符号方法调用，它通常与稍后要介绍的 $proceed 一起使用。

##### $cflow

$ cflow表示 控制流。此只读变量返回特定方法的递归调用的深度。

假设下面所示的方法由CtMethod对象cm表示：	

```
	int fact(int n) {
	    if (n <= 1)
	        return n;
	    else
	        return n * fact(n - 1);
	}
```

要使用 $cflow，首先声明使用 $cflow 监视方法 fact() 的调用：

```
	CtMethod cm = ...;
	cm.useCflow("fact");
```

useCflow()的参数是 $cflow 变量的标识符。任何有效的Java名称都可以用作标识符。标识符还可以包括 . ，例如，“my.Test.fact”是有效的标识符。

然后，$cflow(fact) 表示由 cm 指定的方法的递归调用深度。$cflow(fact) 的值在方法第一次调用时为 0，而当方法在方法中递归调用时为 1。 例如，

```
	cm.insertBefore("if ($cflow(fact) == 0)"
              + "    System.out.println(\"fact \" + $1);");
```

转化了方法fact()，以便它显示参数。因为检查了 $cflow(fact)的值，所以如果在fact()中递归调用，则方法 fact() 不会显示参数。

$cflow 的值是当前线程的最顶层堆栈帧下与 cm 相关联的堆栈帧数。$cflow 也可以不在cm方法中访问。

##### $r

$r 表示方法的结果类型（返回类型）。它用在 cast 表达式中作 cast 转换类型。 下面是一个典型的用法：

```
	Object result = ... ;
	$_ = ($r)result;
```

如果结果类型是原始类型，则 ($r) 遵循特殊语义。首先，如果 cast 表达式的操作数是原始类型，($r) 作为普通转换运算符。另一方面，如果操作数是包装类型，($r) 将从包装类型转换为结果类型。 例如，如果结果类型是 int，那么 ($r) 将从 java.lang.Integer 转换为 int。

如果结果类型为void，那么 ($r) 不转换类型; 它什么也不做。 但是，如果操作数是对 void 方法的调用，则 ($r) 将导致 null。 例如，如果结果类型是 void，而 foo() 是一个 void 方法，那么

```
	$_ = ($r)foo();
```

是一个正确的表达式。

cast 运算符 ($r) 在 return 语句中也很有用。 即使结果类型是 void，下面的 return 语句也是有效的：

```
	return ($r)result;
```

这里，result是局部变量。 因为指定了 ($r)，所以结果值被丢弃。此返回语句被等价于:

```
	return;
```

##### $w

$w 表示包装类型。它用在 cast 表达式中作 cast 转换类型。($w) 把基本类型转换为包装类型。 以下代码是一个示例：

```
	Integer i = ($w)5;
```

包装后的类型取决于 ($w) 后面表达式的类型。如果表达式的类型为 double，则包装器类型为 java.lang.Double。

如果下面的表达式 ($w) 的类型不是原始类型，那么($w) 什么也不做。

##### $_

CtMethod 中的 insertAfter() 和 CtConstructor 在方法的末尾插入编译的代码。传递给insertAfter() 的语句中，不但可以使用特殊符号如 $0，$1。也可以使用 $ 来表示方法的结果值。
该变量的类型是方法的结果类型（返回类型）。如果结果类型为 void，那么 $ 的类型为Object，$ 的值为 null。
虽然由 insertAfter() 插入的编译代码通常在方法返回之前执行，但是当方法抛出异常时，它也可以执行。要在抛出异常时执行它，insertAfter() 的第二个参数 asFinally 必须为true。
如果抛出异常，由 insertAfter() 插入的编译代码将作为 finally 子句执行。$ 的值 0 或 null。在编译代码的执行终止后，最初抛出的异常被重新抛出给调用者。注意，$_ 的值不会被抛给调用者，它将被丢弃。

##### $sig

$sig 的值是一个 java.lang.Class 对象的数组，表示声明的形式参数类型。

##### $type

$type 的值是一个 java.lang.Class 对象，表示结果值的类型。 如果这是一个构造函数，此变量返回 Void.class。

##### $class

$class 的值是一个 java.lang.Class 对象，表示编辑的方法所在的类。 即表示 $0 的类型。

#### addCatch()

addCatch() 插入方法体抛出异常时执行的代码，控制权会返回给调用者。 在插入的源代码中，异常用 $e 表示。

例如：

```
	CtMethod m = ...;
	CtClass etype = ClassPool.getDefault().get("java.io.IOException");
	m.addCatch("{ System.out.println($e); throw $e; }", etype);
```

转换成对应的 java 代码如下：

```
	try {
	    // the original method body
	} catch (java.io.IOException e) {
	    System.out.println(e);
	    throw e;
	}
```

请注意，插入的代码片段必须以 throw 或 return 语句结束。

### 修改方法体

CtMethod 和 CtConstructor 提供 setBody() 来替换整个方法体。他将新的源代码编译成 Java 字节码，并用它替换原方法体。 如果给定的源文本为 null，则替换后的方法体仅包含返回语句，返回零或空值，除非结果类型为 void。

在传递给 setBody() 的源代码中，以 $ 开头的标识符具有特殊含义：

* $0, $1, $2, ...    	依次代表真实的参数
* $args	参数数组，它的类型是Object[]
* $$	所有的真实参数，例如m($$) 等价于 m($1,$2,...)
* $cflow(...)	cflow 变量
* $r	返回结果的类型，用于强制类型转换
* $w	包装器类型，用于强制类型转换
* $sig	代表参数类型的java.lang.Class对象数组。
* $type	一个 java.lang.Class 对象，表示返回值类型
* $class	一个 java.lang.Class 对象，表示当前正在修改的类

注意 $_ 不可用。

#### 替换表达式

Javassist 只允许修改方法体中包含的表达式。javassist.expr.ExprEditor 是一个用于替换方法体中的表达式的类。用户可以定义 ExprEditor 的子类来指定修改表达式的方式。

要运行 ExprEditor 对象，用户必须在 CtMethod 或 CtClass 中调用 instrument()。
例如，

```
	CtMethod cm = ... ;
	cm.instrument(
	    new ExprEditor() {
	        public void edit(MethodCall m) throws CannotCompileException {
	            if (m.getClassName().equals("Point")
	                          && m.getMethodName().equals("move"))
	                m.replace("{ $1 = 0; $_ = $proceed($$); }");
	        }
	    });
```

上述代码，搜索由 cm 表示的方法体，并用使用下面的代码替换 Point 中的 move()调用：

```
	{ $1 = 0; $_ = $proceed($$); }
```

因此 move() 的第一个参数总是0。注意，替换的代码不是一个表达式，而是一个语句或块。 它不能是或包含 try-catch 语句。


方法 instrument() 搜索方法体。如果它找到一个表达式，如方法调用、字段访问和对象创建，那么它调用给定的 ExprEditor 对象上的 edit() 方法。 edit() 的参数表示找到的表达式。edit() 可以检查和替换该表达式。

调用 edit() 参数的 replace() 方法可以将表达式替换为我们给定的语句。如果给定的语句是空块，即执行replace("{}")，则将表达式删除。如果要在表达式之前或之后插入语句（或块），则应该将类似以下的代码传递给 replace()：

```
	{ *before-statements;*
	  $_ = $proceed($$);
	  *after-statements;* }
```

无论表达式是方法调用、字段访问还是对象创建或其他。

如果表达式是读操作，第二个语句应该是：

```
	$_ = $proceed();
```

如果表达式是写操作，则第二个语句应该是：

```
	$proceed($$);
```

如果由 instrument() 搜索的方法是使用 -g 选项（类文件包含一个局部变量属性）编译的，目标表达式中可用的局部变量，也可以传递给 replace() 的源代码中使用。

#### javassist.expr.MethodCall

MethodCall 表示方法调用。MethodCall 的 replace() 方法用于替换方法调用，它接收表示替换语句或块的源代码。和 insertBefore() 方法一样，传递给 replace 的源代码中，以 $ 开头的标识符具有特殊的含义。

* $0, 方法调用的目标对象。它不等于 this，它代表了调用者。 如果方法是静态的，则 $0 为 null
* $1, $2, ...    	方法参数
* $_	方法调用的结果
* $r	返回结果的类型，用于强制类型转换
* $w	包装器类型，用于强制类型转换
* $sig	代表参数类型的java.lang.Class对象数组。
* $type	一个 java.lang.Class 对象，表示返回值类型
* $class	一个 java.lang.Class 对象，表示当前正在修改的类
* $proceed 调用表达式中方法的名称

这里的方法调用意味着由 MethodCall 对象表示的方法。

其他标识符如 $w，$args 和 $$ 也可用。

除非方法调用的返回类型为 void，否则返回值必须在源代码中赋给 $_，$_ 的类型是表达式的结果类型。如果结果类型为 void，那么 $_ 的类型为Object，并且分配给 $_ 的值将被忽略。

$proceed 不是字符串值，而是特殊的语法。 它后面必须跟一个由括号括起来的参数列表。

#### javassist.expr.ConstructorCall

ConstructorCall 表示构造函数调用，例如包含在构造函数中的 this() 和 super()。ConstructorCall 中的方法 replace() 可以使用语句或代码块来代替构造函数。它接收表示替换语句或块的源代码。和 insertBefore() 方法一样，传递给 replace 的源代码中，以 $ 开头的标识符具有特殊的含义。

* $0, 构造调用的目标对象。它等于 this
* $1, $2, ...    	方法参数
* $sig	代表参数类型的java.lang.Class对象数组。
* $class	一个 java.lang.Class 对象，表示当前正在修改的类
* $proceed 调用表达式中方法的名称

这里的构造函数调用是由 ConstructorCall 对象表示的。

其他标识符如 $w，$args 和 $$ 也可用。

由于任何构造函数必须调用超类的构造函数或同一类的另一个构造函数，所以替换语句必须包含构造函数调用，通常是对 $proceed() 的调用。

$proceed 不是字符串值，而是特殊的语法。 它后面必须跟一个由括号括起来的参数列表。

#### javassist.expr.FieldAccess

FieldAccess 对象表示字段访问。 如果找到对应的字段访问操作，ExprEditor 中的 edit() 方法将接收到一个 FieldAccess 对象。FieldAccess 中的 replace() 方法接收替源代码来替换字段访问。

在源代码中，以 $ 开头的标识符具有特殊含义：

* $0, 表达式访问的字段。它不等于 this。this 表示调用表达式所在方法的对象。如果字段是静态的，则 $0 为 null
* $1, $2, ...    	方法参数
* $_	如果表达式是读操作，则结果值保存在 $1 中，否则将舍弃存储在 $_ 中的值
* $r	如果表达式是读操作，则 $r 读取结果的类型。 否则 $r 为 void
* $w	包装器类型，用于强制类型转换
* $sig	代表参数类型的java.lang.Class对象数组。
* $type	一个 java.lang.Class 对象，表示返回值类型
* $class	一个 java.lang.Class 对象，表示当前正在修改的类
* $proceed 调用表达式中方法的名称

如果表达式是读操作，则必须在源文本中将值分配给 $。 $的类型是字段的类型。

#### javassist.expr.NewExpr

NewExpr 表示使用 new 运算符（不包括数组创建）创建对象的表达式。 如果发现创建对象的操作，NewEditor 中的 edit() 方法将接收到一个 NewExpr 对象。NewExpr 中的 replace() 方法接收替源代码来替换字段访问。

在源文本中，以 $ 开头的标识符具有特殊含义：

* $0,  null
* $1, $2, ...    	方法参数
* $_	创建对象的返回值。一个新的对象存储在 $_ 中
* $r	所创建的对象的类型
* $w	包装器类型，用于强制类型转换
* $sig	代表参数类型的java.lang.Class对象数组。
* $type	一个 java.lang.Class 对象，表示返回值类型
* $class	一个 java.lang.Class 对象，表示当前正在修改的类
* $proceed 执行对象创建虚拟方法的名称

javassist.expr.NewArray

NewArray 表示使用 new 运算符创建数组。如果发现数组创建的操作，ExprEditor 中的 edit() 方法代表一个 NewArray 对象。NewArray 中的 replace() 方法可以使用源代码来替换数组创建操作。

在源文本中，以$开头的标识符具有特殊含义：

* $0	null
* $1, $1	每一维的大小
* $_	创建数组的返回值。一个新的数组对象存储在 $_ 中
* $r	所创建的数组的类型
* $type	一个 java.lang.Class 对象，表示创建的数组的类型
* $proceed	执行数组创建虚拟方法的名称

其他标识符如 $w，$args 和 $$ 也可用。

例如，如果按下面的方式创建数组：

```
	String[][] s = new String[3][4];
```

那么 $1 和 $2 的值分别是 3 和 4。 $3 不可用。

例如，如果按下面的方式创建数组：

```
	String[][] s = new String[3][];
```

那么 $1 的值为 3，但 $2 不可用。

#### javassist.expr.Instanceof

一个 InstanceOf 对象表示一个 instanceof 表达式。 如果找到 instanceof 表达式，则ExprEditor 中的 edit() 方法接收此对象。Instanceof 中的 replace() 方法可以使用源代码来替换 instanceof 表达式。

在源文本中，以$开头的标识符具有特殊含义：

* $0	null
* $1	instanceof 运算符左侧的值
* $_	表达式的返回值。类型为 boolean
* $r	instanceof 运算符右侧的值
* $type	一个 java.lang.Class 对象，表示 instanceof 运算符右侧的类型
* $proceed	执行 instanceof 表达式的虚拟方法的名称。它需要一个参数（类型是 java.lang.Object）。如果参数类型和 instanceof 表达式右侧的类型一致，则返回 true。否则返回 false。

其他标识符如 $w，$args 和 $$ 也可用。

#### javassist.expr.Cast

Cast 表示 cast 表达式。如果找到 cast 表达式，ExprEditor 中的 edit() 方法会接收到一个 Cast 对象。 Cast 的 replace() 方法可以接收源代码来替换替换 cast 表达式。

在源文本中，以$开头的标识符具有特殊含义：

* $0	null
* $1	显示类型转换的目标类型（？）
* $_	表达式的结果值。$_ 的类型和被括号括起来的类型相同（？）
* $r	转换之后的类型，即被括号括起来的类型（？）
* $type	一个 java.lang.Class 对象，和 $r 的类型相同
* $proceed	执行类型转换的虚拟方法的名称。它需要一个参数（类型是 java.lang.Object）。并在类型转换完成后返回它
其他标识符如 $w，$args 和 $$ 也可用。

#### javassist.expr.Handler

Handler 对象表示 try-catch 语句的 catch 子句。 如果找到 catch，ExprEditor 中的 edit() 方法会接收此对象。 Handler 中的 insertBefore() 方法会将收到的源代码插入到 catch 子句的开头。

在源文本中，以$开头的标识符具有意义：

* $1	catch 分支获得的异常对象
* $r	catch 分支获得的异常对象的类型，用于强制类型转换
* $w	包装类型，用于强制类型转换
* $type	一个 java.lang.Class 对象，表示 catch 捕获的异常的类型
如果一个新的异常分配给 $1，它将作为捕获的异常传递给原始的 catch 子句。

### 添加新方法和字段

#### 添加新方法

Javassist 可以创建新的方法和构造函数。CtNewMethod 和 CtNewConstructor 提供了几个工厂方法来创建 CtMethod 或 CtConstructor 对象。make() 方法可以通过源代码来CtMethod 或 CtConstructor 对象。

例如：

```
	CtClass point = ClassPool.getDefault().get("Point");
	CtMethod m = CtNewMethod.make(
	                 "public int xmove(int dx) { x += dx; }",
	                 point);
	point.addMethod(m);
```

上面的代码向类 Point 添加了一个公共方法 xmove()。在这个例子中，x 是类 Point 的一个int 字段。

传递给 make() 和 setBody() 的源文本可以包括以 $ 开头的标识符 ($_ 除外)。 如果目标对象和目标方法名也被传递给 make() 方法，源文本中也可以包括 $proceed。

例如：

```
	CtClass point = ClassPool.getDefault().get("Point");
	CtMethod m = CtNewMethod.make(
	                 "public int ymove(int dy) { $proceed(0, dy); }",
	                 point, "this", "move");
```


这个程序创建一个 ymove() 方法，定义如下：

```
	public int ymove(int dy) { this.move(0, dy); }
```

注意，$proceed 已经被替换为 this.move。

Javassist 还提供了另一种添加新方法的方式。 你可以先创建一个抽象方法，然后给它一个方法体：

```
	CtClass cc = ... ;
	CtMethod m = new CtMethod(CtClass.intType, "move",
	                          new CtClass[] { CtClass.intType }, cc);
	cc.addMethod(m);
	m.setBody("{ x += $1; }");
	cc.setModifiers(cc.getModifiers() & ~Modifier.ABSTRACT);
```

因为 Javassist 在类中添加了的方法是抽象的，所以在调用 setBody() 之后，必须将类显式地改回非抽象类。

#### 相互递归的方法

Javassist 不能编译这种方法：如果它调用另一个方法，而另一个方法没有被添加到一个类（Javassist可以编译一个以递归方式调用的方法）。如果要向类添加相互递归方法，需要使用如下的技巧。假设你想要将方法 m() 和 n() 添加到由 cc 表示的类中：

```
	CtClass cc = ... ;
	CtMethod m = CtNewMethod.make("public abstract int m(int i);", cc);
	CtMethod n = CtNewMethod.make("public abstract int n(int i);", cc);
	cc.addMethod(m);
	cc.addMethod(n);
	m.setBody("{ return ($1 <= 0) ? 1 : (n($1 - 1) * $1); }");
	n.setBody("{ return m($1); }");
	cc.setModifiers(cc.getModifiers() & ~Modifier.ABSTRACT);
```

你必须先创建两个抽象方法，并将它们添加到类中。然后设置它们的方法体，即使方法体包括互相递归的调用。 最后，必须将类更改为非抽象类。


#### 添加一个字段

Javassist 还允许用户创建一个新字段。

```
	CtClass point = ClassPool.getDefault().get("Point");
	CtField f = new CtField(CtClass.intType, "z", point);
	point.addField(f);
```

该程序向类 Point 添加一个名为 z 的字段。
如果必须指定添加字段的初始值，那么上面的程序必须修改为：

```
	CtClass point = ClassPool.getDefault().get("Point");
	CtField f = new CtField(CtClass.intType, "z", point);
	point.addField(f, "0");  // initial value is 0
```

现在，方法 addField() 接收两个参数，第二个参数表示计算初始值的表达式。这个表达式可以是任意 Java 表达式，只要其结果与字段的类型匹配。 请注意，表达式不以分号结尾。

此外，上述代码可以重写为更简单代码


```
	CtClass point = ClassPool.getDefault().get("Point");
	CtField f = CtField.make("public int z = 0;", point);
	point.addField(f);
```

#### 删除成员

要删除字段或方法，请在 CtClass 的 removeField() 或 removeMethod() 方法。 一个CtConstructor 可以通过 CtClass 的 removeConstructor() 删除。

### 注解

CtClass，CtMethod，CtField 和 CtConstructor 提供 getAnnotations() 方法，用于读取注解。 它返回一个注解类型的对象。

例如，假设有以下注解：

```
	public @interface Author {
	    String name();
	    int year();
	}
```

下面是使用注解的代码：

```
	@Author(name="Chiba", year=2005)
	public class Point {
	    int x, y;
	}
```

然后，可以使用 getAnnotations() 获取注解的值。 它返回一个包含注解类型对象的数组。

```
	CtClass cc = ClassPool.getDefault().get("Point");
	Object[] all = cc.getAnnotations();
	Author a = (Author)all[0];
	String name = a.name();
	int year = a.year();
	System.out.println("name: " + name + ", year: " + year);
```

这段代码输出：

```
	name: Chiba, year: 2005
```

由于 Point 的注解只有 @Author，所以数组的长度是 1，all[0] 是一个 Author 对象。 注解成员值可以通过调用Author对象的 name() 和 year() 来获取。

要使用 getAnnotations()，注释类型（如 Author）必须包含在当前类路径中。它们也必须也可以从 ClassPool 对象访问。如果未找到注释类型的类文件，Javassist 将无法获取该注释类型的成员的默认值。

###  运行时支持类

在大多数情况下，使用 Javassist 修改类不需要运行 Javassist。 但是，Javassist 编译器生成的某些字节码需要运行时支持类，这些类位于 javassist.runtime 包中（有关详细信息，请阅读该包的API文档）。请注意，javassist.runtime 是修改的类时唯一可能需要使用的包。 修改类的运行时不会再使用其他的 Javassist 类。

### 导入

源代码中的所有类名都必须是完整的（必须包含包名，java.lang 除外）。例如，Javassist 编译器可以解析 Object 以及 java.lang.Object。

要告诉编译器在解析类名时搜索其他包，请在 ClassPool中 调用 importPackage()。 例如，

```
	ClassPool pool = ClassPool.getDefault();
	pool.importPackage("java.awt");
	CtClass cc = pool.makeClass("Test");
	CtField f = CtField.make("public Point p;", cc);
	cc.addField(f);
```

第二行导入了 java.awt 包。 因此，第三行不会抛出异常。 编译器可以将 Point 识别为java.awt.Point。

注意 importPackage() 不会影响 ClassPool 中的 get() 方法。只有编译器才考虑导入包。 get() 的参数必须是完整类名。

### 限制

在目前实现中，Javassist 中包含的 Java 编译器有一些限制：

* J2SE 5.0 引入的新语法（包括枚举和泛型）不受支持。注释由 Javassist 的低级 API 支持。 参见 javassist.bytecode.annotation 包（以及 CtClass 和 CtBehavior 中的 getAnnotations()）。对泛型只提供部分支持。更多信息，请参阅后面的部分；

* 初始化数组时，只有一维数组可以用大括号加逗号分隔元素的形式初始化，多维数组还不支持；
* 编译器不能编译包含内部类和匿名类的源代码。 但是，Javassist 可以读取和修改内部/匿名类的类文件；
* 不支持带标记的 continue 和 break 语句；
* 编译器没有正确实现 Java 方法调度算法。编译器可能会混淆在类中定义的重载方法（方法名称相同，查参数列表不同）。例如：

```
	class A {} 
	class B extends A {} 
	class C extends B {} 
	class X { 
	  void foo(A a) { .. } 
	  void foo(B b) { .. } 
	}
```

如果编译的表达式是 x.foo(new C())，其中 x 是 X 的实例，编译器将产生对 foo(A) 的调用，尽管编译器可以正确地编译 foo((B) new C()) 。

* 建议使用 # 作为类名和静态方法或字段名之间的分隔符。 例如，在常规 Java 中，
```
	javassist.CtClass.intType.getName()
```

在 javassist.CtClass 中的静态字段 intType 指示的对象上调用一个方法 getName()。 在Javassist 中，用户也可以写上面的表达式，但是建议写成这样：

```
	javassist.CtClass#intType.getName()
```
使编译器可以快速解析表达式。



