## 2.Redis概述

### 2.1 基本概念

#### 2.1.1定义

**Redis**即**RE**mote **DI**ctionary **S**erver(远程字典服务器)是一个高性能的(key/value)分布式内存数据库，基于内存运行
并支持持久化的NoSQL数据库。

#### 2.1.2特性

##### 1）持久化 

Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用

##### 2）多种数据类型

Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储

##### 3）备份

Redis支持数据的备份，即master-slave模式的数据备份，此外还支持集群

### 2.2Redis安装（Docker）

#### 2.2.1获取安装包

docker pull redis

#### 2.2.2配置redis.conf文件

##### 1）获取配置文件

由于使用docker安装redis不会自带配置文件，需要从[官网](http://download.redis.io/redis-stable/redis.conf)下载

##### 2）创建目录

为了让docker中的redis中的配置文件使用宿主机的配置文件，需要先创建一个目录来存放配置文件，并将这个目录挂载到容器中的目录，这样容器中的目录会自动复制宿主机目录下的数据

##### 3）修改配置文件

主要修改如下几项

###### （1）bind 127.0.0.1

将这段注释掉以便于远程连接

###### （2）protected-mode no

改为yes，方便远程连接

###### （3）port 6379

当启用不同的redis容器时，需要修改此端口

###### （4）daemonize no

在docker环境下此项不可以改为yes，会导致容器启动失败。可以在run时使用-d来后台运行容器

###### （6）requirepass 123456

当打开权限验证后，之后进入redis时需要使用auth + 123456的方式来完成权限验证

#### 2.2.3启动容器

执行下面命令

```shell
docker run -itd --name redis-1 -v /home/redis-conf/redis.conf:/etc/redis/redis.conf -v /home/redis-conf/data:/data -p 6379:6379 redis redis-server /etc/redis/redis.conf --appendonly yes
```

其中

- **-d:** 后台运行容器，并返回容器ID；
- **-i:** 以交互模式运行容器，通常与 -t 同时使用；
- **-p:** 指定端口映射，格式为：**主机(宿主)端口:容器端口**
- **-t:** 为容器重新分配一个伪输入终端，通常与 -i 同时使用
- **-v**: 给容器挂载存储卷，挂载到容器的某个目录 
- **--name**：为容器起别名

最后添加image的名称，此处为redis，之后的redis-server是其中redis的命令，后面的目录是刚刚挂载的目录，其中存放了配置文件，最后的appendonly yes是指开启.aof文件的数据，是一种持久化的策略，也可以不开启

> 其他常用的docker命令[引用](https://www.jianshu.com/p/67fc4b1cbe1b)
>
> 启动docker   service docker start
>
> 查看docker 状态，确认是否启动   service docker status
>
> 查看所有镜像 docker images
>
> 删除镜像(会提示先停止使用中的容器) docker rmi  镜像name/镜像id
>
> 查看所有容器 docker ps -a
>
> 查看正在运行的容器docker ps(docker container ls)
>
> 查看容器运行日志 docker logs 容器名称/容器id
>
> 停止容器运行 docker stop 容器name/容器id
>
> 终止容器后运行 docker start 容器name/容器id
>
> 容器重启 docker restart 容器name/容器id
>
> 删除容器 docker rm 容器name/容器id
>
> 进入mysql容器 docker exec -it 容器名称 bash
>
> 查看容器使用的IP  docker inspect --format='{{.NetworkSettings.IPAddress}}'  容器ID

### 2.3 Redis常用命令

#### 2.3.1默认脚本

默认安装目录在usr/local/bin，其中有如下脚本
:one:	redis-benchmark:性能测试工具

:two:	redis-check-aof：修复有问题的AOF文件，rdb和aof后面讲

:three:	redis-check-dump：修复有问题的dump.rdb文件

:four:	redis-cli：客户端，操作入口 ，加-p参数可以指定端口号

:five:	redis-sentinel：redis集群使用

:six:	redis-server：Redis服务器启动命令

:seven:	redis-cli shutdown：关闭redis

#### 2.3.2常用命令

:one:	select：切换数据库（0-15，默认16个库）

:two:	dbsize：查看当前数据库的key的数量

:three:	flushdb：清空当前库

:four:	Flushall：清空全部库

:five:	Shutdown：停止当前redis