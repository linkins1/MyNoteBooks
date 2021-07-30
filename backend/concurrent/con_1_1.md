# 1.基础篇

### 1.1 并发编程基础

#### 1.1.1线程

##### 1）定义

线程是进程中的一个实体，是进程的一个执行路径，一个进程中至少有一个线程

进程是资源分配和调度的单位，线程是CPU分配的基本单位

##### 2）特性

- 线程有自己的程序计数器和栈（存储局部变量表），且共享进程中的方法区和堆
- 多个线程一般以时间片轮转的方式上CPU执行
- 启动main函数就创建一个JVM进程，main是主线程

> 如果执行native方法，pc记录为undefined

#### 1.1.2线程的创建

> 线程创建有三种方式，具体的例子参见最后[附]()

##### 1）继承Thread

继承Thread类后，重写其中的`run()`方法，之后创建重写类对象即可

###### （1）`public Thread()`

创建子类对象时，会调用`Thread`的空参构造器

```java
public Thread() {
    init(null, null, "Thread-" + nextThreadNum(), 0);
}
```

init中调用其重载方法

```java
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize) {
    init(g, target, name, stackSize, null, true);
}
```

最终调用下面的init重载方法

```java
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc,
                  boolean inheritThreadLocals) {
    /**
    */
    this.name = name;
    /**
    */
}
```

这里会将此Thread对象的name设定为`"Thread-" + nextThreadNum()`

###### （2）`public synchronized void start()`

```java
public synchronized void start() {
     /*
     * A zero status value corresponds to state "NEW".
     */
    if (threadStatus != 0)
        throw new IllegalThreadStateException();

    boolean started = false;
    try {
        start0();
        started = true;
    } finally {
            /**
            */
        }
    }
}
```

此方法的注释如下

>Causes this thread to **begin execution;** the **Java Virtual Machine** calls the **run method of this thread**.
>The result is that **two threads are running concurrently**: the **current thread** (which returns from the call to the start method) and the **other thread** (which executes its run method).
>It is **never legal** to start a thread **more than once**. In particular, a thread may not be restarted once it has completed execution.

从上面可以看出，当调用此方法后，**JVM**会创建一个线程来执行**`子类中重写`的`run`方法**

##### 2）实现Runnable接口

创建类实现Runnable接口，并重写其中的run方法，之后创建重写类的对象，作为Thread构造器的参数传入并创建对象即可

###### （1）`public Thread(Runnable target)`

首先将重写的对象传入，会调用下面的构造函数，其中调用init

```java
public Thread(Runnable target) {
    init(null, target, "Thread-" + nextThreadNum(), 0);
}
```

这里和使用到的构造器链和继承方式的一致，但是要注意，在最后一个init方法中，存在下面代码

```java
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc,
                  boolean inheritThreadLocals) {
    this.target = target;
}
```

这里看到，会将重写类对象传递给`Thread`类的成员变量`target`

###### （2）`public void run()`

```java
@Override
public void run() {
    if (target != null) {
        target.run();
    }
}
```

此方法是`Thread`类中重写Runnable接口后重写的方法，由于此处没有采用继承的方式，所以在调用`start`方法后，使用的是该run方法，此方法会调用传入的target对象的`run`方法，也就是我们重写类中重写的run方法

类似的，JVM也会创建一个线程来执行**`Thread类`中的`run方法`**

##### 3）实现`Callable`接口

由于run方法返回值为void，为了能够在线程执行完毕后具有返回值，但又不改变现有的`Runnable`接口和`Thread`结构，首先定义了下面一个接口

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/image-20201105144143669.png" alt="image-20201105144143669" style="zoom:80%;" />

其中给出了call方法，其会返回一个泛型值，通过调用call方法即可实现具有返回值的线程，**但是**，JVM中只会调用Thread类的run方法来作为线程的执行入口，为了适应这一点，又定义了下面三个接口和一个类

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/image-20201105142804535.png" alt="image-20201105142804535" style="zoom:80%;" />

首先提供了`Future`接口，其中定义了`get`方法用于获取线程执行完成后的返回值

由于调用Thread构造器传入的参数必须实现`Runnable接口`，所以定义了`RunnableFuture`接口同时继承了`Runnable`和`Future`，其常用实现类为`FutureTask`，其中重写的`run`方法如下

```java
public void run() {
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        /**
        */
    }
}
```

这里就会用到`Callable对象`，来执行其`call`方法返回值为result，这里需要注意，FutureTask中的set方法会在run方法执行成功时设定`ran=true`，并调用set方法将返回值设定给成员变量`outcome`，之后调用get方法获取返回值时，也是返回这个`outcome`

**综上所述，**

如果想创建一个具有返回值的线程，需要如下几步

- 创建类`C`实现`Callable`接口，并创建此类`对象c1`
- 创建`FutureTask`对象`f1`，将`c1`作为构造器参数传入，保证其中的`callable成员变量为c1`
- 由于`FutureTask`实现了`RunnableFuture`接口，所以将`f1`作为Thread类的`构造器参数`传入，创建`t1`对象

至此线程对象创建完毕，当想要获取返回值时，通过`f1对象的get方法`即可获取

#### 1.1.3线程同步方法

##### 1）wait

###### （1）定义

通过调用共享变量的wait方法，调用**线程A**会被阻塞挂起，直到

- 其他线程调用**该共享变量的**notify或notifyAll方法
- 其他线程调用**线程A**的中断方法，抛出InterruptedException异常后返回

###### （2）重载

- `wait(long timeout)`

  当timeout倒计时耗尽时，如果没有收到notify，则不再阻塞，执行wait后的代码

- `wait(long timeout, int nanos)`

  如果nanos>0且<999999，那么在每次timeout时间耗尽时，将timeout++后再次调用wait函数

###### （3）虚假唤醒

在阻塞队列中的线程，即使没有收到notify或者中断，有可能会被唤醒进入就绪队列，为了避免这一点，可以添加等待条件，并利用while循环检查，如果不满足，则再次调用wait函数进入阻塞队列

##### 2）notify

###### （1）定义

通过调用共享变量的notify方法可以从阻塞队列中随机唤醒一个线程进入就绪队列，与就绪队列中的其他线程共同竞争锁

在没有获得对应共享变量的锁的时候，如果调用其notify方法，则会抛出IllegalMonitorStateException异常

###### （2）重载

notifyAll函数是唤醒阻塞队列上的所有线程进入就绪队列，这样可以避免线程饥饿的问题

> 不论是notify还是notifyAll都只会唤醒在调用此方法之前，阻塞队列中的线程

##### 3）join

假设目前启动线程A，在线程B中通过调用线程A的join方法即B从调用点起开始等待A执行完成，同时线程B会进入阻塞队列。由于在A完成之前，B会一直被阻塞，如果此时有线程C调用了B的中断方法，则B会抛出InterruptedException

##### 4）sleep

此方法是Thread类的静态方法，在对应线程中调用此方法会使此线程**进入阻塞队列**，**让出CPU**但**不会**让出休眠前获得的所有锁，如果在休眠期间被其他线程中断，则会抛出InterruptedException，如果正常休眠结束，则会进入就绪队列，再次参与CPU调度

##### 5）yield

同样是一个静态方法，在对应线程中调用此方法即暗示调度器想要让出CPU资源，但是调度器完全可以忽略此暗示。

yield方法在执行过后，**仍然保持在就绪队列**，其只是让出了当前被分配到的**时间片的剩余部分**，很有可能**被再次调度**

##### 6）interrupt

###### （1）定义

如果线程B

- 正在运行

  在线程A中通过调用线程B的interrupt方法可以设定B的**中断标志位为true**，B不会被立即中断，会继续执行后续部分后退出

- 调用过wait/join/sleep

  在线程A中通过调用线程B的interrupt方法会导致**B立即抛出InterruptedException**

###### （2）相关方法

- isInterrupted

  是实例方法，用于判断当前线程的中断状态，不会清除中断标志位

- interrupted

  是静态方法，用于判断当前线程的中断状态，并清除终端标志位（置为false）

#### 1.1.4线程上下文切换

线程上下文是线程执行过程中所需要的环境、局部变量、寄存器状态等，在进行线程切换时，需要记录线程上下文以便于下次恢复断点状态。

线程上下文切换时机有

- 当前线程的CPU时间片耗尽且线程未执行完
- 当前线程被其他线程中断时

#### 1.1.5 线程状态转换

Java中给出了线程的六种状态，存放在枚举类`Thead.State`中，如下所示

```java
public enum State {

    NEW,

    RUNNABLE,

    BLOCKED,

    WAITING,

    TIMED_WAITING,

    TERMINATED;
}
```

##### 1）NEW

代表刚刚被创建但是没有被调度入内存中，在`Thread t1 = new Thread`之后，在调用start之前，`t1`的状态是`NEW`

##### 2）RUNNABLE

代表线程处于运行状态，在调用start方法之后，如果

- 线程在JVM中正常运行，从操作系统角度来讲，即为处于运行态
- 线程正在等待操作系统的资源，从操作系统角度来讲，即为处于就绪队列

则处于`RUNNABLE`，可见其包含了两种状态=运行态+就绪态

##### 3）BLOCKED

代表线程正在等待进入synchronized代码块/方法，有如下两种情况

- 初次进入synchronized时，监视器锁已经被占用
- 调用wait方法后，被其他线程唤醒后，再次进入synchronized时，监视器锁被占用

##### 4）WAITING

代表线程处于等待状态，有如下触发条件

- 调用`wait()`
- 调用`join()`，底层调用的是`wait`方法
- **初次**（未获得许可证）调用`LockSupport.park()`

如果线程想要被唤醒，其他线程应对应的执行如下方法

- 调用`notify()`/`notifyAll`
- 被join的线程退出
- 调用`unpark(t)`方法，`t`为执行park方法的线程

##### 5）TIMED_WAITING

代表线程处于计时等待状态，触发条件为触发WAITING状态的方法的定时版本，定时结束后自动被唤醒，参与调度

##### 6）TERMINATED

代表线程处于终止状态，已经退出

**综上所述**，状态转移图如下所示

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/image-20201117162542068.png" alt="image-20201117162542068" style="zoom:80%;" />

#### 1.1.6线程死锁

##### 1）死锁产生的条件

- 互斥条件

  指共享资源同时只能被一个线程使用

- 请求并持有条件

  指一个线程已经占有一个资源，但是又提出对另一个资源的请求

- 排他性条件

  指线程已经占有的资源不能被其他线程剥夺

- 环路等待条件

  指多个线程循环等待其他线程释放资源，构成闭环

##### 2）避免死锁

一般通过破坏请求并持有条件和环路等待条件来解决死锁

可以通过让各个线程有序的申请共享资源来破坏环路等待条件，这样可以避免线程死锁

#### 1.1.7守护线程（Daemon）与用户线程

##### 1）区别

- 守护线程

  运行于后台，其是否结束不影响JVM进程的退出

- 用户线程

  用户可感知，其是否结束影响JVM进程的退出

##### 2）设定

通过调用线程的`setDaemon(true)`来设定为守护线程

> GC线程是常见的守护线程

### 附

#### #1.线程创建

- 继承

```java
public class CreateByInherit extends Thread{

    @Override
    public void run() {
        System.out.println(getName());
    }

    public static class MyThread extends Thread{
        @Override
        public void run() {
            while(true){
                System.out.println(getName());
            }
        }
    }

    public static void main(String[] args) {
//        CreateByInherit t1 = new CreateByInherit();
//        CreateByInherit t2 = new CreateByInherit();
//        t1.start();
//        t2.start();
        MyThread t3 = new MyThread();
        MyThread t4 = new MyThread();
        t3.start();
        t4.start();
    }
}
```

- Runnable

```java
public class CreateByRunnable implements Runnable {
    private String name;

    public CreateByRunnable() {
    }

    public CreateByRunnable(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    @Override
    public void run() {
        System.out.println(getName());
    }

    public static void main(String[] args) {
        CreateByRunnable r1 = new CreateByRunnable("1");
        Thread t1 = new Thread(r1);
        Thread t2 = new Thread(r1);
        t1.start();
        t2.start();
    }

}
```

- Callable+FutureTask

```java
public class createByFutureTask implements Callable<String> {
    @Override
    public String call() throws Exception {
        return "执行完毕";
    }

    public static void main(String[] args) {
        FutureTask<String> task = new FutureTask<>(new createByFutureTask());
        Thread t1 = new Thread(task);
        t1.start();

        try {
            String s = task.get();
            System.out.println(s);
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }

    }
}
```

