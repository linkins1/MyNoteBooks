## 3. 线程

### 3.1 基本概念

#### 3.1.1 定义

线程是**轻量级的**进程，是一个基本的CPU执行单元（处理机**调度的基本单位**），也是程序执行流的最小单位。

**需要注意！**进程仍然是资源分配的单位，进程中的线程共享所属进程的资源

> ***Q：*****所有系统资源都是以进程为单位进行分配的吗？**
>
> ***A：***不是，CPU执行权是以线程为单位进行分配的

#### 3.1.2特性

##### 1）并发

可以使用类似于多进程的方式，多个线程也可以并发的执行

##### 2）共享进程资源

属于同一个进程的多个线程共享进程中的资源（如进程中的变量），这也使得进行线程切换的开销要小于进程切换

##### 3）独有的资源

每个线程都有单独的程序计数器和局部变量表

##### 4）主线程

每一个进程至少拥有一个线程即主线程

#### 3.1.3分类

##### 1）用户级线程

###### （1）定义

即在用户空间（用户进程内）创建的线程，不需要内核支持，它的内核的切换是由用户态程序自己控制内核的切换，不需要内核的干涉。但是它不能像内核级线程一样更好的运用多核CPU。

###### （2）特性

- 调度单位

  在只有用户级线程的系统内，CPU调度还是以进程为单位，处于运行状态的进程中的多个线程，由用户程序控制线程的轮换运行

- 实现

  可以在不支持线程的操作系统中实现

- 阻塞

  同一进程中只能同时有一个线程在运行，如果有一个线程使用了系统调用而阻塞，那么整个进程都会被挂起，可以节约更多的系统资源

##### 2）内核级线程

###### （1）定义

即在用户态运行、由操作系统内核负责切换调度的线程

###### （2）特性

- 调度

  在内核支持线程的系统内，CPU调度则**以内核级线程为单位**，在进行线程切换调度时需要从用户态切换到内核态，切换完毕后从内核态转换到用户态运行

- 并行

  当有多个处理机时，一个进程的多个线程可以并行执行

##### 3）用户级线程与内核级线程映射

###### （1）定义

在同时支持内核级和用户级线程的系统中，可以按不同规则将用户级线程映射到内核级线程上

###### （2）映射规则

- 多对一

  多个用户级线程映射到一个内核级线程

  - 优缺点

    - 优

      用户级线程的切换在用户态可以完成

    - 缺

      一个用户级线程阻塞后，整个进程都会被阻塞，多个线程不可在多核处理器上并行执行

- 一对一

  一个用户级线程映射到一个内核级线程

  - 优缺点

    - 优

      当一个用户级线程阻塞，其他线程仍可运行

    - 缺

      一个用户进程会占用多个内核级线程，线程切换需要在核心态下完成

- 多对多

  将n个用户级线程映射到m个内核级线程上，假设有3个用户级线程，2个内核级线程，那么在进行线程切换时，由于要在内核态完成，就必定有两个用户级线程会共同映射到一个内核级别线程上。同时最多也只有两个内核级线共同执行

  - 特性
    - 解决了多对一并发度不高的问题
    - 解决了一对一系统资源占用过多的问题

>***Q1：*****操作系统可以负责管理用户级线程吗？**
>
>***A2：***不负责，操作系统只管理内核级线程，也正因如此，**CPU的执行权是按照内核级线程为单位进行分配的**
>
>***Q2：*****更多的CPU核心数（逻辑核心数）意味着更多的内核级线程吗？**
>
>***A2：***不是，内核级线程的个数受限于操作系统支持的数量

### 3.2 线程调度

根据操作系统支持的线程级别不同（用户级线程或内核级线程），采用类似于进程调度的算法对线程进行调度

### 3.3 协程

协程是运行在**用户态**的轻量级的线程，多个协程会在同一个线程中交替执行，每个协程拥有单独的寄存器和程序计数器

由于线程是CPU调度的基本单位，且多个协程运行在一个线程中，因而**不存在内存可见性**的问题，因为同在一个CPU上，此外也不需要锁的机制，因为多个协程是互斥的在线程中执行的，所以对共享变量的修改**不存在竞争问题**

由于协程的寄存器和程序计数器均处于用户态，其调度由用户来决定，因而相较于线程有如下额外特征

- 对CPU透明
- 寄存器和程序计数器占用几十KB，上下文切换开销很小，且发生于同个CPU上
- 当协程内发生阻塞时，整个线程都会被阻塞

由于多个协程是运行于同一个线程中，为了利用CPU多核支持高并发，可以采用多进程+每个进程单线程多协程的方式，这时每个任务会分配给一个协程来完成