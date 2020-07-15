# CountDownLatch、CyclicBarrier和Semaphore用法

## CountDownLatch 

让一些线程阻塞直到另外一些完成后才被唤醒

CountDownLatch主要有两个方法,当一个或多个线程调用`await`方法时,调用线程会被阻塞.其他线程调用`countDown`方法计数器减1(调用`countDown`方法时线程不会阻塞),当计数器的值变为0,因调用`await`方法被阻塞的线程会被唤醒,继续执行;

Demo1 关门案例：
```java
public class CountDownLatchDemo {
    public static void main(String[] args) throws Exception {
        CountDownLatch countDownLatch = new CountDownLatch(6);
        for (int i = 1; i <= 6; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "\t" + "上完自习");
                countDownLatch.countDown();
            }, String.valueOf(i)).start();
        }
        countDownLatch.await();
        System.out.println(Thread.currentThread().getName() + "\t班长锁门离开教室");
    }
} 
```

Demo2 秦灭六国：

枚举的使用,枚举可以当作一个小型的内存数据库中的一张表；
```java
public enum CountryEnum {
    /**
     *
     */
    ONE(1, "齐"),
    /**
     *
     */
    TWO(2, "楚"),
    /**
     *
     */
    THREE(3, "燕"),
    /**
     *
     */
    FOUR(4, "赵"),
    /**
     *
     */
    FIVE(5, "魏"),
    /**
     *
     */
    SIX(6, "韩");

    CountryEnum(Integer code, String name) {
        this.code = code;
        this.name = name;
    }

    @Getter
    private Integer code;
    @Getter
    private String name;

    public static CountryEnum forEach(int index) {
        CountryEnum[] countryEnums = CountryEnum.values();
        for (CountryEnum countryEnum : countryEnums) {
            if (index == countryEnum.getCode()) {
                return countryEnum;
            }
        }
        return null;
    }
}
```

```java
public class CountDownLatchDemo {
    public static void main(String[] args) throws Exception {
        CountDownLatch countDownLatch = new CountDownLatch(6);
        for (int i = 1; i <= 6; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "\t" + "国,灭亡");
                countDownLatch.countDown();
            }, CountryEnum.forEach(i).getName()).start();
        }
        countDownLatch.await();
        System.out.println("秦统一");

    }
}
```

## CyclicBarrier

`CyclicBarrier` 让一组线程到达一个屏障(也可以叫做同步点)时被阻塞,直到最后一个线程到达屏障时,屏障才会开门,所有被屏障拦截的线程才会继续干活,线程进入屏障通过`CyclicBarrier`的`await()`方法.

```java
public class CyclicBarrierDemo {
    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier=new CyclicBarrier(7,()->{
            System.out.println("召唤神龙");
        });

        for (int i = 1; i <=7; i++) {
            final int temp = i;
            new Thread(()->{
             System.out.println(Thread.currentThread().getName()+"\t 收集到第"+ temp +"颗龙珠");
                try {
                    cyclicBarrier.await();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            },String.valueOf(i)).start();
        }
    }
}
```

## SemaPhore 

信号量的主要用户两个目的,一个是用于多个共享资源的相互排斥使用,另一个用于并发资源数的控制.

```java
/**
 * Description
 *
 * @author veliger@163.com
 * @version 1.0
 * @date 2019-04-13 11:08
 **/
public class SemaphoreDemo {
    public static void main(String[] args) {
        //模拟3个停车位
        Semaphore semaphore = new Semaphore(3);
        //模拟6部汽车
        for (int i = 1; i <= 6; i++) {
            new Thread(() -> {
                try {
                    //抢到资源
                    semaphore.acquire();
                    System.out.println(Thread.currentThread().getName() + "\t抢到车位");
					
                    try {TimeUnit.SECONDS.sleep(3);} catch (InterruptedException e) {e.printStackTrace();}
					
                    System.out.println(Thread.currentThread().getName() + "\t 停3秒离开车位");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    //释放资源
                    semaphore.release();
                }
            }, String.valueOf(i)).start();
        }
    }
}
```
