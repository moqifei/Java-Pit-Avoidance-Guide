# ThreadLocal

## 使用类似ThreadLocal工具来存放一些数据时，需要特别注意在代码运行完后，显式地去清空设置的数据

说明：不能认为没有显示开启多线程就不会有线程安全问题，像Tomcat这种Web服务器，程序运行在Tomcat中，执行程序的线程是Tomcat的工作线程，而Tomcat的工作线程是基于线程池的

```
private static final ThreadLocal<String> local = ThreadLocal.withInitial(() -> null);
local.set("foo");
try{
	//do Something
}finally{
	local.remove();
}

```

进阶：ThreadLocal源码分析, 使用完毕，显式地调用remove()方法

1. 每个Thread维护着一个ThreadLocalMap 引用

2. ThreadLocalMap是ThreadLocal的内部类，用Entry来进行存储

3. Thread创建的ThreadLocal是存储在自身的threadLocals中，也就是自身的ThreadLocalMap

4. ThreadLocalMap的键值为ThreadLocal对象，而且可以有多个threadLocal变量，因此保存在map中

5. 在进行get之前，必须先set，否则会报空指针异常，当然也可以初始化一个，但是必须重写initialValue()方法。

6. ThreadLocal本身并不存储值，它只是作为一个key来让线程从ThreadLocalMap获取value。

   实线代表强引用,虚线代表弱引用.

![https://github.com/moqifei/Java-Pit-Avoidance-Guide/blob/main/pic/91ef76c6a7efce1b563edc5501a900dbb58f6512.jpeg]

由于ThreadLocalMap的key是弱引用，而Value是强引用。这就导致了一个问题，ThreadLocal在没有外部对象强引用时，发生GC时弱引用Key会被回收，而Value不会回收，如果创建ThreadLocal的线程一直持续运行，那么这个Entry对象中的value就有可能一直得不到回收，发生内存泄露。



# ConcurrentHashMap

## ConcurrentHashMap 只能保证提供的原子性读写操作是线程安全的，无法保证多个操作之间的状态是一致的，诸如size()、isEmpty()、putAll（）和containsValue（）等聚合方法，在并发情况下可能会反映ConcurrentHashMap的中间状态，并发情形下，这些方法的返回值只能用作参考，如需用于流程控制，则需要手动加锁

说明：一直在进化，最好先理解HashMap

1️⃣整体结构

1.7：Segment + HashEntry + Unsafe
 1.8: 移除 Segment，使锁的粒度更小，Synchronized + CAS + Node + Unsafe

2️⃣put()

1.7：先定位 Segment，再定位桶，put 全程加锁，没有获取锁的线程提前找桶的位置，并最多自旋 64 次获取锁，超过则挂起。
 1.8：由于移除了 Segment，类似 [HashMap](https://www.jianshu.com/p/6c70d265aa7b)，可以直接定位到桶，拿到 first 节点后进行判断：①为空则 [CAS](https://www.jianshu.com/p/98220486426a) 插入；②为 -1 则说明在扩容，则跟着一起扩容；③ else 则加锁 put(类似1.7)

3️⃣get()

基本类似，由于 value 声明为 [volatile](https://www.jianshu.com/p/6c96719bba04)，保证了修改的可见性，因此不需要加锁。

4️⃣resize()

1.7：跟 [HashMap](https://www.jianshu.com/p/6c70d265aa7b) 步骤一样，只不过是搬到单线程中执行，避免了 [HashMap](https://www.jianshu.com/p/6c70d265aa7b) 在 1.7 中扩容时死循环的问题，保证线程安全。
 1.8：支持并发扩容，[HashMap](https://www.jianshu.com/p/6c70d265aa7b) 扩容在1.8中由头插改为尾插(为了避免死循环问题)，ConcurrentHashmap 也是，迁移也是从尾部开始，扩容前在桶的头部放置一个 hash 值为 -1 的节点，这样别的线程访问时就能判断是否该桶已经被其他线程处理过了。

5️⃣size()

1.7：很经典的思路：计算两次，如果不变则返回计算结果，若不一致，则锁住所有的 Segment 求和。
 1.8：用 baseCount 来存储当前的节点个数，这就设计到 baseCount 并发环境下修改的问题。



# CopyOnWriteArrayList

## 在 Java 中，CopyOnWriteArrayList 虽然是一个线程安全的 ArrayList，但因为其实现方式是，每次修改数据时都会复制一份数据出来，所以有明显的适用场景，即读多写少或者说希望无锁读的场景。

说明：注意CopyOnWrite的内存占用问题；CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性



# ThreadLocalRandom

## 不能将ThreadLocalRandom的实例变量设置到静态变量中，在多线程情况下复用

说明：ThreadLocalRandom类中只封装了一些公用的方法，种子存放在各个线程中ThreadLocalRandom中存放一个单例的instance，调用current()方法返回这个instance，每个线程首次调用current()方法时，会在各个线程中初始化seed和probe。nextX()方法会调用nextSeed()，在其中使用各个线程中的种子，计算下一个种子并保存（UNSAFE.getLong(t, SEED) + GAMMA）。如果使用静态变量，直接调用nextX()方法就跳过了各个线程初始化的步骤，只会在每次调用nextSeed()时来更新种子。

```
ThreadLocalRandom.current().nextX(...);
```



# 锁

## 加锁前要清楚锁和被保护的对象是不是一个层面的，静态字段属于类，类级别的锁才能保护；而非静态字段属于类实例，实例级别的锁就可以保护。

反例：

```
class Data {
    @Getter
    private static int counter = 0;
    
    public static int reset() {
        counter = 0;
        return counter;
    }

    public synchronized void wrong() {
        counter++;
    }
}
```

说明：在非静态的 wrong 方法上加锁，只能确保多个线程无法执行同一个实例的 wrong 方法，却不能保证不会执行不同实例的 wrong 方法。而静态的 counter 在多个实例中共享，所以必然会出现线程安全问题。

正例：

```
class Data {
    @Getter
    private static int counter = 0;
    private static Object locker = new Object();

    public void right() {
        synchronized (locker) {
            counter++;
        }
    }
}
```

## 尽可能降低锁的粒度，仅对必要的代码块甚至是需要保护的资源本身加锁

## 业务逻辑中有多把锁时要考虑死锁问题，通常的规避方案是，避免无限等待和循环等待。

## 线程安全之原子性：synchronized同步机制

说明：当声明 synchronized 代码块时，编译而成的字节码将包含 monitorenter 和 monitorexit 指令。关于 monitorenter 和 monitorexit 的作用，我们可以抽象地理解为每个锁对象拥有一个锁计数器和一个指向持有该锁的线程的指针。当执行 monitorenter 时，如果目标锁对象的计数器为 0，那么说明它没有被其他线程所持有。在这个情况下，Java 虚拟机会将该锁对象的持有线程设置为当前线程，并且将其计数器加 1。在目标锁对象的计数器不为 0 的情况下，如果锁对象的持有线程是当前线程，那么 Java 虚拟机可以将其计数器加 1，否则需要等待，直至持有线程释放该锁。当执行 monitorexit 时，Java 虚拟机则需将锁对象的计数器减 1。当计数器减为 0 时，那便代表该锁已经被释放掉了。

偏向锁只会在第一次请求时采用 CAS 操作，在锁对象的标记字段中记录下当前线程的地址。在之后的运行过程中，持有该偏向锁的线程的加锁操作将直接返回。它针对的是锁仅会被同一线程持有的情况。

如果有另外的线程试图锁定某个已经被偏斜过的对象，JVM 就需要撤销（revoke）偏斜锁，并切换到轻量级锁实现。轻量级锁采用 CAS 操作，将锁对象的标记字段替换为一个指针，指向当前线程栈上的一块空间，存储着锁对象原本的标记字段。它针对的是多个线程在不同时间段申请同一把锁的情况。

获取轻量级锁失败，升级为重量级锁，重量级锁会阻塞、唤醒请求加锁的线程。它针对的是多个线程同时竞争同一把锁的情况。Java 虚拟机采取了自适应自旋，来避免线程在面对非常小的 synchronized 代码块时，仍会被阻塞、唤醒的情况。

## 线程安全之原子性：ReentrantLock同步机制

说明：

功能说明：https://www.cnblogs.com/takumicx/p/9338983.html

源码解析：https://www.cnblogs.com/takumicx/p/9402021.html

能力进阶：当线程获取锁失败时如何安全的加入同步等待队列

![https://github.com/moqifei/Java-Pit-Avoidance-Guide/blob/main/pic/1422237-20180805184052040-1494856465.png]

- 1.队列为空的情况:
  因为队列为空,故head=tail=null,假设线程执行2成功,则在其执行3之前,因为tail=null,其他进入该方法的线程因为head不为null将在2处不停的失败,所以3即使没有同步也不会有线程安全问题。
- 2.队列不为空的情况:
  假设线程执行5成功,则此时4的操作必然也是正确的(当前结点的prev指针确实指向了队列尾结点,换句话说tail指针没有改变,如若不然5必然执行失败),又因为4执行成功,当前节点在队列中的次序已经确定了,所以6何时执行对线程安全不会有任何影响,比如下面这种情况

## 线程安全可见性：多线程情形下，控制流程的flag字段必须使用volatile关键字

说明：当一个变量定义为 volatile 之后，将具备两种特性：

　　1.保证此变量对所有的线程的可见性，这里的“可见性”，如本文开头所述，当一个线程修改了这个变量的值，volatile 保证了新值能立即同步到主内存，以及每次使用前立即从主内存刷新。但普通变量做不到这点，普通变量的值在线程间传递均需要通过主内存（详见：[Java内存模型](http://www.cnblogs.com/zhengbin/p/6407137.html)）来完成。

　　2.禁止指令重排序优化。有volatile修饰的变量，赋值后多执行了一个“load addl $0x0, (%esp)”操作，这个操作相当于一个**内存屏障**（指令重排序时不能把后面的指令重排序到内存屏障之前的位置），只有一个CPU访问内存时，并不需要内存屏障；（什么是指令重排序：是指CPU采用了允许将多条指令不按程序规定的顺序分开发送给各相应电路单元处理）。

## 线程池的生命需要手动操作，禁止使用Executors内设的方法创建线程

说明：newFixedThreadPool的工作队列是LinkedBlockingQueue，如果任务较多且执行较慢的化，队列可能会快速积压，导致OOM

newCachedThreadPool的最大线程数量是 Integer.MAX_VALUE，而工作队列SynchronousQueue 是一个没有存储空间的阻塞队列，请求认为到来时需要创建新的线程来处理，如果任务执行时间较长，则会导致OOM

【推荐】要根据任务的“轻重缓急”来指定线程池的核心参数，包括线程数、回收策略和任务队列：

对于执行比较慢、数量不大的 IO 任务，或许要考虑更多的线程数，而不需要太大的队列；

而对于吞吐量较大的计算型任务，线程数量不宜过多，可以是 CPU 核数或核数 *2，但可能需要较长的队列来做缓冲；

任何时候，都应该为自定义线程池指定有意义的名称，以方便排查问题

【进阶】：优先开启更多的线程，而把队列当成一个后备方案：

重写队列的offer方法，造成队列已满的假象；达到最大线程后触发拒绝策略，自定以拒绝策略将任务真正插入队列

https://github.com/apache/tomcat/blob/a801409b37294c3f3dd5590453fb9580d7e33af2/java/org/apache/tomcat/util/threads/ThreadPoolExecutor.java
