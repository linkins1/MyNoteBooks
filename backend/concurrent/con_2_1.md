# 2.高级篇

### 2.1 ThreadLocalRandom原理

#### 2.1.1 Random原理

##### 1）成员变量

```java
private final AtomicLong seed;
```

seed变量负责随机数的生成和更新

##### 2）随机数生成

步骤如下

- 获取一个`Random`实例
- 调用`nextInt(x)`方法，x为上界限

```java
public int nextInt(int bound) {
    if (bound <= 0)
        throw new IllegalArgumentException(BadBound);

    int r = next(31);
    int m = bound - 1;
    if ((bound & m) == 0)  // i.e., bound is a power of 2
        r = (int)((bound * (long)r) >> 31);
    else {
        for (int u = r;
             u - (r = u % bound) + m < 0;
             u = next(31))
            ;
    }
    return r;
}
```

其中`next`方法如下

```java
protected int next(int bits) {
    long oldseed, nextseed;
    AtomicLong seed = this.seed;
    do {
        oldseed = seed.get();
        nextseed = (oldseed * multiplier + addend) & mask;
    } while (!seed.compareAndSet(oldseed, nextseed));
    return (int)(nextseed >>> (48 - bits));
}
```

这里会**CAS**的将旧种子替换成新种子，之后将新种子处理返回后用于生成新的随机数

##### 3）并发问题

上面看到，如果多个线程操作的是同一个`Random对象`，那么初始的种子都是**相同的**，**由于计算新种子和随机数的算法是固定的**，在并发情况下会出现多个线程计算出的新种子和随机数是**相同的**。

为了解决这个问题，Random类中通过**CAS**来完成新旧种子的替换操作，所以在多个线程同时调用nextInt方法获取随机数时，只会有一个线程的CAS能够成功，对于其他CAS失败的线程就需要循环获取最新的种子再计算新种子。

可见上面的CAS会带来大量的自旋开销，所以定义了ThreadLocalRandom类来解决此问题

#### 2.1.2 ThreadLocalRandom原理

##### 1）Thread相关

```java
/** The current seed for a ThreadLocalRandom */
@sun.misc.Contended("tlr")
long threadLocalRandomSeed;

/** Probe hash value; nonzero if threadLocalRandomSeed initialized */
@sun.misc.Contended("tlr")
int threadLocalRandomProbe;

/** Secondary seed isolated from public ThreadLocalRandom sequence */
@sun.misc.Contended("tlr")
int threadLocalRandomSecondarySeed;
```

这里定义了三个基本类型变量，分别是种子，探针还有另一个种子（LongAdder中使用）

上面的三个变量在`Thread`类中的**偏移量**会通过`Unsafe`对象分别存储于`ThreadLocalRandom`中的下面三个常量中

- SEED
- PROBE
- SECONDARY

##### 2）继承关系

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/ThreadLocalRandom123.png" alt="ThreadLocalRandom" style="zoom:67%;" /> 

ThreadLocalRandom是继承于Random类的，其抛弃了`Random`类中的`seed`变量

##### 3）使用步骤

```java
public class ThreadLocalRandomTest {

    static class Worker implements Runnable{

        @Override
        public void run() {
            ThreadLocalRandom random = ThreadLocalRandom.current();
            System.out.println(random.nextInt());
        }
    }

    public static void main(String[] args) {
        Thread t1 = new Thread(new Worker());
        Thread t2 = new Thread(new Worker());
        t1.start();
        t2.start();
    }
}
```

首先获取一个`ThreadLocalRandom`对象，之后调用此对象的`nextInt`方法获取随机数

##### 4）原理

上面例子中，先调用`current()`方法，方法体如下

```java
public static ThreadLocalRandom current() {
    if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)
        localInit();
    return instance;
}
```

其中

- `UNSAFE`是`ThreadLocalRandom`中的一个静态变量，其初始化动作发生在静态代码块中

  ```java
  static {
      try {
          UNSAFE = sun.misc.Unsafe.getUnsafe();
          Class<?> tk = Thread.class;
          SEED = UNSAFE.objectFieldOffset
              (tk.getDeclaredField("threadLocalRandomSeed"));
          PROBE = UNSAFE.objectFieldOffset
              (tk.getDeclaredField("threadLocalRandomProbe"));
          SECONDARY = UNSAFE.objectFieldOffset
              (tk.getDeclaredField("threadLocalRandomSecondarySeed"));
      } catch (Exception e) {
          throw new Error(e);
      }
  }
  ```

  其中，`getUnsafe()`获取的是`Unsafe`的**单例**对象，此外，这里还对前面提到的三个**偏移量**进行了赋值

- 利用UNSAFE判断当前线程中的`threadLocalRandomProbe`是否为0，如果是0，那么代表是第一次使用，需要初始化，调用`localInit()`方法，方法体如下

  ```java
  static final void localInit() {
      int p = probeGenerator.addAndGet(PROBE_INCREMENT);
      int probe = (p == 0) ? 1 : p; // skip 0
      long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));
      Thread t = Thread.currentThread();
      UNSAFE.putLong(t, SEED, seed);
      UNSAFE.putInt(t, PROBE, probe);
  }
  ```

  其中

  - `probeGenerator`是`ThreadLocalRandom`中的`AtomicInteger`对象，同样是单例的，其通过**CAS**方法`addAndGet`来初始化`threadLocalRandomProbe`变量
  - `seeder`是`ThreadLocalRandom`中的`AtomicLong`对象，同样是单例的，其也会通过**CAS**方法`getAndAdd`来初始化`threadLocalRandomSeed`变量

  此处要注意，初始化动作只会**触发一次**，且是通过**CAS**完成的，之所以使用**CAS**是因为多个线程调用`current()`方法时是并发执行的，为了保证每个线程设定的`probeGenerator`和`seeder`是不同且唯一的，所以必须保证操作是原子性的

- 最终返回`instance`对象，其是`ThreadLocalRandom`的一个单例对象，那么多个线程获得的`ThreadLocalRandom`对象其实是相同的，但是其对应线程中初始化的种子和指针是不同的

##### 5）总结

与`ThreadLocal`的逻辑很类似，同样是一个工具类，具有如下特征

- 多个线程操作的工具类对象是同一个
- 多个线程分别存储独立的变量被工具类操作

此外，`ThreadLocalRandom`还具有如下独有的特征

- 在调用`current()`方法时才会触发对应线程使用种子和指针的初始化
- 内部工具类对象均采用单例模式
- 只有初始化动作是CAS的，之后种子和指针的更新都是线程局部的问题，不需要CAS操作

> **一个疑惑**
>
> 在初始化线程中的种子和指针时，是通过`UNSAFE`对象来直接对内存进行操作，而不是在`Thread`类中提供set方法来更新值，我的猜想是，由于`threadLocalRandomProbe`和`threadLocalRandomSeed`都用`@Contended`注解标识，这样可以保证每个变量仍然是独占缓存行的？