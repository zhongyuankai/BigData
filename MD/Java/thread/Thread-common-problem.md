# Java 多线程常见问题

## 上下文切换
多线程并不一定是要在多核处理器才支持的，就算是单核也是可以支持多线程的。
CPU 通过给每个线程分配一定的时间片，由于时间非常短通常是几十毫秒，所以 CPU 可以不停的切换线程执行任务从而达到了多线程的效果。

但是由于在线程切换的时候需要保存本次执行的信息，在该线程被 CPU 剥夺时间片后又再次运行恢复上次所保存的信息的过程就称为上下文切换。

> 上下文切换是非常耗效率的。

通常有以下解决方案:
- 采用无锁编程，比如将数据按照 `Hash(id)` 进行取模分段，每个线程处理各自分段的数据，从而避免使用锁。
- 采用 CAS(compare and swap) 算法，如 `Atomic` 包就是采用 CAS 算法。
- 合理的创建线程，避免创建了一些线程但其中大部分都是处于 `waiting` 状态，因为每当从 `waiting` 状态切换到 `running` 状态都是一次上下文切换。

## 死锁

死锁的场景一般是：线程 A 和线程 B 都在互相等待对方释放锁，或者是其中某个线程在释放锁的时候出现异常如死循环之类的。这时就会导致系统不可用。

![](img/deadLock.png)

常用的解决方案如下：

- 尽量一个线程只获取一个锁。
- 一个线程只占用一个资源。
- 尝试使用定时锁，至少能保证锁最终会被释放。

死锁代码实现：
```java
class HoldThread implements Runnable {

    private String lockA;
    private String lockB;

    public HoldThread(String lockA, String lockB) {
        this.lockA = lockA;
        this.lockB = lockB;
    }

    @Override
    public void run() {
        synchronized (lockA) {
            System.out.println(Thread.currentThread().getName() + "\t 自己持有锁" + lockA + "尝试获得" + lockB);
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (lockB) {
                System.out.println(Thread.currentThread().getName() + "\t 自己持有锁" + lockB + "尝试获得" + lockA);
            }
        }
    }
}

public class DeadLockDemo {
    public static void main(String[] args) {
        String lockA = "lockA";
        String lockB = "lockB";
        new Thread(new HoldThread(lockA, lockB), "threadAAA").start();
        new Thread(new HoldThread(lockB, lockA), "threadBBB").start();
    }
}
```

**死锁的定位分析**：
- jps命令定位进程编号: jps -l
- jstack找到死锁查看: jstack 进程号

## 资源限制

当在带宽有限的情况下一个线程下载某个资源需要 `1M/S`,当开 10 个线程时速度并不会乘 10 倍，反而还会增加时间，毕竟上下文切换比较耗时。

如果是受限于资源的话可以采用集群来处理任务，不同的机器来处理不同的数据，就类似于开始提到的无锁编程。

## ArrayList是线程不安全,请给出解决方案
- Vector是线程安全；
- Collections.synchronizedList(List l)函数返回一个线程安全的ArrayList类；
- concurrent并发包下的CopyOnWriteArrayList类。

## 什么是并发容器的实现？

Java集合类都是**快速失败**的，这就意味着当集合被改变且一个线程在使用迭代器遍历集合的时候，迭代器的next()方法将抛出ConcurrentModificationException异常。

**并发容器支持并发的遍历和并发的更新**。

主要的类有ConcurrentHashMap, CopyOnWriteArrayList 和CopyOnWriteArraySet，












