---
layout: post
comments: true
title: "Java并发编程实战——基础知识"
description: "Java并发编程实战——基础知识"
category: Java
tags: [Java]
---

学姐最近在重读Java并发编程实战这本书。本文是关于第一部分的知识点总结。

主要涉及如下知识点：

- 线程安全性（无状态对象）
- 原子性（原子操作、竞态条件）
- 加锁机制（内置锁、重入）
- 可见性（volatile、加锁）
- 发布与逸出（发布、逸出、安全发布）
- 线程封闭（栈封闭、ThreadLocal）
- 不变性（final、不可变对象、事实不可变对象）
- 基础构建模块（同步容器、并发容器、同步工具类－包括闭锁、FutureTask、信号量、栅栏）

<!--more-->

## 概念梳理：

#### 1、线程安全性

当多个线程访问某个类时，不管运行时环境采用何种调度方式或者这些线程将如何交替执行，并且在主调代码中不需要任何额外的同步或协同，这个类就能表现出正确的行为，那么就称这个类是线程安全的。

**无状态对象**：

既不包含任何域，也不包含任何其他类中域的引用的对象。

无状态对象一定是线程安全的。

#### 2、原子性
假定有两个操作A和B，如果从执行A的线程来看，当另一个线程执行B时，要么将B全部执行完，要么完全不执行B，那么A和B对彼此来说是原子的。

**原子操作**：

对于访问同一个状态的所有操作，以原子方式执行。

**竞态条件**：

当某个计算的正确性取决于多个线程的交替执行时序时，就会发生竞态条件。常见场景`先检查后执行（check-then-act）`

#### 3、加锁机制

**内置锁**：

每个Java对象都可以用做一个实现同步的锁，这些锁称为内置锁，用synchronized表示。
进入同步代码块时自动获得锁，退出同步代码块时自动释放锁。

**重入**：

如果某个线程试图获得一个已经由它自己持有的锁，那么这个请求就会成功。

重入意味着获取锁的操作粒度是“线程”，而不是“调用”。

实现方式：为每个锁关联一个`获取计数值`和一个`所有者线程`。当计数值为0时，这个锁被认为没有被任何线程持有。当线程请求一个未被持有的锁时，JVM将记下锁的持有者，并且将获取计数值置为1。如果同一个线程再次获取这个锁，计数值将递增，而当线程退出同步代码块时，计数值减1。

内置锁是可重入的。

对于可能被多个线程同时访问的可变状态变量，在访问它时都需要持有同一个锁，在这种情况下，我们称状态变量是由这个锁保护的。    
同时使用两种不同的同步机制会带来混乱。（如内置锁synchronized和Atomic原子变量）    
将同步代码块分解得过细并不好，因为获取与释放锁操作需要开销。    
当执行时间较长的计算或者可能无法快速完成的操作时（如I/O），不要持有锁。    

#### 4、可见性

`指令重排`、`volatile`都会影响内存可见性。

**非原子的64位操作**：

对于long和double变量，JVM将64位的读写操作分解为两个32位的操作。    
如果对该变量的读和写操作在不同的线程中执行，那么很可能会读取到某个值的高32位和另一个值的低32位。    
通过volatile或加锁，可以保证安全性。    

**volatile**：

volatile变量没有重排序。    
volatile变量只能确保可见性，不会存储在寄存器或对其他处理器不可见的地方。    

非原子的64位操作    
对于非volatile类型的long和double变量，JVM允许将64位的读或写操作分解为两个32位操作。    
当读取一个非volatile类型的long变量时，如果对该变量的读操作和写操作在不同的线程中执行，那么很可能会读取到某个值的高32位和另一个值的低32位。    
为了保证原子性，可以用volatile或锁保护起来。    

volatile变量的正确使用方式：    

- 确保它们自身状态的可见性
- 确保引用对象状态的可见性
- 标识一些重要的程序生命周期事件的发生，如初始化或关闭

当且仅当满足以下所有条件时，才应该使用volatile变量：    

- 对变量的写入操作不依赖变量的当前值，或者你能确保只有单个线程更新变量的值
- 该变量不会与其他变量一起纳入不变性条件中
- 在访问变量时不需要加锁

**加锁**    

加锁同时确保原子性和可见性。

#### 5、发布与逸出

**发布**

“发布”一个对象的意思是指，使对象能够在当前作用域之外的代码中使用。

常见场景：

- 将一个指向该对象的引用保存到其他代码可以访问的地方
- 在某一个非私有的方法中返回该引用
- 将引用传递到其他类的方法中

发布一个对象时，会发布该对象非私有域中引用的对象。    
发布一个集合时，会发布集合中的元素。    
发布内部类实例时，隐含地发布了外部类实例    

**逸出**

逸出是指，当某个不应该发布的对象被发布时。

**安全发布**

安全发布的常用模式：
- 在静态初始化函数中初始化一个对象引用
- 将对象的引用保存到volatile类型的域或者AtomicReference对象中
- 将对象的引用保存到某个正确构造对象的final类型域中
- 将对象的引用保存到一个由锁保护的域中

安全发布场景1：    
`public static Holder holder = new Holder(42);`    
静态初始化器，由JVM在类的初始化阶段执行。由于在JVM内部存在着同步机制，通过这种方式初始化的任何对象都可以被安全地发布。

安全发布场景2：    
将对象的引用保存到一个由锁保护的域中

- Vector、CopyOnWriteArrayList、CopyOnWriteArraySet、synchronizedList或synchronizedSet
- Hashtable、synchronizedMap、ConcurrentMap
- BlockingQueue、ConcurrentLinkedQueue
- Future、Exchanger

对象的发布需求取决于它的可变性：

- 不可变对象可以通过任意机制来发布
- 事实不可变对象必须通过安全方式来发布
- 可变对象必须通过安全方式来发布，并且必须是线程安全的或者由某个锁保护起来

#### 6、线程封闭

当某个对象封闭在一个线程中时，这种用法将自动实现线程安全性，即使被封闭的对象本身不是线程安全的。
常用线程封闭技术：栈封闭、ThreadLocal

**Ad-hoc线程封闭**    
Ad-hoc线程封闭是指，维护线程封闭的职责完全由程序实现来承担。由于其脆弱性，应尽量少用。

**栈封闭**：    
在栈封闭中，只能通过局部变量才能访问对象。局部变量的属性之一就是封闭在执行线程中。

**ThreadLocal**：    
ThreadLocal提供了get/set方法，这些方法为每个使用该变量的线程都存有一份独立的副本，因此get总是返回由当前执行线程在调用set时设置的最新值。

ThreadLocal使用场景：

- 用于防止对可变的单实例变量或全局变量进行共享
- 当某个频繁执行的操作需要一个临时对象，例如缓冲区，而又希望避免在每次执行时都重新分配该临时对象。

#### 7、不变性

**不可变对象**：

如果某个对象在被创建后其状态就不能被修改，则称该对象为不可变对象。

不可变对象一定是线程安全的。

**final**：

final类型的域是不能修改的。    
final类型的引用不能被修改，但是引用的对象其内容可以被修改。    
因此即使对象中所有的域都是final类型的，这个对象也仍然是可变的，因为在final类型的域中可以保存对可变对象的引用。

当满足以下条件时，对象才是不可变的：

- 对象创建以后其状态就不能修改
- 对象的所有域都是final类型
- 对象是正确创建的（在对象的创建期间，this引用没有逸出）

除非需要某个域是可变的，否则应将其声明为final域

每当需要对一组相关数据以原子方式执行某个操作时，就可以考虑创建一个不可变的类来包含这些数据。

**事实不可变对象**：

事实不可变对象是指，对象从技术上来看是可变的，但其状态在发布后不会再改变。

在没有额外的同步的情况下，任何线程都可以安全地使用被安全发布的事实不可变对象

	public Map<String, Date> lastLogin = Collections.synchronizedMap(new HashMap<String, Date>());

#### 8、基础构建模块

**同步容器**

同步容器类有Vector、Hashtable和Collections.synchronizedXxx等工厂方法创建的类。    
同步容器类都是线程安全的，但是在同步容器上的复合操作是不安全的，如迭代、条件运算（若没有则添加、先检查再运行）。此时需要加锁。

同步容器类通过其自身的锁来保护它的每一个方法。

设计同步容器类的迭代器时，并没有考虑并发修改问题，表现出的行为时`“及时失败”（fail-fast）`，会抛出ConcurrentModificationException。

同步容器可伸缩性低。

**并发容器**

如`ConcurrentHashMap`：    
采用粒度更细的加锁机制——`分段锁`    
任意数量的读取线程可以并发地访问Map，并发环境下实现更高的吞吐量，单线程环境下只损失很小的性能。

其迭代器具有弱一致性，并非“及时失败”，不需要额外加锁。

如`CopyOnWriteArrayList`：    
写入时复制，只要正确地发布一个事实不可变的对象，那么在访问该对象时就不再需要进一步的同步。每次修改时，都会创建并重新发布一个新的容器副本，从而实现可变性。

由于其迭代器保留一个指向底层基础数组的引用，每当修改容器时都会复制底层数组，存在一定开销。因此仅当修改操作较少时，才使用该并发容器。

**生产者－消费者模式**

`阻塞队列`，如果队列已经满了，那么put方法将阻塞直到有空间可用；如果队列为空，那么take方法将会阻塞直到有元素可用。

`生产者－消费者`，将“找出需要完成的工作”与“执行工作”两个过程分开，并把工作放入一个“待完成”列表中，以便随后处理。如Excecutor。

阻塞队列支持 生产者－消费者 模式。

BlockingQueue的几种实现：    
LinkedBlockingQueue、ArrayBlockingQueue、PriorityBlockingQueue、SynchronousQueue。

`双端队列`     
Deque、BlockingDeque 实现在队列头和队列尾的高效插入和移除。    
具体实现：ArrayDeque、LinkedBlockingDeque

双端队列 支持 工作密取

`工作密取`    
每个消费者都有各自的双端队列，如果一个消费者完成了自己双端队列中的全部工作，那么它可以从其他消费者双端队列末尾秘密地获取工作。    
工作密取适用于既是消费者也是生产者问题——当执行某个工作时可能导致出现更多工作，如网页爬虫、搜索图算法、垃圾回收标记。

**阻塞和中断**    
`阻塞`    
阻塞原因：等待I/O操作结束，等待获得一个锁，等待从Thread.sleep方法中醒来，等待另一个线程的计算结果。

`中断`    
中断是一种协作机制。一个线程不能强制其他线程停止正在执行的操作而去执行其他操作。    
当线程A中断B时，A仅仅是要求B在执行到某个可以暂停的地方停止正在执行的操作——前提是如果线程B愿意停下来。    
Thread.interrupt()，用于中断线程或查询线程是否已经被中断。    

中断处理：

- `传递InterruptedException`，不捕获异常，或捕获该异常然后将异常抛出给方法调用者
- `恢复中断`，捕获该异常，并通过调用Thread.currentThread().interrupt()恢复中断状态，从而暴露给更高层。

例子：

	public class TaskRunnable implements Runnable {
		BlockingQueue<Task> queue;
		...
		public void run() {
			try {
				processTask(queue.take());
			} catch(InterruptedException e) {
				//恢复被中断的状态
				Thread.currentThread().interrupt();
			}
		}
	}

**同步工具类**

`闭锁`    
闭锁的作用相当于一扇门：在闭锁到达结束状态之前，这扇门一直是关闭的，并且没有任何线程能通过，当到达结束状态时，这扇门会打开并允许所有的线程通过。当闭锁到达结束状态后，将不再改变状态，因此这扇门将永远保持打开状态。

闭锁用来确保某些活动直到其他活动都完成后才继续执行：

- 确保某个计算在其需要的所有资源都被初始化之后才继续执行
- 确保某个服务在其依赖的所有其他服务都已经启动之后才启动
- 等待直到某个操作的所有参与者都就绪再继续执行

如CountDownLatch    
闭锁状态计数器初始化为一个正数，表示需要等待的事件数量。countDown()递减计数器，表示有一个事件已经发生了。await()等待计数器达到0，表示所有需要等待的事件都已经发生。

例子——计时测试：

	public long timeTasks(int nThreads, final Runnable task) throws InterruptedException {
		final CountDownLatch startGate = new CountDownLatch(1);
		final CountDownLatch endGate = new CountDownLatch(nThreads);
		
		for(int i = 0; i < nThreads; i++) {
			Thread t = new Thread() {
				public void run() {
					try {
						startGate.await();
						try {
							task.run();
						} finally {
							endGate.countDown();
						}
					} catch(InterruptedException e) {
					}
				}
			};
			t.start();
		}

		long start = System.nanoTime();
		startGate.countDown();
		endGate.await();
		long end = System.nanoTime();
		return end - start;
	}

`FutureTask`    
FutureTask也可以用做闭锁，通过Callable来实现，表示一种抽象的可生成结果的Runnable。    
3种状态：等待运行、正在运行、运行完成    

FutureTask.get()：    
若任务已完成，则get立即返回结果；否则get将阻塞直到任务进入完成状态，然后返回结果或者抛出异常。

`信号量`    
技术信号量用来控制同时访问某个特定资源的操作数量，或者同时执行某个指定操作的数量。    
Semaphore中管理着一组虚拟的许可，许可的初始数量可通过构造函数来指定。在执行操作时可以首先获得许可，并在使用以后释放许可。如果没有许可，那么acquire将阻塞直到有许可（或者直到被中断或操作超时），release将返回一个许可给信号量。

例子——为容器设置边界：
	
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

`栅栏`    
栅栏（Barrier）类似于闭锁，它能阻塞一组线程直到某个事件发生。

栅栏与闭锁的区别：

- 所有线程必须同时到达栅栏位置，才能继续执行。
- 闭锁用于等待事件，栅栏用于等待其他线程

CyclicBarrier    
将一个问题拆分成一系列独立的子问题。    
当线程到达栅栏位置时将调用await方法，这个方法将阻塞直到所有线程都到达栅栏位置。    
如果对await的调用超时或者await阻塞饿线程被中断，那么栅栏就被认为是打破了，所有阻塞的await调用都将终止并抛出BrokenBarrierException

另一种形式的栅栏是Exchanger。

例子略。

### 栗子分析

1、线程安全性

无状态对象一定是线程安全的。

	public clas StatelessFactorizer implements Servlet {
		public void service(ServletRequest req, ServletResponse resp) {
			BigInteger i = extractFromRequest(req);
			BigInteger[] factors = factor(i);
			encodeIntoResponse(resp, factors);
		}
	}

	
2、原子性

（1）非原子操作：

	++count；
包含3个操作：读取－修改－写入。读取count的值；将值加1；将计算结果写入count。非原子操作存在竞态条件。
（2）原子操作：

	private final AtomicLong count ＝ new AtomicLong(0);
	count.incrementAndGet();

3、加锁机制

	public class UnsafeCachingFactorizer implements Servlet{
		private final AtomicReference<BigInteger> lastNumber = new AtomicReference<BigInteger>();
		private final AtomicReference<BigInteger[]> lastFactors = new AtomicReference<BigInteger[]>();

		public void service(ServletRequest req, ServletResponse resp) {
			BigInteger i = extractFromRequest(req);
			if (i.equals(lastNumber.get())) {
				encodeIntoResponse(resp, lastFactors.get());
			} else {
				BigInteger[] factors = factor(i);
				lastNumber.set(i);
				lastFactors.set(factors);
				encodeIntoResponse(resp, factors);
			}
		}
	}


	public class CachingFactorizer implements Servlet{
		private BigInteger lastNumber;
		private BigInteger[] lastFactors;

		public void service(ServletRequest req, ServletResponse resp) {
			BigInteger i = extractFromRequest(req);
			BigInteger[] factors = null;
			synchronized (this) {
				if (i.equals(lastNumber)) {
					factors = lastFactors.clone();
				}
			}
			if (factors == null) {
				factors = factor(i);
				synchronized (this) {
					lastNumber = i;
					lastFactors = factors.clone();
				}
			}
			encodeIntoResponse(resp, factors);
		}
	}

UnsafeCachingFactorizer 中lastNumber和lastFactors状态更新存在竞态条件，需要在单个原子操作中。
而CachingFactorizer通过对lastNumber和lastFactors状态操作加锁，不存在竞态条件。
	
可重入例子：

	public class Widget {
		public synchronized void doSomething() {
			......
		}
	}

	public class LoggingWidget extends Widget {
		public synchronized void doSomething() {
			.....
			super.doSomething();
		}		
	}

某个线程访问LoggingWidget的doSomething时，获得Widget的锁，当调用super.doSomething时该线程再次获得Widget的锁。


单个方法是原子操作不能保证符合操作是原子的。如：

	if (!vector.contains(element)) {
		vector.add(element);
	}

此时需要对代码块加锁：

	synchronized (vactor) {
		if (!vector.contains(element)) {
			vector.add(elememt);
		}
	}

或将vector改为并发容器。

4、可见性

	public class NoVisibility {
		private static boolean ready;
		private static int number;
	
		private static class ReaderThread extends Thread {
			public void run() {
				while(!ready)
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

以上代码ready和number变量存在可见性问题，使得主线程中对ready的修改在ReaderThread线程中不一定可见，因此可能导致死循环。

同时`number = 42; ready = true;` 存在重排序问题，使得number的输出可能是0或42。

5、发布与逸出

发布Sample：    
将对象的引用保存到公有静态变量中

	publis static Set<Secret> knownSecrets;
	public void initialize() {
		knownSecrets = new HashSet<Secret>();
	}

逸出Sample：    
本应为私有的数组被发布出去，导致调用者都能修改遮盖数组的内容。

	public class UnsafeStages {
		private String[] states = new String[]{"AK", "AL"};
		public String[] getStates() {
			return states;
		}
	}

隐式逸出Sample：    
当ThisEscape发布内部类EventListener时，也隐含地发布了ThisEscape实例本身。    
同时ThisEscape尚未构造完成，就逸出this引用。    

	public class ThisEscape {
		public ThisEscape(EventSource source) {
			source.registerListener(new EventListener() {
				public void onEvent(Event e) {
					doSomething(e);
				}
			});
		}	
	}

安全的对象构造过程
	
不要在构造过程中使this引用逸出，因为此时类实例尚未构造完成。    
不要在构造函数中启动线程

如果想在构造函数中注册事件监听器，可使用私有构造方法和公有工厂方法避免不正确构造

	public class SafeListener {
		private final EventListener listener;
		private SafeListener() {
			listener = new EventListener() {
				public void onEvent(Event e) {
					doSomething(e);
				}
			};
		}
		public static SafeListener newInstance(EventSource source) {
			SafeListener safe = new SafeListener();
			source.registerListener(safe.listener);
			return safe;
		}
	}

6、线程封闭

栈封闭：    
animals引用被封闭在局部变量中，不会被逸出，保证了对集合操作的安全性。

	public int loadTheArk(Collection<Animal> candidates) {
		SortedSet<Animal> animals;
		int numPairs = 0;
		Animal candidate = null;
		
		animals = new TreeSet<Animal>(new SpeciesGenderComparator());
		animals.addAll(candidates);
		for (Animal a : animals) {
			...
		}
		return numPairs;
	}

ThreadLocal：

	private static ThreadLocal<Connection> connectionHolder
		= new ThreadLocal<Connection>() {
			public Connection initialValue() {
				return DriveManager.getConnection(DB_URL);
			}
		};
	public static Connection getConnection() {
		return connectionHolder.get();	
	}
	
缺点：    
类似于全局变量，降低了代码的可重用性，并在类之间引入隐含的耦合性。

7、不变性

OneValueCache满足不可变类的3个条件：1、对象的所有域都是final类型；2、lastFactors指向factors的一份拷贝，因此满足状态不可变性；3、对象是正确创建的。    
因此该类为不可变类。

每当需要对一组相关数据以原子方式执行某个操作时，就可以考虑创建一个不可变的类来包含这些数据。

lastNumber和lastFactors被包含在不可变类内部，从而保证了操作的原子性。

	public class OneValueCache {
		private final BigInteger lastNumber;
		private final BigInteger[] lastFactors;

		public OneValueCache(BigInteger i, BigInteger[] factors) {
			lastNumber = i;
			lastFactors = Arrays.copyOf(factors, factors.length);
		}

		public BigInteger[] getFactors(BigInteger i) {
			if (lastNumber == null || !latNumber.equals(i))
				return null;
			else
				return Arrays.copyOf(lastFactors, lastFactors.length);
		}
		
	}

使用指向不可变容器对象的volatile类型引用以缓存最新的结果

	public class VolatileCachedFactorizer implements Servlet {
		private volatile OneValueCache cache = new OneValueCache(null, null);
		public void service(ServletRequest req, ServletResponse resp) {
			BigInteger i = extractFromRequest(req);
			BigInteger[] factors = cache.getFactors(i);
			if (factors == null) {
				factors = factor(i);
				cache = new OneValueCache(i, factors);
			}
			encodeIntoResponse(resp, factors);
		}
	}

由于OneValueCache是不可变的，因此与cache相关的操作不会相互干扰，volatile保证了可见性，VolatileCachedFactorizer是线程安全的。

8、安全发布

	public Holder holder;
	public void initialize() {
		holder = new Holder(42);
	}

	public class Holder {
		private int n;
		public Holder(int n) {
			this.n = n;
		}
		public void assertSanity() {
			if (n != n)
				throw new AssertError("This statement is false.");
		}
	}

由于Holder的不安全发布，使得外部调用可以修改holder实例，而n != n是非原子操作，有可能前后两次读取的n不一致，从而抛出异常。

如果Holder是不可变的，那么n也不会变，则即使Holder没有被正确地发布，assertSanity也不会抛出异常。

9、线程安全性的委托

	public class VisualComponent {
		private final List<KeyListener> keyListeners = new CopyOnWriteArrayList<KeyListener>();
		private final List<KeyListener> mouseListeners = new CopyOnWriteArrayList<MouseListener>();
		public void addKeyListener(KeyListener listener) {
			keyListeners.add(listener);
		}
		public void addMouseListener(MouseListener listener) {
			mouseListeners.add(listener);
		}
		public void removeKeyListener(KeyListener listener) {
			keyListeners.remove(listener);
		}
		public void removeMouseListener(MouseListener listener) {
			mouseListeners.remove(listener);
		}
	}

VisualComponent将安全性委托给keyListeners和mouseListeners。    
CopyOnWriteArrayList写入时拷贝，是一个线程安全的链表，而且keyListeners和mouseListeners状态彼此独立，因此不存在安全性问题。

	public class NumberRange {
		private final AtomicInteger lower = new AtomicInteger(0);
		private final AtomicInteger upper = new AtomicInteger(0);
		public void setLower(int i) {
			if (i > upper.get())
				throw new IllegalArgumentException("....");
			lower.set(i);
		}
		public void setUpper(int i) {
			if (i < lower.get())
				throw new IllegalArgumentException("....");
			upper.set(i);
		}
		public boolean isInRange(int i) {
			return (i >= lower.get() && i <= upper.get());
		}
	}

虽然AtomicInteger是线程安全类型，但是公开方法中存在不安全的“先检查后执行”操作，会破坏约束条件lower <= upper，因此存在线程安全性。

	public class SafePoint {
		private int x,y;
		private SafePoint(int[] a) {
			this(a[0], a[1]);
		}
		public SafePoint(SafePoint p) {
			this(p.get());
		}
		public SafePoint(int x, int y) {
			this.x = x;
			this.y = y;
		}
		public synchronized int[] get() {
			return new int[]{x, y};
		}
		public synchronized void set(int x, int y) {
			this.x = x;
			this.y = y;
		}
	}

SafePoint为啥是线程安全的。    
我的理解是，get()和set()都加锁了，且x和y是同时赋值和获取，不会存在两个状态变量不一致的问题。对于构造函数public SafePoint(int x, int y)显然不存在xy不同步的问题，public SafePoint(SafePoint p)中由于真正传进来的是数组拷贝，因此外部调用修改不会影响xy，而private SafePoint(int[] a)是私有方法。    
所以SafePoint是线程安全且可变的。

	public class PublishingVehicleTracker {
		private final Map<String, SafePoint> locations;
		private final Map<String, SafePoint> unmodifiableMap;
		public PublishingVehicleTracker(Map<String, SafePoint> locations) {
			this.locations = new ConcurrentHashMap<String, SafePoint>(locations);
			this.unmodifiableMap = Collections.unmodifiableMap(this.locations);
		}
		public Map<String, SafePoint> getLocations() {
			return unmodifiableMap;
		}
		public SafePoint getLocation(String id) {
			return locations.get(id);
		}
		public void setLocation(String id, int x, int y) {
			if (!locations.containsKey(id))
				throw new IllegalArgumentException("...");
			locations.get(id).set(x, y);
		}
	}

	public class ListHelper<E> {
		public List<E> list = Collections.synchronizedList(new ArrayList<E>());
		public synchronized boolean putIfAbsent(E x) {
			boolean absent = !list.contains(x);
			if (absent)
				list.add(x);
			return absent;
		}
	}

这段代码必须使用的是2个不同的锁，list的锁是list本身，而putIfAbsent的锁是ListHelper，因此没办法保证安全性。

10、同步容器类

	public static Object getLast(Vector list) {
		synchronized(list) {
			int lastIndex = list.size() - 1;
			return list.get(lastIndex);
		}
	}
	public static void deleteLast(Vector list) {
		synchronized(list) {
			int lastIndex = list.size() - 1;
			list.remove(lastIndex);
		}
	}

	synchronized(vector) {
		for (int i = 0; i < vector.size(); i++)
			doSomething(vector.get(i));
	}
	
同步容器类虽然是线程安全的，但是对于复合操作由于存在竞态条件，需要加锁才能保证安全性。



