

## 锁定义
Java 的 synchronized 关键字属于内置锁，除了内置锁外，另外在J.U.C 中的锁是通过 Lock 接口来定义的，提供的锁实现都实现了此接口。
```java
package java.util.concurrent.locks;

import java.util.concurrent.TimeUnit;
    
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```
这些方法的解释如下：

- lock() - 获取锁。
- unlock() - 释放锁。
- tryLock() - 尝试获取锁，仅在调用时锁未被另一个线程持有的情况下，才获取该锁。
- tryLock(long time, TimeUnit unit) - 和 tryLock() 类似，区别仅在于限定了获取锁的超时时间，如果限定时间内未获取到锁，视为失败。
- lockInterruptibly() - 锁未被另一个线程持有，且线程没有被中断的情况下，才能获取锁。
- newCondition() - 返回一个绑定到 Lock 对象上的 Condition 实例。

Condition 实例上提供了挂起、唤醒线程的方法，Object 的 wait、notify 、notifyAll 这些线程通信功能。

Object 类提供的 wait、notify、notifyAll 需要配合 synchronized 使用来进行进程间通信，不适用于 Lock。而使用 Lock 的线程，彼此间通信应该使用 Condition 。这可以理解为，什么样的锁配什么样的钥匙。**内置锁（synchronized）配合内置条件队列（wait、notify、notifyAll ），显式锁（Lock）配合显式条件队列（Condition ）**。

Condition 提供了更丰富的线程间通信功能， 其接口定义如下：
```java
package java.util.concurrent.locks;

import java.util.Date;
import java.util.concurrent.TimeUnit;

public interface Condition {
    void await() throws InterruptedException;
    void awaitUninterruptibly();
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    boolean awaitUntil(Date deadline) throws InterruptedException;
    void signal();
    void signalAll();
}
```
其中，await、signal、signalAll 与Object类的 wait、notify、notifyAll 相对应，功能也相似。除此以外，Condition 提供了更为丰富的功能：

- 每个锁（Lock）上可以存在多个 Condition，这意味着锁的状态条件可以有多个。
- 支持可中断的条件等待，相关方法：awaitUninterruptibly() 。
- 支持可定时的等待，相关方法：awaitNanos(long) 、await(long, TimeUnit)、awaitUntil(Date)。

Condition 需要配合 Lock 使用，进行线程间通信。

JUC 中提供的 Lock 实现，最常用的就是 ReentrantLock 和 ReentrantReadWriteLock，这两个都是可重入锁。

## 可重入锁
**可重入锁，顾名思义，指的是线程可以重复获取同一把锁**。即同一个线程在外层方法获取了锁，在进入内层方法后可以再次获取锁。而第一次没有获取锁的线程，则无法进入内层方法获取到锁。

可重入锁可以在一定程度上避免死锁问题，其实 sychronized 就是一个可重入锁。
```java
class SynchronizedReentrantDemo {
    private synchronized void setA() throws InterruptedException{
        Thread.sleep(1000);
        setB();
    }

    private synchronized void setB() throws InterruptedException{
        Thread.sleep(1000);
    }
}
```
上面的代码就是一个典型场景：成功获取对象锁执行 setA 方法的线程，也能进入 setB 方法，而其他未获取到对象锁的线程，也没办法在调用 setB 这个同步方法时获取对象锁，只能等着。

## 公平锁与非公平锁

- **公平锁** - 公平锁是指 **多线程按照申请锁的顺序来获取锁**。
- **非公平锁** - 非公平锁是指 **多线程不按照申请锁的顺序来获取锁** 。这就可能会出现优先级反转（后来者居上）或者饥饿现象（某线程总是抢不过别的线程，导致始终无法执行）。

公平锁为了保证线程申请顺序，势必要付出一定的性能代价，因此其吞吐量一定会低于非公平锁。Java 的内置锁 sychronized 只支持非公平锁，而我们接下来要学的 ReentrantLock 和 ReentrantReadWriteLock 他们两个默认是非公平锁，但是支持公平锁。

## ReentrantLock
ReentrantLock 见名思议，它是一个可重入锁，**提供了与 synchronized 相同的互斥性、内存可见性和可重入性**，**支持公平锁和非公平锁**（默认）两种模式。ReentrantLock 实现了 JUC 的 Lock 接口，所以更具灵活性（支持TryLock 和 等待锁时允许被中断这些）。
### 创建锁
ReentrantLock 有两个版本的构造方法，默认创建的是非公平锁，创建公平锁需要在 new 实例对象时，给构造函数传递一个 true 布尔值。
```java
ReentrantLock  lock = new ReentrantLock(); // 非公平锁
ReentrantLock fairLock = new ReentrantLock(true); // 公平锁
```
### 获取/释放锁
ReentrantLock 获取锁，使用 Lock 接口的 lock 方法，释放锁使用 unlock 方法。

- lock() - **获取锁**。如果当前线程无法获取锁，则当前线程进入休眠状态，直至当前线程获取到锁。如果该锁没有被另一个线程持有，则获取该锁并立即返回，同时将锁的持有计数设置为 1。
- unlock() - 用于**释放锁**。

由于 ReentrantLock 可重入锁的性质，获取锁之后，线程可以再次通过 lock() 方法获得锁，不过ReentrantLock 在实现上，只是给它加了个计数而已。
```java
Lock lock = new ReentrantLock();

try{
    lock.lock();
    // 临界区代码
} finally {
    lock.unlock();
}
```
请务必牢记，**获取锁的操作 lock() 必须在 try catch 块中进行，并且将释放锁的操作 unlock() 放在 finally 块中进行，以保证锁一定被被释放，防止死锁的发生**。
### 使用示例
下面写一个简单的程序，演示一下 ReentrantLock 的使用，为了好理解，演示方法写的比较简单。
```java
public class ReentrantLockDemo {

    public static void main(String[] args) {
        Task task = new Task();
        for(int i = 1; i <= 3; i++) {
            Thread t = new Thread(new Runnable() {
                @Override
                public void run() {
                    task.execute();
                }
            }, "Thread-" + i);

            t.start();
        }
    }

    static class Task {

        private ReentrantLock lock = new ReentrantLock();

        public void execute() {
            try {
                lock.lock();
                for (int i = 0; i < 3; i++) {
                    System.out.println(lock.toString());

                    // 故意再获取一次锁，查询当前线程 hold 住此锁的次数
                    // 可重入锁会对锁持有数就行累加
                    getHoldCount();

                    // 查询正等待获取此锁的线程数
                    System.out.println("\t queuedLength: " + lock.getQueueLength());

                    // 是否为公平锁
                    System.out.println("\t isFair: " + lock.isFair());

                    // 是否被锁住
                    System.out.println("\t isLocked: " + lock.isLocked());

                    // 是否被当前线程持有锁
                    System.out.println("\t isHeldByCurrentThread: " + lock.isHeldByCurrentThread());

                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            } finally {
                lock.unlock();
            }
        }

        private void getHoldCount() {
            try {
                lock.lock();
                // 查询当前线程 hold 住此锁的次数
                System.out.println("\t holdCount: " + lock.getHoldCount());
            } finally {
                lock.unlock();
            }

        }

    }

}
```
上面例程里面，我们用匿名类实现了 Runnable 接口，开启了三个线程。 线程的执行体里会使用静态内部类 Task 的实例，执行 execute 方法。在 execute 方法和它调用的 getHoldCount 方法的开头我们都应用了 ReentrantLock 获取锁，来演示可重入锁的使用。

例程里除了使用 Lock 接口里定义的 lock 和 unlock 外，还演示了 ReentrantLock 自己定义的一些方法，通过这些方法能够获取锁的状态。

- getHoldCount 查询当前线程 hold 住此锁的次数
- isHeldByCurrentThread 锁是否被当前线程持有
- getQueueLength 查询正等待获取此锁的线程数
- isLocked 查询当前锁是否已经被锁住

### tryLock
与 lock 无条件获取锁，获取不到线程就休眠相比，tryLock 有更完善的容错机制。
tryLock 方法尝试立即获取锁，如果成功返回 true，如果未获取到则返回 false 不会造成阻塞。此外 tryLock 还支持再限定时间内尝试获取锁，在时间内获取不到就返回 false。

- tryLock() - **可轮询获取锁**。如果成功，则返回 true；如果失败，则返回 false。也就是说，这个方法**无论成败都会立即返回**，获取不到锁（锁已被其他线程获取）时不会一直等待。
- tryLock(long, TimeUnit) - **可定时获取锁**。和 tryLock() 类似，区别仅在于这个方法在**获取不到锁时会等待一定的时间**，在时间期限之内如果还获取不到锁，就返回 false。如果如果一开始拿到锁或者在等待期间内拿到了锁，则返回 true。


我们在上一个示例的基础上进行修改，演示 tryLock 的使用。
```java
public void execute() {
    try {
        // 尝试在 1s 内获取到锁，获取不到就退出
        if (lock.tryLock(1, TimeUnit.SECONDS)) {
            try {
                //                lock.lock();
                for (int i = 0; i < 3; i++) {
                    System.out.println(lock.toString());
                    ...
                    }
            } finally {
                lock.unlock();
            }
        } else {
            printLockFailure();
        }
    } catch (InterruptedException ex) {
        ex.printStackTrace();
    }
}

private void printLockFailure() {
    System.out.println(Thread.currentThread().getName() + " failed to get lock");
}
```

### 使用 Condition 实例进行线程间通信
线程使用 Lock 获取锁后，开始逻辑处理，如果需要进行让出锁，或者唤醒其他休眠等待线程的线程间通信时，就不能再使用 Object 类提供的 wait、notify、notifyAll 这些方法了。而使用 Lock 的线程，彼此间通信应该使用 Condition 实例，这个 Condition 实例就是 Lock 接口 newCondition() 方法要求实现类返回的一个绑定到 Lock 对象上的 Condition 实例。

下面还是通过个简单的例子看下怎么使用 Condition 的线程间通信方法。
```java
public class LearnLockWaitNotifyAppMain {
    
    public static void main(String[] args) {
        Lock locker = new ReentrantLock();
        Condition condition = locker.newCondition();
        int workingSec = 2;
        int threadCount = 3;
        for (int i = 0; i < threadCount; i++) {
            new Thread(() -> {
                System.out.println(getName() + ":线程开始工作......");
                try {
                    locker.lock();
                    sleepSec(workingSec);
                    System.out.println(getName() + ":进入等待");
                    // >> TODO await 方法必须在当前线程获取锁之后才能调用
                    // >> TODO await 方法调用后自动失去锁
                    condition.await();
                    System.out.println(getName() + ":线程继续......");
                    sleepSec(workingSec);
                    System.out.println(getName() + ":结束");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    locker.unlock();
                }
            }, "工作线程" + i).start();
        }

        System.out.println("------------- 主线程作为唤醒线程，先sleep -------------");
        sleepSec(workingSec + 1);
        System.out.println("------------- 唤醒线程sleep结束 -------------");
        try {
            locker.lock();
            // >> TODO signal / signalAll 方法必须在当前线程获取锁之后才能调用
            System.out.println("------------- 开始唤醒所有 -------------");
            condition.signalAll();

        } finally {
            locker.unlock();
        }
    }

    private static void sleepSec(int sec) {
        try {
            Thread.sleep(TimeUnit.SECONDS.toMillis(sec));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private static String getName() {
        return Thread.currentThread().getName();
    }
}

```
Condition 的这些方法，必须要在线程获取到 Lock 锁之后调用才有效。

执行上面的例程，会有类似下面的输出：
```java
------------- 主线程作为唤醒线程，先sleep -------------
工作线程0:线程开始工作......
工作线程1:线程开始工作......
工作线程2:线程开始工作......
工作线程2:进入等待
------------- 唤醒线程sleep结束 -------------
工作线程1:进入等待
工作线程0:进入等待
------------- 开始唤醒所有 -------------
工作线程2:线程继续......
工作线程2:结束
工作线程1:线程继续......
工作线程1:结束
工作线程0:线程继续......
工作线程0:结束
```

