## 3.Mybatis关键技术

### 3.1数据库连接池

在doQuery之后，会先获取Connection对象，之后获取PrepareStatement对象用于执行sql

#### 3.1.1获得数据库连接对象

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/mybatis/image-stacktrace.png" alt="image-stacktrace" style="zoom:80%;" />

跟踪preparedStatement对象的构建的trace为上图，可以看出当使用POOLED模式时，使用的是构建preparedStatement对象使用的Connection对象通过PooledDataSource的getConnection方法得到，在深入这个方法体之前，首先来熟悉下面几个类

##### 1）PoolState

PoolState类中封装了两组PooledConnection集合，其中一组是idleConnections代表闲置的PooledConnection对象，另一组是activeConnections代表活跃的PooledConnection对象

```java
protected final List<PooledConnection> idleConnections = new ArrayList<>();
protected final List<PooledConnection> activeConnections = new ArrayList<>();
```

##### 2）PooledConnection

PooledConnection中存在三个重要的对象，其中两个是Connection对象，一个是PooledDataSource对象

###### （1）Connection对象

PooledConnection类中包含两个final的Connection对象，其中一个是真正的Connection对象realConnection，另一个是代理对象proxyConnection。

```java
public PooledConnection(Connection connection, PooledDataSource dataSource) {
  this.hashCode = connection.hashCode();
  this.realConnection = connection;
  this.dataSource = dataSource;
  this.createdTimestamp = System.currentTimeMillis();
  this.lastUsedTimestamp = System.currentTimeMillis();
  this.valid = true;
  this.proxyConnection = (Connection) Proxy.newProxyInstance(Connection.class.getClassLoader(), IFACES, this);
}
```

代理对象proxyConnection使用的InvocationHandler对象是this指针（PooledConnection类 ），这样在使用该代理对象执行方法时触发的是PooledConnection的invoke方法，方法体如下

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
String methodName = method.getName();
if (CLOSE.equals(methodName)) {
  dataSource.pushConnection(this);
  return null;
}
try {
  if (!Object.class.equals(method.getDeclaringClass())) {
    // issue #579 toString() should never fail
    // throw an SQLException instead of a Runtime
    checkConnection();
  }
  return method.invoke(realConnection, args);
} catch (Throwable t) {
  throw ExceptionUtil.unwrapThrowable(t);
}

}
```

###### （2）PooledDataSource对象

##### 3）getConnection方法

```java
public Connection getConnection() throws SQLException {
  return popConnection(dataSource.getUsername(), dataSource.getPassword()).getProxyConnection();
}
```

上面的getConnection方法首先调用的是popConnection方法返回一个PooledConnection对象，方法体如下

```java
private PooledConnection popConnection(String username, String password) throws SQLException {
  boolean countedWait = false;
  PooledConnection conn = null;
  long t = System.currentTimeMillis();
  int localBadConnectionCount = 0;

  while (conn == null) {
    synchronized (state) {
      if (!state.idleConnections.isEmpty()) {
        // Pool has available connection
        conn = state.idleConnections.remove(0);
        if (log.isDebugEnabled()) {
          log.debug("Checked out connection " + conn.getRealHashCode() + " from pool.");
        }
      } else {
        // Pool does not have available connection
        if (state.activeConnections.size() < poolMaximumActiveConnections) {
          // Can create new connection
          conn = new PooledConnection(dataSource.getConnection(), this);
          if (log.isDebugEnabled()) {
            log.debug("Created connection " + conn.getRealHashCode() + ".");
          }
        } else {
          // Cannot create new connection
          PooledConnection oldestActiveConnection = state.activeConnections.get(0);
          long longestCheckoutTime = oldestActiveConnection.getCheckoutTime();
          if (longestCheckoutTime > poolMaximumCheckoutTime) {
            // Can claim overdue connection
            state.claimedOverdueConnectionCount++;
            state.accumulatedCheckoutTimeOfOverdueConnections += longestCheckoutTime;
            state.accumulatedCheckoutTime += longestCheckoutTime;
            state.activeConnections.remove(oldestActiveConnection);
            if (!oldestActiveConnection.getRealConnection().getAutoCommit()) {
              try {
                oldestActiveConnection.getRealConnection().rollback();
              } catch (SQLException e) {
                /*
                   Just log a message for debug and continue to execute the following
                   statement like nothing happened.
                   Wrap the bad connection with a new PooledConnection, this will help
                   to not interrupt current executing thread and give current thread a
                   chance to join the next competition for another valid/good database
                   connection. At the end of this loop, bad {@link @conn} will be set as null.
                 */
                log.debug("Bad connection. Could not roll back");
              }
            }
            conn = new PooledConnection(oldestActiveConnection.getRealConnection(), this);
            conn.setCreatedTimestamp(oldestActiveConnection.getCreatedTimestamp());
            conn.setLastUsedTimestamp(oldestActiveConnection.getLastUsedTimestamp());
            oldestActiveConnection.invalidate();
            if (log.isDebugEnabled()) {
              log.debug("Claimed overdue connection " + conn.getRealHashCode() + ".");
            }
          } else {
            // Must wait
            try {
              if (!countedWait) {
                state.hadToWaitCount++;
                countedWait = true;
              }
              if (log.isDebugEnabled()) {
                log.debug("Waiting as long as " + poolTimeToWait + " milliseconds for connection.");
              }
              long wt = System.currentTimeMillis();
              state.wait(poolTimeToWait);
              state.accumulatedWaitTime += System.currentTimeMillis() - wt;
            } catch (InterruptedException e) {
              break;
            }
          }
        }
      }
      if (conn != null) {
        // ping to server and check the connection is valid or not
        if (conn.isValid()) {
          if (!conn.getRealConnection().getAutoCommit()) {
            conn.getRealConnection().rollback();
          }
          conn.setConnectionTypeCode(assembleConnectionTypeCode(dataSource.getUrl(), username, password));
          conn.setCheckoutTimestamp(System.currentTimeMillis());
          conn.setLastUsedTimestamp(System.currentTimeMillis());
          state.activeConnections.add(conn);
          state.requestCount++;
          state.accumulatedRequestTime += System.currentTimeMillis() - t;
        } else {
          if (log.isDebugEnabled()) {
            log.debug("A bad connection (" + conn.getRealHashCode() + ") was returned from the pool, getting another connection.");
          }
          state.badConnectionCount++;
          localBadConnectionCount++;
          conn = null;
          if (localBadConnectionCount > (poolMaximumIdleConnections + poolMaximumLocalBadConnectionTolerance)) {
            if (log.isDebugEnabled()) {
              log.debug("PooledDataSource: Could not get a good connection to the database.");
            }
            throw new SQLException("PooledDataSource: Could not get a good connection to the database.");
          }
        }
      }
    }

  }

  if (conn == null) {
    if (log.isDebugEnabled()) {
      log.debug("PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
    }
    throw new SQLException("PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
  }

  return conn;
}
```

上面的方法体执行流程如下：

- 先判断idleConnections中是否有剩余的PooledConnection对象，有的话就从中取出。

- 如果没有，则看是否activeConnections中的PooledConnection对象数量是否达到最大，如果没有则新建一个PooledConnection对象并返回，新建PooledConnection对象时传入的Connection对象是由UnpooledDataSource对象调用DriverManager.getConnection创建出的Connection对象。
- 如果activeConnections已满，那么从此集合中去除最老的一个PooledConnection对象，并查看此对象是否已经超时未使用，如果超时则将其从activeConnections中删除，之后查看此PooledConnection对象是否已提交，如果没有则将其回滚，如果提交，则取出PooledConnection中的真正的Connection对象并以此创建一个新的PooledConnection对象，并将原来的PooledConnection对象失效化
- 对于上面获取得到的PooledConnection对象，都会再返回之前先查看事务是否提交，如果没有则回滚，最终将此对象返回

返回了PooledConnection对象后，会调用getProxyConnection()方法返回代理Connection对象

#### 3.1.2代理Connection对象

在上面的popConnection方法中，我们会疑惑idleConnections中的对象是从何而来，通过查看代理对象使用的InvocationHandler的实现类PooledConnection重写的invoke方法可以发现，在业务执行完成后，调用close方法时，都会触发invoke方法中的`dataSource.pushConnection(this);`，这样就实现了将此PooledConnection从activeConnections中拉出，并放入idleConnections中或者直接放入idleConnections中。

当我们调用query，update方法时，invoke方法是通过`method.invoke(realConnection, args)`方法来实现原方法的调用，且使用的是realConnection对象执行此方法。（反射调用）

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
String methodName = method.getName();
if (CLOSE.equals(methodName)) {
  dataSource.pushConnection(this);
  return null;
}
try {
  if (!Object.class.equals(method.getDeclaringClass())) {
    // issue #579 toString() should never fail
    // throw an SQLException instead of a Runtime
    checkConnection();
  }
  return method.invoke(realConnection, args);
} catch (Throwable t) {
  throw ExceptionUtil.unwrapThrowable(t);
}

}
```

> **ps**:在调用pushConnection方法时，会先调用原来的PooledConnection对象的rollback方法，并取出其中真正的Connection对象并新建一个PooledConnection对象放入池中

### 3.2事务管理

#### 3.2.1POOLED

使用这种方式即是用Mybatis自带的数据库连接池来获取连接对象

>如果配置了dataSource为Druid，那么在openConnection时，使用的是Druid提供的Connection对象（从Druid的连接池中获取），而不会去Mybatis提供的PooledDataSource中去获取，那么Connection对象也就由Druid来管理

#### 3.2.2UNPOOLED

使用这种方式即每次执行方法时都会新建一个连接对象并在使用完销毁。

#### 3.2.3JNDI

使用这种方式会指定使用外部的数据库连接池


### 3.3多表查询

#### 3.3.1动态标签

##### 1）if

###### （1）例子

```xml
<select id="findByAddressIf" resultType="pojo.User" parameterType="pojo.User">
    select * from user where 1=1 
        <if test="add != null">
            and address like #{add}
        </if>
        <if test="username != null">
            and username = #{username}
        </if>
</select>
```

###### （2）属性

test属性写的是给定的parameterType的属性相关的判断条件

##### 2）where

对于上面的例子中写的where可以通过where标签替换为下面的代码，可以省去where 1=1的部分

```xml
<select id="findByAddressIf" resultType="pojo.User" parameterType="pojo.User">
    select * from user
    <where>
        <if test="add != null">
            address like #{add}
        </if>
        <if test="username != null">
            and username = #{username}
        </if>
    </where>
</select>
```

##### 3）foreach

###### （1）例子

- xml

```xml
<select id="findByCollection" resultType="pojo.User" parameterType="pojo.UserListCondition">
    <include refid="selectAllUser"></include>
    <where>
        <if test="idList != null and idList.size()!=0">
            <foreach collection="idList" item="id" separator="," open="id in (" close=")">
                #{id}
            </foreach>
        </if>
    </where>
</select>
```

- UserListCondition

  ```java
  public class UserListCondition {
      private List<Integer> idList;
  
      public List<Integer> getIdList() {
          return idList;
      }
  
      public void setIdList(List<Integer> idList) {
          this.idList = idList;
      }
  }
  ```

- 测试方法

  ```java
  @Test
  public void findUserByCollectionXML(){
      UserDaoXML sqlSessionMapper = sqlSession.getMapper(UserDaoXML.class);
      UserListCondition userListCondition = new UserListCondition();
      List<Integer> ids = new ArrayList<>();
      ids.add(41);
      ids.add(42);
      ids.add(43);
      userListCondition.setIdList(ids);
      List<User> users = sqlSessionMapper.findByCollection(userListCondition);
      for (User user : users) {
          System.out.println(user);
      }
  }
  ```

###### （2）属性

collection:代表要遍历的集合元素，注意编写时不要写#{} 

open:代表语句的开始部分 

close:代表结束部分

item:代表遍历集合的每个元素，生成的变量名 

sperator:代表分隔符

这样对应的上面的sql语句为`id in (41,42,43)`

> 由于在之前配置了公用的sql语句如下
>
> ```xml
> <sql id="selectAllUser">
>  select * from user
> </sql>
> ```
>
> 所以`<include refid="selectAllUser"></include>`标签等效于select * from user

#### 3.3.2多表查询

##### 1）一对一查询

###### （1）xml文件

```xml
<resultMap id="AccountUser" type="pojo.UserWithAccount">
    <id property="id" column="aid"></id>
    <result property="uid" column="id"></result>
    <result property="money" column="MONEY"></result>
    <association property="user" javaType="pojo.User">
        <id property="id" column="id"></id>
        <result property="username" column="username"></result>
        <result property="gender" column="gender"></result>
        <result property="birthday" column="birthday"></result>
        <result property="address" column="address"></result>
    </association>
</resultMap>
```



```xml
<select id="findAccountUser" resultMap="AccountUser">
    select a.ID aid,a.UID,a.MONEY,u.* from account a left join user u on a.UID = u.id
</select>
```

其中使用association标签来实现一对一的注入

###### 2）association标签

- association的property

  resultmap的type类中的成员变量名

- javaType

  association的property对应的java类

- id

  代表查询到的来自表中的主键，其中的property代表javaType类中的成员变量名，column代表表中的字段

- result

  代表非主键字段

##### 2）一对多/多对多查询

###### （1）xml

```xml
<resultMap id="UserAccount" type="pojo.UserWithAccount">
    <id property="id" column="id"></id>
    <result property="username" column="username"></result>
    <result property="gender" column="gender"></result>
    <result property="birthday" column="birthday"></result>
    <result property="address" column="address"></result>
    <collection property="accountList" ofType="pojo.Account">
        <id property="id" column="aid"></id>
        <result property="money" column="MONEY"></result>
    </collection>
</resultMap>
```

###### （2）collection标签

- collection的property

  resultmap的type类中的成员变量名

- ofType

  collection的property对应集合中需要的java类

