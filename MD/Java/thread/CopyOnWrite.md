# CopyOnWrite容器 源码分析

Copy-On-Write简称Cow,是一种用于程序设计中的优化策略。从JDK1.5开始Java并发包里提供了两个使用CopyOnWrite机制实现的并发容器,它们是**CopyOnWriteArrayList**和**CopyOnWriteArraySet**, CopyOnWrite容器非常有用,可以在非常多的并发场景中使用到。

`CopyOnWrite`容器即**写时复制的容器**。通俗的理解是当我们往一个容器添加元素的时候,不直接往当前容器添加,而是先将当前容器进行Copy,复制出一个新的容器,然后新的容器里添加元素,添加完元素之后,再将原容器的引用指向新的容器。这样做的好处是我们可以对CopyOnWrite容器进行并发的读,而不需要加锁,因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的;

## CopyOnWriteArrayList的实现原理
```java
public boolean add(E e) {
	// 可重入锁
	final ReentrantLock lock = this.lock;
	// 对容器写之前上锁
	lock.lock();
	try {
		// 写之前拿到原来容器的所有元素
		Object[] elements = getArray();

		int len = elements.length;
		// 将原来的容器元素拷贝到新的数组中，并且长度+1
		Object[] newElements = Arrays.copyOf(elements, len + 1);
		// 将元素添加到尾部
		newElements[len] = e;
		// CopyOnWriteArrayList内部引用指向新的数组
		setArray(newElements);
		return true;
	} finally {
		// 释放锁
		lock.unlock();
	}
}

//setArray() 内部源码
final void setArray(Object[] a) {
	//CopyOnWriteArrayList内部引用指向新的数组
	array = a
}

// 读的方法
public E get(int index) {
	return get(getArray(), index);
}
```


## CopyOnWrite的应用场景

CopyOnWrite并发容器用于**读多写少的并发场景**。比如白名单,黑名单,商品类目的访问和更新场景,假如我们有一个搜索网站,用户在这个网站的搜索框中,输入关键字搜索内容,但是某些关键字不允许被搜索。这些不能被搜索的关键字会被放在一个黑名单当中,黑名单每天晚上更新一次。当用户搜索时,会检查当前关键字在不在黑名单当中,如果在,则提示不能搜索。

## CopyOnWrite的缺点

CopyOnWrite容器有很多优点,但是同时也存在两个问题,即内存占用问题和数据一致性问题。

**内存占用问题**：因为CopyOnWrite的写时复制机制,所以在进行写操作的时候,内存里会**同时驻扎两个对象的内存**,旧的对象和新写入的对象 。如果这些对象占用的内存比较大,比如说200M左右,那么再写入100M数据进去,内存就会占用300M,那么这个时候很有可能造成频繁的Yong GC和Full GC。系统中使用了一个服务由于每晚使用CopyOnWrite机制更新大对象,造成了每晚15秒的Full GC,应用响应时间也随之变长。

**数据一致性问题**：CopyOnWrite容器只能保证数据的最终一致性,**不能保证数据的实时一致性**。所以如果你希望写入的的数据,马上能读到,请不要使用CopyOnWrite容器。






