### CountDownLatch

CountDownLatch适用于在多线程的场景需要等待所有子线程全部执行完毕之后再做操作的场景。

举个例子，早上部门开会，有人在上厕所，这时候需要等待所有人从厕所回来之后才能开始会议。

```java
public class CountDownLatchTest {
    private static int num = 3;
    private static CountDownLatch countDownLatch = new CountDownLatch(num);
    private static ExecutorService executorService = Executors.newFixedThreadPool(num);
    public static void main(String[] args) throws Exception{
        executorService.submit(() -> {
            System.out.println("A在上厕所");
            try {
                Thread.sleep(4000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }finally {
                countDownLatch.countDown();
                System.out.println("A上完了");
            }
        });
        executorService.submit(()->{
            System.out.println("B在上厕所");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }finally {
                countDownLatch.countDown();
                System.out.println("B上完了");
            }
        });
        executorService.submit(()->{
            System.out.println("C在上厕所");
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }finally {
                countDownLatch.countDown();
                System.out.println("C上完了");
            }
        });

        System.out.println("等待所有人从厕所回来开会...");
        countDownLatch.await();
        System.out.println("所有人都好了，开始开会...");
        executorService.shutdown();

    }
}
```

代码执行结果：

```
A在上厕所
B在上厕所
等待所有人从厕所回来开会...
C在上厕所
B上完了
C上完了
A上完了
所有人都好了，开始开会...
```

初始化一个CountDownLatch实例传参3，因为我们有3个子线程，每次子线程执行完毕之后调用countDown()方法给计数器-1，主线程调用await()方法后会被阻塞，直到最后计数器变为0，await()方法返回，执行完毕。他和join()方法的区别就是join会阻塞子线程直到运行结束，而CountDownLatch可以在任何时候让await()返回，而且用ExecutorService没法用join了，相比起来，CountDownLatch更灵活。

CountDownLatch基于AQS实现，volatile变量state维持倒数状态，多线程共享变量可见。

1. CountDownLatch通过构造函数初始化传入参数实际为AQS的state变量赋值，维持计数器倒数状态
2. 当主线程调用await()方法时，当前线程会被阻塞，当state不为0时进入AQS阻塞队列等待。
3. 其他线程调用countDown()时，state值原子性递减，当state值为0的时候，唤醒所有调用await()方法阻塞的线程



### CyclicBarrier

CyclicBarrier叫做回环屏障，它的作用是**让一组线程全部达到一个状态之后再全部同时执行**，而且他有一个特点就是所有线程执行完毕之后是可以重用的。

```java
public class CyclicBarrierTest {
    private static int num = 3;
    private static CyclicBarrier cyclicBarrier = new CyclicBarrier(num, () -> {
        System.out.println("所有人都好了，开始开会...");
        System.out.println("-------------------");
    });
    private static ExecutorService executorService = Executors.newFixedThreadPool(num);
    public static void main(String[] args) throws Exception{
        executorService.submit(() -> {
            System.out.println("A在上厕所");
            try {
                Thread.sleep(4000);
                System.out.println("A上完了");
                cyclicBarrier.await();
                System.out.println("会议结束，A退出");
            } catch (Exception e) {
                e.printStackTrace();
            }finally {

            }
        });
        executorService.submit(()->{
            System.out.println("B在上厕所");
            try {
                Thread.sleep(2000);
                System.out.println("B上完了");
                cyclicBarrier.await();
                System.out.println("会议结束，B退出");
            } catch (Exception e) {
                e.printStackTrace();
            }finally {

            }
        });
        executorService.submit(()->{
            System.out.println("C在上厕所");
            try {
                Thread.sleep(3000);
                System.out.println("C上完了");
                cyclicBarrier.await();
                System.out.println("会议结束，C退出");
            } catch (Exception e) {
                e.printStackTrace();
            }finally {

            }
        });

        executorService.shutdown();

    }
}
```

输出结果为：

```java
A在上厕所
B在上厕所
C在上厕所
B上完了
C上完了
A上完了
所有人都好了，开始开会...
-------------------
会议结束，A退出
会议结束，B退出
会议结束，C退出
```

从结果来看和CountDownLatch非常相似，初始化传入3个线程和一个任务，线程调用await()之后进入阻塞，计数器-1，当计数器为0时，就去执行CyclicBarrier中构造函数的任务，当任务执行完毕后，唤醒所有阻塞中的线程。这验证了CyclicBarrier**让一组线程全部达到一个状态之后再全部同时执行**的效果。

再举个例子来验证CyclicBarrier可重用的效果。

```java
public class CyclicBarrierTest2 {
    private static int num = 3;
    private static CyclicBarrier cyclicBarrier = new CyclicBarrier(num, () -> {
        System.out.println("-------------------");
    });
    private static ExecutorService executorService = Executors.newFixedThreadPool(num);

    public static void main(String[] args) throws Exception {
        executorService.submit(() -> {
            System.out.println("A在上厕所");
            try {
                Thread.sleep(4000);
                System.out.println("A上完了");
                cyclicBarrier.await();
                System.out.println("会议结束，A退出，开始撸代码");
                cyclicBarrier.await();
                System.out.println("C工作结束，下班回家");
                cyclicBarrier.await();
            } catch (Exception e) {
                e.printStackTrace();
            } finally {

            }
        });
        executorService.submit(() -> {
            System.out.println("B在上厕所");
            try {
                Thread.sleep(2000);
                System.out.println("B上完了");
                cyclicBarrier.await();
                System.out.println("会议结束，B退出，开始摸鱼");
                cyclicBarrier.await();
                System.out.println("B摸鱼结束，下班回家");
                cyclicBarrier.await();
            } catch (Exception e) {
                e.printStackTrace();
            } finally {

            }
        });
        executorService.submit(() -> {
            System.out.println("C在上厕所");
            try {
                Thread.sleep(3000);
                System.out.println("C上完了");
                cyclicBarrier.await();
                System.out.println("会议结束，C退出，开始摸鱼");
                cyclicBarrier.await();
                System.out.println("C摸鱼结束，下班回家");
                cyclicBarrier.await();
            } catch (Exception e) {
                e.printStackTrace();
            } finally {

            }
        });

        executorService.shutdown();

    }
}
```

输出结果：

```java
A在上厕所
B在上厕所
C在上厕所
B上完了
C上完了
A上完了
-------------------
会议结束，A退出，开始撸代码
会议结束，B退出，开始摸鱼
会议结束，C退出，开始摸鱼
-------------------
C摸鱼结束，下班回家
C工作结束，下班回家
B摸鱼结束，下班回家
-------------------
```

从结果来看，每个子线程调用await()计数器减为0之后才开始继续一起往下执行，会议结束之后一起进入摸鱼状态，最后一天结束一起下班，这就是**可重用**。

CyclicBarrier还是基于AQS实现的，内部维护parties记录总线程数，count用于计数，最开始count=parties，调用await()之后count原子递减，当count为0之后，再次将parties赋值给count，这就是复用的原理。

1. 当子线程调用await()方法时，获取独占锁，同时对count递减，进入阻塞队列，然后释放锁
2. 当第一个线程被阻塞同时释放锁之后，其他子线程竞争获取锁，操作同1
3. 直到最后count为0，执行CyclicBarrier构造函数中的任务，执行完毕之后子线程继续向下执行



### Semaphore

Semaphore叫做信号量，和前面两个不同的是，他的计数器是递增的。

```java
public class SemaphoreTest {
    private static int num = 3;
    private static int initNum = 0;
    private static Semaphore semaphore = new Semaphore(initNum);
    private static ExecutorService executorService = Executors.newFixedThreadPool(num);
    public static void main(String[] args) throws Exception{
        executorService.submit(() -> {
            System.out.println("A在上厕所");
            try {
                Thread.sleep(4000);
                semaphore.release();
                System.out.println("A上完了");
            } catch (Exception e) {
                e.printStackTrace();
            }finally {

            }
        });
        executorService.submit(()->{
            System.out.println("B在上厕所");
            try {
                Thread.sleep(2000);
                semaphore.release();
                System.out.println("B上完了");
            } catch (Exception e) {
                e.printStackTrace();
            }finally {

            }
        });
        executorService.submit(()->{
            System.out.println("C在上厕所");
            try {
                Thread.sleep(3000);
                semaphore.release();
                System.out.println("C上完了");
            } catch (Exception e) {
                e.printStackTrace();
            }finally {

            }
        });

        System.out.println("等待所有人从厕所回来开会...");
        semaphore.acquire(num);
        System.out.println("所有人都好了，开始开会...");

        executorService.shutdown();

    }
}
```

输出结果为：

```java
A在上厕所
B在上厕所
等待所有人从厕所回来开会...
C在上厕所
B上完了
C上完了
A上完了
所有人都好了，开始开会...
```

稍微和前两个有点区别，构造函数传入的初始值为0，当子线程调用release()方法时，计数器递增，主线程acquire()传参为3则说明主线程一直阻塞，直到计数器为3才会返回。

Semaphore还还还是基于AQS实现的，同时获取信号量有公平和非公平两种策略

1. 主线程调用acquire()方法时，用当前信号量值-需要获取的值，如果小于0，则进入同步阻塞队列，大于0则通过CAS设置当前信号量为剩余值，同时返回剩余值
2. 子线程调用release()给当前信号量值计数器+1(增加的值数量由传参决定)，同时不停的尝试因为调用acquire()进入阻塞的线程



### 总结

CountDownLatch通过计数器提供了比join更灵活的多线程控制方式，CyclicBarrier也可以达到CountDownLatch的效果，而且有可复用的特点，Semaphore则是采用信号量递增的方式，开始的时候并不需要关注需要同步的线程个数，并且提供获取信号的公平和非公平策略。