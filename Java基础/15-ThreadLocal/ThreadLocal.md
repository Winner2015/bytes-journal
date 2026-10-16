# ThreadLocal

# 1 概述

`ThreadLocal` 为解决多线程程序的并发问题提供了另一种思路：不共享，就不会冲突。`ThreadLocal` 并不是一个 `Thread`，而是“线程的局部变量”。

线程局部变量为每个使用它的线程提供一份独立的变量副本。每个线程只能看到与自己关联的值，而看不到别的线程正在使用或修改的副本。`ThreadLocal` 实例通常声明为类中的 `private static final` 字段，用于把某种状态（例如用户 ID、事务 ID、请求追踪 ID）与当前线程关联起来。

只要线程存活并且 `ThreadLocal` 实例可访问，每个线程就会持有对其线程局部变量副本的隐式引用；线程结束后，它持有的所有线程局部副本都可以被垃圾回收（除非还存在对这些副本的其他引用）。

`ThreadLocal` 是如何为每个线程维护变量副本的呢？实现思路其实很简单：`ThreadLocal` 类中定义了一个静态内部类 `ThreadLocalMap`，每个 `Thread` 对象中都有一个该类型的字段 `threadLocals`，用于存储这个线程的所有变量副本。**Map 中元素的键是 `ThreadLocal` 实例本身，值是当前线程的变量副本**。

换句话说，数据并不存放在 `ThreadLocal` 里，而是存放在每个线程自己的 `Thread` 对象里，`ThreadLocal` 只是访问这份数据的“钥匙”。

# 2 源码解读

`ThreadLocal` 类的核心方法如下：

- `T get()`：返回当前线程副本中的值。如果当前线程还没有值，则先调用 `initialValue()` 将其初始化。
- `protected T initialValue()`：返回当前线程的“初始值”。这是一个延迟调用方法，线程第一次调用 `get()` 时才会调用它；但如果线程之前已经调用过 `set(T)`，则不会再调用 `initialValue()`。通常此方法对每个线程最多调用一次，但如果在 `get()` 之后又调用了 `remove()`，再次 `get()` 时会重新调用。默认实现返回 `null`。
- `static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier)`：JDK 8 新增的工厂方法，用 Lambda 指定初始值，取代“匿名子类重写 `initialValue()`”的写法。
- `void set(T value)`：将当前线程副本中的值设置为指定值。
- `void remove()`：移除当前线程副本中的值。如果随后当前线程再次读取且期间没有调用 `set()`，则会重新调用 `initialValue()` 初始化。

下面的源码取自 JDK 8~17。更新的 JDK 为支持虚拟线程把部分逻辑拆到了私有方法中，但整体流程不变。

## 2.1 get

`get()` 方法用来获取 `ThreadLocal` 在当前线程中保存的变量副本：

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T) e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

第一行代码获取当前线程。

第二行代码利用 `getMap()` 方法获取当前线程对应的 `ThreadLocalMap`：

```java
/**
 * Get the map associated with a ThreadLocal. Overridden in
 * InheritableThreadLocal.
 */
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

可见，`getMap()` 方法返回的是 `Thread` 类中一个叫 `threadLocals` 的字段。查看 `Thread` 类的源码可以发现，每个 `Thread` 实例都有一个 `ThreadLocalMap` 类型的成员变量：

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

`ThreadLocalMap` 是 `ThreadLocal` 的静态内部类，它的 `Entry` 定义如下：

```java
/**
 * The entries in this hash map extend WeakReference, using
 * its main ref field as the key (which is always a
 * ThreadLocal object).  Note that null keys (i.e. entry.get()
 * == null) mean that the key is no longer referenced, so the
 * entry can be expunged from table.  Such entries are referred to
 * as "stale entries" in the code that follows.
 */
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

`Entry` 继承了 `WeakReference`，以 `ThreadLocal` 作为键（弱引用），以变量副本作为值（强引用）。正因如此，`get()` 中的这行代码：

```java
ThreadLocalMap.Entry e = map.getEntry(this);
```

传入的参数是 `this`，即当前的 `ThreadLocal` 实例，而不是第一行获取的当前线程 `t`。键为弱引用这一设计与内存泄漏问题密切相关，详见第 5 章。

如果得到的 `Entry` 不为空，直接返回其值；如果为空，则调用 `setInitialValue()` 方法：

```java
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

如果当前线程的 `threadLocals` 不为空，就将初始值放入 `threadLocals`；如果为空，则新建一个 `ThreadLocalMap` 赋值给当前线程的 `threadLocals`：

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

看到这里可能有点乱，从头理一下思路：

1. 每个 `Thread` 内部有一个 `ThreadLocal.ThreadLocalMap` 类型的成员变量 `threadLocals`，用来存储实际的变量副本，键为 `ThreadLocal` 实例，值为变量副本（即 `T` 类型的对象）。
2. 初始时，`Thread` 里的 `threadLocals` 为 `null`。当第一次通过 `ThreadLocal` 调用 `get()` 或 `set()` 时，才会初始化 `threadLocals`，并以当前 `ThreadLocal` 为键、要保存的副本为值存入其中。
3. 之后在当前线程中使用副本变量时，就可以通过 `get()` 在 `threadLocals` 中查找。

## 2.2 set

`set()` 用来设置当前线程中变量的副本：

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

与 `get()` 类似，也是先获取当前线程的 `ThreadLocalMap`，然后把以当前 `ThreadLocal` 为键、指定 `value` 为值的 `Entry` 存入其中（如果 `ThreadLocalMap` 为空，则先创建再存入）。

## 2.3 initialValue 与 withInitial

`initialValue()` 是一个 `protected` 方法，默认直接返回 `null`，需要初始值时可以重写：

```java
protected T initialValue() {
    return null;
}
```

在 JDK 8 之前，通常用匿名内部类重写它：

```java
private static final ThreadLocal<SimpleDateFormat> FORMAT = new ThreadLocal<SimpleDateFormat>() {
    @Override
    protected SimpleDateFormat initialValue() {
        return new SimpleDateFormat("yyyy-MM-dd");
    }
};
```

JDK 8 起推荐使用 `withInitial()`：

```java
private static final ThreadLocal<SimpleDateFormat> FORMAT =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
```

## 2.4 remove

`remove()` 用来移除当前线程中变量的副本，实现很简单，直接删除以当前 `ThreadLocal` 为键的 `Entry`：

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

`remove()` 看似简单，却是正确使用 `ThreadLocal` 的关键，第 5 章会详细说明。

## 2.5 ThreadLocalMap 的结构

`ThreadLocalMap` 没有复用 `HashMap`，而是一个专门定制的哈希表：

- **哈希值**：每个 `ThreadLocal` 实例创建时都会分配一个 `threadLocalHashCode`，由一个全局计数器每次累加 `0x61c88647` 得到。这个数与黄金分割比相关，能让哈希值在 2 的幂次容量的数组中分布得非常均匀。
- **冲突解决**：采用开放寻址法中的线性探测，冲突时依次检查下一个槽位，而不是像 `HashMap` 那样挂链表。
- **扩容**：初始容量为 16，阈值为容量的 2/3。扩容前会先清理所有过期条目（键已被回收的 `Entry`），清理后仍然拥挤才把容量翻倍。
- **过期条目清理**：`get()`、`set()`、`remove()` 在探测过程中遇到键为 `null` 的条目时，会顺带把它清理掉（`expungeStaleEntry`）。

一个线程通常只持有少量 `ThreadLocal`，因此这种结构简单、紧凑且足够高效。

# 3 实例

下面通过代码来体会 `ThreadLocal` 的效果：

```java
public class Test {

    // 第一个 ThreadLocal 变量，用来存储线程 ID
    private final ThreadLocal<Long> longLocal =
            ThreadLocal.withInitial(() -> Thread.currentThread().threadId());

    // 第二个 ThreadLocal 变量，用来存储线程名称
    private final ThreadLocal<String> stringLocal =
            ThreadLocal.withInitial(() -> Thread.currentThread().getName());

    public void set() {
        longLocal.set(Thread.currentThread().threadId());
        stringLocal.set(Thread.currentThread().getName());
    }

    public long getLong() {
        return longLocal.get();
    }

    public String getString() {
        return stringLocal.get();
    }

    public static void main(String[] args) throws InterruptedException {
        Test test = new Test();

        Runnable task = () -> {
            test.set();
            System.out.println(test.getLong());
            System.out.println(test.getString());
        };

        Thread thread1 = new Thread(task);
        Thread thread2 = new Thread(task);

        thread1.start();
        thread1.join();

        thread2.start();
        thread2.join();

        System.out.println(test.getLong());
        System.out.println(test.getString());
    }
}
```

`Thread.threadId()` 是 JDK 19 新增的方法，用来取代被标记为 `@Deprecated` 的 `getId()`（`getId()` 是可被子类重写的非 `final` 方法，不能保证返回真实的线程 ID）。在 JDK 19 之前的版本中运行，可以把 `threadId()` 改回 `getId()`。

打印结果如下（线程 ID 的具体数值与 JDK 版本有关）：

```text
8
Thread-0
9
Thread-1
1
main
```

我们来分析一下上面的代码。需要注意的是，实际上一共运行了三个线程：`thread1`、`thread2` 和主线程 `main`。

虽然三个线程执行了同样的代码：

```java
System.out.println(test.getLong());
System.out.println(test.getString());
```

但三次打印的结果并不一样：

- 首先启动 `thread1`，执行 `test.set()` 时，将 `thread1` 的线程 ID 与名称存入了 `thread1` 自己的 `threadLocals` 中，然后打印出存入的值。
- 然后启动 `thread2`，执行 `test.set()` 时，将 `thread2` 的线程 ID 与名称存入了 `thread2` 的 `threadLocals` 中，与 `thread1` 存入的是互不影响的两份副本。
- 最后，主线程 `main` 并没有调用 `set()`，第一次 `get()` 时触发了 `withInitial()` 中的初始化逻辑，打印出主线程的 ID 与名称，同样与前两个线程互不影响。

由此证明，`ThreadLocal` 达到了在每个线程中创建变量副本的效果。

在 `ThreadLocal` 源码中我们经常看到这样的语句：

```java
Thread t = Thread.currentThread();
ThreadLocalMap map = getMap(t);
```

其精髓就在于 `ThreadLocal` 的 `set()`、`get()` 等方法都是通过 `Thread.currentThread()` 这把钥匙打开当前线程的“小金库”，然后针对每个线程进行互不影响的存取操作。

# 4 典型应用场景

线程局部变量常被用来实现有状态的“单例”或线程安全的共享对象：要么把非线程安全的整个对象封装进 `ThreadLocal`，要么把对象中特定于线程的状态封装进 `ThreadLocal`。

## 4.1 数据库连接

在与数据库紧密联系的应用程序中，很多方法都需要访问数据库，在每个方法中都把 `Connection` 作为参数传递是不方便的。用“单例”来访问连接是一种虽然粗糙、但方便得多的做法。然而，多个线程不能安全地共享同一个 JDBC `Connection`。通过在“单例”中使用 `ThreadLocal`，程序中的任何类都能方便地拿到“当前线程专属”的 `Connection`：

```java
public class ConnectionDispenser {

    private static final String DB_URL = "jdbc:mysql://localhost:3306/test";

    private static final ThreadLocal<Connection> CONNECTION = ThreadLocal.withInitial(() -> {
        try {
            return DriverManager.getConnection(DB_URL);
        } catch (SQLException e) {
            throw new IllegalStateException(e);
        }
    });

    public static Connection getConnection() {
        return CONNECTION.get();
    }

    public static void release() throws SQLException {
        try {
            CONNECTION.get().close();
        } finally {
            CONNECTION.remove();
        }
    }
}
```

实际项目中，连接通常来自连接池（如 HikariCP），并由框架统一管理获取与释放，这里只用来演示思路。

## 4.2 Spring 对有状态 Bean 的改造

`ThreadLocal` 的思想在 Hibernate、Spring 等框架中被广泛使用。下面的例子能够体现 Spring 对有状态 Bean 的改造思路：

```java
public class TopicDao {
    private Connection conn; // 一个非线程安全的变量

    public void addTopic() throws SQLException {
        Statement stat = conn.createStatement(); // 引用非线程安全变量
        // ...
    }
}
```

由于 `conn` 是成员变量，`addTopic()` 方法是非线程安全的，只能在每次使用时创建一个新的 `TopicDao` 实例（非单例）。下面使用 `ThreadLocal` 对 `conn` 这个非线程安全的“状态”进行改造：

```java
public class TopicDao {

    private static final String DB_URL = "jdbc:mysql://localhost:3306/test";

    // 使用 ThreadLocal 保存 Connection 变量
    private static final ThreadLocal<Connection> CONN_HOLDER = new ThreadLocal<>();

    public static Connection getConnection() throws SQLException {
        // 如果当前线程还没有对应的 Connection，就创建一个并保存到线程本地变量中
        Connection conn = CONN_HOLDER.get();
        if (conn == null) {
            conn = DriverManager.getConnection(DB_URL);
            CONN_HOLDER.set(conn);
        }
        return conn;
    }

    public void addTopic() throws SQLException {
        // 从 ThreadLocal 中获取当前线程对应的 Connection
        try (Statement stat = getConnection().createStatement()) {
            // ...
        }
    }
}
```

这样，不同的线程在使用 `TopicDao` 时，各自使用与自己线程相关的 `Connection`，而不会用到其他线程的 `Connection`，因此 `TopicDao` 就可以作为单例共享了。

当然，这个例子本身很粗糙：把 `Connection` 的 `ThreadLocal` 直接放在 DAO 中，只能保证本 DAO 的多个方法共享 `Connection` 时不发生线程安全问题，却无法和其他 DAO 共用同一个 `Connection`。要做到同一事务中多个 DAO 共享同一个 `Connection`，必须在一个公共的外部类中用 `ThreadLocal` 保存 `Connection`。

Spring 正是这样做的：事务管理器开启事务时，把连接绑定到 `TransactionSynchronizationManager` 中的 `ThreadLocal` 上，同一线程内的 `JdbcTemplate`、MyBatis 等通过 `DataSourceUtils.getConnection()` 取到的都是同一个连接，事务结束时再统一提交或回滚并解绑。

## 4.3 其他常见场景

- **非线程安全对象的复用**：例如为每个线程缓存一个 `SimpleDateFormat`。不过从 JDK 8 起，更好的做法是直接使用线程安全的 `DateTimeFormatter`。
- **请求上下文传递**：在 Web 请求入口把当前用户、租户、追踪 ID 放入 `ThreadLocal`，后续各层代码无需层层传参即可获取，例如 Spring 的 `RequestContextHolder`、`SecurityContextHolder`。
- **日志 MDC**：SLF4J/Logback 的 `MDC` 底层也基于 `ThreadLocal`，用于在日志中自动输出请求 ID 等信息。

# 5 内存泄漏与正确用法

## 5.1 为什么会泄漏

回顾 `ThreadLocalMap` 的引用关系：

```text
Thread ──强引用──> ThreadLocalMap ──强引用──> Entry
                                              ├── key   ──弱引用──> ThreadLocal 实例
                                              └── value ──强引用──> 变量副本
```

当 `ThreadLocal` 实例在外部不再有强引用时（例如它不是静态字段，所在对象已被回收），GC 会回收这个 `ThreadLocal`，`Entry` 的键变为 `null`。但 `value` 仍然被 `Thread → ThreadLocalMap → Entry` 这条强引用链引用着，只要线程不结束，`value` 就不会被回收。

之所以把键设计为弱引用，是为了“至少能让键被回收”，从而让过期条目可以被识别出来，并在后续的 `get()`、`set()`、`remove()` 中被顺带清理。但这种清理是被动、不确定的：如果之后再也没有调用这些方法，泄漏就会一直存在。

在普通线程中，线程结束后整个 `ThreadLocalMap` 都会被回收，问题不大。**真正的风险在线程池中**：线程池的线程会被长期复用，几乎不会结束，`ThreadLocal` 中残留的数据会一直存在。

## 5.2 比泄漏更危险的：数据串用

在线程池中，残留数据不仅占用内存，还会被下一个任务读到。比如上一个请求在 `ThreadLocal` 中放入了用户 A 的信息却没有清理，下一个复用该线程的请求就可能“以用户 A 的身份”执行，这类问题往往比内存泄漏更隐蔽、后果也更严重。

## 5.3 正确用法

- **用完必须 `remove()`，并放在 `finally` 中**：

```java
private static final ThreadLocal<User> CURRENT_USER = new ThreadLocal<>();

public void handle(Request request) {
    CURRENT_USER.set(request.getUser());
    try {
        doBusiness();
    } finally {
        CURRENT_USER.remove();
    }
}
```

- **声明为 `private static final`**：`ThreadLocal` 实例本身只需要一个，声明为静态常量可以避免重复创建；同时由于它始终被强引用，键不会被回收，配合 `remove()` 即可彻底避免泄漏。
- **不要存放大对象**：每个线程都会持有一份副本，线程数越多占用越大。
- **在框架的拦截器或过滤器中统一清理**：例如在 Web 请求的 `Filter` 中设置上下文，并在 `finally` 中清理，避免业务代码遗漏。

# 6 InheritableThreadLocal 与线程池

`ThreadLocal` 中的值只属于当前线程，在当前线程中创建的子线程是拿不到的。`InheritableThreadLocal` 解决的就是这个问题：创建子线程时，`Thread` 的构造方法会把父线程 `inheritableThreadLocals` 中的值复制一份给子线程。

```java
private static final InheritableThreadLocal<String> TRACE_ID = new InheritableThreadLocal<>();

public static void main(String[] args) {
    TRACE_ID.set("trace-001");
    new Thread(() -> System.out.println(TRACE_ID.get())).start(); // 输出 trace-001
}
```

需要注意两点：

- 复制只发生在**子线程创建时**，之后父子线程各自修改互不影响；默认是浅复制，如有需要可以重写 `childValue()` 方法自定义复制逻辑。
- **在线程池中基本不可用**：线程池中的线程只在首次创建时继承一次，之后被不同任务复用，任务拿到的是“创建线程那一刻”的父线程值，而不是“提交任务时”的值。

线程池场景下的上下文传递，常见做法有：

- 提交任务时手动捕获上下文，在任务内部设置、执行完再清理（可以封装成 `Runnable` 包装器或 `TaskDecorator`）。
- 使用阿里开源的 TransmittableThreadLocal（TTL），它在任务提交时捕获上下文、执行时回放、结束后恢复。
- 在 JDK 25 及以上版本中，使用 `ScopedValue` 配合结构化并发（见第 8 章）。

# 7 与同步机制的对比

`ThreadLocal` 和线程同步机制都是为了解决多线程中同一变量的访问冲突问题，但思路截然不同。

在同步机制中，通过对象的锁机制保证同一时间只有一个线程访问变量。此时变量是多个线程共享的，使用同步机制要求程序缜密地分析什么时候对变量进行读写、什么时候需要锁定某个对象、什么时候释放对象锁等繁杂问题，程序设计和编写难度相对较大。

而 `ThreadLocal` 则从另一个角度来解决多线程的并发访问：为每个线程提供独立的变量副本，从根本上消除了共享。

概括起来说，对于多线程资源共享的问题，同步机制采用了“**以时间换空间**”的方式，而 `ThreadLocal` 采用了“**以空间换时间**”的方式。前者只提供一份变量，让不同的线程排队访问；后者为每个线程都提供一份变量，因此可以同时访问而互不影响。

不过要注意，二者并不能互相替代：

- `ThreadLocal` 适用于“每个线程各用各的”的场景，例如线程级的上下文、非线程安全对象的线程级复用。
- 如果多个线程确实需要读写**同一份**数据（例如计数器、库存、共享缓存），`ThreadLocal` 就无能为力了，仍然需要锁、原子类或并发容器。

# 8 新版本中的 ThreadLocal

## 8.1 版本演进

| 版本 | 变化 |
|------|------|
| JDK 1.2 | 引入 `ThreadLocal` 与 `InheritableThreadLocal` |
| JDK 5 | 支持泛型 `ThreadLocal<T>`，`get()` 不再需要强制类型转换；新增 `remove()` 方法 |
| JDK 8 | 新增 `withInitial(Supplier)` 工厂方法 |
| JDK 19 | 新增 `Thread.threadId()`，`Thread.getId()` 被标记为 `@Deprecated`；虚拟线程以预览特性引入 |
| JDK 21 | 虚拟线程正式发布（JEP 444），完整支持 `ThreadLocal`；`ScopedValue` 以预览特性引入（JEP 446） |
| JDK 25 | `ScopedValue` 正式发布（JEP 506） |

## 8.2 虚拟线程下的 ThreadLocal

虚拟线程完整支持 `ThreadLocal` 和 `InheritableThreadLocal`，每个虚拟线程都有自己独立的副本，与承载它的平台线程（载体线程）互不相干。但使用时要注意以下两点：

- **不要再用 `ThreadLocal` 做对象缓存**：在平台线程池中，“每个线程缓存一个昂贵对象”是一种有效的复用手段，因为线程数量有限且会被复用。而虚拟线程的设计理念是“每个任务一个线程、用完即弃”，数量可能达到百万级，`ThreadLocal` 缓存不仅起不到复用作用，还可能让内存占用成倍增长。
- **继承的开销**：通过 `Thread.Builder` 创建虚拟线程时，默认会继承父线程的 `InheritableThreadLocal`，可以用 `inheritInheritableThreadLocals(false)` 关闭。

## 8.3 ScopedValue

`ThreadLocal` 存在几个先天的设计问题：

- **可变**：任何代码都能随时调用 `set()`，数据流向难以追踪。
- **生命周期无界**：值会一直存在，直到调用 `remove()` 或线程结束，忘记清理就会泄漏或串用。
- **继承开销大**：`InheritableThreadLocal` 需要为每个子线程复制一份数据，在大量虚拟线程的场景下开销明显。

JDK 25 正式发布的 `ScopedValue`（作用域值）就是为了解决这些问题。它在一个有界的作用域内把值绑定到当前线程，作用域结束时自动解绑：

```java
private static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

public void handle(Request request) {
    ScopedValue.where(CURRENT_USER, request.getUser())
            .run(this::doBusiness); // run 执行期间可以读取到绑定的值
    // 离开 run 后自动解绑，无需手动清理
}

private void doBusiness() {
    User user = CURRENT_USER.get(); // 任意深度的调用链中都可以读取
    // ...
}
```

| 对比项 | `ThreadLocal` | `ScopedValue` |
|--------|---------------|---------------|
| 可变性 | 可以随时 `set()` | 绑定后不可修改，只能在内层作用域重新绑定 |
| 生命周期 | 直到 `remove()` 或线程结束 | 随作用域结束自动解绑 |
| 内存泄漏风险 | 有，需要手动 `remove()` | 无 |
| 子线程继承 | `InheritableThreadLocal` 需要复制数据 | 配合结构化并发 `StructuredTaskScope` 共享，无需复制 |
| 适用场景 | 需要在线程内修改的状态、线程级对象缓存 | 单向传递的上下文（用户、事务、追踪 ID 等） |

在 JDK 25 中，配合 `ScopedValue` 实现子任务继承的 `StructuredTaskScope` 仍是预览特性（JEP 505）。

对于“请求入口设置、调用链深处读取”这类单向传递上下文的场景，在 JDK 25 及以上版本中应优先考虑 `ScopedValue`；而 `ThreadLocal` 依然适用于需要在线程内反复修改状态的场景，并且在大量现有框架中会长期存在。

# 9 总结

- `ThreadLocal` 通过“每个线程持有一份变量副本”来避免共享，数据实际存放在 `Thread` 对象的 `threadLocals`（`ThreadLocalMap`）中，键为 `ThreadLocal` 实例，值为副本。
- `ThreadLocalMap` 的键是弱引用、值是强引用，在线程池中不调用 `remove()` 会导致内存泄漏和数据串用，因此必须在 `finally` 中调用 `remove()`。
- `InheritableThreadLocal` 只在子线程创建时复制一次，不适用于线程池，线程池场景可以使用任务包装器或 TransmittableThreadLocal。
- `ThreadLocal` 以空间换时间，适合线程级上下文和对象隔离，但不能替代同步机制解决真正的共享数据问题。
- 虚拟线程时代不宜再用 `ThreadLocal` 做对象缓存；JDK 25 及以上版本中，单向传递的上下文应优先使用 `ScopedValue`。
