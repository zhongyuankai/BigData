# OOM 分析

## Java 栈内存溢出(Java.lang.StackOverflowError)

虚拟机栈中创建的栈帧过多，导致栈空间不足。

一个方法不断的调用自己将会导致栈内存溢出；

## Java 堆内存溢出（Java.lang.OutOfMemoryError:Java heap space）

在 Java 堆中只要不断的创建对象，并且 `GC-Roots` 到对象之间存在引用链，这样 `JVM` 就不会回收对象。

只要将`-Xms(最小堆)`,`-Xmx(最大堆)` 设置为一样禁止自动扩展堆内存。


当使用一个 `while(true)` 循环来不断创建对象就会发生 `OutOfMemory`，还可以使用 `-XX:+HeapDumpOnOutOfMemoryError` 当发生 OOM 时会自动 dump 堆栈到文件中。

当出现 OOM 时可以通过工具来分析 `GC-Roots` 引用链 ，查看对象和 `GC-Roots` 是如何进行关联的，是否存在对象的生命周期过长，或者是这些对象确实该存在的，那就要考虑将堆内存调大了。

## GC超过了限制（Java.lang.OutOfMemeoryError:GC overhead limit exceeded）

GC回收时间过长时会抛出 `OutOfMemeoryError` 。

过长的定义是，超过`98%`的时间用来做GC并且回收了不到2%的堆内存，连续多次GC都只回收了不到2%的极端情况下才会抛出。

假如不抛出`GC overhead limit`错误，会导致GC清理的那点内存很快会再次填满，迫使GC再次执行。这样就形成恶性循环，`CPU`使用率一直都是`100%`，而GC却没有任何成果。

```java
/**
 * @author zyk
 * -Xms10m -Xmx10m -XX:MaxDirectMemorySize=5m -XX:+PrintGCDetails
 */
public class GCOverheadDemo {

    public static void main(String[] args) {
        int i = 0;
        List<String> list = new ArrayList<>();
        try {
            while(true) {
                // String的intern()方法就是扩充常量池的一个方法；当一个String实例str调用intern()方法时，
				// Java查找常量池中是否有相同Unicode的字符串常量，如果有，则返回其的引用，
				// 如果没有，则在常量池中增加一个Unicode等于str的字符串并返回它的引用
                list.add(String.valueOf(i++).intern());
            }
        } catch (Throwable e) {
		// Throwable类是Java 语言中所有错误或异常的超类。 它的两个子类是Error和Exception；
            System.out.println(i + "=============");
            e.printStackTrace();
        }
    }
}

```

## 直接内存溢出（Java.lang.OutOfMemeoryError:Direct buffer memory）

写NIO程序经常使用`ByteBuffer`来读取或写入数据，这是基于通道（Channel）与缓冲区（Buffer）的`I/O`方式。它可以使用`Native`函数库直接分配对外内存，然后通过一个存储在Java堆里面的`DirectByteBuffer`对象作为这块内存的引用进行操作。

> ByteBuffer.allocate(capability) 分配JVM堆内存，属于GC管辖范围，由于需要拷贝速度相对较慢；
> ByteBuffer.allocateDirect(capability) 分配OS本地内存，不属于GC管辖范围，由于不需要内存拷贝所以速度相对较快。

但是如果不断分配本地内存，堆内存很少使用，那么JVM就不需要执行GC，`DirectBuffer`对象就不会被回收，这时候堆内存充足，但本地内存可能已经用光了，再次尝试分配本地内存就会出现`OutOfMemoryError`；

```java
/**
 * @author zyk
 * -Xms10m -Xmx10m -XX:MaxDirectMemorySize=5m -XX:+PrintGCDetails
 */
public class DirectBufferMemoryDemo {

    public static void main(String[] args) {
	// 最大直接内存5m，实际要分配6m
        ByteBuffer bb = ByteBuffer.allocateDirect(6 * 1024 * 1024);
    }
}
```

## 不能再创建线程（Java.lang.OutOfMemeoryError:unable to create new native thread）

高并发请求服务器时，经常出现这个异常，准确来讲native thread异常与对应的平台有关；

导致原因：
1. 一个应用创建了太多的线程，超过系统承载极限。
2. 服务器不允许你的应用程序创建那么多线程，linux系统默认允许单个进程可以创建的线程数是1024个，超过了这个数就会报`Java.lang.OutOfMemeoryError:unable to create new native thread`

解决办法：
1. 想办法降低创建线程的数量。
2. 对于有的应用，确实需要创建很多线程，远超过linux系统的默认`1024`个线程的限制，可以通过修改linux服务器配置，扩大Linux默认限制。

```java
public class UnableCreateNewThreadDemo {
    public static void main(String[] args) {
        for (int i = 0; ; i++) {
            new Thread(()->{
                try {
                    TimeUnit.SECONDS.sleep(Integer.MAX_VALUE);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }, "" + i).start();
        }
    }
}
```

## MetaSpace（元空间） 内存溢出(Java.lang.OutOfMemeoryError:Metaspace）

> `JDK8` 中将永久代移除，使用 `MetaSpace` 来保存类加载之后的类信息，字符串常量池也被移动到 Java 堆。

`PermSize` 和 `MaxPermSize` 已经不能使用了，在 JDK8 中配置这两个参数将会发出警告。


JDK 8 中将类信息移到到了本地堆内存(Native Heap)中，将原有的永久代移动到了本地堆中成为 `MetaSpace` ,如果不指定该区域的大小，JVM 将会动态的调整。

可以使用 `-XX:MaxMetaspaceSize=10M` 来限制最大元数据。这样当不停的创建类时将会占满该区域并出现 `OOM`。

```java
    public static void main(String[] args) {
        while (true){
            Enhancer  enhancer = new Enhancer() ;
            enhancer.setSuperclass(HeapOOM.class);
            enhancer.setUseCache(false) ;
            enhancer.setCallback(new MethodInterceptor() {
                @Override
                public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                    return methodProxy.invoke(o,objects) ;
                }
            });
            enhancer.create() ;

        }
    }
```
使用 `cglib` 不停的创建新类，最终会抛出:
```
Caused by: java.lang.reflect.InvocationTargetException
	at sun.reflect.GeneratedMethodAccessor1.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at net.sf.cglib.core.ReflectUtils.defineClass(ReflectUtils.java:459)
	at net.sf.cglib.core.AbstractClassGenerator.generate(AbstractClassGenerator.java:336)
	... 11 more
Caused by: java.lang.OutOfMemoryError: Metaspace
	at java.lang.ClassLoader.defineClass1(Native Method)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:763)
	... 16 more
```

注意：这里的 OOM 伴随的是 `java.lang.OutOfMemoryError: Metaspace` 也就是元数据溢出。


















