# 2.高级篇

### 2.1 ThreadLocal相关

#### 2.1.1 ThreadLocal

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

#### 2.1.2 InheritableThreadLocal

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

###### （3）与ThreadLocal比较

- **同**
  - 两者在Thread类中都存储在类型为ThreadLocalMap的变量中，作为key出现
  - 两者对象都通过调用get/set方法对对应线程的map对象进行操作

- **异**
  - ThreadLocal在初始化线程时不会将父线程中的map复制给子线程
  - InheritableThreadLocal在初始化线程时会将父线程中的map复制给子线程

最后给出一张示意图

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/image-20201107172206317.png" alt="image-20201107172206317" style="zoom:150%;" />

#### 2.1.3 ThreadLocalRandom原理

##### 1）Random原理

###### （1）成员变量

```java
private final AtomicLong seed;
```

seed变量负责随机数的生成和更新

###### （2）随机数生成

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

###### （3）并发问题

上面看到，如果多个线程操作的是同一个`Random对象`，那么初始的种子都是**相同的**，**由于计算新种子和随机数的算法是固定的**，在并发情况下会出现多个线程计算出的新种子和随机数是**相同的**。

为了解决这个问题，Random类中通过**CAS**来完成新旧种子的替换操作，所以在多个线程同时调用nextInt方法获取随机数时，只会有一个线程的CAS能够成功，对于其他CAS失败的线程就需要循环获取最新的种子再计算新种子。

可见上面的CAS会带来大量的自旋开销，所以定义了ThreadLocalRandom类来解决此问题

##### 2）ThreadLocalRandom原理

###### （1）Thread相关

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

###### （2）继承关系

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/ThreadLocalRandom123.png" alt="ThreadLocalRandom" style="zoom:67%;" /> 

ThreadLocalRandom是继承于Random类的，其抛弃了`Random`类中的`seed`变量

###### （3）使用步骤

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

###### （4）原理

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

###### （5）总结

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