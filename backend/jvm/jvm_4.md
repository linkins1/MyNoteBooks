## 4. 虚拟机性能监控及故障处理工具

### 4.1 基础故障处理工具

#### 4.1.1jps（JVM Process Status Tool）

和ps命令类似：可以列出正在运行的虚拟机进程，并显示虚拟机执行主类（Main Class，main()函数所在的类）名称以及这些进程的本地虚拟机唯一ID（LVMID，Local Virtual Machine Identifier）

**命令格式**如下

```shell
jps -<option>
```

option可选选项如下

<img src="C:\Users\123\AppData\Roaming\Typora\typora-user-images\image-20200911161056147.png" alt="image-20200911161056147" style="zoom:50%;" />

#### 4.1.2jstat（JVM Statistics Monitoring Tool）

用于监视虚拟机各种运行状态信息的命令行工具，可以显示本地或者远程[1]虚拟机进程中的类加载、内存、垃圾收集、即时编译等运行时数据。

jstat的**命令格式**如下

```shell
jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]
```

`option`可选参数如下

<img src="C:\Users\123\AppData\Roaming\Typora\typora-user-images\image-20200911161342776.png" alt="image-20200911161342776" style="zoom:80%;" />

`-t`代表是否带有时间戳输出

`-h<lines>`代表每lines行打印一次指标头

`vmid`代表对应的虚拟机中的进程的id

`intervals`代表隔多少ms打印一次

`counts`代表打印多少次后结束

#### 4.1.3jinfo（Configuration Info for Java）

作用是实时查看和调整虚拟机各项参数

> 下面引用自[JavaGuide的blog]([https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/jvm/JDK%E7%9B%91%E6%8E%A7%E5%92%8C%E6%95%85%E9%9A%9C%E5%A4%84%E7%90%86%E5%B7%A5%E5%85%B7%E6%80%BB%E7%BB%93.md#jstat-%E7%9B%91%E8%A7%86%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%90%84%E7%A7%8D%E8%BF%90%E8%A1%8C%E7%8A%B6%E6%80%81%E4%BF%A1%E6%81%AF](https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/jvm/JDK监控和故障处理工具总结.md#jstat-监视虚拟机各种运行状态信息))

`jinfo vmid` :输出当前 jvm 进程的全部参数和系统属性 (第一部分是系统的属性，第二部分是 JVM 的参数)。

`jinfo -flag name vmid` :输出对应名称的参数的具体值。比如输出 MaxHeapSize、查看当前 jvm 进程是否开启打印 GC 日志 ( `-XX:PrintGCDetails` :详细 GC 日志模式，这两个都是默认关闭的)。

```
C:\Users\SnailClimb>jinfo  -flag MaxHeapSize 17340
-XX:MaxHeapSize=2124414976
C:\Users\SnailClimb>jinfo  -flag PrintGC 17340
-XX:-PrintGC
```

使用 jinfo 可以在不重启虚拟机的情况下，可以动态的修改 jvm 的参数。尤其在**线上的环境**特别有用,请看下面的例子：

`jinfo -flag [+|-]name vmid` 开启或者关闭对应名称的参数。

```
C:\Users\SnailClimb>jinfo  -flag  PrintGC 17340
-XX:-PrintGC

C:\Users\SnailClimb>jinfo  -flag  +PrintGC 17340

C:\Users\SnailClimb>jinfo  -flag  PrintGC 17340
-XX:+PrintGC
```

#### 4.1.4jmap（Memory Map for Java）

用于生成堆转储快照（一般称为heapdump或dump文件）。也可以通过设定XX：+HeapDumpOnOutOfMemoryError参数在发生OOM时将dump文件输出，或者设定-XX：+HeapDumpOnCtrlBreak参数，这样使用Ctrl+Break或者kill -3来输出dump文件

jmap还可以查询finalize执行队列、Java堆和方法区的详细信息，如空间使用率、当前用的是哪种收集器等。

**命令格式**如下

```shell
jmap [ option ] vmid
```

option可选参数如下

<img src="C:\Users\123\AppData\Roaming\Typora\typora-user-images\image-20200911162458032.png" alt="image-20200911162458032" style="zoom:80%;" />

#### 4.1.5jhat（JVM Heap Analysis Tool）

命令与jmap搭配使用，来分析jmap生成的堆转储快照。一般不使用

#### 4.1.6jstack（Stack Trace for Java）

用于生成虚拟机当前时刻的**线程快照**（一般称为threaddump或者javacore文件）

**线程快照**就是当前虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的目的通常是定位线程出现长时间停顿的原因，如线程间**死锁、死循环、请求外部资源导致的长时间挂起**等，都是导致线程长时间停顿的常见原因

**命令格式**如下

```shell
jstack [ option ] vmid
```

option参数如下

<img src="C:\Users\123\AppData\Roaming\Typora\typora-user-images\image-20200911162715114.png" alt="image-20200911162715114" style="zoom:80%;" />

### 4.2 可视化故障处理工具

#### 4.2.1 JConsole（Java Monitoring and Management Console）

是一款基于JMX（Java Management Extensions）的可视化监视、管理工具。它的主要功能是通过JMX的MBean（Managed Bean）对系统进行信息收集和参数动态调整。

>如果需要使用 JConsole 连接远程进程，可以在远程 Java 程序启动时加上下面这些参数:
>
>```shell
>-Djava.rmi.server.hostname=外网访问 ip 地址 
>-Dcom.sun.management.jmxremote.port=60001   //监控的端口号
>-Dcom.sun.management.jmxremote.authenticate=false   //关闭认证
>-Dcom.sun.management.jmxremote.ssl=false
>```

##### 1）连接进程

启动JConsole时，会列出所有的运行中的进程，相当于执行了jps

<img src="C:\Users\123\AppData\Roaming\Typora\typora-user-images\image-20200911164401045.png" alt="image-20200911164401045" style="zoom: 80%;" />

##### 2）概览 

![image-20200911164556656](C:\Users\123\AppData\Roaming\Typora\typora-user-images\image-20200911164556656.png)

##### 3）内存(相当于jstat)

![image-20200911164616430](C:\Users\123\AppData\Roaming\Typora\typora-user-images\image-20200911164616430.png)

旁边的执行GC是执行一次Full GC

##### 4）线程(相当于jstack)

![image-20200911164634537](C:\Users\123\AppData\Roaming\Typora\typora-user-images\image-20200911164634537.png)

下面的检查死锁可以显示出遇到死锁的线程

##### 5）类加载状况

![image-20200911164711022](C:\Users\123\AppData\Roaming\Typora\typora-user-images\image-20200911164711022.png)

##### 6）VM概览

![image-20200911164723643](C:\Users\123\AppData\Roaming\Typora\typora-user-images\image-20200911164723643.png)

##### 7）MBeans

![image-20200911164757647](C:\Users\123\AppData\Roaming\Typora\typora-user-images\image-20200911164757647.png)

#### 4.2.2Visual VM:多合一故障处理工具

> 摘自[JavaGuide]([https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/jvm/JDK%E7%9B%91%E6%8E%A7%E5%92%8C%E6%95%85%E9%9A%9C%E5%A4%84%E7%90%86%E5%B7%A5%E5%85%B7%E6%80%BB%E7%BB%93.md#jstat-%E7%9B%91%E8%A7%86%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%90%84%E7%A7%8D%E8%BF%90%E8%A1%8C%E7%8A%B6%E6%80%81%E4%BF%A1%E6%81%AF](https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/jvm/JDK监控和故障处理工具总结.md#jstat-监视虚拟机各种运行状态信息))

VisualVM 提供在 Java 虚拟机 (Java Virutal Machine, JVM) 上运行的 Java 应用程序的详细信息。在 VisualVM 的图形用户界面中，您可以方便、快捷地查看多个 Java 应用程序的相关信息。Visual VM 官网：https://visualvm.github.io/ 。Visual VM 中文文档:https://visualvm.github.io/documentation.html。

下面这段话摘自《深入理解 Java 虚拟机》。

> VisualVM（All-in-One Java Troubleshooting Tool）是到目前为止随 JDK 发布的功能最强大的运行监视和故障处理程序，官方在 VisualVM 的软件说明中写上了“All-in-One”的描述字样，预示着他除了运行监视、故障处理外，还提供了很多其他方面的功能，如性能分析（Profiling）。VisualVM 的性能分析功能甚至比起 JProfiler、YourKit 等专业且收费的 Profiling 工具都不会逊色多少，而且 VisualVM 还有一个很大的优点：不需要被监视的程序基于特殊 Agent 运行，因此他对应用程序的实际性能的影响很小，使得他可以直接应用在生产环境中。这个优点是 JProfiler、YourKit 等工具无法与之媲美的。

VisualVM 基于 NetBeans 平台开发，因此他一开始就具备了插件扩展功能的特性，通过插件扩展支持，VisualVM 可以做到：

- **显示虚拟机进程以及进程的配置、环境信息（jps、jinfo）。**
- **监视应用程序的 CPU、GC、堆、方法区以及线程的信息（jstat、jstack）。**
- **dump 以及分析堆转储快照（jmap、jhat）。**
- **方法级的程序运行性能分析，找到被调用最多、运行时间最长的方法。**
- **离线程序快照：收集程序的运行时配置、线程 dump、内存 dump 等信息建立一个快照，可以将快照发送开发者处进行 Bug 反馈。**
- **其他 plugins 的无限的可能性......**