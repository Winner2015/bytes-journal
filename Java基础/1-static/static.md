# static

# 1 static 变量

按照是否被 `static` 修饰，类成员变量可分为两种：

- 被 `static` 修饰的变量，叫**静态变量**或**类变量**，`static` 成员变量按照定义顺序进行初始化；
- 没有被 `static` 修饰的变量，叫**实例变量**。

每个定义该类的 `Class` 对象对应一份静态字段，可通过类名直接访问；也可以通过对象访问，但不推荐。对于实例变量，每创建一个实例，就会为实例变量分配一次内存，不同实例的变量互不影响。

更准确地说，静态变量在类加载的**准备阶段**分配内存并赋零值（如 `0`、`null`），到**初始化阶段**才执行赋值语句和静态代码块，赋予程序中指定的初始值。在 HotSpot 虚拟机中，从 JDK 7 开始静态变量从永久代移出，跟随 `Class` 对象存放在堆中；JDK 8 又用元空间彻底取代了永久代。

所以一般在需要实现以下两个功能时使用静态变量：

- 在对象之间共享值时
- 方便访问变量时

# 2 static 方法

静态方法可以直接通过类名调用，任何的实例也都可以调用，因此**静态方法中不能用 `this` 和 `super` 关键字**，不能直接访问所属类的实例变量和实例方法（就是不带 `static` 的成员变量和成员方法），**只能访问所属类的静态成员变量和静态成员方法**，因为实例成员与特定的对象关联。

但是要注意的是，虽然在静态方法中不能访问非静态成员方法和非静态成员变量，但是在非静态成员方法中是可以访问静态成员方法/变量的。

`static` 方法独立于任何实例，因此 `static` 方法必须被实现，而不能是抽象的 `abstract`。

例如为了方便方法的调用，Java API 中的 `Math` 类中所有的方法都是静态的，而一般类内部的 `static` 方法也是方便其他类对该方法的调用。

静态方法是类内部的一类特殊方法，只有在需要时才将对应的方法声明成静态的，一个类内部的方法一般都是非静态的。

因此，如果说想在不创建对象的情况下调用某个方法，就可以将这个方法设置为 `static`。我们最常见的 `static` 方法就是 `main` 方法：传统写法中 `main` 方法必须是 `public static void main(String[] args)`，因为程序在执行 `main` 方法的时候还没有创建任何对象，只能通过类名来调用。不过从 JDK 25 开始（JEP 512），Java 正式支持实例 `main` 方法，可以直接写 `void main()`，此时由启动器先创建该类的实例，再调用这个方法。

另外要注意，类的构造器**并不是**静态方法。构造器在字节码中对应实例初始化方法 `<init>`，方法体内可以使用 `this`；之所以不需要先有对象就能“调用”构造器，是因为 `new` 指令会先分配对象，再在这个对象上执行构造器。

# 3 static 代码块

`static` 代码块也叫**静态代码块**，是在类中独立于类成员的 `static` 语句块，可以有多个，位置可以随便放，它不在任何的方法体内。JVM 在类的**初始化阶段**（执行 `<clinit>` 方法时）会执行这些静态代码块，如果 `static` 代码块有多个，**JVM 将按照它们在类中出现的先后顺序依次执行它们，每个代码块只会被执行一次**。注意，仅仅把类加载进来（如调用 `ClassLoader.loadClass`）并不会触发初始化，而 `Class.forName`、`new`、访问非常量静态变量、调用静态方法等操作会触发；读取编译期常量（`static final` 基本类型或字面量 `String`）通常不会触发声明类的初始化。

静态代码块适合执行**一次性的类级初始化**操作，避免在每次方法调用中重复创建只需初始化一次的对象。

下面看个例子：

```java
class Person {
	private LocalDate birthDate;

	public Person(LocalDate birthDate) {
		this.birthDate = birthDate;
	}

	boolean isBornBoomer() {
		LocalDate startDate = LocalDate.of(1946, 1, 1);
		LocalDate endDate = LocalDate.of(1965, 1, 1);
		return !birthDate.isBefore(startDate) && birthDate.isBefore(endDate);
	}

}
```

`isBornBoomer` 是用来判断这个人是否是 1946 年至 1964 年出生的，而每次 `isBornBoomer` 被调用的时候，都会生成 `startDate` 和 `endDate` 两个对象，造成了空间浪费，如果改成这样效率会更好：

```java
class Person {
	private static final LocalDate START_DATE;
	private static final LocalDate END_DATE;

	static {
		START_DATE = LocalDate.of(1946, 1, 1);
		END_DATE = LocalDate.of(1965, 1, 1);
	}

	private LocalDate birthDate;

	public Person(LocalDate birthDate) {
		this.birthDate = birthDate;
	}

	boolean isBornBoomer() {
		return !birthDate.isBefore(START_DATE) && birthDate.isBefore(END_DATE);
	}

}
```

因此，很多时候会将一些只需要进行一次的初始化操作都放在 `static` 代码块中进行。

# 4 static final

`static final` 修饰成员变量时，可简单理解为”**类常量**”；它也可以修饰成员方法。

- 对于变量，表示一旦给值就不可修改，并且通过类名可以访问。
- 对于方法，表示子类不能再定义同签名的静态方法将其隐藏，并且可以通过类名直接访问。

对于 `static final` 引用，引用本身不能再指向其他对象，但对于一些容器类型（比如 `ArrayList`、`HashMap`），**不可以改变容器变量本身，但可以修改容器中存放的对象**，这一点在编程中用到很多。如果希望容器的内容也不可变，可以使用 Java 9 引入的 `List.of(...)`、`Map.of(...)` 等不可变集合。

```java
	private static final String strStaticFinalVar = "aaa";
	private static String strStaticVar = null;
	private final String strFinalVar = null;
	private static final int intStaticFinalVar = 0;
	private static final Integer integerStaticFinalVar = Integer.valueOf(8);
	private static final ArrayList<String> alStaticFinalVar = new ArrayList<>();

	private void test() {

		// strStaticFinalVar = "哈哈哈哈"; // 错误，final 表示终态，不可以改变变量本身。
		strStaticVar = "哈哈哈哈"; // 正确，static 表示类变量，值可以改变。

		// strFinalVar = "呵呵呵呵"; // 错误，final 变量只能赋值一次，一旦给定后就不可再更改。
		// intStaticFinalVar = 2; // 错误，final 变量只能赋值一次，一旦给定后就不可再更改。
		// integerStaticFinalVar = Integer.valueOf(8); // 错误，final 变量只能赋值一次，一旦给定后就不可再更改。
		alStaticFinalVar.add("aaa"); // 正确，容器变量本身没有变化，但存放内容发生了变化。这个规则是非常常用的，有很多用途。
		alStaticFinalVar.add("bbb"); // 正确，容器变量本身没有变化，但存放内容发生了变化。这个规则是非常常用的，有很多用途。
	}
```

`final` 变量必须且只能赋值一次，但不一定要在定义时赋值：`static final` 变量可以在静态代码块中赋值，实例 `final` 变量可以在构造方法中赋值。另外，`new Integer(8)` 这类包装类构造方法从 Java 9 起已被标记为废弃，Java 16 起更被标记为待移除，应改用 `Integer.valueOf(8)` 或直接自动装箱。

# 5 执行顺序

静态代码块、静态方法、构造方法等在类加载、实例化的时候的执行顺序是怎样的呢？

下面通过一个例子来验证。有这样两个类：

```java
class Father {

	static int a = before();

	static {
		System.out.println("Father static");
	}

	static int b = after();

	public Father() {
		System.out.println("Father constructor");
	}

	static int before() {
		System.out.println("Father static before");
		return 1;
	}

	static int after() {
		System.out.println("Father static after");
		return 2;
	}
}

class Son extends Father {

	int a = fun();
	int b = fun2();
	static {
		System.out.println("Son static");
	}

	public Son() {
		System.out.println("Son constructor");
	}

	static int fun() {
		System.out.println("Son static function");
		return 1;
	}

	int fun2() {
		System.out.println("Son non-static function");
		return 1;
	}
}
```

用下面的代码测试：

`Class s = Class.forName("Son");`

打印结果如下：

```text
Father static before
Father static
Father static after
Son static
```

`Class.forName` 会把类加载到 JVM 并完成初始化，可见初始化子类之前，需要先初始化父类，并按照出现的顺序执行其中的静态变量赋值和静态代码块（静态变量赋值中如果调用了静态方法，也会在此时执行）。

改用下面的代码测试：

`Son son = new Son();`

打印结果如下：

```text
Father static before
Father static
Father static after
Son static
Father constructor
Son static function
Son non-static function
Son constructor
```

上面的代码直接将 `Son` 实例化，同样需要先将 Class 文件加载至虚拟机并完成初始化，因此前四行的打印结果与上例相同。注意第 6、7 行：`Son static function` 和 `Son non-static function` 来自实例变量 `a = fun()`、`b = fun2()` 的初始化，它们在父类构造方法执行完之后、子类构造方法体执行之前运行。

可见，完整的执行顺序为：

1. 父类的静态变量赋值和静态代码块（按出现顺序）
2. 子类的静态变量赋值和静态代码块（按出现顺序）
3. 父类的实例变量赋值、实例代码块和构造方法
4. 子类的实例变量赋值和实例代码块（按出现顺序）
5. 子类的构造方法体

简单概括就是：**先执行静态代码，再执行实例代码；先执行父类，再执行子类**。其中静态部分只在类第一次初始化时执行一次，实例部分每次 `new` 都会执行。

# 6 Java 版本演进中的 static

随着 Java 版本演进，`static` 的使用规则也有一些变化：

- **Java 8**：接口中可以定义静态方法，只能通过接口名调用，不会被实现类继承。
- **Java 16**（JEP 395）：放宽了内部类（非静态嵌套类）不能声明静态成员的限制，内部类中也可以定义静态变量、静态方法以及 `record` 等隐式静态的成员。
- **Java 25**（JEP 512）：正式支持紧凑源文件和实例 `main` 方法，入门程序不再必须写 `public static void main(String[] args)`。