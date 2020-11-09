## 2.自动内存管理

### 2.1运行时数据区域

#### 2.1.1架构图

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200906143336489.png" alt="image-20200906143336489" style="zoom:80%;" />

从上图可以看出，JVM从主机内存中划分出一块内存，并称作运行时数据区域(**Runtime Data Areas**)，其包括以下几部分

- 程序计数器(PC Registers)
- Java虚拟机栈(Stack Area)
- 本地方法栈(Native Method Stack)
- Java堆区(Heap Area)
- 方法区(Method Area)

其中方法区和堆区为所有**线程共享的**数据区域

JDK1.8之后，架构图如下

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200910135307560.png" alt="image-20200910135307560" style="zoom: 67%;" />

主要的区别是

- 用元空间实现方法区，其中包含的数据包括运行时常量池，类和方法的元信息以及JIT编译产物
- 将字符串常量池和静态变量拉入堆区

#### 2.1.2程序计数器(PC Registers)

##### 1）定义

程序计数器（Program Counter Register）是一块**较小的内存空间**，它可以看作是**当前线程**所执行的**字节码的行号指示器**。字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，它是程序控制流的指示器。分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。

##### 2）线程独有

由于Java程序在运行时，每个线程会运行在单独的内核上，为了保证每个线程的执行顺序正确，需要给每个线程都分配一个**独立的程序计数器**来完成程序的流的正确标记。

##### 3）存储内容

- 对于Java方法来说

  计数器记录的是**正在执行的虚拟机字节码*指令*的**地址；

- 对于本地方法来说

  计数器值则应为空（Undefined）

##### 4）相关异常

此部分内存是**<u>唯一一个</u>**不会发生OutOfMemoryError情况的区域

#### 2.1.3Java虚拟机栈(Stack Area)

##### 1）定义

**虚拟机栈**描述的是Java方法执行的线程内存模型，其中保存的内容是：每个方法被执行的时候，Java虚拟机都会同步创建一个栈帧（Stack Frame）用于存储**局部变量表、操作数栈、动态连接、方法出口**等信息。每一个**方法**被**调用**直至**执行完毕**的过程，就对应着一个**栈帧**在**虚拟机栈中**从**入栈**到**出栈**的过程。

##### 2）线程独有

虚拟机栈是线程独有的，其生命周期和线程一致

##### 3）存储内容

一般在讨论虚拟机栈时，往往会更关注于**局部变量表**，局部变量表的相关信息如下

###### （1）包含的内容

- 编译器可知的基本数据类型变量
- 引用类型的变量
- 返回地址，指向了一条字节码指令的地址

###### （2）存储结构

上面所提到的三种数据类型在局部变量表中以局部变量槽(slot)的形式存储，其中long和double型会占两个slots

###### （3）内存分配

在编译器时就固定分配好slot的数量(固定的)，但是每个slot具体多大，需要看虚拟机的配置

##### 4）相关异常

###### （1）StackOverflowError

当虚拟机栈容量**不可以动态扩展**且当线程请求的栈深度**大于虚拟机规定的大小**时，会抛出栈溢出异常

###### （2）OutOfMemoryError

当虚拟机栈容量**可以动态扩展**且当栈扩展时**无法申请到足够的内存资源**时，会抛出内存耗尽异常

###### ------ SOF&&OOM举例 ------

- 测试例1--减少栈内存容量到128k

```java
/**
* VM Args：-Xss128k
* @author zzm
*/
public class JavaVMStackSOF {
    private int stackLength = 1;
    public void stackLeak() {
    stackLength++;
    stackLeak();
    }
    public static void main(String[] args) throws Throwable {
    JavaVMStackSOF oom = new JavaVMStackSOF();
        try {
        oom.stackLeak();
        } catch (Throwable e) {
        System.out.println("stack length:" + oom.stackLength);
        throw e;
        }
    }
}
```

运行结果

```text
stack length:2402
Exception in thread "main" java.lang.StackOverflowError
at org.fenixsoft.oom. JavaVMStackSOF.leak(JavaVMStackSOF.java:20)
at org.fenixsoft.oom. JavaVMStackSOF.leak(JavaVMStackSOF.java:21)
at org.fenixsoft.oom. JavaVMStackSOF.leak(JavaVMStackSOF.java:21)
```

> 对于不同的操作系统，都会给出最少的栈内存大小，如果分配的大小小于最小值，那么会报错

- 测试例2--定义了大量的本地变量，增大此方法帧中本地变量表的长度

  这种情况对于是否支持栈容量动态扩展的虚拟机来说会出现不同的错误，支持动态扩展的会报OOM错误，否则报SOF错误

- 测试例3--创建大量线程耗尽**直接内存**

```java
/**
* VM Args：-Xss2M （这时候不妨设大些，请在32位系统下运行）
* @author zzm
*/
public class JavaVMStackOOM {
    private void dontStop() {
        while (true) {
        }
    }
    public void stackLeakByThread() {
        while (true) {
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                dontStop();
                }
            });
            thread.start();
        }
    }
    public static void main(String[] args) throws Throwable {
        JavaVMStackOOM oom = new JavaVMStackOOM();
        oom.stackLeakByThread();
    }
}
```

运行结果

```text
Exception in thread "main" java.lang.OutOfMemoryError: unable to create native thread
```

这样产生的内存溢出异常和栈空间是否足够没有关系，在错误提示信息中也**没有指出异常发生在哪个内存空间上**，主要取决于操作系统本身的内存使用状态。在这种情况下，给**每个线程的栈分配的内存越大**，**越容易产生内存溢出异常**，那么在**不能减少线程数量或者更换64位虚拟机的情况下**，就只能通过**减少为每个线程分配的最大堆和栈容量来换取更多的线程**。	


#### 2.1.4本地方法栈(Native Method Stack)

本地方法栈与虚拟机栈很类似，只是服务对象不同(虚拟机栈服务于Java方法的字节码，本地方法栈服务于本地方法)，其**性质和虚拟机栈一致**，也同样会出现**同样的异常**

>对于HotSpot虚拟机，其将本地方法栈和虚拟机栈合二为一。

#### 2.1.5Java堆(Java Heap)

##### 1）定义

Java堆是虚拟机所管理的**内存中最大的**一块，在虚拟机启动时创建。其囊括了**几乎**(起因于逃逸分析技术)所有的**对象实例**以及**数组**。堆由垃圾收集器(Garbage Collector)管理，因而也称作GC堆。

##### 2）线程共有&&独有

Java堆的空间由所有的线程共享。但是为了更好更快的为对象实例分配内存，往往会将堆空间划分出多个线程**私有的**分配缓冲区(Thread Local Allocation Buffer，**TLAB**)。

##### 3）存储内容

用于存储对象实例和数组。此外，Java堆可以处于物理上不连续的内存上，但是必须保证逻辑上是连续的(类似于C语言中的malloc)，但是对于数组类型，为了方便管理，通常会连续的分配内存。

##### 4）相关异常

如果在Java堆中**没有内存完成实例分配**，并且**堆也无法再扩展时**，Java虚拟机将会抛出**OutOfMemoryError**异常

>Java堆既可以被实现成固定大小的，也可以是可扩展的，不过当前主流的Java虚拟机都是按照可扩
>展来实现的，通过参数-Xmx和-Xms设定

###### ------ OOM举例 ------

- 测试代码

```java
/**
* VM Args：-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
* @author zzm
*/
public class HeapOOM {
    static class OOMObject {
    }
    public static void main(String[] args) {
    List<OOMObject> list = new ArrayList<OOMObject>();
        while (true) {
        list.add(new OOMObject());
        }
    }
}
```

上面规定了JVM的内存最大为20MB且不可扩展

- 现象

```text
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid3404.hprof ...
Heap dump file created [22045981 bytes in 0.663 secs]
```

从报错信息看出OOM发生于**Java heap space**，此时仍需进一步判断是内存泄漏(Memory Leak)还是内存溢出(Memory Overflow)

##### 5）划分方式

由于GC工作时会对不同属性（是否消亡，何时创建）的对象进行区分对待，因而得到如下格式的内存结构

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200910134457116.png" alt="image-20200910134457116" style="zoom: 67%;" />

其中年轻代/新生代和老年代同属于Java堆区，而元空间替代了过去方法区的位置。

其中年轻代又分为伊甸区(Eden)和两个幸存者区(Survivor)，分别为To Survivor、 From Survivor，这个两个区域的空间大小是一样的。在[标记-复制算法]()中，采用一个伊甸区+From Survivor区作为实际被占用的新生代内存区域。执行垃圾回收的时候Eden区域不能被回收的对象被放入到空的survivor（也就是To Survivor，同时Eden区域的内存会在垃圾回收的过程中全部释放），另一个survivor（即From Survivor）里不能被回收的对象也会被放入这个survivor（即To Survivor），然后To Survivor 和 From Survivor的标记会互换，始终保证一个survivor是空的，如果另一块Survivor区不足以放下仍然存活的对象，则采用[分配担保机制]()占用一部分老年代的内存将对象复制到其中。

执行流程如下：

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200910141713815.png" alt="image-20200910141713815" style="zoom:67%;" />

:one:将对象分配到Eden区

:two:进行一次GC，将Eden中仍存活的对象放到From Survivor

:three:进行第二次GC，将Eden和From Survivor中仍存活的对象复制到ToSurvivor，如果ToSurvivor已满，采用分配担保机制复制到老年代

:four:交换两个Survivor的标签

> HotSpot中，Eden区和Survivor区比例为**8:1** 

#### 2.1.6方法区(Method Area)

##### 1）定义

**方法区**用于存储**已被虚拟机加载**的**类型信息、常量、静态变量、即时编译器编译后的代码缓存**等数据

> - 在JDK7之前，方法区采用永久代(Permanent Generation)的形式实现，此时永久代逻辑上属于堆内存，但是称作"非堆"，这样GC也可以参与"非堆"的垃圾回收，但是由于受限于堆内存容易产生OOM问题
>
> - 在JDK8之后，采用元空间(Meta Space)来替换永久代，元空间采用本地内存来完成，更不容易出现OOM异常
>
> HotSpot更新历史（逐步采用**本地内存**来实现方法区）
>
> - JDK 7的HotSpot，把原本放在永久代的**字符串常量池、静态变量等移出(移入到堆中)**
> - JDK 7的HotSpot，采用元空间，并把JDK 7中永久代还**剩余的内容（主要是类型信息）全部移到元空间中**

##### 2）线程共有

方法区的空间同样是线程共享的

##### 3）存储内容

存储**已被虚拟机加载**的类型信息、常量、静态变量、即时编译器编译后的代码缓存等数据

##### 4）相关异常

在使用永久代来实现方法区时，可能会因频繁创建动态代理对象而造成的OOM。使用元空间同样可能会因为无法申请到内存而导致OOM

##### 5）深入理解

:point_right:在JDK1.8之后，方法区可以理解为JVM规范中给出的一种概念，具体实施时采用了**元空间**+**堆的一部分**的方式来实现，其中

:one:元空间包含**class文件常量池**包含编译期生成的各种**字面量(Literal)与符号引用(Symbolic References)**。

- 字面量

  一般指**文本字符串**和**final修饰的常量**，以及类元信息和方法元信息，譬如Integer类中的内部类IntegerCache中的变量static final Integer[] cache，就是用于存储-128~127的整形变量，当定义了Integer i = 1；就会在常量池中获取这个对象。

- 符号引用

  - 被模块导出或者开放的包（Package）
  - 类和接口的全限定名（Fully Qualified Name）
  - 字段的名称和描述符（Descriptor）
  - 方法的名称和描述符
  - 方法句柄和方法类型（Method Handle、Method Type、Invoke Dynamic）
  - 动态调用点和动态常量（Dynamically-Computed Call Site、Dynamically-Computed Constant）

:two:堆中和方法区相关的只有**字符串常量池**和**静态变量**（字符串常量池和静态变量**逻辑上**属于方法区，但实际存在于堆）

> 对于String s = "abc"; "abc"这个值会存放在字符串常量池中，而s这个引用会存放在Java虚拟机栈中。而"s"这个字段名称会存储在class文件常量池中(位于元空间)，"s"在常量池中的索引会存在class文件的字段表中。
>
> 对于final static int m=123，由于被final修饰，123会存在class文件常量池中(位于元空间)，"m"这个字段名称也会存在class文件常量池中，"m"在常量池中的索引会存在class文件的字段表中。

#### 2.1.7运行时常量池(Runtime Constant Pool)

##### 1）定义

属于方法区的一部分，是线程共享的。其中的内容可以在编译期生成，如果是JDK7，还包括由运行期(String的intern()方法)生成的字符串常量池。

##### 2）存储内容

常量池表（Constant Pool Table）在类加载后存放到方法区的运行时常量池中

> 常量池表用于存放编译期生成的各种字面量与符号引用，其和类的版本、字段、方法、接口等描述信息一同存储于class文件

##### 3）相关异常

当常量池无法再申请到内存时会抛出OutOfMemoryError异常。

###### ------ OOM举例 ------

> 借助String的intern方法来进行测试
>
> intern方法：如果字符串常量池中**已经包含一个等于此String对象的字符串**，则返回代表池中这个字符串的String对象的引用；否则，会将此String对象包含的字符串**添加到常量池中**，并且返回此String对象的引用。

```java
/**
* VM Args：-XX:PermSize=6M -XX:MaxPermSize=6M
* @author zzm
*/
public class RuntimeConstantPoolOOM {
    public static void main(String[] args) {
        // 使用Set保持着常量池引用，避免Full GC回收常量池行为
        Set<String> set = new HashSet<String>();
        // 在short范围内足以让6MB的PermSize产生OOM了
        short i = 0;
        while (true) {
        set.add(String.valueOf(i++).intern());
        }
    }
}
```

- JDK6下的运行结果

```text
Exception in thread "main" java.lang.OutOfMemoryError: PermGen space
at java.lang.String.intern(Native Method)
at org.fenixsoft.oom.RuntimeConstantPoolOOM.main(RuntimeConstantPoolOOM.java: 18)
```

由于此版本中方法区采用永久代实现，运行时常量池仍然处于方法区，所以在PermGen Space报OOM错误

- 在JDK8下的运行结果

由于此版本中把运行时常量池以及字符串常量池从方法区移到了堆区，所以在电脑内存正常的状态下，会一直循环下去，如果限制了最大堆的大小为6MB则可能出现OOM异常，但这也和方法区无关

#### 附加：直接内存(Direct Memory)

##### 1）定义

直接内存（Direct Memory）并不是虚拟机运行时数据区的一部分，也不是《Java虚拟机规范》中定义的内存区域。其代表通过本地函数从堆外申请内存，并新建一个类的对象来作为这块内存的引用(指针)。

> 在NIO中，使用DirectByteBuffer对象作为这块内存的引用进行操作

##### 2）线程独有

申请到的直接内存属于提交申请的线程

##### 3）存储内容

可以存储任意内容，对于NIO来说即存储IO流的数据

##### 4）相关异常

直接内存不会收到堆内存的大小限制，但是会受到本机总内存的限制，因而也会产生OOM，而且在直接内存申请产生OOM错误时，dump信息会很少，且不会在错误打印中显示相关的内存区域名称(如Heap Area)。

###### ------ OOM举例 ------

直接从内存中循环进行内存分配，当内存被耗尽时，会报OOM异常，但是Heap Dump文件会很小，且不会指出内存溢出的空间(典型的场景为NIO)

### 2.2对象创建及内存分配

#### 2.2.1对象创建流程

##### 1）检查符号引用

当遇到字节码new指令时，先去检查这个指令的**参数**(new 后面的符号)能否**在常量池中**定位到**一个类符号的引用**，并且检查这个符号引用代表的类是否已被加载、解析和初始化过。如果没有，则需要先进行类的加载。

##### 2-1）为新对象分配内存

对象所需内存的大小在类加载完成后便可完全确定，为对象分配内存实际上相当于从Java堆中划分出一块确定的内存区域。划分内存分两种情况

###### （1）Java堆中内存绝对规整

​	**弹动指针(Bump The Pointer)**：由于存在一个指针(分界点指示器)将内存分为两块，一块是**被占用的**内存，另一块是**空闲的**内存。在划分内存时，即将指针向空闲的那一部分移动一块对象需要的内存大小的距离，完成内存分配。

###### （2）Java堆中内存不规整

​	**释放列表(Free List)**：由于被占用的内存和空闲的内存交织在一起，所以虚拟机会将内存信息存入一张表中，并记录占用情况。在分配内存时，会从表中选出对象所需的内存大小并标记为占用。

> Java堆是否规整取决于GC是否带有空间压缩能力。
>
> - 如果支持压缩，那么久采用弹动指针的方式
> - 如果不支持压缩，且使用基于清除算法(如CMS-Concurrent Mark Sweep)的GC时，采用释放列表的方式

##### 2-2）对象创建的同步方式

在并发情况下创建对象时，可能会存在下面情况

```java
class void TestClass(){
    MyClass a;
    Runnable run = new Runnable() {
        @Override
        public void run() {
            a = new MyClass();
        }
    }  
    while (true){
        Thread t = new Thread(run);
    }
}
```

当出现指针a同时进行两个new时，会出现内存分配错误（第一个分配的过程中，第二个分配同时开始），这时存在两种解决方案

###### （1）对内存分配动作做同步

​	虚拟机采用CAS(Compare And Swap)和失败重试的方式保证内存分配的原子性

###### （2）把内存分配的动作在每个线程中独立进行

​	每个线程在Java堆中预先分配一小块内存，称为本地线程分配缓冲（Thread Local AllocationBuffer，TLAB），哪个线程要分配内存，就在哪个线程的本地缓冲区中分配，只有本地缓冲区用完了，分配新的缓存区时才需要同步锁定。

> 在内存分配完成后，虚拟机需要把除了对象头的内存初始化为0（如成员变量）

##### 3）执行`<init>()`方法

上面的步骤将对象的内存已分配好，但是还没执行Class文件中的`<init>()`方法，通常Java编译器会把new关键字生成为两条字节码指令：new指令+`<init>()`。至此一个对象的初始化完成，也即对象创建完成

#### 2.2.2对象的内存布局

对象在堆内存中包括三部分：**对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）**

##### 1）对象头

###### （1）对象自身的运行时数据

包括hashcode、线程持有的锁等，这一部分称为Mark Word

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200907111257214.png" alt="image-20200907111257214" style="zoom:50%;" />

###### （2）类型指针

类型指针即对象指向它的**类型元数据的指针**，Java虚拟机通过这个指针来确定该对象是哪个**类**的实例

如果对象是Java数组，还需要保留数组的长度信息

##### 2）实例数据

包括对象**真正存储的有效信息**，即我们在程序代码里面所定义的各种类型的字段内容，无论是从父类继承下来的，还是在子类中定义的字段都必须记录起来。

##### 3）对齐填充

由于HotSpot虚拟机的自动内存管理系统要求对象起始地址**必须是8字节的整数倍**(对象头已经满足此要求)，所以在实例数据长度不满足要求时，需要对齐填充来补全，因而不是必须存在的部分。

#### 2.2.3对象访问定位

对象访问方式主要分为两种：句柄和直接指针

##### 1）句柄

###### （1）定义

Java**堆**中将可能会**划分出一块内存**来作为**句柄池**，reference中存储的就是**对象的句柄地址**，而句柄中包含了对象实例数据（成员变量）与类型数据（静态变量）**各自具体的地址信息**，其结构如下所示。

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200907134819947.png" alt="image-20200907134819947" style="zoom: 50%;" />

###### （2）优势

使用句柄来访问的最大好处就是reference中存储的是稳定句柄地址，在对象被移动（垃圾收集时移动对象是非常普遍的行为）时只会改变句柄中的实例数据指针，**而reference本身不需要被修改**

##### 2）直接指针

###### （1）定义

Java堆中对象的内存布局就必须考虑如何放置**访问类型数据的相关信息**，reference中存储的直接就是**对象地址**，如果只是访问对象本身的话，就不需要多一次间接访问的开销，如下所示。

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200907134954269.png" alt="image-20200907134954269" style="zoom:50%;" />

###### （2）优势

使用直接指针来访问最大的好处就是**速度更快**，它节省了一次指针定位的时间开销，由于**对象访问在Java中非常频繁**，因此这类开销积少成多也是一项极为可观的执行成本，就HotSpot而言，它主要使用直接指针进行对象访问



