## 3.Redis数据结构

> Redis中所有的数据都是以k-v对的形式存在，支持多种数据类型的位置是value
>
> 全部的命令见[官方文档](http://redisdoc.com/)

### 3.1key

#### 3.1.1 类型

redis中key都是以字符串的形式存在

#### 3.1.2常用命令

| 命令              | 含义                                             |
| ----------------- | ------------------------------------------------ |
| set/get key value | 向库中添加/获取                                  |
| del key           | 从库中删除某个key                                |
| keys *            | 查看所有key                                      |
| exists key        | 判断某个key是否存在                              |
| move key 库编号   | 将本库中的key挪到指定库编号的库中                |
| expire key 秒钟   | 为给定的key设置过期时间                          |
| ttl key           | 查看还有多少秒过期，-1表示永不过期，-2表示已过期 |
| type key          | 查看你的key对应的value是什么类型                            |

### 3.2value

#### 3.2.1String  （一对一）

##### 1）定义

String是redis最基本的类型，**一个key对应一个value**。string类型是**二进制安全**的，意思是redis的string可以包含任何数据。比如jpg图片或者序列化的对象 。string类型是Redis最基本的数据类型，一个redis中字符串value**最多可以是512M**

##### 2）常用命令

| 命令                                                         | 含义                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| append key xxx                                               | 在当前key的value中附加xxx                                    |
| strlen key                                                   | 查看key对应的value的长度                                     |
| Incr key/decr key/incrby key 值/decrby key 值,一定要是数字才能进行加减 | incr和decr是对指定key的value加一/减一   incrby是对key的value增加指定值，decrby同理 |
| getrange key offset len                                      | 获取key的value，从offset索引位置开始，len长度                |
| setrange key offset value                                    | 设定key的value，从offset开始，将原value覆写为value的值       |
| setex(set with expire) key seconds value                     | 设定某个键值对，过期时间为指定的秒数                         |
| setnx(set if not exist) key  value                           | 当某个键不存在时，设定键值对                                 |
| mset/mget/msetnx k1 v1 k2 v2 k3 v3...                        | 批量设定/获取/nx设定键值对                                   |
| getset(先get再set) key value                                 | 先获取key原来的值，再设定值                                  |

#### 3.2.2hash （一对多）

##### 1）定义

Hash（哈希）是一个键值对集合。Redis hash是一个string类型的field和value的映射表，hash特别适合用于存储对象。类似Java里面的Map<String,Object> 

举下面的例子

`Map<String,Object> key1 = new HashMap();`

此处key1对应redis中的一条k-v对中的k，v是一个map。在获取某个数据时，必须给定redis的key(Map对象引用)和hash的key(Map中的key)

> 在操作时，如下
>
> ![image-20201008102156985](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/redis/image-20201008102156985.png)
>
> 在此处，field即为hash的key ；key即为redis的key 

##### 2）常用命令

| 命令                               | 含义                                               |
| ---------------------------------- | -------------------------------------------------- |
| hset/hget/hmset/hmget/hgetall/hdel | 设定/获取/设定多个/获取多个/获取所有/删除 hash     |
| hlen                               | 获取hash中有多少个key                              |
| hexists key                        | 在key里面的某个值的key                             |
| hkeys/hvals                        | 获取hash中所有的key/value                          |
| hincrby/hincrbyfloat               | 只有在value是int/float时才可以操作                 |
| hsetnx                             | 如果k下的某个hash的k-v对不存在，则设定，否则不设定 |

#### 3.2.3list （一对多）

##### 1）定义

List（列表）是简单的字符串列表，按照插入顺序排序。你可以添加一个元素导列表的头部（左边）或者尾部（右边）。它的底层实际是个链表

可以只管的理解为，redis中的key就是一个链表的引用，value就是一个链表

`List<T> k = new LinkedList<>();`

##### 2）常用命令

| 命令                  | 含义                                                         |
| --------------------- | ------------------------------------------------------------ |
| lpush/rpush key value | 从左侧/右侧插入对key指定的list插入节点value                  |
| lrange key start end  | 获取链表key从start到end的所有值                              |
| lpop/rpop             | 从list的左侧/右侧弹出点                                      |
| lindex key index      | 通过索引获取列表中的元素                                     |
| llen                  | 获取链表的长度                                               |
| lrem key count value  | 从左侧开始删除count个value，如果count超过实际个数，则删除链表中的所有value |
| ltrim key start end   | 将链表截断为start到end的部分                                 |
| rpoplpush source dest | 将source的数据从右侧弹出再从左侧插入到dest，结束后dest中的顺序和source一致 |
| lset key index value  | 设定index处为value                                           |

> 由于是linkedlist形式，所以头插尾插都很方便，中间插入和随机访问较慢

#### 3.2.4set（一对多）

####  1）定义

Set（集合）是string类型的无序集合。它是通过HashTable实现实现的

`Set<T> k = new HashSet<>();`

##### 2）常用命令

| 命令                    | 含义                                                         |
| ----------------------- | ------------------------------------------------------------ |
| sadd/smembers/sismember | 向集合中插入值/获取集合中所有值/判断某个值是不是集合中的元素 |
| scard                   | 获取集合里面的元素个数                                       |
| srem key value...       | 删除集合中的某些元素                                         |
| srandmember key count   | 从集合中随机获取count个值                                    |
| spop key count          | 从集合中随机弹出count个值                                    |
| smove key1 key2 member  | 将key1集合中的某个member移动到key2集合                       |
| sdiff key1 keys...      | 在key1集合中而不在keys集合中的项                             |
| sinter key1 keys...     | 获取key1和keys所有的交集                                     |
| sunion                  | 获取并集                                                     |

#### 3.2.5zset（一对多）

##### 1）定义

zset(sorted set：有序集合) 和 set 一样也是string类型元素的集合,且不允许重复的成员。不同的是每个元素都会关联一个double类型的分数。redis通过分数来为集合中的成员进行从小到大的排序。zset的成员是唯一的,但分数(score)却可以重复。举例如下

| 元素 | score |
| ---- | ----- |
| zs1  | 0.1   |
| zs2  | 20    |
| ...  | ...   |

##### 2）常用命令

| 命令                                  | 含义                               |
| ------------------------------------- | ---------------------------------- |
| zadd key score1 v1 score2 v2 ...      | 向zset中加入多个分数-元素对        |
| zrange key start end                  | 获取zset中从start到end的所有元素   |
| zrangebyscore key 开始score 结束score | 获取开始score到结束score的所有元素 |
| zcard                                 | 获取集合中元素个数                 |
| zcount key 开始分数区间 结束分数区间  | 获取分数区间内元素个数             |
| zrank                                 | 获取value在zset中的下标位置        |
| zscore                                | 按照值获得对应的分数               |
| 上面的操作都对应一rev(反向操作)       | ...                                |
