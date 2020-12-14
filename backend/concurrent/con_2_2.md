# 2.高级篇

### 2.2 原子操作类

> 此部分的原子性都是通过**CAS**来实现的

#### 2.2.1 `AtomicLong`

`AtomicXXX`可以以看作是基本数据类型的原子性封装类，主要用于完成变量的原子性更改操作，这里举例AtomicLong

##### 1）类结构

> 类似于此类的还有其他的基本数据类型，如AtomicInteger等，其原理都类似

此类的UML图如下

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/image-20201113132908004.png" alt="image-20201113132908004" style="zoom:67%;" />

其中的成员变量分别为

- `unsafe`

  此对象是进行CAS操作的核心，其通过`Unsafe.getUnsafe()`来获得，其是一个单例对象

- `value`

  封装的long型变量

- `valueOffset`

  `value`变量在`AtomicLong`中的偏移量

- `VM_SUPPORTS_LONG_CAS`

  代表JVM是否支持无锁CAS操作

##### 2）APIs

- ` compareAndSet`

  ```java
  return unsafe.compareAndSwapLong(this, valueOffset, expect, update);
  ```

  其是调用unsafe的CAS方法

- `getAndIncrement`

  ```java
  return unsafe.getAndAddLong(this, valueOffset, 1L);
  ```

  返回原来值并将其加1再CAS的放入

- `getAndAdd`

  ```java
  return unsafe.getAndAddLong(this, valueOffset, delta);
  ```

  可以指定加数为delta

- `incrementAndGet`

  ```java
  return unsafe.getAndAddLong(this, valueOffset, 1L) + 1L;
  ```

  最终设定的值和返回值都是累加过的

- 指定操作函数

  - `LongUnaryOperator`

    单目运算函数，需要重写此接口中的对应方法并指定运算函数

  - `LongBinaryOperator`

    双目运算函数，需要重写此接口中的对应方法并指定运算函数

#### 2.2.2 `LongAdder`

##### 1）产生背景

由于在使用`AtomicLong`类型的变量时，是多个线程共用此变量进行CAS操作，由于只有一个线程可以CAS操作成功，那么其他的线程就会进行**大量无效的自旋**操作，期间一直占用CPU。

为了解决这个困境，想到将最终的结果**result**拆分为多个部分的和，那么**只需将多个线程分配到不同的部分上之后再对各自的部分进行CAS操作**就可以减少对**同一个result**，也即AtomicLong的竞争，进一步提高并发度

##### 2）原理

###### （1）UML图

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/image-20201113140013564.png" alt="image-20201113140013564" style="zoom:80%;" />

从上可见，`LongAdder`是继承于`Striped64`的，其中的成员变量如下

- `Striped64`

  - `NCPU`

    当前机器最大的CPU核心数

  - `base`

    基础值

  - `cells`

    多个附加值构成的数组

  - `cellsBusy`

    cells被占用的状况

  - `UNSAFE`

    Unsafe的单例对象，实际存在于此类的静态内部类`Cell`中

  - `BASE`

    base变量在此类中的偏移量

  - `CELLSBUSY`

    cellsBusy变量在此类中的偏移量

  - `PROBE`

    threadLocalRandomProbe在`Thread`类中的偏移量，类似于ThreadLocalRandom中

- `LongAdder`

  - serialVersionUID

    序列化ID

上面这些变量是**拆分逻辑**的一种实现，如下图所示

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/image-20201113150759254.png" alt="image-20201113150759254" style="zoom:150%;" />

如上图所示，最终的结果由`base`和`cells`数组的和求得，这样最终的result就被打成了多个部分，其中

- base用于一开始竞争不激烈的情况，多个线程如果对base的CAS都可以成功，那就只对base进行CAS累加操作
- cells数组一开始为空，在对base进行CAS操作出现失败时，就会触发这个数组的创建，要被累加的值会包装在`Cell`对象中

下面介绍base的CAS操作和cells的初始化、赋值和扩容操作

###### （2）base

`LongAdder`下的add方法如下

```java
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended = a.cas(v = a.value, v + x)))
            longAccumulate(x, null, uncontended);
    }
}
```

如下方法中，如果cells数组

- 是空

  则调用`Striped64`的caseBase方法，如下

  ```java
  final boolean casBase(long cmp, long val) {
      return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
  }
  ```

  其中调用`Striped64`的UNSAFE对象对base变量进行CAS操作

- 不是空或者caseBase竞争失败

  再次判断cells数组是否为空或者长度为0，如果不是，则调用getProbe函数从Thread类中获取`threadLocalRandomProbe`来计算cells数组中的索引，如果索引位置的cell不为空，则调用cell的cas函数进行累加操作，如果返回值为true，则证明cas操作成功

  除了上面的情况，一律进入longAccumulate函数

> `longAccumulate`执行的顺序是赋值->初始化->扩容->对base操作，这四步循环进行，直到数值被累加成功，成功的情况有如下
>
> - cell对象被插入
> - cell对象cas操作成功
> - 对base操作成功

***下面是`longAccumulate`函数的部分***

###### （3）cells

- **Pre**-`Cell`类

  ```java
  @sun.misc.Contended static final class Cell {
      volatile long value;
      Cell(long x) { value = x; }
      final boolean cas(long cmp, long val) {
          return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
      }
  
      // Unsafe mechanics
      private static final sun.misc.Unsafe UNSAFE;
      private static final long valueOffset;
      static {
          try {
              UNSAFE = sun.misc.Unsafe.getUnsafe();
              Class<?> ak = Cell.class;
              valueOffset = UNSAFE.objectFieldOffset
                  (ak.getDeclaredField("value"));
          } catch (Exception e) {
              throw new Error(e);
          }
      }
  }
  ```

  此类是Stripe64的静态内部类，其中

  - value是这个cell对象中存储的值
  - cas函数用于对该Cell实例的value域进行更新操作
  - 使用`@sun.misc.Contended`注解修饰是由于cell对象是存在于cells数组中的，数组中的内容一般是连续存在于缓存行中的，为了解决**伪共享**的问题，使用Contended注解可以保证每个cell对象都独占缓存行

- 初始化

  - 对threadLocalRandomProbe初始化

    ```java
    int h;
    if ((h = getProbe()) == 0) {
        ThreadLocalRandom.current(); // force initialization
        h = getProbe();
        wasUncontended = true;
    }
    ```
    这里调用`ThreadLocalRandom.current()`函数对threadLocalRandomProbe初始化

  - 对cells数组初始化

    当前面对base的操作失败时，会调用Stripe64的`longAccumulate`函数，此函数较长，其中的初始化部分如下

    ```java
    else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
        boolean init = false;
        try {                           // Initialize table
            if (cells == as) {
                Cell[] rs = new Cell[2];
                rs[h & 1] = new Cell(x);
                cells = rs;
                init = true;
            }
        } finally {
            cellsBusy = 0;
        }
        if (init)
            break;
    }
    ```

    这里再判断cellsBusy(代表是否有人占用cells数组，0代表无占用，1代表占用)不忙且通过casCellsBusy将cellsBusy置为1成功后
    - 将cells数组初始化为大小为2的数组
    - 将要插入的值放入cells数组对应索引上
    - 初始化成功后将cellsBusy置为0

- 赋值

  ```java
  Cell[] as; Cell a; int n; long v;
          if ((as = cells) != null && (n = as.length) > 0) {
              if ((a = as[(n - 1) & h]) == null) {
                  if (cellsBusy == 0) {       // Try to attach new Cell
                      Cell r = new Cell(x);   // Optimistically create
                      if (cellsBusy == 0 && casCellsBusy()) {
                          boolean created = false;
                          try {               // Recheck under lock
                              Cell[] rs; int m, j;
                              if ((rs = cells) != null &&
                                  (m = rs.length) > 0 &&
                                  rs[j = (m - 1) & h] == null) {
                                  rs[j] = r;
                                  created = true;
                              }
                          } finally {
                              cellsBusy = 0;
                          }
                          if (created)
                              break;
                          continue;           // Slot is now non-empty
                      }
                  }
                  collide = false;
              }
              else if (!wasUncontended)       // CAS already known to fail
                  wasUncontended = true;      // Continue after rehash
              else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                           fn.applyAsLong(v, x))))
                  break;
              else if (n >= NCPU || cells != as)
                  collide = false;            // At max size or stale
              else if (!collide)
                  collide = true;
  ```

  - 首先对cells数组再做一次判断，如果未被初始化，则会跳转到之后的初始化环节，否则下一步

  - 首先判断cells数组对应位置的cell元素是否为空

    - 如果是空

      则判断cellsBusy是否忙(这里主要检查是否有线程进行扩容操作或插入cell操作)，如果未被占用，则casCellsBusy，之后进行额外一次检查，保证没有其他线程并发的创建了cell元素，如果没有，则创建cell对象并放入cells数组的对应位置上

    - 如果不是空

      - 如果wasContended为false代表在本线程执行cas之前，其他线程对此cell元素的cas操作失败，那么证明此cell存在大量竞争，那么对此线程来说，就应该**重新计算索引**，获取新的cell，同时会将此标志位置为true
      - 否则，证明此cell不存在大量竞争，则可以对此cell进行cas操作，由于传入的双目运算函数fn是`null`，所以直接进行简单加法并cas
      - 如果上一步cas失败，是判断当前cells数组的长度是否大于CPU的核心数，如果大于则**不再进行扩容**，否则进入下一步
      - 经过上一步，代表此时的cells数组大小已经出现了大量的线程竞争，如果这个线程是第一个发现此现象的，则将collide置为true，等到下一次循环或者被其他线程进行扩容操作
      - 经过前几步，代表此时需要对cells数组进行扩容，下面单独抽取这部分介绍

- 扩容

  ```java
  else if (cellsBusy == 0 && casCellsBusy()) {
      try {
          if (cells == as) {      // Expand table unless stale
              Cell[] rs = new Cell[n << 1];
              for (int i = 0; i < n; ++i)
                  rs[i] = as[i];
              cells = rs;
          }
      } finally {
          cellsBusy = 0;
      }
      collide = false;
      continue;                   // Retry with expanded table
  }
  ```

  这里同样会先检查占用被casCellsBusy，之后将cells数组扩容为两倍，在扩容结束后，将collide置为false

##### 3）总结

`LongAdder`类主要特性有如下几点

- **解决伪共享**

  由于cells数组会连续存在于缓存行，而每个线程是单独的对cell元素做操作，这里通过将Cell类标记为Contended来保证每个cell元素独占缓存行

- **原子性与高并发性**

  其原子性同样是通过CAS操作完成，但是却分散了线程的竞争，将其分布于各个cell上

- **CPU高效性**

  前面发现，当cells数组长度大于CPU核心数时就不再扩容，由于线程和CPU是一对一的关系，这样做让每个线程对每个cell操作时，可以保证cell和线程是一对一的关系，这样cell和CPU也就是一对一的关系，这样cell的cas操作会更高效

#### 2.2.3 `LongAccumulator`

##### 1）产生背景

由于LongAdder中不可以指定累加的规则，只能是传入一个被操作数且只能进行加法操作，为了能让累加的概念更加宽泛，且可以使用多个操作数（如双目运算函数），引入了LongAccumulator

> 其实和`LongAdder`用的是同一个接口方法，只不过`LongAdder`中传入指定运算函数的位置是`null`

##### 2）原理

###### （1）UML图

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/image-20201113152158609.png" alt="image-20201113152158609" style="zoom:67%;" />

###### （2）`accumulate`

```java
public void accumulate(long x) {
    Cell[] as; long b, v, r; int m; Cell a;
    if ((as = cells) != null ||
        (r = function.applyAsLong(b = base, x)) != b && !casBase(b, r)) {
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended =
              (r = function.applyAsLong(v = a.value, x)) == v ||
              a.cas(v, r)))
            longAccumulate(x, function, uncontended);
    }
}
```

这里只看一个函数，其大致逻辑和LongAdder是一致的，但是注意其进行累加操作时，使用的是成员变量function，且将其传入给`longAccumulate`函数，这样就也改变了其中的累加规则

此function对象可以使用下面构造函数赋予

```java
public LongAccumulator(LongBinaryOperator accumulatorFunction,
                       long identity) {
    this.function = accumulatorFunction;
    base = this.identity = identity;
}
```

这里需要注意`LongBinaryOperator`是一个接口，用户需要提供实现类并传入

##### 3）总结

`LongAccumulator`借鉴了`Arrays.sort`中传入自定义`Comparator`实现类的思想，通过引入`LongBinaryOperator`接口，提供了更灵活的累加方式，但是实际的cas操作和cells数组相关的操作仍然和`LongAdder`相同都是使用`Stripe64`中的方法来完成