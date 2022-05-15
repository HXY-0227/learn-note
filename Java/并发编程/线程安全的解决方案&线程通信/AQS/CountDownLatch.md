# CountDownLatch和CyclicBarrier

## CountDownLatch

### 什么是CountDownLatch

A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.

A `CountDownLatch` is initialized with a given *count*. The [`await`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CountDownLatch.html#await--) methods block until the current count reaches zero due to invocations of the [`countDown()`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CountDownLatch.html#countDown--) method, after which all waiting threads are released and any subsequent invocations of [`await`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CountDownLatch.html#await--) return immediately. This is a one-shot phenomenon -- the count cannot be reset. If you need a version that resets the count, consider using a [`CyclicBarrier`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CyclicBarrier.html).

A `CountDownLatch` is a versatile synchronization tool and can be used for a number of purposes. A `CountDownLatch` initialized with a count of one serves as a simple on/off latch, or gate: all threads invoking [`await`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CountDownLatch.html#await--) wait at the gate until it is opened by a thread invoking [`countDown()`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CountDownLatch.html#countDown--). A `CountDownLatch` initialized to *N* can be used to make one thread wait until *N* threads have completed some action, or some action has been completed N times.

A useful property of a `CountDownLatch` is that it doesn't require that threads calling `countDown` wait for the count to reach zero before proceeding, it simply prevents any thread from proceeding past an [`await`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CountDownLatch.html#await--) until all threads could pass.

**Sample usage:** Here is a pair of classes in which a group of worker threads use two countdown latches:

- The first is a start signal that prevents any worker from proceeding until the driver is ready for them to proceed;
- The second is a completion signal that allows the driver to wait until all workers have completed.

上面这段话来自[Oracle官方解释](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CountDownLatch.html)。



`CountDownLatch`是一种用来做线程同步的辅助工具，他可以实现让一个或者多个线程等待，直到其他线程中的一些操作完成，然后前面等待的线程才被释放的效果。他的主要思想呢就是通过一个给定的计数器去初始化，然后给我们提供了两个核心方法，`await()`和`countDown()`方法。通过调用`await()`方法去阻塞线程，直到由于调用了`countDown()`方法使当前的计数器减为0。然后所有被`await()`的线程都被释放，当然这个过程是一次性的，如果要实现重复计数，可以用 `CyclicBarrier`。基于以上的理解，我们可以得到他的两个基本使用场景

1. 初始化计数器为1，模拟一个简单的开关门闩，让所有的线程通过`await()`被挡在门闩外面，直到有协调的线程调用了`countDown()`方法将计数器减1，瞬间所有挡在门外的线程可以通过门闩。
2. 初始化计数器为N，让一个线程等在门外，等N个线程完成他们的工作才能打开门，让等待的线程通过门闩。

下面我们通过两个例子来具体看一下。

### 应用场景

**例1**：学校开运动会，正在进行百米赛跑的项目。所有运动员都在起跑线等待，只要裁判口哨一吹，运动员就可以马上开始比赛，然后等所有运动员都完成比赛，裁判才能记录分数，宣布比赛结束。这个例子就可以抽象成两个多线程的问题：

2. 有N个线程（运动员）需要等待一个线程（裁判）给一个信号，才可以开始工作
2. 有一个线程（裁判）需要等到其他的N个线程（运动员）都完成他们各自的工作，他才可以开始他的工作

```java
public class Referee {

    public static void main(String[] args) throws Exception {
        // 模拟第一个问题
        CountDownLatch startSignal = new CountDownLatch(1);
        // 模拟第二个问题
        CountDownLatch doneSignal = new CountDownLatch(6);

        // 1.1 我们创建6个线程，并调用start方法，模拟同时有六个运动员在准备比赛
        for (int i = 0; i < 6; i++) {
            new Thread(new Player(startSignal, doneSignal), "player " + i).start();
        }

        // 裁判做一些比赛前的检查，检查OK后吹口哨，比赛开始
        doReadyWork(startSignal);
        // 等所有选手比赛结束，记录分数
        recordScore(doneSignal);

    }

    private static void doReadyWork(CountDownLatch startSignal) {
        try {
            // 裁判检查一切是否就绪
            System.out.println("Ready..");
            Thread.sleep(3000);
            System.out.println("Go...");

            // 1.3 调用countDown()方法打开门闩，模拟检查完成，吹口哨，所有运动员开始
            startSignal.countDown();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private static void recordScore(CountDownLatch doneSignal) {
        try {
            // 2.1 裁判通过await()方法阻塞这，等待那边运动员完成比赛，将doneSignal的计数器减为0，他就开始可以工作了
            doneSignal.await();
            System.out.println("Record the score...");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

public class Player implements Runnable{
    private final CountDownLatch startSignal;
    private final CountDownLatch doneSignal;

    public Player(CountDownLatch startSignal, CountDownLatch doneSignal) {
        this.startSignal = startSignal;
        this.doneSignal = doneSignal;
    }

    @Override
    public void run() {
        try {
            // 1.2 调用await()方法让每个线程阻塞，模拟所有运动员等着那边裁判吹口哨，等到吹了口哨就比赛开始
            startSignal.await();
            beginPlay();
            // 2.2 doneSignal的计数器为6，每有一个运动员结束比赛，计数器减1，等到减为0，表示所有人都完成了比赛，
            // 那边通过doneSignal阻塞的线程也可以开始做他的事情了
            doneSignal.countDown();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }

    public void beginPlay() {
        try {
            System.out.println(Thread.currentThread().getName() + " begin...");
            Thread.sleep(5000);
            System.out.println(Thread.currentThread().getName() + " complete...");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

执行结果：
Ready..
Go...
player 1 begin...
player 0 begin...
player 2 begin...
player 4 begin...
player 3 begin...
player 5 begin...
player 2 complete...
player 0 complete...
player 1 complete...
player 4 complete...
player 5 complete...
player 3 complete...
Record the score...
```

### 源码分析

**初始化方法**

```java
// 初始化方法，和其他基于AQS的工具一样，写了一个内部类Sync继承AQS，将计数器的值设置给AQS的state
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

Sync(int count) {
    setState(count);
}
```

**await()方法**

```java
// await()方法，这里还是调用父类模板方法
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 这里定义获取共享锁的模板流程，实际逻辑交给子类去实现，就会走到CountDownLatch的tryAcquireShared(arg)方法中
    // 参考下面的方法，如果tryAcquireShared(arg) > 0,说明计数器已经被其他线程减为0，程序正常执行
    // 如果tryAcquireShared(arg) < 0,说明计数器还没有被减到0，调用await()方法的线程就需要阻塞自己。
    if (tryAcquireShared(arg) < 0) 
        doAcquireSharedInterruptibly(arg);
}

protected int tryAcquireShared(int acquires) {
    // 判断state也就是计数器是否等于0
    return (getState() == 0) ? 1 : -1;
}
```

**countDown()方法**

```java
// countDown()方法，依旧是那个套路，调用AQS的releaseShared(1)方法
public void countDown() {
    sync.releaseShared(1);
}

public final boolean releaseShared(int arg) {
    // 老套路，调用子类重写的释放共享锁的方法
    if (tryReleaseShared(arg)) {
        doReleaseShared();   // 标记2
        return true;
    }
    return false;
}

protected boolean tryReleaseShared(int releases) {
    // 通过自旋的方法保证CAS修改值得操作能成功执行
    for (;;) {
        int c = getState();
        // 如果已经等于0，直接返回
        if (c == 0)
            return false;
        int nextc = c-1;
        // CAS修改计数器的值，返回计数器是否等于0，如果这里得到true，说明当前这个线程将计数器减到0了，就需要进入标记2的代码
        // 释放还在被await()阻塞的线程
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

## CycliBarrier

