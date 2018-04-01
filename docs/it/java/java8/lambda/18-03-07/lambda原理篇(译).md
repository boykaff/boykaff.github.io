### lambda表达式原理篇(译)

时间: 2018年3月7日

> 无它，译之备学。 [译源传送门](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-translation.html)

#### 1. 关于

本文概述了将lambda表达式和方法引用从Java源代码转换为字节码的策略。lambda表达式在 [JSR 335](https://jcp.org/en/jsr/detail?id=335) 进行了详细说明并且在Lambda的 [OpenJDK](http://openjdk.java.net/projects/lambda/)中得以实现，可以在 《[State of the Lambda](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-state-4.html)》 一文中对该语言特性有一个整体认识。本文将主要分析编译器在遇到lambda表达式时要如何转换为字节码以及java运行时如何参与评估lambda表达式，文中的大部分内容会涉及到函数式接口的转换机制。

函数式接口是java的lambda表达式的重要项，一个函数式接口包含一个non-Object(非对象)方法，例如：Runnable、Comparator等(java库常用这些接口进行回调操作)。

lambda表达式只能出现在它们将被分配给类型为函数式接口的变量的地方，如：

```
Runnable r = () -> { System.out.println("hello"); };
```
	
或者

```
Collections.sort(strings, (String a, String b) -> -(a.compareTo(b)));
```

编译器生成用于捕获这些lambda表达式的代码是依赖表达式本身，以及为其分配的函数式接口类型

##### 1.1. 依赖和符号

转化模式依赖于[JSR-292](https://jcp.org/en/jsr/detail?id=292)]中所定义的几个特性，包括invokedynamic，方法句柄以及LDC字节码的增强形态，因为这些特性在java中是不可描述的，所以我们使用伪代码来进行说明：

  - 方法句柄常量： MH([refKind] class-name.method-name)
  - 方法类型常量： MT(method-signature)
  - invokedynamic：INDY((bootstrap, static args...)(dynamic args...))
 
读者们应该对[JSR-292](https://jcp.org/en/jsr/detail?id=292)的特性有一定的了解，转换模式还假设了一个新特性，它正在由 JSR-292 专家组为Java SE8指定:一个用于对常量方法处理的反射的API。

#### 2. 转化策略

能够将lambda表达式转化为字节流的方法有多种，类似于内部类、方法句柄、动态代理等。当然这些方法都有利有弊，在选择策略时，有两个相互制约的目标:通过不提交特定策略、在类文件表示中提供稳定性来最大化以后优化的灵活性。我们可以通过使用[JSR-292](https://jcp.org/en/jsr/detail?id=292)的invokedynamic特性来实现这两个目标，从而将字节码中lambda创建的二进制表示与在运行时评估lambda表达式的机制分离开来。我们不是生成字节码来创建实现lambda表达式的对象(比如调用内部类的构造器)，而是描述构造lambda的方法，并将实际的构建委托给程序运行时来完成。该方法在invokedynamic指令的静态和动态参数列表中进行编码。

使用invokedynamic可以推迟转化策略的选择至运行时，运行时实现可以自由地选择策略来评估lambda表达式。运行时实现选择隐藏在标准化(即平台规范的一部分)用于lambda结构的API，这样静态编译器就可以发出对这个API的调用，JRE的实现可以选择它们的首选策略。invokedynamic机制允许在没有这种延迟绑定方法且可能增加性能成本的情况下完成这项工作。

当编译器遇到lambda表达式时，它会先将该表达式的body体弱化(提取)为一个方法，该方法的参数列表和返回类型同lambda表达式的相匹配，可能会附带一些参数(用于从词法作用域内捕获值)，当lambda表达式被捕获时，将产生一个invokedynamic调用站点，当被调用时，它会返回一个由lambda转换成的函数式接口实例，这个调用站点称为lambda工厂。lambda工厂的动态参数是从词法作用域中获捕获到的值，lambda工厂的启动方法是java运行时库中的标准方法，叫做lambda metafactory(lambda元工厂)，静态引导参数在编译时捕获关于lambda的信息(它将被转换的函数式接口，对desugared lambda体的方法句柄，以及关于SAM类型是否可序列化等信息)。

方法引用与lambda表达式的处理方式相同，但大多数方法引用不需要被到一个新方法里;我们可以简单地为引用的方法加载一个常量方法句柄，并将其传递给metafactory。

#### 3. 提取 lambda body

将lambdas转换为字节码的第一步就是将lambda体提取为一个方法，关于提取有如下几点选择:

- 提取成静态方法或是实例方法?
- 应该提取那儿类?
- 提取方法的可访问性如何?
- 提取方法应如何命名?
- 如果需要适配lambda body签名和函数式接口方法签名之间的差异(比如装箱、开箱、原始扩展或收缩转换、变量转换等)，那么提取方法应该遵循lambda body体的签名、函数式接口签名抑或介于两者之间?由谁负责该适配性工作?
- 如果lambda捕获了闭包作用域的参数，那么在提取方法签名中应该如何表示?(它们可以是添加到参数列表的开始或结束的单独的参数，或者编译器可以将它们收集到一个“框架”参数中。)

与提取lambda体的问题相关的是，方法引用是否需要适配器或桥接器方法的生成。

编译器能够为lambda表达式推导方法签名，包括参数类型、返回类型以及抛出的异常，称为自然签名(natural signature)，Lambda表达式有一个目标类型，它是一个函数式接口;我们将为擦除了目标类型的描述符调用lambda描述符的方法签名。lambda工厂返回的值被称为lambda对象，它实现了函数式接口并捕获了lambda行为。

在条件相同的情况下，私有方法是优于非私有方法，静态方法优于实例方法，如果lambda body体提取到内部类，方法签名应当匹配lambda body的签名，额外的参数应置于参数列表中被捕获的值前面，无需提取方法引用。然而，也有例外情况，我们可能不得不偏离这个基线策略。

##### 3.1. 提取示例--“无状态” lambda

lambda表达式的最简单形式是从它的封闭作用域(无状态的lambda)中捕获任何状态:

```
class A {
	public void foo() {
		List<String> list = ...
		list.forEach( s -> { System.out.println(s); } );
	}
}
```

lambda自然签名(String)V,forEach方法使用一个Block<String>,其lambda描述符为(Object)V,编译器将lambda体提取为自然签名的静态方法并为抽取body体生成方法名。

```
class A {
	public void foo() {
		List<String> list = ...
		list.forEach( [lambda for lambda$1 as Block] );
	}

	static void lambda$1(String s) {
		System.out.println(s);
	}
}
```

##### 3.2. 提取实例——lambdas捕获不可变的值

lambda表达式的另一种形式是捕获包含final类型的(或有效的最终)局部变量，以及来自封闭实例的字段(我们可以将其视为对this的捕获)。

```
class B {
	public void foo() {
		List<Person> list = ...
		final int bottom = ..., top = ...;
		list.removeIf( p -> (p.size >= bottom && p.size <= top) );
	}
}
```
	
lambda从封闭作用域内捕获final类型的局部变量bottom和top。

提取方法的签名将是自然签名(Person)Z，其额外参数在参数列表的最前面。编译器会考虑将这些额外的参数进行何种呈现;它们可以单独预置，装入框架类，或装入一个数组等。最简单的方法是分别对它们进行预处理:

```
class B {
	public void foo() {
		List<Person> list = ...
		final int bottom = ..., top = ...;
		list.removeIf( [ lambda for lambda$1 as Predicate capturing (bottom, top) ]);
	}

	static boolean lambda$1(int bottom, int top, Person p) {
		return (p.size >= bottom && p.size <= top;
	}
}
```

相反，捕获的值(bottom和top)可以被装入框架或数组中;关键是需要保持所提取的lambda方法签名中出现的额外参数的类型和它们作为(动态的)参数显示给lambda工厂的类型之间的一致性。由于编译器控制了这两种方法，并且它们同时生成，因此编译器在如何打包捕获参数方面具有一定的灵活性。

#### 4. lambda 元工厂(Metafactory)

lambda的捕获由invokedynamic调用站点实现，其静态参数描述了lambda body体以及被捕获到的动态参数值的特征，当被调用时，这个调用站点将返回一个lambda对象，以对应lambda的body体、描述符和绑定捕获的值，调用站点启动方法称之为元工厂（可以有一个适应于整个lambda形态的单独的metafactory，或者对常见的情况有专门的版本），VM针对美国调用点仅调用一次metafactory，然后会连接该调用点并离开，调用点都是延迟连接的，因此如果不调用的话就不会连接，metafactory的静态参数列表如下：

```
metaFactory(MethodHandles.Lookup caller, // provided by VM
			String invokedName,          // provided by VM
			MethodType invokedType,      // provided by VM
			MethodHandle descriptor,     // lambda descriptor
			MethodHandle impl)           // lambda body
```

前三个参数（caller, invokeName, invokeType）在callsite链接中被VM自动堆叠；descriptor定义了被转换lambda对应的函数式接口(通过方法句柄的反射API，metafactory能够获取到函数式接口的class类、名字以及主要方法的签名)；impl标识lambda方法，可能是lambda body体方法或是被命名的方法引用。

函数式接口的方法签名和实现方法之间具有差异性，实现方法有一些当前被捕获到值的额外参数，其余的参数也可能不完全匹配；一些例如(子类型，装箱)在适应中是允许的。

##### 4.1. 捕获 lambda

接下来介绍lambda表达式和方法引用向函数式接口的转化，转化示例class A如：

```
class A {
	public void foo() {
			List<String> list = ...
			list.forEach(indy((MH(metaFactory), MH(invokeVirtual Block.apply),
							   MH(invokeStatic A.lambda$1)( )));
		}

	private static void lambda$1(String s) {
		System.out.println(s);
	}
}
```

因为lambda在A中是无状态的，所以lambda工厂站点的动态参数列表是空的，如示例 B，动态参数列表不是空的，因为必须为lambda工厂提供参数bottom和top的值：

```
class B {
	public void foo() {
		List<Person> list = ...
		final int bottom = ..., top = ...;
		list.removeIf(indy((MH(metaFactory), MH(invokeVirtual Predicate.apply),
							MH(invokeStatic B.lambda$1))( bottom, top ))));
	}

	private static boolean lambda$1(int bottom, int top, Person p) {
		return (p.size >= bottom && p.size <= top;
	}
}
```

##### 4.2. 静态方法 VS 实例方法

类似于上述的lambda能够转换为静态方法，因为它们不以任何方式使用封装对象实例(this、super或封闭实例的成员)，总的来说，我们将引用使用this、super或捕获封闭实例成员的lambdas作为实例捕获(instance-capturing) lambdas。非实例捕获(non-instance-capturing)的lambdas被转换为静态的私有方法，实例捕获的lambda将被转换为私有方法，这简化可instance-capturing lambdas的提取，因为lambda body体中的名字和desugared方法的名称相同，并且与可用的实现技术(绑定方法句柄)很好地吻合。当捕获一个实例捕获lambda时，接收者(this)被指定为第一个动态参数。以捕获lambda的minSize字段为例：

```
list.filter(e -> e.getSize() < minSize)
```
将其提取为一个实例方法并将接收者作为第一个参数：

```
list.forEach(INDY((MH(metaFactory), MH(invokeVirtual Predicate.apply),
				   MH(invokeVirtual B.lambda$1))( this ))));

private boolean lambda$1(Element e) {
	return e.getSize() < minSize;
}
```

因为lambda的body体会转换成私有方法，所以在传递行为方法句柄给metafactory是，捕获点需要加载一个常量方法句柄，该句柄的引用类型是用于实例方法的REF_invokeSpecial和用于静态方法的REF_invokeStatic。之所以能够提取私有方法，是因为私有方法可以被捕获类访问，因此可以为metafactory所调用的私有方法获取一个方法句柄(如果metafactory生成字节码来实现目标函数接口，而不是直接调用方法句柄，它将通过Unsafe.defineClass来加载这些类，无需进行可达性检测）。

##### 4.3. 方法引用捕获

方法引用有多种形式如lambda，可以划分为instance-capturing 和 non-instance-capturing，non-instance-capturing的方法引用包含静态方法引用 (Integer::parseInt, 用引用类invokeStatic捕获)，未绑定实例的方法引用(String::length,用引用invokeVirtual捕获)以及顶级构造方法引用(Foo::new,用invoke_newSpecial引用进行捕获)，当捕获一个non-instance-capturing的方法引用时，被捕获的参数列表是空的：

```
list.filter(String::isEmpty)
```

被转换成：

```
list.filter(indy(MH(metaFactory), MH(invokeVirtual Predicate.apply), MH(invokeVirtual String.isEmpty))()))
```

instance-capturing 方法引用形态包括绑定实例方法引用(s::length, 通过引用invokeVirtual捕获)，super方法引用(super::foo, 通过引用invokeSpecial捕获)以及内部类构造引用(Inner::new, 通过引用invokeNewSpecial捕获)，当捕获一个 instance-capturing方法引用，被捕获的参数列表总是有一个单独的参数，是super里的this或内部类放入构造函数方法引用，亦或是绑定实例方法引用所指定接收者。

##### 4.4. 可变参数 (Varargs)

如果对varargs方法的方法引用被转换为一个的函数式接口，而非varargs方法，编译器必须生成一个桥接方法并捕获桥接的方法句柄以替代捕获目标方法本身。桥接必须处理任何需要的参数类型的修改以及从varargs到非varargs的修改，如：

```
interface IIS {
	void foo(Integer a1, Integer a2, String a3);
}

class Foo {
	static void m(Number a1, Object... rest) { ... }
}

class Bar {
	void bar() {
		SIS x = Foo::m;
	}
}
```

在此，编译器需要生成一个桥接来执行从Number到Integer的第一个参数类型的适配，以及将剩余的参数收集到一个对象数组中:

```
class Bar {
	void bar() {
		SIS x = indy((MH(metafactory), MH(invokeVirtual IIS.foo),
					  MH(invokeStatic m$bridge))( ))
	}

	static private void m$bridge(Integer a1, Integer a2, String a3) {
		Foo.m(a1, a2, a3);
	}
}
```

##### 4.5. 适应性

提取lambda方法有一个参数列表和返回类型(A1..An) -> Ra (如果接收方法是实例方法，那么接收者被认为是其中的第一个参数)，类似地，函数式接口同样有一个参数列表和返回类型：(F1..Fm) -> Rf（无接收者参数），工厂站点的动态参数列表有参数类型(D1..Dk)。如果lambda是实例捕获，那么第一个动态参数必须是接收者。

它们的长度必须加起来如下:k+m == n，也就是说，lambda体参数列表长度应该等于动态参数列表和函数式接口方法参数列表长度之和。
将lambda body参数列表A1..An划分为(D1 . .Dk H1..Hm，D参数对应于“额外”(动态)参数，H参数对应于函数式接口参数。
我们要求Hi在1到m范围内能适应Fi。同样地，我们要求Ra可以适应Rf,类型T适用于类型U:

- T == U
- T为基本数据类型，U为引用数据类型，T能够通过装箱操作转换为U
- T为引用数据类型，U为基本用数据类型，T能够通过拆箱操作转换为U
- T和U都为基本数据类型，且T能够通过扩展转换为U
- T和U都为引用数据类型，T能够转换为U

适应性由metafactory在链接时进行验证，并在执行时进行捕获。

##### 4.6. metafactory 变体

对所有的lambda形态都用一个元数据是是可行的。然而，最好还是将 metafactory 划分为多个版本:

- 一个“快速路径(fast path)”版本，它支持不可序列化的 lambda 和不可序列化的静态或未绑定实例方法引用;
- 一个可序列化的版本，支持可序列化的 lambda 和各种方法引用;
- 如果有必要，一个“kitchen sink”版本，它支持任意组合的转换特性。

kitchen sink版本将会使用一个额外的flag标志参数来选择选项，可能还有其他特定选项的参数。serializable版本可能会接受与序列化有关的附加参数。

由于metafactories不是由用户直接调用的，所以通过多种方法来做相同的事情，就不会产生混淆。通过消除不必要的参数，类文件变得更小。快速路径选项降低了VM对lambda转化操作的内部限制，使它可以被当作一个“装箱”操作，并促进了拆箱优化。

#### 5. 序列化

我们的动态转化策略要求一个动态的序列化策略。如果希望能够从内部类转到使用动态代理，或者序列化的对象必须在反序列化时变成动态代理。这可以通过为lambda表达式定义一个中立的序列化形式，并使用readResolve和writeReplace在lambda对象和序列化形态之间进行转换。序列化形态必须包含通过metafactory重新创建对象所需的所有信息。

```
public interface SerializedLambda extends Serializable {
	// capture context
	String getCapturingClass();

	// SAM descriptor
	String getDescriptorClass();
	String getDescriptorName();
	String getDescriptorMethodType();

	// implementation
	int getImplReferenceKind();
	String getImplClass();
	String getImplName();
	String getImplMethodType();

	// dynamic args -- these will individually need to be Serializable too
	Object[] getCapturedArgs();
}
```

这里，SerializedLambda接口提供了原始lambda捕获站点上的所有信息。在捕获可序列化的lambda时，metafactory将不得不返回一个实现writeReplace方法的对象，返回一个SerializedLambda实现，该实现具有一个readResolve方法，该方法负责重新创建lambda对象。

##### 5.1. 可达性

反序列化代码需要为实现方法构造一个方法句柄。虽然序列化形式提供了所有信息——种类、类、名称和类型——因此可以通过暴露在MethodHandles.Lookup上的findXxx方法来进行构造。被引用的lambda方法或方法可能无法访问SerializedLambda实现(可能是方法不可访问的，或者是封闭类不可访问)。这对于lambda工厂站点来说并不是问题，因为实现的方法句柄是使用该类的可访问特权权限加载的。但是，为了避免引入安全风险，我们希望在反序列化时尽量少使用任何高级特权，当然也不要修改JVM的可访问性规则。

一种确保序列化lambda或方法引用的实现方法是公共类的公共方法。这可能已经是正确的(方法引用String::length)，或者可以很容易地成为true(对于公共类，我们可以提取lambda到public方法)。但是这也是不可取的，因为它以公共方法公开了内部信息，并且与我们对不可序列化的lambda的转化不一致。在某些情况下，它还需要一些重要的操作，比如在非公开类上创建一个公共的“sidecar”类。(在某种程度上，“公开”是不可避免的，因为这就是序列化所做的事情——为类提供了一个外部的公共实际构造函数。但我们希望尽量减少这种操作。

更好的方法是将反序列化返回给执行lambda捕获的类。一个关键的安全问题是，不能让反序列化机制允许攻击者构造一个lambda对象，并通过构造一个被修改的字节流来调用任意的私有方法。序列化自然地暴露了“构造函数”(函数接口、行为方法、捕获的arg类型)的特定组合;通过将其委派回捕获类，它可以验证字节流在继续之前是否为一个有效的组合。一旦验证成功，它就可以通过metafactory调用构造lambda，并使用它自己的可访问性权限来加载方法句柄。
要做到这一点，捕获类应该有一个可以被序列化层调用的帮助方法，比如有readObject、writeObject、readResolve和writeReplace。我们称之为$deserialize$(SerializedLambda)。序列化层需要的唯一特权操作是调用这个(可能是私有的)方法。
当编译一个捕获serializable lambdas的类时，编译器知道(函数接口、行为方法、捕获的参数类型)的组合被捕获为可序列化的lambda。$deserialize$方法应该只支持这些组合的反序列化。
以捕获两个可序列化的lambda为例:

```
class Foo {
	void moo() {
		SerializableComparator<String> byLength = (a,b) -> a.length() - b.length();
		SerializablePredicate<String> isEmpty = String::isEmpty;
		...
	}
}
```

可以转换成如下形式

```
class Foo {
	void moo() {
		SerializableComparator<String> byLength
			= indy(MH(serializableMetafactory), MH(invokeVirtual SerializableComparator.compare),
				   MH(invokeStatic lambda$1))());
		SerializablePredicate<String> isEmpty
			= indy(MH(serializableMetafactory), MH(invokeVirtual SerializablePredicate.apply),
				   MH(invokeVirtual String.isEmpty)());
		...
	}

	private static int lambda$1(String a, String b) { return a.length() - b.length(); }

	private static $deserialize$(SerializableLambda lambda) {
		switch(lambda.getImplName()) {
		case "lambda$1":
			if (lambda.getSamClass().equals("com/foo/SerializableComparator")
				 && lambda.getSamMethodName().equals("compare")
				 && lambda.getSamMethodDesc().equals("...")
				 && lambda.getImpleReferenceKind() == REF_invokeStatic
				 && lambda.getImplClass().equals("com/foo/Foo")
				 && lambda.getImplDesc().equals(...)
				 && lambda.getInvocationDesc().equals(...))
					 return indy(MH(serializableMetafactory),
								 MH(invokeVirtual SerializableComparator.compare),
								 MH(invokeStatic lambda$1))(lambda.getCapturedArgs()));
			break;

		case "isEmpty":
			if (lambda.getSamClass().equals("com/foo/SerializablePredicate"))
				 && lambda.getSamMethodName().equals("apply")
				 && lambda.getSamMethodDesc().equals("...")
				 && lambda.getImpleReferenceKind() == REF_invokeVirtual
				 && lambda.getImplClass().equals("java/lang/String")
				 && lambda.getImplDesc().equals(...)
				 && lambda.getInvocationDesc().equals(...))
					 return indy(MH(serializableMetafactory),
								 MH(invokeVirtual SerializablePredicate.apply),
								 MH(invokeVirtual String.isEmpty)(lambda.getCapturedArgs));

			break;
		}
		throw new ...;
	}
}
```

$deserialize$方法知道该类捕获了哪些lambda，因此可以在列表中检查提供的序列化形态，然后使用一个相同的调用站点重构lambda，它可以与捕获站点共享相同的bootstrap索引。(或者，它可以共享相同的实际捕获站点，因此，通过将捕获分解为私有方法，可以共享相同的链接状态，这可以简化下面的类缓存中的一些问题。
如果一个调用者欺骗我们去反序列化一个恶意的字节流，那么它只会为那些实际上是该编译单元中lambda转换的目标的方法工作，如果我们将其转换为可序列化的内部类，那么我们就会暴露出来。因为它与提取方法在同一个编译单元中，在重新编译时不引入额外的名称。
这有一个简单的、适用于所有lambda主体的提取策略——对可序列化和不可序列化的lambda使用相同的策略。它能让所有提取lambda body体的私有化，消除了对sidecar类或易访问性桥接方法的需求，并且唯一的特权操作是$deserialize$。
减少暴露在类加载的攻击(攻击者创建一个序列化lambda描述的意图迫使类加载静态初始化器)，最好是由SerializedLambda接口处理所有的标识符的类名,而非类对象。

##### 5.2/ 类缓存(Class caching)

在一些转化策略中，我们可能需要生成新类。例如，如果我们为每一个lambda生成一个类(在运行时而不是编译时的内部类)，我们将在第一次调用给定的lambda工厂站点时生成该类，此后对该lambda工厂站点的调用将重用第一次调用生成的类。
对于可序列化的lambda，可以触发类生成的两个点:捕获站点，以及在$deserialize$代码中对应的工厂站点。无论哪条种被触发，都是可取的(尽管不是必需的)，通过任一路径生成的对象都具有相同的类，这需要每个lambda捕获的唯一密钥，以及在两个捕获站点之间共享一个给定serializable lambda的缓存。

```
class SerializationExperiment {
	interface Foo extends Serializable { int m(); }

	public static void main(String[] args) {
		Foo f1, f2;
		if (args[0].equals("r")) {
			// read file 'foo.ser' and deserialize into f1
		}

		f2 = () -> 3;

		if (args[0].equals("w")) {
			// serialize f2 and write into file 'foo.ser'
			// read file 'foo.ser' and deserialize into f1
		}

		assert f1.getClass() == f2.getClass();
	}
}
```

如果将上面这段程序运行两次：

```
java -ea SerializationExperiment w
java -ea SerializationExperiment r
```

如果运行是成功的，无论是第一次反序列化的类还是第一次调用metafactory，都是可取的.

##### 5.3. 性能影响(Performance impact)

Serializability在lambdas上增加了一些额外开销，因为lambda对象需要携带足够的状态来有效地重新创建metafactory的静态和动态参数列表。这可能意味着实现类中的额外字段、额外的构造函数初始化工作以及对转化策略的约束(例如，我们不能使用方法处理代理，因为结果对象不会实现所需的writeReplace方法)。因此，最好单独对待序列化lambda，而不是让所有的lambdas序列化以及将这些开销强加于所有的lambda表达式。

#### 6. 其他(Miscellaneous)

##### 6.1. 桥接方法

函数式接口实际上可能有多个非对象方法，因为它可能有桥接方法。例如，在功能接口B中:

```
interface A<T> { void m(T t); }
interface B extends A<String> { void m(String s); }
```

B的主要方法是m(String)，但是B也有一个方法m(Object)，它是m(String)的桥接方法。(如果将B转换为A，并在结果上调用m，则调用将失败。)
当我们将lambda表达式转换为实现一个函数式接口(如 B)的对象时，我们必须确保所有的桥接方法都正确地连接到主方法，并使用适当的参数或返回类型适应(casting)。通过恶意的字节码生成或单独的编译工件，也可以找到在编译时不存在于函数式接口的额外方法。我们可以采用MethodHandleProxy所采取的快捷方式，而不是执行完整的JLS桥接计算算法和桥接程序，它将以相同的名称和方式将所有方法与主方法连接起来。(如果发现其中任何一个都与主方法不兼容，那么在调用时将出现ClassCastException，这只比将会抛出的链接错误信息稍微少一些。)我们可以让编译器在metafactory中包含一个已知的有效的编译时桥接签名的列表，但是这会增加类文件的大小。

##### 6.2. toString

一般来说，对lambda对象的toString方法是从Object继承而来的。但是，对于公共非合成方法的方法引用，可能希望用实现方法中的类和方法名称来实现toString。例如，对于String::size转换为IntFn，我让toString返回String::size()， java.lang.String::size()， String::size()为IntFn，等等。
TODO:如果我们支持lambda的理念，我们可能希望将toString结果从名称中派生出来，在这种情况下，名称必须以某种方式传入metafactory。