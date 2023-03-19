# JVM原理

## 类加载机制

### 加载过程

1. 加载（获取来自任意来源的字节流并转换成运行时数据结构，生成Class对象）
2. 验证（验证字节流信息符合当前虚拟机的要求，防止被篡改过的字节码危害JVM安全）
3. 准备（为类变量分配内存并设置初始值）
4. 解析（将常量池的符号引用替换为直接引用，符号引用是用一组符号来描述所引用的目标，直接引用是指向目标的指针）
5. 初始化（执行类构造器、类变量赋值、静态语句块）

### 类加载器

* 启动类加载器：用C++语言实现，是虚拟机自身的一部分，它负责将 <JAVA_HOME>/lib路径下的核心类库，无法被Java程序直接引用
* 扩展类加载器：用Java语言实现，它负责加载<JAVA_HOME>/lib/ext目录下或者由系统变量-Djava.ext.dir指定位路径中的类库，开发者可以直接使用
* 系统类加载器：用Java语言实现，它负责加载系统类路径ClassPath指定路径下的类库，开发者可以直接使用

### 双亲委派

定义：如果父类加载器可以完成类加载任务，就成功返回，倘若父类加载器无法完成此加载任务，子加载器才会尝试自己去加载，这就是**双亲委派模式**。

优点：采用双亲委派模式的是好处是Java类随着它的类加载器一起具备了一种带有优先级的层次关系，通过这种层级关可以避免类的重复加载，当父亲已经加载了该类时，就没有必要子ClassLoader再加载一次。其次防止恶意覆盖Java核心API。

三次大型破坏双亲委派模式的事件：

1. 在双亲委派模式出来之前，用户继承ClassLoader就是为了重写loadClass方法，但双亲委派模式需要这个方法，所以1.2之后添加了findClass供以后的用户重写
2. 如果基础类要调回用户的代码，如JNDI/JDBC需要调用ClassPath下的自己的代码来进行资源管理，Java团队添加了一个线程上下文加载器，如果该加载器没有被设置过，那么就默认是应用程序类加载器
3. 为了实现代码热替换，OSGi是为了实现自己的类加载逻辑，用平级查找的逻辑替换掉了向下传递的逻辑。但其实可以不破坏双亲委派逻辑而是自定义类加载器来达到代码热替换。比如[这篇文章](https://www.cnblogs.com/pfxiong/p/4070462.html)

## GC回收机制

### 回收对象

不可达对象：通过一系列的GC Roots的对象作为起点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连时则此对象是不可用的。
GC Roots包括：虚拟机栈中引用的对象、方法区中类静态属性引用的对象、方法区中常量引用的对象、本地方法栈中JNI（Native方法）引用的对象。

彻底死亡条件：
条件1：通过GC Roots作为起点的向下搜索形成引用链，没有搜到该对象，这是第一次标记。
条件2：在finalize方法中没有逃脱回收（将自身被其他对象引用），这是第一次标记的清理。

### 如何回收

新生代因为每次GC都有大批对象死去，只需要付出少量存活对象的复制成本且无碎片所以使用“复制算法”
老年代因为存活率高、没有分配担保空间，所以使用“标记-清理”或者“标记-整理”算法

复制算法：将可用内存按容量划分为Eden、from survivor、to survivor，分配的时候使用Eden和一个survivor，Minor GC后将存活的对象复制到另一个survivor，然后将原来已使用的内存一次清理掉。这样没有内存碎片。
标记-清除：首先标记出所有需要回收的对象，标记完成后统一回收被标记的对象。会产生大量碎片，导致无法分配大对象从而导致频繁GC。
标记-整理：首先标记出所有需要回收的对象，让所有存活的对象向一端移动。

### Minor GC条件

当Eden区空间不足以继续分配对象，发起Minor GC。

### Full GC条件

1. 调用System.gc时，系统建议执行Full GC，但是不必然执行
2. 老年代空间不足（通过Minor GC后进入老年代的大小大于老年代的可用内存）
3. 方法区空间不足

## 垃圾收集器

## 串行收集器

串行收集器Serial是最古老的收集器，只使用一个线程去回收，可能会产生较长的停顿

新生代使用Serial收集器 `复制`算法、老年代使用Serial Old `标记-整理`算法

参数：`-XX:+UseSerialGC`，默认开启 `-XX:+UseSerialOldGC`

## 并行收集器

并行收集器Parallel关注**可控的吞吐量**，能精确地控制吞吐量与最大停顿时间是该收集器最大的特点，也是1.8的Server模式的默认收集器，使用多线程收集。ParNew垃圾收集器是Serial收集器的多线程版本。

新生代 `复制`算法、老年代 `标记-整理`算法

参数：`-XX:+UseParallelGC`，默认开启 `-XX:+UseParallelOldGC`

## 并发收集器

并发收集器CMS是以**最短停顿时间**为目标的收集器。G1关注能在大内存的前提下精确控制**停顿时间**且垃圾回收效率高。

CMS针对老年代，有初始标记、并发标记、重新标记、并发清除四个过程，标记阶段会Stop The World，使用 `标记-清除`算法，所以会产生内存碎片。

参数：`-XX:+UseConcMarkSweepGC`，默认开启 `-XX:+UseParNewGC`

G1将堆划分为多个大小固定的独立区域，根据每次允许的收集时间优先回收垃圾最多的区域，使用 `标记-整理`算法，是1.9的Server模式的默认收集器

参数：`-XX:+UseG1GC`

## Stop The World

Java中Stop-The-World机制简称STW，是在执行垃圾收集算法时，Java应用程序的其他所有线程都被挂起。Java中一种全局暂停现象，全局停顿，所有Java代码停止，native代码可以执行，但不能与JVM交互

STW总会发生，不管是新生代还是老年代，比如CMS在初始标记和重复标记阶段会停顿，G1在初始标记阶段也会停顿，所以并不是选择了一款停顿时间低的垃圾收集器就可以避免STW的，我们只能尽量去减少STW的时间。

那么为什么一定要STW？因为在定位堆中的对象时JVM会记录下对所有对象的引用，如果在定位对象过程中，有新的对象被分配或者刚记录下的对象突然变得无法访问，就会导致一些问题，比如部分对象无法被回收，更严重的是如果GC期间分配的一个GC Root对象引用了准备被回收的对象，那么该对象就会被错误地回收。

## Java内存模型

定义：JMM是一种规范，目的是解决由于多线程通过共享内存进行通信时，存在的本地内存数据不一致、编译器会对代码指令重排序、处理器会对代码乱序执行等带来的问题。目的是保证并发编程场景中的原子性、可见性和有序性

实现：volatile、synchronized、final、concurrent包等。其实这些就是Java内存模型封装了底层的实现后提供给程序员使用的一些关键字。

主内存：所有变量都保存在主内存中
工作内存：每个线程的独立内存，保存了该线程使用到的变量的主内存副本拷贝，线程对变量的操作必须在工作内存中进行

每个线程都有自己的本地内存共享副本，如果A线程要更新主内存还要让B线程获取更新后的变量，那么需要：

1. 将本地内存A中更新共享变量
2. 将更新的共享变量刷新到主内存中
3. 线程B从主内存更新最新的共享变量

## happens-before

我们无法就所有场景来规定某个线程修改的变量何时对其他线程可见，但是我们可以指定某些规则，这规则就是happens-before。特别关注在多线程之间的内存可见性。

它是判断数据是否存在竞争、线程是否安全的主要依据，依靠这个原则，我们解决在并发环境下两操作之间是否可能存在冲突的所有问题。

1. 程序次序规则：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作；
2. 锁定规则：一个unLock操作先行发生于后面对同一个锁额lock操作；
3. volatile变量规则：对一个变量的写操作先行发生于后面对这个变量的读操作；
4. 传递规则：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C；
5. 线程启动规则：Thread对象的start()方法先行发生于此线程的每个一个动作；
6. 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生；
7. 线程终结规则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值手段检测到线程已经终止执行；
8. 对象终结规则：一个对象的初始化完成先行发生于他的finalize()方法的开始；

## JVM调优

前提：在进行GC优化之前，需要确认项目的架构和代码等已经没有优化空间

目的：优化JVM垃圾收集性能从而增大吞吐量或减少停顿时间，让应用在某个业务场景上发挥最大的价值。吞吐量是指应用程序线程用时占程序总用时的比例。暂停时间是应用程序线程让与GC线程执行而完全暂停的时间段

对于交互性web应用来说，一般都是减少停顿时间，所以有以下方法：

1. 如果应用存在大量的短期对象，应该选择较大的年轻代；如果存在相对较多的持久对象，老年代应该适当增大
2. 让大对象进入年老代。可以使用参数-XX:PetenureSizeThreshold 设置大对象直接进入年老代的阈值。当对象的大小超过这个值时，将直接在年老代分配
3. 设置对象进入年老代的年龄。如果对象每经过一次 GC 依然存活，则年龄再加 1。当对象年龄达到阈值时，就移入年老代，成为老年对象
4. 使用关注系统停顿的 CMS 回收器

# 多线程

## 线程的生命周期

新建 -- 就绪 -- 运行 -- 阻塞 -- 就绪 -- 运行 -- 死亡

## 你怎么理解多线程的

1. 定义：多线程是指从软件或者硬件上实现多个线程并发执行的技术。具有多线程能力的计算机因有硬件支持而能够在同一时间执行多于一个线程，进而提升整体处理性能。
2. 存在的原因：因为单线程处理能力低。打个比方，一个人去搬砖与几个人去搬砖，一个人只能同时搬一车，但是几个人可以同时一起搬多个车。
3. 实现：在Java里如何实现线程，Thread、Runnable、Callable。
4. 问题：线程可以获得更大的吞吐量，但是开销很大，线程栈空间的大小、切换线程需要的时间，所以用到线程池进行重复利用，当线程使用完毕之后就放回线程池，避免创建与销毁的开销。

## 线程间通信的方式

1. 等待通知机制 wait()、notify()、join()、interrupted()
2. 并发工具synchronized、lock、CountDownLatch、CyclicBarrier、Semaphore

## 锁

### 锁是什么

锁是在不同线程竞争资源的情况下来分配不同线程执行方式的同步控制工具，只有线程获取到锁之后才能访问同步代码，否则等待其他线程使用结束后释放锁

### synchronized

通常和wait，notify，notifyAll一块使用。
wait：释放占有的对象锁，释放CPU，进入等待队列只能通过notify/all继续该线程。
sleep：则是释放CPU，但是不释放占有的对象锁，可以在sleep结束后自动继续该线程。
notify：唤醒等待队列中的一个线程，使其获得锁进行访问。
notifyAll：唤醒等待队列中等待该对象锁的全部线程，让其竞争去获得锁。

### lock

拥有synchronize相同的语义，但是添加一些其他特性，如中断锁等候和定时锁等候，所以可以使用lock代替synchronize，但必须手动加锁释放锁

### 两者的区别

* 性能：资源竞争激烈的情况下，lock性能会比synchronized好；如果竞争资源不激烈，两者的性能是差不多的
* 用法：synchronized可以用在代码块上，方法上。lock通过代码实现，有更精确的线程语义，但需要手动释放，还提供了多样化的同步，比如公平锁、有时间限制的同步、可以被中断的同步
* 原理：synchronized在JVM级别实现，会在生成的字节码中加上monitorenter和monitorexit，任何对象都有一个monitor与之相关联，当且一个monitor被持有之后，他将处于锁定状态。monitor是JVM的一个同步工具，synchronized还通过内存指令屏障来保证共享变量的可见性。lock使用AQS在代码级别实现，通过Unsafe.park调用操作系统内核进行阻塞
* 功能：比如ReentrantLock功能更强大
  1. ReentrantLock可以指定是公平锁还是非公平锁，而synchronized只能是非公平锁，所谓的公平锁就是先等待的线程先获得锁
  2. ReentrantLock提供了一个Condition（条件）类，用来实现分组唤醒需要唤醒的线程们，而不是像synchronized要么随机唤醒一个线程要么唤醒全部线程
  3. ReentrantLock提供了一种能够中断等待锁的线程的机制，通过lock.lockInterruptibly()来实现这个机制

**我们写同步的时候，优先考虑synchronized，如果有特殊需要，再进一步优化。ReentrantLock和Atomic如果用的不好，不仅不能提高性能，还可能带来灾难。**

### 锁的种类

* 公平锁/非公平锁

公平锁是指多个线程按照申请锁的顺序来获取锁ReentrantLock通过构造函数指定该锁是否是公平锁，默认是非公平锁。Synchronized是一种非公平锁

* 可重入锁

指在同一个线程在外层方法获取锁的时候，在进入内层方法会自动获取锁ReentrantLock, 他的名字就可以看出是一个可重入锁，其名字是Re entrant Lock重新进入锁Synchronized,也是一个可重入锁。可重入锁的一个好处是可一定程度避免死锁

* 独享锁/共享锁

独享锁是指该锁一次只能被一个线程所持有。共享锁是指该锁可被多个线程所持有ReentrantLock是独享锁。但是对于Lock的另一个实现类ReadWriteLock，其读锁是共享锁，其写锁是独享锁读锁的共享锁可保证并发读是非常高效的，读写，写读 ，写写的过程是互斥的Synchronized是独享锁

* 乐观锁/悲观锁

悲观锁在Java中的使用，就是各种锁乐观锁在Java中的使用，是CAS算法，典型的例子就是原子类，通过CAS自旋实现原子操作的更新

* 偏向锁/轻量级锁/重量级锁

针对Synchronized的锁状态：偏向锁是为了减少无竞争且只有一个线程使用锁的情况下，使用轻量级锁产生的性能消耗。指一段同步代码一直被一个线程所访问，在无竞争情况下把整个同步都消除掉轻量级锁是为了减少无实际竞争情况下，使用重量级锁产生的性能消耗。指当锁是偏向锁的时候，被另一个线程所访问，偏向锁就会升级为轻量级锁，其他线程会通过CAS自旋的形式尝试获取锁，不会阻塞重量级锁是指当锁为轻量级锁的时候，另一个线程虽然是自旋，但自旋不会一直持续下去，当自旋一定次数的时候，还没有获取到锁，就会进入阻塞，该锁膨胀为重量级锁

* 自旋锁/自适应自旋锁

指尝试获取锁的线程不会立即阻塞，而是采用循环的方式去尝试获取锁，这样的好处是减少线程上下文切换的消耗，缺点是循环会消耗CPU，默认自旋次数为10
自适应自旋锁的自旋次数不再固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态决定，是虚拟机对锁状况的一个预测

### volatile

功能：

1. 主内存和工作内存，直接与主内存产生交互，进行读写操作，保证可见性；
2. 禁止 JVM 进行的指令重排序。

### ThreadLocal

使用 `ThreadLocal<UserInfo> userInfo = new ThreadLocal<UserInfo>()`的方式，让每个线程内部都会维护一个ThreadLocalMap，里边包含若干了 Entry（K-V 键值对），每次存取都会先获取到当前线程ID，然后得到该线程对象中的Map，然后与Map交互。

## 线程池

### 起源

new Thread弊端：

* 每次启动线程都需要new Thread新建对象与线程，性能差。线程池能重用存在的线程，减少对象创建、回收的开销。
* 线程缺乏统一管理，可以无限制的新建线程，导致OOM。线程池可以控制可以创建、执行的最大并发线程数。
* 缺少工程实践的一些高级的功能如定期执行、线程中断。线程池提供定期执行、并发数控制功能

### 线程池时核心参数

* corePoolSize：核心线程数量，线程池中应该常驻的线程数量
* maximumPoolSize：线程池允许的最大线程数，非核心线程在超时之后会被清除
* workQueue：阻塞队列，存储等待执行的任务
* keepAliveTime：线程没有任务执行时可以保持的时间
* unit：时间单位
* threadFactory：线程工厂，来创建线程
* rejectHandler：当拒绝任务提交时的策略（抛异常、用调用者所在的线程执行任务、丢弃队列中第一个任务执行当前任务、直接丢弃任务）

### 创建线程的逻辑

以下任务提交逻辑来自ThreadPoolExecutor.execute方法：

1. 如果运行的线程数 < corePoolSize，直接创建新线程，即使有其他线程是空闲的
2. 如果运行的线程数 >= corePoolSize
   2.1 如果插入队列成功，则完成本次任务提交，但不创建新线程
   2.2 如果插入队列失败，说明队列满了
   2.2.1 如果当前线程数 < maximumPoolSize，创建新的线程放到线程池中
   2.2.2 如果当前线程数 >= maximumPoolSize，会执行指定的拒绝策略

### 阻塞队列的策略

* 直接提交。SynchronousQueue是一个没有数据缓冲的BlockingQueue，生产者线程对其的插入操作put必须等待消费者的移除操作take。将任务直接提交给线程而不保持它们。
* 无界队列。当使用无限的 maximumPoolSizes 时，将导致在所有corePoolSize线程都忙时新任务在队列中等待。
* 有界队列。当使用有限的 maximumPoolSizes 时，有界队列（如ArrayBlockingQueue）有助于防止资源耗尽，但是可能较难调整和控制。

## 并发包工具类

### CountDownLatch

计数器闭锁是一个能阻塞主线程，让其他线程满足特定条件下主线程再继续执行的线程同步工具。

![](https://github.com/xbox1994/Java-Interview/raw/master/images/countdownlatch.png)
图中，A为主线程，A首先设置计数器的数到AQS的state中，当调用await方法之后，A线程阻塞，随后每次其他线程调用countDown的时候，将state减1，直到计数器为0的时候，A线程继续执行。

使用场景:
并行计算：把任务分配给不同线程之后需要等待所有线程计算完成之后主线程才能汇总得到最终结果
模拟并发：可以作为并发次数的统计变量，当任意多个线程执行完成并发任务之后统计一次即可

### Semaphore

信号量是一个能阻塞线程且能控制统一时间请求的并发量的工具。比如能保证同时执行的线程最多200个，模拟出稳定的并发量。

```java
public class CountDownLatchTest {

    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();
        Semaphore semaphore = new Semaphore(3); //配置只能发布3个运行许可证
        for (int i = 0; i < 100; i++) {
            int finalI = i;
            executorService.execute(() -> {
                try {
                    semaphore.acquire(3); //获取3个运行许可，如果获取不到会一直等待，使用tryAcquire则不会等待
                    Thread.sleep(1000);
                    System.out.println(finalI);
                    semaphore.release(3);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
        executorService.shutdown();
    }
}
```

由于同时获取3个许可，所以即使开启了100个线程，但是每秒只能执行一个任务

使用场景:
数据库连接并发数，如果超过并发数，等待（acqiure）或者抛出异常（tryAcquire）

## 编程题

交替打印奇偶数

```java
public class PrintOddAndEvenShu {
    private int value = 0;

    private synchronized void printOdd() {
        while (value <= 100) {
            if (value % 2 == 1) {
                System.out.println(Thread.currentThread() + ": -" + value++);
                this.notify();
            } else {
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

        }

    }

    private synchronized void printEven() {
        while (value <= 100) {
            if (value % 2 == 0) {
                System.out.println(Thread.currentThread() + ": --" + value++);
                this.notify();
            } else {
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        PrintOddAndEvenShu print = new PrintOddAndEvenShu();
        Thread t1 = new Thread(print::printOdd);
        Thread t2 = new Thread(print::printEven);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
```

多线程交替打印alibaba： https://blog.csdn.net/CX610602108/article/details/106427979

```java

public class Test {
    private int currentI = 0;

    private synchronized void printSpace(int inputLength) {
        while (true) {
            if (currentI == inputLength) {
                currentI = 0;
                System.out.print(" ");
                this.notifyAll();
            } else {
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    private synchronized void printLetter(int i, char c) {
        while (true) {
            if (i == currentI) {
                currentI++;
                System.out.print(c);
                this.notifyAll();
            } else {
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    private synchronized void print(String input) throws Exception {
        Test test = new Test();
        for (int i = 0; i < input.length(); i++) {
            char c = input.charAt(i);
            int finalI = i;
            Thread tt = new Thread(() -> test.printLetter(finalI, c));
            tt.start();
        }
        Thread space = new Thread(() -> test.printSpace(input.length()));
        space.start();
        space.join();
    }


    public static void main(String[] args) throws Exception {
        Test test = new Test();
        test.print("alibaba");
    }
}

```

# 集合

## ArrayList与LinkedList区别

|ArrayList|LinkedList|

|--------|--------|

|数组|双向链表|

|增删的时候在扩容的时候慢，通过索引查询快，通过对象查索引慢|增删快，通过索引查询慢，通过对象查索引慢|

|当数组无法容纳下此次添加的元素时进行扩容|无|

|扩容之后容量为原来的1.5倍|无|

## HashMap

1. JDK 1.8 以前 HashMap 的实现是 数组+单链表，即使哈希函数取得再好，也很难达到元素百分百均匀分布。当 HashMap 中有大量的元素都存放到同一个桶中时，这个桶下有一条长长的链表，这个时候 HashMap 就相当于一个单链表，假如单链表有 n 个元素，遍历的时间复杂度就是 O(n)，完全失去了它的优势。针对这种情况，JDK 1.8 中引入了红黑树（查找时间复杂度为 O(logn)）来优化这个问题
2. 为什么线程不安全？多线程PUT操作时可能会覆盖刚PUT进去的值；扩容操作会让链表形成环形数据结构，形成死循环
3. 容量的默认大小是 16，负载因子是 0.75，当 HashMap 的 size > 16*0.75 时就会发生扩容(容量和负载因子都可以自由调整)。
4. 为什么容量是2的倍数？在根据hashcode查找数组中元素时，取模性能远远低于与性能，且和2^n-1进行与操作能保证各种不同的hashcode对应的元素也能均匀分布在数组中

## HashMap rehash过程

1. 空间不够用了，所以需要分配一个大一点的空间，然后保存在里面的内容需要重新计算 hash
2. 如果两个线程都发现HashMap需要重新调整大小了，它们会同时试着调整大小
3. 在调整大小的过程中，存储在链表中的元素的次序会反过来，因为移动到新的bucket位置的时候，HashMap并不会将元素放在链表的尾部，而是放在头部，这是为了避免尾部遍历(tail traversing)。如果条件竞争发生了，那么就死循环了

## ConcurrentHashMap原理

HashTable 在每次同步执行时都要锁住整个结构。ConcurrentHashMap 锁的方式是稍微细粒度的。 ConcurrentHashMap 将 hash 表分为 16 个桶（默认值）

最大并发个数就是Segment的个数，默认值是16，可以通过构造函数改变一经创建不可更改，这个值就是并发的粒度，每一个segment下面管理一个table数组，加锁的时候其实锁住的是整个segment

# IO

## BIO

Block-IO：InputStream和OutputStream，Reader和Writer。属于同步阻塞模型

同步阻塞：一个请求占用一个进程处理，先等待数据准备好，然后从内核向进程复制数据，最后处理完数据后返回

## NIO

NonBlock-IO：Channel、Buffer、Selector。属于IO多路复用的同步非阻塞模型

同步非阻塞：进程先将一个套接字在内核中设置成非阻塞再等待数据准备好，在这个过程中反复轮询内核数据是否准备好，准备好之后最后处理数据返回

IO多路复用：同步非阻塞的优化版本，区别在于IO多路复用阻塞在select，epoll这样的系统调用之上，而没有阻塞在真正的IO系统调用上。换句话说，轮询机制被优化成通知机制，多个连接公用一个阻塞对象，进程只需要在一个阻塞对象上等待，无需再轮询所有连接

在Java的NIO中，是基于Channel和Buffer进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。Selector用于监听多个通道的事件（比如：连接打开，数据到达）

因此，单个线程可以监听多个数据通道，Selector的底层实现是epoll/poll/select的IO多路复用模型，select方法会一直阻塞，直到channel中有事件就绪：

与BIO区别如下：

1. 通过缓冲区而非流的方式进行数据的交互，流是进行直接的传输的没有对数据操作的余地，缓冲区提供了灵活的数据处理方式
2. NIO是非阻塞的，意味着每个socket连接可以让底层操作系统帮我们完成而不需要每次开个线程去保持连接，使用的是selector监听所有channel的状态实现
3. NIO提供直接内存复制方式，消除了JVM与操作系统之间读写内存的损耗

## AIO

Asynchronous IO：属于事件和回调机制的异步非阻塞模型

AIO得到结果的方式：

* 基于回调：实现CompletionHandler接口，调用时触发回调函数
* 返回Future：通过isDone()查看是否准备好，通过get()等待返回数据

但要实现真正的异步非阻塞IO，需要操作系统支持，Windows支持而Linux不完善
