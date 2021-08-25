## 3.GC与内存分配策略

### 3.1 什么对象需要被回收？

Java内存运行时区域的各个部分，其中**程序计数器、虚拟机栈、本地方法栈**3个区域**随线程而生，随线程而灭**，因而这几个区域的内存相对来说**较为固定**，不需要过多考虑对其进行垃圾回收(内存随着线程消亡而被回收)；**但是**，**Java堆和方法区**这两个区域则有着**很显著的不确定性**，只有在运行期间才可以发现真正被创建的对象数，因而一般的垃圾收集应该关注于这类区域。

### 3.2 对象消亡准则

#### 3.2.1确认存活算法

##### 1）引用计数算法(Reference Counting)

###### （1）定义

在对象中添加一个**引用计数器**，每当有一个地方**引用它时**，计数器值就**加一**；当引用**失效时**，计数器值就**减一**；任何时刻计数器**为零**的对象就是**不可被使用**的。

###### （2）优势

- 简单
- 效率高

###### （3）劣势

观察下面例子

```java
/**
* testGC()方法执行后，objA和objB会不会被GC呢？
* @author zzm
*/
public class ReferenceCountingGC {
    public Object instance = null;
        private static final int _1MB = 1024 * 1024;
        /**
        * 这个成员属性的唯一意义就是占点内存，以便能在GC日志中看清楚是否有回收过
        */
        private byte[] bigSize = new byte[2 * _1MB];
        public static void testGC() {
        ReferenceCountingGC objA = new ReferenceCountingGC();
        ReferenceCountingGC objB = new ReferenceCountingGC();
        objA.instance = objB;
        objB.instance = objA;
        objA = null;
        objB = null;
        // 假设在这行发生GC，objA和objB是否能被回收？
        System.gc();
    }
}
```

运行结果

```text
[Full GC (System) [Tenured: 0K->210K(10240K), 0.0149142 secs] 4603K->210K(19456K), [Perm : 2999K->2999K(21248K)], Heap
...
```

由于objA和objB互相存在引用，因而两者的引用计数器都为1，当令两者为null时，引用并未失效，因而不会被回收，观察回收空间可以发现，仍有210k没有被回收也即objA和objB

##### 2）可达性分析算法(Reachability Analysis)

###### （1）定义

通过一系列称为“GC Roots”的根对象作为起始节点集，从这些节点开始，根据引用关系向下搜索，搜索过程所走过的路径称为“引用链”（Reference Chain），如果某个对象到GC Roots间没有任何引用链相连，那么从GC Roots到这个对象**不可达**时，则证明此对象是**不能再被使用**的。

下图中object5、6、7互相可达，但是和GC Roots不可达，因而被判定为需要被回收

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200909170222361.png" alt="image-20200909170222361" style="zoom:50%;" />

###### （2）GC Root对象可以为如下种类

- 在虚拟机栈（栈帧中的本地变量表）中引用的对象，譬如各个线程被调用的方法堆栈中使用到的参数、局部变量、临时变量等
- 在方法区中类静态属性引用的对象，譬如Java类的引用类型静态变量
- 在方法区中常量引用的对象，譬如字符串常量池（String Table）里的引用
- 在本地方法栈中JNI（即通常所说的Native方法）引用的对象
- Java虚拟机内部的引用，如基本数据类型对应的Class对象，一些常驻的异常对象（比如
  NullPointExcepiton、OutOfMemoryError）等，还有系统类加载器
- 所有被同步锁（synchronized关键字）持有的对象
- 反映Java虚拟机内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等

除了这些固定的GC Roots集合以外，根据用户所选用的垃圾收集器以及当前回收的内存区域不同，还可以有其他对象“临时性”地加入，共同构成完整GC Roots集合

###### （3）对象消亡条件

对象被判定为消亡需要经历**两次标记**，步骤如下

- 如果对象没有和GC Root关联，那么被标记一次，放入F-Queue等待第二次筛选
  - 如果对象所属的类没有覆盖finalize方法，则不执行finalize，并进行第二次标记，宣告消亡
  - 如果对象所属的类覆盖了finalize方法，则会执行此方法
    - 如果在finalize方法内重新让此对象引用某块内存，则逃离此次筛选，由于finalize方法只执行一次，所以下次回收时必被回收
    - 如果finalize方法中没有让此对象重新指向某片内存，则被标记第二次，宣告消亡

**但是**，**建议使用try-finally来替换finalize方法！**

#### 3.2.2 引用分级

> 引用计算算法和可达性分析算法都需要依赖于引用关系来确定一个对象是否真的消亡。狭义的引用是指如下类型
>
> StringBuilder **sb** = new StringBuilder()；
>
> 诸如变量sb这类保存了一个对象的地址的变量被称为是狭义的引用类型。

##### 1）强引用（Strongly Reference）

强引用是最传统的“引用”的定义，是指在程序代码之中普遍存在的引用赋值，即类似“Object obj=new Object()”这种引用关系。无论任何情况下，**只要强引用关系还存在，垃圾收集器就永远不会回收掉被引用的对象**

##### 2）软引用（Soft Reference）

软引用是用来描述一些**还有用，但非必须的**对象。只被软引用关联着的对象，在系统**将要发生内存溢出异常前**，会把这些对象列进回收范围之中进行回收，如果**还没有足够的内存**，才会**抛出内存溢出异常**。JDK1.2版之后提供了**SoftReference类**来实现软引用

##### 3）弱引用（Weak Reference）

弱引用也是用来描述那些非必须对象，强度**比软引用更弱**，被弱引用关联的对象**只能生存到下一次垃圾收集发生**。当垃圾收集器开始工作，**无论当前内存是否足够，都会回收掉只被弱引用关联的对象**。在JDK 1.2版之后提供了WeakReference类来实现弱引用。

##### 4）虚引用（Phantom Reference）

虚引用也称为“幽灵引用”或者“幻影引用”，它是**最弱的一种引用**关系。一个对象是否有虚引用的存在，**完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例**。为一个对象设置虚引用关联的**目的只是为了能在这个对象被收集器回收时收到一个系统通知**。在JDK 1.2版之后提供了PhantomReference类来实现虚引用

#### 3.2.3 方法区的回收

> 方法区回收成本高，效果一般，因而一般不做回收

##### 1）回收的内容

废弃的常量和不再使用的类型，参加下面**例子**

假如一个字符串**“java”曾经进入常量池中**，但是当前系统又没有任何一个字符串对象的值是“java”(引用“java”)。如果在这时发生内存回收，而且垃圾收集器判断确有必要的话，这个“java”常量就将会被系统清理出常量池。常量池中其他类（接口）、方法、字段的符号引用也与此类似。

##### 2）类型被允许回收的条件

- 该类所有的实例都已经被回收，也就是Java堆中不存在该类及其任何派生子类的对象

- 加载该类的类加载器已经被回收，这个条件除非是经过精心设计的可替换类加载器的场景，如OSGi、JSP的重加载等，否则通常是很难达成的。

- 该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

### 3.3 垃圾回收算法

>垃圾收集算法可以划分为**“引用计数式垃圾收集”**（Reference Counting GC）和**“追踪式垃圾收集”**（Tracing GC）两大类，这两类也常被称作“直接垃圾收集”和“间接垃圾收集”，目前主流的是**“追踪式垃圾收集”**

#### 3.3.1 分代收集理论

分代收集理论中存在下面三个经验法则

##### 1）弱分代假说（Weak Generational Hypothesis）：绝大多数对象都是朝生夕灭的。

##### 2）强分代假说（Strong Generational Hypothesis）：熬过越多次垃圾收集过程的对象就越难以消亡。

##### 3）跨代引用假说（Intergenerational Reference Hypothesis）：跨代引用相对于同代引用来说仅占极少数。

这通过这三个法则可知，我们需要将Java堆的内存进行划片管理，常用的划片方式是分为新生代(young generation)和老年代(old generation)，新生代中存放朝生夕灭的对象，老年代中存放难以消亡的对象。

对于存在跨代引用的对象，假设老年代对象a引用新生代对象b，那么在a未消亡时，b就会一直存活，会被从新生代拉到老年代，这样跨代引用也就消除了，因而跨代引用的情况一般较少。

#### 3.3.2 标记-清除算法（Mark-Sweep）

##### 1）定义

算法分为“**标记**”和“**清除**”**两个**阶段：首先标记出所有需要回收的对象，在标记完成后，统一回收掉所有被标记的对象，也可以反过来，标记存活的对象，统一回收所有未被标记的对象。标记过程就是对象是否属于垃圾的判定过程。

##### 2）图示

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-202010011328173091.png" alt="image-20201001132817309" style="zoom:67%;" />

##### 3）特性

- 优势

  实现简单

- 劣势

###### （1）效率不稳定

如果Java堆中包含大量对象，而且其中大部分是需要被回收的，这时必须进行大量标记和清除的动作，导致标记和清除两个过程的执行效率都随对象数量增长而降低

###### （2）内存空间的碎片化

标记、清除之后会产生**大量不连续的内存碎片**，空间碎片太多可能会导致当以后在程序运行过程中**需要分配较大对象时无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作**

#### 3.2.3 标记-复制算法（Mark-Copy）

##### 1）定义

利用“半区复制”（Semispace Copying）垃圾收集算法，它将可用内存**按容量划分为大小相等的两块**，每次**只使用其中的一块**。当这一块的内存用完了，就**将还存活着的对象复制到另外一块上面**，**然后再把已使用过的内存空间一次清理掉**

##### 2）图示

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-202009101337459871.png" alt="image-20200910133745987" style="zoom: 67%;" />

##### 3）特性

- 优势

  对于**多数对象都是可回收**的情况，算法需要**复制的就是占少数的存活对象**，而且每次都是针对整个半区进行内存回收，分配内存时也就**不用考虑有空间碎片**的复杂情况，只要移动堆顶指针，按顺序分配即可。这样实现简单，运行高效。

- 劣势

  如果内存中多数对象都是存活的，这种算法将会产生**大量的内存间复制的开销**，此外，这种复制回收算法的代价是将可用内存缩小为了原来的一半，**浪费了大量空间**

##### 4）存在问题

在对象存活率较高时就要进行较多的复制操作，效率将会降低。更关键的是，如果不想浪费50%的空间，就需要有额外的空间进行分配担保，以应对被使用的内存中所有对象都100%存活的极端情况，所以在**老年代一般不能直接选用这种算法**。

#### 3.2.4 标记-整理算法（老年代适用）

##### 1）定义

对存活对象进行标记，让所有存活的对象都向内存空间一端移动，然后直接清理掉边界以外的内存

##### 2）图示

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200910155050139.png" alt="image-20200910155050139" style="zoom: 67%;" />

##### 3）特性

- 优势

  由于每次不是直接回收内存，而是将存活对象向一端移动，那么这样在回收对象后，可以得到一段连续的内存，**解决了内存碎片化的问题**

- 劣势

  移动存活对象并更新所有引用这些对象的地方将会是一种**极为负重**的操作，而且这种对象移动操作必须全程暂停用户应用程序才能进行

##### 4）和标记-清除算法比较

本质区别是标记-清除是一种非移动式的回收算法，而标记-整理是移动式的，从整个程序的吞吐量来看，移动对象会更划算。即使不移动对象会使得收集器的效率提升一些，但因内存分配和访问相比垃圾收集频率要高得多，这部分的耗时增加，总吞吐量仍然是下降的

### 3.4 HotSpot算法细节

---

**​*待办!!!***

---

### 3.5 垃圾收集器

#### 3.5.1 分类

按大类划分，GC可以分为下面几种

##### 1）部分收集（Partial GC）：

指目标不是完整收集整个Java堆的垃圾收集，其中又分为：

###### （1）新生代收集（Minor GC/Young GC）：指目标只是新生代的垃圾收集。

###### （2）老年代收集（Major GC/Old GC）：指目标只是老年代的垃圾收集(只有CMS)。

###### （3）混合收集（Mixed GC）：指目标是收集整个新生代以及部分老年代的垃圾收集(只有G1)。

##### 2）整堆收集（Full GC）：收集整个Java堆和方法区的垃圾收集。

#### 3.5.2 经典垃圾收集器

##### A.新生代收集器

##### 1）Serial收集器

###### （1）定义

收集器是一个单线程工作的收集器，采用标记复制算法。下图是Serial/SerialOld搭配的示意图，其中SerialOld采用标记整理

![image-20200910161234841](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200910161234841.png)

###### （2）特性

- 优势

  对于内存资源受限的环境，它是所有收集器里额外**内存消耗（Memory Footprint）最小的**；对于单核处理
  器或处理器核心数较少的环境来说，Serial收集器由于**没有线程交互的开销**，专心做垃圾收集自然可以获得最高的单线程收集效率。

- 劣势

  需要暂停用户线程，降低了程序执行效率

###### （3）应用场景

HotSpot虚拟机运行在客户端模式下的默认新生代收集器

##### 2）ParNew收集器

###### （1）定义

实质上是**Serial收集器的多线程并行版本**，除了同时使用多条线程进行垃圾收集之外完全一致。下图是ParNew/SerialOld搭配的示意图

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200910162040370.png" alt="image-20200910162040370" style="zoom:80%;" />

此处所讲的并行是指多个GC线程并行执行，仍然会暂停用户线程

###### （2）特性

默认开启的收集线程数与处理器核心数量相同，在处理器核心非常多（譬如32个，现在CPU都是多核加超线程设计，服务器达到或超过32个逻辑核心的情况非常普遍）的环境中，可以使用-XX：ParallelGCThreads参数来限制垃圾收集的线程数。

在单线程状态下，ParNew的效率并不会高于Serial

> 只有ParNew能和CMS（老年代）搭配

##### 3）Parallel Scavenge收集器

###### （1）定义

基于标记复制算法的多线程并行收集器

###### （2）特性

Parallel Scavenge收集器的目标则是达到一个可控制的吞吐量（Throughput），其中

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200910163801503.png" alt="image-20200910163801503" style="zoom: 33%;" />

- 吞吐量举例

  将每10秒收集一次，每次收集占用100ms，改为，每5秒收集一次，每次收集占用70ms，那么吞吐量就从10/10.1=0.99变为5/5.07=0.98，这样吞吐量也会下降

###### （3）调整策略

垃圾收集的**自适应的调节策略**（GC Ergonomics）不需要人工指定新生代的大小（-Xmn）、Eden与Survivor区
的比例（-XX：SurvivorRatio）、晋升老年代对象大小（-XX：PretenureSizeThreshold）等细节参数了，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量

使用Parallel Scavenge收集器配合自适应调节策略，把内存管理的调优任务交给虚拟机去完成也许是一个很不错的选择。

- 通过XX：+UseAdaptiveSizePolicy开启自适应调节策略
- 把基本的内存数据设置好（如-Xmx设置最大堆）
- 使用-XX：MaxGCPauseMillis参数（更关注最大停顿时间）或-XX：GCTimeRatio（更关注吞吐量）参数给虚拟机设立一个优化目标，那具体细节参数的调节工作就由虚拟机完成了。

##### B.老年代收集器

##### 1）Serial Old收集器

###### （1）定义

Serial Old是**Serial收集器的老年代版本**，它同样是一个**单线程收集器**，使用**标记-整理**算法

##### 2）Parallel Old收集器
###### （1）定义

Parallel Old是**Parallel Scavenge收集器的老年代版本**，支持多线程**并发收集**，基于**标记-整理**算法实现，下图为Parallel Scavenge/Parallel Old搭配使用

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200910165058126.png" alt="image-20200910165058126" style="zoom: 67%;" />

在**注重吞吐量或者处理器资源较为稀缺的场合**，都可以优先考虑上面这个组合

##### 3）CMS收集器（Concurrent Mark Sweep）
###### （1）定义

CMS收集器基于**标记-清除**算法，具体讲是改进的**三色标记法**，基础的标记-清除算法的整个标记过程是STW，三色标记法是分步完成的，部分步骤可以并发

###### （2）三色标记法

方法使用黑白灰三种颜色来代表对象被标记的状态，具体步骤如下

- 堆中的对象都设定成白色（每一轮开始都如此，原来的灰色、黑色会被刷新为白色）
- 将GC ROOTs可以直接关联的白色对象标记为灰色，STW
- 将灰色对象引用的白色对象标记为灰色，并将原灰色对象标记为黑色
- 重复上面步骤，直到没有灰色对象
- 清除所有白色对象

从上面步骤可见，对于GC期间被黑色、灰色或GC ROOTs新引用的对象，如果其被标记成白色，就会被回收掉。

在CMS中，通过**增加重新标记**这一步骤，来标记引用产生变动的对象，这一步是STW的

> 在Go语言中，通过加入插入写屏障（不是写屏障指令，只是一种额外保险操作）的方式，具体为在新建对象时同时染色为灰色，但是在多协程下，这种方式开销很大，所以在Go1.8之后，采用插入写屏障和删除写屏障构成的混合写屏障，且**写屏障只作用于堆**，混合写屏障具体步骤如下
>
> - GC 开始，将栈上的**全部可达对象**标记为**黑色**，之后便不再需要进行重新扫描
> - GC 期间，任何在栈上**新创建的对象**都标记为**黑色**
> - 写屏障将被**删除的对象**标记为**灰色**
> - 写屏障将新**添加的对象**标记为**灰色**
>
> 这样就满足了弱三色不变性
>
> 有了混合写屏障，那么整体的GC步骤就是三色标记法+混合写屏障结合，三色标记法负责常规堆内存的回收，混合写屏障负责处理GC期的新增和删除的堆内存
>
> [Golang 三色标记和混合写屏障 - thepoy - 博客园 (cnblogs.com)](https://www.cnblogs.com/thepoy/p/14598447.html)
>
> [图文结合，白话 Go 的垃圾回收原理 - 码农桃花源](https://mp.weixin.qq.com/s/VIheVqCL9O_Jy7rff-StZA)

###### （3）CMS回收步骤

其标记过程分为**四步**

:one:初始标记（CMS initial mark）-- "Stop The World"

标记GCRoots能直接关联到的对象，速度很快

:two:并发标记（CMS concurrent mark）-- 与用户线程并发

从GC Roots的直接关联对象开始遍历整个对象图的过程，这个过程耗时较长但是不需要停顿用户线程，可以与垃圾收集线程一起并发运行

:three:重新标记（CMS remark）-- "Stop The World"

因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，需要进行并发标记的修正，具体为对并发标记过程中漏掉的一些可达对象进行标记，远比并发标记阶段的时间短

:four:并发清除（CMS concurrent sweep）-- 与用户线程并发

清理删除掉标记阶段判断的已经死亡的对象，由于不需要移动存活对象，所以这个阶段也是可以与用户线程同时并发的

下图为CMS的执行流程图

![image-20200910165715990](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200910165715990.png)

###### （4）特性

- 优势

  由于CMS是一种以获取最短回收停顿时间为目标的收集器，可以保证极端的停等时间，适用于服务端

- 劣势

  - CMS收集器对处理器资源非常敏感，占用处理器资源，降低吞吐量
  - CMS收集器无法处理“浮动垃圾”（Floating Garbage 运行期间产生的垃圾），可能会触发Full GC
  - 囿于标记清除算法，会产生大量的空间碎片

##### 4）Garbage First收集器

###### （1）定义

开创了收集器面向局部收集的设计思路和基于Region的内存布局形式，是适用于服务端，面向全Java堆的**多线程**垃圾收集器（新老结合）。G1从**整体来看是基于“标记-整理”**算法实现的收集器，但从**局部（两个Region之间）上看又是基于“标记-复制”算法**实现

###### <u>*:grey_question:何为Region*</u>

把连续的Java堆划**分为多个大小相等的独立区域（Region）**，每一个Region都可以**根据需要，扮演**新生代的Eden空间、Survivor空间，或者老年代空间。收集器能够对扮演不同角色的Region采用不同的策略去处理，这样无论是新创建的对象还是已经存活了一段时间、熬过多次收集的旧对象都能获取很好的收集效果。

###### <u>*:grey_question:何为Region的价值*</u>

价值即回收所获得的空间大小以及回收所需时间的经验值

###### <u>*:grey_question:如何回收Region*</u>

在后台维护一个**优先级列表**，每次根据用户设定允许的收集停顿时间（使用参数-XX：MaxGCPauseMillis指定，默
认值是200毫秒），优先处理**回收价值收益最大的那些Region**

G1的**标记过程如下四步**

- **初始标记**（Initial Marking）
- **并发标记**（Concurrent Marking）
- **最终标记**（Final Marking）
- **筛选回收**（Live Data Counting and Evacuation）

最终，可以通过下图看出G1的并发和独占的流程

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/jvm/image-20200910172354829.png" alt="image-20200910172354829" style="zoom:80%;" />

###### （2）与CMS对比

- 优势
  - 由于采用**整体“标记-整理”局部（两个Region之间）“标记-复制”算法**，因而可以减少内存碎片的产生
  - 可以设置不同的期望停顿时间，可使得G1在不同应用场景中取得关注吞吐量和关注延迟之间的最佳平衡，但是不可以将停顿时间设置过小，这样会导致垃圾堆积，触发Full GC，最终效率下降
  - 增大并行的比重，进一步有效利用多核处理器
- 劣势
  - 内存占用高（存储卡表）
  - 额外执行负载大（维护卡表）

#### 3.5.3 低延迟垃圾收集器

>衡量垃圾收集器的三项最重要的指标是：内存占用（Footprint）、吞吐量（Throughput）和延迟低（Latency），三者共同构成了一个“不可能三角

##### 1）Shenandoah收集器 

##### 2）ZGC收集器

### 3.6 内存分配与垃圾回收

> 下面的分析都是**基于Serial/SerialOld的组合**

#### Pre 参数说明

##### 1）new generation total 

Eden区+From Survivor的总大小，也即新生代的内存大小

##### 2）eden Space

Eden区的内存大小

##### 3）From/To Space

From / To Survivor 的内存大小

##### 4）tenured Space

老年代的内存大小

#### 3.6.1 对象优先在Eden分配

##### 1）经验准则

对象一般在Eden中进行内存分配，当Eden中内存区域不够时会触发一次Minor GC

##### 2）测试代码

```java
private static final int _1MB = 1024 * 1024;
/**
* VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
*/
public static void testAllocation() {
    byte[] allocation1, allocation2, allocation3, allocation4;
    allocation1 = new byte[2 * _1MB];
    allocation2 = new byte[2 * _1MB];
    allocation3 = new byte[2 * _1MB];
    allocation4 = new byte[4 * _1MB]; // 出现一次Minor GC
}
```

运行结果如下

```text
[GC [DefNew: 6651K->148K(9216K), 0.0070106 secs] 6651K->6292K(19456K), 0.0070426 secs] [Times: user=0.00 sys=0.00, 
Heap
    def new generation total 9216K, used 4326K [0x029d0000, 0x033d0000, 0x033d0000)
        eden space 8192K, 51% used [0x029d0000, 0x02de4828, 0x031d0000)
        from space 1024K, 14% used [0x032d0000, 0x032f5370, 0x033d0000)
        to space 1024K, 0% used [0x031d0000, 0x031d0000, 0x032d0000)
    tenured generation total 10240K, used 6144K [0x033d0000, 0x03dd0000, 0x03dd0000)
        the space 10240K, 60% used [0x033d0000, 0x039d0030, 0x039d0200, 0x03dd0000)
    compacting perm gen total 12288K, used 2114K [0x03dd0000, 0x049d0000, 0x07dd0000)
        the space 12288K, 17% used [0x03dd0000, 0x03fe0998, 0x03fe0a00, 0x049d0000)
No shared spaces configured.
```

上面测试中，规定了Java堆的大小为20MB，其中新生代的大小为10MB，且Eden:From Survivor=8:1，所以Eden占8MB，From Survivor占1MB。

在做内存分配时，allocation1、2、3一共占用了6MB，会先被分配在Eden区，这时Eden还剩下2MB，但是allocation4占用4MB，因而触发一次Minor GC，**通过分配担保机制**，将1、2、3拉入老年代，并将4分配到Eden区。

上面显示Eden区占用了4326k，大于4096k，这是其他的新对象的占用导致的，且From Space也不为空，也是存放了其他的没有被Minor GC回收的对象。

#### 3.6.2 大对象直接进入老年代

##### 1）经验准则

长数组就属于大对象，可以通过指定阈值PretenureSizeThreshold(预老年化阈值)，当分配的对象超过这个大小时，直接将对象分配到老年代，从而避免在Eden区和Survivor区来回复制的消耗。

>PretenureSizeThreshold只对Serial和ParNew有效

##### 2）测试代码

```java
private static final int _1MB = 1024 * 1024;
/**
* VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
* -XX:PretenureSizeThreshold=3145728
*/
public static void testPretenureSizeThreshold() {
    byte[] allocation;
    allocation = new byte[4 * _1MB]; //直接分配在老年代中
}
```

运行结果

```text
Heap
    def new generation total 9216K, used 671K [0x029d0000, 0x033d0000, 0x033d0000)
        eden space 8192K, 8% used [0x029d0000, 0x02a77e98, 0x031d0000)
        from space 1024K, 0% used [0x031d0000, 0x031d0000, 0x032d0000)
        to space 1024K, 0% used [0x032d0000, 0x032d0000, 0x033d0000)
    tenured generation total 10240K, used 4096K [0x033d0000, 0x03dd0000, 0x03dd0000)
    	the space 10240K, 40% used [0x033d0000, 0x037d0010, 0x037d0200, 0x03dd0000)
    compacting perm gen total 12288K, used 2107K [0x03dd0000, 0x049d0000, 0x07dd0000)
    	the space 12288K, 17% used [0x03dd0000, 0x03fdefd0, 0x03fdf000, 0x049d0000)
No shared spaces configured.
```

从测试结果可以看出，4MB直接分配到了老年代，新生代几乎没有占用空间

#### 3.6.3 长期存活的对象将进入老年代

##### 1）经验准则

虚拟机给每个对象定义了一个对象年龄（Age）计数器，存储在对象头中。对象通常在Eden区里诞生，如果经过第一次Minor GC后仍然存活，并且能被Survivor容纳的话，该对象会被移动到Survivor空间中，并且将其对象年龄设为1岁。对象在Survivor区中每熬过一次Minor GC，年龄就增加1岁，当它的年龄增加到一定程
度（默认为15），就会被晋升到老年代中。

> 对象晋升老年代的年龄阈值，可以通过参数-XX：MaxTenuringThreshold设置。

##### 2）测试代码

```java
private static final int _1MB = 1024 * 1024;
/**
* VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:Survivor-
Ratio=8 -XX:MaxTenuringThreshold=1
* -XX:+PrintTenuringDistribution
*/
@SuppressWarnings("unused")
public static void testTenuringThreshold() {
    byte[] allocation1, allocation2, allocation3;
    allocation1 = new byte[_1MB / 4]; // 什么时候进入老年代决定于XX:MaxTenuring-Threshold设置
    allocation2 = new byte[4 * _1MB];
    allocation3 = new byte[4 * _1MB];
    allocation3 = null;
    allocation3 = new byte[4 * _1MB];
}
```

当设定MaxTenuringThreshold=1时，发生第二次Minor GC时，就会将Eden区的对象转存到老年代

当设定MaxTenuringThreshold=15时，发生第二次Minor GC时，Eden区的对象仍然存放在Eden区

#### 3.6.4 动态对象年龄判定

##### 1）经验准则

为了能更好地适应不同程序的内存状况，HotSpot虚拟机并不是永远要求对象的年龄必须达到-XX：MaxTenuringThreshold才能晋升老年代，其会将对象按年龄自然排序，从头开始累加，当加到某个对象a时当前和大于了Survivor的一半，假设对象a的年龄为x，那么就会将年龄>=x的对象全部拉入老年代，无须等到-XX：MaxTenuringThreshold中要求的年龄。

##### 2）测试代码

```java
private static final int _1MB = 1024 * 1024;
/**
* VM参数：-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
-XX:MaxTenuringThreshold=15
* -XX:+PrintTenuringDistribution
*/
@SuppressWarnings("unused")
public static void testTenuringThreshold2() {
    byte[] allocation1, allocation2, allocation3, allocation4;
    allocation1 = new byte[_1MB / 4]; // allocation1+allocation2大于survivo空间一半
    allocation2 = new byte[_1MB / 4];
    allocation3 = new byte[4 * _1MB];
    allocation4 = new byte[4 * _1MB];
    allocation4 = null;
    allocation4 = new byte[4 * _1MB];
}
```

上面的虚拟参数设定MaxTenuringThreshold=15，上面代码中allocation1、2占用的内存相同，且两者会先都分配在Eden区中，当经历一次Minor GC，会将这两个对象剪切到Survivor区，这时Survivor区中相同年龄的对象1和2加起来占用了0.5MB=一半的Suvivor区的容量，所以会将这两个对象直接转存到老年代。

**如果注释掉1、2任意一个，那么另一个就不会被拉入到老年代**

##### 3）年龄累计

Hotspot遍历所有对象时，就会开始对所有对象进行动态年龄判定，如果某个年龄N的对象总和大于等于一半的Survivor区的大小，那么**取这个年龄N和MaxTenuringThreshold中更小的一个值**，作为新的晋升年龄阈值。

之所以这么取的原因是，可以动态发现当前运行代码中实际的同龄对象的累计状况，从而将这些对象移入老年代，以避免同龄的长存对象创建过多而引起Survivor区的内存压力

#### 3.6.5 空间分配担保

##### 1）经验准则

###### 分配担保机制

在新生代没有内存可以分配时，会先计算一次老年代的最大可用连续空间的大小，与新生代中所有对象的大小总和比较，如果

- 大于，则可以安全的进行垃圾回收，回收后如果剩余存活的对象大小和小于Survivor区，则放入Survivor，否则拉入老年代
- 小于，则执行下面步骤

    :one:虚拟机会先查看-XX：HandlePromotionFailure参数的设置值是否允许担保失败（Handle Promotion Failure）；如果允许，那会继续检查**老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小**，如果大于，将尝试进行一次Minor GC，**尽管这次Minor GC是有风险的**（风险在于平均大小只是**经验值**），Minor GC后，如果剩余存活的对象大小和小于Survivor区，则放入Survivor，否则拉入老年代

    :two:如果小于或者-XX：HandlePromotionFailure设置为不允许，则触发一次Full GC

    > 通常都会将-XX：HandlePromotionFailure设置为允许，这样可以避免Full GC过于频繁。JDK6之后默认打开

#### 参考

[JVM性能调优（2） —— 垃圾回收器和回收策略 - bojiangzhou - 博客园 (cnblogs.com)](https://www.cnblogs.com/chiangchou/p/jvm-2.html)
