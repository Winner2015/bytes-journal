# Lambda

# 1 Lambda 表达式

## 1.1 什么是 Lambda

Lambda 表达式是 Java 8 引入的最重要的特性之一，它可以让我们用简洁流畅的代码完成一个功能。很长一段时间里，Java 被吐槽是冗长、缺乏函数式编程能力的语言，随着函数式编程的流行，Java 8 也引入了这种编程风格。Lambda 表达式可以捕获外部作用域中的变量，因此常被看作 Java 对闭包（Closure）的一种有限支持。

Lambda 表达式是一段可以传递的代码，它的核心思想是将面向对象中的传递数据变成传递行为。我们回顾一下在使用 Java 8 之前要做的事，之前我们编写一个线程时是这样的：

```java
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.println("do something.");
    }
};
```

这实际上是一个**代码即数据**的例子，`run` 方法中是线程要执行的一个任务，但上面的代码中任务内容已经被规定死了。当我们有多个不同的任务时，需要重复编写如上代码。

设计匿名内部类的目的，就是为了方便 Java 程序员将代码作为数据传递。不过，匿名内部类还是不够简便。为了执行一个简单的任务逻辑，不得不加上好几行冗繁的样板代码。那如果是 Lambda 该怎么做？

`Runnable r = () -> System.out.println("do something.");`

这是一个没有名字的函数，也没有任何参数，再简单不过了。使用 `->` 将参数和实现逻辑分离，当运行这个线程的时候执行的是 `->` 之后的代码片段，且编译器帮助我们做了类型推导。

如上所示，Lambda 表达式一个常见的用法是取代某些匿名内部类，但 Lambda 表达式的作用不限于此。

刚接触 Lambda 表达式可能觉得它很神奇：不需要声明类或者方法的名字，就可以直接定义函数。这看似是编译器为匿名内部类简写提供的一个小把戏，但事实上并非如此，Lambda 表达式实际上是通过 `invokedynamic` 指令来实现的（见第 6 章）。

## 1.2 基础语法

Lambda 表达式有以下两种基本形式：

```text
(parameters) -> expression
(parameters) -> { statements; }
```

- **parameters**：参数列表，可以有零个、一个或多个参数，只有一个参数且省略类型时可以省略圆括号；
- **expression / statements**：我们实现的代码逻辑部分，可以是一个表达式，也可以是用大括号括起来的代码块。

下面是 Lambda 表达式几种可能的书写形式。

```java
Runnable run = () -> System.out.println("Hello World"); // 1
ActionListener listener = event -> System.out.println("button clicked"); // 2
Runnable multiLine = () -> { // 3
    System.out.println("Hello ");
    System.out.println("World");
};
BinaryOperator<Long> add = (Long x, Long y) -> x + y; // 4
BinaryOperator<Long> addImplicit = (x, y) -> x + y; // 5
```

通过上例可以发现：

- Lambda 表达式是有类型的，赋值操作的左边就是类型。Lambda 表达式的类型实际上是对应接口的类型。
- Lambda 表达式可以包含多行代码，需要用大括号把代码块括起来，就像写函数体那样。

大多数时候，Lambda 表达式的参数表可以省略类型，就像代码 2 和 5 那样。这得益于 `javac` 的类型推导机制，编译器可以根据上下文推导出类型信息。

表面上看起来每个 Lambda 表达式都是原来匿名内部类的简写形式，该内部类实现了某个函数式接口（Functional Interface），但事实比这稍微复杂一些（见第 6 章）。Java 是强类型语言，无论有没有显式指明，每个变量和对象都必须有明确的类型，没有显式指定的时候编译器会尝试确定类型。

## 1.3 函数式接口

来看下 JDK 8 中的 `Runnable` 源码：

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

`Runnable` 接口只有一个抽象方法，大多数回调接口都拥有这个特征：比如 `Callable` 接口和 `Comparator` 接口。我们把这些只拥有一个抽象方法的接口称为**函数式接口**。

我们并不需要额外的工作来声明一个接口是函数式接口：编译器会根据接口的结构自行判断（判断过程并非简单地对接口方法计数：一个接口可能冗余地定义了一个 `Object` 已经提供的方法，比如 `toString()`，或者定义了静态方法或默认方法，这些都不属于函数式接口方法的范畴）。不过 API 作者们可以通过 `@FunctionalInterface` 注解来显式指定一个接口是函数式接口（以避免无意中破坏了函数式接口的约束），加上这个注解之后，编译器就会验证该接口是否满足函数式接口的要求。

Lambda 表达式必须对应一个函数式接口，方法体其实就是函数式接口中抽象方法的实现。编译器利用 Lambda 表达式所在上下文所期待的类型进行推导，这个被期待的类型被称为目标类型。Lambda 表达式只能出现在目标类型为函数式接口的上下文中。

# 2 方法引用

我们通常使用 Lambda 表达式来创建匿名方法。然而，有时候我们仅仅是调用了一个已存在的方法。如下：

`Arrays.sort(stringsArray, (s1, s2) -> s1.compareToIgnoreCase(s2));`

实际上，`compareToIgnoreCase` 就是 `String` 类中现成的一个方法，在 Java 8 中，我们可以直接通过方法引用来简写 Lambda 表达式中已经存在的方法。

`Arrays.sort(stringsArray, String::compareToIgnoreCase);`

这种特性就叫做**方法引用**（Method Reference）。

方法引用其实是 Lambda 表达式的一个简化写法，所引用的方法其实是 Lambda 表达式的方法体实现，语法也很简单，如下所示：

`类名或对象引用::方法名`

方法引用是用来直接访问类或者实例的已经存在的方法。计算时，方法引用会创建函数式接口的一个实例。

当 Lambda 表达式中只是执行一个方法调用时，不用 Lambda 表达式，直接通过方法引用的形式可读性更高一些。方法引用是一种更简洁易懂的 Lambda 表达式。

方法引用的类型可分为以下四种：

## 2.1 静态方法引用

组成语法格式：`ClassName::staticMethodName`

静态方法引用比较容易理解，和静态方法调用相比，只是把“.”换为“::”。在目标类型兼容的任何地方，都可以使用静态方法引用。

例子：

- `String::valueOf` 等价于 Lambda 表达式 `(s) -> String.valueOf(s)`
- `Math::pow` 等价于 Lambda 表达式 `(x, y) -> Math.pow(x, y)`

## 2.2 特定对象的实例方法引用

这种语法与用于静态方法的语法类似，只不过这里使用对象引用而不是类名。

### 2.2.1 实例上的实例方法引用

组成语法格式：`instanceReference::methodName`

如下示例，引用的方法是 `myComparisonProvider` 对象的 `compareByName` 方法。

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

### 2.2.2 this 与 super 上的实例方法引用

组成语法格式：`this::methodName`、`super::methodName`

方法的名称由 `methodName` 指定，通过使用 `this` 可以引用当前对象的方法，通过使用 `super` 可以引用方法的超类版本。

例子：

- `this::equals` 等价于 Lambda 表达式 `x -> this.equals(x)`
- `super::toString` 等价于 Lambda 表达式 `() -> super.toString()`

## 2.3 特定类型任意对象的实例方法引用

组成语法格式：`ClassName::methodName`

这种形式与静态方法引用的语法一样，编译器会根据实际情况做出决定。它不太容易理解：实例方法要通过对象来调用，方法引用对应 Lambda，**Lambda 的第一个参数会成为调用实例方法的对象**，其余参数作为方法的参数。

例子：

- `String::toString` 等价于 Lambda 表达式 `(s) -> s.toString()`
- `String::compareToIgnoreCase` 等价于 Lambda 表达式 `(a, b) -> a.compareToIgnoreCase(b)`

如下示例，这里引用的是字符串数组中任意一个对象的 `compareToIgnoreCase` 方法。

```java
String[] stringArray = { "Barbara", "James", "Mary", "John", "Patricia", "Robert", "Michael", "Linda" };
Arrays.sort(stringArray, String::compareToIgnoreCase);
```

注意：若类型的实例方法是泛型的，就需要在 `::` 分隔符前提供类型参数，或者（多数情况下）利用目标类型推导出其类型。一般我们不需要指定方法引用中的参数类型，因为编译器往往可以推导出结果。

## 2.4 构造方法引用

构造方法引用又分构造方法引用和数组构造方法引用。

### 2.4.1 构造方法引用（也可以称作构造器引用）

组成语法格式：`ClassName::new`

构造方法并不是静态方法（在字节码中它是名为 `<init>` 的实例初始化方法），这里只是借用 `new` 关键字作为方法名。具体引用哪个构造方法，由目标类型决定。

例子：

- `Supplier<String> s = String::new;` 等价于 Lambda 表达式 `() -> new String()`
- `Function<String, String> f = String::new;` 等价于 Lambda 表达式 `str -> new String(str)`

### 2.4.2 数组构造方法引用

组成语法格式：`TypeName[]::new`

例子：

`int[]::new` 是一个含有一个参数的构造器引用，这个参数就是数组的长度，等价于 Lambda 表达式 `x -> new int[x]`。可以理解为假想存在一个接收 `int` 参数的数组构造方法：

```java
IntFunction<int[]> arrayMaker = int[]::new;
int[] array = arrayMaker.apply(10); // 创建数组 int[10]
```

数组构造方法引用常用于 Stream 转数组，例如 `stream.toArray(String[]::new)`。

# 3 变量作用域

## 3.1 匿名内部类中的外部变量

在 Java 8 之前，如果在匿名内部类（或局部内部类）中引用外部方法的局部变量，必须将其声明为 `final`。

为什么必须要为 `final` 呢？

首先我们知道在内部类编译成功后，它会产生一个 class 文件，该 class 文件与外部类并不是同一个 class 文件，仅仅只保留对外部类的引用。当外部方法的参数需要被内部类使用时，从 Java 程序的角度来看是直接被调用：

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

从上面代码中看好像 `name` 参数是被内部类直接调用的？其实不然，Java 编译之后，内部类大致相当于如下代码：

```java
class OuterClass$1InnerClass {
    final OuterClass this$0;
    final String val$name;

    OuterClass$1InnerClass(OuterClass this$0, String name) {
        this.this$0 = this$0;
        this.val$name = name;
    }

    void display() {
        System.out.println(this.val$name);
    }
}
```

所以从上面代码来看，内部类并不是直接使用方法传递的参数，而是利用自身的构造器对传入的参数进行备份，自己内部方法调用的实际上是自己的属性，而不是外部方法传递进来的参数。另外，编译器只会捕获内部类中实际用到的变量，没有用到的 `age` 不会被复制。

内部类中的属性和外部方法的参数两者从外表上看是同一个东西，但实际上却是两份独立的拷贝。如果允许修改，那么在内部类中对属性的改变并不会影响到外部的形参，反之亦然。然而从程序员的角度来看，这两个根本就是同一个变量，如果一边改变了而另一边没有改变，这是难以理解和不可接受的。所以为了保持两者的一致性，就规定使用 `final` 来保证变量不被修改。

简单理解就是：Java 的内部类捕获的是变量值的拷贝，为了避免原变量或拷贝被修改而导致两边的值不一致，于是用 `final` 让该变量不可改变。这同时也避免了多个线程共享可变局部变量带来的并发问题。

相比之下，JavaScript、Python 等语言的闭包捕获的是变量本身而不是值的拷贝，因此可以在闭包中修改外部变量（Python 需要使用 `nonlocal` 声明）。

## 3.2 Lambda 中的外部变量

Java 8 引入了**事实上的 final**（effectively final）的概念：如果一个局部变量在初始化后从未被重新赋值，即使没有声明 `final`，编译器也会将其视为 `final`。因此，现在我们可以这样写 Lambda 或者匿名类了：

```java
public static Supplier<Integer> testClosure() {
    int i = 1;
    return () -> {
        return i;
    };
}
```

这里我们不用写 `final` 了，但本质没有变，Lambda 捕获的依然是变量值的拷贝。如果变量被重新赋值，它就不再是“事实上的 final”，也就不能被 Lambda 引用：

```java
public static Supplier<Integer> testClosure() {
    int i = 1;
    i++;
    return () -> {
        return i; // 这里会出现编译错误
    };
}
```

编译器会报错：`local variables referenced from a lambda expression must be final or effectively final`。值得注意的是，编译错误出现在 Lambda 表达式内部引用 `i` 的地方，而不是修改变量值的地方，初学者往往会对此感到困惑。

需要说明的是，这个限制只针对**局部变量**。Lambda 中可以自由读写外部类的成员变量和静态变量，因为它捕获的是 `this` 引用，而不是成员变量的拷贝。如果确实需要在 Lambda 中修改一个计数器之类的局部状态，可以使用 `AtomicInteger`、单元素数组等可变容器，但更推荐的做法是改用 Stream 的 `reduce`、`count` 等无副作用的写法。

# 4 默认方法

在 Java 语言中，一个接口中定义的方法必须由实现类提供实现。但是当接口中加入新的 API 时，实现类按照约定也要修改实现，而 Java 8 的 API 对现有接口也添加了很多方法，比如 `List` 接口中添加了 `sort` 方法。如果按照之前的做法，那么所有的实现类都要实现 `sort` 方法，这会破坏大量已有代码的兼容性。

Java 8 中引入了新的机制，支持在接口中声明方法的同时提供实现。

## 4.1 接口内定义默认方法

我们来看看在 JDK 8 中上述 `List` 接口添加方法的问题是如何解决的：

```java
default void sort(Comparator<? super E> c) {
    Object[] a = this.toArray();
    Arrays.sort(a, (Comparator) c);
    ListIterator<E> i = this.listIterator();
    for (Object e : a) {
        i.next();
        i.set((E) e);
    }
}
```

翻阅 `List` 接口的源码，其中加入一个默认方法 `default void sort(Comparator<? super E> c)`。在返回值之前加入 `default` 关键字，有了这个方法我们可以直接调用 `sort` 方法进行排序。

```java
List<Integer> list = Arrays.asList(2, 7, 3, 1, 8, 6, 4);
list.sort(Comparator.naturalOrder());
System.out.println(list); // [1, 2, 3, 4, 6, 7, 8]
```

`Comparator.naturalOrder()` 是一个自然排序的实现，这里可以自定义排序方案。你经常看到使用 Java 8 操作集合的时候可以直接 `forEach` 的原因也是在 `Iterable` 接口中新增了一个默认方法：`forEach`，该方法功能和 for 循环类似，但是允许用户使用一个 Lambda 表达式作为循环体。

和其它方法一样，默认方法也可以被继承。不过，当类型或者接口的超类拥有多个具有相同签名的方法时，我们就需要一套规则来解决这个冲突：

- 类的方法声明优先于接口默认方法，无论该方法是具体的还是抽象的。
- 被其它类型所覆盖的方法会被忽略。这条规则适用于超类型共享一个公共祖先的情况。

为了演示第二条规则，我们假设 `Collection` 和 `List` 接口均提供了 `removeAll` 的默认实现，然后 `Queue` 继承并覆盖了 `Collection` 中的默认方法。在下面的 `implements` 子句中，`List` 中的方法声明会优先于 `Queue` 中的方法声明：

`class LinkedList<E> implements List<E>, Queue<E> { ... }`

当两个独立的默认方法相冲突或是默认方法和抽象方法相冲突时会产生编译错误。这时程序员需要显式覆盖超类方法。一般来说我们会定义一个默认方法，然后在其中通过 `接口名.super.方法名()` 显式选择超类方法：

```java
interface Robot extends Artist, Gun {
    default void draw() { Artist.super.draw(); }
}
```

最后，接口在 `implements` 和 `extends` 子句中的声明顺序和它们被实现的顺序无关。

## 4.2 接口内定义静态方法

除了默认方法，Java 8 还允许在接口中定义静态方法。这使得我们可以从接口直接调用和它相关的辅助方法，而不是从其它的类中调用（之前这样的类往往以对应接口的复数命名，例如 `Collections`）。比如，我们一般需要使用静态辅助方法生成实现 `Comparator` 的比较器，在 Java 8 中我们可以直接把该静态方法定义在 `Comparator` 接口中：

```java
public static <T, U extends Comparable<? super U>> Comparator<T> comparing(
        Function<? super T, ? extends U> keyExtractor) {
    Objects.requireNonNull(keyExtractor);
    return (Comparator<T> & Serializable)
        (c1, c2) -> keyExtractor.apply(c1).compareTo(keyExtractor.apply(c2));
}
```

其中 `(Comparator<T> & Serializable)` 是交叉类型转换，让返回的 Lambda 同时实现 `Serializable` 接口。

## 4.3 接口内定义私有方法

Java 9 进一步允许在接口中定义 `private` 方法（包括 `private static` 方法），用于在多个默认方法或静态方法之间复用代码，而不必把这些辅助逻辑暴露为公共 API：

```java
public interface Logger {
    default void info(String msg) { log("INFO", msg); }

    default void warn(String msg) { log("WARN", msg); }

    private void log(String level, String msg) {
        System.out.println("[" + level + "] " + msg);
    }
}
```

# 5 常用的函数式接口

Java 8 在 `java.util.function` 包中内置了一批常用的函数式接口，其中最核心的是以下四个：

| 接口             | 抽象方法            | 说明                       |
| ---------------- | ------------------- | -------------------------- |
| `Function<T, R>` | `R apply(T t)`      | 接收一个参数，返回一个结果 |
| `Consumer<T>`    | `void accept(T t)`  | 接收一个参数，没有返回值   |
| `Predicate<T>`   | `boolean test(T t)` | 接收一个参数，返回布尔值   |
| `Supplier<T>`    | `T get()`           | 不接收参数，返回一个结果   |

## 5.1 Function

该接口用来根据一个类型的数据得到另一个类型的数据，有进有出。

`Function` 接口包含四个方法：一个抽象方法 `apply()`，两个 `default` 方法 `andThen()` 和 `compose()`，以及一个静态方法 `identity()`。

实例化 `Function` 的时候需要实现其中的 `apply()` 方法，`apply` 方法接收一个泛型类型 `T` 的参数，返回一个泛型类型 `R` 的结果。

```java
public class FunctionDemo {
    public static void main(String[] args) {
        // 整数字符串
        String str = "123";
        // 将字符串转换为整型数据 123
        // 使用匿名内部类创建 Function 接口的实现类对象
        Function<String, Integer> f = new Function<String, Integer>() {
            @Override
            public Integer apply(String s) {
                return Integer.parseInt(s);
            }
        };
        // 调用 apply 方法进行转换
        int num01 = f.apply(str);
        System.out.println(num01);
        // 使用 Lambda 表达式简化
        Function<String, Integer> ff = s -> Integer.parseInt(s);
        int num02 = ff.apply(str);
        System.out.println(num02);
        // 使用方法引用简化 Lambda 表达式
        Function<String, Integer> fff = Integer::parseInt;
        int num03 = fff.apply(str);
        System.out.println(num03);
    }
}
```

`andThen` 方法接收一个 `Function` 接口的实例，通过 `andThen` 可以将任意多个 `Function` 的 `apply` 方法调用连接起来。

```java
Function<String, String> function1 = string -> {
    System.out.println("function1 输出了：" + string);
    return string;
};
Function<String, String> function2 = string -> {
    System.out.println("function2 输出了：" + string);
    return string;
};
function1.andThen(function2).apply("hello world");
```

输出结果：

```text
function1 输出了：hello world
function2 输出了：hello world
```

可以看到调用顺序是先调用 `function1`，然后调用 `function2`。

`compose` 方法和 `andThen` 方法一样接收另一个 `Function` 作为参数，但是顺序与 `andThen` 恰恰相反。

接下来的测试用例，保持前面的不变，只把最后一句由 `andThen` 改成：

`function1.compose(function2).apply("hello world");`

输出结果：

```text
function2 输出了：hello world
function1 输出了：hello world
```

可以看到这次先执行了 `function2`，之后再执行 `function1`。

`identity` 方法是一个静态方法，作用是返回一个 `Function` 对象，返回的对象总是返回它被传入的值。

## 5.2 Consumer

`Consumer` 接口包含两个方法：一个是 `accept` 方法，用来对输入的参数进行自定义操作，因为是个抽象方法，所以需要在实例化对象的时候重写；另一个是 `andThen` 方法，跟 `Function` 的同名方法一样是一个 `default` 方法，已经有内部实现所以不需要用户重写，具体功能也跟 `Function` 差不多。`Consumer` 的中文意思是消费者，意即通过传递进一个参数来对参数进行操作。

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
        for (int i : data) {
            consumer.accept(i);
        }
    }
}
```

在上面的代码中，借助 Java 8 引入的 Lambda 表达式，其中的

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

`Predicate` 接口最重要的是 `test` 方法，这是一个抽象方法，需要编程者自己去重写。另外还有三个 `default` 方法：`and`、`or` 和 `negate`，它们内部都使用到了 `test` 方法。其中 `and` 和 `or` 方法与前面两个接口的 `andThen` 方法类似，都接受另一个 `Predicate` 对象作为参数：`and` 方法返回两个对象分别调用 `test` 方法之后得到的布尔值的逻辑与，相当于 `predicate1.test(t) && predicate2.test(t)`；`or` 方法返回两个对象分别调用 `test` 方法之后得到的布尔值的逻辑或，相当于 `predicate1.test(t) || predicate2.test(t)`。两者都具有短路特性。`negate` 方法则返回逻辑非。

此外还有静态方法 `isEqual`，Java 11 又新增了静态方法 `Predicate.not`，可以方便地对方法引用取反，例如 `filter(Predicate.not(String::isBlank))`。

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

```text
and: false
or: true
negate: true
```

同样，可以简化成 Lambda 表达式：

```java
Predicate<Integer> predicate1 = integer -> integer <= 0;
Predicate<Integer> predicate2 = integer -> integer > 0;
```

## 5.4 Supplier

`Supplier` 的中文意思是提供者，跟 `Consumer` 接口相反，`Supplier` 接口用于提供对象，它只有一个 `get` 方法，是一个抽象方法，需要编程者自定义想要返回的对象。

```java
Supplier<Integer> supplier = new Supplier<Integer>() {
    @Override
    public Integer get() {
        return ThreadLocalRandom.current().nextInt(100);
    }
};

int[] ints = new int[10];
for (int i = 0; i < 10; i++) {
    ints[i] = supplier.get();
}
Arrays.stream(ints).forEach(System.out::println);
```

首先自定义了一个 `Supplier` 对象，对于其 `get` 方法，每次都返回一个 100 以内的随机数，并在之后利用这个对象给一个长度为 10 的 `int` 数组赋值并输出。

## 5.5 其他函数式接口

`java.util.function` 包中一共有 43 个函数式接口，绝大部分都是由上面这四种衍生而来的，主要有以下几类：

- **双参数版本**：`BiFunction<T, U, R>`、`BiConsumer<T, U>`、`BiPredicate<T, U>`。
- **参数与返回值类型相同的版本**：`UnaryOperator<T>`（继承自 `Function<T, T>`）、`BinaryOperator<T>`（继承自 `BiFunction<T, T, T>`）。
- **基本类型特化版本**：如 `IntFunction`、`ToIntFunction`、`IntPredicate`、`IntSupplier`、`LongConsumer` 等，用于避免装箱拆箱带来的性能开销。

这些接口在 JDK 的很多地方都被用到，尤其是 Stream API，更多情况下是作为方法参数来使用，配合简洁的 Lambda 表达式使得程序可读性变得更好。

# 6 实现原理

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

        print(name, (t) -> System.out.println(prefix + t));
        // 与上一行不同的是，这个 Lambda 表达式的函数体中没有引用外部变量 prefix
        print(name, (t) -> System.out.println(t));
    }

}

@FunctionalInterface
interface Print {
    void print(String name);
}
```

例子很简单，定义了一个函数式接口 `Print`，`main` 方法中有两处代码以 Lambda 表达式的方式实现了 `Print` 接口，分别打印出带前缀与不带前缀的名字。

运行程序，打印结果如下：

```text
hello, Chen Longfei
Chen Longfei
```

而 `(t) -> System.out.println(prefix + t)` 与 `(t) -> System.out.println(t)` 之类的 Lambda 表达式到底是怎样被编译与调用的呢？

我们知道，编译器编译 Java 代码时经常在背地里“搞鬼”，比如类的全限定名的补全、泛型的类型推断等，编译器耍的这些小聪明可以帮助我们写出更优雅、简洁、高效的代码。鉴于编译器的一贯作风，我们有理由怀疑，新颖而另类的 Lambda 表达式在编译时很可能会被改造过了。

下面通过 `javap` 反编译 class 文件一探究竟（以下反编译结果基于 JDK 8，`major version: 52`）。

`javap` 是 JDK 自带的一个字节码查看工具及反编译工具：

用法：`javap <options> <classes>`

其中，可能的选项包括：

```text
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

```text
interface test.Print {
  public abstract void print(java.lang.String);
}
```

`javap -p LambdaTest.class`

结果如下：

```text
Compiled from "LambdaTest.java"
public class test.LambdaTest {
  public test.LambdaTest();
  public static void print(java.lang.String, test.Print);
  public static void main(java.lang.String[]);
  private static void lambda$main$1(java.lang.String);
  private static void lambda$main$0(java.lang.String, java.lang.String);
}
```

可见，`Print` 接口编译后没有变化（接口中的抽象方法本身就隐式地带有 `public abstract` 修饰符，`javap` 只是把它们显示了出来），而 `LambdaTest` 的变化就比较大了，编译器为它添加了两个私有静态方法：

```text
private static void lambda$main$1(java.lang.String);
private static void lambda$main$0(java.lang.String, java.lang.String);
```

对比原生的 Java 代码，很容易做出推测，这两个静态方法与两处 Lambda 表达式相关，Lambda 的方法体被编译器“脱糖”（desugar）成了这两个方法，被捕获的外部变量 `prefix` 则变成了 `lambda$main$0` 的第一个参数：

```java
print(name, (t) -> System.out.println(prefix + t)); // 对应 lambda$main$0
print(name, (t) -> System.out.println(t));          // 对应 lambda$main$1
```

如果 Lambda 表达式中使用了 `this` 或实例成员，生成的 `lambda$xxx$N` 方法则是实例方法而不是静态方法。

到底有什么关联呢？使用 `javap -p -v -c LambdaTest.class` 查看更加详细的反编译结果：

```text
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
  #12 = Methodref          #10.#48        // java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
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
  #24 = Utf8               lambda$main$1
  #25 = Utf8               (Ljava/lang/String;)V
  #26 = Utf8               lambda$main$0
  #27 = Utf8               (Ljava/lang/String;Ljava/lang/String;)V
  #28 = Utf8               SourceFile
  #29 = Utf8               LambdaTest.java
  #30 = NameAndType        #16:#17        // "<init>":()V
  #31 = Class              #52            // test/Print
  #32 = NameAndType        #20:#25        // print:(Ljava/lang/String;)V
  #33 = Utf8               Chen Longfei
  #34 = Utf8               hello,
  #35 = Utf8               BootstrapMethods
  #36 = MethodHandle       #6:#53         // invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #37 = MethodType         #25            //  (Ljava/lang/String;)V
  #38 = MethodHandle       #6:#54         // invokestatic test/LambdaTest.lambda$main$0:(Ljava/lang/String;Ljava/lang/String;)V
  #39 = NameAndType        #20:#55        // print:(Ljava/lang/String;)Ltest/Print;
  #40 = NameAndType        #20:#21        // print:(Ljava/lang/String;Ltest/Print;)V
  #41 = MethodHandle       #6:#56         // invokestatic test/LambdaTest.lambda$main$1:(Ljava/lang/String;)V
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
  #53 = Methodref          #67.#68        // java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #54 = Methodref          #14.#69        // test/LambdaTest.lambda$main$0:(Ljava/lang/String;Ljava/lang/String;)V
  #55 = Utf8               (Ljava/lang/String;)Ltest/Print;
  #56 = Methodref          #14.#70        // test/LambdaTest.lambda$main$1:(Ljava/lang/String;)V
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
  #68 = NameAndType        #72:#76        // metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #69 = NameAndType        #26:#27        // lambda$main$0:(Ljava/lang/String;Ljava/lang/String;)V
  #70 = NameAndType        #24:#25        // lambda$main$1:(Ljava/lang/String;)V
  #71 = Utf8               java/lang/invoke/LambdaMetafactory
  #72 = Utf8               metafactory
  #73 = Class              #78            // java/lang/invoke/MethodHandles$Lookup
  #74 = Utf8               Lookup
  #75 = Utf8               InnerClasses
  #76 = Utf8               (Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
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

  private static void lambda$main$1(java.lang.String);
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

  private static void lambda$main$0(java.lang.String, java.lang.String);
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
      #38 invokestatic test/LambdaTest.lambda$main$0:(Ljava/lang/String;Ljava/lang/String;)V
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
      #41 invokestatic test/LambdaTest.lambda$main$1:(Ljava/lang/String;)V
      #37 (Ljava/lang/String;)V
```

这个 class 文件展示了三个主要部分：

- 常量池
- 构造方法和 `main`、`print`、`lambda$main$0`、`lambda$main$1` 方法
- `InnerClasses` 和 `BootstrapMethods` 属性。其中 `BootstrapMethods` 是引导方法表，记录了每条 `invokedynamic` 指令对应的引导方法及其参数；`InnerClasses` 记录的是 `MethodHandles$Lookup` 这个内部类，并不是 Lambda 生成的类——编译期根本不会为 Lambda 生成类文件。

重点看下 `main` 方法的实现：

```text
public static void main(java.lang.String[]);
descriptor: ([Ljava/lang/String;)V
flags: ACC_PUBLIC, ACC_STATIC
Code:
  stack=2, locals=3, args_size=1
     // 将字符串常量 "Chen Longfei" 从常量池压入操作数栈
     0: ldc           #3                  // String Chen Longfei

     // 将栈顶引用型数值存入第二个本地变量，即 String name = "Chen Longfei"
     2: astore_1

     // 将字符串常量 "hello, " 从常量池压入操作数栈
     3: ldc           #4                  // String hello,

     // 将栈顶引用型数值存入第三个本地变量，即 String prefix = "hello, "
     5: astore_2

     // 将第二个引用类型本地变量推送至栈顶，即 name
     6: aload_1

     // 将第三个引用类型本地变量推送至栈顶，即 prefix
     7: aload_2

     // 通过 invokedynamic 指令创建 Print 接口的实例，实现 (t) -> System.out.println(prefix + t)，prefix 作为被捕获的参数传入
     8: invokedynamic #5,  0              // InvokeDynamic #0:print:(Ljava/lang/String;)Ltest/Print;

     // 调用静态方法 print
    13: invokestatic  #6                  // Method print:(Ljava/lang/String;Ltest/Print;)V

     // 将第二个引用类型本地变量推送至栈顶，即 name
    16: aload_1

     // 通过 invokedynamic 指令创建 Print 接口的实例，实现 (t) -> System.out.println(t)
    17: invokedynamic #7,  0              // InvokeDynamic #1:print:()Ltest/Print;

     // 调用静态方法 print
    22: invokestatic  #6                  // Method print:(Ljava/lang/String;Ltest/Print;)V
    25: return
    ……
```

注意 `lambda$main$0` 中的字符串拼接使用的是 `StringBuilder`，这是 JDK 8 的编译结果；从 JDK 9 开始（JEP 280），javac 会把字符串拼接也编译成 `invokedynamic` 指令，引导方法为 `StringConcatFactory.makeConcatWithConstants`。

两个 Lambda 对应的实现类，是在运行时由 `BootstrapMethods` 中的引导方法创建的：

```text
InnerClasses:
     // 记录的是 MethodHandles$Lookup 这个内部类，与 Lambda 无关
     public static final #74= #73 of #77; //Lookup=class java/lang/invoke/MethodHandles$Lookup of class java/lang/invoke/MethodHandles
BootstrapMethods:
     // 调用静态工厂 LambdaMetafactory.metafactory 创建实现类 1，实现了 (t) -> System.out.println(prefix + t)
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
         // 该类会调用静态方法 LambdaTest.lambda$main$0
         #38 invokestatic test/LambdaTest.lambda$main$0:(Ljava/lang/String;Ljava/lang/String;)V
         #37 (Ljava/lang/String;)V

     // 调用静态工厂 LambdaMetafactory.metafactory 创建实现类 2，实现了 (t) -> System.out.println(t)
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
         // 该类会调用静态方法 LambdaTest.lambda$main$1
         #41 invokestatic test/LambdaTest.lambda$main$1:(Ljava/lang/String;)V
         #37 (Ljava/lang/String;)V
```

在 JDK 8 中，可以在运行时加上 `-Djdk.internal.lambda.dumpProxyClasses=<目录>` 参数，运行时会将生成的实现类 class 文件输出到指定目录下。从 JDK 21 开始，该参数被替换为 `-Djdk.invoke.LambdaMetafactory.dumpProxyClassFiles=true`，class 文件会输出到当前目录的 `DUMP_LAMBDA_PROXY_CLASS_FILES` 文件夹中。

`javap -p -c` 反编译两个文件（类名中的序号由运行时生成的顺序决定，带前缀的 Lambda 先执行，因此是 `$$Lambda$1`）：

```text
// print(name, (t) -> System.out.println(prefix + t)) 的实例
final class test.LambdaTest$$Lambda$1 implements test.Print {

  private final java.lang.String arg$1;

  private test.LambdaTest$$Lambda$1(java.lang.String);
    Code:
       0: aload_0
       1: invokespecial #13                 // Method java/lang/Object."<init>":()V
       4: aload_0
       5: aload_1
       // final 变量 arg$1 由构造方法传入
       6: putfield      #15                 // Field arg$1:Ljava/lang/String;
       9: return

  // 该方法返回一个 LambdaTest$$Lambda$1 实例
  private static test.Print get$Lambda(java.lang.String);
    Code:
       0: new           #2                  // class test/LambdaTest$$Lambda$1
       3: dup
       4: aload_0
       5: invokespecial #19                 // Method "<init>":(Ljava/lang/String;)V
       8: areturn

  // 实现 test.Print 接口方法
  public void print(java.lang.String);
    Code:
       0: aload_0
       1: getfield      #15                 // Field arg$1:Ljava/lang/String;
       4: aload_1
       // 调用静态方法 LambdaTest.lambda$main$0
       5: invokestatic  #27                 // Method test/LambdaTest.lambda$main$0:(Ljava/lang/String;Ljava/lang/String;)V
       8: return
}

// print(name, (t) -> System.out.println(t)) 的实例
final class test.LambdaTest$$Lambda$2 implements test.Print {
  private test.LambdaTest$$Lambda$2(); // 构造方法
    Code:
       0: aload_0
       1: invokespecial #10                 // Method java/lang/Object."<init>":()V
       4: return

  // 实现 test.Print 接口方法
  public void print(java.lang.String);
    Code:
       0: aload_1
       // 调用静态方法 LambdaTest.lambda$main$1
       1: invokestatic  #18                 // Method test/LambdaTest.lambda$main$1:(Ljava/lang/String;)V
       4: return
}
```

对比两个实例，可以发现，由于表达式 `print(name, (t) -> System.out.println(prefix + t))` 引用了局部变量 `prefix`，`LambdaTest$$Lambda$1` 类多了一个 `final` 字段：

`private final java.lang.String arg$1`

该字段由构造方法传入，用来存储 `main` 方法中的局部变量 `prefix`：

`String prefix = "hello, ";`

由于外部类的 `main` 方法与实现类 `LambdaTest$$Lambda$1` 引用了同一份变量，该变量虽然在代码层面独立存储于两个地方，但是在逻辑上需要保持一致，所以实现类中的字段加上了 `final` 关键字，而外部类中虽然没有为 `prefix` 显式地添加 `final`，但是在被 Lambda 表达式引用后，该变量就自动隐含了 `final` 语义（再次更改会报错），这与第 3 章讲的内容是一致的。

另外，没有捕获变量的 Lambda（如 `$$Lambda$2`）不需要每次调用都创建新对象，在 HotSpot 的实现中，`invokedynamic` 调用点链接完成后会一直返回同一个实例；而捕获变量的 Lambda 每次执行都会创建一个新实例。不过 Java 语言规范并不保证这一点，因此不要依赖 Lambda 对象的同一性。

## 6.2 InvokeDynamic

通过上面的例子可以发现，Lambda 表达式由虚拟机指令 `invokedynamic` 实现方法调用。

### 6.2.1 方法调用

方法调用不等同于方法执行，方法调用阶段的唯一任务就是确定被调用方法的版本（即确定具体调用哪一个方法），不涉及方法内部具体运行。

Java 虚拟机中提供了 5 条方法调用的字节码指令：

- **invokestatic**：调用静态方法。
- **invokespecial**：调用实例构造器 `<init>` 方法、私有方法、父类方法。
- **invokevirtual**：调用虚方法。
- **invokeinterface**：调用接口方法，在运行时再确定一个实现该接口的对象。
- **invokedynamic**：运行时动态解析出调用的方法，然后去执行该方法。

`invokedynamic` 是 Java 7 通过 JSR 292 引入的一条新的虚拟机指令，这是自 Java 1.0 以来第一次引入新的方法调用指令。它最初是为了让 JRuby、Groovy 等 JVM 上的动态语言更高效地实现方法分派，直到 Java 8，javac 才第一次在编译 Java 代码时生成这条指令，用在 Lambda 表达式和方法引用中。`invokedynamic` 与其他 invoke 指令不同的是，它允许由应用级的代码来决定方法解析。

### 6.2.2 指令规范

根据 JVM 规范的规定，`invokedynamic` 的操作码是 `186（0xBA）`，格式是：

`invokedynamic indexbyte1 indexbyte2 0 0`

`invokedynamic` 指令有四个操作数，前两个操作数构成一个索引 `(indexbyte1 << 8) | indexbyte2`，指向类的常量池，后两个操作数保留，必须是 0。

查看上例中 `LambdaTest` 类的反编译结果，第二处 Lambda 表达式

`print(name, (t) -> System.out.println(t));`

对应的指令为：

`17: invokedynamic #7,  0              // InvokeDynamic #1:print:()Ltest/Print;`

常量池中 `#7` 对应的常量为：

`#7 = InvokeDynamic      #1:#42         // #1:print:()Ltest/Print;`

其类型为 `CONSTANT_InvokeDynamic_info`，这是 Java 7 新引入 class 文件的常量类型，它包含两项内容：

- `#1`：指向 `BootstrapMethods` 表的索引，表示使用哪个引导方法（bootstrap method）；
- `#42`：指向一个 `NameAndType` 常量，即方法名 `print` 和方法描述符 `()Ltest/Print;`。描述符的参数列表是被捕获的变量（这里没有），返回值是要生成的函数式接口类型。

`#1` 代表 `BootstrapMethods` 表中的索引，即：

```text
// 第一个
0: #36 ……

// 第二个
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
      #41 invokestatic test/LambdaTest.lambda$main$1:(Ljava/lang/String;)V
      #37 (Ljava/lang/String;)V
```

也就是说，最终调用的是 `java.lang.invoke.LambdaMetafactory` 类的静态方法 `metafactory()`。

### 6.2.3 执行过程

为了更深入地了解 `invokedynamic`，先来看几个术语：

- **dynamic call site**：程序中每一条 `invokedynamic` 指令都被称作一个动态调用点（dynamic call site）。
- **bootstrap method**：引导方法。Java 中普通的 Lambda 统一使用 `LambdaMetafactory.metafactory` 作为引导方法（需要序列化的 Lambda 使用 `LambdaMetafactory.altMetafactory`），它在运行期动态生成函数式接口的实现类，并返回一个 `CallSite` 对象。
- **call site object**：调用点对象，即 `java.lang.invoke.CallSite` 的实例，它持有一个 `MethodHandle` 作为它的 target，是引导方法成功调用后的结果，将会与 dynamic call site 永久绑定。之后每次执行到这条 `invokedynamic` 指令时，JVM 都会直接调用 `CallSite` 的 target，所需的参数（被捕获的变量）会从操作数栈中取出，最终得到一个实现了函数式接口的对象。

`invokedynamic` 首先需要生成一个 `CallSite`（调用点对象），`CallSite` 由 bootstrap method 返回，也就是调用 `LambdaMetafactory.metafactory` 方法。JDK 8 中该方法的源码如下：

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

前三个参数是固定的，由 JVM 自动传入：

- `MethodHandles.Lookup caller`：代表 `invokedynamic` 指令所在的类的查找上下文（在上例中就是 `LambdaTest`），可以通过 `Lookup#lookupClass()` 获取这个类。
- `String invokedName`：要实现的方法名（在上例中就是 `Print` 接口的方法名“print”）。
- `MethodType invokedType`：调用点的方法类型，参数是被捕获的变量，返回值是函数式接口类型，如 `(Ljava/lang/String;)Ltest/Print;`。

接下来就是附加参数，这些参数是灵活的，由 `BootstrapMethods` 表提供：

- `MethodType samMethodType`：函数式接口中抽象方法的类型（擦除后）。
- `MethodHandle implMethod`：编译器生成的脱糖方法（如 `lambda$main$0`）的方法句柄，也可以是方法引用所指向的方法。
- `MethodType instantiatedMethodType`：调用时抽象方法的具体类型。因为方法定义可能是泛型的，传入时可能是 `String` 之类的具体类型，需要据此做类型校验和强制转换。

新版本 JDK 中这些参数被重命名为 `interfaceMethodName`、`factoryType`、`interfaceMethodType`、`implementation`、`dynamicMethodType`，含义不变。

`LambdaMetafactory.metafactory` 方法会在内存中生成实现类的字节码，然后直接定义到 JVM 中，而不会写到文件里。具体实现随 JDK 版本有所变化：

- **JDK 8 ~ 14**：使用 JDK 内置的 ASM 生成字节码，通过 `Unsafe.defineAnonymousClass` 定义为 VM Anonymous Class。这种类不会注册到任何类加载器中，不能通过类名查找，可以在不再被引用时独立卸载。
- **JDK 15 及之后**：改用 JEP 371 引入的隐藏类（Hidden Class），通过标准 API `MethodHandles.Lookup.defineHiddenClass` 定义，具备与 VM Anonymous Class 相同的不可发现、可独立卸载等特性；`Unsafe.defineAnonymousClass` 则在 JDK 17 中被移除。隐藏类的类名形如 `LambdaTest$$Lambda/0x0000000800c03000`。
- 字节码生成方面，JDK 内部已逐步从 ASM 迁移到 JDK 24 正式发布的 Class-File API（JEP 484）。

### 6.2.4 MethodHandle

要让 `invokedynamic` 正常运行，一个核心的概念就是**方法句柄**（Method Handle）。它代表了一个可以从 `invokedynamic` 调用点进行调用的方法。每个 `invokedynamic` 指令都会与一个特定的方法关联（也就是 bootstrap method，简称 BSM）。当 JVM 第一次执行到某条 `invokedynamic` 指令的时候，BSM 会被调用，返回一个包含了方法句柄的 `CallSite` 对象，这个对象表明了调用点要实际执行哪个方法。

Java 7 API 中加入了 `java.lang.invoke.MethodHandle`（及其子类），通过它们来代表 `invokedynamic` 指向的方法。一个 Java 方法可以视为由四个基本内容所构成：

- 名称
- 签名（包含返回类型）
- 定义它的类
- 实现方法的字节码

这意味着如果要引用某个方法，我们需要有一种有效的方式来表示方法签名（而不是反射中强制使用的令人讨厌的 `Class<?>[]` 方式）。

方法句柄首先需要一种表达方法签名的方式，以便于查找。在 Java 7 引入的 Method Handles API 中，这个角色是由 `java.lang.invoke.MethodType` 类来完成的，它使用一个不可变的实例来代表签名。要获取 `MethodType`，我们可以使用 `methodType()` 工厂方法。这是一个参数可变的方法，以 `Class` 对象作为参数。

第一个参数所使用的 `Class` 对象，对应着签名的返回类型；剩余参数中所使用的 `Class` 对象，对应着签名中方法参数的类型。例如：

```java
// toString() 的签名
MethodType mtToString = MethodType.methodType(String.class);

// setter 方法的签名
MethodType mtSetter = MethodType.methodType(void.class, Object.class);

// Comparator 中 compare() 方法的签名
MethodType mtStringComparator = MethodType.methodType(int.class, String.class, String.class);
```

现在我们就可以使用 `MethodType`，再组合方法名称以及定义方法的类来查找方法句柄。要实现这一点，我们需要调用静态的 `MethodHandles.lookup()` 方法。这样的话，会给我们一个“查找上下文（lookup context）”，这个上下文基于当前正在执行的方法（也就是调用 `lookup()` 的方法）的访问权限。

查找上下文对象有一些以“find”开头的方法，例如 `findVirtual()`、`findConstructor()`、`findStatic()` 等。这些方法将会返回实际的方法句柄，需要注意的是，只有在创建查找上下文的类能够访问（调用）被请求方法的情况下，才会返回句柄。这与反射不同，方法句柄中并没有与 `setAccessible()` 对应的方法，无法随意绕过访问控制。如果确实需要访问其他类的私有成员，Java 9 提供了 `MethodHandles.privateLookupIn()`，但它同样受模块系统的访问规则约束。例如：

```java
public MethodHandle getToStringMH() {
    MethodHandle mh = null;
    MethodType mt = MethodType.methodType(String.class);
    MethodHandles.Lookup lk = MethodHandles.lookup();

    try {
        mh = lk.findVirtual(getClass(), "toString", mt);
    } catch (NoSuchMethodException | IllegalAccessException mhx) {
        throw new AssertionError(mhx);
    }

    return mh;
}
```

`MethodHandle` 中有两个方法能够触发对方法句柄的调用，那就是 `invoke()` 和 `invokeExact()`。这两个方法都以接收者（receiver）和方法参数作为参数，它们在源码中的声明为：

```java
public final native @PolymorphicSignature Object invoke(Object... args) throws Throwable;
public final native @PolymorphicSignature Object invokeExact(Object... args) throws Throwable;
```

这两个方法是**签名多态**（signature polymorphic）的：虽然声明为 `Object...`，但编译器不会把参数装进数组，而是按照调用处实际的参数类型和强制转换的返回类型生成方法描述符。

两者的区别在于，`invokeExact()` 要求调用处的方法描述符与方法句柄的类型严格一致，否则抛出 `WrongMethodTypeException`。而 `invoke()` 与之不同，在需要的时候，`invoke()` 能够调整参数和返回值的类型。`invoke()` 会执行一个 `asType()` 转换，它会根据如下的这组规则来进行转换：

- 如果需要的话，原始类型会进行装箱操作
- 如果需要的话，装箱后的原始类型会进行拆箱操作
- 如果必要的话，原始类型会进行扩展
- `void` 返回类型会转换为 0（对于返回原始类型的情况），而对于预期得到引用类型的返回值的地方，将会转换为 `null`
- `null` 值会被视为正确的，不管静态类型是什么都可以进行传递

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

上面的代码调用了 `String` 的 `hashCode()` 方法（`rcvr.getClass()` 是 `String.class`），看到这里，你肯定会说这不就是 Java 的反射吗？

确实，`MethodHandle` 和反射（Reflection）实现的功能有太多相似的地方，都是运行时解析方法调用，理解方法句柄的一种方式就是将其视为以安全、现代的方式来实现反射的核心功能，在这个过程会尽可能地保证类型的安全。

但是，究其本质，两者之间还是有区别的：

反射中的 `java.lang.reflect.Method` 对象远比 `java.lang.invoke.MethodHandle` 对象所包含的信息来得多。前者是方法在 Java 一端的全面映像，包含了方法的签名、描述符以及方法属性表中各种属性的 Java 端表示方式，还包含有执行权限等的运行期信息。而后者仅仅包含着与执行该方法相关的信息。用开发人员通俗的话来讲，**反射是重量级，而 MethodHandle 是轻量级**。

- 从性能角度上说，访问检查在创建 `MethodHandle` 的时候就已经完成了，而不是像反射一样每次调用时都检查；当 `MethodHandle` 存放在 `static final` 字段中时，JIT 还可以把它当作常量进行内联优化，性能接近直接调用。值得一提的是，从 JDK 18 开始（JEP 416），反射的 `Method.invoke` 等核心实现本身也改为基于方法句柄实现。
- 反射是在模拟 Java 代码层次的方法调用，而 `MethodHandle` 是在模拟字节码层次的方法调用。
- `MethodHandle` 是结合 `invokedynamic` 指令一起为动态语言服务的，也就是说 `MethodHandle`（更准确地说是其设计理念）服务于所有运行在 JVM 之上的语言，而反射则主要适用于 Java 语言本身。

# 7 新版本中的 Lambda

Lambda 自 Java 8 引入以来，语法本身变化不大，但周边特性和底层实现一直在演进：

| 版本    | 变化                                                         |
| ------- | ------------------------------------------------------------ |
| Java 9  | 接口支持 `private` 方法（见 4.3 节）；字符串拼接改用 `invokedynamic` 实现（JEP 280） |
| Java 10 | 引入 `var` 局部变量类型推断，但 Lambda 必须有目标类型，`var f = () -> {};` 无法编译 |
| Java 11 | Lambda 参数可以使用 `var`（JEP 323），便于给参数加注解，如 `(@NonNull var x, var y) -> x + y`；新增 `Predicate.not` |
| Java 11 | 引入动态常量 `CONSTANT_Dynamic`（JEP 309），把“运行时引导”的思路从方法调用扩展到常量 |
| Java 15 | Lambda 实现类改用隐藏类（JEP 371）                           |
| Java 16 | Record 的 `toString`、`equals`、`hashCode` 通过 `invokedynamic` 和 `ObjectMethods` 引导方法生成 |
| Java 21 | `switch` 模式匹配（JEP 441）通过 `invokedynamic` 和 `SwitchBootstraps` 实现类型分派 |
| Java 22 | 未命名变量 `_`（JEP 456），不使用的 Lambda 参数可以写成 `_`，如 `map.forEach((_, v) -> System.out.println(v))` |

最后再总结一下 Lambda 与匿名内部类的几点区别：

- **`this` 的含义不同**：匿名内部类中的 `this` 指向匿名类实例本身；Lambda 中的 `this` 指向外部类实例，Lambda 不会引入新的作用域。
- **变量遮蔽**：匿名内部类可以声明与外部局部变量同名的变量；Lambda 的参数和局部变量不能与外部作用域的局部变量重名。
- **编译产物不同**：匿名内部类在编译期生成独立的 `Outer$1.class` 文件；Lambda 只生成一个私有的脱糖方法，实现类在运行时才通过 `invokedynamic` 生成。
- **适用范围不同**：Lambda 只能实现函数式接口；匿名内部类可以继承抽象类、实现有多个抽象方法的接口，也可以拥有自己的状态。

# 8 总结

- Lambda 表达式是 Java 8 引入的函数式编程特性，本质上是函数式接口的一个实例，必须依赖目标类型才能确定其类型。
- 方法引用是 Lambda 的简写形式，分为静态方法引用、特定对象的实例方法引用、特定类型任意对象的实例方法引用和构造方法引用四类。
- Lambda 只能引用 `final` 或事实上 `final` 的局部变量，因为它捕获的是变量值的拷贝。
- Java 8 允许接口定义默认方法和静态方法，Java 9 又允许定义私有方法，使得接口可以在不破坏兼容性的前提下演进。
- `java.util.function` 包提供了以 `Function`、`Consumer`、`Predicate`、`Supplier` 为核心的 43 个函数式接口。
- 在实现上，javac 把 Lambda 方法体编译为私有的 `lambda$xxx$N` 方法，并在调用处生成 `invokedynamic` 指令；运行时由 `LambdaMetafactory` 生成实现类（JDK 15 起为隐藏类）并绑定到调用点。`invokedynamic` 与 `MethodHandle` 这套机制如今也被字符串拼接、Record、模式匹配等新特性广泛使用。
