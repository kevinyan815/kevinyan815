
因为普通的容器类是非线程安全的，如果是在多线程并发的情况下使用，我们就需要手动对其加锁才会安全，这样的话就很麻烦，所以 Java 就提供了同步容器，它来帮我们自动加锁。
## 从普通容器到同步容器
在 Java 中，同步容器主要包括 2 类：

- 普通类：Vector、Stack、HashTable。
- 内部类：Collections创建的内部类，比如Collections.SynchronizedList、 Collections.SynchronizedSet等。

同步容器的同步原理就是在其 get、set、size 等主要方法上用 synchronized 修饰。 synchronized 可以保证在同一个时刻，只有一个线程可以执行同步方法或者同步代码块。

我们把使用普通容器 ArrayList 的例子换成使用同步容器 Collections.SynchronizedList，就能解决并发泄数据不一致的问题。
```java
public class SyncCollectionDemo {

    private List<Integer> listSync;

    public SyncCollectionDemo() {
        // 这里包装一个空的 ArrayList 生成同步 List 容器
        this.listSync = Collections.synchronizedList(new ArrayList<>());
    }

    public void addSync(int temp){
        listSync.add(temp);
    }

    public List<Integer> getListSync() {
        return this.listSync;
    }

    public static void main(String[] args) throws InterruptedException {
        SyncCollectionDemo demo = new SyncCollectionDemo();
        // 创建一个核心线程数、最大线程数都是10 的线程池
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        // 给线程池提交10个任务
        for (int i = 0; i < 10; i++) {
            // 每个线程执行1000次添加操作
            executorService.submit(()->{
                for (int j = 0; j < 1000; j++) {
                    demo.addSync(j);
                }});
        }
        executorService.shutdown();
        executorService.awaitTermination(3, TimeUnit.SECONDS);
        // 多线程并发写同步容器不会造成数据不一致
        System.out.println(demo.getListSync().size());
    }
}
```
ArrayList 被 Collections.synchronizedList 包装成了一个线程安全的 List，多线程并发读写同步容器不会造成数据不一致。

### 同步容器的问题
同步容器实现线程安全的方式就是将它们的状态封装起来，**并在需要同步的方法上加上关键字synchronized。**
这样做的代价是锁住了整个容器，从而削弱了并发性。当多个线程共同竞争容器级的锁时，吞吐量就会降低。


**为了解决同步容器的性能问题，所以后来 Java 中又引入了并发容器。**
## 从同步容器到并发容器
同步容器将所有对容器状态的访问都串行化，以保证线程安全性，这种策略会严重降低并发性。
Java 1.5 后在 java.util.concurrent 包--即J.U.C 中提供了多种并发容器，**使用并发容器来替代同步容器，可以极大地提高伸缩性并降低风险**。

J.U.C 包中提供了几个非常有用的并发容器作为线程安全的容器：

| 并发容器 | 对应的普通容器 | 描述 |
| --- | --- | --- |
| ConcurrentHashMap | HashMap | Java 1.8 之前采用分段锁机制细化锁粒度，降低阻塞，从而提高并发性；Java 1.8 之后基于 CAS 实现。 |
| ConcurrentSkipListMap | SortedMap | 基于跳表实现的 |
| CopyOnWriteArrayList | ArrayList |  |
| CopyOnWriteArraySet | Set | 基于 CopyOnWriteArrayList 实现。 |
| ConcurrentSkipListSet | SortedSet | 基于 ConcurrentSkipListMap 实现。 |
| ConcurrentLinkedQueue | Queue | 线程安全的无界队列。底层采用单链表。支持 FIFO。 |
| ConcurrentLinkedDeque | Deque | 线程安全的无界双端队列。底层采用双向链表。支持 FIFO 和 FILO。 |
| ArrayBlockingQueue | Queue | 数组实现的阻塞队列。 |
| LinkedBlockingQueue | Queue | 链表实现的阻塞队列。 |
| LinkedBlockingDeque | Deque | 双向链表实现的双端阻塞队列。 |


J.U.C 包中提供的并发容器根据命名一般分为三类：

- Concurrent
   - 这类型的锁竞争相对于 CopyOnWrite 要高一些，但写操作代价要小一些。
   - 此外，**Concurrent 往往提供了较低的遍历一致性，即：当利用迭代器遍历时，如果容器发生修改，迭代器仍然可以继续进行遍历。代价就是，在获取容器大小 size() ，容器是否为空等方法，不一定完全精确，但这是为了获取并发吞吐量的设计取舍**，可以理解。与之相比，如果是使用同步容器，就会出现 fail-fast 问题，即：检测到容器在遍历过程中发生了修改，则抛出 ConcurrentModificationException，不再继续遍历。
- CopyOnWrite - 一个线程写，多个线程读。读操作时不加锁，写操作时通过在副本上加锁保证并发安全，空间开销较大。
- Blocking - 内部实现一般是基于锁，提供阻塞队列的能力。

> fail-fast ---快速失败机制一般出现在当某个线程在遍历容器时，其他线程恰好修改了这个容器的长度。这种机制会报错提示ConcurrentModificationException，一般使用同步容器的时候才会出现这个问题。


### 并发Map
J.U.C 提供的并发 Map 容器有 ConcurrentHashMap 和 ConcurrentSkipListMap，它们从应用的角度来看，主要区别在于**ConcurrentHashMap 的 key 是无序的，而 ConcurrentSkipListMap 的 key 是有序的**。所以如果你需要保证 key 的顺序，就只能使用 ConcurrentSkipListMap。

**使用 ConcurrentHashMap 和 ConcurrentSkipListMap 需要注意的地方是，它们的 key 和 value 都不能为空，否则会抛出NullPointerException这个运行时异常。**

ConcurrentHashMap 的实现包含了 HashMap 所有的基本特性，如：数据结构、读写策略等，所以它的基本操作与 HashMap 的用法基本一样。

#### **ConcurrentHashMap 使用示例**
```java
public class ConcurrentHashMapDemo {

    public static void main(String[] args) throws InterruptedException {

        // HashMap 在并发迭代访问时会抛出 ConcurrentModificationException 异常
        // Map<Integer, String> map = new HashMap<>();
        Map<Integer, String> map = new ConcurrentHashMap<>();

        Thread writeThread = new Thread(() -> {
            System.out.println("写操作线程开始执行");
            for (int i = 0; i < 26; i++) {
                map.put(i, String.valueOf((char) ('a' + i)));
            }
        });

        Thread readThread = new Thread(() -> {
            System.out.println("读操作线程开始执行");
            for (Map.Entry<Integer, String> entry : map.entrySet()) {
                System.out.println(entry.getKey() + " - " + entry.getValue());
            }
        });
        writeThread.start();
        readThread.start();
        Thread.sleep(2000);
    }
}
```
使用 ConcurrentHashMap 时的注意事项：

- 使用了 ConcurrentHashMap，不代表对它的多个操作之间的状态是一致的，是没有其他线程在操作它的，如果需要确保需要手动加锁。
- **诸如 size、isEmpty 和 containsValue 等聚合方法，在并发情况下可能会反映 ConcurrentHashMap 的中间状态，并非实时精确值。这是一种策略上的权衡**，在并发环境下，这类方法由于总在不断变化，所以获取其实时精确值的意义不大。ConcurrentHashMap 弱化这类方法，以换取更重要操作（如：get、put、containesKey、remove 等）的性能。
- 因此在并发情况下，这些方法的返回值只能用作参考，而不能用于流程控制。显然，利用 size 方法计算差异值，是一个流程控制。
- 诸如 putAll 这样的聚合方法也不能确保原子性，在 putAll 的过程中去获取数据可能会获取到部分数据。



#### ConcurrentHashMap 的实现原理
ConcurrentHashMap 一直在演进，尤其在 Java 1.7 和 Java 1.8，其数据结构和并发机制有很大的差异。

- Java 1.7
   - 数据结构：数组＋单链表
   - 并发机制：采用分段锁机制细化锁粒度，把整个Map 切分为16个分段（segment），相当于一个Map 有16把锁，各管不同的分段从而降低阻塞，提高并发性。
- Java 1.8
   - 数据结构：数组＋单链表＋红黑树
   - 并发机制：取消分段锁，基于 CAS + synchronized 实现。

### 并发List-- CopyOnWriteArrayList
CopyOnWriteArrayList 是线程安全的 ArrayList。CopyOnWrite 字面意思为写的时候会将共享变量新复制一份出来。复制的好处在于读操作是无锁的（也就是无阻塞）。

CopyOnWriteArrayList 仅适用于写操作非常少的场景，而且能够容忍读写的短暂不一致。如果读写比例均衡或者有大量写操作的话，使用 CopyOnWriteArrayList 的性能会非常糟糕。

#### CopyOnWriteArrayList 使用示例
```java
public class CopyOnWriteArrayListDemo {

    static class ReadTask implements Runnable {

        List<String> list;

        ReadTask(List<String> list) {
            this.list = list;
        }

        public void run() {
            for (String str : list) {
                System.out.println(str);
            }
        }
    }

    static class WriteTask implements Runnable {

        List<String> list;
        int index;

        WriteTask(List<String> list, int index) {
            this.list = list;
            this.index = index;
        }

        public void run() {
            list.remove(index);
            list.add(index, "write_" + index);
        }
    }

    public static void main(String[] args) {
        final int THREAD_NUM = 10;
        // ArrayList 在并发迭代访问时会抛出 ConcurrentModificationException 异常
        // List<String> list = new ArrayList<>();
        CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
        for (int i = 0; i < THREAD_NUM; i++) {
            list.add("main_" + i);
        }
        ExecutorService executorService = Executors.newFixedThreadPool(THREAD_NUM);
        for (int i = 0; i < THREAD_NUM; i++) {
            executorService.execute(new ReadTask(list));
            executorService.execute(new WriteTask(list, i));
        }
        executorService.shutdown();
    }
}
```

### 并发Queue
J.U.C 里面 Queue 这类并发容器是最复杂的，可以从两个维度来对它们进行分类。一个维度是阻塞与非阻塞，**所谓阻塞指的是：当队列已满时，入队操作阻塞；当队列已空时，出队操作阻塞**。另一个维度是单端与双端，**单端指的是只能队尾入队，队首出队；而双端指的是队首队尾皆可入队出队**。J.U.C 阻塞队列都用 Blocking 关键字标识，单端队列使用 Queue 标识，双端队列使用 Deque 标识。

并发 Queue 的核心接口是 BlockingQueue。 顾名思义，是一个阻塞队列，该接口继承自 J.U.C 的 Queue 接口。
```java
public interface BlockingQueue<E> extends Queue<E> {}
```

**BlockingQueue 基本都是基于锁实现。在 BlockingQueue 中，当队列已满时，入队操作阻塞；当队列已空时，出队操作阻塞。**

J.U.C 中提供了以下阻塞队列的实现：

- ArrayBlockingQueue - 一个由数组结构组成的有界阻塞队列。
- LinkedBlockingQueue - 一个由链表结构组成的有界阻塞队列。
- PriorityBlockingQueue - 一个支持优先级排序的无界阻塞队列。
- SynchronousQueue - 一个不存储元素的阻塞队列，队列其实是虚的，即队列容量为 0。数据必须从某个写线程交给某个读线程，而不是写到队列中等待被消费。
- DelayQueue - 一个使用优先级队列实现的无界阻塞队列。
- LinkedTransferQueue - 一个由链表结构组成的无界阻塞队列。

下面以常用的 LinkedBlockingQueue 对阻塞队列的使用做一下说明。

#### LinkedBlockingQueue 的使用方法
LinkedBlockingQueue 是由链表结构组成的有界阻塞队列，容易被误解为无边界，但其实其行为和内部代码都是基于有界的逻辑实现的，只不过如果在创建队列时没有指定容量，那么其容量限制就自动被设置为 Integer.MAX_VALUE ，容量太大，就成为了无界队列。

下面是 LinkedBlockingQueue 常用的一些方法，以及他们代表的功能。
```java
public class LinkedBlockingQueueAppMain {

    public static void main(String[] args) throws InterruptedException {
        // TODO 默认是 Integer.MAX_VALUE 这么大
        // TODO 元素不允许为null
        LinkedBlockingQueue<String> linkedBlockingQueue = new LinkedBlockingQueue<>(128);

        // TODO 查看队列头上的元素，但是不出队列
        linkedBlockingQueue.peek();

        // TODO 将元素放入队列，返回是否放入成功。一般在限制队列大小的情况下才会失败，毕竟到达Integer.MAX_VALUE程序可能就因为没有内存挂了
        boolean added = linkedBlockingQueue.offer("");
        // TODO 这个方法也有超时版本
        boolean addedInTime = linkedBlockingQueue.offer("", 1, TimeUnit.SECONDS);

        // TODO 队列里取出数据，没有就返回空，这个方法也有超时重载版本
        linkedBlockingQueue.poll();

        try {
            // TODO 将元素加入队列，如果队列满了，就等着
            linkedBlockingQueue.put("");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // TODO 一定要拿到一个，否则就无限等待
        try {
            linkedBlockingQueue.take();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // TODO put和take可以方便的实现生产者消费者模式

    }
}
```
使用 LinkedBlockingQueue 的 put 和 take 方法可以方便的实现生产者消费者模式。

#### 各种并发 Queue 怎么选择
Queue 被广泛使用在生产者 - 消费者场景。而在并发场景，利用 BlockingQueue 的阻塞机制，可以减少很多并发协调工作。
这么多并发 Queue 的实现，如何选择呢？

- 考虑应用场景中对队列边界的要求，ArrayBlockingQueue 是有明确的容量限制的，而 LinkedBlockingQueue 则取决于我们是否在创建时指定，SynchronousQueue 则干脆不能缓存任何元素。
- 从空间利用角度，数组结构的 ArrayBlockingQueue 要比 LinkedBlockingQueue 紧凑，因为其不需要创建所谓节点，但是其初始分配阶段就需要一段连续的空间，所以初始内存需求更大。
- 通用场景中，LinkedBlockingQueue 的吞吐量一般优于 ArrayBlockingQueue，因为它实现了更加细粒度的锁操作。
- ArrayBlockingQueue 实现比较简单，性能更好预测，属于表现稳定的“选手”。
- 可能令人意外的是，很多时候 SynchronousQueue 的性能表现，往往大大超过其他实现，尤其是在队列元素较小的场景。
