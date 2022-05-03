线程并发时的同步控制 (加锁)，是为了保证多个线程对共享数据争用时的正确性的。那如果一个操作本身不涉及对共享数据的使用，相反，只是希望变量只能由创建它的线程使用（即线程隔离）就需要到线程本地存储了。

Java 通过 ThreadLocal  提供了程序对线程本地存储的使用。通过创建 ThreadLocal 类的实例，让我们能够创建只能由同一线程读取和写入的变量。因此，即使两个线程正在执行相同的代码，并且代码引用了相同名称的 ThreadLocal 变量，这两个线程也无法看到彼此的存储在 ThreadLocal 里的值。否则也就不能叫线程本地存储了。

## ThreadLocal 
ThreadLocal 是 Java 内置的类，全称 java.lang.ThreadLoal， java.lang 包里定义的类和接口在程序里都是可以直接使用，不需要导入的。

ThreadLocal 的类定义如下：
```java
public class ThreadLocal<T> {
    public T get() {
        // ...
    }
    public void set(T value) {
    	// ...
    }
    public void remove() {
    	// ...
    }
    public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
    	// ...
    }
    
    // ...
}
```
上面只是列出了 ThreadLocal 类里我们经常会用到的方法，他们的说明如下。

- get - 用于获取 ThreadLocal 在当前线程中保存的变量副本。
- set - 用于设置当前线程中 变量的副本。
- remove - 用于删除当前线程中变量的副本。
- initialValue - 为 ThreadLocal 设置默认的 get 初始值，需要重写 initialValue 方法 。

下面我们详细看一下 ThreadLocal 的使用。

## 创建和读写 ThreadLocal

通过上面 ThreadLocal 类的定义我们能看出来， ThreadLocal 是支持泛型的，所以在创建 ThreadLocal 时没有啥特殊情况，我们都会为其提供类型参数，这样在读取使用 ThreadLocal 变量时就能免去类型转换的操作。
```java
private ThreadLocal threadLocal = new ThreadLocal();
threadLocal.set("A thread local value");
// 创建时没有使用泛型指定类型，默认是 Object
// 使用时要先做类型转换
String threadLocalValue = (String) threadLocal.get();
```
上面这个例子，在创建 ThreadLocal 时没有使用泛型指定类型，所以存储在其中的值默认是 Object 类型，这样就需要在使用时先做类型转换才行。

下面再看一个使用泛型的版本
```java
private ThreadLocal<String> myThreadLocal = new ThreadLocal<String>();

myThreadLocal.set("Hello ThreadLocal");
String threadLocalValue = myThreadLocal.get();
```
现在我们只能把 String 类型的值存到 ThreadLocal 中，并且从 ThreadLocal 读取出值后也不再需要进行类型转换。

想要删除一个 ThreadLocal 实例里存储的值，只需要调用实例的 remove 方法即可。
```java
myThreadLocal.remove();
```
当然，这个删除只是删除的变量在本地线程的副本，其他线程不受影响。下面我们把 ThreadLocal 的创建、读写和删除攒一个简单的例子，做下演示。
```java
public class ThreadLocalExample {

    private  ThreadLocal<Integer> threadLocal = new ThreadLocal<>();

    private void setAndPrintThreadLocal() {
        threadLocal.set((int) (Math.random() * 100D) );
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println( Thread.currentThread().getName() + ": " + threadLocal.get() );

        if ( threadLocal.get() % 2 == 0) {
            // 测试删除 ThreadLocal
            System.out.println(Thread.currentThread().getName() + ": 删除ThreadLocal");
            threadLocal.remove();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        ThreadLocalExample tlExample = new ThreadLocalExample();
        Thread thread1 = new Thread(() -> tlExample.setAndPrintThreadLocal(), "线程1");
        Thread thread2 = new Thread(() -> tlExample.setAndPrintThreadLocal(), "线程2");

        thread1.start();
        thread2.start();

        thread1.join();
        thread2.join();
    }
}
```
上面的例程会有如下输出，当然如果恰好两个线程里 threadLocal 变量里存储的都是偶数的话，就不会有第三行输出啦。
```java
线程2: 97
线程1: 64
线程1: 删除ThreadLocal
```
## 为 ThreadLocal 设置初始值
我们可以为 ThreadLocal 设置一个初始值，这样在没有使用 set 方法给 ThreadLocal 设置值的情况下调用 get() 将返回 ThreadLocal 的初始值。

有两种方式可以指定 ThreadLocal 的初始值：

- 创建一个 ThreadLocal 的子类，覆盖 initialValue() 方法，程序中使用子类创建 ThreadLocal 实例
- 使用 ThreadLocal 类的静态方法 withInital 创建 ThreadLocal 实例，该方法接收一个函数式接口 Supplier 的实现作为参数，在 Supplier 实现中为 ThreadLocal 设置初始值。
### 使用子类覆盖 initialValue() 设置初始值
通过 ThreadLocal 子类覆盖 initialValue() 方法的方式给 ThreadLocal 变量设置初始值的方式，可以使用匿名类，简化创建子类的步骤。

下面我们在程序里创建 ThreadLocal 实例时，直接使用匿名类来覆盖 initialValue() 方法。
```java
public class ThreadLocalExample {

    private ThreadLocal threadLocal = new ThreadLocal<Integer>() {
        @Override protected Integer initialValue() {
            return (int) System.currentTimeMillis();
        }
    };
    
	......   
}
```
其实还能使用 Lambda 让它更简化些，不过这样就看不出来我们是覆盖的 initialValue () 方法啦，Lambda 我们放到使用 withInital 静态方法设置初始值时再演示。
### 通过 withInital 静态方法设置初始值
为 ThreadLocal 实例变量指定初始值的第二种方法是使用 ThreadLocal 类的静态工厂方法 withInitial 。withInitial 方法接收一个函数式接口 Supplier 的实现作为参数，在 Supplier 实现中为 ThreadLocal 设置初始值。

> Supplier 接口是一个函数式接口，表示提供某种值的函数。 Supplier 接口也可以被认为是工厂接口。
> 
> @FunctionalInterface
> public interface Supplier<T> { T get(); }


下面的程序里，我们用 ThreadLocal 的 withInitial 方法为 ThreadLocal 实例变量设置了初始值
```java
public class ThreadLocalExample {

    private ThreadLocal<Integer> threadLocal = ThreadLocal.withInitial(new Supplier<Integer>() {
        @Override
        public String get() {
            return (int) System.currentTimeMillis();
        }
    });
    
	......   
}
```
对于函数式接口，理所当然会想到用 Lambda 来实现。上面这个 withInitial 的例子用 Lambda 实现的话能进一步简化成：
```java
public class ThreadLocalExample {

	private ThreadLocal<Integer> threadLocal = ThreadLocal.withInitial(() -> (int) System.currentTimeMillis());
	......
}
```



## ThreadLocal 的实现原理
在 Thread 类中维护着一个 ThreadLocal.ThreadLocalMap 类型的成员 threadLocals。这个成员就是用来存储当前线程独占的变量副本的。
```java
public class Thread implements Runnable {
    // ...
    ThreadLocal.ThreadLocalMap threadLocals = null;
    // ...
}
```
ThreadLocalMap 是 ThreadLocal 的静态内部类。
```java
package java.lang;

public class ThreadLocal<T> {
    // ...
	static class ThreadLocalMap {
    	// ...
    	static class Entry extends WeakReference<ThreadLocal<?>> {
        	/** The value associated with this ThreadLocal. */
        	Object value;

        	Entry(ThreadLocal<?> k, Object v) {
            	super(k);
            	value = v;
        	}
    	}
    	// ...
	}
}
```
它维护着一个 Entry 数组，Entry 继承了 WeakReference ，所以是弱引用。 Entry 用于保存键值对，其中：

- key 是 ThreadLocal 对象；
- value 是传递进来的对象（变量副本）。

ThreadLocalMap 虽然是类似 Map 结构的数据结构，但它解决 Hash 冲突的方式并非像 HashMap 那样使用拉链法（用链表保存冲突的元素）。

实际上，ThreadLocalMap 采用了线性探测的方式来解决 Hash 冲突。所谓线性探测，就是根据初始 key 的 hashcode 值确定元素在哈希表数组中的位置，如果发现这个位置上已经被其他的 key 值占用，则利用固定的算法寻找一定步长的下个位置，依次判断，直至找到能够存放的位置。

