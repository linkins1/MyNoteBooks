# 3.常见问题

## 3.1 volatile

为了保证内存可见性，可以使用MESI（缓存行锁）或者总线锁。被volatile修饰的字段会加上lock指令，其可以使用总线锁（古早），也可以使用MESI（现代）

### *A.可见性*

#### 3.1.1 总线锁

如果某个CPU发出lock，那么就会将总线**只赋予**给此CPU使用，从而达到内存可见性的目的

可见其封锁粒度大，将整个总线上锁

#### 3.1.2 MESI

##### 1）机制

为了解决总线锁封锁粒度大的问题，提出使用缓存锁，主流实现为MESI

- Modified

  当前CPU缓存中变量的值和主存中不一致

- Exclusive

  当前CPU缓存中的数据不出现在其他CPU缓存中，且是有效的

- SHARED

  当前CPU缓存和其他CPU缓存中的数据是一致的

- Invalid

  当前CPU中存储的数据是无效的，和其他CPU/主存中的数据不一致

MESI通过**总线嗅探机制**保证了内存的可见性，过程如下

假设变量a原来存储在CPU0和CPU1中，且状态为S，那么，当CPU0对变量a进行修改并同步到本地缓存后，

- CPU0会将其状态修改为M，之后，向其他所有CPU发出invalidate通知
- 当CPU1收到此通知后，需要返回一个ACK
- CPU0收到全部ACK后，再将缓存中修改的值同步到主存

从上可见，CPU0是**阻塞等待**其他CPU的ACK的，因而会降低执行效率，为了能让CPU0不阻塞等待ACK，因而引入了StoreBuffer

##### 2）StoreBuffer

同样是前面的例子，

- CPU0此时无需等待invalidate通知的ACK，而是将被修改的数据先写入StoreBuffer并发出相应的invalidate通知，之后就可以做其他事了
- CPU1也不会同步处理收到的invalide消息，而是将其先放入invalidate-queue中，之后异步处理queue中的消息，将对应的缓存状态变为I
- **当**StoreBuffer中的数据的ACK全部收到时，就可以通知CPU0将缓存中的数据写入主存了

现在假设一种情况，由于CPU1是异步处理invalidate-queue的，如果没有立即处理queue，且对要无效化的变量进行了读取，由于读取是不会处理invalidate-queue的，因而会产生**脏读**，也即这时，CPU0并未将更新的值Store入主存，CPU1就提前读取了旧值，就产生了一次**脏读**

### *B.指令重排*

#### 3.1.3 机制

除了上面提到的**脏读**情况，编译器有时为了优化执行会对编译后的代码进行重排序，以提高执行效率

因而提出使用内存屏障的方式来解决了StoreBuffer和InvalidQueue带来的脏读问题**以及**指令重排的问题，同时也保证了数据的可见性

| 类型       | 作用                                               |
| ---------- | -------------------------------------------------- |
| LOADLOAD   | 此指令后的LOAD指令必须晚于此指令前的LOAD指令执行   |
| LOADSTORE  | 此指令后的STORE指令必须晚于此指令前的LOAD指令执行  |
| STORELOAD  | 此指令后的LOAD指令必须晚于此指令前的STORE指令执行  |
| STORESTORE | 此指令后的STORE指令必须晚于此指令前的STORE指令执行 |

通过在代码中插入内存屏障指令，可以保证指令不会产生重排序，其中LOAD指令是从主存中读入到缓存，STORE是从缓存刷新到主存

对于volatile来说，实际使用的内存屏障如下

- 读

  在volatile域读操作之后加入LOADLOAD和LOADSTORE，这样保证了在之后的普通变量的读/写操作都位于此变量之后，由于volatile域的读操作是从主存中读取到最新的变量，这样如果后面普通变量的赋值使用到了volatile域，就可以保证数据最新

- 写

  在volatile域写操作之前加入STORESTORE，之后加入STORELOAD，这样保证了volatile域的写操作不会发生在前方的写操作之前，也不会发生在后方的读操作之后（或者说对后方的读操作可见），这样就保证了volatile域之前的普通变量赋值不会受到此volatile域的影响，且之后的可能和volatile域相关的赋值也可以使用到及时更新的值

此外，

对于写屏障来讲，在遇到此指令时，会强制StoreBuffer中的数据必须立即同步到主存中

对于读屏障来讲，在遇到此指令时，会强制InvalidQueue中的数据必须被立即处理

这样volatile域的读写就可保证在任意CPU上都是S状态，因为对于

- 写者来讲，其会受到STORE屏障限制，那么StoreBuffer中存储的volatile的信息就会被立即同步到主存
- 读者来讲，其会受到LOAD屏障限制，那么InvalidQueue中的消息就会被立即处理，那么缓存中存储的数据就会被失效化，并从主存中获取最新的值

#### 3.1.4 两个原则

- as-if-serial

  "好像是串行"，即单线程下指令重排序后和重排序前的执行结果是一致的

- happens-before

  指任一线程对volatile域的写操作一定**对**其他所有线程**后续的**读操作**可见**
  
  具体有如下规则
  
  - 程序执行顺序
  
    对于同一线程，每个操作happens-before任意后续操作
  
  - synchronized
  
    对监视器锁释放happens-before下一次上锁
  
  - volatile
  
    对volatile域的写happens-before下一次读
  
  - 传递性
  
    A早于B，B早于C，A早于C
  
  - start
  
    线程A中调用B.start，那么start之前的操作happens-before B中任意操作
  
  - join
  
    A中调用B.join，那么B中的任意操作happens-before A中join后的任意操作

### # 参考

[并发编程（七）volatile原理——解决可见性、有序性问题_语言日记-CSDN博客](https://blog.csdn.net/qq_38249409/article/details/112707231)

[高速缓存一致性协议MESI与内存屏障 - 小熊餐馆 - 博客园 (cnblogs.com)](https://www.cnblogs.com/xiaoxiongcanguan/p/13184801.html)

## 3.2 ThreadLocal内存泄漏

### 3.2.1 ThreadLocal强引用

对于ThreadLocal而言，其实现原理是将ThreadLocal对象作为key放入ThreadLocalMap中，如果ThreadLocal对象存在强引用，那么ThreadLocal对象就一直不会被回收，如下面这种情况

```java
public class ThreadLocalExp {
    static ThreadLocal<String> stringThreadLocal = new ThreadLocal<>();
    static Object object = new Object();
    public static void main(String[] args) {
         Thread threadA = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (object){
                    stringThreadLocal.set("Thread-A");
                }
            }
        });
    }
}
```

stringThreadLocal是静态变量，其引用的对象属于GC ROOT，那么在stringThreadLocal没有被置为null时，Map中的Entry对象的key就不会被回收，value同样不会被回收，这样就产生了内存泄漏

### 3.2.3 value内存泄漏

如果将stringThreadLocal置为null或者直接new一个匿名的ThreadLocal对象，由于在栈帧的局部变量表等处没有强引用，且此Entry中的key被标记为WeakReference，那么只会生存到下一次垃圾收集，这样**key会被成功回收**，对于这种情况，如果

- 使用线程池

  如果线程没有被回收，那么Entry中的value存在这样一条引用链，Thread->ThreadLocalMap->Entry->value，这样value就永远不能被回收，产生内存泄露

  对于这种情况，需要在每次使用完ThreadLocal后都调用remove方法清除ThreadLocalMap中的Entry

- 不使用线程池

  由于线程会消亡，那么其对应的栈会被回收，资源也就被释放了

ThreadLocal的实现中考虑到了这种情况，在每次调用get/set/remove方法时，会触发对key为null的value的回收，此外set时如果遇到key为null的情况，会将此位置的Entry替换

### 3.2.4 SimpleDateFormat的线程安全

有如下一段代码

```java
public class TimeTest {
    static SimpleDateFormat format = new SimpleDateFormat();
    public static void main(String[] args) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                String dateString = TimeTest.format.format(new Date());
                Date parse = null;
                try {
                    parse = format.parse(dateString);
                } catch (ParseException e) {
                    e.printStackTrace();
                }
                String dateStringCompare = TimeTest.format.format(parse);
            }
        });

    }
}
```

在将new Date传入format方法时，会执行如下代码段

```java
private StringBuffer format(Date date, StringBuffer toAppendTo,
                            FieldDelegate delegate) {
    // Convert input date to time field list
    calendar.setTime(date);
```

calendar是SimpleDateFormat的成员变量，这里调用setTime会赋值给Calendar的time属性，之后将time属性转换并赋值给Calendar的fields属性，之后在format的过程中会使用到此feilds属性，如果多线程都执行format函数，那么其中的fields属性是线程不安全的

因而应当将上面代码修改为

```java
public class TimeTest {
    static ThreadLocal<SimpleDateFormat> formatLocal = ThreadLocal.withInitial(SimpleDateFormat::new);

    public static void main(String[] args) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                SimpleDateFormat format = formatLocal.get();
                String dateString = format.format(new Date());
                Date parse = null;
                try {
                    parse = format.parse(dateString);
                } catch (ParseException e) {
                    e.printStackTrace();
                }
                String dateStringCompare = format.format(parse);
            }
        });

    }
}
```

这样可以在每次获取SimpleDateFormat对象时可以获取到当前线程的对象

### # 参考

[深入分析 ThreadLocal 内存泄漏问题](https://www.jianshu.com/p/1342a879f523)

## 3.3 线程安全

### 3.3.1 static/final

这两个关键字和线程安全并无关系，比如下面的单例模式

```java
public class SingletonFactory{
    private static final Object singleton;
    
    public static Object getSingleton(){
        if(singleton == null){					//1
            singleton = new Object();		//2
        }
        return singleton;						//3
    }
}
```

假设执行过程中，线程1先执行到代码//1位置，判断singleton为null，之后进入//2；这时切换到线程2执行，此时singleton仍然为null，因而进入代码//2；这样就发生了线程安全问题，因为两个线程**返回的对象不同**

***<u>改进1</u>***

很容易想到加锁的方式保证线程安全，如下所示

```java
public class SingletonFactory {
    private static final Object singleton;

    public static Object getSingleton() {
        if (singleton == null) {//1
            synchronized (SingletonFactory.class) {
                if (singleton == null) {
                    singleton = new Object();//2
                }
            }
        }
        return singleton;//3
    }
}
```

这样便保证了线程安全，但仍然存在问题，因为new操作实际是分三步进行的

```text
1.分配内存//malloc
2.在分配好的内存处进行对象初始化//调用构造函数等
3.将对象引用指向初始化好的内存地址//引用赋值
```

从上可以看出，当singleton没有被volatile修饰时，可能会发生指令重排序，如果3排到2之前，那么有可能发生如下情况：

线程1执行到代码//2，且发生乱序，3排到2之前，这时发生调度，线程2执行代码//1，判断对象非空，因而返回一个未初始化的对象

***<u>改进2</u>***

`private static volatile Object singleton;`

这样即可保证对象new的操作不会发生指令重排序

> 注意volatile和final不可以同时出现，由于volatile代表在多线程下的修改互相可见，final代表只可以初始化不可以修改，两者语义上是冲突的

#### 3.3.2 Bean是线程安全的吗

Spring中的Bean有五种作用域

- Singleton

  单例对象，如果此对象是无状态的，那么就不存在线程安全问题，否则会出现线程安全问题

- ProtoType
  每次获取时都会创建，此对象是线程安全的

- request

  每次http请求创建一个对象

- session

  每个会话使用一个对象

- global-session

  所有会话使用一个对象

#### 3.3.3 System.out.println()是线程安全的吗

##### 1）synchronized

```java
public void println(int x) {
    if (getClass() == PrintStream.class) {
        writeln(String.valueOf(x));
    } else {
        synchronized (this) {
            print(x);
            newLine();
        }
    }
}
```

```java
private void writeln(String s) {
    try {
        synchronized (this) {
            ensureOpen();
            textOut.write(s);
            textOut.newLine();
            textOut.flushBuffer();
            charOut.flushBuffer();
            if (autoFlush)
                out.flush();
        }
    }
    catch (InterruptedIOException x) {
        Thread.currentThread().interrupt();
    }
    catch (IOException x) {
        trouble = true;
    }
}
```

从这里可以看出，在打印单个对象、数字、字符(串)时，其是线程安全的，因为有synchronized关键字

由于synchronized关键字有锁粗化和锁消除两个优化，因而在使用sout时，会对代码产生一定影响

##### 2）JIT的优化

JIT将字节码编译为机器码时，会对代码进行分析，将一些冗余的上锁、解锁操作进行消除和减少，也就是所谓的锁粗化和锁消除

###### （1）锁消除

- 逃逸分析

  对于一个方法内所引用的对象，如果此对象在方法外也被引用，也即在方法退出时，在别的方法内被引用，那么此对象就是**"逃逸对象"**，因为在方法结束时，此对象不能被GC回收掉

- 锁消除

  根据逃逸分析，如果方法中被临界区中使用的变量都不是**逃逸对象**，那么就可以将锁消除，也即如下情况

  ```java
  public void add(){
      synchronized(this){
          StringBuilder builder = new StringBuilder();
          builder.append('1');
      }
  }
  ```
  

在多线程调用此方法的情况下，由于builder对象在临界区中每次都是被新建得到，因而不会出现线程安全的问题，builder对象的**作用域仅局限于**add方法内，因而可以将锁消除，变为如下情况

```java
  public void add(){
      StringBuilder builder = new StringBuilder();
      builder.append('1');
  }
```

###### （2）锁粗化

假设对于下面一段代码

```java
public void add(){
    for(int i=0;i<10;i++){
        System.out.println(i);
    }
}
```

由于sout被synchronized修饰，那么在每一次执行循环体时，就会触发一次上锁和解锁操作，JIT在对这段代码进行编译时，会将频繁的上锁、解锁操作减少，相当于变为如下情况

```java
public void add(){
    synchronized(this){
        for(int i=0;i<10;i++){
            System.out.println(i);
        }
    }
}
```

这样就将上锁、解锁操作减少为一次

> 此外，这里还要注意到，由于synchronized的存在，每次进入代码块会从主存中同步数据，退出时将修改的数据同步回主存

