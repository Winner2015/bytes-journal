# 1、static变量 

按照是否静态的对类成员变量进行分类可分两种：

- 被`static`修饰的变量，叫**静态变量**或**类变量**，`static`成员变量的初始化顺序按照定义的顺序进行初始化；
- 没有被static修饰的变量，叫**实例变量**。

两者的区别是：对于静态变量在内存中只有一个拷贝（节省内存），**JVM只为静态分配一次内存，在加载类的过程中完成静态变量的内存分配**，可用类名直接访问（方便），当然也可以通过对象来访问（但是这是不推荐的）。对于实例变量，每创建一个实例，就会为实例变量分配一次内存，实例变量可以在内存中有多个拷贝，互不影响。 

所以一般在需要实现以下两个功能时使用静态变量：

- 在对象之间共享值时
- 方便访问变量时

# 2、static方法

静态方法可以直接通过类名调用，任何的实例也都可以调用，因此**静态方法中不能用`this`和`super`关键字**，不能直接访问所属类的实例变量和实例方法(就是不带static的成员变量和成员成员方法)，**只能访问所属类的静态成员变量和成员方法**，因为实例成员与特定的对象关联。

但是要注意的是，虽然在静态方法中不能访问非静态成员方法和非静态成员变量，但是在非静态成员方法中是可以访问静态成员方法/变量的。

`static`方法独立于任何实例，因此static方法必须被实现，而不能是抽象的`abstract`。

例如为了方便方法的调用，Java API中的`Math`类中所有的方法都是静态的，而一般类内部的`static`方法也是方便其它类对该方法的调用。

静态方法是类内部的一类特殊方法，只有在需要时才将对应的方法声明成静态的，一个类内部的方法一般都是非静态的
因此，如果说想在不创建对象的情况下调用某个方法，就可以将这个方法设置为`static`。我们最常见的`static`方法就是`main`方法，至于为什么main方法必须是static的，现在就很清楚了。因为程序在执行main方法的时候没有创建任何对象，因此只有通过类名来访问。

另外记住，即使没有显示地声明为static，**类的构造器实际上也是静态方法**。

# 3、static代码块 

`static`代码块也叫**静态代码块**，是在类中独立于类成员的`static`语句块，可以有多个，位置可以随便放，它不在任何的方法体内，JVM加载类时会执行这些静态的代码块，如果static代码块有多个，**JVM将按照它们在类中出现的先后顺序依次执行它们，每个代码块只会被执行一次**。

`static`关键字还有一个比较关键的作用就是，用来形成静态代码块以优化程序性能。

为什么说static块可以用来优化程序性能，是因为它的特性：**只会在类加载的时候执行一次**。

下面看个例子:

```java
class Person {
	private Date birthDate;

	public Person(Date birthDate) {
		this.birthDate = birthDate;
	}

	boolean isBornBoomer() {
		Date startDate = Date.valueOf("1946");
		Date endDate = Date.valueOf("1964");
		return birthDate.compareTo(startDate) >= 0 && birthDate.compareTo(endDate) < 0;
	}

}
```

`isBornBoomer`是用来这个人是否是1946-1964年出生的，而每次`isBornBoomer`被调用的时候，都会生成`startDate`和`birthDate`两个对象，造成了空间浪费，如果改成这样效率会更好：

```java
class Person {
	private Date birthDate;
	private static Date startDate, endDate;
	static {
		startDate = Date.valueOf("1946");
		endDate = Date.valueOf("1964");
	}

	public Person(Date birthDate) {
		this.birthDate = birthDate;
	}

	boolean isBornBoomer() {
		return birthDate.compareTo(startDate) >= 0 && birthDate.compareTo(endDate) < 0;
	}

}
```

因此，很多时候会将一些只需要进行一次的初始化操作都放在`static`代码块中进行。

# 4、final static

`static final`用来修饰成员变量和成员方法，可简单理解为“**全局常量**”。

- 对于变量，表示一旦给值就不可修改，并且通过类名可以访问。 
- 对于方法，表示不可覆盖，并且可以通过类名直接访问。

对于被static和final修饰过的实例常量，实例本身不能再改变了，但对于一些容器类型（比如，ArrayList、HashMap）的实例变量，**不可以改变容器变量本身，但可以修改容器中存放的对象**，这一点在编程中用到很多。

```java
	private static final String strStaticFinalVar = "aaa";
	private static String strStaticVar = null;
	private final String strFinalVar = null;
	private static final int intStaticFinalVar = 0;
	private static final Integer integerStaticFinalVar = new Integer(8);
	private static final ArrayList<String> alStaticFinalVar = new ArrayList<String>();

	private void test() {
		
		// strStaticFinalVar="哈哈哈哈"; //错误，final表示终态,不可以改变变量本身.
		strStaticVar = "哈哈哈哈"; // 正确，static表示类变量,值可以改变.
		
		// strFinalVar="呵呵呵呵"; //错误, final表示终态，在定义的时候就要初值（哪怕给个null），一旦给定后就不可再更改。
		// intStaticFinalVar=2; //错误, final表示终态，在定义的时候就要初值（哪怕给个null），一旦给定后就不可再更改。
		// integerStaticFinalVar=new Integer(8); //错误, final表示终态，在定义的时候就要初值（哪怕给个null），一旦给定后就不可再更改。
		alStaticFinalVar.add("aaa"); // 正确，容器变量本身没有变化，但存放内容发生了变化。这个规则是非常常用的，有很多用途。
		alStaticFinalVar.add("bbb"); // 正确，容器变量本身没有变化，但存放内容发生了变化。这个规则是非常常用的，有很多用途。
	}
```

# 5、执行顺序

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

> Father static before
> Father static
> Father static after
> Son static

`Class.forName`是将类加载到JVM，可见加载子类之前，需要先加载父类，并按照出现的顺序执行其中的静态代码块、静态方法（如果有调用）。

改用下面的代码测试：

`Son son = new Son();`

打印结果如下：

> Father static before
> Father static
> Father static after
> Son static
> Father constructor
> Son static function
> Son non-static function
> Son constructor

上面的代码直接将`Son`实例化，同样需要先将Class文件加载至虚拟机，因此前四行的打印结果与上例相同。
可见，代码的执行顺序为：**先执行静态代码，再执行构造方法；先执行父类，在执行子类**。