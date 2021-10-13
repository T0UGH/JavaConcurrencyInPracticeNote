# Java并发编程实战 笔记

> 原书地址：https://book.douban.com/subject/10484692/

## Part 1 基础知识

### 第1章 简介

#### 1.1 并发简史

1. 早期计算机不包含操作系统,只能有一个程序,可以访问任何资源
2. 操作系统使得多进程出现,资源利用率上升
   1. 进程 拥有存储指令和数据的空间,串行执行,拥有指令存储器,是资源分配的基本单位
   2. 线程 共享进程资源,拥有独立的PC,栈和局部变量等,是调度的基本单位

#### 1.2 线程优势

1. 多处理器背景下,提高利用率,吞吐率
2. GUI里面提高响应(事件分发线程)
3. 建模简单化,将一个复杂任务分解为多个并行的任务组合
4. 异步处理

#### 1.3 线程带来的风险

1. 竞争条件(多个线程抢一个资源)
2. 活跃性(死锁,饥饿,活锁)
3. 性能(频繁切换导致开销过大)

#### 1.4 无处不在的线程

1. 框架通过在框架线程中调用应用程序代码将并发性引入到程序中.在代码中将不可避免的访问应用程序状态,因此所有访问这些状态的代码路径都必须是线程安全的
2. Timer类.TimerTask在Timer管理的线程中执行
3. Servlet(每个请求会包装成一个线程)
4. RMI(由RMI负责打包拆包远程对象)
5. Swing(具有异步性)

### 第2章 线程安全性



**要编写线程安全的代码，核心是要对状态访问操作进行管理**，特别是对共享的(shared)和可变的(Mutable)状态的访问。共享就是说变量可以被多个线程访问。



一个对象是否需要线程安全取决于它是否需要被多个线程访问。



要使得对象是线程安全的，需要采用**同步机制**来**协同**对对象可变状态的访问。



**面向对象技术更有利于写出线程安全的程序**，因为面向对象技术可以很好的控制外部代码对数据的访问（封装性）。



#### 2.1 什么是线程安全性



线程安全性：当多个线程访问某个类时，不管运行时环境采用何种调度方式或者这些线程将如何交替执行，并且在主调代码中**不需要任何额外的同步或协同**，这个类都能表现出正确的行为，我们就称这个类时线程安全的。



大多数Servlet是无状态的,**无状态就是线程安全的**：无状态不包含任何域,也不包含对其他域的引用,只有计算过程的局部变量，比如下面这个计算因式分解的程序

```java
public class StatelessFactorizer implements Servlet {
	public void service(ServletRequest req, ServletResponse resp) {
		BigInteger i = extractFromRequest(req);
		BigInteger[] factors = factor(i);
		encodeIntoResponse(resp, factors);
	}
}
```



#### 2.2 原子性



**Race Condition**(竞态条件)：由于不恰当的执行时序导致出现不正确的结果(最常见的就是先检查后执行)。换句话说，当某个计算的正确性依赖于多个线程的交替执行时序时，就会出现竞态条件。举个例子

```go
func testAndSet(int i) {
	i ++;
}
// 先读取了这个变量，然后进行写入，这不是个原子的操作
```



**“先检查后执行”式竞态条件的本质**：基于一种可能过期的失效观测结果来做出判断或者执行某种计算



#### 2.3 加锁机制

##### 2.3.1 内置锁

 每个对象都可以作为一把锁，这种锁称为**内置锁**或者监视器锁

````java
synchronized(lock){

}
// synchronized 关键字后的参数是一个对象，也就是把这个对象作为锁了
````





##### 2.3.2 重入锁



如果某个线程试图获取一个已经由它自己持有的锁，那么这个请求就会成功。“**重入**”意味着获取锁的粒度是“线程”，而不是“调用”。



 重入锁为锁(就是lock对象)关联一个持有者和计数值,持有者再次进入次数加一。



#### 2.4　用锁来保护状态



如果用同步来协调对某个变量的访问，那么能访问到这个变量的所有代码段都需要加锁，而且得加同一个锁。



一个**常用的加锁约定**是，将所有的可变状态封装在一个对象内部，并通过对象的内置锁对所有访问可变状态的代码路径进行同步，使得在该对象上不会发生并发访问。



如果需要让**多个变量的修改**具有原子性，需要**使用同一把锁**来保护所有的这些变量



vector

```java
if (!vector.contains(element)) {
	vector.add(element)
}
```

- 虽然sync保证了单个操作的原子性，但是如果要把多个操作合并为一个复合操作，还需要额外的加锁机制。



#### 2.5　活跃性与性能



加锁会影响性能，因此要保证sync代码段**只锁定需要同步的区域**，不要太粗糙。

另外由于获取和释放锁需要一定的开销，所以也**不要把sync代码段拆分的太过细致**。



### 第3章　对象的共享

本章将介绍如何**共享**和**发布**对象，从而使它们能够**安全地由多个线程共同访问**。

` synchronized`不仅可以保证原子性还有**内存可见性**。换句话说，我们不仅希望防止某个线程正在使用对象状态而另一个线程在同时修改该状态，而且希望确保当一个线程修改了对象状态后，其他线程能够看到状态被修改了。如果没有同步，这种情况就无法实现。

#### 3.1　可见性



通常，我们**无法确保执行读操作的线程能适时地看到其他线程写入的值**。为了确保多个线程之间对内存写入操作的可见性，必须使用同步机制。



在没有同步的情况下，编译器、处理器以及运行时等都可能对操作的执行顺序进行一些意想不到的调整（**重排序**），在缺乏足够同步的多线程程序中，要想对内存操作的执行顺序进行判断，几乎无法得出正确结论。



下面举个例子

```java
public class NoVisibility {
    private static boolean ready;
    private static int number;
    private static class ReaderThread extends Thread {
        public void run() {
            while (!ready)
                Thread.yield();
            System.out.println(number);
        }
    }
    public static void main(String[] args) {
        new ReaderThread().start();
        number = 42;
        ready = true;
    }
}
```

- 这个程序并不会按照想象中执行
- 因为会发生指令重排序，并且还有内存可见性等问题。所以`ReaderThread`完全有可能看到`ready==true`之后，还是看不到`number==42`



##### 3.1.1 失效数据

失效数据是指，当一个线程修改数据,但其他线程不能立马看到



同步可以解决失效数据的问题，通过加锁的方式，一个线程修改了数据，另一个线程100%能看到数据的修改

```java
public class SynchronizedInteger {
    private int value;
    public synchronized int get() {return value;}
    public synchronized void set(int value) {this.value = value};
}
```



##### 3.1.2 非原子的64位操作



**最低安全性**(失效值至少是由之前的线程设置,而不是随机值)



大多数变量都是最低安全性,除了**非volatile类型的64位操作**(double和long),因为jvm允许将64位操作拆解为2个 32位操作，所以有可能出现脏值



##### 3.1.3 加锁与可见性



内置锁保证可见性



当线程B执行由锁保护的同步代码块时，可以看到线程A之前在同一个同步代码块中的所有操作结果

![](https://healthlung.oss-cn-beijing.aliyuncs.com/image-20210922173930688.png)



> 加锁的含义不仅仅局限于互斥行为，还包括内存可见性。为了确保所有线程都能看到共享变量的最新值，所有执行读操作和写操作的线程都必须在同一个锁上同步



##### 3.1.4 Volatile变量



Java提供了一种稍弱的同步机制，即**volatile变量**，用来**确保将变量的更新操作通知到其他线程**。当把变量声明为volatile类型后，**编译器与运行时都会注意到这个变量是共享的**，因此**不会**将该变量上的操作与其他内存操作一起**重排序**。volatile变量**不会被缓存到寄存器**或者其他对处理器不可见的地方，因此读取volatile类型的变量时总是能返回最新写入的值。



当线程A首先写入一个volatile变量并且线程B随后读取这个变量时，**在写入volatile变量之前对A可见的所有变量的值，在B读取了volatile变量后，对B也是可见的**。



volatile变量通常用做某个操作完成、发生中断或者状态的**标志**，例子如下

```java
volatile boolean asleep;
...
while (!asleep)
    doSomething();
```



> **加锁机制可以保证可见性和原子性，而volatile变量只能保证可见性。**



当且仅当满足以下所有条件时,才应该使用volatile变量

- 对变量的写入操作不依赖变量的当前值或者只有单个线程更新值
- 该变量不会与其他状态变量一起纳入不变性条件
- 在访问变量时不需要加锁



#### 3.2 发布与逸出

**发布** 使对象能够在当前作用域之外的代码中使用，举个简单的例子，就是`public void get(){...}`，外部代码调用`get()`就能拿到这个对象。

**逸出** 当某个不应该发布的对象被发布



当且仅当对象的构造函数返回时，对象才处于可预测的和一致的状态，因此当从对象的构造函数中发布对象时，只是发布了一个尚未构造完成的对象。即使发布对象的语句位于构造函数的最后一行也是如此。如果this引用在构造过程中逸出，那么这种对象就被认为是**不正确构建**。

> **不要在构造函数中发布对象**



先举个错误的例子

```java
public class Monkey implements Animal {
    private boolean canFly;
    public Monkey(Zoo zoo) {
        zoo.registerAnimal(this)
        canFly = false;
    }
    @override
    public String buck() {
        return "duludulu" 
    }
}
```

- 还没构造完就把它注册出去了，也就是this逸出



然后再把它改成正确的

```java
public class Monkey implements Animal {
    private boolean canFly;
    public Monkey() {
        canFly = false;
    }
    @override
    public String buck() {
        return "duludulu" 
    }
    
    public static Monkey newInstance(Zoo zoo){
        Monkey m = new Mondey();
        zoo.registerAnimal(m);
        return m;
    }
}
```

- 这样可以确保在注册之前肯定是初始化好的



#### 3.3 线程封闭



一种避免使用同步的方式就是不共享数据，**如果仅在单线程访问数据,就不需要同步**，这种技术称为**线程封闭**



举个例子： `JDBC.Connection`对象 大多数请求都是单线程采用同步方式处理，并且在`Connection`对象返回之前，连接池不会再将它分配给其他线程，因此，这种连接管理模式在处理请求时隐含地将`Connection`对象封闭在线程中。



##### 3.3.1 Ad-hoc 线程封闭



Ad-hoc线程封闭是指**线程封闭的职责完全由程序实现来承担**(脆弱,少用)



##### 3.3.2 栈封闭



栈封闭：只能通过局部变量访问对象(Java基本类型或者局部变量)



**局部变量**的一个固有属性就是封闭在执行线程中，它们**位于执行线程的栈**中，其他线程无法访问这个栈。



```java
public int loadTheArk(Collection<Animal> candidates) {
    SortedSet<Animal> animals;
    int numPairs = 0;
    Animal candidate = null;
    animals = new TreeSet<Animal>();
    animals.addAll(candidates);
    for (Animal a: animals) {
        if(candidate == null || !candidate.isPotentialMate(a))
            candidate = a;
        else{
            ark.load(new AnimalPair(candidate, a));
            ++ numPairs;
            candidate = null;
        }
    }
    return numPairs;    
}
```

- 这段代码中，numPairs变量是存到执行线程的栈上的，其他线程绝对不可能访问到。
- 而animals变量的引用也是存到执行线程的栈上的，并且这个对象的引用只有这一个，所以其他线程也不可能访问到。



##### 3.3.3 ThreadLocal类

ThreadLocal类，可以**为每个使用该变量的线程存有一份独立的副本**（每个线程都有一个持有这个对象，线程之间不共享）。也可以理解为线程内部的全局变量。



 从概念上讲，可以将`ThreadLocal<T>`视为包含了`Map<Thread,T>` 对象,其中保存了特定于该线程的值,但ThreadLocal 的实现并非如此。这些特定于线程的值保存在Thread对象中，当线程终结后，这些值会GC。



[ThreadLocal的使用场景](https://cloud.tencent.com/developer/article/1636025)



但ThreadLocal变量类似于全局变量，它能**降低代码的可重用性**并在类之间**引入隐含的耦合性**。



#### 3.4 不变性



满足同步需求的一种方法使用**不可变对象**。



> 不可变对象一定是线程安全的



满足以下条件,对象才是不可变的

1. 创建以后状态不可修改
2. 所有域都是final
3. 对象被正确创建



下面构建一个不可变类

```java
public final class ThreeStooges {
    private final Set<String> stooges = new HashSet<String>();
    public ThreeStooges(){
        stooges.add("Moe");
        stooges.add("Larry");
        stooges.add("Curly");
    }
}
```



##### 3.4.1 final 域



在Java内存模型中，**final域还能确保初始化过程的安全性**，从而可以不受限制地访问不可变对象，并在共享这些对象时无须同步。



> 除非某个域是可变的，否则应将其声明为final域，这是一个良好的编程习惯



3.4.2 示例：使用volatile类型来发布不可变对象



在某些情况下，不可变对象能够提供一种弱形式的原子性。



每当需要对一组相关数据以原子方式执行某个操作的时候，可以考虑创建一个不可变里的类来包含这组数据

```java
class OneValueCache {
    private final BigInteger lastNumber;
    private final BigInteger[] lastFactors;
    public OneValueCache(BigInteger i, BigInteger[] factors) {
        lastNumber = i;
        lastFactors = factors;
    }
    public BigInteger[] getFactors(BigInteger i) {
        if (lastNumber == null || !lastNumber.equals(i))
            return null;
        else
            return Arrays.copyOf(lastFactors, lastFactors.length);
    }
}


public class CachedFactorizer implements Servlet {
    private volatile OneValueCache cache = new OneValueCache(null, null);
    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger factors = cache.getFactors(i);
        if (factors == null){
            factors = factor(i);
            cache = new OneValueCache(i, factors);
        }
        encodeIntoResponse(resp, factors);
    }
}
```

- 因为对象是不可变的，所以只能通过创建一个新的对象并赋值给引用的方式更新，这样就不会出现不完整的状态

- 当一个线程将volatile类型的cache设置为引用一个新的OneValueCache时，其他线程就会立刻看到新缓存的数据。

  



#### 3.5 安全发布

不正确的发布正确的对象被破坏(没有同步,失效值)

由于不可变对象很重要,Java内存模型为**不可变对象**的共享提供一种**特殊的初始化安全性保证**.

一个正确构造的对象可以通过以下方式安全发布

1. 静态初始化函数中初始化一个对象引用
2. 引用保存到volatile域或者AtomicReference对象中
3. 引用保存到某个正确构造对象的final域
4. 引用保存到锁保护的域(容器也可)



**事实不可变对象**：虽然从技术上来说这个对象可变，但是我承诺我不会改变它。



> 对象的发布需求取决于它的可变性
>
> - 不可变对象可以通过任何机制发布
> - 事实不可变对象必须通过安全方式发布
> - 可变对象必须通过安全方式来发布，并且必须是线程安全的或者由某个锁保护起来。



共享对象实用策略

1. **线程封闭**：对象只由一个线程拥有
2. **只读共享**：对象是共享的但不可变，包括不可变对象和事实不可变对象
3. **线程安全共享**：线程安全的对象在其内部实现同步，因此多个线程可以通过对象的公用接口来访问而不需要进一步的同步
4. **保护对象**：被保护的对象只能通过持有特定的锁来访问



### 第4章 对象的组合



我们希望**将一些现有的线程安全组件组合成更大规模的组件或程序**。本章将介绍一些**组合模式**，这些模式能够使一个类更容易成为线程安全的，并且在维护这些类时不会无意中破坏类的安全性保证



#### 4.1 设计线程安全的类



在设计线程安全类的过程中，常会包含以下三个基本要素 

1. 找出构成对象**状态**的所有**变量**。
2. 找出约束状态变量的**不变性条件**(变量的值不管如何改变，这些不变性条件不变)。
3. 建立对象状态的**并发访问管理策略(同步策略)**。
   - 同步策略规定了如何将不可变性、线程封闭与加锁机制等结合起来以维护线程的安全性，并且规定了哪些变量由哪些锁来保护



对象的状态分析，要从对象的域开始

- 如果域是**基本类型**，那些它是一个状态
- 如果域是**引用类型**，那么它所引用的对象里的所有域也是状态



##### 4.1.1 收集同步需求



**不可变条件**：变量的值不管如何改变，这些不变性条件不变（例如，计数器Counter必须是大于等于0的值）

**后验条件**：状态依赖于之前的状态（例如，如果Counter的当前状态为17，下一个状态）



由于**不变性条件**和**后验条件**在状态及状态转换上施加了各种约束，因此需要额外的**同步**与**封装**。

- 如果**某些状态是无效的**，那么必须对底层的状态变量进行**封装**，否则客户代码可能会使对象处于无效状态。
- 如果在某个操作中**存在无效的状态转换**，那么该**操作**必须是**原子**的。
- 另外，如果在类中**没有施加这种约束**，那么就应该**放宽**封装性或者序列化等需求，以获取更高的灵活性或性能。



在类中也可能包含同时约束多个状态变量的不变性条件（例如，NumberRange类同时有上界和下界，要**确保上界小于下界**）。

- 类似于这种包含多个变量的不变性条件会带来原子性需求，**这些相关的变量必须在单个原子操作中进行读取或更新**
- 如果在一个不变性条件中包含多个变量，那么在执行**任何访问相关变量的操作时，都必须持有保护这些变量的那个锁**



##### 4.1.2 依赖状态的操作



如果在某个操作中包含基于状态的先验条件，那么这个操作就是**依赖状态的操作**。（例如，在删除元素前，队列必须处于非空状态。从队列删除元素就是个依赖状态的操作）



在并发程序中，先验条件可能会由于其他线程执行的操作而变成真。在并发程序中要一直等到先验条件为真，然后再执行该操作。



要想实现某个**等待先验条件为真时**才**执行**的操作，一种更简单的方法是通过现有库中的类来实现依赖状态的行为。（例如阻塞队列[Blocking Queue]或信号量[Semaphore]）



##### 4.1.3 状态的所有权



略



#### 4.2 实例封闭



封装简化了线程安全类的实现过程 ，它提供了一种**实例封闭机制**。**当一个对象被封装到另一个对象中时，能够访问被封装对象的所有代码路径都是已知的。**通过将封装机制与合适的加锁策略结合，可以确保以线程安全的方式来使用非线程安全的对象。



> 将数据封装在对象内部，可以将数据的访问限制在对象的方法上，从而更容易确保线程在访问数据时总能持有正确的锁。



对象可以**封闭在类的一个实例中**（例如作为类的一个私有成员），或者**封装在某个作用域中**（例如作为一个局部变量），或者**封闭在一个线程内**（例如在某个线程中将对象从一个方法传递到另一个方法，而不是在多个线程之间共享该对象）。



类库提供了**包装器工厂方法**(例如**Collections.synchronizedList**)，可以**使得**这些**非线程安全的类**在**多线程**环境中**安全**地使用。这些工厂方法通过装饰器模式将容器类封装在一个同步的包装器对象中，而包装器能将接口中的每个方法都实现为同步方法，并将调用请求转发到底层的容器对象上。



##### 4.2.1 Java监视器模式



**监视器模式**：遵循Java监视器模式的对象会把对象的所有可变状态都封装起来，并由对象自己的**内置锁**来保护。



**例子，基于监视器模式的车辆追踪器**

````java
@ThreadSafe
public class MonitorVehicleTracker {
    @GuardedBy("this")
    private final Map<String, MutablePoint> locations;
    public MonitorVehicleTracker(Map<String, MutablePoint> locations) {
        this.locations = deepCopy(locations);
    }
    public synchronized Map<String, MutablePoint> getLocations() {
        return deepCopy(locations);
    }
    public synchronized MutablePoint getLocation(String id) {
        MutablePoint loc = locations.get(id);
        return loc == null ? null : new MutablePoint(loc);
    }
    public synchronized void setLocation(String id, int x, int y) {
        MutablePoint loc = locations.get(id);
        if (loc == null)
            throw new IllegalArgumentException("No such ID:" + id);
        loc.x = x;
        loc.y = y;
    }
    private static Map<String, MutablePoint> deepCopy(Map<String, MutablePoint> m) {
        Map<String, MutablePoint> result = new HashMap<String, MutablePoint>();
        for (String id: m.keySet()) {
            result.put(id, new MutablePoint(m.get(id)));
        }
        return Collections.unmodifiableMap(result);
    }
}

@NotThreadSafe
public class MutablePoint {
    public int x, y;
    public MutablePoint() {x = 0; y = 0;}
    public MutablePoint(MutablePoint p) {
        this.x = p.x;
        this.y = p.y;
    }
}
````



#### 4.3 线程安全性的委托



如果类中的**各个组件**都已经是线程**安全**的，那我们是否需要再增加一个额外的线程安全层？答案是“**视情况而定**”。



在下面这个例子中，组件线程安全则整体也线程安全。例子，基于委托的车辆追踪器

````java
@Immutable
public class Point {
    private final int x, y;
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
}

@ThreadSafe
public class DelegatingVehicleTracker {
    private final ConcurrentMap<String, Point> locations;
    private final Map<String, Point> unmodifiableMap;
    public DelegatingVehicleTracker(Map<String, Point> points) {
        locations = new ConcurrentHashMap<String, Point>(points);
        unmodifialeMap = Collections.unmodifialeMap(locations);
    }
    public Map<String, Point> getLocations() {
        return unmodifiableMap;
    }
    public Point getLocation<String id){
        return locations.get(id);
    }
    public void setLocation(String id, int x, int y) {
        if(locations.replace(id, newPoint(x, y) == null)) {
            throw new IllegalArgumentException("invalid vehicle name: " + id);
        }
    }
}
````



##### 4.3.2 独立的状态变量



我们可以将线程安全性委托给多个状态变量，前提是这些变量是彼此独立的。



##### 4.3.3 当委托失效时



如果多个状态变量之间存在**组合的不变性条件**，就不能简单地将线程安全性委托给多个安全的状态变量



例子，**线程不安全**的NumberRange

```java
public class NumberRange {
    private final AtomicInteger lower = new AtomicInteger(0);
    private final AtomicInteger upper = new AtomicInteger(0);
    public void setLower(int i) {
        // 这里有testAndSet类型的race condition
        if (i > upper.get())
            throw new IllegalArgumentException("xxx");
        lower,set(i);
    }
    public void setUpper(int i) {
        if (i < lower.get())
            throw new IllegalArgumentException("xxx");
		upper.set(i)
    }
    public boolean isInRange(int i) {
        return (i >= lower.get() && i <= upper.get());
    }
}
```

- 虽然AtomicInteger是线程安全的，但经过组合得到的类却不是。由于状态变量lower和upper不是彼此独立的，因此不能将NumberRange的线程安全性委托给它的两个状态变量。



##### 4.3.4 发布底层的状态变量



如果一个状态变量是线程安全的,并且没有任何不变型条件约束它,变量操作也没有不允许的状态,那么就可以安全发布该变量



#### 4.4 在现有的线程安全类中添加功能



Java类库中包含了许多有用的“基础模块”类，我们应该优先选择重用那些现有的类而不是创建新的类。但有的时候现有的类不能满足我们所有的需求。

例如，假设有一个线程安全的链表，它需要一个原子的“若没有则添加操作”(PutIfAbsent)。这个时候我们需要想办法实现这个操作。通常有三种方式，将在下面三小节阐述。



##### 4.4.1 通过继承来添加功能



最安全的添加方式是，直接修改原始的类的代码，但通常这都办不到

因此，我们可以**扩展这个类**，例如下面

```java
@ThreadSafe
public class BetterVector<E> extends Vector<E> {
    public synchronized boolean putIfAbsent(E x) {
        boolean absent = !contains(x);
        if (absent)
            add(x);
        return absent;
    }
}
```



但是扩展太**脆弱**，因为现在的**同步策略实现**被**分到多个单独维护的源代码文件**中了。此外，有些类不允许扩展，或者访问不到内部锁或内部数据。



##### 4.4.2  通过创建辅助类来添加功能



也可以将扩展代码放到一个辅助的类中去，但是要**确保**：**用的是同一把锁**



> Tips: 在Vector和同步封装器类的文档中指出，它们通过使用Vector或者封装器容器的内置锁来支持客户端加锁。



```java
@ThreadSafe
public class ListHelper<E> {
    public List<E> list = Collections.synchronizedList(new ArrayList<E>());
  	public boolean putIfAbsent(E x) {
        synchonized(list) {
            boolean absent = !list.contains(x);
            if(absent)
                list.add(x);
            return absent;
        }
    }
}
```



##### 4.4.3 通过组合来添加功能



```java
@ThreadSafe
public class ImprovedList<T> implements List<T> {
    private final List<T> list;
    public ImprovedList(List<T> list) {this.list = list;}
    public synchronized boolean putIfAbsent(T x) {
        boolean absent = !list.contains(x);
        if (absent)
            list.add(x);
        return absent;
    }
    public synchronized void clear() {list.clear();}
    // ... 按照类似的方式来委托List中的其他方法
}
```

- `ImprovedList`通过自身的内置锁添加了一层额外的加锁。它并不关心底层的`List`是否是线程安全的。额外的同步层可能会导致轻微的性能损失（很轻微，因为在**底层List上的同步不存在竞争**，速度很快。）



### 第5章 基础构建模块



Java平台类库包含了丰富的并发基础构建模块，例如线程安全的**容器类**以及各种用于协调多个相互协作的线程控制流的**同步工具类**(Synchronizer)



#### 5.1 同步容器类



同步容器包括：

- Vector和Hashtable
- 由Collections.synchronizedXxx等工厂方法创建的容器



##### 5.1.1 同步容器类的问题



在某些情况下同步容器需要额外的**客户端加锁**来**保护复合操作**，比如：迭代、跳转（根据指定顺序找到当前元素的下一个元素）、条件运算（例如，若没有则添加）等。



例如

```java
public static Object getLast(Vector list) {
    int lastIndex = list.size() - 1;
    return list.get(lastIndex);
}

public static void deleteLast(Vector list) {
    int lastIndex = list.size() - 1
    list.remove(lastIndex);
}

```

- 从方法调用者的角度来说，如果线程A在getLast()的同时，线程B删除了一个元素，那么getLast就会抛出`ArrayIndexOutOfBoundsException`异常。
- 如下图所示![](https://healthlung.oss-cn-beijing.aliyuncs.com/20211013195445.png)



解决方案：同步容器类通过其自身的锁来保护它的每个方法，通过获得容器类的锁，我们可以使`getLast`和`deleteLast`成为原子操作。

```java
public static Object getLast(Vector list) {
    synchronized(list){
    	int lastIndex = list.size() - 1;
    	return list.get(lastIndex);    
    }
}

public static void deleteLast(Vector list) {
    synchronized(list){
    	int lastIndex = list.size() - 1
    	list.remove(lastIndex);    
    }
}

```



##### 5.1.2 迭代器与`ConcurrentModificationException`



当一个线程对容器类进行迭代时，如果有另一个线程并发地修改容器，那么就会抛出一个`ConcurrentModificationException`，这是一种及时失败(fail-fast)机制。

- 它们采用的实现方式是，将计数器的变化与容器联系起来：如果在迭代过程中计数器被修改了，那么`hasNext`或`next`将抛出`ConcurrentModificationException`。



这种情况下，要么在迭代过程中**加锁**。要么“**克隆容器**”，并在副本上进行迭代。



##### 5.1.3 隐藏的迭代器



在某种情况下，迭代器会隐藏起来



> 容器的`hashCode`、`equals`、`toString`等方法会间接地执行迭代操作。同样，`containAll`、`removeAll`、`retainAll`等方法，以及把容器作为参数的构造函数，都会对容器进行迭代。所有这些间接地迭代操作都可能抛出`ConcurrentModificationException`。



#### 5.2 并发容器



并发容器包括

1. ConcurrentHashMap，它用来替代同步且基于散列的Map
2. CopyOnWriteArrayList，用于在**遍历为主要操作的情况**下代替同步的List
3. 在新的`ConcurrentMap`中添加了对一些常见复合操作的支持，例如“若没有则添加”、替换以及有条件删除。
4. ConcurrentLinkedQueue，它是线程安全的先进先出队列
5. BlockingQueue，它增加了**可阻塞的插入和获取操作**，如果队列为空，获取操作将一直阻塞知道队列中出现一个可用元素；如果队列为满，插入操作将一直阻塞知道出现可用空间。在“**生产者-消费者**”这种设计模式中，阻塞队列是非常有用的。
6. `CurrentSkipListMap`作为`SortedMap`的替代，`ConcurrentSkipListSet`作为`SortedSet`的替代



##### 5.2.1 ConcurrentHashMap



ConcurrentHashMap并不是将每个方法都在同一个锁上同步并使得每次只能有一个线程访问容器，

- 而是使用一种粒度更细的加锁机制来实现更大程度的共享，这种机制称为**分段锁**。
- 在这种机制下，任意数量的读取线程可以**并发地访问Map**，
- 执行**读取操作**的线程和执行**写入操作**的线程可以**并发地访问Map**，
- 并且**一定数量的写入线程**能够**并发地修改Map**



ConcurrentHashMap的迭代器不会抛出`ConcurrentModificationException`，因此不需要在迭代过程中加锁。它可以**容忍并发地修改**，当创建迭代器时会遍历已有的元素，并可以在迭代器被构建后将修改操作反映给容器。



但有一个TradeOff，这种容器的`size`和`isEmpty`只能返回一个估计值而不是准确值。



在ConcurrentHashMap中没有对Map进行加锁来提供独占访问



大多数情况下，用`ConcurrentHashMap`来替代同步的Map能进一步提升代码的可伸缩性。只有当应用程序需要加锁`Map`以进行独占访问时，才应该放弃使用`ConcurrentHashMap`。



##### 5.2.2 额外的原子Map操作



ConcurrentMap接口提供了一些常见的复合操作如下

```java
public interface ConcurrentMap<K,V> extends Map<K,V> {
    // 仅当K没有相应的映射值时才插入
    V putIfAbsent(K key, V value);
    // 仅当K被映射到V时才移除
    boolean remove(K key, V value);
    // 仅当K被映射到oldValue时才替换为newValue
    boolean replace(K key, V oldValue, V newValue);
    // 仅当K被映射为某个值时才替换为newValue
    V replace(K key, V newValue);
}
```



##### 5.2.3 CopyOnWriteArrayList

> https://segmentfault.com/a/1190000037678123

在每次修改时，都会创建并重新发布一个新的容器副本，从而实现可变性。

多个线程可以同时对容器进行迭代，而不会彼此干扰或者与修改容器的线程相互干扰



**数据一致性**

CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性。所以如果你希望写入的的数据，马上能读到，请不要使用CopyOnWrite容器



**内存消耗**

有数组拷贝自然有内存问题。如果实际应用数据比较多，而且比较大的情况下，占用内存会比较大，这个可以用ConcurrentHashMap来代替。



#### 5.3 阻塞队列、生产者消费者模式



阻塞队列提供了可阻塞的`put`和`take`方法，支持超时的`offer`和`poll`方法。



阻塞队列支持生产者消费者这种模式

- 该模式将“找出需要完成的工作”与“执行工作”这两个过程分离开来
- 将“生产数据的过程”与“使用数据的过程解耦开来”以简化工作负载的管理



类库中提供了多种BlockQueue的实现

- **LinkedBlockingQueue**，与LinkedList类似
- **ArrayBlockingQueue**，与ArrayList类似
- **PriorityBlockingQueue**，优先队列
- **SynchronousQueue**，它实际上不是个队列，它不会存储队列中的元素，但它维护一组线程。当工作到达时直接移交给线程。这种直接交付方式可以减少延迟。但是，因为SynchronousQueue没有存储功能，因此put和take会一直阻塞，直到有另一个线程已经准备好参与到交付过程中。



##### 5.3.1 举例



生产者

```java
public class FileCrawler implements Runnable {
    private final BlockingQueue<File> fileQueue;
    private final FileFilter fileFilter;
    private final File root;
    public void run() {
        try {
            crawl(root);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    public void crawl(File root) throws InterruptedException {
        File[] entries = root.listFiles(fileFilters);
        if (entries != null) {
            for (File entry: entries)
                if (entry.isDirectory())
                    crawl(entry);
            	else if(!alreadyIndexed(entry))
                    fileQueue.put(entry)
        }
    }
}
```



消费者

```java
public class Indexer implements Runnable {
    private final BlockingQueue<File> queue;
    public Indexer(BlockingQueue<File> queue) {
        this.queue = queue;
    }
    public void run() {
        try {
            while (true) {
                indexFile(queue.take());
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```



主程序

```java
public static void startIndexing(File[] roots) {
    BlockingQueue<File> queue = new LinkedBlockingQueue<File>(BOUND);
    FileFilter filter = new FileFilter () {
        public boolean accept(File file) {return true};
    }
    for (File root: roots)
        new Thread(new FileCrawler(queue, filter, root)).start();
    for (int i = 0; i < N_CONSUMERS; i ++) {
        new Thread(new Indexer(queue)).start();
    }
}
```

- 将文件遍历和建立索引这两个功能分解为了独立的操作
- 阻塞队列将负责所有的控制流调度，因此每个功能的代码都更加简单和清晰
- 但是，消费者线程永远不会推出，所以程序永远无法终止



##### 5.2.2 串行线程封闭



> **安全**地将**对象所有权**从一个线程**转移**到另一个线程



生产者消费者模式与阻塞队列一起，促成了串行线程封闭

1. **线程封闭对象**只能由单个线程拥有，但可以通过安全发布该对象来转移所有权
2. 在转移所有权后，也**只有另一个线程能获取这个对象的访问权限**，并且发布对象的线程不会再访问它
3. 这种安全的发布确保了对象状态对于新的所有者是可见的，并且由于最初的所有者不会再访问它，因此**对象将被封闭在新的线程**中
4. 新的所有者线程可以对该对象做任意的修改，因为它具有**独占的访问权**



##### 5.3.3 双端队列与工作密取



Deque是双端队列，实现了在队列头和队列尾的高效插入和删除，具体实现包括`ArrayDeque`和`LinkedBlockingDeque`



双端队列适合工作密取模式

- 每个消费者都有各自的双端队列
- 如果一个消费者完成了自己双端队列中的全部工作，那么它可以从其他消费者的双端队列末尾秘密地获取工作
- 工作密取适合即使消费者也是生产者的问题——当执行某个工作时可能导致出现更多的这类工作（例如， 网页爬虫程序中处理一个页面时，通常会发现有更多的页面需要处理）



#### 5.4 阻塞方法 与 中断方法



> https://www.liaoxuefeng.com/wiki/1252599548343744/1306580767211554



阻塞：线程可能会阻塞。当线程阻塞时，它通常被挂起，并处于某种阻塞状态(`BLOCKED`、`WAITING`或`TIMED_WAITING`)。



中断是一种协作机制，一个线程不能强制其他线程终止。当线程A中断B时，A仅仅是要求B在执行到某个可以终止的地方终止正在执行的任务——前提是B愿意停下来。



当某个方法抛出InterruptedException,说明该方法是阻塞方法并且接受到了外部的中断信号,会努力提前结束阻塞状态



#### 5.5 同步工具类



同步工具类可以是任何一个对象，只要它根据其自身的状态来协调线程的控制流。**阻塞队列**可以作为同步工具类，其他类型的同步工具类还包括**信号量(Semaphore)**、**栅栏(Barrier)**以及**闭锁(Latch)**。



所有的同步工具类都包含一些特定的结构化属性：它们**封装了一些状态**，这些**状态**将**决定**执行同步工具类的线程是**继续执行**还是**等待**



##### 5.5.1 闭锁Latch



**CountDownLatch**是闭锁的一种。

- 它可以使一个或多个线程**等待一组事件发生**。
- 闭锁状态包括一个**计数器**，该计数器被初始化为一个正数，表示需要等待的事件数量
- **countDown**方法递减计数器，表示有一个事件已经发生了
- **await**方法等待计数器达到零，这表示所有需要等待的事件都已经发生了
- 如果计数器的值非零，那么await会一直阻塞直到计数器为零，或者等待中的线程中断，或者等待超时。



例子

```java
public class TestHarness {
    public long timeTasks(int nThreads, final Runnable task) throws InterruptedException{
        final CountDownLatch startGate = new CountDownLatch(1);
        final CountDownLatch endGate = new CountDownLatch(nThreads);
        for(int i = 0; i < nThreads; i ++) {
            Thread t = new Thread(){
                public void run() {
                    startGate.await();
                    try {
                        task.run();
                    } finally {
                        endGate.countDown();
                    }
                }
            }
        }
        long start = System.nanoTime();
        startGate.countDown();
        endGate.await();
        long end = System.nanoTime();
        return end - start;
    }
}
```

- 启动门使得主线程能够同时释放所有工作线程
- 结束门使得主线程能够等待最后一个线程执行完成。



##### 5.5.2 FutureTask



FutureTask也能实现闭锁的效果。

- FutureTask实现了Future语义，表示一种抽象的可生成结果的计算
- 一个Task有三种状态：等待运行、运行中、运行完成
- `Future.get`的行为取决于任务的状态。如果任务已经完成，那么get会立刻返回结果，否则get将阻塞知道任务进入完成状态，然后返回结果或者抛出异常。



例子：`Preloader`使用`FutureTask`来执行一个高开销的计算，并且计算结果将在稍后使用。

```java
public class MyCallable implements Callable<String> {

    private long waitTime;

    public MyCallable(int timeInMillis){
        this.waitTime=timeInMillis;
    }
    @Override
    public String call() throws Exception {
        Thread.sleep(waitTime);
        //return the thread name executing this callable task
        return Thread.currentThread().getName();
    }

}

public class FutureTaskExample {

    public static void main(String[] args) {
        MyCallable callable1 = new MyCallable(1000);
        MyCallable callable2 = new MyCallable(2000);

        FutureTask<String> futureTask1 = new FutureTask<String>(callable1);
        FutureTask<String> futureTask2 = new FutureTask<String>(callable2);

        ExecutorService executor = Executors.newFixedThreadPool(2);
        executor.execute(futureTask1);
        executor.execute(futureTask2);

        while (true) {
            try {
                if(futureTask1.isDone() && futureTask2.isDone()){
                    System.out.println("Done");
                    //shut down executor service
                    executor.shutdown();
                    return;
                }

                if(!futureTask1.isDone()){
                //wait indefinitely for future task to complete
                System.out.println("FutureTask1 output="+futureTask1.get());
                }

                System.out.println("Waiting for FutureTask2 to complete");
                String s = futureTask2.get(200L, TimeUnit.MILLISECONDS);
                if(s !=null){
                    System.out.println("FutureTask2 output="+s);
                }
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }catch(TimeoutException e){
                //do nothing
            }
        }

    }

}
/*
FutureTask1 output=pool-1-thread-1
Waiting for FutureTask2 to complete
Waiting for FutureTask2 to complete
Waiting for FutureTask2 to complete
Waiting for FutureTask2 to complete
Waiting for FutureTask2 to complete
FutureTask2 output=pool-1-thread-2
Done
*/

```



##### 5.5.3 信号量



Semaphore中管理着一组虚拟的许可(permit)。在执行操作时可以首先获取许可（只要还有剩余的许可），并在使用以后释放许可。如果没有许可，那么`acquire`将阻塞直到有许可。



二值信号量(0,1)也叫做互斥体(mutex)，它具备不可重入的加锁语义。



Semaphore可以用来实现资源池。也可以使用Semaphore将任何一种容器变成有界的阻塞容器。



例子：Semaphore为容器设置边界

```java
public class BoundedHashSet<T> {
    private final Set<T> set;
    private final Semaphore sem;
    
    public BoundedHashSet(int bound) {
        this.set = Collections.synchronizedSet(new HashSet<T>());
        sem = new Semaphore(bound);
    }
    
    public boolean add(T o) throws InterruptedException {
    	sem.acquire();
        boolean wasAdded = false;
        try {
            wasAdded = set.add(o);
            return wasAdded;
        } finally {
            if (!wasAdded)
                sem.release();
        }
    }
    
    public boolean remove(Object o) {
        boolean wasRemoved = set.remove(o);
        if (wasRemoved)
            sem.release();
        return wasRemoved;
    }
}
```



##### 5.5.4 栅栏Barrier



栅栏也会阻塞一组线程直到某个事件发生，所有线程必须同时到达栅栏位置，才能继续执行。

> 例如几个家庭决定在某个地方集合：“所有人在6点到麦当劳碰头，到了以后要等其他人，之后再讨论下一步做什么”



`CyclicBarrier`可以使一定数量的参与方反复在栅栏位置汇合，在并行迭代算法中有用。

```java
public static void main(String[] args) { 
    CyclicBarrier cyclicBarrier = new CyclicBarrier(7,() ->{ 
        System.out.println("****召唤神龙"); 
    }); 
    for(int i = 1;i <= 7; i++){ 
        int finalI = i; 
        new Thread(() -> { 
                for(int i = 0; i < 10; i ++) {
                                System.out.println(Thread.currentThread()
                                   .getName() + "\t 收集到第"+ finalI +"颗龙珠"); 
                try { 
                    cyclicBarrier.await(); 
                } catch (InterruptedException e) { 
                    e.printStackTrace(); 
                } catch (BrokenBarrierException e) { 
                    e.printStackTrace(); 
                } 
            }
        },String.valueOf(i)).start(); 
    } 
} 
// 这个代码一共能召唤10次神龙
```



另一个形式的栅栏是`Exchanger`：它用于两方在栅栏交换数据。例如当一个线程向缓冲区写入数据，另一个线程向缓冲区读取数据。这些线程可以使用`Exchanger`来汇合，并将满的缓冲区与空的缓冲区交换。

>  例子看这里: https://blog.csdn.net/carson0408/article/details/79477280



#### 5.6 构建高效且可伸缩的缓存



```java
public interface Computable<A,V> {
    V compute(A arg) throws InterruptedException;
}


public class Memoizer<A, V> implements Computable<A, V> {
    private final ConcurrentMap<A, Future<V> cache = new ConcurrentHashMap<A, Future<V>>();
    private final Computable<A, V> c;
    public Memoizer(Computable<A, V> c) {
        this.c = c;
    }
    public V compute(final A arg) throws InterruptedException {
        while(true) {
            Future<V> f = cache.get(arg);
            if (f == null) {
                Callable<V> eval = new Callable<V>(){
                    public V call() throws InterruptedException {
                        return c.compute(arg);
                    }
                };
                FutureTask<V> ft = new FutureTask<V>(eval);
                // 原子操作，最终只会有一个arg对应的task被放进去
                f = cache.putIfAbsent(arg, ft);
                if (f == null) {
                    f = ft;
                    ft.run();
                }
            }
            try {
                return f.get();
            } catch (CancellationException) {
                cache.remove(arg, f);
            } catch (ExecutionException e) {
                throw launderThrowable(e.getCause());
            } 
        }
    }
}
```



#### Part 1 小结



可变状态越少，就越容易实现线程安全性

尽量把域声明为final类型，除非需要它们是可变的

不可变的对象一定是线程安全的

封装有助于管理复杂性

用锁来保护每一个可变变量

当保护同一个不可变条件中的所有变量时，要使用同一个锁

在执行复合操作期间，要持有锁

如果从多个线程中访问同一个可变变量时没有同步机制，那么程序会出现问题



## Part 2 结构化并发应用程序



### 第6章　任务执行



#### 6.1　在线程中执行任务



大多数服务器应用程序都提供了一种自然的任务边界选择方式：以独立的客户请求为边界。这既可以实现任务的独立性，又可以实现合理的任务规模。



##### 6.1.1 串行执行任务



串行的WEB服务器

```java
class SingleThreadServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while(true) {
            Socket connection = socket.accept();
            handleRequest(connection);
        }
    }
}
```

- 这种方式响应慢,服务器资源利用率低
- 在Web请求的处理中包含了一组不同的运算与I/O操作。服务器必须处理套接字I/O以读取请求和写回响应，这些操作通常会由于网络拥塞或连通性问题而被阻塞。此外，服务器还可能处理文件I/O或者数据库请求，这些请求同样会阻塞。
- 这样会导致服务器的资源利用率很低，因为当单线程在等待I/O操作完成时，CPU将处于空闲状态。





##### 6.1.2 显式地为任务创建线程



```java
class ThreadPerTaskServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while(true) {
            final Socket connection = socket.accept();
            Runnable task = new Runnable() {
                public void run() {
                    handleRequest(connection);
                }
            }
            new Thread(task).start();
        }
    }
}
```



显式为每个请求申请一个线程
1. 任务处理线程从主线程分离,**提高响应速度**
2. 任务可以**并行**处理,**提高吞吐量**
3. **任务处理代码必须是线程安全的**,多个线程会并发执行



##### 6.1.3 无限制创建线程的不足



线程生命周期的开销非常高：线程的创建和销毁有代价

资源消耗：活跃的线程会消耗系统资源，尤其是内存。

- 如果可运行的线程数量多于可用的处理器数量，那么有些线程会被闲置。
- 大量闲置的线程会占用许多内存，给垃圾回收器带来压力，
- 而且大量线程在竞争CPU资源时还将产生其他的性能开销。
- 如果你已经拥有足够多的线程使所有CPU保持忙碌状态，那么再创建更多的线程反而会降低性能。

稳定性：如果创建的线程多于平台限制，可能会抛出`OutOfMemoryError`异常。



#### 6.2　Executor框架



在Java类库中，任务执行的主要抽象不是`Thread`，而是`Executor`。线程池简化了线程管理工作，并且`java.util.concurrent`提供了一种灵活的线程池实现作为`Executor`框架的一部分。



```java
public interface Executor {
    void execute(Runnable command);
}
```



Executor为灵活且强大的异步任务执行框架提供了基础，

- 该框架能够支持多种不同类型的任务执行策略。
- 它提供了一种标准的方法将任务的**提交过程**与**执行过程** **解耦**开来，并用Runnable来表示任务。
- Executor的实现还提供了对生命周期的支持
- Executor基于生产者-消费者模式。如果要在程序中实现一个生产者消费者的设计，最简单的方式就是用Executor



##### 6.2.1 示例：基于Executor的Web服务器



```java
class TaskExecutionWebServer {
    private static final int NTHREADS = 100;
    private static final Executor exec = Executors.newFixedThreadPool(NTHREADS);
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while(true) {
            final Socket connection = socket.accept();
            Runnable task = new Runnable () {
                public void run() {
                    handleRequest(connection);
                }
            }
            exec.execute(task);
        }
    }
}
```



##### 6.2.2 执行策略



通过将任务的提交与执行解耦，从而无须太大的困难就可以为某种类型的任务指定和修改执行策略。在执行策略中定义了任务执行的“What、Where、When、How”

- What 在什么线程中执行任务,按什么顺序执行

- How Many 多少并发,多少等待

- Which 拒绝选择什么任务

- How 怎么通知任务成功失败



通过限制并发任务的数量，可以确保应用程序不会由于资源耗尽而失败

通过将任务的提交与任务的执行策略分离开来，有助于在部署阶段选择与可用硬件资源最匹配的执行策略



> 每当看见下面这种形式的代码时
>
> `new Thread(runnable).start()`
>
> 并且你希望获得一种更灵活的执行策略时，请考虑使用`Executor`来代替`Thread`



##### 6.2.3 线程池



线程池是管理一组同构工作线程的资源池



线程池的优势

1. 分摊在线程创建和销毁过程中产生的巨大开销
2. 不会由于等待创建线程而延迟任务的执行
3. 通过适当调整线程池的大小，可以创建足够多的线程以便使处理器保持忙碌状态，同时还可以防止过多线程相互竞争资源而使应用程序耗尽内存或失败。



类库中提供了一个灵活的线程池，可以通过调用`Executors`中的静态工厂方法之一来创建一个线程池

- `newFixedThreadPool`: 创建一个固定长度的线程池
- `newCachedThreadPool`: 创建一个可缓存的线程池，线程数量动态变化。
- `newSingleThreadExecutor`：单线程的`Executor`
- `newScheduledThreadPool`: 创建一个固定长度的线程池，而且以延迟或定时的方式来执行任务



线程池内部有一个**工作队列**，execute方法会将任务提交到工作队列，**工作线程反复地从工作队列中取出任务并且执行它们**。



##### 6.2.4 Executor的生命周期



**ExecutorService**扩展了Executor接口，添加了一组用于**生命周期管理**的方法

```java
public interface ExecutorService extends Executor {
    void shutdown();
    List<Runnable> shutdownNow();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedExcepion;
}
```

Executor 生命周期有三种状态 

- **运行**  对象新建时就是运行状态

- **关闭**  
  - shutdown: 不接受新任务,同时等待已有任务完成,包括未执行的任务,关闭后任务再提交由 拒绝执行处理器 处理或者直接抛异常
  - shutdownNow: 取消所有运行中的任务，并且不再启动Queue里尚未开始执行的任务

- **终止**  关闭后任务完成
  - 可以调用`awaitTermination`来等待`ExecutorService`到达终止状态，或者调用`isTerminated`来轮询`ExecutorService`是否已经终止。



例子，下面的代码通过增加生命周期支持来扩展Web服务器的功能

```java
class LifecycleWebServer {
    private final ExecutorService exec = ...;
    public void start() throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (! exec.isShutdown()) {
            try {
                final Socket conn = socket.accept();
                exec.execute(new Runnable(){
                   public void run(){handleRequest(conn);} 
                });
            } catch (RejectedExecutionException e) {
                if(!exec.isShutdown())
                    log("task submission rejected", e)
            }
        }
    }
    // 调用这个方法可以shutdown服务器
    public void stop() {exec.shutdown();}
    // 发送某种特定的req也可以shutdown服务器
    void handleRequest(Socket conn) {
        Request req = readRequest(conn);
        if (isShutdownRequest(req))
            stop();
        else
            dispatchRequest(req);
    }
}
```



##### 6.2.5 延迟任务和周期任务



`Timer`类负责管理延迟任务（在100ms后执行此任务）和周期任务（每10ms执行一次该任务）。

`Timer`的缺陷

- **只用一个线程执行所有定时任务**,假如某个任务耗时过长,会影响其他任务的定时准确性.
- 除此之外,**不支持抛出异常**,发生异常将终止线程(已调度未执行线程不会执行,称为线程泄露)



建议使用`ScheduledThreadPoolExecutor`来代替`Timer`。

此外，如果要构建自己的定时调度服务，可以使用`DelayQueue`作为辅助。DelayQueue是一个无界的BlockingQueue，用于放置实现了Delayed接口的对象，其中的对象只能在其到期时才能从队列中取走。这种队列是有序的，即队头对象的延迟到期时间最长。
> https://www.cnblogs.com/myseries/p/10944211.html

#### 6.3 找出可利用的并行性



即使是服务器程序，在单个客户请求中仍可能存在可发掘的并行性。本节我们将开发不同版本的页面渲染功能，它作为浏览器的一个组件，用于将HTML界面绘制到图像缓存中。

##### 6.3.1 示例：串行的页面渲染器

```java
public class SingleThreadRender {
	void renderPage(CharSequence source){
		renderText(source);
		List<ImageData> imageData = new ArrayList<ImageData>();
		for (ImageInfo imageInfo : scanForImageInfo(source)) {
			imageData.add(imageInfo.downloadImage());
		}
		for (ImageData data: imageData)
			renderImage(data);
	}
}
```
- 这个程序先绘制文本元素，在处理完一遍文本后，再下载图像，然后绘制图像
- 图像下载过程是串行的，而且都是在等待IO操作执行完成，在这期间CPU几乎不做任何工作，这种串行执行没有充分地利用CPU

##### 6.3.2  Callable与Future

Runnable接口有很大的局限性，`run`方法不能返回一个值或者抛出一个异常

而Callable接口的`call`方法能够返回一个值或者抛出一个异常。

```java
public interface Callable<V> {
	V call() throws Exception;
}
```

Executor中执行的任务有4个生命周期阶段：创建、提交、开始、完成。
- 已提交但是尚未开始的任务可以取消
- 已开始的任务，只有它们能响应中断时，才能取消

Future可以表示一个任务的生命周期，
- 它提供了相应的方法来判断是否完成或者取消，
- 还可以获取任务的结果和取消任务

```java
public interface Future<V> {
	boolean cancel(boolean mayInterruptIfRunning)
	boolean isCanceled();
	boolean isDone();
	V get() throws InterruptedException, ExecutionException, CancellationException
	V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, CancellationException, TimeoutException;
}
```

ExecutorService中的所有submit方法都将返回一个`Future`，从而将`Runnable`或`Callable`提交给`Executor`。

在将Runnable或Callable提交到Executor的过程中，包含了一个安全发布过程，即将Runnable或Callable从提交线程发布到最终执行任务的线程。类似地，在设置Future结果的过程中也包含 一个安全发布，即将这个结果从计算它的线程发布到任何通过get获得它的线程。

##### 6.3.3 示例：使用Future实现页面渲染器

```java
public class FutureRender {
	private final ExecutorService executor = ...;
	void renderPage(CharSequence source) {
		final List<ImageInfo> imageInfos = scanForImageInfo(source);
		Callable<List<ImageData>> task 
			= new Callable<List<ImageData>>(){
			for (ImageInfo iamageInfo: imageInfos)
				result.add(imageInfo.downloadImage());
			}
		};
		Future<List<ImageData>> future = executor.submit(task);
		renderText(source);
		try {
			List<ImageData> imageDatas = future.get();
			for (ImageData data: imageDatas)
				renderIamge(data);
		} catch (InterruptedException e) {
			// 重新设置线程的中断状态
			Thread.currentThread.interrupt();
			// 由于不需要结果，因此取消任务
			future.cancel(true);
		} catch (ExecutionException e) {
			throw launderThrowable(e.getCause());
		}
	}
}
```
- 渲染过程被分解为两个任务，一个是渲染所有的文本（CPU密集型），另一个是下载所有的图片（IO密集型）
- 上面的程序中创建了一个Callable来下载所有图像，并将其提交到一个`ExecutorService`。这将返回一个描述任务执行情况的`Future`。
- 当主任务需要图像时，它会等待`Future.get()`的调用结果。

##### 6.3.4 在异构任务并行化中存在的局限

只有当大量**相互独立**且**同构**的任务可以**并发进行处理**时，才能体现出将程序的工作负载分配到多个任务中带来的性能提升。

##### 6.3.5 CompletionService

CompletionService将Executor和BlockingQueue的功能融合在一起。你可以将Callable任务提交给它执行，然后使用类似队列操作的take和poll等方法，按照任务执行速度的快慢来获取已完成的结果，这些结果被封装在一个Future中

##### 6.3.6 使用CompletionService来实现页面渲染器

```java
public class Renderer {
	private final ExecutorService executor;
	Render(ExecutorService executor) {this.executor = executor;}
	void renderPage(CharSequence source) {
		List<ImageInfo> info = scanForImageInfo(source);
		CompletionService<ImageData> completionService = new ExecutorCompletionService<ImageData>(executor);
		for (final ImageData imageData: info) {
			completionService.submit(new Callable<ImageData>(){
				public ImageData call() {
					return imageInfo.downloadImage();
				}
			});
		}
		renderText(source);
		try {
			for (int t = 0, n = info.size(); t < n; t ++) {
				// 每获取到一个结果就会render一次
				Future<ImageData> f = completionService.take();
				ImageData imageData = f.get();
				renderImage(imageData);
			}
		} catch (InterruptedException e) {
			Thread.currentThread().interrupt();
		} catch (ExecutionException e)
			throw launderThrowable(e.getCause());
	}
}
```
- CompletionService就相当于一组计算的句柄，这与Future是单个计算的句柄类似

##### 6.3.7 为任务设置时限

如果某个任务无法在指定时间内完成，那么将不再需要它的结果。Future.get可以指定一个超时时间参数，如果在指定的时间中没有计算出结果就抛出一个`TimeoutException`。

下面这个例子假设我们的页面需要从广告服务器获取一个广告，如果超时就显示默认广告
```java
Page renderWithAd() throws InterruptedException {
	long endNanos = System.nanoTime() + TIME_BUDGET;
	Future<Ad> f = exec.submit(new FetchAdTask());
	Page page = renderPageBody();
	Ad ad;
	try {
		long timeLeft = endNanos - System.nanoTime();
		ad = f.get(timeLeft, NANOSECONDS); // 带有超时标识
	} catch (ExecutionException e) {
		ad = DEFAULT_AD;
	} catch (TimeoutException e) { // 处理超时
		ad = DEFAULT_AD;
		f.cancel(true);
	}
	page.setAd(ad);
	return page;
}
```

##### 6.3.8 invokeAll

当你需要并发执行一组任务，并统一获得返回结果时，可以使用`invokeAll`
- 这个方法会阻塞，必须等待所有的任务执行完成后统一返回

下面举个从不同航空公司获取机票价格的例子，其中如果某个公司的服务器响应超时就不管它的价格了
```java
private class QuoteTask implements Callable<TravelQuote> {
	private final TravelCompany company;
	private final TravelInfo travelInfo;
	...
	public TravelQuote call() throws Exception {
		return company.solicitQuote(travelInfo);
	}
}
public List<TravelQuote> getRandedTravelQuotes (TravelInfo travelInfo, Set<TravelCompany> campanies, Comparator<TravelQuote> ranking, long time, TimeUnit unit) throws InterruptedException {
	List<QuoteTask> tasks = new ArrayList<QuoteTask>();
	for (TravelCompany company: companies)
		tasks.add(new QuoteTask(company, travelInfo));
	List<Future<TravelQuote>> futures = exec.invokeAll(tasks, time, unit);
    List<TravelQuote> quotes = new ArrayList<TravelQuote>();
    Iterator<QuoteTask> taskIter = tasks.iterator();
    for (Future<TravelQuote> f: futures) {
    	QuoteTask task = taskIter.next();
    	try {
    		quotes.add(f.get());
    	} catch (ExecutionException e) {
    		quotes.add(task.getFailureQuote);
    	} catch (CancellationException e) {
    		quotes.add(task.getTimeoutQuote(e));
    	}
    }
    Collections.sort(quotes, ranking);
    return quotes;
}
```


### 第7章 取消和关闭


Java没有提供任何机制来安全地终止线程。但它提供了**中断**(Interruption)，这是一种协作机制，能够**使一个线程通知另一个线程终止当前工作**。

因为任务本身的代码肯定比发出取消请求的代码更清楚如何执行清除工作以停止任务，所以通过中断机制，让任务本身做一些清除工作然后再退出是更好的方式


#### 7.1 任务取消

如果**外部代码**能够在某个操作**正常完成之前将其置入“完成”状态**，那么这个操作可以称为**可取消的**



例如下面这些操作都是可取消的

- 用户请求取消：用户点击界面上的取消按钮
- 有时间限制的操作：任务需要在有限的事件内完成
- 应用程序事件：当其中一个任务找到了最佳解决方案时，所有其他仍在寻找方案的任务应该被取消
- 错误：当磁盘满了时，所有任务都需要被取消
- 关闭：当程序关闭时，必须对正在处理和还没处理的工作执行某种操作，这些工作有可能被取消也可能会等它们完成之后程序再关闭



最简单的一种协作机制是手动设置某个“已请求取消”标志，而任务将定期地查看这个标志，如果这个标志被设置为`true`，任务将提前结束。例子如下

````java
@ThreadSafe
public class PrimeGenerator implements Runnable {
	@GuardedBy("this")
	private final List<BigInteger> primes = new ArrayList<BigInteger>();
	private volatile boolean cancelled; // “已请求取消”标志
	public void run(){
		BigInteger p = BigInteger.ONE;
		while (!cancelled) {
			p = p.nextProbablePrime();
			synchronized(this) {
				primes.add(p);
			}
		}
	}
	public void cancel() {cancelled = true;}
	public synchronized List<BigInteger> get() {
		return new ArrayList<BigInteger>(primes);
	}
}

// 如何使用这个PrimeGenerator如下
List<BigInteger> aSecondOfPrimes() throws InterruptedException {
    PrimeGenerator generator = new PrimeGenerator();
    new Thread(generator).start();
    try {
        SECONDS.sleep(1);
    } finally {
        generator.cancel(); // 放到finally里确保一定能cancel
    }
    return generator.get();
}
````
- cancel方法由finally块调用，从而确保即使在调用sleep时被中断也能取消素数生成器的执行
- 如果cancel没有被调用，那么搜索素数的线程将永远运行下去，不断消耗CPU的时钟周期，并使得JVM不能正常退出



一个可取消的任务必须拥有取消策略，包括：How、When、What

- How: 其他代码如何请求取消该任务
- When: 任务何时检查是否已经请求了取消
- What: 在响应取消请求时应该执行哪些操作



以PrimeGenerator为例

- How: 客户代码通过调用cancel来取消
- When：PrimeGenerator在每次搜索素数前首先检查是否存在取消请求，如果存在就取消
- What：略



##### 7.1.1 中断



上面设置标志的方法有个缺点：当线程处于阻塞状态，就无法检查标志位。换句话说，标志位无法停止线程的阻塞状态。这时候我们需要使用中断



每个线程都有一个`boolean`类型的中断状态。当中断线程时，这个线程的中断状态将被设置为`true`。在`Thread`API中包含了中断线程以及查询线程中断状态的方法，如下所示

```java
public class Thread {
    public void interrupt(){...}
    public boolean isInterrupted(){...}
    public static boolean interrupted(){...}
}
```

- `interrupt`方法能中断目标线程或者将目标线程从中断中恢复
- `isInterrupted`方法能返回目标线程的中断状态
- 静态的`interruped`将清除当前线程的中断状态



Java库中的阻塞方法，例如`Thread.sleep`、`Object.wait`等，都会检查线程何时中断，并且在发生中断时提前返回，它们会干两件事

1. 清除中断状态
2. 抛出InterruptedException异常来表示阻塞操作由于中断而提前结束。



而当线程在**非阻塞状态**下**中断**，它的**中断状态会被设置为true**，然后需要在用户的**任务代码**的合适位置**检查中断**，并做出一些处理。因此，**调用interrupt并不意味着立刻停止目标线程正在进行的工作，而只是传递了请求中断的消息。**



> 对中断操作的正确理解是：它并不会真正地中断一个正在运行的线程，而只是发出中断请求，然后由线程在下一个合适的时刻中断自己



下面的代码通过中断来取消PrimeGenerator的执行

```java
class PrimeProducer extends Thread {
    private final BlockingQueue<BigInteger> queue;
    PrimeProducer(BlocingQueue<BigInteger> queue) {
        this.queue = queue;
    }
    public void run() {
        try {
            BigInteger p = BigInteger.ONE;
            while (!Thread.currentThread().isInterrupted())
                queue.put(p = p.nextProbablePrime())
        } catch (InterruptedException consumed) {
            // 收到InterruptedException直接退出就行
        }
    }
    public void cancel() {interrupt()};
}
```

- 有两个位置可以检查出中断
  1. 在阻塞的put方法调用，这个方法会检查中断，重置中断状态并且抛出异常
  2. 在循环时手动检查中断



##### 7.1.2 中断策略



线程所有者是指创建这个线程的类



线程应该包含中断策略，它规定了线程如何解释中断：

- 当发生中断请求时，应该做哪些工作
- 哪些工作单元对于中断来说是原子操作
- 以多快的速度来响应中断



对于非线程所有者的代码来说，除了要对中断响应，还需要小心地保存并传递线程的中断状态，给调用栈上方的线程所有者代码。例如：对于阻塞的`put`方法来说，它重置了中断状态，但是它抛出了异常，方便上层的线程所有者代码对中断进行进一步处理。如果它不抛出异常，那么上层代码就感知不到这个中断了。



> 对任务来说，最合理的中断处理策略是：尽快退出执行流程，并把中断信息传递给调用者，从而使调用栈中的上层代码可以采取进一步的操作。



> 由于每个线程都有自己的中断策略，因此除非你知道中断对于该线程的含义，否则就不应该调用`interrupt`来中断它
>
> 线程只能由它的所有者中断，所有者可以将线程的中断策略信息封装在某个合适的取消机制中，例如`ThreadPool`将线程的中断封装到了`shutdown`和`shutdownNow`中。（所以客户代码不要直接调用线程的`interrupt`而是应该调用线程池的`shutdown`）



##### 7.1.3 响应中断



任务代码有两种实用策略来处理InterruptedException

- 传递异常：最简单的方式是直接`throws`抛出这个异常，让上层代码继续处理。这样你的方法也变成了可中断的阻塞方法
- 恢复中断状态：`Thread.currentThread().interrupt()`，重新设置中断状态，从而让上层代码能够感知到发生了中断。



此外，对于那些不支持取消，但是仍然要在代码中调用可中断阻塞方法的操作，它们可以写成下面这样

```java
public Task getNextTask(BlockingQueue<Task> queue) {
    boolean interrupted = false;
    try {
        while (true) {
            try {
                return queue.take();
            } catch (InterruptedException e) {
                interrupted = true;
            }
        }
    } finally {
        if (interrupted)
            Thread.currentThread().interrupt();
    }
}
```

- 这个代码保证，即使发生了`InterruptedException`也可以`queue.take()`到值
- 并且它重置了中断位，向上层代码传递了发生中断这件事，它没有屏蔽中断



> 只有实现了线程中断策略的线程所有者代码才可以屏蔽中断请求。在任务代码或者库代码中都不应该屏蔽中断请求。



##### 7.1.4 示例：计时运行



第一版代码

````java
private static final ScheduledExecutorService cancel Exec = ...;
public static void timedRun(Runnable r, long timeout, TimeUnit, unit) {
    final Thread taskThread = Thread.currentThread();
    cancelExec.schedule(new Runnable(){
        public void run() {taskThread.interrupt();}
    }, timeout, unit);
    r.run();
}
````

- 这个代码用另一个线程(cancelExec)来中断任务代码所在的线程
- 这会带来问题：
  - 因为timedRun可以从任意一个线程中调用，因此它无法知道调用方线程的中断策略。如果任务提前完成，那么interrupt()就中断不到当前任务了，有可能把调用方线程执行的下个任务给中断掉，这样就出了问题了。



第二版代码能够改掉这个问题

```java
public static void timedRun(final Runnable r, long timeout, TimeUnit unit) throws InterruptedException {
	class RethrowableTask implements Runnable {
		private volatile Throwable t;
		public void run() {
			try {
				r.run();
			} catch (Throwable t) {
				this.t = t;
			}
		}
		void rethrow() {
			if (t != null)
				throw launderThrowable(t);
		}
	}
	RethrowableTask task = new RethrowableTask();
	final Thread taskThread = new Thread(task);
	taskThread.start();
	cancelExec.schedule(new Runnable(){
		public void run() {
			taskThread.interrupt();
		}
	}, timeout, unit);
	taskThread.join(unit.toMillis(timeout));
	task.rethrow();
}
```

- 在taskThread中执行task
- 在cancelThread中设置中断
- 在调用者线程中用join来等待taskThread完成
- 这样，由于taskThread就是我创建的，我知道这个线程就算提前完成后再收到中断也不会有负面影响
- 并且使用join来等待task完成，这样就能拿到task上的异常。
- 但是这版代码还是有个问题，join方法无法知道线程是正常执行完任务后退出了还是因为join超时退出了



> 注意：`taskThread.join(unit.toMillis(timeout));`，不会中断taskThread，而是从当前线程调用join开始计时，1000毫秒若taskThread线程还没有结束，当前线程就继续往下执行



##### 7.1.5 通过Future来实现取消



Future拥有一个cancel方法，该方法带有一个`boolean`类型的参数`mayInterruptIfRunning`

- 如果mayInterruptIfRunning设置为true并且任务当前正在某个线程中运行，那么这个线程将被中断
- 如果mayInterruptIfRunning设置为false，那么意味着如果这个任务还没有启动，就不要再启动它运行它了。这种情况适合那些不支持中断的任务。
- 如果任务在标准`Executor`中运行，那么在取消任务时，应该设置`mayInterruptIfRunning`为`true`
- 当尝试取消任务时，不应该直接调用线程池中`thread`上的`interrupt`，而应该调用任务`future`的`cancel`



下面看第三版timedRun

```java
public static void timedRun(Runnable r, long timeout, TimeUnit unit) throws InterruptedException {
    Future<?> task = taskExec.submit(r);
    try {
        task.get(timeout, unit);
    } catch (TimeoutException e) {
        //超时则直接退出，这样任务会被取消
    } catch (ExecutionException e) {
        //如果任务中抛出了异常(任务应该已经停止执行了)，则继续抛出异常
        throw lauderThrowable(e.getCause());
    } finally {
        task.cancel(true);
    }
}
```



> 当`Future.get`抛出`InterruptedException`或`TimeoutException`时，如果你确定不再需要任务的结果，那么可以调用`Future.cancel`来取消任务。



##### 7.1.6 处理不可中断的阻塞



在Java库中，并不是所有的可阻塞方法或者阻塞机制都能响应中断，下面这些方法都不能响应中断

- **Java.io包中的同步Socket I/O**：在服务器程序中最常见的阻塞I/O形式就是对套接字的读取和写入。虽然`InputStream`和`OutputStream`的`read`和`write`等方法都不会响应中断，但通过**关闭底层的套接字**，可以使得由于执行`read`或`write`等方法而被阻塞的线程抛出`SocketException`
- Java.io包中的同步I/O
- Selector的异步I/O: 如果一个线程在调用`Selector.select`方法时阻塞了，那可调用`close`或`wakeup`方法会使线程抛出`ClosedSelectorException`并提前返回
- 获取某个锁：如果一个线程**由于等待某个内置锁而阻塞，那么将无法响应中断**，因为线程认为它肯定会获得锁，所以不会理会中断请求。但是，`Lock`类中提供了`lockInterruptily`方法，该方法允许在等待一个锁的同时仍能响应中断。



下面的例子实现了一个`ReaderThread`，它可以实现中断`Socket`的`read`

````java
public class ReaderThread extends Thread {
    private final Socket socket;
    private final InputStream in;
    
    public ReaderThread(Socket socket) throws IOException {
        this.socket = socket;
        this.in = socket.getInputStream();
    }
    
    // 重写了interrupt
    public void interrupt() {
        try {
            socket.close();
        }
        catch (IOException ignored) {}
        finally {
            super.interrupt();
        }
    }
    
    public void run () {
        try {
            byte[] buf = new byte[BUFSZ];
            while (true) {
                int count = in.read(buf);
                if (count < 0)
                    break;
                else if (count > 0)
                    processBuffer(buf, count);
            }
        } catch (IOException e) {}
    }
}
````





````java
class BrokenPrimeProducer extends Thread {
	private final BlockingQueue<BigInteger> queue;
	private volatile boolean cancelled = false;
	BrokenPrimeProducer(BlockingQueue<BigInteger> queue) {
		this.queue = queue;
	}
	public void run() {
		try {
			BigInteger p = BigInteger.ONE;
			while(!cancelled)
				queue.put(p = p.nextProbablePrime());
		} catch (InterruptedException consumed) {}
	}
	public void cancel() { cancelled = true; }
}

// 消费者
void consumePrimes() throws InterruptedException {
	BlockingQueue<BigInteger> primes = ...;
	BrokenPrimeProducer producer = new BrokenPrimeProducer(primes);
	producer.start();
	try {
		while(needMorePrimes())
			consume(primes.take())
	} finally {
		producer.cancel();
	}
}
````



#### 7.2 停止基于线程的服务



除非拥有某个线程，否则不要对该线程进行直接操控。



线程池是其工作者线程的所有者，因此要中断工作者线程，应该使用线程池提供的方法，而不是直接调用工作者线程的`interrupt`。



类似线程池这样的服务应该提供生命周期方法来关闭它自己以及它所拥有的线程。这样当应用程序关闭服务时，服务就能关闭所有的线程了。例如，在`ExecutorService`中就提供了`shutdown`和`shutdownNow`。



##### 7.2.1 示例：日志服务



先给出一种简单的LogWriter实现

```java
public class LogWriter {
    private final BlockingQueue<String> queue;
    private final LoggerThread logger;
    
    public LogWriter(Writer writer) {
        this.queue = new LinkedBlockingQueue<String>(CAPACITY);
        this.logger = new LoggerThread(writer);
    }
    public void start() {logger.start();}
    public void log(String msg) throws InterruptedException {
        queue.put(msg);
    }
    public void close() {
        logger.interrupt();
    }
    private class LoggerThread extends Thread {
        private final PrintWriter writer;
        ...
        public void run() {
            try {
                while (!isInterruped())
                    writer.println(queue.take());
            } catch(InterruptedException ignored) {
                
            } finally {
                writer.close();
            }
        }    
    }
}
```

- 这样，当收到中断时，Writer就会退出
- 但是这会出现一个情况，直接退出会丢失那些正在等待被写入到日志中的消息，并且更严重的是，日志生产者们将在调用log时被阻塞，而日志队列是满的（因为消费者不消费了），因此这些线程将无法解除这些阻塞！！！



下面给出第二版代码，它将解决上述问题

```java
public class LogService {
	private final BlockingQueue<String> queue;
	private final LoggerThread loggerThread;
	private final PrintWriter writer;
	@GuardedBy("this") private boolean isShutdown;
	@GuardedBy("this") private int reservations;
	public void start () {
		loggerThread.start();
	}
	public void stop() {
		synchronized(this) {
			isShutdown = true;
		}
		loggerThread.interrupt();
	}
	public void log(String msg) throws InterruptedException {
		synchronized(this) {
			if (isShutdown)
				throw new IllegalStateException(...);
			++reservations;	
		}
		queue.put(msg);
	}
	private class LoggerThread extends Thread {
		public void run() {
			try {
				while (true) {
					try {
						synchronized (LogService.this) {
							if (isShutdown && reservations == 0)
								break;
						}
						String msg = queue.take();
						synchronized(LogService.this) {
							reservation --;
						}
						writer.println(msg);
					} catch (InterruptedException e) { //重试}
				}
			} finally {
				writer.close();
			}
		}
	}
}
```
- 此方法通过原子的方式去检查关闭请求，并且有条件地递增一个计数器来“保持”提交消息的权利

##### 7.2.2 关闭ExecutorService

ExecutorService有两种关闭方法
- shutdown: 不再接收提交任务的请求，但是等待所有已提交的任务完成后，再关闭。更安全
- shutdownNow：关闭所有正在执行的任务，并且返回所有尚未启动的任务清单。更快速

在复杂的应用程序中，通常会将`ExecutorService`封装在某个更高级的服务中，并且该服务能提供其自己的生命周期方法，所有权链上的各个成员都将管理它所拥有的服务或线程的生命周期。例子，如下



```java
public class LogService {
    private final ExecutorService exec = newSingleThreadExecutor();
    ...
    public void start() {}
    private void stop() throws InterruptedException {
        try {
            exec.shutdown();
            exec.awitTermination(TIMEOUT, UNIT);
        } finally {
            writer.close();
        }
    }
    public void log(String msg) {
        try {
            exec.execute(new WriteTask(msg));
        } catch (RejectedExecutionException ignored) {}
    }
    
}
```



##### 7.2.3 “毒丸”对象



另一种关闭生产者-消费者服务的方式是使用"毒丸"对象：“毒丸”是指一个放在队列上的对象，其含义是：“当得到这个对象时，立刻停止”。“毒丸”可以保证，在提交“毒丸”之前提交的所有工作都会被处理。



例子略



注意：单消费者多生产者（只需要每个生产者往队列中放一个毒丸，当消费者收集到N个毒丸时停止），或单生产者多消费者（生产者往队列中放入N个毒丸，每个毒丸都能毒死一个消费者）都可以使用毒丸。但多消费者多生产者不能使用毒丸（太复杂）。



##### 7.2.4 示例：只执行一次的服务



例子：检查很多邮箱服务器看看有没有新邮件到达

````java
boolean checkMail(Set<String> hosts, long timeout, TimeUnit unit) throws InterruptedException {
    ExecutorService exec = Executors.newCachedThreadPool();
    final AtomicBoolean hasNewMail = new AtomicBoolean(false);
    try {
        for (final String host: hosts)
            exec.execute(new Runnable() {
                public void run() {
                    if (checkMail(host))
                        hasNewMail.set(true);
                }
            });
    } finally {
        // 提交完所有任务立刻shutdown，不再接收新任务但是会执行完所有老任务
        exec.shutdown();
        // 设置了一个超时等待
        exec.awaitTermination(timeout, unit)
    }
    return hasNewMail.get();
}
````



##### 7.2.5 shutdownNow的局限性



使用shutdownNow时，我们无法通过常规的方法找出哪些任务已经开始但尚未结束。



下面这个例子对ExecutorService进行改造，使得我们可以知道哪些任务已经开始但尚未结束。


```java
public class TrackingExecutor extends AbstractExecutorService {
	private final ExecutorService exec;
	private final Set<Runnable> tasksCancelledAtShutdown = Collections.synchronizedSet(new HashSet<Runnable>());
	...
	public List<Runnable> getCancelledTasks() {
		if(!exec.isTerminated)
			throw new IllegalStateException(...);
		return new ArrayList<Runnable>(tasksCancelledAtShutdown);
	}
	
	public void execute(final Runnable runnable) {
		exec.execute(new Runnable(){
			public void run() {
				try {
                    // 假设这个runnable在收到中断时能抛出异常或者重置中断状态
					runnable.run();
				} finally {
                    // 如果已经进入停止状态了，并且当前线程收到了中断信号
					if (isShutdown() && Thread.currentThread().isInterrupted())
                        // 在shutdownNow中加入这个runnable
						tasksCancelledAtShutdown.add(runnable);
				}
			}
		})
	}
}
```



下面来看看这个改造之后的TrackingExecutor的用法：网页爬虫程序的工作通常是无穷尽的，但是我们希望在爬虫被关闭的时候能够保存没有完成的所有任务

````java
public abstract class WebCrawler {
    private volatile TrackingExecutor exec;
    @GuardedBy("this")
    private final Set<URL> urlsToCrawl = new HashSet<URL>();
    
    public synchronized void start() {
        exec = new TrackingExecutor(Executors.newCachedThreadPool());
        for (URL url: urlsToCrawl)
            submitCrawTask(url);
        urlsToCrawl.clear();
    }
    
    public synchronized void stop () throws InterruptedException {
        try {
            saveUnCrawled(exec.shutdownNow());
            if (exec.awaitTermination(TIMEOUT, UNIT))
                saveUnCrawled(exec.getCancelledTasks());
        } finally {
            exec = null;
        }
    }
    
    // 将还没爬取的url存到urlsToCrawl里
    private void saveUncrawled(List<Runnable> uncrawled) {
        for (Runnable task: uncrawled)
            urlsToCrawl.add((CrawlTask)task.getPage());
    }
    
    private void submitCrawlTask(URL u) {
        exec.execute(new CrawlTask(u));
    }
    
    private class CrawlTask implements Runnable {
        private final URL url;
        ...
        public void run() {
            for (URL link: processPage(url)) {
                if(Thread.currentThread().isInterrupted())
                    return;
                // 相当于递归的提交任务
                submitCrawlTask(link);
            }
        }
        public URL getPage() {return url;}
    }
}
````





#### 7.3 处理非正常的线程终止



当单线程程序由于发生了一个未捕获的异常而终止时，程序将停止运行，并产生与程序正常输出非常不同的栈追踪信息，这种情况是很容易理解的。



然后当并发程序的某个线程由于未捕获异常而终止时，虽然也会打印堆栈信息，但是程序不会停止运行。这样就容易遗漏这个终止。



导致线程提前死亡的最主要原因就是`RuntimeException`。任何代码都可能抛出`RuntimeException`，每当调用另一个方法时，都要对它的行为保持怀疑，不要盲目地认为它一定会正常返回。



因此，一种预防`RuntimeException`的方式，就是用try-catch包裹住，下面的例子展示了一个典型的线程池工作者线程，它使用try-catch将要执行的任务包裹住

```java
public void run() {
    Throwable throw = null;
    try {
        while (!isInterrupted)
            runTask(getTaskFromWorkQueue());
    } catch (Throwable e) {
        throw = e;
    } finally {
        threadExited(this, thrown);
    }
}
```

- 如果任务抛出一个未检查异常，那么它将会让工作者线程终结
- 但是在终结前，`threadExited(this, thrown)`方法会把这次终结以及异常传递给线程池，线程池会在这个方法中对这次终结进行各种处理



另一种预防`RuntimeException`的方式是注册一个全局的`UncaughtExceptionHandler`



接口如下

```java
public interface UncaughtExceptionHandler {
    void uncaughtException(Thread t, Throwable e);
}
```



使用方式如下，这样所有线程如果有未捕获的异常都将使用这个处理器的逻辑进行处理。

```java
public static void main(String[] args) {
	// 设置未捕获异常处理器，这里是默认的，当然你可以自己新建一个类，然后实现UncaughtExceptionHandler接口即可
	Thread.setDefaultUncaughtExceptionHandler(new UncaughtExceptionHandler() {

		@Override
		public void uncaughtException(Thread t, Throwable e) {
            Logger logger = Logger.getAnonymousLogger();
			logger.log(Level.SEVERE, "Thread terminated with exception: " + t.getName(), e);
		}
	});
	Thread thread = new Thread(new Task());
	thread.start();
    }
}
```



> 在运行时间比较长的应用程序中，通常会为所有线程的未捕获异常指定同一个异常处理器，并且该处理器至少会将异常消息记录到日志中。





注意：在Executor中有个问题

- 通过`execute`提交的任务才能交给`UncaughtExceptionHandler`处理,
- `submit`提交的任务会被包装在`get`里面，调用`get`之后，再抛出`ExecutionException`

#### 7.4 JVM关闭

 正常关闭 最后一个非守护线程结束,调用System.exit() 或者 Ctrl-C 

 强行关闭 Runtime.halt()  操作系统杀死jvm进程

1. 关闭钩子
    - 正常关闭中,JVM首先调用所有已注册的关闭钩子(通过Runtime.addShutdownHook注册单位开始的线程).
    - 关闭应用程序线程,如果有线程仍在运行,钩子线程并行执行.
    - 当所有钩子线程结束,如果runFinalizersOnExit为true,那么jvm将运行终结器(finalizer，其他书叫这个是析构器).
2. 守护线程
    - 和普通线程差别在于线程退出的操作,如果只剩下守护线程,jvm会自动关闭,
    - 守护线程不会执行finally也不会执行回卷栈，因此除非是内存的清理工作不然不要使用守护线程
3. 终结器

    - 垃圾回收器发现这些对象实现了finalize()方法。因为会把它们添加到java.lang.ref.Finalizer.ReferenceQueue队列中
    - Finalizer线程会处理这个队列，将里面的对象逐个弹出，并调用它们的finalize()方法。
    - finalize()方法调用完后，Finalizer线程会将引用从Finalizer类中去掉，因此在下一轮GC中，这些对象就可以被回收了



### 第8章 线程池的使用



本章将介绍对线程池进行配置与调优的一些高级选项，并分析在使用任务执行框架时需要注意的各种风险



#### 8.1　在任务与执行策略之间的隐性耦合



Executor框架可以将任务的提交与任务的执行策略解耦，但并非所有的任务都适用所有的执行策略。有些类型的任务需要明确地指定执行策略，包括

- 依赖性任务：如果提交给线程池的任务需要依赖其他的任务，那么应该注意**线程饥饿死锁**的问题
- 使用线程封闭机制的任务：单线程的`Executor`能够对并发性做出更强的承诺，对象可以封闭在任务线程中，使得在该线程中执行的任务在访问该对象时不需要同步。因此为单线程设计的任务通常只能在单线程中运行，改到多线程环境中会出问题
- 对响应时间敏感的任务
- 使用ThreadLocal的任务
	- ThreadLocal使每个线程都可以拥有某个变量的一个私有的版本
	- 在标准的`Executor`实现中，当执行需求较低时将回收空闲线程，而当需求增加时将添加新的线程，并且如果从任务中抛出一个未检查异常，那么将用一个新的工作者线程来替代抛出异常的线程。
	- 在线程池的线程中，不应该使用ThreadLocal在任务之间传递值

##### 8.1.1 线程饥饿死锁

线程饥饿死锁：只要线程池中的任务需要无限期地等待一些必须由池中其他任务才能提供的资源或条件，那么除非线程池足够大，否则将发生线程饥饿死锁。
- 例子1：在单线程的Executor中，如果一个任务将另一个任务提交到同一个Executor，并且等待这个被提交任务的结果，那么通常会引发死锁。第二个任务停留在工作队列中，并等待第一个任务完成，而第一个任务又无法完成，因为它在等待第二个任务的完成
- 例子2：在有界的线程池中，如果所有正在执行任务的线程都由于等待其他仍处于工作队列中的任务而阻塞，同样会发生这种问题


```java
public class ThreadDeadlock {
    ExecutorService exec = Executors.newSingleThreadExecutor();
    public class RenderPageTask implements Callable<String> {
        public String call() throws Exception {
            Future<String> header, footer;
            header = exec.submit(new LoadFileTask("header.html"));
            footer = exec.submit(new LoadFileTask("footer.html"));
            String page = renderBody();
            // 将发生死锁，因为任务在等待子任务的完成，父任务占据了唯一一个工作线程，子任务没有办法开始
            return header.get() + page + footer.get();
        }
    }
}
```



> 每当提交了一个**有依赖性的Executor任务**时，要清楚地知道可能会出现线程“饥饿”死锁，因此需要在代码或配置`Executor`的配置文件中记录线程池的大小限制或配置限制

#####　8.1.2 运行耗时时间长的任务

执行时间较长的任务会造成线程池阻塞，甚至还会增加执行时间较短任务的服务时间。如果线程池中线程的数量远小于在稳定状态下执行时间较长任务的数量，那么到最后可能所有的线程都会运行这些执行时间较长的任务，从而影响整体的响应度。

解决方案：限制任务等待资源的时间，而不要无限制等待。在平台类库的大多数可阻塞方法中，都同时定义了限时版本和无限时版本，例如：`Thread.join`、`BlockingQueue.put`、`CountDownLatch.await`以及`Selector.select`。如果等待超时，就可以将任务标记为失败，然后中止任务或者将任务放回队列重新执行。

#### 8.2 设置线程池的大小

设置线程池大小时，要避免过大过小两种极端
- 如果线程池设置过大，那么大量的线程将在相对很少的CPU和内存资源上发生竞争
- 如果线程池设置过小，那将导致很多空闲的处理器无法执行工作，从而降低吞吐率。

> 如果需要执行不同类型的任务，并且它们之间的行为相差很大，那么应该考虑使用多个线程池来执行每种类型的任务。

计算密集型任务： Ncpu个处理器,线程池大小Ncpu+1型任务

包含IO操作的任务，计算公式如下

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20211013195541.png)



#### 8.3 配置ThreadPoolExecutor



`ThreadPoolExecutor`是一个灵活的、稳定的线程池，它允许进行各种定制。其中一个构造函数如下所示

```java
public ThreadPoolExecutor(int corePoolSize,
                         int maximumPoolSize,
                         long keepAliveTime,
                         TimeUnit unit,
                         BlockingQueue<Runnable> workQueue,
                         ThreadFactory threadFactory,
                         RejectedExecutionHandler handler) {...}
```



##### 8.3.1 线程创建与销毁

**基本大小**(核心线程数) 没有任务执行时候线程池大小,只有工作队列满了才会创建超出这个数量的线程

**最大大小** 表示可同时活动的线程数量上限    

**存活时间** 如果线程空闲时间超过存活时间,会被标记可回收,当线程池超过基本大小，可回收的线程有可能会被终止

newFixedThreadPool工厂方法将线程池的基本大小和最大大小设置为参数中指定的值，而且创建的线程池不会超时。

newCachedThreadPool工厂方法将线程池的最大大小设置为`Integer.MAX_VALUE`，而将基本大小设置为零，并将超时时间设置为1分钟



##### 8.3.2 管理队列任务



在**有限的线程池**中会**限制**可**并发**执行的任务数量。



如果**采用固定大小的线程池**，在**高负载**的情况下，应用程序**还是可能会耗尽资源**

- 如果新请求的到达速率超过了线程池的处理速率，那么新到来的请求将累积起来。
- 在线程池中，这些请求会在一个由Executor管理的Runnable队列中等待，这同样会消耗内存，如果无限堆积，资源同样会耗尽



`ThreadPoolExecutor`允许**提供一个`BlockingQueue`来保存等待执行的任务**，基本的任务排队方法有3种

- **无界队列**
- **有界队列**
- **同步移交**(Synchronous Handoff)



`newFixedThreadPool`和`newSingleThreadExecutor`在默认情况下将使用一个无界的`LinkedBlockingQueue`。



一种更稳妥的资源管理策略是使用**有界队列**，当队列填满时，新的任务需要使用**饱和策略**处理



对于非常大或者无界线程池，可以**通过SynchronousQueue避免任务排队**

- SynchronousQueue不是一个真正的队列，而是一种在线程之间进行移交的机制
- 要将一个元素放入`SynchronousQueue`中，必须有另一个线程正在等待接受这个元素
- 如果没有线程正在等待，并且线程池的大小没有达到上限，就创建一个新线程
- `newCachedThreadPool`使用的是`SynchronousQueue`



如果想进一步控制任务执行顺序，可以使用`PriorityBlockingQueue`，这个队列将根据**优先级**来安排任务。



##### 8.3.3 饱和策略



ThreadPool的**饱和策略**可以通过`setRejectedExecutionHandler`来修改



JDK提供了多种不同的`RejectExecutionHandler`，每种实现都包含了不同的饱和策略

- **Abort**：默认策略,抛出未检查的`RejectedExecutionException`
- **Discard**：悄悄丢弃任务

- **Discard-Oldest**：丢弃最旧的任务

- **Caller-Runs**：将回退到调用者，然后会在调用者线程中执行这个任务。



可以在前几章的WebServer例子中使用Caller-Runs策略来控制流量

- 当线程池中的**所有线程都被占用**，并且工作队列被填满时，**下一个任务**会在调用`execute`时在**主线程**中执行
- 由于执行任务需要一定的时间，因此主线程至少在一段时间内不能提交任何任务，从而使得工作者线程有时间来处理完正在执行的任务
- 在这期间，主线程不会调用`accept`，因此到达的请求将被保存在TCP层的队列而不是应用程序的队列中
- 如果**持续过载**，**TCP层**将最终发现它的**请求队列**被**填满**，因此同样会开始抛弃请求
- 当服务器过载时，这种过载情况会逐渐向外蔓延开来：从线程池到工作队列到应用程序再到TCP层，最终达到客户端，导致服务器在高负载下实现一种平缓的性能下降。



例子：如何设置**饱和策略**

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(N_THREADS, N_THREADS, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>(CAPACITY));
executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
```





##### 8.3.4 线程工厂



每当线程池需要创建一个线程时，都是通过线程工厂完成的。

- 默认的线程工厂将创建一个新的、非守护的线程并且不包含特殊的配置信息
- 通过指定一个线程工厂方法，可以定制线程池的配置信息



`ThreadFactory`接口定义如下

```java
public interface ThreadFactory {
    Thread newThread(Runnable r);
}
```

 

下面举一个自定义线程工厂的例子

```java
public class MyThreadFactory implements ThreadFactory {
    private final String poolName;
    public MyThreadFactory(String poolName) {
        this.poolName = poolName;
    }
    public Thread newThread(Runnable runnable) {
        return new MyAppThread(runnable, poolName);
    }
}

public class MyAppThread extends Thread {
    public static final String DEFAULT_NAME = "MyAppThread";
    private static volatile boolean debugLifeCycle = false;
    private static final AtomicInteger created = new AtomicInteger();
    private static final AtomicInteger alive = new AtomicInteger();
    private static final Logger log = Logger.getAnonymousLogger();
    
    public MyAppThread(Runnable r) { this(r, DEFAULT_NAME)};
    
    public MyAppThread(Runnable runnable, String name) {
        super(runnable, name + "-" + created.incrementAndGet());
        setUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler(){
            public void uncaughtException(Thread t, Throwable e) {
                log.log(LEVEL.SEVERE, "UNCAUGHT in thread " + t.getName(), e);
            }
        })
    }
    
    public void run() {
        boolean debug = debugLifecycle;
        if (debug)
            log.log(Level.FINE, "Created " + getName());
        try {
            alive.incrementAndGet();
            super.run();
        } finally {
            alive.decrementAndGet();
            if (debug) log.log(Level.FINE, "Exiting " + getName());
        }
    }
    
    public static int getThreadsCreated() { return created.get(); }
    public static int getThreadsAlive() { return alive.get(); }
    public static boolean getDebug() { return debugLifeCycle; }
    public static void setDebug(boolean b) { debugLifeCycle = b; }
}
```



##### 8.3.5 调用构造函数后再定制ThreadPoolExecutor



在调用完ThreadPoolExecutor的构造函数后，仍然可以通过设置函数(`Setter`)来修改大多数通过构造函数传递的参数

但是在Executors中有一个`unconfigurableExecutorService`工厂方法，该方法对ExecutorService进行封装，使它只暴露`ExecutorService`方法，因此不能对它进行配置。这样用户代码就无法修改配置了。



#### 8.4 扩展ThreadPoolExecutor



ThreadPoolExecutor提供了几个可以在子类中改写的方法：`beforeExecute`、`afterExecute`和`terminated`



`beforeExecute` 如果`beforeExecute`抛出一个`RuntimeException`，那么任务将不再执行，并且`afterExecute`也不会被调用

`afterExecute`: 无论任务是从`run`中正常返回还是抛出一个异常而返回，`afterExecute`一定会被调用

`terminated` 在线程池完成关闭操作时调用`terminated`，也就是在所有任务都已经完成并且所有工作者线程都关闭后



下面举一个通过改写这几个方法来统计线程池运行时间的例子

```java
public class TimingThreadPool extends ThreadPoolExecutor {
    private final ThreadLocal<Long> startTime = new ThreadLocal();
    private final Logger log = Logger.getLogger("TimingThreadPool");
    private final AtomicLong numTasks = new AtomicLong();
    private final AtomicLong totalTime = new AtomicLong();
    
    protected void beforeExecute(Thread t, Runnable r) {
        super.beforeExecute(t, r);
        log.fine(String.format("Thread %s: start %s", t, r));
        startTime.set(System.nanoTime());
    }
    
    protected void afterExecute(Runnable r, Throwable t) {
        try {
            long endTime = System.nanoTime();
            long taskTime = endTime - startTime.get();
            numTasks.incrementAndGet();
            totalTime.addAndGet(taskTime);
            log.fine(String.format("Thread %s: end %s, time=%dns", t, r, taskTime));
        } finally {
            super.afterExecute(r, t);
        }
    }
    
    protected void terminated() {
        try {
            log.info(String.format("Terminated: avg time=%dns", totalTime.get()/numTasks.get()));
        } finally {
            super.terminated();
        }
    }
}
```





#### 8.5 递归算法并行化

1. 如果迭代操作之间时独立的,直接可以并行执行
2. 递归不依赖于后续递归的结果



实际例子略



### 第9章 GUI

 GUI限制在单程程,几乎所有工具包都被实现为单线程子系统

#### 9.1 为什么GUI是单线程的

早期GUI都是单线程的,在主事件循环进行处理事件.当前的GUI采用事件分发线程进行处理事件.采用一个专门的线程从队列中抽取事件,并将他们转发到应用程序定义的事件处理器

多线程处理容易引发死锁,其次就是MVC设计模式

1. 串行事件处理

    优势 代码编写简单
   
    劣势 处理耗时长的任务,发生无响应现象(可以委派给另一个线程完成)

2. Swing的线程封闭机制

    所有Swing组件和数据模型对象都封闭在事件线程中,任何访问它们的代码必须在事件线程里
   
    invokeLater 和 invokeAndWait两个方法酷似 Executor

#### 9.2 短时间的GUI任务

事件在事件线程中产生,并通过气泡上升 的方式床底给应用程序

Swing将大多数可视化组件分为两个对象(模型对象和视图对象),模型对象保存数据,可以通过引发事件表示模型发生变化,视图对象通过订阅接收事件

#### 9.3 长时间的GUI任务

对于长时间的任务可以使用线程池

1. 取消 使用Future
2. 进度标识 

#### 9.4 共享数据模型

1. 只要阻塞操作不会过度影响响应性,那么事件线程和后台线程就可以共享该模型
2. 分解数据模型.将共享的模型通过快照共享



#### 9.5 其他形式单线程

每当某个工具需要被实现为单线程子系统都可以使用.一些原生的库



## Part 3 活跃性 性能与测试



#### 第10章 避免活跃性危险



如果过度地使用加锁，则可能导致**锁顺序死锁**(Lock-Ordering Deadlock)。同样，如果我们使用线程池和信号量来限制对资源的使用，那么这些被限制的行为可能会导致**资源死锁**(Resource Deadlock)。



Java应用程序无法从死锁中恢复过来，因此在程序设计时，一定要排除那些可能导致死锁出现的条件。



本章主要介绍死锁，以及其他的一些活跃性故障。



#### 10.1 死锁



在数据库服务器，当它检测到一组事务上发生了死锁（通过在表示等待关系的有向图中搜索循环），将选择一个牺牲者并放弃这个事务，牺牲者会释放它所持有的资源，从而使其他事务继续执行。



JVM中，如果一组Java线程发生了死锁，那么就Gameover了，这组线程永远也不能使用了。



##### 10.1.1 锁顺序死锁



在下面代码中可能会出现死锁

```java
public class LeftRightDeadlock {
    private final Object left = new Object();
    private final Object right = new Object();
    public void leftRight() {
        synchronized (left) {
            synchronized (right) {
                dosomething();
            }
        }
    }
    public void rightLeft() {
        synchronized(right) {
            synchronized (left) {
                dosomething();
            }
        }
    }
}
```



发生死锁的原因如下图

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20211013195613.png)

- 两个线程试图以不同的顺序来获取相同的锁。如果按照相同的顺序来请求锁，那么就不会出现循环的加锁依赖性。



> 如果所有线程以固定的顺序获取锁，那么在程序中就不会出现锁顺序死锁问题



要想验证锁顺序的一致性，需要对程序中的**加锁行为**进行**全局分析**。



##### 10.1.2 动态的锁顺序死锁



下面这段看似无害的代码也可能导致死锁，它试图将资金从一个账号转移到另一个账号

```java
public void transferMoney(Account fromAccount, Account toAccount, DollarAmount amount) throws InsufficientFundsException {
    synchronized (fromAccount) {
        synchronized (toAccount) {
            if (fromAccount.getBalance().compareTo(amount) < 0)
                throw new InsufficientFundsException();
            else {
                fromAccount.debit(amount);
                toAccount.credit(amount);
            }
        }
    }
}
```

- 如果X向Y转账的同时，Y向X转账就可能死锁。



解决方法还是限制加锁的顺序

````java
private static final Object tieLock = new Object();
public void transferMoney(final Account fromAcct, 
                          final Account toAcct, 
                          final DollarAmount amount) throws InsufficientFundsException {
    class Helper {
        public void transfer() throws InsufficientFundsException {
            if (fromAcct.getBalance().compareTo(amount) < 0)
                throw new InsufficientFundsException();
            else {
                fromAcct.debit(amount);
                toAcct.credit(amount);
            }
        }
    }
   int fromHash = System.identityHashCode(fromAcct);
   int toHash = System.identityHashCode(toAcct);
   if (fromHash < toHash) {
       synchronized (fromAcct) {
           synchronized (toAcct) {
               new Helper().transfer();
           }
       }
   } else if (fromHash > toHash) {
       synchronized (toAcct) {
           synchronized (fromAcct) {
               new Helper().transfer();
           }
       }
   } else {
       synchronized (fromAcct) {
           synchronized (toAcct) {
               new Helper().transfer();
           }
       }
   }
}
````

- 如果两个对象拥有相同的hash值，那么就使用“加时赛(Tie-Breaking)”锁。在获得两个Account锁之前，首先获得这个“加时赛”锁，从而保证每次都只有一个线程以未知的顺序来获得这两个锁，从而消除了死锁发生的可能性。
- 如果在account中存在某些唯一标识符，那么就不需要散列也不需要加时赛锁了



> 使用`System.identityHashCode()`可以获得一个`Object`的`hashCode`



##### 10.1.3 在协作对象之间发生的死锁



某些获取多个锁的操作并不明显，这些锁并不一定在一个方法中被获取。



考虑下面这段代码，有两个互相协作的类，`Taxi`表示一个出租车对象，`Dispatcher`表示出租车的调度器

```java
class Taxi {
    @GuardedBy("this") private Point location destination;
    private final Dispatcher dispatcher;
    public Taxi(Dispatcher dispatcher) {
        this.dispatcher = dispatcher;
    }
    public synchronized Point getLocation () {
        return location;
    }
    public void synchronized void setLocation (Point location) {
        this.location = location;
        if (location.equals(destination))
            dispatcher.notifyAvailable(this);
    }
}

class Dispatcher {
    @GuardedBy("this") private final Set<Taxi> taxis;
    @GuardedBy("this") private final Set<Taxi> availableTaxis;
    public Dispatcher() {
        taxis = new HashSet<Taxi>();
        availableTaxis = new HashSet<Taxi>();
    }
    
    public synchronized void notifyAvailable(Taxi taxi) {
        availableTaxis.add(taxi);
    }
    
    public synchronized Image getImage() {
        Image image = new Image();
        for (Taxi t: taxis)
            image.drawMarker(t.getLocation());
        return image;
    }
}
```

- 上面这段代码死锁
  - setLocation()会先请求Taxi的锁再请求Dispatcher的锁
  - 而反过来，getImage()会先请求dispatcher的锁，然后再获取每一个Taxi的锁



> 如果在持有锁的情况下调用某个外部方法，就需要警惕死锁



##### 10.1.4 开放调用



方法调用相当于一种抽象屏障，因而你无须了解在被调用方法中所执行的操作，但也正是因为不知道在被调用方法中执行的操作，因此在持有锁的时候对调用某个外部方法将难以进行分析，从而可能出现死锁



如果在调用某个方法时不需要持有锁，那么这种调用被称为**开放调用**(Open Call)。分析一个完全依赖于开放调用的程序的活跃性，要比分析那些不依赖开放调用的程序的活跃性简单。



我们可以很容易的将10.1.3中的例子改为使用开放调用，从而消除发生死锁的风险，这需要使同步代码块仅用于保护那些涉及共享状态的操作。



```java
@ThreadSafe
class Taxi {
    @GuardedBy("this") private Point location destination;
    private final Dispatcher dispatcher;
    public Taxi(Dispatcher dispatcher) {
        this.dispatcher = dispatcher;
    }
    public synchronized Point getLocation () {
        return location;
    }
    public void setLocation (Point location) {
        boolean reachedDestination; 
        synchronized {
            this.location = location;
            reachedDestination = location.equals(destination);
        }
        if (reachedDestination)
            dispatcher.notifyAvailable(this);
    }
}

@ThreadSafe
class Dispatcher {
    @GuardedBy("this") private final Set<Taxi> taxis;
    @GuardedBy("this") private final Set<Taxi> availableTaxis;
    public Dispatcher() {
        taxis = new HashSet<Taxi>();
        availableTaxis = new HashSet<Taxi>();
    }
    
    public synchronized void notifyAvailable(Taxi taxi) {
        availableTaxis.add(taxi);
    }
    
    public Image getImage() {
        Image image = new Image();
        Set<Taxi> copy;
        synchronized (this) {
            copy = new HashSet<Taxi>(taxis);
        }
        for (Taxi t: copy)
            image.drawMarker(t.getLocation());
        return image;
    }
}
```



> 在程序中应尽量使用开放调用，与那些在持有锁时调用外部方法的程序相比，更易于对依赖于开放调用的程序进行死锁分析



##### 10.1.5 资源死锁



当多个线程在相同的资源集合中等待时，也可能死锁



如果一个服务器应用程序需要连接两个数据库，但是在请求两个资源时不会始终按照同一个顺序。那么就有可能死锁



还有一种死锁是线程饥饿死锁，见8.1.1



有界线程池或有界资源池不能与相互依赖的任务一起使用。



#### 10.2 死锁的避免与诊断



如果一个程序每次至多获取一个锁，那么就不会出现锁顺序死锁

如果必须获取多个锁，就要在设计时考虑锁的顺序

尽可能使用开发调用，这能简化分析过程，如果所有的调用都是开放调用，那么要发现获取多个锁的实例是非常简单的。



##### 10.2.1 支持定时的锁



显式地使用Lock类中的定时`tryLock`功能，也可以避免死锁



##### 10.2.2 通过线程转储(Thread Dump)消息来分析死锁



JVM能通过线程转储来帮助识别死锁的发生



下图展示了某次死锁的线程转储信息

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20211013195639.png)

- ![image-20210930194630943](C:\Users\MSI-NB\AppData\Roaming\Typora\typora-user-images\image-20210930194630943.png)



#### 10.3 其他活跃性危险



##### 10.3.1 饥饿



饥饿：线程由于无法访问它所需要的资源而不能继续执行



如果线程在持有锁时执行一些无法完成的结构，比如无限循环，就会导致其他线程饥饿

此外，线程优先级也有可能导致饥饿发生



在ThreadAPI中定义了10个优先级，JVM根据需要将它们分别映射到操作系统的调度优先级。因此在某个操作系统中两个不同的Java优先级可能被映射到同一个操作系统优先级



> 要避免使用线程优先级，因为这会增加平台依赖性，并可能导致活跃性问题。在大多数并发程序中，都可以使用默认的线程优先级。



##### 10.3.2 糟糕的响应性



如果某个线程长期持有锁，其他线程的响应性就变得很差



##### 10.3.3 活锁



活锁的例子：比如两个过于礼貌的人在半路上面对面相遇，他们彼此都想让出对方的路，然而又在另一条路上碰见了，就这样反复的避让下去，永远不会结束。

线程不断重复执行相同的操作,而且总会失败(常见于处理事务消息的应用程序),可以**在重试机制里面加入随机性**

### 第11章 性能与可伸缩性



**线程**的最主要目的是**提高程序的性能**

1. 线程可以使程序**充分发挥**系统的可用**处理能力**，从而**提高系统的资源利用率**
2. 线程可以使程序**在运行现有任务的情况下立即开始处理新的任务**，从而**提高系统的响应性**
3. 但是，使用线程会**增加复杂性**，因此就增加了在安全性和活跃性上发生失败的风险



首先要保证程序能够正确执行，然后仅当程序的性能需求和测试结果需要程序执行得更快时，才应该设法提高程序的性能。



#### 11.1 对性能的思考



> 当操作性能由于某种特定的资源而收到限制时，我们通常将该操作称为xx密集型操作，例如：CPU密集型、IO密集型等。



使用多线程会引入一些额外的性能开销

- 线程之间的协调（加锁、内存同步等）
- 上下文切换开销
- 线程的创建和销毁
- 线程的调度



##### 11.1 性能与可伸缩性



> 可伸缩性是指：当增加计算资源时，程序的吞吐量或者处理能力能够相应地增加



通常，能够提高单一线程执行速度的方法会降低可伸缩性。而在并发编程中，我们通常会接受每个工作单元执行更长的时间或消耗更多的计算资源，以换取应用程序在增加更多的计算资源时处理更高的负载。



#### 11.2 Amdahl定律



大多数并发程序都是由一系列并行工作和串行工作组成的



Amdahl定律描述的是，在增加计算资源的情况下，程序在理论上能够实现的最高加速比
$$
SpeedUp  \le \frac {1} {F + \frac{1-F}{N}}
$$

- 当N趋近于无穷大时，最大的加速比趋近于$1/F$，因此如果程序中有50%的计算需要串行执行，那么最高的加速比只能是2.



> 在所有并发程序中都包含一些串行部分



##### 11.2.1 示例：隐藏的串行部分



本节给出了一个简单的例子，其中多个线程反复地从一个共享`Queue`中取出元素进行处理。



下图是对两个线程安全的Queue的吞吐率的比较：其中一个采用`synchronizedLinkedList`、另一个采用`ConcurrentLinkedQueue`。

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20211013195658.png)



吞吐率的差异来源于两个队列中不同比例的串行部分。

- 同步的`LinkedList`采用单个锁来保护整个队列的状态。
- 而`ConcurrentLinkedList`采用的是一种非阻塞的队列算法



#### 11.3 线程引入的开销



对于为了提升性能而引入的线程来说，并行带来的性能提升必须超过并发导致的开销。



##### 11.3.1 上下文切换



如果可运行的线程数大于CPU的数量，那么操作系统最终会将某个正在运行的线程调度出去，从而使其他线程能够使用CPU。这将导致一次上下文切换。



切换上下文需要一定的开销

- 在线程调度过程中需要访问由操作系统和JVM共享的数据结构。在JVM和操作系统中消耗的CPU周期越多，用户代码能够使用的CPU周期就越少
- 上下文切换将导致缓存缺失，第一次运行的线程所需要的数据可能并不在当前处理器的本地缓存中，需要读取内存



当线程由于等待某个锁或者等待某个IO操作，就会发生阻塞，JVM会将阻塞的线程切换为挂起状态，并且允许它被切换出去。



> 关于上下文切换可以看这个https://zhuanlan.zhihu.com/p/52845869



##### 11.3.2 内存同步



`synchronized`和`volatile`机制提供的可见性保证会使用一些特殊的指令，被称为内存栅栏(memory barrier)。

- 内存栅栏会刷新缓存，使缓存无效。
- 并且内存栅栏会抑制编译器的重排序



`synchronized`会对无竞争的同步进行优化：这块可以看Java并发编程的艺术介绍的轻量级锁那些。



现代的JVM能通过优化来去掉一些不会发生竞争的锁，从而减少不必要的同步开销



并且编译器可以执行锁粒度粗化，可以将相邻的同步代码块用同一个锁合并起来



某个线程中的同步可能会影响其他线程的性能。同步会增加共享内存总线上的通信量，总线的带宽是有限的，并且所有的处理器都共享这根总线。如果有多个线程竞争同步带宽，那么所有使用了同步的线程都会受到影像。



##### 11.3.3 阻塞



JVM在实现阻塞行为时，可以采用自旋等待，或者通过操作系统挂起被阻塞的线程。这两种方式的效率取决于上下文切换的开销以及在成功获取锁之前需要的等待时间

- 自旋会消耗CPU，使CPU空转
- 挂起会导致上下文切换



#### 11.4 减少锁的竞争



我们可以看到，串行操作会降低可伸缩性，而上下文切换会降低性能。在锁上发生竞争将同时导致这两种问题，因此减少锁竞争能够提高性能和可伸缩性。

> 在并发程序中，对可伸缩性的最主要威胁就是独占方式的资源锁。独占：只允许一个线程访问



有两种因素会影像在锁上发生竞争的可能性

1. 锁的请求频率
2. 每次持有锁的时间



有三种方式能够降低锁的竞争

1. 减少锁的持有时间
2. 降低锁的请求频率
3. 使用带有协调机制的独占锁，这些机制允许更高的并发性



##### 11.4.1 缩小锁的范围



跟前面章节表述的一样，尽量只在确实需要同步的地方加锁，不要设置很多粗粒度的锁



> 仅当可以将大量的计算或阻塞操作从同步代码块移出时，才应该考虑同步代码块的大小



##### 11.4.2 减少锁的粒度



可以通过锁分段或者锁分解技术来降低线程请求锁的频率。但是与此同时，使用的锁越多，发生死锁的可能性就越大



例如下面这个例子

```java
@ThreadSafe
public class ServerStatus {
    @GuardedBy(this) public final Set<String> users;
    @GuardedBy(this) public final Set<String> queries;
    ...
    public synchronized void addUser(String u) {users.add(u)};
    public synchronized void addQuery(String q) {queries.add(q);}
    public synchronized void removeUser(String u) {
        users.remove(u);
    }
    public synchronized void removeQuery(String q) {
        queries.remove(q);
    }
}
```



因为queries和users是相互独立的，因此直接拆成两个锁

```java
@ThreadSafe
public class ServerStatus {
    @GuardedBy("users") public final Set<String> users;
    @GuardedBy("queries") public final Set<String> queries;
    ...
    public void addUser(String u) {
        synchronized (users) {
            user.add(u);
        }
    };
    public void addQuery(String q) {
       synchronized (queries) {
        	queries.add(q);   
       } 
    }
    public synchronized void removeUser(String u) {
        users.remove(u);
    }
    public synchronized void removeQuery(String q) {
        queries.remove(q);
    }
}
```



##### 11.4.3 锁分段



例如，在Java1.5中，在`ConcurrentHashMap`的实现中使用了一个包含16个锁的数组，每个锁保护所有散列桶的1/16。正是这项技术使得`ConcurrentHashMap`能够支持多达16个并发的写入器。（1.8改了）



> 1.8的`ConcurrentHashMap`看这里https://juejin.cn/post/6844903678491492359



锁分段的劣势就是，对于那些需要加锁整个容器的操作，锁分段使用起来不方便，因为需要同时获取多个锁。



下面给出一个锁分段的例子

```java
@ThreadSafe
public class StripedMap {
    private static final int N_LOCKS = 16
    private final Node[] buckets;
    private final Object[] locks;
    
    public StripedMap(int numBuckets) {
        buckets = new Node[numBuckets];
        locks = new Object[N_LOCKS];
        for (int i = 0; i < N_LOCKS; i ++) {
            locks[i] = new Object();
        }
    }
    
    private final int hash(Object key) {
        return Math.ads(key.hashCode() % buckets.length);
    }
    
    public Object get(Object key) {
        int hash = hash(key);
        synchronized (locks[hash % N_LOCKS]) {
            for (Node m = buckets[hash]; m != null; m = m.next)
                if (m.key.equals(key))
                    return m.value;
        }
        return null;
    }
    
    public void clear() {
        for (int i = 0; i < buckets.length; i ++) {
            synchronized (locks[i % N_LOCKS]) {
                buckets[i] = null;
            }
        }
    }
}
```



##### 11.4.4 避免热点域



比如如果我们为`Map`保存一个`size`变量，那么并发调用`sizeOf`时，就会在`size`变量上形成热点域。



`ConcurrentHashMap`的解决方式是为每个分段都维护一个独立的计数，并通过每个分段的锁来维护这个值。并且在`sizeOf`时不会同时请求所有的锁，这样虽然会导致`sizeOf`算出来的值不精确，但是是一种tradeoff。



##### 11.4.5 一些替代独占锁的方法



第三种降低竞争锁的影响的技术是放弃使用独占锁

- 可以使用读写锁ReadWriteLock

- 可以使用原子变量：它更细粒度



##### 11.4.6 监测CPU的利用率



如果CPU的利用率不充分，应该考虑以下几个原因

1. 负载不充分：测试程序并没有提供足够的负载
2. I/O密集：应用的瓶颈不是CPU而是I/O或者网络
3. 外部限制：如果应用程序依赖于外部服务，比如数据库或者缓存服务器，那么性能瓶颈可能并不在自己的代码中
4. 锁竞争：有可能在一些锁上发生了激烈的竞争



如果CPU的利用率很高并且总会有可运行的线程在等待CPU，那么当增加更多的CPU时，程序的性能会得到提升。



##### 11.4.7 不要使用对象池



因为Java创建对象的消耗已经很低了，不要再使用对象池了。

重新创建对象不会引起同步问题，但是用对象池中之前的对象可能会。



#### 11.6 减少上下文开销



本节的例子是第7.2.1节中的日志服务



有两种记录日志方案

1. 当需要记录日志时，由任务线程将日志写入对应的文件
2. 记录日志的工作由某个专门的后台线程处理，任务线程只是将工作提交到后台线程的queue中就行



第二种方式将IO操作从处理请求的线程中分离出来，缩短了处理请求的平均服务时间，调用log方法的线程将不会再因为等待输出流的锁或者等待I/O完成而被阻塞，它们只需要将消息放到队列，然后就返回到各自的任务中。



> 我们将工作分散开来，并将IO操作移到另一个用户感知不到开销的线程中，避免了阻塞同时还通过线程封闭消除了输出流上的竞争



### 第12章 并发程序的测试

性能测试 吞吐量 响应性 可伸缩性

#### 12.1 正确性测试

1. 调用各个方法,验证后验条件和不变型条件

2. 阻塞操作的测试当方法成功阻塞后必须是方法解除阻塞

    使用终端,在一个单独线程中启动一个阻塞操作,等到线程阻塞后再中断它,然后宣告阻塞成功.
   
    Thread.getState不可靠.被阻塞线程不需要Waiting状态,jvm可以通过自旋实现阻塞.类似Object.wait和Condition.wait存在伪唤醒

3. 安全性测试

    生产者-消费者 可以通过一个对吮吸敏感的校验和计算函数来计算所有入列原色和出列元素的校验和,一致则通过

4. 资源管理测试

    使用堆查看的工具

5. 使用回调

    线程池可以使用自定的线程工厂

6. 产生更多的交替操作

    Thread.yield(Thread.sleep虽然更慢,但是更可靠一些)

#### 12.2 性能测试

1. 增加计时功能

    使用栅栏进行计时

2. 多种算法的比较
3. 响应性衡量

#### 12.3 避免性能测试的陷阱

1. 垃圾回收(无法预测)

    要么不执行,要么执行多次

2. 动态编译

    运行足够长时间或者与先运行一段时间并且不测试代码性能

3. 对代码路径的不真实采样

    JVM可以与执行过程特定的信息生成更优的代码,编译器也会优化

4. 不真实的竞争程度

    尽量模拟真实情况,CPU密集型或者IO密集型

5. 无用代码的删除

    技巧 计算某个派生类的散列值,与任意值比较,加入相等就输出一个无用且可被忽略的消息

#### 12.4 其他测试方法

1. 代码审查
2. 静态分析工具(FindBugs等)
3. 面向方面的测试技术(06年就有AOP,感慨一下)
4. 分析与检测工具(JMX等)

## Part4 高级主题

### 第13章 显式锁



Java5.0新增了`ReentrantLock`，但它并不是一种替代内置加锁的方法，而是当内置加锁机制不适用时，作为一种可选择的高级功能。



#### 13.1 Lock和ReentrantLock



`Lock`提供了一种无条件的、可轮询的、定时的以及可中断的锁获取操作



`Lock`接口如下

```java
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```



`ReentrantLock`实现了`Lock`接口，并且提供了与`synchronized`相同的互斥性和内存可见性。与`synchronized`一样，`ReentrantLock`还提供了可重入的加锁语义。



为什么要创建一种与内置锁如此相似的新加锁机制？因为内置锁有很多功能上的限制

- 无法中断一个正在等待获取锁的线程
- 无法在请求获取一个锁时有限地等待下去（无法超时跳出）
- 无法提供轮询锁（调用`lock`时不阻塞，而是快速返回一个`false`代表现在获取不了锁）



下面给出`Lock`接口的标准使用方式，它必须在`finally`块中释放锁

```java
Lock lock = new ReentrantLock();
...
lock.lock();
try {
    // 更新对象状态
    // 捕获异常，并且在必要时恢复不变性条件
} finally {
    lock.unlock();
}
```



`ReentrantLock`不能完全替代`synchronized`的原因是，它更加危险，因为当程序的执行控制离开被保护的代码块时，不会自动清除锁。



##### 13.1.1 轮询锁和定时锁



可定时的锁: `boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException;)`

轮询锁:`boolean tryLock();`



使用轮询锁或者定时锁能够解决死锁问题



##### 13.1.2 可中断的锁获取操作



内置的锁是无法响应中断的，但是通过`Lock`接口的`lockInterruptibly`方法能够在获取锁的同时保持对中断的响应。此外，定时的`tryLock`也能响应中断。



下面举个例子

```java
public boolean sendOnSharedLine(String message) throws InterruptedException{
    lock.lockInterruptibly();
    try {
        // 如果在执行这个方法时发生了中断，则中断可以被抛出，并且锁会在finally块中被释放
        return cancellableSendOnSharedLine(message);
    } finally {
        lock.unlock();
    }
}
```



##### 13.1.3 非块结构的加锁



如果使用`synchronized`同步块，只能在加锁的方法中同时释放锁。有的时候这不够灵活。如果使用`Lock`API就可以实现在一个位置加锁，但是在另一个位置释放锁。



#### 13.2 性能因素



内置锁和显式锁性能差不多



#### 13.3 公平性



`ReentrantLock`的构造函数提供了两种公平性的选择

- 创建一个非公平的锁（默认）
- 创建一个公平的锁



在非公平的锁上，允许“插队”：

- 当一个线程请求非公平的锁时，如果在发出请求的同时该锁的状态变为可用，那么这个线程将跳过队列中所有的等待线程并且获取这个锁。



公平性会降低性能，非公平性可以提高性能,原因如下

- 假设线程A持有一个锁，并且线程B请求这个锁
- 由于这个锁被线程A持有，因此线程B将被挂起
- 当A释放锁时，B将被唤醒，因此会再次尝试获取锁
- 与此同时，如果C也请求这个锁，那么C很可能在B被完全唤醒之前获得、使用以及释放这个锁。
- 这种情况是一种“双赢”的局面



#### 13.4  在synchronized和ReentrantLock之间进行选择



在一些内置锁无法满足需求的情况下，`ReentrantLock`可以作为一种高级工具。当需要一些高级功能时才应该使用`ReentrantLock`，这些功能包括：可定时的、可轮询的与可中断的锁获取操作，公平队列，以及非块结构的锁。否则，应该优先使用`synchronized`



#### 13.5 读写锁



在大多情况下，对于数据的访问，是读操作占大部分。因此如果能够放宽加锁需求，允许多个执行读操作的线程同时访问数据结构，那么将提升程序的性能。



读写锁接口如下

```java
public interface ReadWriteLock {
    Lock readLock();
    Lock writeLock();
}
```

要读取由`ReadWriteLock`保护的数据，必须首先获得读取锁，当需要修改`ReadWriteLock`保护的数据时，必须先获得写入锁。尽管这两个锁看上去是彼此独立的，但是读取锁和写入锁只是一个锁对象的不同视图。



读取锁和写入锁有很多交互方式，`ReadWriteLock`中有很多可选的配置

- 释放优先：当一个写入操作释放写入锁时，并且在队列中同时存在读线程和写线程，那么应该优先选择读线程还是写线程还是等待时间最长的线程
- 读线程插队：如果锁是由读线程持有的，但是有写线程在队列中等待，那么新到达的读线程是否能够立刻获得访问权
- 重入性：读取锁和写入锁是否是可重入的
- 降级：如果一个线程持有写入锁，那么它是否能够在不释放该锁的情况下直接获取读取锁
- 升级：读取锁是否优先于其他正在等待的读线程和写线程而升级为一个写入锁（这可能导致死锁）



### 第14章 构建自定义的同步工具



类库包含了许多存在状态依赖性的类，例如`FutureTask`、`Semaphore`和`BlockingQueue`等。在这些类的一些操作中有着**基于状态**的**前提条件**。例如，不能从一个空的队列中删除元素，不能获取一个尚未结束的任务的计算结果。



创建状态依赖类的最简单方法通常是在类库中现有状态依赖类的基础上进行构建。

但如果类库没有提供你需要的功能，那么还可以使用Java语言和类库提供的底层机制来构建自己的同步机制，包括内置的条件队列、显式的`Condition`对象以及`AbstractQueuedSynchronizer`框架



#### 14.1 状态依赖性的管理



在单线程程序中调用一个方法时，如果某个基于状态的前提条件未得到满足，那么这个条件将永远无法满足。但是在并发程序中，基于状态的条件有可能会由其他线程的操作而改变。因此，对于并发对象上依赖状态的方法，与其直接让其失败，不如阻塞以等待前提条件为真。



内置的条件队列可以实现：让某个线程一直阻塞，直到基于状态的条件满足，并且当条件满足时可以再唤醒阻塞的线程。



为了凸显出条件队列的价值，我们在这一小节首先介绍通过轮询和休眠来勉强地解决状态依赖性问题。



可阻塞的状态依赖操作形式上如下所示

```
acquire lock on object state //获取状态上的锁
while (precondition does not hold) // 如果条件不满足就重试
	release lock //释放锁
	wait until precondition might hold // 等待直到条件可能被满足
	optionally fail if interrupted or timeout expires //如果发生了中断或超时就直接失败
	reacquire lock //重新请求锁
}
perform action //执行操作
release lock //释放锁
```

1. 构成前提条件的状态变量必须由对象的锁来保护，从而使它们在测试前提条件的同时保持不变
2. 如果前提条件尚未满足，就必须释放锁，以便其他线程能够修改对象的状态（如果不释放锁，前提条件永远无法为真）
3. 再次测试前提条件之前，必须重新获得锁



这一章有一个大例子贯穿整章：假设我们现在要实现一个有界缓存：不能从空缓存中取出元素也不能把元素放到满缓存中。首先给出一个抽象类，然后我们用各种不同的手段来扩展这个抽象类

```java
package com.edu.neu.multiThreadSample.ch14;

public abstract class BaseBoundedBuffer<V> {
    private final V[] buf;
    private int tail;
    private int head;
    private int count;

    protected BaseBoundedBuffer(int capacity) {
        this.buf = (V[]) new Object[capacity];
    }

    protected synchronized final void doPut(V v) {
        buf[tail] = v;
        if (++tail == buf.length)
            tail = 0;
        ++ count;
    }

    protected synchronized final V doTake() {
        V v = buf[head];
        buf[head] = null;
        if (++head == buf.length) {
            head = 0;
        }
        -- count;
        return v;
    }

    public synchronized final boolean isFull() {
        return count == buf.length;
    }

    public synchronized final boolean isEmpty() {
        return count == 0;
    }

    public abstract void put(V v) throws InterruptedException;

    public abstract V take() throws InterruptedException;
}
```

- 接下来将用各种不同的机制来扩展抽象方法`put`和`take`



##### 14.1.1 示例：直接将前提条件的失败传递给调用者



```java
package com.edu.neu.multiThreadSample.ch14;

public class GrumpyBoundedBuffer<V> extends BaseBoundedBuffer<V>{

    protected GrumpyBoundedBuffer(int capacity) {
        super(capacity);
    }

    @Override
    public synchronized void put(V v) throws InterruptedException {
        if (isFull())
            throw new IllegalStateException();
        doPut(v);
    }

    @Override
    public synchronized V take() throws InterruptedException {
        if (isEmpty())
            throw new IllegalStateException();
        return doTake();
    }
}
```

- 抛出异常意味着“对不起，请重试一次”，但这种方法并没有解决问题
- 这种方法相当于把问题扔给了调用者，调用者用起来会很麻烦



调用者代码如下

```java
while (true) {
    try {
        V item = buffer.take();
        break;
    } catch (BufferEmptyException e) {
        Thread.sleep(SLEEP_GRANULARITY);
    }
}
```



##### 14.1.2 示例：通过轮询与休眠实现简单的阻塞



```java
package com.edu.neu.multiThreadSample.ch14;

public class SleepyBoundedBuffer<V> extends BaseBoundedBuffer<V> {
    protected SleepyBoundedBuffer(int capacity) {
        super(capacity);
    }

    @Override
    public void put(V v) throws InterruptedException {
        while(true) {
            synchronized (this) {
                if (!isFull()) {
                    doPut(v);
                    return;
                }
            }
            Thread.sleep(10);
        }
    }

    @Override
    public V take() throws InterruptedException {
        while(true) {
            synchronized (this) {
                if (!isEmpty()) {
                    return doTake();
                }
            }
            Thread.sleep(10);
        }
    }
}
```

- 这种方法通过一种简单的轮询与休眠机制来实现`put`和`take`
- 休眠的间隔越小，响应度就越高，但是消耗的CPU资源也越高
- 但是这种方法会出现大量的上下文切换，而且消耗的CPU资源也很多



##### 14.1.3 条件队列 



“条件队列”这个名字来源于：

- 它使得一组线程能够通过某种方式来等待特定的条件变成真，变成真之后线程就可以再次开始运行。
- 条件队列中的元素是一个个正在等待相关条件的线程



正如每个对象可以作为一个锁，每个对象同样可以作为一个条件队列，

- 并且`Object`中的`wait`、`notify`、`notifyAll`方法就构成了内部条件队列的API。
  - 要调用对象X中条件队列的任何一个方法，必须持有对象X上的锁，否则就会在编译时抛出`IllegalMonitorStateException`
  - `Object.wait`会自动释放锁，并请求操作系统挂起当前线程，从而使其他线程能够获得这个锁并且修改对象的状态
  - 当被挂起的线程醒来后，它将尝试重新获取锁。



下面使用条件队列来实现一遍有界缓存

```java
package com.edu.neu.multiThreadSample.ch14;

public class BoundedBuffer<V> extends BaseBoundedBuffer<V> {
    protected BoundedBuffer(int capacity) {
        super(capacity);
    }

    @Override
    public synchronized void put(V v) throws InterruptedException {
        while (isFull())
            wait();
        doPut(v);
        notifyAll();
    }

    @Override
    public synchronized V take() throws InterruptedException {
        while(isEmpty())
            wait();
        V v = doTake();
        notifyAll();
        return v;
    }
}

```

- 这个代码更简单
- 更高效（当状态变量没有变化时，线程醒来的次数很少）
- 响应度更高（当发生特定状态变化时将立刻醒来）



> 如果某个功能用“轮询和休眠”无法实现，那么用条件队列同样无法实现



#### 14.2 使用条件队列



条件队列使构建高效以及可高响应性的状态依赖类变得更容易，但是同时也很容易被不正确地使用



##### 14.2.1 条件谓词



要想正确地使用条件队列，关键是找出对象在哪个条件谓词上等待。条件谓词是使得某个操作成为状态依赖操作的前提条件。



在条件等待中存在一种重要的三元关系，包括加锁、`wait`方法和一个条件谓词

1. 在条件谓词中包含多个状态变量，而状态变量由一个锁来保护，因此在测试条件谓词之前必须先持有这个锁
2. 锁对象与条件队列对象必须是同一个对象



> 每一次**`wait`调用**都会**隐式**地与特定的**条件谓词** **关联**起来。当调用某个特定条件谓词的`wait`时，调用者**必须**已经**持有**与**条件队列相关的锁**，并且这个**锁**必须**保护**着**构成条件谓词的状态变量**。



##### 14.2.2 过早唤醒



`wait`方法的返回并不一定意味着线程正在等待的条件谓词变成真

- 因为内置的条件队列极有可能与多个条件谓词同时使用（比如前面那个例子，非空和非满就是使用同样的条件队列）



因此，每当线程从`wait`中唤醒时，都必须再次测试条件谓词。由于线程在条件谓词不为真的情况下也可以反复地醒来，因此必须在一个循环中调用`wait`，并在每次迭代中都测试条件谓词

```java
void stateDependentMethod() throws InterruptedException {
    synchronized(lock) {
        while (!conditionPredicate())
            lock.wait();
    }
}
```



##### 14.2.3 丢失的信号



除了活锁和死锁外，还有一种活跃性故障叫做丢失的信号。

丢失的信号是指：线程必须等待一个已经为真的条件，但在开始等待之前没有检查条件谓词。那么就可能出现一种情况，在该线程等待之前，另一个线程就已经把条件变为真并且`notify`了，这样该线程就一直在等待。

解决方法是：在每次调用wait之前都检查一遍条件谓词



##### 14.2.4 通知



前面介绍了等待，本节介绍通知



有两种方式

- `notify`: JVM会在条件队列上等待的多个线程中选择一个来唤醒
- `nofityAll`: JVM会唤醒所有在这个条件队列上等待的线程



由于多个线程可以基于不同的条件谓词在同一个条件队列上等待，因此使用`notify`可能导致信号被劫持（线程正在等待一个已经发生过的信号）



> 只有当以下两个条件都满足时，才能使用`notify`
>
> - 所有等待线程的类型都相同：在这个条件队列上只有一种条件谓词
> - 单进单出：在条件变量上的每次通知，最多只能唤醒一个线程来执行



但是`notifyAll`性能更差，它唤醒所有线程，并且使得它们都在同一个锁上竞争。然后它们中的大多数或者全部都会回到休眠状态。因此，在每个线程执行一个事件的同时，将出现大量的上下文切换操作以及发生竞争的锁获取操作。



##### 14.2.6 子类的安全性问题



对于状态依赖的类，要么将其等待和通知等协议完全向子类公开，要么完全阻止子类参与到等待和通知等过程中。



#### 14.3 显式的Condition对象



正如`Lock`是一种广义的内置锁，`Condition`也是一种广义的内置条件队列。

```java
public interface Condition {
    void await() throws InterruptedException;
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    void awaitUninterruptibly();
    boolean awaitUntil(Date deadline) throws InterruptedException;
    void signal();
    void signalAll();
}
```



内置条件队列存在一些缺陷，每个内置锁都只能有一个相关联的条件队列。如果想编写带有多个条件谓词的并发对象，那么就可以使用显式的`Lock`和`Condition`。



一个`Condition`需要与一个`Lock`相关联，就像一个条件队列和一个内置锁相关联一样。要创建一个`Condition`，可以在相关联的`Lock`上调用`Lock.newCondition`方法。



`Condition`提供了更丰富的功能

- 在每个锁上可以存在多个条件队列
- 条件等待可以是可中断的
- 可以实现基于时限的等待
- 可以实现非公平或公平的队列操作



下面将前面的例子改成条件队列的版本

```java
package com.edu.neu.multiThreadSample.ch14;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class ConditionBoundedBuffer<T>{
    private static final int BUFFER_SIZE = 10;
    protected final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    private final T[] items = (T[]) new Object[BUFFER_SIZE];
    private int tail,head,count;
    
    
    public void put(T t) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await();
            }
            items[tail] = t;
            if (++tail == items.length) {
                notEmpty.signal();
            }
        } finally {
            lock.unlock();
        }
    }
    
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)
                notEmpty.await();
            T t = items[head];
            items[head] = null;
            if (++head == items.length)
                head = 0;
            --count;
            notFull.signal();
            return t;
        } finally {
            lock.unlock();
        }
    }
}
```

- 使用两个条件变量更加清晰
- 而且这样就能使用`signal`，它比`signalAll`更高效



#### 14.4 Synchronizer剖析



在`ReentrantLock`和`Semaphore`这两个接口之前存在很多共同点

1. 这两个类都已用做一个“阀门”，即每次都只允许一定数量的线程通过
2. 当线程到达“阀门”时，可以通过，可以等待，还可以取消



我们可以使用`ReentrantLock`来实现`Semaphore`，也可以通过`Semaphore`来实现`ReentrantLock`。



事实上，两者在实现时使用了一个共同的基类：`AbstractQueuedSynchronizer(AQS)`，这个类也是其他许多同步类的基类。



#### 14.5 AbstractQueueSynchronizer



大多数开发者都不会直接使用`AQS`，标准同步器类的集合能够满足绝大多数情况的需求。



在基于AQS构建的同步器类中，最基本的操作包括各种形式的获取操作和释放操作

1. 获取操作是一种依赖状态的操作，并且通常会阻塞
2. 释放并不是一个可阻塞的操作，当执行“释放”操作时，所有在请求时被阻塞的线程都会开始执行。



AQS还负责管理同步器类中的状态，它管理要给整数状态信息，可以通过`getState`、`setState`以及`compareAndSetState`等`protected`类型方法来进行操作。



下面代码给出了AQS中的获取操作和释放操作的形式

```
boolean acquire() throws InterruptedException {
	while (当前状态不允许获取操作) {
		if (需要阻塞获取请求){
			如果当前线程不在队列中，将它插入队列
			阻塞当前线程
		}else
			返回失败
		可能更新同步器的状态
        如果线程位于队列中，则将其移出队列
        返回成功
	}
}

void release() {
	更新同步器的状态
	if (新的状态允许某个被阻塞的线程获取成功)
		解除队列中一个或多个线程的阻塞状态
}
```

- 如果某个同步器支持独占的获取操作，那么需要实现一些保护方法，包括`tryAcquire`、`tryRelease`、`isHeldExclusively`等
- 对于支持共享的同步器则应该实现`tryAcquireShared`、`tryReleaseShared`等方法
- 在同步器的子类中，可以根据其获取操作和释放操作的语义，使用`getState`、`setState`以及`compareAndSetState`来检查和更新状态，并通过返回的状态值告诉基类“获取”或是“释放”同步器操作是否成功。例如，如果`tryAcquireShared`返回一个负值，那么表示操作失败，返回零值表示同步器通过独占方式被获取，返回正值表示同步器通过非独占方式被获取。



下面举一个使用AQS自定义同步器的例子，它实现了要给简单的二元闭锁

```java
public class OneShotLatch {
    private final Sync sync = new Sync();
    public void signal() {sync.releaseShared(0);}
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(0);
    }
    private class Sync extends AbstractQueuedSynchronizer {
        // 获取
        protected int tryAcquireShared(int ignored) {
            // 如果latch被open过，那么这个操作将成功
            return (getState() == 1) ? 1 : -1 
        }
        // 释放(开锁)
        protected boolean tryReleaseShared(int ignored) {
            setState(1); // 现在打开闭锁
            return true; // 现在其他线程可以获取这个闭锁了
        }
    }
}
```

- 它没有直接扩展AQS，而是使用一个内部类扩展AQS





#### 14.6 JUC同步器类中的AQS（略）

1. ReentrantLock只支持独占操作,它实现`tryAcquire`、`tryRelease`和`isHeldExclusively`
2. Semaphore 和 CountDownLatch
3. FutureTask 不像一个同步器,AQS同步状态用来保存任务状态
4. ReentrantReadWriteLock 使用一个16位状态表示读锁的计数,AQS内部维护一个等待队列,记录某个线程独占访问还是共享访问



> 关于AQS的部分还是应该看《JAVA并发编程的艺术》



### 第15章 原子变量与非阻塞同步机制



本章主要介绍原子变量和非阻塞的同步机制



非阻塞算法是指通过底层的原子机器指令代替锁来确保数据在并发访问中的一致性。非阻塞算法被广泛用于在操作系统和JVM中实现线程调度、进程调度、垃圾回收、锁等



非阻塞算法

- 不会造成线程阻塞
- 也不会造成死锁



原子变量可以看作一种更好的`volatile`变量，它提供了与`volatile`变量相同的内存语义，并且还支持原子的更新操作。



#### 15.1 锁的劣势



JVM对非竞争锁获取和释放进行了优化，但是如果有多个线程同时请求锁，就要借助操作系统，这开销很大

volatile是更轻量的同步机制，但是它只能提供可见性，不能提供原子性



#### 15.2 硬件对并发的支持



独占锁是悲观锁,总是假设最坏情况,只有确保其他线程不会干扰才能执行



对于一些细粒度的操作（操作中没有几条指令，很快就能执行完），可以采用一种**乐观**的方法：这种方法需要先检测在更新过程中是否有其他线程的干扰，如果存在，这个操作将失败，但是可以重试。



几乎所有的现代处理器中都包含了某种形式的原子读-改-写指令，例如比较并交换（CAS，Compare-and-Swap）或者关联加载/条件存储(Load-Linked/Store-Conditional)。操作系统和JVM使用这些指令来实现锁和并发的数据结构。



##### 15.2.1 比较并交换



CAS包含了3个操作数

- 需要读写的内存位置V
- 进行比较的值A
- 拟写入的新值B
- 当且仅当V的值等于A时，CAS才会采用原子方式用新值B来更新V的值，否则不会执行任何操作。无论位置V的值是否等于A，都将返回V原有的值



当多个线程尝试使用CAS同时更新同一个变量时，只有其中一个线程能够更新变量的值，而其他线程都将失败。然而，失败的线程不会挂起，而是被告知在这次竞争中失败，并可以再次尝试。



##### 15.2.2 非阻塞的计数器



基于CAS的计数器，比如java的`AtomicInteger`类，即使在最坏的情况下也比基于锁的计数器执行更少的操作。



虽然Java语言的加锁解锁语法很简洁，但是JVM在管理锁时需要完成的工作却不简单。

- 在实现锁定时需要遍历JVM中一条十分复杂的代码路径，并且可能导致操作系统级的锁定、线程挂起以及上下文切换等操作
- 在最好的情况下，在锁定时至少需要一次CAS，因此虽然在使用锁时没有用到CAS，但实际上也无法节约任何执行开销
- 而如果在程序内部执行CAS，就不需要执行JVM的代码、系统调用等。



CPU厂商迫于竞争压力，一定会提高多核处理器中CAS指令的性能。



##### 15.2.3 JVM对CAS的支持



在Java中通过Unsafe类提供了对`int`、`long`和对象引用等类型的CAS操作支持，并且JVM会把它编译成底层硬件提供的最有效方式。

- 在支持CAS的平台上，运行时将它们编译为相应的多条机器指令
- 在最坏的情况下，如果不支持CAS指令，那么JVM将使用自旋锁



在原子变量类中(在`java.util.concurrent.atomic`中的`AtomicXxx`)中使用了这些底层的JVM支持，来为数字类型和引用类型提供一种高效的CAS操作。而在`java.util.concurrent`中的大多数类在实现时则直接或间接地使用了这些原子变量类。



#### 15.3 原子变量类



原子变量比锁的粒度更细、更轻量，它将发生竞争的范围缩小到单个变量上（假设算法能够基于这种细粒度实现）。



在使用基于原子变量而非锁的算法中，线程在执行时更不容易发生延迟，如果遇到竞争，更加容易恢复。



原子变量类相当于**一种泛化的volatile变量**，能够支持原子的和有条件的读-改-写操作，在发生竞争的情况下伸缩性更好，因为直接利用了硬件对并发地支持。



共有12个原子变量类，可分为4组

- 标量类：`AtomicInteger`、`AtomicLong`、`AtomicBoolean`、`AtomicReference`，这些类都支持CAS操作
- 更新器类
- 原子数组类：例如`AtomicIntegerArray`、`AtomicLongArray`、`AtomicReferenceArray`。它提供了对数组中元素的原子更新。在功能上`AtomicIntegerArray`与`AtomicInteger[]`等价，但是在实现上有区别，`AtomicIntegerArray`直接使用`Unsafe`类中提供的各种操作内存的方法以及`compareAndSetInt`这种CAS方法。
- 复合变量类



##### 15.3.1 原子变量是一种更好的volatile



```java
public class CasNumberRange {
    @Immutable
    private static class IntPair {
        final int lower; // 不变性条件：lower <= upper
        final int upper;
    }
    private final AtomicReference<IntPair> values = new AtomicReference<IntPair>(new IntPair(0,0));
    
    public int getLower() {return values.get().lower;}
    public int getUpper() {return values.get().upper;}
    
    public void setLower(int i) {
        while (true) {
            IntPair oldv = values.get();
            if (i > oldv.upper)
                throw new IllegalArgumentException();
            IntPair newv = new IntPair(i, oldv.upper);
            if (values.compareAndSet(oldv, newv))
                return;
        }
    }
}
```

- 这个算法使用了不可变的`IntPair`
- 外加一个原子的引用来控制`NumberRange`的更新，避免了race condition



##### 15.3.2 性能比较：锁与原子变量



![](https://healthlung.oss-cn-beijing.aliyuncs.com/20211013195735.png)



如图，在高度竞争的情况下，锁的性能将超过原子变量的性能。（因为锁在竞争时会挂起线程，从而降低CPU的使用率和共享内存总线上的同步通信量，而原子变量在竞争失败时会重试，于是激烈的竞争引发了更激烈的竞争。）



在实际使用情况（竞争程度适中）中，原子变量的性能要比锁好一些



#### 15.4 非阻塞算法



**非阻塞算法**：如果在某种算法中,一个线程的失败或者挂起不会导致其他线程也失败或挂起,那么这种算法称为非阻塞算法

**无锁算法**：如果在算法的每个步骤都存在某个线程能够执行下去,这种算法称为无锁算法（没看懂但不重要）

1. 

2. 非阻塞的链表 Michale-scott算法

3. 原子的域更新器

   表示现有volatile域的一种基于反射的视图,能够在已有的volatile域上使用cas

4. ABA问题

   取到V值以后测试等于A,在写入之前,其他线程偷偷改了B,再改回A.适用版本号可以解决



##### 15.4.1 非阻塞的栈



创建非阻塞算法的关键在于，找出如何将原子修改的范围缩小到单个变量上，同时还要维护数据的一致性。



栈可以将原子修改缩小到头结点的引用上，代码如下

```java
@ThreadSafe
public class ConcurrentStack<E> {
    AtomicReference<Node<E>> top = new AtomicReference<Node<E>>();
    
    public void push (E item) {
        Node<E> newNode = new Node<>(item);
        Node<E> oldHead;
        do {
            oldHead = top.get();
            newHead.next = oldHead;
        } while (!top.compareAndSet(oldHead, newHead));
    }
    
    public E pop() {
        Node<E> oldHead;
        Node<E> newHead;
        do {
            oldHead = top.get();
            if (oldHead == null)
                return null;
            newHead = oldHead.next;
        // CAS，判断top还是不是oldHead，如果是的话就将top设置为newHead    
        } while (!top.compareAndSet(oldHead, newHead));
        return oldHead.item;
    }
    
    private static class Node<E> {
        public final E item;
        public Node<E> next;
        
        public Node(E item) {
            this.item = item;
        }
    }
}
```

- `compareAndSet`既能提供原子性，又能提供可见性。当一个线程需要改变栈的状态时，将调用`compareAndSet`，这个方法与写入`volatile`类型的变量有着相同的内存效果
- 当线程需要检查栈的状态时3，可以调用`get`方法，这个方法与读取`volatile`类型的变量有相同的内存效果



##### 15.4.2 非阻塞的链式队列



链式队列更加复杂：它需要维护头结点和尾节点，因此有两个指针指向队尾元素：当前最后一个元素的next指针以及尾节点。当插入一个元素时，这两个指针都需要用原子操作来更新。



因此我们需要一些技巧，直接看代码

```java
@ThreadSafe
public class LinkedQueue<E> {
    private static class Node<E> {
        final E item;
        final AtomicReference<Node<E>> next;
        public Node(E item, Node<E> next) {
            this.item = item;
            this.next = new AtomicReference<Node<E>>(next);
        }
    }
    private final Node<E> dummy = new Node<E>(null, null);
    private final AtomicReference<Node<E>> head = new AtomicReference<Node<E>>(dummy);
    private final AtomicReference<Node<E>> tail = new AtomicReference<Node<E>>(dummy);
    public boolean put(E item) {
        Node<E> newNode = new Node<E>(item, null);
        while (true) {
            Node<E> curTail = tail.get();
            Node<E> tailNext = curTail.next.get();
            if (curTail == tail.get()){
                // 如果尾节点的指针还没有往后指，就先帮它往后指
                if (tailNext != null) {
                    tail.compareAndSet(curTail, tailNext);
                // 如果当前一切正常    
                } else {
                    // 先插入新节点
                	if (curTail.next.compareAndSet(null, newNode)) {
                    	// 再将尾节点往后指，就算失败了，也提前占了位置，就算失败了，这个方法也会return，但是无所谓，其他线程会帮忙把尾节点的指针修正。
                        tail.compareAndSet(curTail, newNode);
                    	return true;
                    }
                }
            }
        }
    }
}
```

- 如果当B到达时发现A正在修改数据结构，那么在数据结构中应该有足够的信息，使得B能够帮助A完成A还没完成的更新操作
- 如果B帮助A完成了更新操作，那么B可以开始执行自己的操作，而不需要等待A的操作完成
- 当A恢复后再试图完成其操作时，会发现B已经替它完成了



> 具体可以看Michael-Scott算法



##### 15.4.3 原子的域更新器



原子的域更新器类表示现有volatile域的一种基于反射的视图，从而使得在已有的volatile域上使用CAS。

它可以避免我们为大量相同类型的对象创建`AtomicReferce`，因此提升了性能

```java
private static class Node<E> {
    public final E item;
    public Node<E> next;

    public Node(E item) {
        this.item = item;
    }
}
private static AtomicReferenceFieldUpdater<Node, Node> updater = 
    AtomicReferenceUpdater.newUpdater(Node.class, Node.class, "next");
```



##### 15.4.4 ABA问题



如果V的值首先从A变成B，然后再从B变成A，那么普通的原子类检测不出这种变化。



这个时候我们需要一个自带版本号的原子类，它会同时更新值和一个版本号，比如`AtomicStampedReference`，而`AtomicMarkableReference`可以在更新的同时更新一个`boolean`类型的变量。



### 第16章 Java内存模型



本章将介绍Java内存模型的底层需求以及所提供的保证，此外还将介绍在本书中给出的一些搞成设计原则背后的原理



#### 16.1 什么是内存模型



如果缺少同步，那么将有很多因素使得线程无法立即甚至永远，看到另一个线程的操作结果

1. 在编译器中生成的指令顺序，可以与源代码中的顺序不同
2. 编译器还会把变量保存在寄存器而不是内存
3. 处理器可以采用乱序或并行等方式来执行指令
4. 缓存可能会改变将写入变量提交到主内存的次序
5. 保存在处理器本地缓存中的值，对其他处理器是不可见的



> 只要程序的最终执行结果与严格串行环境中执行的结果相同，那么上述这些操作都是允许的。



**计算性能的提升**在很大程度上要归功于这些**重新排序**措施



在**多线程环境**中**，维护程序的串行性**将导致很大的**开销**。只有当多个线程需要共享数据时，才必须协调它们之间的操作，并且**JVM** **依赖** **程序** 通过**同步操作**来**找出**这些**协调操作将在什么地方发生**。



##### 16.1.1 平台的内存模型



在共享内存的多处理器体系结构中，**每个处理器**都拥有自己的**缓存**，并且定期地与**主内存**进行协调。平台可能只能提供最小的保证：允许不同的处理器在任意时刻从同一个存储位置上看到不同的值。而**操作系统**、**编译器**以及**运行时**需要**弥合**这种在**硬件能力与线程安全需求**中的差距。



为了使Java开发人员无须关心不同架构上的内存模型之间的差异，Java还提供了自己的内存模型，并且JVM通过在合适的位置上插入内存栅栏来屏蔽在JMM与底层平台内存模型之间的差异。



**串行一致性**：

- 程序只存在唯一的操作执行顺序，并且在每次读取变量时，都能获得在执行顺序中最近一次写入该变量的值。
- 这种一致性太过严格，Java内存模型也不会提供这种一致性



##### 16.1.2 重排序



在没有正确同步的情况下，即使要推测最简单的并发程序的行为也是非常困难的。如下代码所示

```java
public class PossibleReordering {
    static int x = 0, y = 0;
    static int a = 0, b = 0;
	
    public static void main(String[] args) throws InterruptedException{
        Thread one = new Thread(new Runnable(){
            public void run() {
                a = 1;
                x = b;
            }
        });
        Thread other = new Thread(new Runnable(){
           public void run() {
               b = 2;
               y = a;
           } 
        });
    }
}
```

- 这个代码会有四种执行结果
- 因为重排序机制，因此`a=1`、`x=b`、`b=2`、`y=a`这些操作可以乱序执行



同步将限制编译器、运行时和硬件对内存操作的重排序，从而在实施重排序时不会破坏JMM提供的可见性保证。



##### 16.1.3 Java内存模型简介



JMM为程序中的所有操作定义了一个偏序关系，叫做Happens-Before。要想保证执行操作B的线程能够看到操作A的结果，那么A与B之间必须满足Happens-Before关系。如果两个操作之间缺乏Happens-Before关系，那么JVM可以对它们任意地重排序。



**Happens-Before规则**包括

**单线程程序顺序规则** 在代码中如果A操作在B操作之前,那么单线程中，A操作将在B操作之前（从运行结果上看是这样的，如果操作之间没有数据依赖关系，也有可能重排序，毕竟结果一样）

 **监视器锁规则** 在监视器锁上的解锁操作必须在同一个监视器锁的加锁操作之前执行

 **volatile变量规则** 对一个volatile域的写，happens-before于任意后续对这个volatile域的读

 **线程启动规则** 线程上对Thread.start的调用必须在该线程中执行任何操作之前执行

 **线程结束规则** 线程中任何操作必须在其他线程检测到该线程已经结束之前执行或者从Thread.join返回或者调用Thread.isAlive返回false

 **中断规则** 当一个线程在另一个线程上调用interrupt,必须在被中断县城检测到interrupt调用之前执行

 **终结器规则**对象的构造函数必须在启动该对象的终结器之前执行完成

 **传递性** 操作A在操作B之前执行,B在C之前,那么A在C之前



下面这个例子说明，FutureTask如何借助AQS中加锁解锁的HappensBefore规则来实现`result`的可见性

```java
private final class Sync extends AbstractQueuedSynchronizer {
    private static final int RUNNING = 1, RAN = 2, CANCELLED = 4;
    private V result;
    private Exception exception;
    
    void innerSet(V v) {
        while (true) {
            int s = getState();
            if (ranOrCancelled(s))
                return;
            if (compareAndSetState(s, RAN))
                break;
        }
        result = v;
        releaseShared(0);
        done();
    }
    
    // 通过releaseShared和acquireSharedInterruptibly之间的happens-before规则，可以保证innerSet写入的result能被之后的innerGet看到
    V innerGet() throws InterruptedException, ExecutionException {
        acquireSharedInterruptibly(0);
        if (getState() == CANCELLED)
            throw new CancellationException();
        if (exception != null)
            throw new ExecutionException(exception);
        return result;
    }
}
```

- 这种技术被称为借助：它使用一种现有的Happens-Before顺序来确保对象X的可见性，而不是专门为了发布X而创建一种Happens-Before顺序。



在类库中提供的其他Happens-Before规则包括：

- 将一个元素放入一个线程安全容器的操作将在另一个线程从该容器中获得这个元素的操作之前执行
- 在`CountDownLatch`上的倒数操作将在线程从闭锁上的`await`方法中返回之前执行
- 释放`Semaphore`许可的操作将在从该`Semaphore`上获得一个许可之前执行
- `Future`表示的任务的所有操作将在从`Future.get`中返回之前执行
- 向`Executor`提交一个`Runnable`或`Callable`的操作将在任务开始执行之前执行





#### 16.2 发布



##### 16.2.1 不安全的发布

当缺少Happens-Before关系时候,就可能出现重排序问题。这就解释了为什么没有同步情况下发布一个对象会导致另一个线程看到一个只被部分构造的对象

>  除了不可变对象以外,使用被另一个线程初始化对象通常都是不安全的，除非对象发布操作在使用该对象的线程之前执行



##### 16.2.3 安全初始化模式



第一种：创建对象的getInstance直接加锁(假如竞争不激烈,性能不会差)

```java
@ThreadSafe
public class SafeLazyInitialization {
    private static Resource resource;
    public synchronized static Resource getInstance() {
        if (resource == null)
            resource = new Resource();
        return resource;
    }
}
```



静态初始化器是由jvm在类初始化阶段执行，并获得一个锁（类的锁）,每个线程至少获取一次这个锁以确保类一被加载,因此静态代码段中初始化的field,对所有线程可见



由此，我们得到第二种方法，提前初始化

```java
@ThreadSafe
public class EagerInitialization {
    private static Resource resource = new Resource();
    public static Resource getResource() {return resource;}
}
```



由于JVM只有在用到一个类的时候才会对它进行加载，因此我们得到第三种方法，延长初始化占位符模式

```java
@ThreadSafe
public class ResourceFactory {
    private static class ResourceHolder {
        // 可以保证在类加载过程中进行new Resource()
        public static Resource resource = new Resource();
    }
    // 可以保证用到ResourceHolder时再进行类加载
    public static Resource getResource() {
        return ResourceHolder.resource;
    }
}
```



16.2.4 双重检测加锁

```java
@NotThreadSafe
public class DoubleCheckedLocking {
    private static Resource resource;
    public static Resource getInstance() {
        if (resouce == null) {
            synchronized (DoubleCheckedLocking.class) {
                if (resource == null)
                    resource = new Resource();
            }
        }
    }
}
```

- 因为内存重排序机制，线程可能会看到一个被部分构造的Resource
- 但是如果给`resource`声明为`volatile`类型，那么就是对的，并且这种方式对性能的影响很小。
- 有了volatile，就能保证 对一个volatile域的写，happens-before于任意后续对这个volatile域的读。这样就能保证在读之前，写是全写完了的。



#### 16.3 初始化过程的安全



对于含有final域的对象，**初始化安全性**能够保证**在对象引用指向对象之前**，**所有的final域都被初始化完成**，从而保证了安全性。

