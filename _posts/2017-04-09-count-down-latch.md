---
layout: post
title: "Introduction of CountDownLatch"
---

>【名词解释】
Latch: 门闩，闸
CountDown：对计数值做减法
CountDownLatch：闭锁

闭锁的作用相当于一扇门：在闭锁达到结束状态前，这扇门是关闭的，所有的线程均不能通过；当到达结束状态时，这扇门会打开允许所有的线程通过。`闭锁可以用来确保某些活动直到其他活动都完成后才继续执行`。例如：
 * 等待指导某个操作的所有参与者（例如，在多玩家游戏中的所有玩家）都就绪后再继续执行。在这种情况下，当所有玩家准备就绪时，闭锁将达到结束状态。
 
CountDownLatch是闭锁的一种实现，它可以使一个或多个线程等待一组事件的发生。
 * 闭锁状态包含一个计数器，初始时计数器为一个正值，表示需要等待的事件的数量。
 * countDown方法递减计数器表示一个事件发生了；await方法等待计数器到达0，这表示所有需要等待的事件都已经发生。
 * 如果计数器值非0，await方法会一直阻塞直到计数器为0，或者等待的线程中断，或等待超时。
 
```java
public class TestCountDownLatch {

    public void test(int nThreads, final Runnable task) throws InterruptedException {

        //启动门
        final CountDownLatch startGate = new CountDownLatch(1);
        //结束门
        final CountDownLatch endGate = new CountDownLatch(nThreads);

        for(int i = 0; i < nThreads; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        //所有线程先等待启动，startGate达到0的状态
                        startGate.await();
                        task.run();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        //逐步释放
                        endGate.countDown();
                    }

                }
            }).start();
        }

        long startTime = System.nanoTime();
        //主线程启动所有线程
        startGate.countDown();

        //主线程等待所有线程执行结束
        endGate.await();

        long endTime = System.nanoTime();

        System.out.println("test method cost: " + (endTime - startTime) + "ms");
    }
}
```

上述程序为什么要startGate，而不是在所有线程创建后立即启动？或许，我们要测试n个线程并发执行某个任务需要的时间，如果在创建线程后立即启动他们，那么先启动的线程“领先”于后启动的线程，并且活跃线程数会随着时间的推移而增加或减少，竞争度也在不断发生变化。启动门是的主线程同时释放所有工作线程。
