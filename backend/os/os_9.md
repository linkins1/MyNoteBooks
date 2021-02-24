## 9. 磁盘

### 9.1 结构

#### 9.1.1 基本概念

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/os/image-2021022215351233432.png" alt="image-20210222153533432" style="zoom:67%;" />

- 盘片

- 盘面

  盘片的正反两面或单面

- 磁道

  一个盘面的一个同心圆

- 柱面

  多个盘片垂直方向重合的的磁道的集合

- 扇区

  盘面被等分后的其中一个区域

- 磁头

  用于读取磁道信息

- 磁臂

  用于带动磁头寻找磁道

- 磁盘块

  一个磁道、一个扇区和一个盘面的交集

读写磁盘块需要经历如下步骤

- 根据柱面号移动磁臂找到对应磁道
- 根据盘面号激活对应磁头
- 旋转盘面到指定扇区并开始读写

#### 9.1.2 分类

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/os/image-20210222154345059.png" alt="image-20210222154328408" style="zoom:50%;" /> 

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/os/image-20210222154345059.png" alt="image-20210222154345059" style="zoom:50%;" /> 

### 9.2 磁盘调度算法

#### 9.2.1 读写时间

> 这里指读写磁盘一次需要耗费的时间

- 寻道时间

  磁臂带动磁头移动到指定磁道的时间Ts

  - 启动磁臂

    设为t1

  - 移动磁头

    设跨过一个磁道要δt，一共跨n个磁道

  Ts=t1+δt*n

- 延迟时间

  旋转盘面到指定扇区的时间

  设磁盘转速为r转/s，那么平均延迟时间Tr=1/2r

- 传输时间

  从磁盘读出/向磁盘写入所需的时间，设要读/写字节数为B，每个磁道的字节数为N，那么传输时间

  Tt=B/Nr

所以读写一次需要的总时间Ta=Ts+Tr+Tt

#### 9.2.2 先来先服务（FCFS）

![image-20210222161336191](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/os/image-20210222161331236191.png)

#### 9.2.3 最短寻找时间优先（SSTF）

![image-20210222161506268](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/os/image-20210222161506123268.png)

#### 9.2.4 扫描算法（SCAN）（反弹+碰壁）

![image-20210222161645289](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/os/image-20210222161645289123.png)

#### 9.2.5 LOOK算法（反弹+不碰壁）

![image-20210222161720258](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/os/image-20210222161720123258.png)

#### 9.2.6 C-SCAN（复位+碰壁）

![image-20210222161815968](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/os/image-20210222161815968.png)

#### 9.2.7 C-LOOK（复位+不碰壁）

![image-20210222161919873](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/os/image-20210222161231919873.png)

#### 9.2.8 减少延迟时间的方法

##### 1）问题

由于在读取一个磁盘块后需要一段时间来处理所读内容，所以如果要读取两个相邻的磁盘块时，就无法读入下一个相邻的磁盘块（因为在处理从上一个磁盘块读入的内容），那么就需要**再旋转一圈**才能读入下一个磁盘块，这就拉长了延迟时间

##### 2）解决方案

- 交替编号

  让逻辑上相邻的磁盘块在物理上间隔的分布

- 错位命名

  当要读取的数据是扇区号连续，但盘面不连续的情况，由于所有盘面是同时旋转的，所以可以让下方盘面相较于上方盘面沿磁头旋转方向移动一格，这样就留出处理时间而不需额外转动一圈盘面

  ![image-20210222164649002](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/os/image-20210222164649002.png)

- （柱面号，盘面号，扇区号）

  这样的顺序的好处在于，随着访问的磁盘块号的增加，当整个磁道都旋转过后，可以先不移动磁头，而是垂直方向上激活下方磁头继续旋转，在所有的垂直方向均访问结束后，再移动磁头（因为柱面号在最高位），之后再循环往复

  如下图，从0号盘面0号扇区绿色磁道开始开始，到1号盘面3号扇区蓝色磁道为止
  
  <img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/os/image-20210222164624036.png" alt="image-20210222164624036" style="zoom:67%;" />

### 9.3 磁盘管理

#### 9.3.1 初始化

![image-20210222164951653](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/os/image-20210222164951653.png)

#### 9.3.2 引导块

![image-20210222165419145](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/os/image-20210222165419145.png)

#### 9.3.3 坏块处理

![image-20210222165550881](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/os/image-20210222165512350881.png)



































