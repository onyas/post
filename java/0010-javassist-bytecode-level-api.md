---
title: javassist入门之字节码接口
tags:
	- java
	- javassist
	- 翻译
	
---

Javassist 还提供了用于直接编辑类文件的低级级 API。 使用此 API之前，你需要详细了解Java 字节码和类文件格式，因为它允许你对类文件进行任意修改。

如果你只想生成一个简单的类文件，使用javassist.bytecode.ClassFileWriter就足够了。 它比javassist.bytecode.ClassFile更快而且更小。

### 获取 ClassFile 对象

javassist.bytecode.ClassFile 对象表示类文件。要获得这个对象，应该调用 CtClass 中的 getClassFile() 方法。
你也可以直接从类文件构造 javassist.bytecode.ClassFile 对象。 例如：

```
	BufferedInputStream fin
    = new BufferedInputStream(new FileInputStream("Point.class"));
	ClassFile cf = new ClassFile(new DataInputStream(fin));
```

这代码段从 Point.class 创建一个 ClassFile 对象。
ClassFile 对象可以写回类文件。ClassFile 的 write() 将类文件的内容写入给定的 DataOutputStream。

### 添加和删除成员

ClassFile 提供了 addField()，addMethod() 和 addAttribute()，来向类添加字段、方法和类文件属性。

注意，FieldInfo，MethodInfo 和 AttributeInfo 对象包括到 ConstPool（常量池表）对象的链接。 ConstPool 对象必须对 ClassFile 对象和添加到该 ClassFile 对象的 FieldInfo（或MethodInfo 等）对象是通用的。 换句话说，FieldInfo（或MethodInfo等）对象不能在不同的ClassFile 对象之间共享。

要从 ClassFile 对象中删除字段或方法，必须首先获取包含该类的所有字段的 java.util.List 对象。 getFields() 和 getMethods() 返回列表。可以通过在List对象上调用 remove() 来删除字段或方法。可以以类似的方式去除属性。在 FieldInfo 或 MethodInfo 中调用 getAttributes() 以获取属性列表，并从列表中删除一个。

### 遍历方法体

使用 CodeIterator 可以检查方法体中的每个字节码指令，要获得 CodeIterator 对象，参考以下代码：

```
	ClassFile cf = ... ;
	MethodInfo minfo = cf.getMethod("move");    // we assume move is not overloaded.
	CodeAttribute ca = minfo.getCodeAttribute();
	CodeIterator i = ca.iterator();
```

CodeIterator 对象允许你逐个访问每个字节码指令。下面展示了一部分 CodeIterator 中声明的方法：

* void begin（）移动到第一条指令。
* void move（int index）移动到指定位置的指令。
* boolean hasNext（）是否有下一条指定
* int next（）返回下一条指令的索引。注意，它不返回下一条指令的操作码。
* int byteAt（int index）返回索引处的无符号8位整数。
* int u16bitAt（int index）返回索引处的无符号16位整数。
* int write（byte [] code，int index）在索引处写入字节数组。
* void insert（int index，byte [] code）在索引处插入字节数组。自动调整分支偏移量。

以下代码段打印了方法体中所有的指令：

```
	CodeIterator ci = ... ;
	while (ci.hasNext()) {
	    int index = ci.next();
	    int op = ci.byteAt(index);
	    System.out.println(Mnemonic.OPCODE[op]);
	}
```

### 生成字节码序列

Bytecode 对象表示字节码指令序列。它是一个可扩展的字节码数组。
以下是示例代码段：

```
	ConstPool cp = ...;    // constant pool table
	Bytecode b = new Bytecode(cp, 1, 0);
	b.addIconst(3);
	b.addReturn(CtClass.intType);
	CodeAttribute ca = b.toCodeAttribute();
```

这段代码产生以下序列的代码属性：

```
	iconst_3
	ireturn
```

您还可以通过调用 Bytecode 中的 get() 方法来获取包含此序列的字节数组。获得的数组可以插入另一个代码属性。

Bytecode 提供了许多方法来添加特定的指令，例如使用 addOpcode() 添加一个 8 位操作码，使用 addIndex() 用于添加一个索引。每个操作码的值定义在 Opcode 接口中。

addOpcode() 和添加特定指令的方法，将自动维持最大堆栈深度，除非控制流没有分支。可以通过调用 Bytecode 的 getMaxStack() 方法来获得这个深度。它也反映在从 Bytecode对象构造的 CodeAttribute 对象上。要重新计算方法体的最大堆栈深度，可以调用 CodeAttribute 的 computeMaxStack() 方法。

### 注释（元标签）

注释作为运行时不可见（或可见）的注记属性，存储在类文件中。调用 getAttribute（AnnotationsAttribute.invisibleTag）方法，可以从 ClassFile，MethodInfo 或 FieldInfo 中获取注记属性。更多信息，请参阅 javassist.bytecode.AnnotationsAttribute 和javassist.bytecode.annotation 包的 javadoc 手册。

Javassist还允许您通过更高级别的API访问注释。 如果要通过CtClass访问注释，请在CtClass或CtBehavior中调用getAnnotations（）。

