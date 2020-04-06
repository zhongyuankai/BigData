# 阻塞队列

- 当阻塞队列是空时,从队列中获取元素的操作将会被阻塞.
- 当阻塞队列是满时,往队列中添加元素的操作将会被阻塞.

## 为什么要使用阻塞队列

不需要关心什么时候需要阻塞线程,什么时候需要唤醒线程,因为BlockingQueue都一手给你包办好了；

在concurrent包发布以前,在多线程环境下,我们每个程序员都必须自己去控制这些细节,尤其还要兼顾效率和线程安全,而这会给我们的程序带来不小的复杂度.

## 阻塞队列的种类

- **ArrayBlockingQueue: 由数组结构组成的有界阻塞队列**
- **LinkedBlockingDeque: 由链表结构组成的有界(但大小默认值Integer>MAX_VALUE)阻塞队列**
- PriorityBlockingQueue: 支持优先级排序的无界阻塞队列
- DelayQueue: 使用优先级队列实现的延迟无界阻塞队列
- **SynchronousQueue:不存储元素的阻塞队列,也即是单个元素的队列**
- LinkedTransferQueue:由链表结构组成的无界阻塞队列
- LinkedBlockingDeque:由了解结构组成的双向阻塞队列

## 场景

- 生产者消费者模式

传统版：
```java
/**
 * 共享资源类
 */
class ShareData {
    private int num = 0;
    private Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();

    public void increment() throws Exception {
        lock.lock();
        try {
            //判断
            while (num != 0) {
                //等待 不生产
                condition.await();
            }
            //干活
            num++;
            System.out.println(Thread.currentThread().getName() + "\t" + num);
            //通知唤醒
            condition.signalAll();
        } finally {
            lock.unlock();
        }
    }

    public void deIncrement() throws Exception {
        lock.lock();
        try {
            //判断
            while (num == 0) {
                //等待 不生产
                condition.await();
            }
            //干活
            num--;
            System.out.println(Thread.currentThread().getName() + "\t" + num);
            //通知唤醒
            condition.signalAll();
        } finally {
            lock.unlock();
        }
    }
}
/**
 * Description
 * 一个初始值为0的变量 两个线程交替操作 一个加1 一个减1来5轮
 **/
public class ProdConsumerTraditionDemo {
    public static void main(String[] args) {
        ShareData shareData = new ShareData();
        new Thread(() -> {
            for (int i = 1; i <= 5; i++) {
                try {
                    shareData.increment();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }, "AA").start();
        new Thread(() -> {
            for (int i = 1; i <= 5; i++) {
                try {
                    shareData.deIncrement();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }, "BB").start();
    }
}
```

阻塞队列版：
```java
class MyResource {
    /**
     * 默认开启 进行生产消费的交互
     */
    private volatile boolean flag = true;
    /**
     * 默认值是0
     */
    private AtomicInteger atomicInteger = new AtomicInteger();

    private BlockingQueue<String> blockingQueue = null;

    public MyResource(BlockingQueue<String> blockingQueue) {
        this.blockingQueue = blockingQueue;
        System.out.println(blockingQueue.getClass().getName());
    }

    public void myProd() throws Exception {
        String data = null;
        boolean returnValue;
        while (flag) {
            data = atomicInteger.incrementAndGet() + "";
            returnValue = blockingQueue.offer(data, 2L, TimeUnit.SECONDS);
            if (returnValue) {
                System.out.println(Thread.currentThread().getName() + "\t 插入队列数据" + data + "成功");
            } else {
                System.out.println(Thread.currentThread().getName() + "\t 插入队列数据" + data + "失败");
            }
            TimeUnit.SECONDS.sleep(1);
        }
        System.out.println(Thread.currentThread().getName() + "\t 停止 表示 flag" + flag);
    }

    public void myConsumer() throws Exception {
        String result = null;
        while (flag) {
            result = blockingQueue.poll(2L, TimeUnit.SECONDS);
            if(null==result||"".equalsIgnoreCase(result)){
                flag=false;
                System.out.println(Thread.currentThread().getName()+"\t"+"超过2m没有取到 消费退出");
                System.out.println();
                System.out.println();
                return;
            }
            System.out.println(Thread.currentThread().getName() + "消费队列" + result + "成功");

        }
    }
    public void stop() throws Exception{
        flag=false;
    }
}

/**
 * volatile/CAS/atomicInteger/BlockQueue/线程交互/原子引用
 **/
public class ProdConsumerBlockQueueDemo {
    public static void main(String[] args) throws Exception {
        MyResource myResource = new MyResource(new ArrayBlockingQueue<>(10));
        new Thread(()->{
            System.out.println(Thread.currentThread().getName()+"\t生产线程启动");
            try {
                myResource.myProd();
            } catch (Exception e) {
                e.printStackTrace();
            }
        },"Prod").start();

        new Thread(()->{
            System.out.println(Thread.currentThread().getName()+"\t消费线程启动");
            try {
                myResource.myConsumer();
            } catch (Exception e) {
                e.printStackTrace();
            }
        },"consumer").start();
        try { TimeUnit.SECONDS.sleep(5); } catch (InterruptedException e) { e.printStackTrace(); }
		

        System.out.println();
        System.out.println("时间到,停止活动");
        myResource.stop();
    }
}
```

- 线程池

 - 消息中间件


 进一步了解阻塞队列可以查看源码。