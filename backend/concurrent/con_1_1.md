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

此方法是Thread类的静态方法，在对应线程中调用此方法会使此线程**进入阻塞队列**，**让出CPU**并且**保留休眠前获得的所有锁**，如果在休眠期间被其他线程中断，则会抛出InterruptedException，如果正常休眠结束，则会进入就绪队列，再次参与CPU调度

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

#### 1.1.8ThreadLocal

##### 1）定义

利用ThreadLocal可以实现在不同子线程中拥有父线程中**某一个资源的独立拷贝**，这样保证了每个线程可以单独操作各自的拷贝资源

##### 2）示例

```java
public class ThreadLocalExp {
    static ThreadLocal<String> stringThreadLocal = new ThreadLocal<>();
    static ThreadLocal<ArrayList<Integer>> arrayListThreadLocal = new ThreadLocal<>();

    static Object object = new Object();

    static void printThreadLocals(){
        System.out.println(stringThreadLocal.get());
        System.out.println(arrayListThreadLocal.get());
    }

    public static void main(String[] args) {
        stringThreadLocal.set("main");
        ArrayList<Integer> integers = new ArrayList<>();
        integers.add(0);
        arrayListThreadLocal.set(integers);
        printThreadLocals();

        Thread threadA = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (object){
                    stringThreadLocal.set("Thread-A");
                    ArrayList<Integer> integers = new ArrayList<>();
                    integers.add(1);
                    arrayListThreadLocal.set(integers);
                    printThreadLocals();
                }
            }
        });

        Thread threadB = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (object){
                    stringThreadLocal.set("Thread-B");
                    ArrayList<Integer> integers = new ArrayList<>();
                    integers.add(2);
                    arrayListThreadLocal.set(integers);
                    printThreadLocals();
                }
            }
        });

        threadA.start();
        threadB.start();

    }
}
```

其使用步骤如下

- 在主线程中定义`ThreadLocal`对象`xxxthreadLocal`
- 在重写子线程`run`方法时，调用刚定义好的`xxxthreadLocal`对象的set方法，设定指定泛型的值

其输出如下

```console
main
[0]
Thread-A
[1]
Thread-B
[2]
```

##### 3）原理

###### （1）threadLocals

在`Thread`类中，有一个成员变量如下

```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```

其是`ThreadLocal`的静态内部类`ThreadLocalMap`，其部分定义如下

```java
static class ThreadLocalMap {
    
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    /**
     * The initial capacity -- MUST be a power of two.
     */
    private static final int INITIAL_CAPACITY = 16;

    /**
     * The table, resized as necessary.
     * table.length MUST always be a power of two.
     */
    private Entry[] table;
```

其中存在一个`Entry`类继承于`WeakReference`，其类似于HashMap中的Entry，其中

- k是`ThreadLocal`对象
- v是Object

此外，`table`是一个Entry数组，用于存放多个Entry

>弱引用也是用来描述那些非必须对象，强度**比软引用更弱**，被弱引用关联的对象**只能生存到下一次垃圾收集发生**。当垃圾收集器开始工作，**无论当前内存是否足够，都会回收掉只被弱引用关联的对象**。

###### （2）ThreadLocal

上面例子中，首先创建了一个ThreadLocal对象，会调用空参构造器，之后调用`set`方法，其定义如下

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

方法完成了

- 获取当前线程`t`

- 获取`ThreadLocalMap`对象

  ```java
  ThreadLocalMap getMap(Thread t) {
      return t.threadLocals;
  }
  ```

  这里由于传入的是当前线程的`Thread`对象，所以这里获取到的也就是前面提到的`threadLocals`

- 如果

  - map为空，调用`createMap`

    ```java
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
    ```

    这里新建了`ThreadLocalMap`对象并赋给了`当前线程的threadLocals变量`，其中，调用如下构造器

    ```java
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }
    ```

    其中

    - 先新建了Entry数组赋给前面提到的`table`数组
    - 之后计算出传入的`ThreadLocal`对象的hash值并对容量取模得到索引`i`
    - 创建Entry条目，将其放入数组中索引`i`处

    这里计算哈希值时利用的`AtomicInteger`来保证线程安全

  - 否则直接将`this`(ThreadLocal引用)和value作为k-v对设定入map

    ```java
    private void set(ThreadLocal<?> key, Object value) {
    
        Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1);
    
        for (Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            ThreadLocal<?> k = e.get();
    
            if (k == key) {
                e.value = value;
                return;
            }
    
            if (k == null) {
                replaceStaleEntry(key, value, i);
                return;
            }
        }
    
        tab[i] = new Entry(key, value);
        int sz = ++size;
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            rehash();
    }
    ```

    其

    - 首先从ThreadLocal的静态内部类ThreadLocalMap中拿到刚刚创建好的`table`数组
    - 之后根据传入的`ThreadLocal`的hash值计算索引`i`
    - 设定时
      - 首先判断数组`i`位置是否为空，如果为空，则直接创建Entry对象并放入
      - 否则，
        - 先判断对应位置的key与传入的key是否为同一对象，如果是则**替换**原来的值
        - 如果不是同一对象，则继续向后查找
        - 如果原来的key为null，则将其替换为当前key-value构成的`Entry`对象

之后通过`get`方法，以`ThreadLocal对象为key`查询到对应的value，方法体如下

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

这里

- 先获取到当前线程对象`t`

- 和set方法中一致，利用`t`获取对应的map

- 如果map

  - 不是空，则直接取出其中的Entry条目`e`；如果`e`不是空，则转型后返回，否则调用setInitialValue

  - 否则，调用setInitialValue

    ```java
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
    ```

    - 将value置为null
    - 获取到当前线程对象t
    - 根据t获得map
    - 如果
      - map不是空，则直接以this为key，null为值创建一个Entry并设定
      - 否则调用createMap

#### 1.1.9InheritableThreadLocal

##### 1）定义

由于ThreadLocal定义的变量不能再父子线程间继承，所以定义了此类，其继承于`ThreadLocal`，重写了下面三个方法

```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    protected T childValue(T parentValue) {
        return parentValue;
    }
    
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }
    
	void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

由于调用set和get方法时仍然是调用父类ThreadLocal的方法，`getMap`和`createMap`被重写后，获取到的map对象就变为`InheritableThreadLocal`对象

##### 2）示例

```java
public class InheritableThreadLocalExp {
    static InheritableThreadLocal<String> stringThreadLocal = new InheritableThreadLocal<>();
    static InheritableThreadLocal<ArrayList<Integer>> arrayListThreadLocal = new InheritableThreadLocal<>();

    static Object object = new Object();

    static void printThreadLocals(){
        System.out.println(stringThreadLocal.get());
        System.out.println(arrayListThreadLocal.get());
    }

    public static void main(String[] args) {
        System.out.println("------------------main---------------------");
        stringThreadLocal.set("main");
        ArrayList<Integer> integers = new ArrayList<>();
        integers.add(0);
        arrayListThreadLocal.set(integers);
        printThreadLocals();
        System.out.println("------------------main---------------------");

        Thread threadA = new Thread(new Runnable() {
            InheritableThreadLocal<String> stringThreadLocalA = new InheritableThreadLocal<>();
            InheritableThreadLocal<ArrayList<Integer>> arrayListThreadLocalA = new InheritableThreadLocal<>();
            @Override
            public void run() {
                synchronized (object){
                    System.out.println("------------------A---------------------");
                    stringThreadLocalA.set("Thread-A");
                    ArrayList<Integer> integers = new ArrayList<>();
                    integers.add(1);
                    arrayListThreadLocalA.set(integers);
                    printThreadLocals();
                    printThreadLocalsA();
                    System.out.println("------------------A---------------------");
                }
            }
            private void printThreadLocalsA(){
                System.out.println(stringThreadLocalA.get());
                System.out.println(arrayListThreadLocalA.get());
            }
        });

        Thread threadB = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (object){
                    System.out.println("------------------B---------------------");
                    //由于不论是inheritableThreadLocal还是ThreadLocal都是以对象引用作为key，所以B中设定的值
//                    会覆盖父线程中设定的value
                    stringThreadLocal.set("Thread-B");
                    ArrayList<Integer> integers = new ArrayList<>();
                    integers.add(2);
                    arrayListThreadLocal.set(integers);
                    printThreadLocals();
                    System.out.println("------------------B---------------------");
                }
            }
        });

        threadA.start();
        threadB.start();

    }
}
```

- 定义`InheritableThreadLocal`对象
- 在重写run方法时，无需调用该对象的set方法就可以获取父线程中存储的变量信息

##### 3）原理

###### （1）inheritableThreadLocals

```java
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

上面也注意到，此类中重写的方法中将map对象赋值给了`Thread类`中的`inheritableThreadLocals`成员变量，且getMap是也是获取的此对象

###### （2）init

在创建线程时，前面提到会调用init函数来初始化线程，其相关函数体如下

```java
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc,
                  boolean inheritThreadLocals) {
   Thread parent = currentThread();
   if (inheritThreadLocals && parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
}
```

此处传入的`inheritThreadLocals`参数默认为`true`，`parent`为当前线程，也即新建此线程的线程是父线程

此处会调用ThreadLocal的静态方法`createInheritedMap`，将当前线程的inheritableThreadLocals参数传入，其返回值为`ThreadLocalMap`，在传递给此Thread对象中的`inheritableThreadLocals`变量，方法调用如下

```java
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    return new ThreadLocalMap(parentMap);
}
```

```java
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```

此私有重载构造器执行下面步骤

- 获取到父线程`inheritableThreadLocals`变量的`table`数组

- 创建新的`table`数组

- 遍历父线程的table

  - 获取每一个Entry对象的key（ThreadLocal对象）
  - 调用childValue方法，返回转型后的Entry条目的value
  - 利用父线程的key和刚转型过的value创建Entry对象
  - 计算哈希值及索引，将其放入子线程的table中

  这里注意到，由于获取到的是父线程中的k-v来新建entry对象，其实也就是父entry的一个拷贝，**但是**，其中存储的k仍然是同一个引用，因而计算得到的哈希值和索引在子table中和父table是一致的，**这也是为什么子线程中如果使用同一个ThreadLocal变量设定值会覆盖父线程中设定的值的原因**

##### 4）与ThreadLocal比较

###### （1）同

- 两者在Thread类中都存储在类型为ThreadLocalMap的变量中，作为key出现
- 两者对象都通过调用get/set方法对对应线程的map对象进行操作

###### （2）异

- ThreadLocal在初始化线程时不会将父线程中的map复制给子线程
- InheritableThreadLocal在初始化线程时会将父线程中的map复制给子线程

最后给出一张示意图

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/image-20201107172206317.png" alt="image-20201107172206317" style="zoom:150%;" />

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

