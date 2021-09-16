##  10. 常见问题

### 10.1 OOM异常

#### 10.1.1 PC 计数器

不会产生OOM异常，也不允许产生

#### 10.1.2 Java虚拟机栈

当无限递归且栈空间可动态扩展时产生OOM

> 当不可动态扩展时，会报SOF；当创建大量线程时，由于Java虚拟机栈是线程独有的，因而可能会报OOM，这时可以适当减少线程独有的区域所占的大小，如栈、TLAB

#### 10.1.3 堆

当创建了大量对象实例时，可能会报OOM，具体可能为**内存泄露**或**内存溢出**

#### 10.1.4 方法区

当使用**永久代时**，

- 如果运行时创建了大量的动态代理类，则可能会报OOM，因为永久代是内存上限的
- 如果在JDK1.7之前，由于字符串常量池也存在于永久代，则大量的intern操作也会引起OOM

### 10.2 各类栈

#### 10.2.1 Java虚拟机栈

在方法调用时，会使用到Java虚拟机栈，一个方法的调入和调出对应着栈帧的压入和弹出，Java虚拟机栈中的局部变量表中，会存储方法内定义的基本数据类型变量、引用类型变量的引用（实例存放在堆）

#### 10.2.2 本地方法栈

在调用本地方法时，会使用到此结构，功能和Java虚拟机栈一致，**但是**，执行本地方法时，PC 计数器中的值为undefined

#### 10.2.3 操作数栈

作为栈帧的组成部分，其存在于Java虚拟机栈中，由于Java的字节码指令只有操作码没有操作数，因而为了弥补这个缺陷，创建了操作数栈，和字节码指令一一对应，如load指令，对应一个局部变量被压入栈，store指令对应一个变量被存储到栈，add指令对应弹出两个操作数栈的数字，相加后再压入栈

#### # 参考

[Java中局部变量、实例变量和静态变量在方法区、栈内存、堆内存中的分配_leunging的博客-CSDN博客](https://blog.csdn.net/leunging/article/details/80599282)

### 10.3 垃圾回收

#### 10.3.1 垃圾判定

垃圾判定可以采用引用计数法和可达性分析法

- 引用计数法

  每个对象需要记录被多少对象引用，每有一个引用计数值+1，当值为0时可以将对象回收

  这种方式更容易出现内存泄漏

- 可达性分析算法

  给定了一个集合名为GC Roots，凡是可以被GC Roots根据引用链引用到的对象，都是存活对象；如果没有被GC Roots引用，则是垃圾对象

  GC Roots的主要类型有

  - 被栈帧中局部变量表所引用的对象
  - 常量对象（如果仍然被引用）
  - 引用类型的类静态变量
  - 被synchronized持有的对象
  - class对象，引导类加载器

#### 10.3.2 触发时机

> 对于年轻代的垃圾回收，是由young GC来完成，对于整个堆+方法区的回收，由full GC来完成
>
> 只有CMS是只针对老年代回收的，也即Old GC

按照下面配置讨论

-Xms 堆内存初始大小为20M 

-Xmx 堆内存最大大小为20M 

-Xmn 新生代大小为10M 

-XX:SurvivorRatio Eden:Survivor = 8:1

#### 10.3.1 Young GC

> 采用标记-复制

由于对象优先在Eden区进行分配，当Eden区没有足够内存进行分配时，会触发一次Young GC，如果新生代**所有对象**总大小**小于**当前老年代最大可用连续空间大小（保证全部对象都存活的情况安全），那么**可以安全触发Young GC**；如果**大于**，

- 允许分配担保失败**且**历代晋升对象平均值小于老年代最大可用连续空间

  触发一次Young GC，Young GC后如果**存活对象**大小

  - 小于Survivor区

    那么可以将存活对象拉入Survivor区

  - 大于Survivor区

    - 如果小于老年代最大可用连续空间大小

      将存活对象拉入老年代

    - 大于老年代可用大小
    
      触发Full GC

- 不允许分配担保失败或历代晋升对象平均值大于老年代最大可用连续空间

  **不触发Young GC**，直接触发Full GC

**从上可见**，Young GC在触发时，一定会先计算一次**新生代所有对象的总和**并与当前老年代的最大连续空间大小做对比，如果小于老年代，则可以完成一次安全的Young GC，那么对于应该将年轻代的那些对象拉入老年代的规则，有如下五种

- 允许分配担保机制失败

  也即前面提到的不安全的young GC之后，对于Survivor区<x<老年代最大可用连续空间大小的，拉入老年代

- 正常分配担保

  当新生代对象总量小于老年代最大连续可用空间时，这是一次安全的young GC，可以将对象安全的拉入到老年代

- 大对象直接拉入老年代

  对于分配在Eden区的且大小大于-XX:PretenureSizeThreshold（小于Eden）的，直接将其拉入老年代

- 长期存活对象拉入老年代

  由于没进行一次young GC，存活在Survivor中的对象年龄就会+1，当年龄大于-XX:MaxTenuringThreshold时，就会将其拉入老年代

- 动态对象年龄判断

  将Survivor中的对象按年龄自然排序，从头开始累加，当加到某个对象a时当前和大于了Survivor的一半，假设对象a的年龄为x，那么就会将年龄>=x的对象全部拉入老年代
  
  这样**就不必**等到对象年龄大于-XX:MaxTenuringThreshold时才拉入老年代

#### 10.3.2 Full GC

> 一般采用标记-整理

##### 1）堆区

如前面提到两种情况，如果新生代中所有对象总量**大于**当前老年代最大可用连续空间大小，如果

- 不允许分配担保失败
- 允许分配担保失败
  - 老年代可用最大连续大小仍小于历代晋升平均值
  - 老年代可用最大连续空间大于历代晋升平均值，**且Young GC后**，存活对象大小仍然大于Survivor区&&老年代最大可用连续空间大小

那么会触发Full GC，如果Full GC触发后内存仍然不够用，则报OOM

##### 2）方法区

回收的对象包括

- 废弃的常量

- 不再使用的类型

  需要满足如下条件

  - 类的所有实例被回收
  - 类加载器被回收
  - class对象没有被引用

#### 10.3.3 Old GC

只发生于CMS（采用标记-清除），其触发的标准是老年代的占用内存达到某一阈值时，就触发Old GC

#### #参考

[JVM性能调优（2） —— 垃圾回收器和回收策略 - bojiangzhou - 博客园 (cnblogs.com)](https://www.cnblogs.com/chiangchou/p/jvm-2.html)

### 10.4 破坏双亲委派机制

#### 10.4.1 重写loadClass()

在不破坏双亲委派模型的前提下想要自定义类加载器时，需要重写的是findClass方法来指定类的数据源并完成加载；如果想要完全破坏类加载机制，只需要重写loadClass方法，自定义类加载机制即可

ClassLoader的原有loadClass()如下

```java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

对于一个自定义的类，判断类是否已经加载过，如果

- 是

  直接解析类即可

- 否

  - 如果父类加载器
    - 不是空，则交给其父类加载器完成加载
    - 是空，则调用引导类加载器完成加载

  - 对于上面的规则按照如下顺序进行
    - AppClassLoader，parent是ExtClassLoader
    - ExtClassLoader，parent是空，调用到BootStrapClassLoader

  对于没有重写loadClass来加载指定类的，上面步骤中最终会由AppClassLoader来加载，否则，会交由自定义的类加载器完成加载，自定义类加载器的parent为AppClassLoader

#### 10.4.2 ServiceLoader

Java中通过**S**ervice **P**rovider **I**nterface的方式为第三方提供了统一的规范接口，实现方负责接口的实现，调用方负责使用接口。举例而言，对于数据库驱动对象的加载，是由DriverManager来完成的，MySQL负责了Driver接口的实现，由于DriverManager是由引导类加载器完成加载的，由于**当一个类加载器负责加载某个 CIass的时候, 该 Class所依赖的和引用的其他Class也将由该类加载器负责载入,除非显式使用另外一个类加载器来载入**，那么Driver接口的实现类理论上就需要给引导类加载器完成加载，这显然是不可能的，为了能够完成加载，提出了**ThreadContextClassLoader**，其实际上是AppClassLoader，那么在Driver加载类的过程中，**显示的调用**`Thread.currentThread().getContextClassLoader()`来获取此类加载器，这样就可以完成类加载

从上面过程可以看出，DriverManager是由引导类加载器加载的，为了能够加载SPI的第三方实现类，不得不引入AppClassLoader完成加载，这也就是在引导类加载器加载的过程中使用到了AppClassLoader完成加载，是一个父调用子的逻辑，这样便完成了类加载机制的破坏

#### 10.4.3 Tomcat的WebAppClassLoader

由于一个Tomcat容器中一个host可以配置多个context，不同可能存在多个版本的类库，对于不同版本类库中全限定类名相同的类，使用的是使用双亲委派机制无法正确完成加载，由于都使用AppClassLoader加载，不能加载两个标记相同的类

为了解决此问题，引入了WebAppClassLoader用于加载各个引用单独的类，达到了隔离性，避免一个类加载器加载多个同名的类的情况

***具体机制参加下文***

[Tomcat 架构原理解析到架构设计借鉴 - SegmentFault 思否](https://segmentfault.com/a/1190000023475177#:~:text=Tomcat 的整体架构包含,交流，容器负责内部处理。&text=连接器通过适配器 Adapter,复杂系统的基本思路。)

### 10.5 CMS和G1

#### 10.5.1 CMS

##### 1）算法

基于标记-清除

##### 2）步骤

- 初始标记（STW）

  标记GC ROOT可直接关联的对象

- 并发标记

  在上一步基础上查找引用链

- 重新标记（STW）

  对并发标记过程中漏掉的一些对象进行标记

- 并发回收

  回收标记好的对象

##### 3）优劣

- 优

  - 并发收集，并发回收
  - 低停顿

- 劣

  - 无法处理浮动垃圾

    由于重新标记只检查是否少标记了一些对象，但是，对于1，2阶段中原本被标记，却又被用户线程取消引用的对象a，就不会被重新标记这个过程检查到，那么这个没有被任何对象引用的对象a，就只能等到下一次gc清除

  - 内存碎片多，容易触发full gc

    受限于标记-清除算法，会带来此问题

#### 10.5.2 G1

不再采用CMS中将老年代和年轻代物理分割的方式管理堆内存，而是将整个堆划分成大小相同的region。这种划分下，保留了eden、survivor和old的逻辑概念，每个region可以作为上述任意一种类型

<img src="https://raw.githubusercontent.com/linkins1/MyNoteBooks/master/resources/imgs/temp/g1.png" alt="123" style="zoom:50%;" />

##### 1）算法

整体看基于标记-整理，局部看基于标记-复制

##### 2）步骤

- 初始标记（STW）

  标记GC ROOT可直接关联的对象

- 并发标记

  在上一步基础上查找引用链

- 最终标记（STW）

  对并发标记过程中漏掉的一些对象进行标记

- 并发回收

  回收标记好的对象，利用多核开启多个线程STW执行，但是用户可以设定执行时间，从而和用户线程并发执行

##### 3）优劣

- 优势

  - 低延迟

    - 利用多核特性，进一步降低时延
    - 老年代中的对象作为GC ROOTs可能会引用到新生代的对象，这些被引用的对象也会被当作GC ROOTs，从而触发全堆扫描。G1为老年代每个对象提供了一张卡表，对于有引用新生代对象的，其中存储了其引用的新生代对象，成为**脏表**，这样在minor gc时，只需要查询**脏表**中的新生代对象来避免扫描整个老年代，加快gc

  - 内存碎片少，不易触发full gc

    得益于标记-复制、标记-整理算法的特点

  - 面向整个堆

- 劣势

  - 存储卡表会占用额外的堆内存

### 10.6 类加载

- 加载

  读取class文件为二进制字节流，放入方法区，并在堆创建class对象

- 链接

  - 验证

    对class文件格式做验证，对类的合法性，字段的访问权限做验证

  - 准备

    为静态变量分配内存并初始化，注意ConstantValue这个特例

  - 解析

    将符号引用转为直接引用，期间会穿插加载过程，以及之前的验证过程

- 初始化

  调用clinit方法对静态变量赋值

### 10.7 类型擦除

下面两个方法不构成重载，因为类型擦除后，两者相同；此时，如果只是为了通过编译，可以让返回值不同

```java
public static String method(List<String> list) {
    System.out.println("invoke method(List<String> list)");
    return "";
}
public static int method(List<Integer> list) {
    System.out.println("invoke method(List<Integer> list)");
    return 1;
}
```
