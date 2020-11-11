## 5.Mybatis注解

### 5.1基本CRUD注解

#### 5.1.1@Select

##### 1）原xml写法

```xml
<select id="findAllUserWithXML" resultType="pojo.User">
    select * from user;
</select>
```

##### 2）注解写法

```java
@Select("select * from user")
List<User> findAllUserWithAnn();
```

#### 5.1.2@Insert

##### 1）原xml写法

##### 2）注解写法

与@Select注解相同

#### 5.1.3@Update

##### 1）原xml写法

##### 2）注解写法

与@Select注解相同

#### 5.1.4@Delete

##### 1）原xml写法

##### 2）注解写法

与@Select注解相同

#### 5.1.5 @Param

这个注解有如下应用场景

##### 1）方法有多个参数

当mapper中的接口方法存在多个参数时，需要对每个参数都添加@Param注解，如下所示

###### 1）Mapper

```java
Integer updateUserface(@Param("url") String url, @Param("id") Integer id);
```

###### 2）xml

```xml
<update id="updateUserface">
    update hr set userface = #{url} where id=#{id};
</update>
```

##### 2）xml中使用$取值

由于存在sql注入问题，一般只有在order by获取列名时使用

###### 1）Mapper

```java
List<User> getAllUsers(@Param("order_by")String order_by);
```

###### 2）xml

```xml
<select id="getAllUsers" resultType="org.javaboy.helloboot.bean.User">
    select * from user
    <if test="order_by!=null and order_by!=''">
        order by ${order_by} desc
    </if>
</select>
```

##### 3）动态sql

当xml中将mapper方法的某个参数作为了动态判断条件，那么即使只有一个参数也需要添加@Param

###### 1）Mapper

```xml
List<User> getUserById(@Param("id")Integer id);
```

###### 2）xml

```xml
<select id="getUserById" resultType="org.javaboy.helloboot.bean.User">
    select * from user
    <if test="id!=null">
        where id=#{id}
    </if>
</select>
```

> @Param部分借鉴自[此贴](https://juejin.im/post/6844903894997270536)

### 5.2映射注解

#### 5.2.1@Results

可以与@Result一起使用，封装多个结果集 

#### 5.2.2@Result

实现结果集封装 

##### 1） @One（代替了`<assocation>`标签）

实现一对一结果集封装

###### （1）select 

指定用来多表查询的sqlmapper

###### （2）fetchType

会覆盖全局的配置参数lazyLoadingEnabled，确定加载时机(**FetchType.*LAZY***)

> 可以不像xml中指定javaType的值，由于可以通过反射得到类型

##### 2）@Many（代替了`<Collection>`标签）

实现一对多结果集封装

> 属性同@One，且可以不像xml中指定ofType的值，由于可以通过反射得到类型

#### 5.2.3@ResultMap

引用@Results定义的封装

#### 示例

##### 1）xml格式

```xml
<select id="findByIdAccount" parameterType="int" resultType="pojo.Account">
    select * from account where UID=#{id}
</select>
<resultMap id="UserAccountLoad" type="pojo.UserWithAccount">
    <id property="id" column="id"></id>
    <result property="username" column="username"></result>
    <result property="gender" column="gender"></result>
    <result property="birthday" column="birthday"></result>
    <result property="address" column="address"></result>
    <collection property="accountList" ofType="pojo.Account" select="findByIdAccount" column="id">
    </collection>
</resultMap>
<select id="findUserAccountLoad" resultMap="UserAccountLoad" useCache="true">
    select * from user
</select>
```

##### 2）注解格式

```java
@Select("select * from account where UID=#{id}")
//使用user表中的id查询对应的Accounts
List<Account> findByIdAccount(Integer id);

//查询出所有的User以及对应的账户信息
@Results(id = "UserAccountLoad",value = {@Result(id = true,column = "id",property = "id")
,@Result(column = "username",property = "username")
,@Result(column = "gender",property = "gender")
,@Result(column = "birthday",property = "birthday")
,@Result(column = "address",property = "address")
,@Result(column = "id",property = "accountList",many = @Many(select = "dao.UserDaoAnno.findByIdAccount",fetchType = FetchType.LAZY))})
@Select("select * from user")
List<UserWithAccount> findUserAccountLoad();
```

嵌套关系为@Results(@Result(@One/@Many))

> **使用ResultMap注解时，引用的是Result注解中的id属性**

### 5.3其他注解

#### 5.3.1 @SelectProvider

实现动态SQL映射 

#### 5.3.2@CacheNamespace

实现注解二级缓存的使用，需要标记在接口之上

```java
@CacheNamespace(blocking=true)
public interface UserDaoAnno {...}
```

