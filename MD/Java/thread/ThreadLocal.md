# 深入剖析ThreadLocal

ThreadLocal，很多地方叫做线程本地变量，也有些地方叫做线程本地存储。当使用**ThreadLocal维护变量时ThreadLocal的确是数据的隔离,但是并非数据的复制,而是在每一个线程中创建一个新的数据对象,然后每一个线程使用的是不一样的**。

## ThreadLocal 提供的方法

```java
// 用来获取ThreadLocal在当前线程中保存的变量副本
public T get() { }
// 用来设置当前线程中变量的副本
public void set(T value) { }
// 用来移除当前线程中变量的副本
public void remove() { }
// 是一个protected方法，一般是用来在使用时进行重写的，它是一个延迟加载方法
protected T initialValue() { }
```

## 原理
`ThreadLocal`连接`ThreadLocalMap`和`Thread`,用来处理Thread中的TheadLocalMap属性,包括init初始化属性赋值、get对应的变量, set设置变量等。

通过获取当前线程来获取线程上的`ThreadLocalMap`属性，对数据进行get,set等操作。ThreadLocalMap是实际存储数据的,采用类似hashmap机制,**存储了以threadLocal为key,需要隔离的数据为value的Entry键值对数组结构**。

## ThreadLocal、ThreadLocalMap和Thread的关系
`ThreadLocalMap`是`ThreadLocal`静态内部类;

`Thread`有`ThreadLocal.ThreadLocalMap`类型的属性;

但是`Thread`中的`ThreadLocalMap`属性是由`ThreadLocal`来初始化的。

Thread源码如下:

```java
public
class Thread implements Runnable {
/*...其他属性...*/
/* ThreadLocal values pertaining to this thread. This map is maintained
* by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
/*
* InheritableThreadLocal values pertaining to this thread. This map is
* maintained by the InheritableThreadLocal class.
*/
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

ThreadLocal源码如下：
```java
public class ThreadLocal<T> {

	static class ThreadLocalMap {

		// Entry 继承了弱引用，方便GC
		static class Entry extends WeakReference<ThreadLocal<?>> {
		/** The value associated with this ThreadLocal. */
			Object value;
			Entry(ThreadLocal<?> k, Object v) {
				super(k);
				value = v;
			}
		}
	}
```

由ThreadLocal对Thread的ThreadLocalMap进行赋值:
```java
/**
* Create the map associated with a ThreadLocal. Overridden in
* InheritableThreadLocal.
*
* @param t the current thread
* @param firstValue value for the initial entry of the map
*/
void createMap(Thread t, T firstValue) {
t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

## Thread同步机制的比较ThreadLocal和线程同步机制相比的优势

`Synchronized`用于线程间的数据共享,而`ThreadLocal`则用于线程间的数据隔离。

在同步机制中,通过对象的锁机制保证同一时间只有一个线程访问变量。这时该变量是多个线程共享的,使用同步机制要求程序慎密地分析什么时候对变量进行读写,什么时候需要锁定某个对象,什么时候释放对象锁等繁杂的问题,程序设计和编写难度相对较大。

而ThreadLocal则从另一个角度来解决多线程的并发访问。ThreadLocal会为每一个线程提供一个独立的变量,从而隔离了多个线程对数据的访问冲突。因为每一个线程都拥有自己的变量,从而也就没有必要对该变量进行同步了。

ThreadLocal提供了线程安全的共享对象,在编写多线程代码时,可以把不安全的变量封装进ThreadLocal.

概括起来说,对于多线程资源共享的问题,同步机制采用了"以时间换空间"的方式,而ThreadLocal采用了"以空间换时间"的方式。前者仅提供一份变量,让不同的线程排队访问,而后者为每一个线程都提供了一份变量,因此可以同时访问而互不影响。

## 总结:

ThreadLocal是解决线程安全问题一个很好的思路,它通过为每个线程提供一个独立的变量解决了变量并发访问的冲突问题。在很多情况下, ThreadLocal比直接使用synchronized同步机制解决线程安全问题更简单,更方便,且结果程序拥有更高的并发性。


[ThreadLocal更多的源码分析可以点击这里](https://www.cnblogs.com/dolphin0520/p/3920407.html)











