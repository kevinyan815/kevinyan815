## 线程池框架 Executor
 J.U.C 里的 Executor 框架中提供了一系列组件帮助我们以更简便的方式管理线程池。Executor 是框架的最顶层接口，在其基础上框架扩展出功能更多的接口以及提供了它们的实现。Executor 框架里的核心组件如下：

| 类型 | 名称 | 描述 |
| --- | --- | --- |
| 接口 | Executor  | 最上层接口，定义了执行任务的 execute 方法。 |
| 接口 | ExecutorService | 扩展了 Executor 接口，支持有返回值的线程任务和管理线程的生命周期 |
| 接口 | ScheduledExecutorService | 扩展了ExecutorService，支持定期地执行任务 |
| 实现类 | ThreadPoolExecutor | 核心实现类，是基础、标准线程池的实现 |
| 实现类 | ScheduledThreadPoolExecutor | 扩展了ThreadPoolExecutor，同时实现了ScheduledExecutorService接口，支持定时执行任务 |
| 工具类 | Executors | 可以通过调用 Executors 的静态工厂方法创建线程池，并返回一个 ExecutorService 对象。Exectors 中提供了创建几种J.U.C默认提供的线程池的工厂方法，这些线程池都是基于 ThreadPoolExecutor 实现的 |


### Executor
Executor 接口是线程池最上层的接口，只定义了一个 execute 方法，用于接收一个 Runnable 对象。
```java
public interface Executor {
    void execute(Runnable command);
}
```
这个方法用来向线程池提交一个任务，交由线程池去执行。
### ExecutorService
ExecutorService 接口扩展了 Executor 接口，除了 execute 方法外它还支持 invokeAll、invokeAny、submit这些提交有返回值的线程任务的方法，这几个方法都支持传入 Callable 接口实现作为线程任务，同时返回一个 Future 实现用于查询线程执行的结果或者控制取消线程执行。

除此之外 ExecutorService 还增加了管理线程池生命周期的 shutdown、shutdownNow 等方法。
```java
public interface ExecutorService extends Executor {
    
    // 优雅关闭线程池，之前提交的任务将被执行，但是不会接受新的任务。
    void shutdown();

    // 尝试停止所有正在执行的任务，停止等待任务的处理，并返回等待执行任务的列表。
    List<Runnable> shutdownNow();

    // 如果此线程池已关闭，则返回true.
    boolean isShutdown();

    // 如果关闭后的所有任务都已完成，则返回true
    boolean isTerminated();

    // 监测ExecutorService是否已经关闭，直到所有任务完成执行，或超时发生，或当前线程被中断。
    boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;

    // 提交一个用于执行的Callable返回任务，并返回一个Future，用于获取Callable执行结果。
    <T> Future<T> submit(Callable<T> task);

    // 提交可运行任务以执行，并返回Future，执行结果为传入的result
    <T> Future<T> submit(Runnable task, T result);

    // 提交可运行任务以执行，并返回Future对象，执行结果为null
    Future<?> submit(Runnable task);

    // 执行给定的任务集合，执行完毕后，则返回结果。
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException;

    // 执行给定的任务集合，执行完毕或者超时后，则返回结果，其他任务终止。
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException;

    // 执行给定的任务，任意一个执行成功则返回结果，其他任务终止。
    <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException;

    // 执行给定的任务，任意一个执行成功或者超时后，则返回结果，其他任务终止
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,long timeout, TimeUnit unit)throws InterruptedException, ExecutionException, TimeoutException;
}
```
相比于 Executor 接口，ExecutorService 接口主要的扩展是：

- 支持提交有返回值的线程任务 - sumbit、invokeAll、invokeAny 方法中都支持传入Callable 对象。
- 支持管理线程生命周期 - shutdown、shutdownNow、isShutdown 等方法。
### ScheduledExecutorService
ScheduledExecutorService 扩展了 ExecutorService 接口，增加了对定时线程调度的支持。
```java
public interface ScheduledExecutorService extends ExecutorService {

    /**
     * 创建并执行一个一次性任务，过了延迟时间就会被执行
     */
    public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit);

    /**
     * 创建并执行一个一次性任务，过了延迟时间就会被执行
     */
    public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit);

    /**
     * 创建并执行一个周期性任务，过了给定的初始化延迟时间，会第一次被执行。执行过程中发生了异常，那么任务停止
     * 一次任务执行时长超过了周期时间，下一次任务会等到该次任务执行结束后，立刻执行，这也是它和scheduleWithTixedDelay的重要区别
     */
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);

    /**
     * 创建并执行一个周期性任务，过了给定的初始化延迟时间，会第一次被执行。执行过程中发生了异常，那么任务停止
     * 一次任务执行时长超过了周期时间，下一次任务会在该次任务执行结束的时间基础上，计算执行延时。
     * 对于超时周期的长时间处理任务的不同处理方式，这是它和scheduleAtFixedRate的重要区别
     */
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);

}

```
### ThreadPoolExecutor
ThreadPoolExecutor 是 ExecutorService 接口实现类，它是 Executor 框架中最核心的类，是基础、标准线程池的实现，框架中提供的各种规格的线程池都是基于 ThreadPoolExecutor 实现的。

ThreadPoolExecutor类提供了 4 个版本的构造方法，可根据需要来自定义一个线程池。参数最全的构造方法包含 7 个参数：
```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        // 省略...
    }
```
这 7 个参数的含义分别是：
（1）corePoolSize：核心线程数，线程池中始终存活的线程数。
（2）maximumPoolSize: 最大线程数，线程池中允许的最大线程数。
（3）keepAliveTime: 存活时间，线程没有任务执行时最多保持多久时间会终止。
（4）unit: 单位，参数keepAliveTime的时间单位，7种可选。

| 参数 | 描述 |
| --- | --- |
| TimeUnit.DAYS | 天 |
| TimeUnit.HOURS | 小时 |
| TimeUnit.MINUTES | 分 |
| TimeUnit.SECONDS | 秒 |
| TimeUnit.MILLISECONDS | 毫秒 （1毫秒等于一千分之一秒） |
| TimeUnit.MICROSECONDS | 微妙（1微秒等于一百万分之一秒） |
| TimeUnit.NANOSECONDS | 纳秒（1纳秒等于十亿分之一秒） |

（5）workQueue: 一个阻塞队列，用来存储等待执行的任务，均为线程安全，7种可选。

| 参数 | 描述 |
| --- | --- |
| ArrayBlockingQueue | 一个由数组结构组成的有界阻塞队列。 |
| LinkedBlockingQueue | 一个由链表结构组成的有界阻塞队列。 |
| SynchronousQueue | 一个不存储元素的阻塞队列，即直接提交给线程不保持它们。 |
| PriorityBlockingQueue | 一个支持优先级排序的无界阻塞队列。 |
| DelayQueue | 一个使用优先级队列实现的无界阻塞队列，只有在延迟期满时才能从中提取元素。 |
| LinkedTransferQueue | 一个由链表结构组成的无界阻塞队列。与SynchronousQueue类似，还含有非阻塞方法。 |
| LinkedBlockingDeque | 一个由链表结构组成的双向阻塞队列。 |

较常用的是LinkedBlockingQueue和Synchronous。线程池的排队策略与BlockingQueue有关。
（6）threadFactory: 线程工厂，主要用来创建线程，默及正常优先级、非守护线程。
（7）handler：拒绝策略，拒绝处理任务时的策略，4种可选，默认为AbortPolicy。

| 参数 | 描述 |
| --- | --- |
| AbortPolicy | 拒绝并抛出异常。 |
| CallerRunsPolicy | 重试提交当前的任务，即再次调用运行该任务的execute()方法。 |
| DiscardOldestPolicy | 抛弃队列头部（最旧）的一个任务，并执行当前任务。 |
| DiscardPolicy | 抛弃当前任务。 |

### 线程池执行任务逻辑和规则
执行逻辑说明：
- 判断核心线程数是否已满，核心线程数由 corePoolSize 参数指定，未满则创建线程执行任务。
- 若核心线程池已满，判断队列是否满，队列是否满和 workQueue 参数有关，若未满则加入队列中。
- 若队列已满，判断线程池是否已满，线程池已满即已达到最大线程数，这个数由 maximumPoolSize 参数指定，若未满创建线程执行任务。
- 若线程池已满，则采用拒绝策略处理无法执执行的任务，拒绝策略和handler参数有关

### Executors 工具类
Executors 类里提供了创建几种固定规格的线程池的静态工厂方法，调用他们可以创建线程池并返回管理线程池的 ExecutorService 对象。
> 不推荐使用 Executors 提供的线程池创建方式，它创建出来的线程池都有一定的问题，容易造成OOM。应该使用 ThreadPoolExecutor 自己定制线程池。


#### newFixedThreadPool(int nThreads)
创建一个固定大小、任务队列容量为无界队列的线程池。核心线程数=最大线程数。
#### newCachedThreadPool()
创建的是一个缓冲线程池，池的核心线程数=0，最大线程=Integer.MAX_VALUE，任务队列是同步队列SynchronousQueue，任务直接提交给线程不保存它们。因为Integer.MAX_VALUE非常大，可以认为是可以无限创建线程的，在资源有限的情况下容易引起OOM异常
#### newSingleThreadExecutor()
只有一个线程来执行无界任务队列的单一线程池。该线程池可以确保任务按照加入顺序执行。当唯一的线程因任务异常中止时，将创建一个新的线程来继续执行后续的任务。

因为往队列中可以插入无限多的任务，在资源有限的时候容易引起OOM异常，同时因为无界队列，maximumPoolSize和keepAliveTime参数将无效，压根就不会创建非核心线程
#### newScheduledThreadPool(int corePoolSize)
能定时执行任务的线程池。该池的核心线程数由参数指定，最大线程数=Integer.MAX_VALUE。
#### newWorkStealingPool
Java 8 才引入，其内部会构建 ForkJoinPool，利用 Work-Stealing 算法，并行地处理任务，不保证处理顺序。

#### Executors 创建的线程池问题总结

- FixedThreadPool 和 SingleThreadExecutor  允许的请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而引起OOM异常
- CachedThreadPool  和  ScheduledThreadPool 允许创建的线程数为Integer.MAX_VALUE，可能会创建大量的线程，从而引起OOM异常

这就是为什么禁止使用 Executors 去创建线程池，而是推荐自己去创建 ThreadPoolExecutor 的原因。文章最后我们会给出使用 ThreadPoolExecutor 创建线程池的最佳实践。现在让我们来看一下线程池常用的任务处理方法都是怎么用的。

## 线程池的常用方法示例

ExecutorService 里定义了多种向线程池提交任务执行的方法：

- execute(Runnable)
- submit(Runnable)
- submit(Callable)
- invokeAny(...)
- invokeAll(...)

这几个方法的参数和返回值我们在上面已经看过，像submit 、invokeAny、invokeAll 都是有多个重载版本的，具体的参数可以看文章上面介绍 ExecutorService 接口定义的部分。下面我们看下这几个方法如何使用。
### execute 方法
execute 方法接收一个 Runnable 类型的参数，把它作为任务执行体提交给线程池的线程执行。
```java
package com.learnconcurrent.threadpool;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.*;

public class ThreadPoolExecutorDemo {

    public ExecutorService executorService;

    public static void main(String[] args) {
        ThreadPoolExecutorDemo threadPoolDemo = new ThreadPoolExecutorDemo();
        threadPoolDemo.executeDemo();
    }

    public ThreadPoolExecutorDemo() {
        // 用 ThreadPoolExecutor 创建一个线程池
        executorService = new ThreadPoolExecutor(2, 10,
                1, TimeUnit.MINUTES, new ArrayBlockingQueue<>(5, true),
                Executors.defaultThreadFactory(), new ThreadPoolExecutor.AbortPolicy());
    }

    public void executeDemo() {
        for (int i = 0; i < 10; i++) {
            final int index = i;
            executorService.execute(() -> {
                String pattern = "yyyy-MM-dd HH:mm:ss";
                SimpleDateFormat simpleDateFormat = new SimpleDateFormat(pattern);
                // 获取线程名称,默认格式:pool-1-thread-1
                System.out.println(simpleDateFormat.format(new Date()) + " " + Thread.currentThread().getName() + " " + index);
                // 等待2秒
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
    }
}

```
上面例程里，我们创建了一个核心线程数为 2 最大线程数为 10 任务队列长度为 5 的阻塞队列。用 execute 方法向提交了 10 个任务，根据线程池的执行逻辑：因为队列为 5 所以会把后5个任务放入队列中，其他5个先执行，因此我们的例程只会创建 5 个线程，重复使用完成所有任务。

例程运行后的输出如下：
```java
2022-02-27 09:28:58 pool-1-thread-5 9
2022-02-27 09:28:58 pool-1-thread-4 8
2022-02-27 09:28:58 pool-1-thread-2 1
2022-02-27 09:28:58 pool-1-thread-3 7
2022-02-27 09:28:58 pool-1-thread-1 0
2022-02-27 09:29:00 pool-1-thread-3 3
2022-02-27 09:29:00 pool-1-thread-1 4
2022-02-27 09:29:00 pool-1-thread-2 2
2022-02-27 09:29:00 pool-1-thread-5 6
2022-02-27 09:29:00 pool-1-thread-4 5
```
### submit 方法
ExecutorService 里有两个版本的 submit 方法，他们分别支持向线程池提交 Runnable 或者 Callable 实现作为线程要执行的任务。
> Callable 和 Runnable 的区别，请查看[线程基础中的章节](https://www.yuque.com/u21195004/grw5kw/anukmg#OFTK5)


用 submit 提交 Runnable 任务，跟直接使用 execute 方法的区别不大
```java
Future future = executorService.submit(new Runnable() {
    public void run() {
        System.out.println("task-1");
    }
});

future.get();  // 线程执行结束后，get 方法会返回 null
```
由于 Runnable 是不带返回值的线程任务执行体，所以 submit 返回的 Future 实现，在线程执行结束后 get 到的是 null。

Callable 代表是带返回值的线程任务执行体，用 submit 方法提交给线程池后，可以通过 submit 方法返回的 Future 实现获取线程任务执行后的返回值。
```java
public class ThreadPoolExecutorDemo {

    public ExecutorService executorService;

    public static void main(String[] args) {
        ThreadPoolExecutorDemo threadPoolDemo = new ThreadPoolExecutorDemo();
//        threadPoolDemo.executeDemo();
        threadPoolDemo.submitCallableDemo();
    }
    
    ...

    public void submitCallableDemo() {
        Future future = executorService.submit(new Callable(){
            public Object call() throws Exception {
                System.out.println(Thread.currentThread().getName() + " 线程开始执行任务。");
                return "Callable Result";
            }
        });

        try {
            System.out.println("执行结果是：" + future.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}

```
上面例程的执行后的输出如下
```java
pool-1-thread-1 线程开始执行任务。
执行结果是：Callable Result
```
### 
### invokeAny 方法
使用 invokeAny 方法可以向线程池提交一组任务，它接收一个 Callable 对象组成的 Collection 集合作为参数，返回结果是这组任务中第一个执行完的任务的返回值，与此同时线程池会终止这组任务中的其他任务。除此之外，如果 invokeAny 提交的一组任务中，某个任务执行中出现异常，其他任务也会被线程池终止。

```java
public class ThreadPoolExecutorDemo {

    public ExecutorService executorService;

    public static void main(String[] args) {
        ThreadPoolExecutorDemo threadPoolDemo = new ThreadPoolExecutorDemo();

        threadPoolDemo.invokeAnyDemo();
        // 关闭线程池
        threadPoolDemo.executorService.shutdown();
    }


    public void invokeAnyDemo() {
        Set<Callable<String>> callables = new HashSet<>();
        callables.add(() -> "Task 1");
        callables.add(() -> "Task 2");
        callables.add(() -> "Task 3");
        try {
            String result = executorService.invokeAny(callables);
            System.out.println("result = " + result);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```
上面的程序，可以试着多执行几次，会观察到不同的返回结果。
### invokeAll 方法
invokeAll 也是接收一个 Callable 对象组成的 Collection 集合作为参数，把它们提交给线程池，让线程池去执行这些任务。与 invokeAny 不同的是它返回的是一个 Future 列表，列表中的每个 Future 对象都与其中一个 Callable 任务相关联，通过它们能够查到每个 Callable 任务的执行结果。
```java

public class ThreadPoolExecutorDemo {

    public ExecutorService executorService;

    public static void main(String[] args) {
        ThreadPoolExecutorDemo threadPoolDemo = new ThreadPoolExecutorDemo();

        threadPoolDemo.invokeAllDemo();
        // 关闭线程池
        threadPoolDemo.executorService.shutdown();
    }


    public void invokeAllDemo() {
        Set<Callable<String>> callables = new HashSet<>();

        callables.add(() -> "Task 1");
        callables.add(() -> "Task 2");
        callables.add(() -> "Task 3");

        try {
            List<Future<String>> futures = executorService.invokeAll(callables);
            for(Future<String> future : futures){
                System.out.println("future.get = " + future.get());
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}

```
运行上面例程，会有类似下面的输出：
```java
future.get = Task 3
future.get = Task 1
future.get = Task 2
```

## 线程池管理相关的方法
ExecutorService 中还定义了线程池管理相关的方法：

- shutdown：不会立即终止线程池，而是要等所有任务缓存队列中的任务都执行完后才终止，但再也不会接受新的任务。
- shutdownNow:  立即终止线程池，并尝试打断正在执行的任务，并且清空任务缓存队列，返回尚未执行的任务列表。
- isShutdown：调用了 shutdown 或 shutdownNow 方法后，isShutdown 方法就会返回 true。
- isTerminated: 当所有的任务都已关闭后，才表示线程池关闭成功，这时调用 isTerminaed 方法会返回 true。

除了 ExecutorService 里定义的这几个方法外，在 ThreadPoolExecutor 类中也定义了一些监控线程池状态的方法。

- getTaskCount： 线程池已经执行的和未执行的任务总数；
- getCompletedTaskCount： 线程池已完成的任务数量，该值小于等于 taskCount。
- getLargestPoolSize： 线程池曾经创建过的最大线程数量。通过这个数据可以知道线程池是否满过，也就是达到了线程池的最大线程数。
- getPoolSize - 线程池当前的线程数量。
- getActiveCount - 当前线程池中正在执行任务的线程数量。

## 线程池的最佳实践
### 如认合适的线程数量
一般多线程执行的任务类型可以分为 CPU 密集型和 I/O 密集型，根据不同的任务类型，我们计算线程数的方法也不一样。规则是：线程等待时间(IO)所占比例越高，需要越多线程；线程 CPU 时间所占比例越高，需要越少线程。

**CPU 密集型任务**：这种任务消耗的主要是 CPU 资源，可以将线程数设置为 **N（CPU 核心数）+1**，比 CPU 核心数多出来的一个线程是为了防止线程偶发的缺页中断，或者其它原因导致的任务暂停而带来的影响。一旦任务暂停，CPU 就会处于空闲状态，而在这种情况下多出来的一个线程就可以充分利用 CPU 的空闲时间。

**I/O 密集型任务**：执行这种任务，系统会用大部分的时间来处理 I/O 交互，而线程在处理 I/O 的时间段内不会占用 CPU 时间，这时就可以将 CPU 交出给其它线程使用。因此在 I/O 密集型任务的应用中，我们可以多配置一些线程，**推荐的设置是把线程数设置成 CPU 核心数的两倍**。 

### 使用有界阻塞队列
上面介绍 Executors 的静态工厂方法创建的线程池不建议使用，不建议使用 Executors 最主要的原因是，Executors 提供的很多方法默认使用的都是无界的 LinkedBlockingQueue，高负载情境下，无界队列很容易导致 OOM，而 OOM 会导致所有请求都无法处理，这是致命问题。所以强烈建议使用有界队列。

我们上面的线程池使用示例中，就是使用的 ThreadPoolExecutor 创建的线程池，把线程池的任务队列设置成了有界队列。
```java
// 用 ThreadPoolExecutor 创建一个线程池，指定核心线程数、最大线程数、任务队列等参数配置
new ThreadPoolExecutor(2, 10,
                       1, TimeUnit.MINUTES, 
                       new ArrayBlockingQueue<>(5, true),
                       Executors.defaultThreadFactory(),
                       new ThreadPoolExecutor.AbortPolicy());
```

### 重要任务应该自定义拒绝策略
使用有界队列，当任务过多时，线程池会触发执行拒绝策略，线程池默认的拒绝策略会 throw RejectedExecutionException 这是个运行时异常，对于运行时异常编译器并不强制 catch 它，所以开发人员很容易忽略。因此默认拒绝策略要慎重使用。如果线程池处理的任务非常重要，建议自定义自己的拒绝策略；并且在实际工作中，自定义的拒绝策略往往和降级策略配合使用。



