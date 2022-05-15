# 线程通信之wait&notify

在前面几节我们说过，java中每个线程在工作的时候，都会为其在栈区分配私有的线程栈，将共享变量拷贝到私有线程栈的工作内存中，进行处理。当线程运行完业务逻辑以后，私有的线程栈会被销毁。因此多个线程之间是无法感知相互的状态，无法进行通信的。但是有时候我们又确实需要让多个线程之间进行通信，比如最经典的**生产者消费者模型**。

## 什么是生产者消费者模型

生产者消费者模型是我们生活中很常见的一种模型，比如中午排队买饭，后厨的师傅会将做好的各种菜放在盘子里，我们排队打饭，然后结账，那么在这个过程中，后厨师傅就是生产者，我们就是消费者；再比如快递小哥会将一个个快递放到菜鸟驿站，我们从菜鸟驿站取走属于我们的快递，那么快递小哥就是生产者，我们就是消费者。在第一个模型中，后厨和顾客通过盘子传递饭菜，第二个模型中，快递小哥和顾客通过菜鸟驿站传递快件，这里的盘子和菜鸟驿站的作用都是用来保存生产者生产的东西。

在计算机的世界里我们通常会将盘子和菜鸟驿站抽象成一个**队列**。生产者只负责生产数据，然后将他们保存到一个队列中，然后消费者负责从队列中消费数据。比如我们在说线程的时候提到的阻塞队列，当有大量任务提交到线程池中的时候，如果核心线程处理不过来，线程池则会将这些任务缓存到阻塞队列，这个时候提交任务的程序就是任务的生产者，等到核心线程空闲下来，会从阻塞队列中取出任务进行处理，那么核心线程就是消费者。在比如我们常见的消息队列，也是生产者消费者模型的典型应用，比如下图就是从RabbitMQ官网下载的一张图。他很好的诠释了RabbitMQ的基本模型，那就是消息的生产者负责生产消息，然后将消息投递到一个队列中，然后消息的消费者负责从队列中取出消息，进行处理。

![](./img/生产者消费者模型.png)

## 通过notify和wait实现简单的生产者消费者模型

理解了生产者消费者模型，那我们一起来实现这样一个需求吧：消费者从一个队列中取数据，如果队列不为空，则获取数据，如果队列为空，则等待生产者把数据放入到队列中，在重新获取。（简单的阻塞队列）

分析一下这个需求，首先我们需要一个队列，我们可以用LinkedList来模拟，涉及多线程操作的线程安全问题，我们可以用之前学的synchroized关键字解决，可是如何实现当队列为空的时候，让消费者等待，当有生产者往其中放入数据的时候，在通知到消费者让其去消费呢？这就用到了我们今天所要说的关于线程通信的手段：`wait()`和`notify()`方法。

我们说了两个线程会独立加载自己的线程栈，他们拥有自己的线程空间，井水不犯河水。因此要让他们通信，就得找一个能和多个线程都说上话的**中间人**，谁能胜任这份工作呢，那就是他们都要争抢的那一把锁了。当某一个线程T去获取锁的时候，如果需要等待，让锁调用`wait()`方法告诉T，T接到命令，则去锁给他提供的地方进行等待（等待集）。当T不用等待，可以去工作的时候，锁在通过其他线程把等待集中的T唤醒，让它准备开始工作。

### wait(long timeout)

`wait(long timeout)`是Object类提供的一个方法，他会让当前线程（线程T）将自己放到一个等待集合中（当前线程必须拥有这个对象的监视器），然后放弃在当前对象上获取到的所有同步锁，从此以后，CPU将不会在调度到线程T，除非发生以下几件事情：

- 其他线程调用了这个对象的`notify()`方法，而线程T正好是被唤醒的那个线程
- 其他线程调用了这个对象的`notifyAll()`方法
- 其他线程中断了线程T
- 调用`wait(long timeout)`的时候传入了一个超时时间，等这个时间过去之后

当上面几件事发生后，线程重新变为Runnable状态，可以等待被CPU重新调度。一旦当线程T被CPU调度到，重新获得对象的控制权，那么他就会回复到之前的状态，继续执行后续的业务代码。线程也有可能在没有被通知、中断、超时的情况下被唤醒，这种情况我们称之为**虚假唤醒**，为了避免这种情况，我们应该让`wait()`方法总是在循环中被调用。（后续会解释何谓虚假唤醒）

### notify() & notifyAll()

`notify()`也是Object类提供的一个方法，用来唤醒该对象监视器的单个线程，如果有多个线程在等待，那么选择谁被唤醒，这是一个随机的操作。

`notifyAll()`也是Object类提供的一个方法，用来唤醒该对象监视器的所有线程。

了解了`wait()`和`notify()`方法后，我们就可以实现我们的需求了：

```java
private final List<String> syncList;

public Main() {
    syncList = new LinkedList<>();
}

// 删除列表中的元素
public void removeElement() {
    synchronized (syncList) {
        try {
            // 列表为空就等待
            while (syncList.isEmpty()) {
                System.out.println("List is empty, wait add element...");
                syncList.wait();
            }
            String remove = syncList.remove(0);
            if (null != remove) {
                System.out.println("Thread " + Thread.currentThread().getName() + " remove element " + remove + " success...");
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

// 添加元素到列表
public void addElement(String element) {
    synchronized (syncList) {
        // 添加一个元素，并通知元素已存在
        syncList.add(element);
        System.out.println("Add Element: " + element + " success...");

        syncList.notifyAll();
        System.out.println("notifyAll called!");
    }
}

public static void main(String[] args) {
    Main main = new Main();

    new Thread(main::removeElement).start();
    new Thread(main::removeElement).start();
    new Thread(main::removeElement).start();

    new Thread(() -> main.addElement("element1")).start();
    new Thread(() -> main.addElement("element2")).start();

}

// 运行结果
List is empty, wait add element...
List is empty, wait add element...
List is empty, wait add element...
Add Element: element1 success...
notifyAll called!
Thread Thread-2 remove element element1 success...
Add Element: element2 success...
notifyAll called!
Thread Thread-1 remove element element2 success...
List is empty, wait add element...
```

## 虚假等待

如果我们将上述代码该成下面这样，看看执行结果是否正常：

```java
// 删除列表中的元素
public void removeElement() {
    if (syncList) {
        try {
            // 列表为空就等待
            if (syncList.isEmpty()) {
                System.out.println("List is empty, wait add element...");
                syncList.wait();
            }
            String remove = syncList.remove(0);
            if (null != remove) {
                System.out.println("Thread " + Thread.currentThread().getName() + " remove element " + remove + " success...");
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

public static void main(String[] args) {
    Main main = new Main();

    new Thread(main::removeElement, "C_A").start();
    new Thread(main::removeElement, "C_b").start();
    new Thread(main::removeElement, "C_C").start();

    new Thread(() -> main.addElement("element1"), "P_A").start();
    new Thread(() -> main.addElement("element2"), "P_B").start();

}

// 执行结果
List is empty, wait add element...
List is empty, wait add element...
List is empty, wait add element...
Add Element: element1 success...
notifyAll called!
Add Element: element2 success...
notifyAll called!
Thread C_C remove element element1 success...
Thread C_A remove element element2 success...
java.lang.IndexOutOfBoundsException: Index: 0, Size: 0
```

从逻辑上说，`if (syncList)`是符合逻辑的，如果队列为空，则当前线程等待，但是为什么会发生越界的异常呢？我们看日志一起分析下：

1. 线程：C_A、C_B、C_C从执行从队列中取数据，发现队列为空，都执行了`wait()`方法，释放了锁，进入等待集合，阻塞在第8行
2. 线程：P_A、P_B执行往队列中放数据，然后执行了`notifyAll()`方法，C_A、C_B、C_C都被唤醒，C_A，C_C接着从第9行执行，执行了第10行代码，成功从队列中取出数据
3. 线程C_B也被唤醒了，也接着从第9行执行，执行到第10行代码，结果队列已经空了，故而抛出了异常。因此当一个线程被唤醒后，应该在重新判断一次之前的条件是否还成立，如果成立才执行业务逻辑，如果不成立，则继续阻塞，因此`if`应该改成`while`。

在来看一个例子：

```java
private int number = 0;

public synchronized void increment() throws InterruptedException {
    if (number != 0) {
        this.wait();
    }
    number++;
    System.out.println(Thread.currentThread().getName() + "生产了数据:" + number);
    this.notify();
}

public synchronized void decrement() throws InterruptedException {
    if (number == 0) {
        this.wait();
    }
    number--;
    System.out.println(Thread.currentThread().getName() + "消费了数据:" + number);
    this.notify();
}

public static void main(String[] args) {
    Main1 main = new Main1();
    new Thread(() -> {
        for (int i = 0;i < 2; i++) {
            try {
                main.increment();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }, "P_A").start();

    new Thread(() -> {
        for (int i = 0;i < 2; i++) {
            try {
                main.increment();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }, "P_B").start();


    new Thread(() -> {
        for (int i = 0;i < 2; i++) {
            try {
                main.decrement();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }, "C_A").start();

    new Thread(() -> {
        for (int i = 0;i < 2; i++) {
            try {
                main.decrement();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }, "C_B").start();
}

// 执行结果
P_A生产了数据:1
C_A消费了数据:0
P_A生产了数据:1
P_B生产了数据:2
C_A消费了数据:1
P_B生产了数据:2
C_B消费了数据:1
C_B消费了数据:0
```

上面这段代码，我们希望的是多个生产者和消费者依次执行，对number做加1和减1的操作。但是实际和预期结果不一致，出现了下面的结果，我们一起分析下看看为什么：

```
P_A生产了数据:1 
C_A消费了数据:0
P_A生产了数据:1
P_B生产了数据:2
```

1. 26行第1次循环，P_A调用`increment()`方法，if条件不满足，执行第7行代码，number 变为 1，然后执行`notify()`方法唤醒等待集中的线程，此时没有线程在等待集中，
2. 26行第2次循环，P_A调用`increment()`方法，if条件满足，执行第5行代码，P_A被放到等待集中
3. 34行第1次循环，P_B调用`increment()`方法，if条件满足，执行第5行代码，P_B被放到等待集中
4. 45行第1次循环，C_A调用`decrement()`方法，if条件不满足，执行弟16行代码，number 变为 0，然后执行`notify()`方法唤醒等待集中的线程，此时P_A被唤醒，接着从上次被阻塞的第7行开始执行，number 变为 1，然后`notify()`方法被执行，唤醒等待集中的线程，P_B被唤醒
5. P_B接着从上次被阻塞的第7行开始执行，number 变为 2

我们把上面这种情况叫做虚假唤醒，解决方法就是将`if`换成`while`表达式，当一个在等待集中的线程被唤醒后，要让他重新判断一下之前符合的条件是否还符合。

