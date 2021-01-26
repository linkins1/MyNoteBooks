# 2.高级篇

### 2.5 JUC线程池

#### 2.5.1 `ThreadPoolExecutor`

##### 1）UML

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/image-20201211095526312378.png" alt="image-20201211095526378" style="zoom:67%;" />

从上面图中可以看出`Executor`接口是线程池操作器的顶级接口，`ExecutorService`接口继承了此接口并给出了一些重要扩展方法，有如下几个

- `execute`--Executor
- `submit`--ExecutorService
- `shutdownNow`--ExecutorService

###### （1）线程池状态相关

对于这两个接口的常用实现类通常为`ThreadPoolExecutor`，此类包含以下重要属性

- `RUNNING`（1110 0000 0000 0000 0000 0000 0000 0000）

  线程池的初始状态，代表新的任务和队列中的任务都会被执行

- `SHUTDOWN`（0000 0000 0000 0000 0000 0000 0000 0000）

  代表不接受新加入线程池的任务，但是会处理队列中的任务

- `STOP`（0010 0000 0000 0000 0000 0000 0000 0000）

  代表既不处理新任务也不处理队列中的任务并且会中断正在执行的任务

- `TIDYING`（0100 0000 0000 0000 0000 0000 0000 0000）

  所有的任务都结束且workCount为0。转移到此状态的线程会执行`terminated()`方法

- `TERMINATED`（0110 0000 0000 0000 0000 0000 0000 0000）

  在`terminated()`方法执行完成后处于该状态

注意到上面5个状态字段均为整型变量且**只有高三位**有值，且五种状态的数值是**递增的**，这便于判断线程处于何种状态。

- `CAPACITY`

  值为0001 1111 1111 1111 1111 1111 1111 1111，用于获取当前的工作线程数量也即workerCount

- `ctl`

  是一个整型变量，其高三位保存状态，低27位保存工作线程数。此变量由`new AtomicInteger(ctlOf(RUNNING, 0))`得到，也即在初始状态时ctl的值和RUNNING一致

###### （2）任务相关

- `workQueue`

  用于存放没有被及时处理的任务，此对象默认创建为`LinkedBlockingQueue`对象

- `workers`

  `HashSet`对象，用于存放所有的工作线程

- `mainLock`

  `ReentrantLock`实例，在需要修改线程池状态时需要上锁

- `termination`

  mainLock的一个条件变量，用于配合awaitTermination方法使用

###### （3）线程池容量相关

- `corePoolSize`

  代表允许的最大核心工作线程数

- `maximumPoolSize`

  代表允许的最大工作线程数，包括核心线程数量，也即此变量可以设定为大于等于`corePoolSize`

- `allowCoreThreadTimeOut`

  默认为false，代表核心线程即使处于空闲状态也不会终止

- `keepAliveTime`

  当线程池中的线程数量大于`corePoolSize`或者`allowCoreThreadTimeOut`为true时，如果此值**不为0**，那么工作线程包括核心线程会存活到keepAliveTime结束，如果为0，则直接结束；**否则**，会一直存活等待任务出现

  > 这里的等待反映在从workQueue获取任务时调用的是定时的阻塞获取（`poll(keepAliveTime, TimeUnit.NANOSECONDS) `）还是不定时的阻塞获取（`workQueue.take()`）

- `defaultHandler`

  此变量是RejectedExecutionHandler类型，这是一个接口，有四个实现类，当线程池满(大于`maximumPoolSize`)且工作队列满时，会根据这个类的四个不同的实现类来做出相应的策略，分别如下

  - `CallerRunsPolicy`

    将任务交给调用execute方法的线程来完成

  - `DiscardPolicy`

    仅将任务丢弃

  - `DiscardOldestPolicy`

    从工作队列中将队头也即最老的任务丢弃，并执行当前任务

  - `AbortPolicy`

    默认策略，抛出`RejectedExecutionException`异常并丢弃任务

##### 2）线程池的创建

###### （1）手动创建--推荐√

直接调用构造器new对象即可，推荐这种方式，这样可以更清楚的知道创建出的线程池的情况，可以按需定制线程池

###### （2）使用`Executors`创建--不推荐×

这是一个工具类，提供了多种静态方法用来创建不同类型的线程池

- `newFixedThreadPool`

  ```java
  public static ExecutorService newFixedThreadPool(int nThreads) {
      return new ThreadPoolExecutor(nThreads, nThreads,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>());
  }
  ```

  创建一个线程数大小固定的线程池，其中核心线程数和最大线程数相同，keepAliveTime为0，代表核心线程会一直等待新任务的出现，并设定workQueue为`LinkedBlockingQueue`对象，此外，ThreadFactory对象默认为`DefaultThreadFactory`对象

  此方法还存在一个可以指定ThreadFactory对象的重载方法。

- `newWorkStealingPool`

  ```java
  public static ExecutorService newWorkStealingPool(int parallelism) {
      return new ForkJoinPool
          (parallelism,
           ForkJoinPool.defaultForkJoinWorkerThreadFactory,
           null, true);
  }
  ```

  创建一个ForkJoinPool，其中可以指定并发度，也即要使用的CPU核心数

  此方法存在一个重载，默认使用最大CPU核心数

- `newSingleThreadExecutor`

  ```java
  public static ExecutorService newSingleThreadExecutor() {
      return new FinalizableDelegatedExecutorService
          (new ThreadPoolExecutor(1, 1,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>()));
  }
  ```

  此方法会创建一个核心线程数和最大线程数都为1的线程池，keepAliveTime为0，且workQueue为`LinkedBlockingQueue`对象

- `newCachedThreadPool`

  ```java
  public static ExecutorService newCachedThreadPool() {
      return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
  }
  ```

  创建一个核心线程数为0，最大线程数为`Integer.MAX_VALUE`的线程池，其中keepAliveTime为60s，且workQueue为`SynchronousQueue`对象，此类阻塞队列的特征是一个线程对队列完成的插入操作必须有另一个线程完成对应的删除操作。在这种情况下，由于设定了keepAliveTime为60s，那么当一个空闲线程在等待60s内主线程没有提交新的任务，那么就会终止

##### 3）任务提交的两种方式

###### （1）`execute`

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

这个方法**分三步走**

- **尝试创建核心工作线程**
  
  如果当前运行的工作线程数量**小于`corePoolSize`** ，那么尝试新建一个线程（**传入command**作为其第一个执行的任务），在调用addWorker创建新线程时会首先再次检查当前线程池的状态，如果不能再新增工作线程则会终止创建。如果工作线程数大于等于核心工作线程数或者新建线程失败，**则进入下一步**

- **将任务入队**

  如果一个任务（command）可以被成功入队，那么我们仍然需要再次检查能否新增一个工作线程，如果这期间可能有旧的工作线程退出使得工作线程数为0，那么新增一个线程（**不传入command，等待从队列中poll取**）；如果这期间线程池已经关闭，那么需要回滚入队操作。如果线程池不处于运行状态或者工作队列插入任务失败，**则进入下一步**

- **尝试创建工作线程**

  尝试新增一个工作线程（**传入command**），如果创建失败，那么意味着线程池已经关闭或者饱和，此时拒绝此任务，并根据饱和策略来做相应的处理

> 这里要注意，核心工作线程和工作线程都是工作线程，也即都是`Worker`对象，区别仅在于核心工作线程是在线程池中的线程数小于corePoolSize时创建的，而工作线程是在大于等于corePoolSize小于maximumPoolSize时创建的，且线程池会动态的保留corePoolSize个线程作为核心线程

###### （2）`submit`

此方法有三个重载，分别如下

- `submit(Runnable task)`

  ```java
  public Future<?> submit(Runnable task) {
      if (task == null) throw new NullPointerException();
      RunnableFuture<Void> ftask = newTaskFor(task, null);
      execute(ftask);
      return ftask;
  }
  ```

- `submit(Runnable task, T result)`

  ```java
  public <T> Future<T> submit(Runnable task, T result) {
      if (task == null) throw new NullPointerException();
      RunnableFuture<T> ftask = newTaskFor(task, result);
      execute(ftask);
      return ftask;
  }
  ```

- `submit(Callable<T> task)`

  ```java
  public <T> Future<T> submit(Callable<T> task) {
      if (task == null) throw new NullPointerException();
      RunnableFuture<T> ftask = newTaskFor(task);
      execute(ftask);
      return ftask;
  }
  ```

前两个方法均为将传入的Runnable对象封装为FutureTask对象，由于Runnable接口方法的返回值为null，所以第一个方法返回的FutureTask中的result为null，第二个方法返回的FutureTask的result为方法传入的result

第三个方法是传入一个Callable对象，此对象会赋值给FutureTask对象中的callable属性，由于Callable对象有返回值，因而这里执行过后，返回的FutureTask对象中的result为Callable的返回值

**这三个方法的共性是**，在包装好FutureTask后，仍然调用execute方法来完成任务执行，**由此，我们也可以发现**，可以自定义FutureTask对象并传入给execute方法，之后使用自定义FutureTask对象同样可以拿到其中的result

##### 4）创建工作线程

上面提到的execute方法中，使用addWorker方法来完成工作线程的创建，在分析此方法之前，先分析`ThreadPoolExecutor`中的`Worker`类

###### （1）`Worker`

- **UML**

  <img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/image-20201212145329343.png" alt="image-20201212145329343" style="zoom:67%;" />

  <img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/image-20201212144733709.png" alt="image-20201212144733709" style="zoom: 50%;" />

  - `thread`

    final类型的Thread对象，用于存放这个Worker代表的线程

  - `firstTask`

    用于存放要执行的第一个任务，可能为null

  - `completedTasks`

    此Worker代表的线程完成的任务数

  这里注意到，此类继承于AQS，其重写的`tryAcquire`和`tryRelease`方法如下

  - `tryAcquire`

    ```java
    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }
    ```

  - `tryRelease`

    ```java
    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }
    ```

  这两个方法分别在lock和unlock中调用，可以看到其使用的独占方式来操作state变量，其意图将在`runWorker`方法中体现

  此外，此类还实现了Runnable接口，重写了其中的run方法，这是为了将run执行的具体过程交由`runWorker`方法来完成，如下所示

  ```java
  /** Delegates main run loop to outer runWorker  */
  public void run() {
      runWorker(this);
  }
  ```

- **构造器**

  ```java
  Worker(Runnable firstTask) {
      setState(-1); // inhibit interrupts until runWorker
      this.firstTask = firstTask;
      this.thread = getThreadFactory().newThread(this);
  }
  ```

  此构造器中会先设定state变量为`-1`，代表刚刚创建，将传入的firstTask传入给成员变量firstTask，并调用DefaultFactory.newThread方法来完成线程创建。

  **这里需要说明两点，为何设定为-1，以及newThread的创建逻辑**

  - **为何设定为-1**

    在Worker类中存在下面方法

    ```java
    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
    ```

    此方法在state大于等于0，线程对象不为空且未被中断过的情况下会调用线程的interrupt方法来中断自己。

    这里设定为-1，那么代表在Worker被创建的初始状态不允许被中断

  - **newThread的创建逻辑**

    由于一般使用`DefaultThreadFactory`对象来完成，此类是Executors的内部类，这里首先介绍此类

    ```java
    static class DefaultThreadFactory implements ThreadFactory {
        private static final AtomicInteger poolNumber = new AtomicInteger(1);
        private final ThreadGroup group;
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;
    
        DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                                  Thread.currentThread().getThreadGroup();
            namePrefix = "pool-" +
                          poolNumber.getAndIncrement() +
                         "-thread-";
        }
    
        public Thread newThread(Runnable r) {
            Thread t = new Thread(group, r,
                                  namePrefix + threadNumber.getAndIncrement(),
                                  0);
            if (t.isDaemon())
                t.setDaemon(false);
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY);
            return t;
        }
    }
    ```

    此处包含四个属性

    - `poolNumber`

      AtomicInteger类型，代表线程池编号

    - `group`

      ThreadGroup类型，代表线程处于的分组

    - `threadNumber`

      AtomicInteger类型，代表对应池中的线程编号

    这里注意到，poolNumber为静态成员变量，也即此变量是所有实例共享的，又注意到当调用此类的构造器新建线程池时，此变量会调用getAndIncrement()来增加线程池编号

    而对于threadNumber变量，其会在线程池实例调用newThread方法时将此变量增1，用于创建新的线程，此外，还会将新创建出的线程设定为非守护线程并设定为常规优先级最终返回新创建好的线程

###### （2）`addWorker(Runnable firstTask, boolean core)`

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

此方法执行步骤如下

- 循环**检查线程池当前的状态**（📌），如果是大于SHUTDOWN或者是SHUTDOWN且（firstTask为null或者任务队列为空）时，会返回false，否则下一步
- 如果要创建核心工作线程，也即创建线程的最大数量要求为corePoolSize（此时core为true），否则为创建工作线程此时core为false，线程池最大数量要求为maximumPoolSize。先检查是否超出限制，如果没有，则尝试CAS增加线程数，如果
  - 失败则检查线程池状态是否已经改变，如果改变，从头开始，否则再次执行内循环
  - 成功则跳出两层循环
- 至此，线程池中的线程计数变量成功加1，之后创建Worker对象
- 首先新建Worker对象，并取出其中的线程对象，正常情况下此对象不为null
- 调用mainLock的lock方法，锁定线程池状态，这里需要再次检查线程池状态，因为**距离上一次检查期间**（📌），线程池状态可能已经发生变化，如果线程池状态为RUNNING或者为SHUTDOWN且firstTask为null，那么将新创建的worker对象加入workers集合中，并设定`workerAdded = true`，否则会抛出异常，最终会执行到`addWorkerFailed(w)`将加入workers集合中的对象移除
- 执行mainLock的unlock方法，之后如果workerAdded为true，则调用线程的start方法，这时会调用到Worker中重写的run方法，其中会执行`runWorker`方法，并设定workStarted为true
- 返回workStarted

###### （3）`runWorker(Worker w)`

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            //如果线程池停止了，则确保线程被中断，否则确保其不被中断。
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

此方法执行步骤如下

- 取出其中的firstTask对象，并将其设定为null，**之后调用 w.unlock()方法，将state设定为0，**注意到，**这时Worker对象会记录线程中断情况**
- 之后进入循环体，只要当前Worker的firstTask不是null或者workQueue不是空，则一直会处理等待任务，此处首先会执行`w.lock();`方法，设定state为1。如果初次进入循环体失败，则completedAbruptly仍然为true，直接去执行`processWorkerExit(w, completedAbruptly)`方法
- 判断线程池状态，如果线程池停止了，则确保线程被中断，否则确保其不被中断
- 调用`task.run`方法，也即传入的Runnable对象中的run方法
- 退出循环体后，会设定completedAbruptly为false，并执行`processWorkerExit(w, completedAbruptly)`方法

###### （4）`getTask()`

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

这个方法是体现keepAliveTime的核心，当设定allowCoreThreadTimeOut为true或者线程数量大于corePoolSize时，会令timed为true，这时会执行带计时的poll方法，否则会执行take方法一直阻塞到队列中出现新任务，由此可以发现，对于最先创建的几个线程，其数量小于corePoolSize，那么其会执行take方法阻塞等待任务，而多出来的那部分线程会计时等待。因而，线程池中会**动态的保留corePoolSize个线程来一直处理任务队列**，这些线程也即核心线程，**之所以说动态**，是因为并不是最初创建的几个线程一直都是核心线程，这和任务队列中任务的提交情况有关。

###### （5）`processWorkerExit(Worker w, boolean completedAbruptly)`

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    tryTerminate();

    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```

这里主要关注completedAbruptly带来的影响，当其为true时，代表当前worker在runWorker的try（执行具体任务）中出现了异常，但凡是正常处理完任务以及工作队列中任务的worker都会将completedAbruptly置为false。那么在completedAbruptly

- 为true时，会将此出现异常的worker删除并将计数值减1，之后判断线程池状态是否小于STOP，也即为SHUTDOWN或RUNNING，如果是则创建并添加一个新的工作线程

- 为false时，如果allowCoreThreadTimeOut为

  - true且工作队列不为空，则至少保留一个工作线程用于处理任务
  
  - false则至少保留corePoolSize个工作线程于线程池内

之后会判断线程池状态是否为RUNNING或者SHUTDOWN，如果是则在worker执行过任务时，也即completedAbruptly为false时会判断是否有必要新增一个worker对象，且不传入command

##### 5）线程池状态转换

###### （1）RUNNING->SHUTDOWN

调用`shutdown()`方法时

###### （2）(RUNNING or SHUTDOWN) -> STOP

调用`shutdownNow()`方法时

###### （3）SHUTDOWN -> TIDYING

当队列和池都为空时

###### （4）STOP -> TIDYING

当池为空

###### （5）TIDYING -> TERMINATED

当`terminated()`方法执行完毕

> 下面部分借鉴自[JavaGuide](https://snailclimb.gitee.io/javaguide/#/./docs/java/multi-thread/java线程池学习总结?id=_432-isterminated-vs-isshutdown)

这里需要注意`shutdown()`和`shutdownNow()`的区别如下

- `shutdown()` :关闭线程池，线程池的状态变为 `SHUTDOWN`。线程池不再接受新任务了，但是队列里的任务得执行完毕。
- `shutdownNow()`:关闭线程池，线程的状态变为 `STOP`。线程池会终止当前正在运行的任务，并停止处理排队的任务并返回剩余的任务队列。

此外，`isTerminated()` 和 `isShutdown()`区别如下

- `isShutDown`当调用 `shutdown()` 方法后返回为 true。
- `isTerminated`当调用 `shutdown()` 方法后，并且所有提交的任务完成后返回为 true

##### 6）总结

总体的执行流程如下

- 提交任务

- 线程数是否大于corePoolSize？

  - 大于

    workerQueue入队成功？

    - 失败

      线程数是否大于maximumPoolSize？

      - 是

        按照指定的策略完成善后

      - 否

        创建线程执行任务

    - 成功

      将任务入队

  - 小于等于

    创建线程执行任务

但凡涉及到新增Worker的部分，都需要检查线程池的状态是否允许新增工作线程

#### 2.5.2 `ScheduledThreadPoolExecutor`

这个类继承于`ThreadPoolExecutor`，主要是为了能够定时执行任务，使用频率较低

#### 2.5.3 `ForkJoinPool`

***//待办***
