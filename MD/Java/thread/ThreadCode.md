**大厂面试手写题**

# 1. 手写阻塞队列

- 思路分析：
    1. 当队列为空的时候，take操作应该阻塞。
    2. 当队列为满的时候，put操作应该阻塞。
	
- `synchronized` 锁代码实现：
```java
class MyBlockingQueue<T> {

    // 队列，存放资源
    private Object[] queue;

    // 存在的资源个数
    private int count;

    // 存数据指针和取数据指针
    private int putIndex, takeIndex;

    // 队列满时的对象锁
    private Object full;

    // 队列空时对象锁
    private Object empty;

    public MyBlockingQueue(int size) {
        queue = new Object[size];
        full = new Object();
        empty = new Object();
    }

    // put元素
    public void put(T obj) {

        // 当队列满，则应该阻塞
        synchronized (full) {
            while (count == queue.length) {
                try {
                    System.out.println("put blocking");
                    full.wait();
                } catch (InterruptedException e) {
                    break;
                }
            }
        }

        // 当线程到这里时，说明可以存放元素了
        synchronized (empty) {
            count++;
            queue[putIndex] = (Object) obj;

            if (++putIndex == queue.length) {
                putIndex = 0;
            }
            //  唤醒其他线程
            empty.notify();
        }
    }

    public T take() {
        // 队列为空，则阻塞
        synchronized (empty) {
            while (count == 0) {
                try {
                    empty.wait();
                    System.out.println("take blocking");
                } catch (InterruptedException e) {
                    return null;
                }
            }
        }
        synchronized (full) {
            T res = (T) queue[takeIndex];
            count--;

            if (++takeIndex == queue.length) {
                takeIndex = 0;
            }

            full.notify();

            return res;
        }
    }
}
```
- `ReentrantLock` 代码实现：
```java
class MyBlockingQueue<T> {
    // 队列，存放资源
    final Object[] items;

    // 记录队列数据的个数
    private int count;

    // 存数据的索引
    private int putIndex;

    // 取数据的索引
    private int takeIndex;

    final ReentrantLock lock;

    // 队列不为空、线程的状态
    private final Condition notEmpty;

    private final Condition notFull;

    public MyBlockingQueue2(int capcity) {
        items = new Object[capcity];
        lock = new ReentrantLock();
        notEmpty = lock.newCondition();
        notFull = lock.newCondition();
    }

    public T take() {
        lock.lock();

        T obj = null;
        try {
            // 当队列为空时，线程应该阻塞
            while (count == 0) {
                notEmpty.await();
            }

            // 为什么要多写这句话，详情见
            final Object[] items = this.items;
            obj = (T) items[takeIndex];

            // 将这块内存释放掉
            items[takeIndex] = null;

            // 如果takeIndex已经到达数组的末尾了，应该让他指向开始
            if (++takeIndex == items.length) {
                takeIndex = 0;
            }

            count--;
            // 唤醒生成的线程
            notFull.signal();

        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
        return obj;
    }

    public void put(Object obj) {
        // 检查当前对象是否为空
        checkNotNulll(obj);
        lock.lock();
        try {
            final Object[] items = this.items;

            // 当队列满时，应该阻塞取的线程
            while (count == items.length) {
                notFull.await();
            }

            items[putIndex++] = obj;

            // 当putIndex达到末尾时，将其指向0
            if (putIndex == items.length) {
                putIndex = 0;
            }

            count++;
            // 唤醒消费的线程
            notEmpty.signal();

        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }

    }

    private void checkNotNulll(Object obj) {
        if (obj == null) {
            throw new NullPointerException();
        }
    }

}
```

# 2. 两个线程交替打印1-100

- 等待通知机制：两个线程通过对同一对象调用等待 wait() 和通知 notify() 方法来进行通讯。

- 代码实现：
```java
/**
 * 两个线程交替执行打印 1~100
 *
 * @author zyk
 */
public class TwoThreadPrint {

    // 线程操作的资源
    private int count;

    // 用于线程间通信的标志，注意用volatile保证内存可见性
    private volatile boolean flag;

    public void one() {
        // 锁住资源对象
        synchronized (this) {
            // flag为true时，奇数线程等待
            while (flag) {
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            count++;
            // 打印当前的数字
            System.out.println(Thread.currentThread().getName() + "----" + count);
            // 将flag改为true
            flag = true;
            // 唤醒偶数线程
            this.notify();
        }
    }

    public void two() {
        // 与奇数线程相反
        synchronized (this) {
            while (!flag) {
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            count++;
            flag = false;
            System.out.println(Thread.currentThread().getName() + "-----" + count);
            this.notify();
        }
    }

    public static void main(String[] args) {
        TwoThreadPrint twoThreadPrint = new TwoThreadPrint();
        new Thread(() -> {
            for (int i = 0; i < 50; i++) {
                twoThreadPrint.one();
            }
        }).start();
        new Thread(() -> {
            for (int i = 0; i < 50; i++) {
                twoThreadPrint.two();
            }
        }).start();
        System.out.println("test");

        // 锁Class对象，wait的时候也必须是Class对象
        synchronized (Object.class) {
            try {
                Object.class.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

    }

}

```

# 3. 三个线程交替打印1-100

- 三个线程通过Condition来精准通信；

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 三个线程轮流打印1-100
 *
 * @author zyk
 */
public class ThreeThreadPrint {

    // 资源
    private int count = 1;

    // 锁对象
    private final ReentrantLock lock;

    // 控制第一个线程
    private final Condition cond1;

    // 控制第二个线程
    private final Condition cond2;

    // 控制第三个线程
    private final Condition cond3;

    public ThreeThreadPrint() {
        this.lock = new ReentrantLock();
        this.cond1 = this.lock.newCondition();
        this.cond2 = this.lock.newCondition();
        this.cond3 = this.lock.newCondition();
    }

    public void onePrint() {
        while (count < 100) {
            this.lock.lock();
            try {
                // 这里需要判断当前count是不是满足打印要求，不满足等待
                while (count % 3 != 1) {
                    cond1.await();
                }

                System.out.println(Thread.currentThread().getName() + "---" + count);
                count++;

                // 唤醒下一个打印的线程
                cond2.signal();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                this.lock.unlock();
            }
        }
    }

    public void twoPrint() {
        while (count < 100) {

            this.lock.lock();
            try {

                while (count % 3 != 2) {
                    cond2.await();
                }

                System.out.println(Thread.currentThread().getName() + "---" + count);
                count++;
                cond3.signal();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                this.lock.unlock();
            }
        }
    }

    public void threePrint() {
        while (count < 100) {

            this.lock.lock();
            try {

                while (count % 3 != 0) {
                    cond3.await();
                }

                System.out.println(Thread.currentThread().getName() + "---" + count);
                count++;
                cond1.signal();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                this.lock.unlock();
            }
        }
    }

    public static void main(String[] args) {
        ThreeThreadPrint threeThreadPrint = new ThreeThreadPrint();

        new Thread(() -> {
            threeThreadPrint.onePrint();
        }, "Thread-1").start();
        new Thread(() -> {
            threeThreadPrint.twoPrint();
        }, "Thread-2").start();
        new Thread(() -> {
            threeThreadPrint.threePrint();
        }, "Thread-3").start();
    }
}
```