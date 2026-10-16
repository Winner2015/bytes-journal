

# 1、Lambda表达式

## 1.1 什么是Lambda

从java8出现以来**Lambda**，也可称为闭**包**（closure），是最重要的特性之一，它可以让我们用简洁流畅的代码完成一个功能。 很长一段时间java被吐槽是冗余和缺乏函数式编程能力的语言，随着函数式编程的流行java8种也引入了这种编程风格。

Lambda表达式是一段可以传递的代码，它的核心思想是将面向对象中的传递数据变成传递行为。 我们回顾一下在使用java8之前要做的事，之前我们编写一个线程时是这样的：

```java
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.println("do something.");      
    }
}
```

这实际上是一个**代码即数据**的例子，在run方法中是线程要执行的一个任务，但上面的代码中任务内容已经被规定死了。当我们有多个不同的任务时，需要重复编写如上代码。

设计匿名内部类的目的，就是为了方便 Java 程序员将代码作为数据传递。不过，匿名内部类还是不够简便。为了执行一个简单的任务逻辑，不得不加上6 行冗繁的样板代码。那如果是Lambda该怎么做?

`Runnable r = () -> System.out.println("do something.");`

这是一个没有名字的函数，也没有任何参数，再简单不过了。 使用`->`将参数和实现逻辑分离，当运行这个线程的时候执行的是`->`之后的代码片段，且编译器帮助我们做了类型推导。

如上所示，Lambda表达式一个常见的用法是取代某些匿名内部类，但Lambda表达式的作用不限于此。

刚接触Lambda表达式可能觉得它很神奇：不需要声明类或者方法的名字，就可以直接定义函数。这看似是编译器为匿名内部类简写提供的一个小把戏，但事实上并非如此，Lambda表达式实际上是通过`invokedynamic`指令来实现的。

## 1.2 基础语法

在Lambda中我们遵循如下的表达式来编写：

expression = (variable) -> action

- **variable**： 这是一个变量,一个占位符。像x,y,z,可以是多个变量；
- **action**：这是我们实现的代码逻辑部分,它可以是一行代码也可以是一个代码片段。

下面是Lambda表达式几种可能的书写形式。

```java
	Runnable run = () -> System.out.println("Hello World");// 1
	ActionListener listener = event -> System.out.println("button clicked");// 2
	Runnable multiLine = () -> {// 3
		System.out.println("Hello ");
		System.out.println("World");
	};
	BinaryOperator<Long> add = (Long x, Long y) -> x + y;// 4
	BinaryOperator<Long> addImplicit = (x, y) -> x + y;// 5
```

通过上例可以发现：

- Lambda表达式是有类型的，赋值操作的左边就是类型。Lambda表达式的类型实际上是对应接口的类型。
- Lambda表达式可以包含多行代码，需要用大括号把代码块括起来，就像写函数体那样。

大多数时候，Lambda表达式的参数表可以省略类型，就像代码2和5那样。这得益于`javac`的类型推导机制，编译器可以根据上下文推导出类型信息。

表面上看起来每个Lambda表达式都是原来匿名内部类的简写形式，该内部类实现了某个函数接口(`Functional Interface`)，但事实比这稍微复杂一些，这里不再展开。Java是强类型语言，无论有没有显式指明，每个变量和对象都必须有明确的类型，没有显式指定的时候编译器会尝试确定类型。

## 1.3 函数式接口

来看下jdk 8中的Runnable源码

```java
@FunctionalInterface
public interface Runnable {
	public abstract void run();
}
```

`Runnable`接口只有一个方法，大多数回调接口都拥有这个特征：比如`Callable`接口和`Comparator`接口。我们把这些只拥有一个方法的接口称为**函数式接口**。

我们并不需要额外的工作来声明一个接口是函数式接口：编译器会根据接口的结构自行判断（判断过程并非简单的对接口方法计数：一个接口可能冗余的定义了一个`Object`已经提供的方法，比如`toString()`，或者定义了静态方法或默认方法，这些都不属于函数式接口方法的范畴）。不过API作者们可以通过`@FunctionalInterface`注解来显式指定一个接口是函数式接口（以避免无意声明了一个符合函数式标准的接口），加上这个注解之后，编译器就会验证该接口是否满足函数式接口的要求。

Lambda表达式必须对应一个函数式接口，方法体其实就是函数接口的实现。编译器利用Lambda表达式所在上下文所期待的类型进行推导，这个被期待的类型被称为目标类型。Lambda表达式只能出现在目标类型为函数式接口的上下文中。

# 2、方法引用

我们通常使用Lambda表达式来创建匿名方法。然而，有时候我们仅仅是调用了一个已存在的方法。如下：

`Arrays.sort(stringsArray,(s1,s2)->s1.compareToIgnoreCase(s2));`

实际上，`compareToIgnoreCase`就是`String`类中现成的一个方法，在Java8中，我们可以直接通过方法引用来简写Lambda表达式中已经存在的方法。

`Arrays.sort(stringsArray, String::compareToIgnoreCase);`

这种特性就叫做**方法引用**(Method Reference)。

方法引用其实是Lambda表达式的一个简化写法，所引用的方法其实是Lambda表达式的方法体实现，语法也很简单，如下所示：

`ObjectReference::methodName`

方法引用是用来直接访问类或者实例的已经存在的方法。计算时，方法引用会创建函数式接口的一个实例。

当Lambda表达式中只是执行一个方法调用时，不用Lambda表达式，直接通过方法引用的形式可读性更高一些。方法引用是一种更简洁易懂的Lambda表达式。

方法引用的类型可分为以下四种：

## 2.1 静态方法引用

组成语法格式：`ClassName::staticMethodName`

静态方法引用比较容易理解，和静态方法调用相比，只是把【.】换为【::】。在目标类型兼容的任何地方，都可以使用静态方法引用。

例子：

　　`String::valueOf `  等价于Lambda表达式`(s) -> String.valueOf(s)`
　　`Math::pow`等价于Lambda表达式  `(x, y) -> Math.pow(x, y);`

## 2.2 特定实例对象的方法引用

这种语法与用于静态方法的语法类似，只不过这里使用对象引用而不是类名。实例方法引用又分以下三种类型：

### 2.2.1 实例上的实例方法引用

组成语法格式：`instanceReference::methodName`

如下示例，引用的方法是`myComparisonProvider` 对象的`compareByName`方法。

```java
	class ComparisonProvider {

		public int compareByName(Person a, Person b) {
			return a.getName().compareTo(b.getName());
		}

		public int compareByAge(Person a, Person b) {
			return a.getBirthday().compareTo(b.getBirthday());
		}
	}

	ComparisonProvider myComparisonProvider = new ComparisonProvider();
	Arrays.sort(rosterAsArray, myComparisonProvider::compareByName);
```

### 2.2.2 超类上的实例方法引用

组成语法格式：`super::methodName`

方法的名称由`methodName`指定，通过使用`super`，可以引用方法的超类版本。

例子：

还可以捕获`this`指针，`this::equals`  等价于Lambda表达式  `x -> this.equals(x);`

### 2.2.3 类型上的实例方法引用

组成语法格式：`ClassName::methodName`

注意：若类型的实例方法是泛型的，就需要在`::`分隔符前提供类型参数，或者（多数情况下）利用目标类型推导出其类型。

静态方法引用和类型上的实例方法引用拥有一样的语法。编译器会根据实际情况做出决定。一般我们不需要指定方法引用中的参数类型，因为编译器往往可以推导出结果，但如果需要我们也可以显式在`::`分隔符之前提供参数类型信息。

例子：

`String::toString` 等价于Lambda表达式 `(s) -> s.toString()`

这里不太容易理解，实例方法要通过对象来调用，方法引用对应Lambda，Lambda的第一个参数会成为调用实例方法的对象。

## 2.3 任意对象（属于同一个类）的实例方法引用

如下示例，这里引用的是字符串数组中任意一个对象的`compareToIgnoreCase`方法。

```java
String[] stringArray = { "Barbara", "James", "Mary", "John", "Patricia", "Robert", "Michael", "Linda" };
Arrays.sort(stringArray, String::compareToIgnoreCase);
```



## 2.4 构造方法引用

构造方法引用又分构造方法引用和数组构造方法引用。

### 2.4.1 构造方法引用（也可以称作构造器引用）

组成语法格式：`Class::new`

构造函数本质上是静态方法，只是方法名字比较特殊，使用的是new 关键字。
例子：

`String::new`， 等价于Lambda表达式 `() -> new String()`

### 2.4.2 数组构造方法引用

组成语法格式：`TypeName[]::new`

例子：

`int[]::new` 是一个含有一个参数的构造器引用，这个参数就是数组的长度。等价于Lambda表达式 ` x -> new int[x]`。

假想存在一个接收int参数的数组构造方法

```java
IntFunction<int[]> arrayMaker = int[]::new;
int[] array = arrayMaker.apply(10) // 创建数组 int[10]
```

# 3、变量作用域

## 3.1 匿名内部类中的外部变量

在Java的经典著作《Effective Java》、《Java Concurrency in Practice》里，大神们都提到：匿名函数里的变量引用，也叫做**变量引用泄露**，会导致线程安全问题，因此在Java8之前，如果在匿名类内部引用函数局部变量，必须将其声明为final，即不可变对象。(Python和Javascript从一开始就是为单线程而生的语言，一般也不会考虑这样的问题，所以它的外部变量是可以任意修改的)。

为什么必须要为final呢？

首先我们知道在内部类编译成功后，它会产生一个class文件，该class文件与外部类并不是同一class文件，仅仅只保留对外部类的引用。当外部类传入的参数需要被内部类调用时，从java程序的角度来看是直接被调用：

```java
public class OuterClass {
	public void display(final String name, String age) {
		class InnerClass {
			void display() {
				System.out.println(name);
			}
		}
	}
}
```

从上面代码中看好像name参数应该是被内部类直接调用？其实不然，在java编译之后实际的操作如下：

```java
public class OuterClass$InnerClass {
	public InnerClass(String name,String age){
        this.InnerClass$name = name;
        this.InnerClass$age = age;
    }

	public void display() {
		System.out.println(this.InnerClass$name + "----" + this.InnerClass$age);
	}
}
```

所以从上面代码来看，内部类并不是直接调用方法传递的参数，而是利用自身的构造器对传入的参数进行备份，自己内部方法调用的实际上时自己的属性而不是外部方法传递进来的参数。

在内部类中的属性和外部方法的参数两者从外表上看是同一个东西，但实际上却不是，所以他们两者是可以任意变化的，也就是说在内部类中我对属性的改变并不会影响到外部的形参，而然这从程序员的角度来看这是不可行的，毕竟站在程序的角度来看这两个根本就是同一个，如果内部类改变了，而外部方法的形参却没有改变，这是难以理解和不可接受的，所以为了保持参数的一致性，就规定使用final来避免形参的不改变。

简单理解就是，拷贝引用，为了避免引用值发生改变，例如被外部类的方法修改等，而导致内部类得到的值不一致，于是用final来让该引用不可改变。

## 3.2 Lambda中的外部变量

在Java8里，有了一些改动，现在我们可以这样写Lambda或者匿名类了：

```java
	public static Supplier<Integer> testClosure() {
		int i = 1;
		return () -> {
			return i;
		};
	}
```

这里我们不用写`final`了。但是，Java大神们说的引用泄露怎么办呢？其实本质没有变，只是Java8这里加了一个语法糖：在Lambda表达式以及匿名类内部，如果引用某局部变量，则直接将其视为final。我们直接看一段代码吧：

```java
	public static Supplier<Integer> testClosure() {
		int i = 1;
		i++;
		return () -> {
			return i; // 这里会出现编译错误
		};
	}
```

其实这里我们仅仅是省去了变量的`final`定义，这里i会强制被理解成`final`类型。很搞笑的是编译错误出现在Lambda表达式内部引用i的地方，而不是改变变量值的地方。这也是Java的Lambda的一个被人诟病的地方。只能说，强制闭包里变量必须为`final`，出于严谨性还可以接受，但是这个语法糖有点酸酸的感觉，还不如强制写`final`……

# 4、默认方法

在Java语言中，一个接口中定义的方法必须由实现类提供实现。但是当接口中加入新的API时，实现类按照约定也要修改实现，而Java8的API对现有接口也添加了很多方法，比如List接口中添加了`sort`方法。 如果按照之前的做法，那么所有的实现类都要实现`sort`方法，JDK的编写者们一定非常抓狂。

Java8种引入新的机制，支持在接口中声明方法同时提供实现。

## 4.1 接口内定义默认方法。

我们来看看在JDK8中上述List接口添加方法的问题是如何解决的

```java
	default void sort(Comparator<? super E> c) {
	    Object[] a = this.toArray();
	    Arrays.sort(a, (Comparator) c);
	    ListIterator<E> i = this.listIterator();
	    for (Object e ： a) {
	        i.next();
	        i.set((E) e);
	    }
	}
```

翻阅`List`接口的源码，其中加入一个默认方法`default void sort(Comparator<? super E> c)`。 在返回值之前加入`default`关键字，有了这个方法我们可以直接调用`sort`方法进行排序。

```java
	List<Integer> list = Arrays.asList(2, 7, 3, 1, 8, 6, 4);
	list.sort(Comparator.naturalOrder());
	System.out.println(list);
```

`Comparator.naturalOrder()`是一个自然排序的实现，这里可以自定义排序方案。你经常看到使用Java8操作集合的时候可以直接`foreach`的原因也是在`Iterable`接口中也新增了一个默认方法：`forEach`，该方法功能和 for 循环类似，但是允许用户使用一个Lambda表达式作为循环体。

和其它方法一样，默认方法也可以被继承。不过，当类型或者接口的超类拥有多个具有相同签名的方法时，我们就需要一套规则来解决这个冲突：

- 类的方法声明优先于接口默认方法。无论该方法是具体的还是抽象的。
- 被其它类型所覆盖的方法会被忽略。这条规则适用于超类型共享一个公共祖先的情况。

为了演示第二条规则，我们假设`Collection`和`List`接口均提供了`removeAll`的默认实现，然后`Queue`继承并覆盖了`Collection`中的默认方法。在下面的`implement`从句中，`List`中的方法声明会优先于`Queue`中的方法声明：

`class LinkedList<E> implements List<E>, Queue<E> { ... }`

当两个独立的默认方法相冲突或是默认方法和抽象方法相冲突时会产生编译错误。这时程序员需要显式覆盖超类方法。一般来说我们会定义一个默认方法，然后在其中显式选择超类方法：

```java
interface Robot implements Artist, Gun {
  default void draw() { Artist.super.draw(); }
}
```

最后，接口在`inherits`和`extends`从句中的声明顺序和它们被实现的顺序无关。

## 4.2 接口内定义静态方法 

除了默认方法，Java SE 8还在允许在接口中定义静态方法。这使得我们可以从接口直接调用和它相关的辅助方法，而不是从其它的类中调用（之前这样的类往往以对应接口的复数命名，例如`Collections`）。比如，我们一般需要使用静态辅助方法生成实现`Comparator`的比较器，在Java SE 8中我们可以直接把该静态方法定义在`Comparator`接口中：

```java
	public static <T, U extends Comparable<? super U>> Comparator<T> comparing(Function<T, U> keyExtractor) {
		return (c1, c2) -> keyExtractor.apply(c1).compareTo(keyExtractor.apply(c2));
	}
```

# 5、函数式接口

## 5.1 Function

该接口用来根据一个类型的数据得到另一个类型的数据，前者称为前置条件，后者称为后置条件，有进有出。

`Function`类包含四种方法，其中一个抽象方法`apply()`，两个`default`方法`andThen()`和`compose()`，以及一个静态方法`identity()`。 

实例化`Function`的时候需要实现其中的`apply()`方法，`apply`方法接收一个模板类型T作为输入参数， 返回模板类型R作为输出参数。

```java
public class FunctionDemo {
	public static void main(String[] args){
        // 整数字符串
        String str = "123";
        // 将字符串转换为整型数据 123
        // 使用匿名内部类创建Function接口的实现类对象
        Function<String,Integer> f = new Function<String,Integer>(){
            @Override
            public Integer apply(String s) {
                return Integer.parseInt(s);
            }
        };
        // 调用apply方法进行转换
        int num01 = f.apply(str);
        System.out.println(num01);
        // 使用Lambda表达式简化
        Function<String,Integer> ff = s ‐> Integer.parseInt(s);
        int num02 = ff.apply(str);
        System.out.println(num02);
        // 使用方法引用简化Lambda表达式
        Function<String,Integer> fff = Integer::parseInt;
        int num03 = fff.apply(str);
        System.out.println(num03);
	}
}
```

`andThen`方法接收一个`Function`类的实例，通过`andThen`可以将任意多个`Function`的`apply`方法调用连接起来。

```java
Function<String, String> function1 = string -> {
    System.out.println("function1输出了： " + string);
    return string;
};
Function<String, String> function2 = string -> {
    System.out.println("function2输出了： " + string);
    return string;
};
function1.andThen(function2).apply("hello world");
```

输出结果：

> function1输出了： hello world
> function2输出了： hello world

可以看到调用顺序是先调用`funtion1`然后调用`function2`。

`compose`方法和`andThen`方法一样接收一个另一个`Function`作为参数，但是顺序与`andThen`恰恰相反。 

接下来的测试用例，保持前面的不变，只把最后一句由`andThen`改成

`function1.compose(function2).apply("hello world");`

输出结果：

> function2输出了： hello world
> function1输出了： hello world

可以看到这次先执行了`function2`之后再执行`function1`。

`identity`方法是一个静态方法，作用是返回一个`Function`对象，返回的对象总是返回它被传入的值。 

## 5.2 Consumer

`Consumer`类包含两个方法，一个`accept`方法用来对输入的参数进行自定义操作，因为是个抽象方法，所以需要实例化对象的时候进行`Override`，另一个`andThen`方法跟`Function`的方法一样是一个`default`方法，已经有内部实现所以不需要用户重写，并且具体功能也跟`Function`差不多。`Consumer`的中文意思是消费者，意即通过传递进一个参数来对参数进行操作。 

```java
public class Test {
	public static void main(String[] args) {
		Foo f = new Foo();
		f.foo(new Consumer<Integer>() {
			@Override
			public void accept(Integer integer) {
				System.out.println(integer);
			}
		});
	}
}

class Foo {
	private int[] data = new int[10];

	public Foo() {
		for (int i = 0; i < 10; i++) {
			data[i] = i;
		}
	}

	public void foo(Consumer<Integer> consumer) {
		for (int i : data)
			consumer.accept(i);
	}
}
```

在上面的代码中，由于Java8引入的LambdaLambda表达式，所以其中的

```java
	f.foo(new Consumer<Integer>() {
		@Override
		public void accept(Integer integer) {
			System.out.println(integer);
		}
	});
```

可以简写成

`f.foo(integer -> System.out.println(integer));`

或者进一步简写成

`f.foo(System.out::println);`

## 5.3 Predicate

`Predicate`类包含5个方法，最重要的是`test`方法，这是一个抽象方法，需要编程者自己去`Override`，其他的三个`default`方法里都使用到了这个方法，这三个方法分别是`and`方法，`negate`方法和`or`方法，其中`and`和`or`方法与前面两个类的`andThen`方法类似，这两个方法都接受另一个`Predicate`对象作为参数，`and`方法返回这两个对象分别调用`test`方法之后得到的布尔值的并，相当于`predicate1.test() && predicate2.test()`，`or`方法返回这两个对象分别调用`test`方法之后得到的布尔值的或，相当于`predicate1.test() || predicate2.test()`。

```java
	Predicate<Integer> predicate1 = new Predicate<Integer>() {
		@Override
		public boolean test(Integer integer) {
			return integer <= 0;
		}
	};
	Predicate<Integer> predicate2 = new Predicate<Integer>() {
		@Override
		public boolean test(Integer integer) {
			return integer > 0;
		}
	};
	System.out.println("and: " + predicate1.and(predicate2).test(1));
	System.out.println("or: " + predicate1.or(predicate2).test(1));
	System.out.println("negate: " + predicate1.negate().test(1));
```

输出结果：

> and: false
> or: true
> negate: true

同样，可以简化成Lambda表达式

```java
	Predicate<Integer> predicate1 = integer -> integer <= 0;
	Predicate<Integer> predicate2 = integer -> integer > 0;
```

## 5.4 Supplier 

`supplier`的中文意思是提供者，跟`Consumer`类相反，`Supplier`类用于提供对象，它只有一个`get`方法，是一个抽象方法，需要编程者自定义想要返回的对象。 

```java
	Supplier<Integer> supplier = new Supplier<Integer>() {
		@Override
		public Integer get() {
			return new Random().nextInt(100);
		}
	};

	int[] ints = new int[10];
	for (int i = 0; i < 10; i++) {
		ints[i] = supplier.get();
	}
	Arrays.stream(ints).forEach(System.out::println);
```


首先自定义了一个`Supplier`对象，对于其`get`方法，每次都返回一个100以内的随机数，并在之后利用这个对象给一个长度为10的int数组赋值并输出。

util里的function包里并不仅仅只有这四个类，只是其中绝大部分都是由这四种衍生而来的，这个包主要是用于实现Java8最大的特性函数式编程，所以在很多其他的包的一些类中的很多地方都用到了这个function包里的类，更多的情况是作为方法中的一个匿名对象参数来使用，配合简洁的Lambda表达式使得程序可读性变得更好。



# 6、实现原理

## 6.1 实例解析

先从一个例子开始：

```java
public class LambdaTest {

	public static void print(String name, Print print) {
		print.print(name);
	}

	public static void main(String[] args) {
		String name = "Chen Longfei";
		String prefix = "hello, ";

		print(name, (t) -> System.out.println(t));

		// 与上一行不同的是，Lambda表达式的函数体中引用了外部变量‘prefix’
		print(name, (t) -> System.out.println(prefix + t));
	}

}

@FunctionalInterface
interface Print {
	void print(String name);
}
```



例子很简单，定义了一个函数式接口`Print` ，`main`方法中有两处代码以Lambda表达式的方式实现了print接口，分别打印出不带前缀与带前缀的名字。

运行程序，打印结果如下：

> Chen Longfei
> hello, Chen Longfei

而`(t) -> System.out.println(t)`与`(t) -> System.out.println(prefix + t))`之类的Lambda表达式到底是怎样被编译与调用的呢？

我们知道，编译器编译Java代码时经常在背地里“搞鬼”比如类的全限定名的补全，泛型的类型推断等，编译器耍的这些小聪明可以帮助我们写出更优雅、简洁、高效的代码。鉴于编译器的一贯作风，我们有理由怀疑，新颖而另类的Lambda表达式在编译时很可能会被改造过了。

下面通过`javap`反编译class文件一探究竟。

`javap`是jdk自带的一个字节码查看工具及反编译工具：

用法: `javap <options> <classes>`

其中, 可能的选项包括:

```
  -help  --help  -?        输出此用法消息
  -version                 版本信息
  -v  -verbose             输出附加信息
  -l                       输出行号和本地变量表
  -public                  仅显示公共类和成员
  -protected               显示受保护的/公共类和成员
  -package                 显示程序包/受保护的/公共类
                           和成员 (默认)
  -p  -private             显示所有类和成员
  -c                       对代码进行反汇编
  -s                       输出内部类型签名
  -sysinfo                 显示正在处理的类的
                           系统信息 (路径, 大小, 日期, MD5 散列)
  -constants               显示最终常量
  -classpath <path>        指定查找用户类文件的位置
  -cp <path>               指定查找用户类文件的位置
  -bootclasspath <path>    覆盖引导类文件的位置
```

`javap -p Print.class`

结果如下：

```java
interface test.Print {
  public abstract void print(java.lang.String);
}
```

`javap -p LambdaTest.class`

结果如下：

```
	// Compiled from "LambdaTest.java"
	public class test.LambdaTest
	{
	  public test.LambdaTest();

		public static void print(java.lang.String, test.Print);

		public static void main(java.lang.String[]);

		private static void Lambda$main$1(java.lang.String);

		private static void Lambda$main$0(java.lang.String, java.lang.String);
	}
```

可见，编译器对`Print`接口的改造比较小，只是为`print`方法添加了`public abstract`关键字，而对LambdaTest的变化就比较大了，添加了两个静态方法：

```java
private static void Lambda$main$1(java.lang.String);

private static void Lambda$main$0(java.lang.String, java.lang.String);
```

对比原生的java代码，很容易做出推测，这两个静态方法与两处Lambda表达式相关：

```java
print(name, (t) -> System.out.println(t));
print(name, (t) -> System.out.println(prefix + t));
```

到底有什么关联呢？使用`javap -p -v -c LambdaTest.class`查看更加详细的反编译结果：

```java
public class test.LambdaTest
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:  						 
   #1 = Methodref          #15.#30        // java/lang/Object."<init>":()V
   #2 = InterfaceMethodref #31.#32        // test/Print.print:(Ljava/lang/String;)V
   #3 = String             #33            // Chen Longfei
   #4 = String             #34            // hello,
   #5 = InvokeDynamic      #0:#39         // #0:print:(Ljava/lang/String;)Ltest/Print;
   #6 = Methodref          #14.#40        // test/LambdaTest.print:(Ljava/lang/String;Ltest/Print;)V
   #7 = InvokeDynamic      #1:#42         // #1:print:()Ltest/Print;
   #8 = Fieldref           #43.#44        // java/lang/System.out:Ljava/io/PrintStream;
   #9 = Methodref          #45.#46        // java/io/PrintStream.println:(Ljava/lang/String;)V
  #10 = Class              #47            // java/lang/StringBuilder
  #11 = Methodref          #10.#30        // java/lang/StringBuilder."<init>":()V
  #12 = Methodref          #10.#48        // java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder
;
  #13 = Methodref          #10.#49        // java/lang/StringBuilder.toString:()Ljava/lang/String;
  #14 = Class              #50            // test/LambdaTest
  #15 = Class              #51            // java/lang/Object
  #16 = Utf8               <init>
  #17 = Utf8               ()V
  #18 = Utf8               Code
  #19 = Utf8               LineNumberTable
  #20 = Utf8               print
  #21 = Utf8               (Ljava/lang/String;Ltest/Print;)V
  #22 = Utf8               main
  #23 = Utf8               ([Ljava/lang/String;)V
  #24 = Utf8               Lambda$main$1
  #25 = Utf8               (Ljava/lang/String;)V
  #26 = Utf8               Lambda$main$0
  #27 = Utf8               (Ljava/lang/String;Ljava/lang/String;)V
  #28 = Utf8               SourceFile
  #29 = Utf8               LambdaTest.java
  #30 = NameAndType        #16:#17        // "<init>":()V
  #31 = Class              #52            // test/Print
  #32 = NameAndType        #20:#25        // print:(Ljava/lang/String;)V
  #33 = Utf8               Chen Longfei
  #34 = Utf8               hello,
  #35 = Utf8               BootstrapMethods
  #36 = MethodHandle       #6:#53         // invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/inv
oke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/M
ethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #37 = MethodType         #25            //  (Ljava/lang/String;)V
  #38 = MethodHandle       #6:#54         // invokestatic test/LambdaTest.Lambda$main$0:(Ljava/lang/String;Ljava/lang/St
ring;)V
  #39 = NameAndType        #20:#55        // print:(Ljava/lang/String;)Ltest/Print;
  #40 = NameAndType        #20:#21        // print:(Ljava/lang/String;Ltest/Print;)V
  #41 = MethodHandle       #6:#56         // invokestatic test/LambdaTest.Lambda$main$1:(Ljava/lang/String;)V
  #42 = NameAndType        #20:#57        // print:()Ltest/Print;
  #43 = Class              #58            // java/lang/System
  #44 = NameAndType        #59:#60        // out:Ljava/io/PrintStream;
  #45 = Class              #61            // java/io/PrintStream
  #46 = NameAndType        #62:#25        // println:(Ljava/lang/String;)V
  #47 = Utf8               java/lang/StringBuilder
  #48 = NameAndType        #63:#64        // append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  #49 = NameAndType        #65:#66        // toString:()Ljava/lang/String;
  #50 = Utf8               test/LambdaTest
  #51 = Utf8               java/lang/Object
  #52 = Utf8               test/Print
  #53 = Methodref          #67.#68        // java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHan
dles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;L
java/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #54 = Methodref          #14.#69        // test/LambdaTest.Lambda$main$0:(Ljava/lang/String;Ljava/lang/String;)V
  #55 = Utf8               (Ljava/lang/String;)Ltest/Print;
  #56 = Methodref          #14.#70        // test/LambdaTest.Lambda$main$1:(Ljava/lang/String;)V
  #57 = Utf8               ()Ltest/Print;
  #58 = Utf8               java/lang/System
  #59 = Utf8               out
  #60 = Utf8               Ljava/io/PrintStream;
  #61 = Utf8               java/io/PrintStream
  #62 = Utf8               println
  #63 = Utf8               append
  #64 = Utf8               (Ljava/lang/String;)Ljava/lang/StringBuilder;
  #65 = Utf8               toString
  #66 = Utf8               ()Ljava/lang/String;
  #67 = Class              #71            // java/lang/invoke/LambdaMetafactory
  #68 = NameAndType        #72:#76        // metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava
/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/
lang/invoke/CallSite;
  #69 = NameAndType        #26:#27        // Lambda$main$0:(Ljava/lang/String;Ljava/lang/String;)V
  #70 = NameAndType        #24:#25        // Lambda$main$1:(Ljava/lang/String;)V
  #71 = Utf8               java/lang/invoke/LambdaMetafactory
  #72 = Utf8               metafactory
  #73 = Class              #78            // java/lang/invoke/MethodHandles$Lookup
  #74 = Utf8               Lookup
  #75 = Utf8               InnerClasses
  #76 = Utf8               (Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/
lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #77 = Class              #79            // java/lang/invoke/MethodHandles
  #78 = Utf8               java/lang/invoke/MethodHandles$Lookup
  #79 = Utf8               java/lang/invoke/MethodHandles
{
  public test.LambdaTest();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 6: 0

  public static void print(java.lang.String, test.Print);
    descriptor: (Ljava/lang/String;Ltest/Print;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_1
         1: aload_0
         2: invokeinterface #2,  2            // InterfaceMethod test/Print.print:(Ljava/lang/String;)V
         7: return
      LineNumberTable:
        line 9: 0
        line 10: 7

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: ldc           #3                  // String Chen Longfei
         2: astore_1
         3: ldc           #4                  // String hello,
         5: astore_2
         6: aload_1
         7: aload_2
         8: invokedynamic #5,  0              // InvokeDynamic #0:print:(Ljava/lang/String;)Ltest/Print;
        13: invokestatic  #6                  // Method print:(Ljava/lang/String;Ltest/Print;)V
        16: aload_1
        17: invokedynamic #7,  0              // InvokeDynamic #1:print:()Ltest/Print;
        22: invokestatic  #6                  // Method print:(Ljava/lang/String;Ltest/Print;)V
        25: return
      LineNumberTable:
        line 13: 0
        line 14: 3
        line 16: 6
        line 18: 16
        line 19: 25

  private static void Lambda$main$1(java.lang.String);
    descriptor: (Ljava/lang/String;)V
    flags: ACC_PRIVATE, ACC_STATIC, ACC_SYNTHETIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #8                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: aload_0
         4: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         7: return
      LineNumberTable:
        line 18: 0

  private static void Lambda$main$0(java.lang.String, java.lang.String);
    descriptor: (Ljava/lang/String;Ljava/lang/String;)V
    flags: ACC_PRIVATE, ACC_STATIC, ACC_SYNTHETIC
    Code:
      stack=3, locals=2, args_size=2
         0: getstatic     #8                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: new           #10                 // class java/lang/StringBuilder
         6: dup
         7: invokespecial #11                 // Method java/lang/StringBuilder."<init>":()V
        10: aload_0
        11: invokevirtual #12                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        14: aload_1
        15: invokevirtual #12                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        18: invokevirtual #13                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        21: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        24: return
      LineNumberTable:
        line 16: 0
}
SourceFile: "LambdaTest.java"
InnerClasses:
     public static final #74= #73 of #77; //Lookup=class java/lang/invoke/MethodHandles$Lookup of class java/lang/invoke/MethodHandles
BootstrapMethods:
  0: #36 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(
  Ljava/lang/invoke/MethodHandles$Lookup;
  Ljava/lang/String;
  Ljava/lang/invoke/MethodType;
  Ljava/lang/invoke/MethodType;
  Ljava/lang/invoke/MethodHandle;
  Ljava/lang/invoke/MethodType;)
  Ljava/lang/invoke/CallSite;
    Method arguments:
      #37 (Ljava/lang/String;)V
      #38 invokestatic test/LambdaTest.Lambda$main$0:(Ljava/lang/String;Ljava/lang/String;)V
      #37 (Ljava/lang/String;)V
  
  1: #36 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(
  Ljava/lang/invoke/MethodHandles$Lookup;
  Ljava/lang/String;
  Ljava/lang/invoke/MethodType;
  Ljava/lang/invoke/MethodType;
  Ljava/lang/invoke/MethodHandle;
  Ljava/lang/invoke/MethodType;)
  Ljava/lang/invoke/CallSite;
    Method arguments:
      #37 (Ljava/lang/String;)V
      #41 invokestatic test/LambdaTest.Lambda$main$1:(Ljava/lang/String;)V
      #37 (Ljava/lang/String;)V

```

这个 class 文件展示了三个主要部分：

- 常量池
- 构造方法和 main、print、Lambda$main$0、Lambda$main$1方法
- Lambda表达式生成的内部类。

重点看下`main`方法的实现：

```java
public static void main(java.lang.String[]);
descriptor: ([Ljava/lang/String;)V
flags: ACC_PUBLIC, ACC_STATIC
Code:
  stack=2, locals=3, args_size=1
  	 // 将字符串常量"Chen Longfei"从常量池压栈到操作数栈
  	 0: ldc           #3                  // String Chen Longfei
  	 
     // 将栈顶引用型数值存入第二个本地变，即 String name = "Chen Longfei"
     2: astore_1
     
     // 将字符串常量"hello,"从常量池压栈到操作数栈
     3: ldc           #4                  // String hello,
     
     // 将栈顶引用型数值存入第三个本地变量， 即  String prefix = "hello, "
     5: astore_2
     
     //将第二个引用类型本地变量推送至栈顶，即  name
     6: aload_1
     
     //将第三个引用类型本地变量推送至栈顶，即 prefix
     7: aload_2
     
     //通过invokedynamic指令创建Print接口的实匿名内部类，实现 (t) -> System.out.println(prefix + t)
     8: invokedynamic #5,  0              // InvokeDynamic #0:print:(Ljava/lang/String;)Ltest/Print;
    
     //调用静态方法print
     13: invokestatic  #6                  // Method print:(Ljava/lang/String;Ltest/Print;)V
    
     //将第二个引用类型本地变量推送至栈顶，即  name
     16: aload_1
     
     //通过invokedynamic指令创建Print接口的匿名内部类，实现 (t) -> System.out.println(t)
    17: invokedynamic #7,  0              // InvokeDynamic #1:print:()Ltest/Print;
    
     //调用静态方法print
    22: invokestatic  #6                  // Method print:(Ljava/lang/String;Ltest/Print;)V
    25: return
    ……

```

两个匿名内部类是通过`BootstrapMethods`方法创建的：

```java
   //匿名内部类
    InnerClasses:
        public static final #74= #73 of #77; //Lookup=class java/lang/invoke/MethodHandles$Lookup of class java/lang/invoke/MethodHandles
   BootstrapMethods:
	 //调用静态工厂LambdaMetafactory.metafactory创建匿名内部类1。实现了 (t) -> System.out.println(prefix + t)
     0: #36 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(
     Ljava/lang/invoke/MethodHandles$Lookup;
     Ljava/lang/String;
     Ljava/lang/invoke/MethodType;
     Ljava/lang/invoke/MethodType;
     Ljava/lang/invoke/MethodHandle;
     Ljava/lang/invoke/MethodType;)
     Ljava/lang/invoke/CallSite;
       Method arguments:
         #37 (Ljava/lang/String;)V
       //该类会调用静态方法LambdaTest.Lambda$main$0
         #38 invokestatic test/LambdaTest.Lambda$main$0:(Ljava/lang/String;Ljava/lang/String;)V
         #37 (Ljava/lang/String;)V
     
     //调用静态工厂LambdaMetafactory.metafactory创建匿名内部类2，实现了 (t) -> System.out.println(t)
     1: #36 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(
     Ljava/lang/invoke/MethodHandles$Lookup;
     Ljava/lang/String;
     Ljava/lang/invoke/MethodType;
     Ljava/lang/invoke/MethodType;
     Ljava/lang/invoke/MethodHandle;
     Ljava/lang/invoke/MethodType;)
     Ljava/lang/invoke/CallSite;
       Method arguments:
         #37 (Ljava/lang/String;)V
         //该类会调用静态方法LambdaTest.Lambda$main$1
         #41 invokestatic test/LambdaTest.Lambda$main$1:(Ljava/lang/String;)V
         #37 (Ljava/lang/String;)V

```

可以在运行时加上`-Djdk.internal.Lambda.dumpProxyClasses=%PATH%`，加上这个参数后，运行时，会将生成的内部类class输出到`%PATH%`路径下。

`javap -p -c` 反编译两个文件：

```java
//print(name, (t) -> System.out.println(t))的实例
final class test.LambdaTest$$Lambda$1 implements test.Print {
	  private test.LambdaTest$$Lambda$1(); //构造方法
	    Code:
	       0: aload_0
	       1: invokespecial #10                 // Method java/lang/Object."<init>":()V
	       4: return
	    		   
	  //实现test.Print接口方法
	  public void print(java.lang.String);
	    Code:
	       0: aload_1
	       //调用静态方法LambdaTest.Lambda$1
	       1: invokestatic  #18                 // Method test/LambdaTest.Lambda$1:(Ljava/lang/String;)V
	       4: return
	}

	
	//print(name, (t) -> System.out.println(prefix + t))的实例
	final class test.LambdaTest$$Lambda$2 implements test.Print {
	  
	  private final java.lang.String arg$1;

	  private test.LambdaTest$$Lambda$2(java.lang.String);
	    Code:
	       0: aload_0
	       1: invokespecial #13                 // Method java/lang/Object."<init>":()V
	       4: aload_0
	       5: aload_1
	       //final变量arg$1由构造方法传入
	       6: putfield      #15                 // Field arg$1:Ljava/lang/String;
	       9: return

	  //该方法返回一个 LambdaTest$$Lambda$2实例
	  private static test.Print get$Lambda(java.lang.String);
	    Code:
	       0: new           #2                  // class test/LambdaTest$$Lambda$2
	       3: dup
	       4: aload_0
	       5: invokespecial #19                 // Method "<init>":(Ljava/lang/String;)V
	       8: areturn
	       
	  //实现test.Print接口方法
	  public void print(java.lang.String);
	    Code:
	       0: aload_0
	       1: getfield      #15                 // Field arg$1:Ljava/lang/String;
	       4: aload_1
	       
	       //调用静态方法LambdaTest.Lambda$0
	       5: invokestatic  #27                 // Method test/LambdaTest.Lambda$0:(Ljava/lang/String;Ljava/lang/String;)V
	       8: return
	}
```

对比两个实例，可以发现，由于表达式`print(name, (t) -> System.out.println(prefix + t))`引用了局部变量`prefix`，`LambdaTest$$Lambda$2`类多了一个`final`参数：

`private final java.lang.String arg$1`

该参数由构造方法传入，用来存储`main`方法中的局部变量`prefix`：

`String prefix = "hello, ";`
		

由于外部类的`main`方法与匿名内部类`LambdaTest$$Lambda$2`引用了同一份变量，该变量虽然在代码层面独立存储于两个类当中，但是在逻辑上具有一致性，所以匿名内部类中加上了`final`关键字，而外部类中虽然没有为`prefix`显式地添加`final`，但是在被Lambda表达式引用后，该变量就自动隐含了`final`语意（再次更改会报错）。
## 6.2 InvokeDynamic

通过上面的例子可以发现，Lambda表达式由虚拟机指令`InvokeDynamic`实现方法调用。

### 6.2.1 方法调用

方法调用不等同于方法执行，方法调用阶段的唯一任务就是确定被调用方法的版本（即确定具体调用那一个方法），不涉及方法内部具体运行。

java虚拟机中提供了5条方法调用的字节码指令：

- **invokestatic**：调用静态方法
- **invokespecial**：调用实例构造器<init>方法、私有方法、父类方法
- **invokevirtual**：调用虚方法。
- **invokeinterface**：调用接口方法，在运行时再确定一个实现该接口的对象
- **invokedynamic**：运行时动态解析出调用的方法，然后去执行该方法。

`invokeDynamic`是 java 7 引入的一条新的虚拟机指令，这是自 1.0 以来第一次引入新的虚拟机指令。到了 java 8 这条指令才第一次在 java 应用，用在 Lambda 表达式中。`invokeDynamic`与其他invoke指令不同的是它允许由应用级的代码来决定方法解析。

### 6.2.2 指令规范

根据JVM规范的规定，`invokeDynamic`的操作码是`186（0xBA）`，格式是：

`invokedynamic indexbyte1 indexbyte2 0 0`

`invokeDynamic`指令有四个操作数，前两个操作数构成一个索引[ (indexbyte1 << 8) | indexbyte2 ]，指向类的常量池，后两个操作数保留，必须是0。

查看上例中LambdaTest类的反编译结果，第一处Lambda表达式

`print(name, (t) -> System.out.println(t));`

对应的指令为：

 `17: invokedynamic #7,  0              // InvokeDynamic #1:print:()Ltest/Print;`

常量池中`#7`对应的常量为：  

`#7 = InvokeDynamic      #1:#42         // #1:print:()Ltest/Print;`

其类型为`CONSTANT_InvokeDynamic_info`，`CONSTANT_InvokeDynamic_info`结构是Java7新引入class文件的，其用途就是给`invokeDynamic`指令指定启动方法（bootstrap method）、调用点`call site（）`等信息, 实际上是个 `MethodHandle`（方法句柄）对象。

`#1`代表BootstrapMethods表中的索引，即

BootstrapMethods：

```java

//第一个
0: #36 …… 

 //第二个
  1: #36 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(
  Ljava/lang/invoke/MethodHandles$Lookup;
  Ljava/lang/String;
  Ljava/lang/invoke/MethodType;
  Ljava/lang/invoke/MethodType;
  Ljava/lang/invoke/MethodHandle;
  Ljava/lang/invoke/MethodType;)
  Ljava/lang/invoke/CallSite;
    Method arguments:

      # 37 (Ljava/lang/String;)V
      # 41 invokestatic test/LambdaTest.Lambda$main$1:(Ljava/lang/String;)V
      # 37 (Ljava/lang/String;)V
```


也就是说，最终调用的是`java.lang.invoke.LambdaMetafactory`类的静态方法`metafactory()`。

### 6.2.3 执行过程

为了更深入的了解`invokeDynamic`，先来看几个术语：

- **dynamic call site**
  程序中出现Lambda的地方都被称作`dynamic call site`，CallSite 就是一个 `MethodHandle`（方法句柄）的 `holder`。方法句柄指向一个调用点真正执行的方法。
- **bootstrap method**
  java里对所有Lambda的有统一的`bootstrap method`(`LambdaMetafactory.metafactory`)，`bootstrap`运行期动态生成了匿名类，将其与CallSite绑定，得到了一个获取匿名类实例的`call site object`
- **call site object**
  `call site object`持有`MethodHandle`的引用作为它的target，它是`bootstrap method`方法成功调用后的结果，将会与 `dynamic call site`永久绑定。`call site object`的target会被JVM执行，就如同执行一条`invokevirtual`指令，其所需的参数也会被压入`operand stack`。最后会得一个实现了`functional interface`的对象。

`InvokeDynamic` 首先需要生成一个 `CallSite`（调用点对象），`CallSite` 是由 `bootstrap method` 返回，也就是调`LambdaMetafactory.metafactory`方法。

```java
	public static CallSite metafactory(MethodHandles.Lookup caller, String invokedName, MethodType invokedType,
			MethodType samMethodType, MethodHandle implMethod, MethodType instantiatedMethodType)
			throws LambdaConversionException {
		AbstractValidatingLambdaMetafactory mf;
		mf = new InnerClassLambdaMetafactory(caller, invokedType, invokedName, samMethodType, implMethod,
				instantiatedMethodType, false, EMPTY_CLASS_ARRAY, EMPTY_MT_ARRAY);
		mf.validateMetafactoryArgs();
		return mf.buildCallSite();
	}
```

前三个参数是固定的，由VM自动压栈：

- `MethodHandles.Lookup caller`代表`InvokeDynamic` 指令所在的类的上下文（在上例中就是LambdaTest），可以通过 `Lookup#lookupClass()`获取这个类
- `String invokedName`表示要实现的方法名（在上例中就是Print接口的方法名“print”）
- `MethodType invokedType call site object`所持有的`MethodHandle`需要的参数和返回类型（signature）

接下来就是附加参数，这些参数是灵活的，由`Bootstrap methods`表提供： 

- `MethodType samMethodType`表示要实现`functional interface`里面抽象方法的类型
- `MethodHandle implMethod`表示编译器给生成的 `desugar` 方法，是一个 `MethodHandle`
- `MethodType instantiatedMethodType`即运行时的类型，因为方法定义可能是泛型，传入时可能是具体类型String之类的，要做类型校验强转等等 

`LambdaMetafactory.metafactory` 方法会创建一个`VM Anonymous Class`，这个类是通过 ASM 编织字节码在内存中生成的，然后直接通过 `UNSAFE` 直接加载而不会写到文件里。`VM Anonymous Class` 是真正意义上的匿名类，不需要 ClassLoader 加载，没有类名，当然也没其他权限管理等操作，这意味着效率更高(不必要的锁操作)、GC 更方便（没有 ClassLoader）。

### 6.2.4 MethodHandle

要让`invokedynamic`正常运行，一个核心的概念就是**方法句柄**（method handle）。它代表了一个可以从`invokedynamic`调用点进行调用的方法。每个`invokedynamic`指令都会与一个特定的方法关联（也就是bootstrap method或BSM）。当编译器遇到`invokedynamic`指令的时候，BSM会被调用，会返回一个包含了方法句柄的对象，这个对象表明了调用点要实际执行哪个方法。

Java 7 API中加入了`java.lang.invoke.MethodHandle`（及其子类），通过它们来代表`invokedynamic`指向的方法。
一个Java方法可以视为由四个基本内容所构成：

- 名称
- 签名（包含返回类型）
- 定义它的类
- 实现方法的字节码

这意味着如果要引用某个方法，我们需要有一种有效的方式来表示方法签名（而不是反射中强制使用的令人讨厌的`Class<?>[] hack`方式）。

方法句柄首先需要的一个表达方法签名的方式，以便于查找。在Java 7引入的Method Handles API中，这个角色是由`java.lang.invoke.MethodType`类来完成的，它使用一个不可变的实例来代表签名。要获取`MethodType`，我们可以使用`methodType()`工厂方法。这是一个参数可变的方法，以class对象作为参数。

第一个参数所使用的class对象，对应着签名的返回类型；剩余参数中所使用的class对象，对应着签名中方法参数的类型。例如：

```java
//toString()的签名
MethodType mtToString = MethodType.methodType(String.class);

// setter方法的签名
MethodType mtSetter = MethodType.methodType(void.class, Object.class);

// Comparator中compare()方法的签名
MethodType mtStringComparator = MethodType.methodType(int.class, String.class, String.class); 
```

现在我们就可以使用`MethodType`，再组合方法名称以及定义方法的类来查找方法句柄。要实现这一点，我们需要调用静态的`MethodHandles.lookup()`方法。这样的话，会给我们一个“查找上下文（lookup context）”，这个上下文基于当前正在执行的方法（也就是调用lookup()的方法）的访问权限。

查找上下文对象有一些以“find”开头的方法，例如，`findVirtual()`、`findConstructor()`、`findStatic()`等。这些方法将会返回实际的方法句柄，需要注意的是，只有在创建查找上下文的方法能够访问（调用）被请求方法的情况下，才会返回句柄。这与反射不同，我们没有办法绕过访问控制。换句话说，方法句柄中并没有与`setAccessible()`对应的方法。例如

```java
	public MethodHandle getToStringMH() {
		MethodHandle mh = null;
		MethodType mt = MethodType.methodType(String.class);
		MethodHandles.Lookup lk = MethodHandles.lookup();

		try {
			mh = lk.findVirtual(getClass(), "toString", mt);
		} catch (NoSuchMethodException | IllegalAccessException mhx) {
			throw (AssertionError) new AssertionError().initCause(mhx);
		}

		return mh;
	}
```

`MethodHandle`中有两个方法能够触发对方法句柄的调用，那就是`invoke()`和`invokeExact()`。这两个方法都是以接收者（receiver）和调用变量作为参数，所以它们的签名为：

```java
public final Object invoke(Object... args) throws Throwable;
public final Object invokeExact(Object... args) throws Throwable; 
```

两者的区别在于，`invokeExact()`在调用方法句柄时会试图严格地直接匹配所提供的变量。而`invoke()`与之不同，在需要的时候，`invoke()`能够稍微调整一下方法的变量。`invoke()`会执行一个`asType()`转换，它会根据如下的这组规则来进行变量的转换：

- 如果需要的话，原始类型会进行装箱操作

- 如果需要的话，装箱后的原始类型会进行拆箱操作

- 如果必要的话，原始类型会进行扩展

- void返回类型会转换为0（对于返回原始类型的情况），而对于预期得到引用类型的返回值的地方，将会转换为null

- null值会被视为正确的，不管静态类型是什么都可以进行传递


接下来，我们看一下考虑上述规则的简单调用样例：

```java
	Object rcvr = "a";
	try {
		MethodType mt = MethodType.methodType(int.class);
		MethodHandles.Lookup l = MethodHandles.lookup();
		MethodHandle mh = l.findVirtual(rcvr.getClass(), "hashCode", mt);

		int ret;
		try {
			ret = (int) mh.invoke(rcvr);
			System.out.println(ret);
		} catch (Throwable t) {
			t.printStackTrace();
		}
	} catch (IllegalArgumentException | NoSuchMethodException | SecurityException e) {
		e.printStackTrace();
	} catch (IllegalAccessException x) {
		x.printStackTrace();
	}
```

上面的代码调用了`Object`的`hashcode()`方法，看到这里，你肯定会说这不就是 Java 的反射吗？

确实，MethodHandl` 和 Reflection`实现的功能有太多相似的地方，都是运行时解析方法调用，理解方法句柄的一种方式就是将其视为以安全、现代的方式来实现反射的核心功能，在这个过程会尽可能地保证类型的安全。

但是，究其本质，两者之间还是有区别的：

Reflection中的`java.lang.reflect.Method`对象远比MethodHandl`机制中的`java.lang.invoke.MethodHandle`对象所包含的信息来得多。前者是方法在Java一端的全面映像，包含了方法的签名、描述符以及方法属性表中各种属性的Java端表示方式，还包含有执行权限等的运行期信息。而后者仅仅包含着与执行该方法相关的信息。用开发人员通俗的话来讲，**Reflection是重量级，而MethodHandle是轻量级**。

- 从性能角度上说，MethodHandle 要比反射快很多，因为访问检查在创建的时候就已经完成了，而不是像反射一样等到运行时候才检查
- Reflection是在模拟Java代码层次的方法调用，而MethodHandle是在模拟字节码层次的方法调用。
- MethodHandle 是结合 `invokedynamic` 指令一起为动态语言服务的，也就是说MethodHandle （更准确的来说是其设计理念）是服务于所有运行在JVM之上的语言，而 Relection 则只是适用 Java 语言本身。
