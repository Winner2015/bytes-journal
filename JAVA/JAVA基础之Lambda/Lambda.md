

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





