# 2.高级篇

### 2.4 并发容器

### :one: List

#### 2.4.1`CopyOnWriteArrayList`

##### 1） UML图

<img src="C:%5CUsers%5C123%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20201115140908269.png" alt="image-20201115140908269" style="zoom: 67%;" />

此类实现了如下**接口**

- `Cloneable`

  支持clone操作，提供的是浅拷贝

- `RandomAccess`

  支持随机访问，隐喻底层是数组实现

其主要字段如下

- `array`

  其是一个Object的数组，且被**volatile**关键字修饰，保证了内存可见性

- `lock`

  此成员变量被final修饰，是`ReentrantLock`的实例，其是一个可重入的独享锁

- `lockOffset`

  lock变量在此类中的偏移量

- `UNSAFE`

  用于设定lock变量状态的unsafe对象，一般在克隆对象后来重置锁的状态

##### 2） 示例

```java
public class COWExample {

    public static void main(String[] args) {
        CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
        list.add("my");
        list.add("name");
        list.add("is");
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                list.add("haha");
                list.remove(1);
                list.set(1, "ok");
            }
        });

        Iterator<String> iterator = list.iterator();
        t1.start();
        while(iterator.hasNext()){
            System.out.println(iterator.next());
        }
        System.out.println("---------- after changing --------------");
        for (String s : list) {
            System.out.println(s);
        }
    }
}
```

输出结果为

```shell
my
name
is
---------- after changing --------------
my
ok
haha
```

下面逐步讲解例子中的方法

##### 3） 基本操作

###### （1）add

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

步骤如下

- 获取lock并尝试上锁，如果成功，其他线程将被阻塞直到锁被释放
- 调用getArray方法获取`array`对象，并将引用赋给`elements`
- 将`elements`拷贝到一个长度+1的新数组`newElements`，并将新添加的元素放入新数组的末尾
- 调用setArray方法将新数组赋给`array`对象
- 最终释放锁

可见，这里的添加操作是**对原数组长度+1的拷贝**进行操作的，在添加成功之前`array`对象保持不变

###### （2）set

```java
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);

        if (oldValue != element) {
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len);
            newElements[index] = element;
            setArray(newElements);
        } else {
            // Not quite a no-op; ensures volatile write semantics
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

步骤如下

- 获取lock对象并上锁

- 调用getArray方法获取`array`对象，并将引用赋给`elements`

- 调用get方法返回要更新的索引位置的旧元素并赋给`oldValue`，如果index越界则抛出越界异常

- 比较`oldValue`和要设定的`element`，如果

  - 不相等

    拷贝`elements`到新数组`newElements`，并对`newElements`的index位置做更新操作，最终调用setArray方法将新数组赋给`array`对象

  - 相等

    直接调用setArray方法，尽管前后的对象引用相同，但这保证了volatile的内存语义

- 返回`oldValue`并释放锁

可见，这里同样是**对原数组的拷贝**进行操作的，在更新完成之前，`array`对象保持不变

###### （3）remove

```java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

步骤如下

- 获取lock对象并上锁

- 调用getArray方法获取`array`对象，并将引用赋给`elements`

- 调用get方法返回要更新的索引位置的元素并赋给`oldValue`，如果index越界则抛出越界异常

- 计算出删除index后，index之后的数组长度为`numMoved`

- 如果`numMoved`

  - 是0

    那么代表index等于len-1，也即要删除的是最后一个元素，那么只需要将原数组从`0->len-2`的数组元素拷贝到新数组中，并设定给`array`对象即可

  - 不是0

    那么代表删除的元素处于数组中央，那么先创建一个新数组`newElements`，长度为旧数组长度-1，然后分两部分进行拷贝，最终将`newElements`设定给`array`对象即可

- 返回被删除索引位置原来的元素并释放锁

可见，删除操作也是**将原数组中除了删除位置的元素拷贝到新数组**中来完成，删除成功前`array`对象保持不变

###### （4）iterator

**:one:`COWIterator`**

此类的迭代器是使用其静态内部类`COWIterator`来完成的，其UML如下

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/image-20201115144401024.png" alt="image-20201115144401024" style="zoom: 50%;" />

其中的元素

- `snapshot`

  其是一个final的Object数组，用于存储获取迭代器时的`array`对象

- `cursor`

  遍历时使用的索引

其构造器如下

```java
private COWIterator(Object[] elements, int initialCursor) {
    cursor = initialCursor;
    snapshot = elements;
}
```

**:two: `iterator()`**

```java
public Iterator<E> iterator() {
    return new COWIterator<E>(getArray(), 0);
}
```

这里首先调用getArray方法获取到`array`对象然后赋值给`snapshot`，并将索引初始化为0

**注意！**获取迭代器不需要进行上锁，也即其获取到的array对象是当前的array对象，且在使用期间，array对象保持不变

##### 4） 弱一致性

从上面可以看出，

- 凡是**写操作**

  包括增删改，都是先拿到array对象的拷贝后对拷贝进行操作，在拷贝的期间，array对象保持不变，在写操作结束后，再将拷贝赋值给array

- 凡是**读操作**

  都是直接获取当前的array对象，并在使用期间array对象不会改变

那么这是就出现了弱一致性的问题，如果在读的期间进行了写操作，那么这个写操作对读操作是不可见的，因为是针对的不同的对象；同样的，在写操作期间，如果执行读，那么写操作的结果同样不会反映到此时的读操作中

但是，在写操作完成之后，进行读操作是可以获取到写的结果的，因为此时array对象已经发生了改变，这便是**弱一致性**

这种特性带来的**好处**是

- 在进行大量读操作时可以不用获取锁，提高了并发读的效率
- 在读的期间进行写不会影响到读，尽管此时读的数据不是最新的，但是能保证最终是最新的

带来的**坏处**是

- 写的结果不能及时的反应在读上

### :two: NonBlockingQueue

#### 2.4.2 `ConcurrentLinkedQueue`

##### 1）UML

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/ConcurrentLinkedQueue1.png" alt="ConcurrentLinkedQueue" style="zoom: 80%;" />

从UML中可以看出，此类继承于AbstractQueue，其中包含下面重要属性

- `head`

  用于指向链表的头部

- `headOffset`

  head的偏移量

- `tail`

  用于指向链表的尾部

- `tailOffset`

  tail的偏移量

##### 2）关键方法

###### （1）插入

正常情况下，会一直**尾插**到链表尾部，期间保持head指向哨兵节点，tail指向新的尾部节点
插入过程中如果碰到**自引用**情况，也即链表被其他线程删除干净的情况，则需要重新找到head和tail再进行尾插

###### （2）删除

每次删除时，将哨兵节点的后一个点哨兵化，并让head指向新哨兵，让原来的head**自引用**，等待被回收

删除过程中如果出现**自引用**的情况 那么证明链表已经被其他线程删干净了 那么只需要重新寻找到head即可

##### 3）总结

- 无界非阻塞单向链表
- CAS保证原子性

### :three: BlockingQueue

> 下面的三个类都实现了`BlockingQueue`接口，也即都属于阻塞队列

#### 2.4.3 `ArrayBlockingQueue`

> 本质解决的是***生产者消费者***问题

##### 1）UML

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/ArrayBlockingQueue.png" alt="ArrayBlockingQueue" style="zoom:67%;" />

- `items`

  Object数组，用于存放数据的底层数组

- `takeIndex`

  取数据时使用的索引

- `putIndex`

  存数据时使用的索引

- `count`

  int类型，记录元素个数

- `lock`

  可重入锁，在插入和删除之前都需要调用此对象

- `notEmpty`

  lock的一个条件变量

- `notFull`

  lock的另一个条件变量

##### 2）关键方法

###### （1）尾插&头删

- 在对队列进行多线程删除操作时，需要使用lock先对数组上锁，

  - 如果上锁后发现数组**为空**（count==0），

    - 如果是阻塞方法，如`take`

      那么调用notEmpty.await，等待生产者唤醒

    - 如果是非阻塞方法，如`poll`、`peek`

      那么直接返回null

  - 否则，调用`dequeue`方法直接删除，同时让takeIndex++，count--，并调用notFull.signal唤醒生产者线程

  最终释放lock

- 在对队列进行多线程插入操作时，需要使用Lock先对数组上锁，

  - 如果上锁后发现队列**为满**（count==items.length），

    - 如果是阻塞方法，如`put`

      那么调用notFull.await，等待消费者唤醒

    - 如果是非阻塞方法，如`offer`

      那么直接返回false

  - 否则，调用`enqueue`方法直接插入，同时让putIndex++，count++，并调用notEmpty.signal唤醒消费者线程

  最终释放putLock

###### （2）删除&查询

在调用remove方法删除队列中第一个匹配的点时，需要用lock上锁，这样才能保证遍历期间，数组内容以及两个索引不会发生改动，遍历时从takeIndex开始遍历，一直到putIndex为止，如果takeIndex大于putIndex，那么循环遍历

调用contains方法同理，也需要用lock上锁，才可以保证查询期间数组内容以及两个索引没有发生变化

###### （3）总结

- 循环操作

  从上面可知，在插入和删除时，要操作的位置是由takeIndex和putIndex给定的，那么当putIndex抵达数组末尾，且takeIndex不为0（头部取出过一些点）时，putIndex会重新设定为0继续插入直到putIndex遇到takeIndex为止；takeIndex同理，其会一直取数据，如果在遇到putIndex之前抵达了数组末尾，同样会设定为0，一直遍历到putIndex为止

- 不变数组

  在一开始就会给数组设定好容量，其中变化的是putIndex和takeIndex，这样省去了对数组resize的开销

- 一把可重入锁

  仅仅使用一个可重入锁和两个条件变量来保证同步

##### 3）总结

- 有界阻塞队列
- 使用可重入锁来实现原子性
- 使用条件变量来保证操作最终会被执行

#### 2.4.4 `LinkedBlockingQueue`

> 本质是解决了***生产者消费者***问题

##### 1）UML

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/LinkedBlockingQueue.png" alt="LinkedBlockingQueue" style="zoom: 67%;" />

- `capacity`

  指定的队列容量，如果未指定则默认`Integer.MAX_VALUE`

- `count`

  AtomicInteger类型，用于记录队列中元素的个数

- `head`

  队列的头

- `last`

  队列的尾

- `takeLock`

  取元素时用的可重入锁

- `notEmpty`

  takeLock对应的条件变量

- `putLock`

  加入元素时用的可重入锁

- `notFull`

  putLock对应的条件变量

从上面可知，此类中自定义了Node类，且给出了一对锁和对应的一对条件变量，用锁来完成对阻塞操作的支持

##### 2）关键方法

###### （1）尾插&头删

- 在对队列进行多线程删除操作时，需要使用takeLock先对队列上锁，

  - 如果上锁后发现队列**为空**，

    - 如果是阻塞方法，如`take`

      那么调用notEmpty.await，等待生产者唤醒，同时调用signalNotFull方法，先获得putLock再调用notFull.signalAll唤醒所有生产者

    - 如果是非阻塞方法，如`poll`、`peek`

      那么直接返回null

  - 否则，直接删除，如果删除过后队列仍然**不为空**，那么调用notEmpty.signal唤醒其他消费者

  最终释放takeLock

- 在对队列进行多线程插入操作时，需要使用putLock先对队列上锁，

  - 如果上锁后发现队列**为满**，

    - 如果是阻塞方法，如`put`

      那么调用notFull.await，等待消费者唤醒，同时调用signalNotEmpty方法，先获得takeLock再调用notEmpty.signalAll唤醒所有消费者

    - 如果是非阻塞方法，如`offer`

      那么直接返回false

  - 否则，直接插入，如果插入过后队列仍然**不为满**，那么调用notFull.signal唤醒其他生产者

  最终释放putLock

###### （2）删除&查询

在调用remove方法删除队列中第一个匹配的点时，需要对两把锁都上锁，这样才能保证遍历期间，链表结构不会发生改动

调用contains方法同理，也需要对两把锁都上锁，才可以保证查询期间链表结构没有发生变化

###### （3）总结

- 哨兵

  此队列也使用哨兵节点，也即起初时，head和last都指向item为null的Node，且在头删时，也是对head的下一个点哨兵化，并让head指向这个点，而tail则是一直指向尾部节点

- count

  由于可能同时存在两个线程分别执行尾插和头删操作，因而count需要使用AtomicLong，用CAS来保证计数值正确

- 阻塞/非阻塞

  此类属于阻塞队列的含义是生产者之间互相阻塞，消费者之间互相阻塞，因为需要拿到锁才可以操作队列

  而上面提到的阻塞和非阻塞方法是指拿到锁后，在队列为空或满时是否要await进入条件阻塞队列

- 中断

  插入和删除操作有些会抛出中断异常，这是因为调用了lock.lockInterruptibly()，而不抛出中断异常的调用的是lock.lock()，底层分别对应acquireInterruptibly和acquire

##### 3）总结

- 有界阻塞队列
- 使用可重入锁来实现原子性
- 使用条件变量来保证操作最终会被执行

##### 4）与ArrayBlockingQueue的区别

两者都是解决了生产者消费者问题，但是有如下区别

- 锁的个数

  ArrayBlockinigQueue是用一个可重入锁来保证同步，而此类使用两把锁来保证同步，这么做的影响是

  - ArrayBlockingQueue同时只有一个线程在操作数组，而此类同时会有两个线程也即一个生产者一个消费者同时操作队列

  - ArrayBlockingQueue为了能保证队列为空/满时支持阻塞等待，其是使用同一个lock对象创建了两个条件变量来完成阻塞操作，而此类中的条件变量分别只和对应的锁相关联，这样在signal对方时必须先拿到锁才可以

- 底层结构

  ArrayBlockingQueue使用固定大小的数组来完成，而此类使用链表

#### 2.4.5 `PriorityBlockingQueue`

##### 1）UML

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/concurrent/PriorityBlockingQueue.png" alt="PriorityBlockingQueue" style="zoom:67%;" />

- `queue`

  Object数组，用于存储二叉堆

- `size`

  元素数量

- `lock`

  可重入锁，在操作数组前需要获取锁

- `notEmpty`

  lock的条件变量，消费者消费时，如果队列为空，那么就进入条件阻塞队列

- `allocationSpinLock`

  volatile int类型，在对数组结构进行改变（扩容）时，需要CAS设定其为1，并在改变后设定为0

- `UNSAFE`

  用于操作`allocationSpinLock`

- `q`

  PriorityQueue对象，在序列化和反序列化时使用

- `DEFAULT_INITIAL_CAPACITY`

  默认数组大小为11

- `MAX_ARRAY_SIZE`

  最大数组大小为Integer.MAX_VALUE - 8

- `comparator`

  在插入到数组中的元素没有实现Comparable接口时，如果给定了Comparator的实现类，那么将用此实现类来完成比较

##### 2）关键方法

###### （1）插入

```java
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    lock.lock();
    int n, cap;
    Object[] array;
    while ((n = size) >= (cap = (array = queue).length))
        tryGrow(array, cap);
    try {
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            siftUpComparable(n, e, array);
        else
            siftUpUsingComparator(n, e, array, cmp);
        size = n + 1;
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
    return true;
}
```

此方法中会执行如下步骤

- 检查数组是否越界，如果是，则调用tryGrow进行扩容
- 扩容后，如果comparator是空，则调用数组内元素的compareTo方法做比较，否则调用Comparator的compare方法作比较
- 在选定比较方法后，调用`siftUpxxx`对元素进行位置确定，由于要构建二叉堆，如果是
  - 最大堆，那么插入的元素会一直和其父节点（索引-1/2的位置）相比较，如果大于，则交换位置，一直这样比较直到小于父节点或者称为根节点（索引为0）
  - 最小堆，那么和最大堆的区别只是根节点为最小值

- 在插入完成后，将size增1，并调用`notEmpty.signal()`方法唤醒条件队列中阻塞等待的消费者
- 最终释放锁

###### （2）扩容

```java
private void tryGrow(Object[] array, int oldCap) {
    lock.unlock(); // must release and then re-acquire main lock
    Object[] newArray = null;
    if (allocationSpinLock == 0 &&
        UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                 0, 1)) {
        try {
            int newCap = oldCap + ((oldCap < 64) ?
                                   (oldCap + 2) : // grow faster if small
                                   (oldCap >> 1));
            if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                int minCap = oldCap + 1;
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                    throw new OutOfMemoryError();
                newCap = MAX_ARRAY_SIZE;
            }
            if (newCap > oldCap && queue == array)
                newArray = new Object[newCap];
        } finally {
            allocationSpinLock = 0;
        }
    }
    if (newArray == null) // back off if another thread is allocating
        Thread.yield();
    lock.lock();
    if (newArray != null && queue == array) {
        queue = newArray;
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}
```

方法步骤如下

- 释放lock，这样可能会有多个线程同时扩容，将newArray设定为null

- 判断allocationSpinLock是否为0，如果是，则CAS为1，CAS成功
  - 在设定newCap时，如果oldCap<64，那么newCap = 2*oldCap+2，否则newCap=1.5oldCap
  - 判断newCap是否溢出，如果是则判断oldCap+1是否溢出，如果是则抛出OutOfMemoryError异常，否则设定newCap为Integer.MAX_VALUE - 8
  - 如果newCap>oldCap且queue没有被修改过，那么newArray = new Object[newCap]
  - 将allocationSpinLock设定为0，此处不用CAS是因为只有一个线程可以CAS成功将allocationSpinLock设定为1，且其是volatile的，保证了内存可见性
- 如果CAS失败，代表有其他线程也在执行扩容操作，此时newArray是null，调用yeild让出CPU，让正在扩容的线程进行扩容
- lock上锁
- 将旧数组复制到新的数组中

###### （3）删除

:one: `poll`

```java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

此方法会直接调用dequeue方法，在元素个数为0时，会返回null，否则会删除索引为0的位置，也即根节点，如果是

- 最大堆，那么会将最后一个元素覆盖到根节点，并和其两个子节点中较大的比较，如果小于较大的，那么交换两者，并继续比较直到该点大于任意一个子节点
- 最小堆，和最大堆的区别是根节点是最小值，比较的逻辑相反

:two: `take`

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {
        while ( (result = dequeue()) == null)
            notEmpty.await();
    } finally {
        lock.unlock();
    }
    return result;
}
```

此方法也为删除头节点，但是不同在于当队列为空时，该线程会调用`notEmpty.await()`进入阻塞队列，等待生产者向队列中添加元素后被唤醒

##### 3）总结

- 无界阻塞优先级队列
- 使用CAS方式拿到自旋锁保证扩容安全
- 使用可重入锁+条件变量方式保证添加和删除元素的安全

> ***Q：如何理解无界？***
>
> ***A：***这里指的无界是指相较于前两种阻塞队列，**其没有capacity的限制**，因为capacity可以设定为小于Integer.MAX_VALUE的值，然而优先级阻塞队列仍然是由上界的，但是Integer.MAX_VALUE足够大，实际应用中足够代表无界。

### :four: Map 

#### 2.4.7 `ConcurrentSkipListMap`

参见[JavaGuide的ConcurrentSkipListMap部分]([JavaGuide (gitee.io)](https://snailclimb.gitee.io/javaguide/#/docs/java/multi-thread/并发容器总结?id=六-concurrentskiplistmap))