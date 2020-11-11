## 4.Mybatis加载与缓存

### 4.1加载

#### 4.1.1延迟加载

> **核心思想**：将一个复杂的含有join的sql语句进行拆分，将两个部分分别写为单独的sql，并通过association或collection标签中的select属性、column属性(select属性对应的方法把column字段当作方法的参数值)和fetchType属性来将两个独立的sql语句拼接，完成等效于join的效果
>
> 或者理解为先做一次查询，得到了**一组结果**，之后再根据这组结果去进行第二次查询，且第二次查询的**时机为需要时**才查询

##### 1）定义

在需要用到数据时才进行加载，不需要用到数据时就不加载数据。延迟加载也称**懒加载(*LAZY*)**

##### 2）优势 

先从单表查询，**需要时再**从关联表**去关联查询**，大大提高数据库性能，因为查询单表要比关联查询多张表速度要快。

##### 3）劣势

只有**当需要用到数据时，才会进行数据库查询**，这样在大批量数据查询时，因为查询工作也**要消耗时间**，所以可能造成用户等待时间变长，造成用户体验下降。

##### 4）使用方法

###### （1）把collection或association标签中的fetchType属性置为fetchType="lazy"

###### （2）在sql-config.xml中配置`<setting name="lazyLoadingEnabled" value="true"/>`

##### 5）示例

```xml
<select id="findById" resultType="pojo.User" parameterType="INT">
        select * from user where id=#{id};
</select>
<!--一对一-->
<resultMap id="AccountUserLoad" type="pojo.Account">
    <id property="id" column="id"></id>
    <result property="uid" column="uid"></result>
    <result property="money" column="MONEY"></result>
    <association property="user" javaType="pojo.User" column="uid" select="findById">
    </association>
</resultMap>
<select id="findAccountUserLoad" resultMap="AccountUserLoad">
    select * from account
</select>
<!--一对多-->
<select id="findByIdAccount" parameterType="int" resultType="pojo.Account">
    select * from account where UID=#{id}
</select>
<resultMap id="UserAccountLoad" type="pojo.UserWithAccount">
    <id property="id" column="id"></id>
    <result property="username" column="username"></result>
    <result property="gender" column="gender"></result>
    <result property="birthday" column="birthday"></result>
    <result property="address" column="address"></result>
    <collection property="accountList" ofType="pojo.Account" select="dao.UserDaoXML.findByIdAccount" column="id">
    </collection>
</resultMap>
<select id="findUserAccountLoad" resultMap="UserAccountLoad" useCache="true">
    select * from user
</select>
```

> collection标签中指定的select属性的值对应的方法的返回值需要是**集合类型**

#### 4.1.2立即加载

##### 1）定义

在查询时将所有查询到的数据一并查出并进行加载(*<u>**EAGER**</u>*)

##### 2）优势 

可以立即得到需要查到的所有数据

##### 3）劣势

在关联数据量非常大且不需要全部关联数据时，会导致**加载时间长**的问题

### 4.2缓存

#### 4.2.1一级缓存

*一般来说，一级缓存是指**sqlSession对象中的缓存区域**。*

##### 1）定义

sqlSession对象的缓存实际是Executor(**BasedExecutor**)中的缓存(localCache对象)，且保存缓存数据的缓存对象(localCache)属于**PerpetualCache**类(实现了Cache接口)，缓存的存储格式是以**Map**形式存在

##### 2）缓存作用域

一级缓存只在一个sqlSession对象中保留，那么在进行两次相同的查询时，第二次就会从缓存中获取已查询到的值。一旦缓存被清除，那么就会进行两次sql查询

##### 3）缓存清除

通过SqlSession(**DefaultSqlSession**)对象clearCache方法时会调用对象包含的Executor(**BasedExecutor**)对象的clearLocalCache方法，或者调用update、commit、rollback、query方法会调用BasedExecutor中的update、commit、rollback、符合条件的query也会调用clearLocalCache将缓存清除

#### 4.2.2二级缓存

##### 1）定义

二级缓存是共享于不同的sqlSession对象之间的缓存域，也即一个sqlSession的结束不影响另一个sqlSession对象从中取值

##### 2）缓存作用域

二级缓存共享于不同的sqlSession对象。如果在sql-config.xml中配置了**cacheEnabled为true(默认为true)**，那么会新建一个**CachingExecutor**对象并返回，CachingExecutor中的**TransactionalCacheManager**对象中存在一个Map对象如下：

`private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap<>();`

这个对象的key是Cache对象，value是TransactionalCache对象，这样可以根据不同的Cache对象获取不同的TransactionalCache对象，缓存的数据也是存在于这个对象中。也即，**TransactionalCache**对象为二级缓存，其内部也是通过**Map**集合来完成缓存数据的存储

>**TransactionalCache的注释**
>
>The **2nd level** cache transactional buffer.
>This **class holds all cache entries that are to be added to the 2nd level cache during a Session**. Entries are **sent to the cache** **when commit is called or discarded** if the Session is rolled back. Blocking cache support has been added. Therefore any get() that returns a cache miss will be followed by a put() so any lock associated with the key can be released

##### 3）使用步骤

###### （1）在sql-config.xml中配置`<setting name="cacheEnabled" value="true"/>`

###### （2）在对应mapper.xml的对应的mapper标签中添加`<cache/>`

###### （3）把select标签中的useCache标签设置为true，`useCache="true"`

##### 4）注意

当我们在使用二级缓存时，所缓存的类一定要实现**java.io.Serializable**接口，这种就可以使用序列化方式来保存对象。

