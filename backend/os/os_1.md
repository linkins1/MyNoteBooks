## 1.操作系统概述

### 1.1基本概念

#### 1.1.1 定义

是一种软件，向下采用高效的方式对硬件资源进行管理、调度；向上提供对系统资源的调用接口，并将上层请求的资源合理的分配在硬件上。最终目的是为了让用户可以更方便高效的操作硬件资源，是利用软件的方式对硬件系统进行增强

#### 1.1.2 角色

##### 1）向下

对资源进行管理，主要是对下面四个方面

- 处理机管理
- 存储器管理
- 文件管理
- 设备管理

##### 2）向上

提供如下接口

- GUI

- 命令接口

  - 联机命令接口：也称为交互式命令接口，用户输入一个命令，系统执行一个命令（如在shell终端输入ls）
  - 脱机命令接口：也称为批处理命令接口，用于用户输入一些命令，操作系统批量执行（类比于.sh）

- 系统调用：操作系统作为用户和计算机硬件之间的接口，需要向上提供一些简单易用的服务。主要包括**命令接**
  **口和程序接口**。其中，**程序接口由一组系统调用组成**。

  **系统调用主要为了让用户能请求内核态的服务**

  如socket通信时使用read函数从缓冲区读出数据到指定缓存，read就会触发系统调用，printf也会触发系统调用

  下图引自[《现代操作系统》]()，其中的陷入内核步骤，需要调用TRAP指令（也称访管指令）从**用户态切换到内核态**

  <img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/os/image-20201021201354175.png" alt="image-20201021201354175" style="zoom:67%;" />

> **Q1.库函数与系统调用的区别**
>
> 库函数中涉及系统调用的函数接口是将系统调用进行了封装，如上面的read函数，其中封装了执行TRAP指令的过程
>
> 系统调用是操作系统提供给上层的接口，如TRAP指令
>
> **Q2.系统调用的应用**
>
> 设备管理/文件管理/进程控制/进程通信/内存管理

### 1.2 特征

#### 1.2.1 并发

**指两个或多个事件在同一时间间隔内发生**。这些事件**宏观上是同时**发生的，但**微观上是交替**发生的。
操作系统中同时运行的进程在单核状态下，是并发执行的；在多核状态下，是并行执行的（"真"同时）

#### 1.2.2 共享

共享即资源共享，是指系统中的资源可供内存中多个并发执行的进程**共同使用**。

共享方式分下面两种

- 互斥共享方式：多个进程互斥的共享（打印机资源）
- 同时共享方式：多个进程"同时"的共享（硬盘资源）

**共享是并发的前提，并发是共享的目的**

#### 1.2.3 虚拟

是指将物理上存在的内存资源通过复用的方式提供给多个进程使用，从而让用户感知到资源的扩充

- 空分复用：通过划分一部分磁盘资源供给RAM，从而达到扩充RAM的效果
- 时分复用：通过并发实现，多个进程时分复用一块内存，从而提高内存的利用率

#### 1.2.4 异步

异步指在并发的状态下，由于操作系统调度算法的作用，各个进程会切换着执行。

### 1.3 分类

#### 1.3.1 手工操作

人工送入纸带的方式输入和输出，中间由计算机处理

#### 1.3.2 批处理阶段

##### 1）单道批处理系统

通过引入外围机，通过同时读入多个作业（纸带），输出到磁带中，并交由计算机处理。加快了IO操作，减少了CPU的等待时间

##### 2）多道批处理系统

通过并发的输入和输出，使得CPU不会在IO操作阶段做等待操作，进一步提升了CPU使用效率和IO效率

#### 1.3.3 分时操作系统

计算机**以时间片为单位轮流**为各个用户/作业服务，各个用户可通过终端与计算机进行交互。

各个用户可以实现分别和计算机进行独立的交互

#### 1.3.4 实时操作系统

在分时操作系统的基础上增加了任务的优先级，可以立即对高优先级的任务做出响应，进一步提高了响应速度，保证用户体验

#### 1.3.5 个人操作系统

windows/linux

#### 1.3.6 嵌入式操作系统

linux等

### 1.4 体系架构

> 摘自王道考研

![image-20201021205246495](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/os/image-20201021205246495.png)

#### 1.4.1 内核

如上图所示，操作系统内核是操作系统的核心，**其中包含的功能是对硬件资源最直接最基础的操作。**

##### 1）分类

###### （1）单体

将上图所示的**所有内核功能都集于内核**中的即为大内核

###### （2）微内核

将内核中的不同功能进行划分，**将最重要的功能划分让内核成为微内核**，其他功能**不太重要的放入用户态**，作为普通用户进程运行

##### 2）对比

单体内核可以减少用户态和内核态之间的切换次数，提高内核运行效率

微内核可以压缩内核大小，减少内核出现错误的几率，通过牺牲不太重要的内核功能（放入用户态）来提高微内核的鲁棒性

#### 1.4.2 指令与处理机状态

- 特权指令

  运行在核心态

- 非特权指令

  运行在用户态

### 1.5 中断

#### 1.5.1 定义

用户态切换到内核态通过中断可以实现状态的转换，从而实现操作系统对CPU的占用

- 内核态->用户态

  操作系统在完成内核作业后，执行特权指令，主动让出CPU

- 用户态->内核态

  通过外部中断和内部中断，使得操作系统抢夺回对CPU的使用权

通过PCB中的PSW位来标识程序当前处于什么状态

#### 1.5.2 分类

##### 1）内部中断（异常）

###### （1）定义

与当前执行的指令有关，中断信号来源于CPU内部（如执行TRAP指令），具体可分为如下几类

- 陷入 TRAP
- 故障 fault，如缺页中断
- 终止 abort

###### （2）执行时机

在指令执行过程中立即执行

##### 2）外部中断（中断）

###### （1）定义

与当前执行的指令无关，中断信号来源于CPU外部（如插入U盘），具体分如下几类

- 时钟中断
- IO中断

###### （2）执行时机

在指令周期结束后检查是否有外部中断要执行

#### 1.5.3 中断向量表

其中存储了中断号和中断处理程序之间的对应关系，这样操作系统就可以根据中断号查询中断向量表来执行对应的中断处理程序