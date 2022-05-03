计算机系统里每个进程（Process）都代表着一个运行着的程序，进程是对运行时程序的封装，系统进行资源调度和分配的基本单位。

一个进程下可以有很多个线程，线程是进程的子任务，**是CPU调度和分派的基本单位**，**用于保证程序的实时性，实现进程内部的并发；线程是操作系统可识别的最小执行和调度单位**。

在 Java 里线程是程序执行的载体，我们写的代码就是由线程运行的。

## Java 中的线程
到目前为止，我们写的所有 Java 程序代码都是在由JVM给创建的 Main Thread 中单线程里执行的。Java 线程就像一个虚拟 CPU，可以在运行的 Java 应用程序中执行 Java 代码。当一个 Java 应用程序启动时，它的入口方法 main() 方法由主线程执行。主线程（Main Thread）是一个由 Java 虚拟机创建的运行你的应用程序的特殊线程。

因为 Java  里一切皆对象，所以线程也是用对象表示的，线程是类 java.lang.Thread 类或者其子类的实例。在 Java 应用程序内部， 我们可以通过 Thread 实例创建和启动更多线程，这些线程可以与主线程并行执行应用程序的代码。 

### 创建和启动线程
在 Java 中创建一个线程，就是创建一个 Thread 类的实例
```java
  Thread thread = new Thread();
```
启动线程就是调用线程对象的 start() 方法
```java
  thread.start();
```
当然，这个例子没有指定线程要执行的代码，所以线程将在启动后立即停止。 

## 指定线程要执行的代码

有两种方法可以给线程指定要执行的代码。

- 第一种是，创建 Thread 的子类，覆盖父类的 run() 方法，在run() 方法中指定线程要执行的代码。
- 第二种是，将实现 Runnable (java.lang.Runnable) 的对象传递给 Thread 构造方法，创建Thread 实例。

下面我们看看这三种的用法和区别。
### 通过 Thread 子类指定要执行的代码
通过继承 Thread 类创建线程的步骤：

1. 定义 Thread 类的子类，并覆盖该类的 run() 方法。run() 方法的方法体就代表了线程要完成的任务，因此把 run 方法称为执行体。
1. 创建 Thread 子类的实例，即创建了线程对象。
1. 调用线程对象的 start 方法来启动该线程。
```java
public class ThreadDemo {

    public static void main(String[] args) {
        // 实例化线程对象
        MyThread threadA = new MyThread("Thread 线程-A");
        MyThread threadB = new MyThread("Thread 线程-B");
        // 启动线程
        threadA.start();
        threadB.start();
    }

    static class MyThread extends Thread {

        private int ticket = 5;

        MyThread(String name) {
            super(name);
        }

        @Override
        public void run() {
            while (ticket > 0) {
                System.out.println(Thread.currentThread().getName() + " 卖出了第 " + ticket + " 张票");
                ticket--;
            }
        }

    }

}
```
上面的程序，主线程启动调用A、B两个线程的 start() 后，并没有通过wait() 等待他们执行结束。A、B两个线程的执行体，会并发地被系统执行，等线程都直接结束后，程序才会退出。
### 通过实现 Runnable 接口指定要执行的代码
Runnable 接口里，只有一个 run() 方法的定义：
```java
package java.lang;
public interface Runnable {
    public abstract void run();
}
```
其实，Thread 类实现的也是 Runnable 接口。
在 Thread 类的重载构造方法里，支持接收一个实现了 Runnale 接口的对象作为其 target 参数来初始化线程对象。
```java
public class Thread implements Runnable {
    
    ...
	public Thread(Runnable target) {
        init(null, target, "Thread-" + nextThreadNum(), 0);
    }
    
    ...
    public Thread(Runnable target, String name) {
        init(null, target, name, 0);
    }
    ...

}

```
通过实现 Runnable 接口创建线程的步骤：

1. 定义 Runnable 接口的实现，实现该接口的 run 方法。该 run 方法的方法体同样是线程的执行体。
1. 创建 Runnable 实现类的实例，并以此实例作为 Thread 的 target 来创建 Thread 对象，该 Thread 对象才是真正的线程对象。
1. 调用线程对象的 start 方法来启动线程并执行。
```java

public class RunnableDemo {

    public static void main(String[] args) {
        // 实例化线程对象
        Thread threadA = new Thread(new MyThread(), "Runnable 线程-A");
        Thread threadB = new Thread(new MyThread(), "Runnable 线程-B");
        // 启动线程
        threadA.start();
        threadB.start();
    }

    static class MyThread implements Runnable {

        private int ticket = 5;

        @Override
        public void run() {
            while (ticket > 0) {
                System.out.println(Thread.currentThread().getName() + " 卖出了第 " + ticket + " 张票");
                ticket--;
            }
        }

    }

}
```
运行上面例程会有以下输出，同样程序会在所以线程执行完后退出。
```java

Runnable 线程-B 卖出了第 5 张票
Runnable 线程-B 卖出了第 4 张票
Runnable 线程-B 卖出了第 3 张票
Runnable 线程-B 卖出了第 2 张票
Runnable 线程-B 卖出了第 1 张票
Runnable 线程-A 卖出了第 5 张票
Runnable 线程-A 卖出了第 4 张票
Runnable 线程-A 卖出了第 3 张票
Runnable 线程-A 卖出了第 2 张票
Runnable 线程-A 卖出了第 1 张票

Process finished with exit code 0
```
既然是给 Thread 传递 Runnable 接口的实现对象即可，那么除了普通的定义类实现接口的方式，我们还可以使用匿名类和 Lambda 表达式的方式来定义 Runnable 的实现。

- 使用 Runnable 的匿名类作为参数创建 Thread 对象:
```java
Thread threadA = new Thread(new Runnable() {
    private int ticket = 5;
    @Override
    public void run() {
        while (ticket > 0) {
            System.out.println(Thread.currentThread().getName() + " 卖出了第 " + ticket + " 张票");
            ticket--;
        }
    }
}, "Runnable 线程-A");
```

- 使用实现了 Runnable 的 Lambda 表达式作为参数创建 Thread 对象:
```java
Runnable runnable = () -> { System.out.println("Lambda Runnable running"); };
Thread threadB = new Thread(runnable, "Runnable 线程-B");

```
因为，Lambda 是无状态的，定义不了内部属性，这里就举个简单的打印一行输出的例子了，理解一下这种用法即可。

### 通过Callable 和 Future 获取线程任务的返回值
上面两种方法虽然能指定线程执行体里要执行的任务，但是都没有返回值，如果想让线程的执行体方法有返回值，且能被外部创建它的父线程获取到返回值，就需要结合J.U.C 里提供的 Callable、Future 接口来实现线程的执行体方法才行。
> J.U.C 是 java.util.concurrent 包的缩写，提供了很多并发编程的工具类，后面会详细学习。


Callable 接口只声明了一个方法，这个方法叫做 call()：
```java
package java.util.concurrent;

public interface Callable<V> {
    V call() throws Exception;
}
```
Future 就是对于具体的 Callable 任务的执行进行取消、查询是否完成、获取执行结果的。**可以通过 get 方法获取 Callable 的 call 方法的执行结果，但是要注意该方法会阻塞直到任务返回结果**。
```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
Java 的 J.U.C 里给出了  Future 接口的一个实现 FutureTask，它同时实现了 Future 和 Runnable 接口，所以，FutureTask 既可以作为 Runnable 被线程执行，又可以作为 Future 得到 Callable 的返回值。

下面是一个 Callable 实现类和 FutureTask 结合使用让主线程获取子线程执行结果的一个简单的示例：
```java

public class CallableDemo implements Callable<Integer> {
    @Override
    public Integer call() {
        int i = 0;
        for (i = 0; i < 20; i++) {
            if (i == 5) {
                break;
            }
            System.out.println(Thread.currentThread().getName() + " " + i);
        }
        return i;
    }

    public static void main(String[] args) {
        CallableDemo tt = new CallableDemo();
        FutureTask<Integer> ft = new FutureTask<>(tt);
        Thread t = new Thread(ft);
        t.start();
        try {
            System.out.println(Thread.currentThread().getName() + " " + ft.get());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
上面我们把 FutureTask 作为 Thread 构造方法的 Runnable 类型参数 target 的实参，在它的基础上创建线程，
执行逻辑。所以本质上 Callable + FutureTask 这种方式也是第二种通过实现 Runnable 接口给线程指定执行体的，只不过是由 FutureTask 包装了一层，由它的 run 方法再去调用 Callable 的 call 方法。例程运行后的输出如下：
```java
Thread-0 0
Thread-0 1
Thread-0 2
Thread-0 3
Thread-0 4
main 5
```
> **Callable 更常用的方式是结合线程池来使用，在线程池接口  ExecutorService 中定义了多个可接收 Callable 作为线程执行任务的方法 submit、invokeAny、invokeAll 等，这个等学到线程池了我们再去学习。**


## 初学者常见陷阱--在主线程里调用 run()
在刚开始接触和学习 Java 线程相关的知识时，一个常见的错误是，在创建线程的线程里，调用 Thread 对象的 run() 方法而不是调用 start() 方法。
```java
Runnable myRunnable = new Runnable() {
    @Override
    public void run() {
		System.out.println("Anonymous Runnable running");
    }
};
Thread newThread = new Thread(myRunnable);
newThread.run();  // 应该调用 newThread.start();
```
起初你可能没有注意到这么干有啥错，因为 Runnable 的 run() 方法正常地被执行，输出了我们想要的结果。

但是，这么做 run() 不会由我们刚刚创建的新线程执行，而是由创建 newThread 对象的线程执行的 。要让新创建的线程--newThread 调用 myRunnable 实例的 run() 方法，必须调用 newThread.start() 方法才行。

## 线程的基本用法

### 线程对象常用的方法
Thread 线程常用的方法有以下这些：

| 方法 | 描述 |
| --- | --- |
| run | 线程的执行实体，不需要我们主动调用，调用线程的start() 就会执行run() 方法里的执行体 |
| start | 线程的启动方法。 |
| Thread.currentThread | Thread 类提供的静态方法，返回对当前正在执行的线程对象的引用。 |
| setName | 设置线程名称。 |
| getName | 获取线程名称。 |
| setPriority | 设置线程优先级。Java 中的线程优先级的范围是 [1,10]，一般来说，高优先级的线程在运行时会具有优先权。可以通过 thread.setPriority(Thread.MAX_PRIORITY) 的方式设置，默认优先级为 5。 |
| getPriority | 获取线程优先级。 |
| setDaemon | 设置线程为守护线程。 |
| isDaemon | 判断线程是否为守护线程。 |
| isAlive | 判断线程是否启动。 |
| interrupt | 中断线程的运行。 |
| Thread.interrupted | 测试当前线程是否已被中断。 |
| join | 可以使一个线程强制运行，线程强制运行期间，其他线程无法运行，必须等待此线程完成之后才可以继续执行。 |
| Thread.sleep | 静态方法。将当前正在执行的线程休眠。 |
| Thread.yield | 静态方法。将当前正在执行的线程暂停，让出CPU，让其他线程执行。 |

### 线程休眠
**使用 Thread.sleep 方法可以使得当前正在执行的线程进入休眠状态。**
使用 Thread.sleep 需要向其传入一个整数值，这个值表示线程将要休眠的毫秒数。
Thread.sleep 方法可能会抛出 InterruptedException，因为异常不能跨线程传播回主线程中，因此必须在本地进行处理。线程中抛出的其它异常也同样需要在本地进行处理。
```java
public class ThreadSleepDemo {

    public static void main(String[] args) {
        new Thread(new MyThread("线程A", 500)).start();
        new Thread(new MyThread("线程B", 1000)).start();
        new Thread(new MyThread("线程C", 1500)).start();
    }

    static class MyThread implements Runnable {

        /** 线程名称 */
        private String name;

        /** 休眠时间 */
        private int time;

        private MyThread(String name, int time) {
            this.name = name;
            this.time = time;
        }

        @Override
        public void run() {
            try {
                // 休眠指定的时间
                Thread.sleep(this.time);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(this.name + "休眠" + this.time + "毫秒。");
        }

    }

}
```
上面例程开启了3个线程，在各自的线程执行体里让各自线程休眠了 500、1000 和 1500 ms ，线程 C 休眠结束后，整个程序退出。
```java
线程A休眠500毫秒。
线程B休眠1000毫秒。
线程C休眠1500毫秒。

Process finished with exit code 0
```
### 终止线程
当一个线程运行时，另一个线程可以直接通过 interrupt 方法中断其运行状态。
```java
public class ThreadInterruptDemo {

    public static void main(String[] args) {
        MyThread mt = new MyThread(); // 实例化Runnable实现类的对象
        Thread t = new Thread(mt, "线程"); // 实例化Thread对象
        t.start(); // 启动线程
        try {
            Thread.sleep(2000); // 主线程休眠2秒
        } catch (InterruptedException e) {
            System.out.println("主线程休眠被终止");
        }
        t.interrupt(); // 中断 mt 线程的执行
    }

    static class MyThread implements Runnable {

        @Override
        public void run() {
            System.out.println("1、进入run()方法");
            try {
                Thread.sleep(10000); // 线程休眠10秒
                System.out.println("2、已经完成了休眠");
            } catch (InterruptedException e) {
                System.out.println("3、MyThread线程休眠被终止");
                return; // 返回调用处
            }
            System.out.println("4、run()方法正常结束");
        }
    }
}
```
如果一个线程的 run 方法执行一个无限循环，并且没有执行 sleep 等会抛出 InterruptedException 的操作，那么调用线程的 interrupt 方法就无法使线程提前结束。

不过调用 interrupt 方法会设置线程的中断标记，此时被设置中断标记的线程再调用 interrupted 方法会返回 true。因此可以在线程的执行体循环体中使用 interrupted 方法来判断当前线程是否处于中断状态，从而提前结束线程。

看下面这个，可以有效终止线程执行的示例
```java
package com.learnthread;

import java.util.concurrent.TimeUnit;

public class ThreadInterruptEffectivelyDemo {
    public static void main(String[] args) throws Exception {
        MyTask task = new MyTask();
        Thread thread = new Thread(task, "线程-A");
        thread.start();
        TimeUnit.MILLISECONDS.sleep(50);
        thread.interrupt();
    }

    private static class MyTask implements Runnable {

        private volatile long count = 0L;

        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + " 线程启动");
            // 通过 Thread.interrupted 和 interrupt 配合来控制线程终止
            while (!Thread.interrupted()) {
                System.out.println(count++);
            }
            System.out.println(Thread.currentThread().getName() + " 线程终止");
        }
    }
}
```
主线程在启动线程-A后，主动休眠50毫秒，线程-A的执行体里会不断打印计数器的值，等休眠结束后主线程通过调用线程-A的 interrupt 方法设置了线程的中断标记，这时线程-A的执行体中通过 Thread.interrupted() 就能判断出线程被设置了中断状态，随后结束执行退出。


## 线程的生命周期
java.lang.Thread.State 中定义了 6 种不同的线程状态，在给定的一个时刻，线程只能处于其中的一个状态。

以下是各状态的说明，以及状态间的联系：

- **新建（New）** - 尚未调用 start 方法的线程处于此状态。此状态意味着：**线程创建了但尚未启动**。
- **就绪（Runnable）** - 已经调用了 start 方法的线程处于此状态。此状态意味着：**线程已经在 JVM 中运行**。但是在操作系统层面，它可能处于运行状态，也可能等待资源调度（例如处理器资源），资源调度完成就进入运行状态。所以该状态的可运行是指可以被运行，具体有没有运行要看底层操作系统的资源调度。
- **阻塞（Blocked）** - 此状态意味着：**线程处于被阻塞状态**。表示线程在等待 synchronized 的隐式锁（Monitor lock）。被 synchronized 修饰的方法、代码块同一时刻只允许一个线程执行，其他线程只能等待，即处于阻塞状态。当占用 synchronized 隐式锁的线程释放锁，并且等待的线程获得 synchronized 隐式锁后，就又会从 BLOCKED 转换到 RUNNABLE 状态。
- **等待（Waiting）** - 此状态意味着：**线程无限期等待，直到被其他线程显式地唤醒**。 阻塞和等待的区别在于，阻塞是被动的，它是在等待获取 synchronized 的隐式锁。而等待是主动的，线程通过调用 Object.wait 等方法进入。

- **定时等待（Timed waiting）** - 此状态意味着：**无需等待其它线程显式地唤醒，在一定时间之后会被系统自动唤醒，**线程通过调用设置了 Timeout 参数的Object.wait 等方法进入。

- **终止(Terminated)** - 线程执行完 run 方法，或者因异常退出了 run 方法。此状态意味着：线程结束了生命周期。

下面这张图更生动地展示了线程状态切换的occasion 和 trigger。

![image](https://user-images.githubusercontent.com/8792672/166452574-0a93cc6a-1387-45cf-83f6-cf4d301b3387.png)



