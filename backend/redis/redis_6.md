## 6.Redis主从复制及集群

> 本文借鉴自[半路雨歌的掘金](https://juejin.im/post/6844904097116585991)

### 6.1主从复制

![170f23858652a465](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/redis/image-170f23858652a465.png)

#### 6.1.1 概念

##### 1）定义

通过设定主机和多个从机 ，并将从机挂载在主机上，将主机输出的.rdb文件存入从机中用做大量数据的备份。

这样可以主要对主机进行写操作，对从机进行读操作，这样就可以减缓主机的读压力，达到**读写分离**。

此外，在主机发生故障时，可以通过读取从机中的备份来完成数据恢复

##### 2）原理

###### （1）建立连接

- 读取配置文件或执行slaveof指令，创建socket与主服务器通信
- 从服务器向主服务器发送ping，正常状态下，主回复pong
- 如果主服务器开启了密码（requirepass），则从服务器需要设定masterauth指定密码，或者使用auth+密码
- 从服务器发送监听端口，主服务器记录

###### （2）数据同步

:one: 初次同步

在初次同步时一般都会触发**完整重同步**，从机向主机发送SYNC命令，主机收到后触发bgsave，将最新的完整的rdb文件发往所有从服务器，从服务器将本地的rdb文件更新为最新的文件

:two: 断连重连

由于在进行同步时，从服务器会保留主服务器的id和offset，如果主服务器和从服务器重建连接后这两个信息匹配，而且主机中的记录没有丢失，那么会接着上一次传输的位置继续进行同步，这称为**部分重同步**；如果前面的信息出现偏差，那么会触发**完整重同步**

#### 6.1.2 配置

*下面配置以一主二从来举例*

##### 1）拷贝多份redis-xxx.conf

为每一个主机都创建一份单独的配置文件并编号，需要修改log文件名称和rdb文件的名称。如果在一台机器上模拟，需要修改port参数

##### 2）启动多台机器

可以通过info replication来查看当前的主从信息。在刚启动时，每一台机器都是master

##### 3）配置主从关系

###### （1）手动配置

在从机上通过`salveof 主机ip 主机端口`指令来手动完成主从关系指定，在下次开机时，关系解除

###### （2）配置文件

通过在配置文件中添加`replicaof 主机ip 主机端口`，这样可以实现永久的主从关系配置

#### 6.1.3 主从关系维系

##### 1）断连

如果断开连接，从机仍然会保持slave的角色，等待主机重新启动再次建立连接

##### 2）接龙

可以为从机设定从机，这样就可以实现链式传递，被挂在从机的从机仍然会保持slave的身份（info replication查看），但是相对于其管理的从机，是主机的角色

##### 3）解除主从关系

在从机执行`slaveof no one`来解除和主机的关系

### 6.2哨兵模式（Sentinel Mode）

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/redis/image-170f23893704295e.png" alt="image-170f23893704295e" style="zoom:80%;" />

#### 6.2.1基本概念

##### 1）定义

哨兵模式是用于解决在主从之间断连时，从机持续等待停止工作的问题。通过设定redis对某个主机进行监控，在其宕机时，会立即查看其挂载的从机的票数（通过）来选取票数最高的作为新的主机。当旧主机再次回来时，会成为从机挂载到新的主机上。

##### 2）哨兵工作

###### （1）建立两条连接

:one: 一条连接用于获取master频道及其他在该频道下的哨兵信息

:two: 一条连接用于定期向master发送info命令来查看master的信息

###### （2）同步

:one: 定时向master和slave发送info命令

:two: 定期向master和slave的频道发送自己的信息

:three: 定期向master和slave发送ping，检查是否下线

#### 6.2.2 配置

##### 1）sentinel.conf

在redis.conf的同目录下创建sentinel.conf文件，并添加`sentinel monitor 被监控的名字(自定义) 被监控的IP 被监控的PORT votes`。同一个sentinel可以监控多个机器，因而可以设定多个被监控的IP和端口

##### 2）指定sentinel.conf文件并执行

通过命令`redis-server sentinel1.conf --sentinel`来指定启动redis时使用的哨兵配置文件

### 6.3集群

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/redis/image-170f238bca333f96.png" alt="image-170f238bca333f96" style="zoom:80%;" />

#### 6.3.1基本概念

##### 1）水平扩容与垂直扩容

###### （1）主从复制与集群

由于主从复制+哨兵模式只是将容量进行**垂直扩容**，整体的存储容量仍然**受制于平均每台机器的容量**。为了能够进行**水平扩容**，真正提升一个存储节点的容量，**并**更好的完成故障自动转移，达到高可用，引入了集群的概念。

###### （2）对比

垂直扩容（主从复制）可以增强由主从机构成个一个节点的**可靠性**，可以更好的支持高效的读写操作。

水平扩容（集群）可以增加集群整体的容量，且可以更好的支持高可用，作为一个节点提高了整体**的有效性**。

##### 2）集群

###### （1）定义

集群即位将多个机器划分为一组成为一个集群，这些机器共同完成数据存储，实现容量扩增。集群中的机器之间地位平等，也可以设定主从关系增加可靠性。

###### （2）Cluster-mode

对于Redis中的Cluster-mode，其采用无中心化的结构，有如下特征

:one: 所有的redis节点彼此互联(PING-PONG机制),内部使用二进制协议优化传输速度和带宽

:two: 节点的fail是通过集群中超过半数的节点检测失效时才生效

:three: 客户端与redis节点直连，不需要中间代理层。客户端不需要连接集群所有节点，连接集群中任何一个可用节点即可

其具体的工作机制如下

:one: 在Redis的每个节点上，都有一个**插槽**（slot），取值范围为0-16383

:two: 当我们存取key的时候，Redis会根据CRC16的算法得出一个结果，然后把结果对16384求余数，这样每个key都会**对应一个编号在0-16383之间的哈希槽，通过这个值，去找到对应的插槽所对应的节点**，然后直接自动跳转到这个对应的节点上进行存取操作

:three: 为了保证高可用，Cluster模式也引入主从复制模式，一个主节点对应一个或者多个从节点，当主节点宕机的时候，就会启用从节点

:four: 当其它节点ping一个节点A时，如果半数以上的节点与A通信超时，那么认为节点A宕机了。如果节点A和它的从节点都宕机了，那么该集群就无法再提供服务了

#### 6.3.2配置

*此处演示6主6从的集群*

##### 1）创建node-xxx.conf文件

对6个节点分配创建一个node-xxx.conf文件作为每个节点使用的集群配置文件，不用手动进行配置，由系统来自动进行维护

> 可以通过`cluster-config-file 文件名`在对应的redis.conf中指定node文件名

##### 2）分别启动

将6个redis分别指定相应的redis.conf文件来启动

##### 3）划分集群

```shell
redis-cli --cluster create --cluster-replicas 1 127.0.0.1:7100 127.0.0.1:7200 127.0.0.1:7300 127.0.0.1:7400 127.0.0.1:7500 127.0.0.1:7600 -a passw0rd
```

其中cluster-replicas后的参数是指每个主机挂载几个从机，系统会自动进行主从关系构造

#### 6.3.3启动

通过`redis-cli -c -p 任意端口号`后便以集群模式进入，这样在存储数据时，会自动跳转到存储到的slot所在的redis主机上；如果不用-c参数，必须跳转到相应的redis主机中再进行操作

#### 6.3.4 优缺点

##### 1）优

- 无中心架构，数据按照slot分布在多个节点，且集群中的每个节点都是平等的关系，从任意一个节点都可以获取到整个集群中存储的内容
- 能够实现自动故障转移
- 能够实现水平扩展

##### 2）劣

- slave充当“冷备”，不能缓解读压力
- 批量操作限制，目前只支持具有相同slot值的key执行批量操作，对mset、mget、sunion等操作支持不友好
- 事务操作支持有限，只支持多key在同一节点的事务操作，多key分布不同节点时无法使用事务功能
