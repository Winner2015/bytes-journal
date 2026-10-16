# 1、概述

ThreadLocal`为解决多线程程序的并发问题提供了一种新的思路。使用这个工具类可以很简洁地编写出优美的多线程程序，ThreadLocal并不是一个Thread，而是Thread的局部变量。

线程局部变量高效地为每个使用它的线程提供单独的线程局部变量值的副本。每个线程只能看到与自己相联系的值，而不知道别的线程可能正在使用或修改它们自己的副本。ThreadLocal 实例通常是类中的 `private static` 字段，它们希望将状态与某一个线程（例如，用户 ID 或事务 ID）相关联。

每个线程都保持对其线程局部变量副本的隐式引用，只要线程是活动的并且 `ThreadLocal` 实例是可访问的；在线程消失之后，其线程局部实例的所有副本都会被垃圾回收（除非存在对这些副本的其他引用）。

`ThreadLocal`是如何做到为每一个线程维护变量的副本的呢？其实实现的思路很简单：在`ThreadLocal`类中定义了一个`ThreadLocalMap`，每一个`Thread`中都有一个该类型的变量——`threadLocals`——用于存储每一个线程的变量副本，Map中元素的键为线程对象，而值对应线程的变量副本。

# 2、源码解读

ThreadLocal类的四个方法如下：

- `T get() `
  返回此线程局部变量的当前线程副本中的值。如果变量没有用于当前线程的值，则先将其初始化为调用 initialValue() 方法返回的值。
- `protected  T  initialValue()` 
  返回此线程局部变量的当前线程的“初始值”。这个方法是一个延迟调用方法，线程第一次使用 get() 方法访问变量时将调用此方法，但如果线程之前调用了 set(T) 方法，则不会对该线程再调用 initialValue 方法。通常，此方法对每个线程最多调用一次，但如果在调用 get() 后又调用了 remove()，则可能再次调用此方法。该实现返回 null；如果程序员希望线程局部变量具有 null 以外的值，则必须为 ThreadLocal 创建子类，并重写此方法。通常将使用匿名内部类完成此操作。
- `void remove() `
  移除此线程局部变量当前线程的值。如果此线程局部变量随后被当前线程读取，且这期间当前线程没有设置其值，则将调用其 initialValue() 方法重新初始化其值。这将导致在当前线程多次调用 initialValue 方法。
- `void set(T value) `
  将此线程局部变量的当前线程副本中的值设置为指定值。大部分子类不需要重写此方法，它们只依靠 initialValue() 方法来设置线程局部变量的值。

`get()`方法是用来获取`ThreadLocal`在当前线程中保存的变量副本。 

先看下get方法的实现：

```java
	/**
	 * Returns the value in the current thread's copy of this thread-local variable.
	 * If the variable has no value for the current thread, it is first initialized
	 * to the value returned by an invocation of the {@link #initialValue} method.
	 *
	 * @return the current thread's value of this thread-local
	 */
	public T get() {
		Thread t = Thread.currentThread();
		ThreadLocalMap map = getMap(t);
		if (map != null) {
			ThreadLocalMap.Entry e = map.getEntry(this);
			if (e != null)
				return (T) e.value;
		}
		return setInitialValue();
	}
```

第一行代码获取当前线程。

第二行代码利用`getMap()`方法获取当前线程对应的`ThreadLocalMap`。

getMap()方法的代码如下：

```java
	/**
	 * 
	 * Get the map associated with a ThreadLocal. Overridden in
	 * InheritableThreadLocal.
	 *
	 * @param t
	 *            the current thread
	 * @return the map
	 */
	ThreadLocalMap getMap(Thread t) {
		return t.threadLocals;
	}
```

可见，`getMap()`方法返回的是`Thread`类中一个叫`threadLocals`的字段。查看`Thread`类的源码，可以发现，每个`Thread`实例都有一个`ThreadLocalMap`类型的成员变量：

```java
	/*
	 * ThreadLocal values pertaining to this thread. This map is maintained by the
	 * ThreadLocal class.
	 */
	ThreadLocal.ThreadLocalMap threadLocals = null;
```

`ThreadLocalMap`实际上是`ThreadLocal`类中的一个静态内部类：

```java
    /**
     * The entries in this hash map extend WeakReference, using
     * its main ref field as the key (which is always a
     * ThreadLocal object).  Note that null keys (i.e. entry.get()
     * == null) mean that the key is no longer referenced, so the
     * entry can be expunged from table.  Such entries are referred to
     * as "stale entries" in the code that follows.
     */
    static class Entry extends WeakReference<ThreadLocal> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal k, Object v) {
            super(k);
            value = v;
        }
    }

```

`ThreadLocalMap`的`Entry`继承了`WeakReference`，并且使用`ThreadLocal`作为键值。正因如此，第四行代码：

```java
ThreadLocalMap.Entry e = map.getEntry(this);
```

获取`Entry`时传入的参数是`this`，即当前的`ThreadLocal`实例，而非第一行代码获取的当前线程`t`。

如果得到的`Entry`不为空，直接返回，如果为空，则调用`setInitialValue()`方法：

```java
	/**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * @return the initial value
     */
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```

如果当前线程的`threadLocals`不为空，将初始化值放入`threadLocals`，如果为空，则新建一个`ThreadLocalMap`，赋值给当前线程的`threadLocals`。

```java
	  /**
     * Create the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the map
     * @param map the map to store.
     */
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

看到到这里可能有点乱，从头理一下思路。首先，在每个线程Thread内部有一个`ThreadLocal.ThreadLocalMap`类型的成员变量`threadLocals`，这个`threadLocals`就是用来存储实际的变量副本的，键值为当前`ThreadLocal`变量，`value`为变量副本（即T类型的变量）。

初始时，在`Thread`里面的`threadLocals`为null，当通过`ThreadLocal`变量调用`get()`方法或者`set()`方法，就会对`Thread`类中的`threadLocals`进行初始化，并且以当前`ThreadLocal`变量为键值，以`ThreadLocal`要保存的副本变量为`value`，存到`threadLocals`。

在当前线程里面，如果要使用副本变量，就可以通过`get`方法在`threadLocals`里面查找。

`set()`用来设置当前线程中变量的副本：

```java
	/**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

与`get()`类似，也是先获取当前线程的`ThreadLocalMap`，然后以当前`ThreadLocal`、指定`value`为键值对的`Entry`存入该`ThreadLocalMap`（如果`ThreadLocalMap`为空，则需要先创建再存入）。

`initialValue()`是一个`protected`方法，默认直接返回null，一般需要重写的，用以设置初始值。

```java
	/**
     * Returns the current thread's "initial value" for this
     * thread-local variable.  This method will be invoked the first
     * time a thread accesses the variable with the {@link #get}
     * method, unless the thread previously invoked the {@link #set}
     * method, in which case the <tt>initialValue</tt> method will not
     * be invoked for the thread.  Normally, this method is invoked at
     * most once per thread, but it may be invoked again in case of
     * subsequent invocations of {@link #remove} followed by {@link #get}.
     *
     * <p>This implementation simply returns <tt>null</tt>; if the
     * programmer desires thread-local variables to have an initial
     * value other than <tt>null</tt>, <tt>ThreadLocal</tt> must be
     * subclassed, and this method overridden.  Typically, an
     * anonymous inner class will be used.
     *
     * @return the initial value for this thread-local
     */
    protected T initialValue() {
        return null;
    }
```

`remove()`用来移除当前线程中变量的副本，实现起来很简单，直接删除以当前`ThreadLocal`为键值的`Entry`：

```java
	/**
     * Removes the current thread's value for this thread-local
     * variable.  If this thread-local variable is subsequently
     * {@linkplain #get read} by the current thread, its value will be
     * reinitialized by invoking its {@link #initialValue} method,
     * unless its value is {@linkplain #set set} by the current thread
     * in the interim.  This may result in multiple invocations of the
     * <tt>initialValue</tt> method in the current thread.
     *
     * @since 1.5
     */
     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
```

# 3、实例

下面通过代码来体会`ThreadLocal`的魅力：

```java
public class Test {

	// 该类的第一个ThreadLocal变量，用来存储线程ID
	ThreadLocal<Long> longLocal = new ThreadLocal<Long>() {
		protected Long initialValue() { // 覆写initialValue()方法
			return Thread.currentThread().getId();
		};
	};

	// 该类的第二个ThreadLocal变量，用来存储线程名称
	ThreadLocal<String> stringLocal = new ThreadLocal<String>() {
		;
		protected String initialValue() { // 覆写initialValue()方法
			return Thread.currentThread().getName();
		};
	};

	public void set() {
		longLocal.set(Thread.currentThread().getId());
		stringLocal.set(Thread.currentThread().getName());
	}

	public long getLong() {
		return longLocal.get();
	}

	public String getString() {
		return stringLocal.get();
	}

	public static void main(String[] args) throws InterruptedException {
		final Test test = new Test();

		Thread thread1 = new Thread() {
			public void run() {
				test.set();
				System.out.println(test.getLong());
				System.out.println(test.getString());
			};
		};

		Thread thread2 = new Thread() {
			public void run() {
				test.set();
				System.out.println(test.getLong());
				System.out.println(test.getString());
			};
		};

		thread1.start();
		thread1.join();

		thread2.start();
		thread2.join();

		System.out.println(test.getLong());
		System.out.println(test.getString());
	}
}
```

打印结果如下：

> 8
> Thread-0
> 9
> Thread-1
> 1
> main

我们来分析一下上面的代码，需要注意的是，实际上一共运行了三个线程：线程thread1、线程thread2和主线程main。

虽然三个线程执行了同样的代码：

```java
System.out.println(test.getLong());
System.out.println(test.getString());
```

但三次打印的结果并不一样。

首先启动thread1，执行`test.set()`的时候，是将thread1的线程ID与名称存入了thread1的`threadLocals`变量中，然后打印出存入的线程ID与名称。

然后启动thread1，执行`test.set()`的时候，是将thread2的线程ID与名称存入了thread2的`threadLocals`变量中，与thread1实例中存入的变量是互不影响的两个副本。

同样的，最后打印的是主线程main的线程ID与名称，与thread1和thread2当中存入的变量副本也是互不影响。

由此证明，通过`ThreadLocal`达到了在每个线程中创建变量副本的效果。

在ThreadLocal源码中我们经常看到这样的语句：

```java
Thread t = Thread.currentThread();
ThreadLocalMap map = getMap(t);
```

其精髓就在于`ThreadLocal`的`set()`方法、`get()`等方法都是通过`Thread.currentThread()`这把钥匙打开了当前线程的“小金库”，然后针对每个线程进行互不影响的存取操作。

线程局部变量常被用来描绘有状态“单例”（Singleton） 或线程安全的共享对象，或者是通过把不安全的整个变量封装进 `ThreadLocal`，或者是通过把对象的特定于线程的状态封装进 `ThreadLocal`。

例如，在与数据库有紧密联系的应用程序中，程序的很多方法可能都需要访问数据库。在系统的每个方法中都包含一个 `Connection` 作为参数是不方便的。用“单例”来访问连接可能是一个虽然更粗糙，但却方便得多的技术。然而，多个线程不能安全地共享一个 JDBC Connection。通过使用“单例”中的 `ThreadLocal`，我们就能让我们的程序中的任何类容易地获取每线程 Connection 的一个引用。当要给线程初始化一个特殊值时，需要自己实现`ThreadLocal`的子类并重写`initialValue()`方法（该方法缺省地返回null），通常使用一个内部匿名类对`ThreadLocal`进行子类化。

如下例所示：

```java
public class ConnectionDispenser {
	private static class ThreadLocalConnection extends ThreadLocal {
		public Object initialValue() {
			return DriverManager.getConnection(ConfigurationSingleton.getDbUrl());
		}
	}

	private ThreadLocalConnection conn = new ThreadLocalConnection();

	public static Connection getConnection() {
		return (Connection) conn.get();
	}
}
```

这是非常常用的一种方案。EasyDBO中创建jdbc连接上下文就是这样做的：

```java
public class JDBCContext {

	private static Logger logger = Logger.getLogger(JDBCContext.class);
	private DataSource ds;
	protected Connection connection;

	public static JDBCContext getJdbcContext(javax.sql.DataSource ds) {
		if (jdbcContext == null)
			jdbcContext = new JDBCContextThreadLocal(ds);
		JDBCContext context = (JDBCContext) jdbcContext.get();
		if (context == null)
			context = new JDBCContext(ds);

		return context;
	}

	// 继承了ThreadLocal的内部类
	private static class JDBCContextThreadLocal extends ThreadLocal {

		public javax.sql.DataSource ds;

		public JDBCContextThreadLocal(javax.sql.DataSource ds) {
			this.ds = ds;
		}

		// 重写initialValue()方法
		protected synchronized Object initialValue() {
			return new JDBCContext(ds);
		}
	}
}
```

`ThreadLocal`的思想在Hibernate、Spring框架中广为使用。

下面的实例能够体现Spring对有状态Bean的改造思路：

```java
	public class TopicDao {
		private Connection conn;// 一个非线程安全的变量

		public void addTopic(){
			Statement stat = conn.createStatement();//引用非线程安全变量
			…
		}
	}
```

由于`conn`是成员变量，因为`addTopic()`方法是非线程安全的，必须在使用时创建一个新`TopicDao`实例（非singleton）。下面使用`ThreadLocal`对`conn`这个非线程安全的“状态”进行改造：

```java
	public class TopicDao {
		// 使用ThreadLocal保存Connection变量
		private static ThreadLocal<Connection> connThreadLocal = new ThreadLocal<Connection>();

		public static Connection getConnection() {
			// 如果connThreadLocal没有本线程对应的Connection创建一个新的Connection，并将其保存到线程本地变量中。
		}

		public void addTopic() {
			// 从ThreadLocal中获取线程对应的Connection
			Statement stat = getConnection().createStatement();
		}
	}
```

不同的线程在使用`TopicDao`时，这样，就保证了不同的线程使用线程相关的`Connection`，而不会使用其它线程的`Connection`。因此，这个`TopicDao`就可以做到singleton共享了。

当然，这个例子本身很粗糙，将`Connection`的`ThreadLocal`直接放在DAO只能做到本DAO的多个方法共享`Connection`时不发生线程安全问题，但无法和其它DAO共用同一个`Connection`，要做到同一事务多DAO共享同一`Connection`，必须在一个共同的外部类使用`ThreadLocal`保存`Connection`。

# 4、与同步机制的对比

`ThreadLocal`和线程同步机制相比有什么优势呢？`ThreadLocal`和线程同步机制都是为了解决多线程中相同变量的访问冲突问题。

在同步机制中，通过对象的锁机制保证同一时间只有一个线程访问变量。这时该变量是多个线程共享的，使用同步机制要求程序慎密地分析什么时候对变量进行读写，什么时候需要锁定某个对象，什么时候释放对象锁等繁杂的问题，程序设计和编写难度相对较大。

而`ThreadLocal`则从另一个角度来解决多线程的并发访问。在编写多线程代码时，可以把不安全的变量封装进`ThreadLocal`。

由于`ThreadLocal`中可以持有任何类型的对象，低版本JDK所提供的`get()`返回的是`Object`对象，需要强制类型转换。但JDK 5.0通过泛型很好的解决了这个问题，在一定程度地简化`ThreadLocal`的使用。

概括起来说，对于多线程资源共享的问题，同步机制采用了“**以时间换空间**”的方式，而`ThreadLocal`采用了“**以空间换时间**”的方式。前者仅提供一份变量，让不同的线程排队访问，而后者为每一个线程都提供了一份变量，因此可以同时访问而互不影响。

`ThreadLocal`是解决线程安全问题一个很好的思路，它通过为每个线程提供一个独立的变量副本解决了变量并发访问的冲突问题。在很多情况下，`ThreadLocal`比直接使用`synchronized`同步机制解决线程安全问题更简单，更方便，且结果程序拥有更高的并发性。