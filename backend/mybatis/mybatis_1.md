# Mybatis学习

---

# 1. Mybatis框架概述

##  1.1基本概念 

### 1.1.1什么是框架

在编程领域，软件框架是指一种抽象形式，它提供了一个具有通用功能的软件，这些功能可以由使用者编写代码来有选择的进行更改，从而提供服务于特定应用的软件。软件框架提供了一种标准的方式来构建并部署应用。

软件框架是一种通用的、可复用的软件环境，它提供特定的功能，作为一个更大的软件平台的一部分，用以促进软件应用、产品和解决方案的开发工作。软件框架可能会包含支撑程序、编译器、代码、库、工具集以及 API，它把所有这些部件汇集在一起，以支持项目或系统的开发。

> 引自https://insights.thoughtworks.cn/what-is-software-framework/

### 1.1.2框架的作用

由于框架的存在，可以将平时冗余重复的代码抽离出并放入框架，这样在编写代码时可以更注重于功能和业务本身，大大提高了开发效率。又得益于Java中的反射机制，使得框架的开发更便捷。

### 1.1.3什么是Mybatis框架

#### 1）[官方定义](https://mybatis.org/mybatis-3/zh/index.html)

> *MyBatis 是一款优秀的**持久层框架**，它支持**自定义 SQL、存储过程以及高级映射**。MyBatis 免除了几乎所有的 **JDBC 代码**以及**设置参数**和**获取结果集**的工作。MyBatis 可以通过简单的 **XML 或注解**来配置和映射原始类型、接口和 Java POJO（Plain Old Java Objects，普通老式 Java 对象）为数据库中的记录。*

#### 2）ORM(Object-relational mapping)

ORM通过其名称可以看出是“对象”和“关系”的映射，对象代表着所有面向对象语言中的对象，关系代表关系型数据库中的数据。由于两者天然的存在类似性，我们可以举例如下：

- 类 对应于 表的定义
- 对象 对应于 表中的记录

ORM要解决的问题是如何将以对象存储的数据和以表中的记录存储的数据之间进行**互相转换**。

#### 3）Mybatis为什么不是完全的ORM框架

ORM是Object和Relation之间的映射，包括Object->Relation和Relation->Object**两方面**。Hibernate是个完整的ORM框架，**而MyBatis完成的是Relation->Object，也就是其所说的Data Mapper Framework**。

JPA是ORM框架标准，主流的ORM框架都实现了这个标准。MyBatis**没有实现JPA**(Java Persistence API)，它和ORM框架的设计思路不完全一样。MyBatis是拥抱SQL，而ORM则更靠近面向对象，不建议写SQL，实在要写，则推荐你用框架自带的类SQL代替。MyBatis是SQL映射框架而不是ORM框架，当然**ORM和MyBatis都是持久层框架**。

最典型的ORM 框架是Hibernate，它是全自动ORM框架，而MyBatis是**半自动的**。Hibernate完全可以通过对象关系模型实现对数据库的操作，拥有完整的JavaBean对象与数据库的映射结构来自动生成SQL。而MyBatis仅有基本的字段映射，对象数据以及对象实际关系仍然需要通过手写SQL来实现和管理。

Hibernate数据库移植性远大于MyBatis。Hibernate通过它强大的映射结构和HQL语言，大大降低了对象与数据库（oracle、mySQL等）的耦合性，而MyBatis由于需要手写SQL，因此与数据库的耦合性直接取决于程序员写SQL的方法，如果SQL不具通用性而用了很多某数据库特性的SQL语句的话，移植性也会随之降低很多，成本很高。

总而言之，Mybatis支持我们手写sql，并对sql进行优化，框架所完成的部分，其实是除了手写sql和执行sql之外的所有部分(如获取连接、准备PreparedStatement对象等)，其最大的贡献是完成了关系型数据库中的**记录到POJO的映射**

> Mybatis会将日志功能托付给slf4j来完成

## 1.2入门案例

### 1.2.1 引入Mybatis依赖

```xml
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.5</version>
</dependency>
```

### 1.2.2 创建数据库

表格如下

![mybatis1_usertableinfo](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/mybatis/mybatis1_usertableinfo.png)

### 1.2.3 编写POJO

```java
public class User implements Serilizable {
    private Integer id;
    private String username;
    private Date birthday;
    private String gender;
    private String address;

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", username='" + username + '\'' +
                ", birthday=" + birthday +
                ", gender='" + gender + '\'' +
                ", address='" + address + '\'' +
                '}';
    }
}
```

### 1.2.4编写DAO接口

在dao/下编写下面的接口

#### 1）基于XML开发

```java
public interface UserDaoXML {
    //查询所有用户，使用xml配置
    List<User> findAllUserWithXML();
}
```

#### 2）基于注解开发

```java
public interface UserDaoAnno {
   //查询所有用户，使用注解
    @Select("select * from user")
    List<User> findAllUserWithAnn();
}
```

### 1.2.5 编写Mybatis-Config.xml文件

此xml用于配置和数据库连接相关的信息，在编写之前要指名html的文档类型，需要引入DOCTYPE如下

```html
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
```

用于指定这个html文件是一个配置文件，之后编写Mybatis的配置信息

```xml
<configuration>
    <environments default="mysql">
        <environment id="mysql">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://123.57.16.192:3306/mybatis_learn"/>
                <property name="username" value="root"/>
                <property name="password" value="19961227"/>
            </dataSource>
        </environment>
    </environments>
</configuration>
```

#### 1）基于XML开发

##### （1）编写userdao-mapper.xml

之后是配置自定义接口方法 与 sql语句和返回值类型组成的mapper信息之间的映射关系，此处采用xml配置，首先需要引入DOCTYPE指明文件类型是mapper

```xml
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
```

之后编写映射关系

```xml
<mapper namespace="dao.UserDaoXML">
<!--要注明返回值要匹配的类型，由于select语句是逐条查询，所以对于每一条数据来说，都应该对应一个实体类-->
    <select id="findAllUserWithXML" resultType="pojo.User">
        select * from user;
    </select>
</mapper>
```

上面的mapper标签代表**dao.UserDaoXml.findAllUserWithXML这个key**映射的是**包含了sql语句为select * from user以及返回值类型为pojo.User的**对象

##### （2）编写Mybatis-Config.xml

根据上面所写的mapper标签，需要在此Configuration xml中引入mapper标签，并引用上面的xml文件

```xml
<mappers>
    <!--利用xml解析-->
    <mapper resource="userdao-mapper.xml"/>
</mappers>
```

#### 2）基于注解开发

```xml
<!--mapper的配置不可以对同一个类mapper两次-->
<mappers>
    <!--利用注解解析-->
    <mapper class="dao.UserDaoAnno"/>
</mappers>
```

这里需要注意使用的属性名称为class，且需要给出接口方法的全限定类名

### 1.2.6 编写测试方法

#### 1）基于xml

```java
@Test
public void testFindAllXML(){
    //1.读取配置文件
    InputStream in = null;
    SqlSession sqlSession = null;
    try {
        in = Resources.getResourceAsStream("sql-config.xml");
        //2.创建SqlSessionFactory工厂
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(in);
        //3.使用工厂创建SqlSession对象
        sqlSession = sqlSessionFactory.openSession();
        //4.使用SqlSession对象创建代理对象（AOP），已经对findAllUser方法做了增强
        UserDaoXML userDao = sqlSession.getMapper(UserDaoXML.class);
        //5.使用代理对象执行方法
        List<User> allUser = userDao.findAllUserWithXML();
        for (User user : allUser) {
            System.out.println(user);
        }
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        //6.释放资源
        assert sqlSession != null;
        sqlSession.close();
        try {
            in.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

}
```

#### 2）基于注解

```java
@Test
public void testFindAllAnn(){
    //1.读取配置文件
    InputStream in = null;
    SqlSession sqlSession = null;
    try {
        in = Resources.getResourceAsStream("sql-config.xml");
        //2.创建SqlSessionFactory工厂
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(in);
        //3.使用工厂创建SqlSession对象
        sqlSession = sqlSessionFactory.openSession();
        //4.使用SqlSession对象创建代理对象（AOP），已经对findAllUser方法做了增强
        UserDaoAnno userDao = sqlSession.getMapper(UserDaoAnno.class);
        //5.使用代理对象执行方法
        List<User> allUser = userDao.findAllUserWithAnn();
        for (User user : allUser) {
            System.out.println(user);
        }
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        //6.释放资源
        assert sqlSession != null;
        sqlSession.close();
        try {
            in.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

}
```

## 1.3案例分析

### 1.3.1POJO类编写

#### 1）规则一---无构造器

在定义POJO类时，如果不想定义额外的构造器，那么就必须保证POJO类中的成员变量名称和类型必须和要查询表中的字段名称完全一致

#### 2）规则二---有构造器

如果不想保证POJO类中的成员变量名和表中的字段完全一致，那么必须提供构造器，对于User类来说，假设将address改名为add，由于address在表中的字段相对位置是最后一个，那么在提供构造器时，必须保证构造器如下

```java
public User(Integer id, String username, Date birthday, String gender, String add) {
    this.id = id;
    this.username = username;
    this.birthday = birthday;
    this.gender = gender;
    this.add = add;
}
```

即在构造器中的相对位置也需要是最后一个，如果定义如下

```java
public User(String add) {
    this.id = id;
    this.username = username;
    this.birthday = birthday;
    this.gender = gender;
    this.add = add;
}
```

由于表中处于第一个位置的字段是id，那么add就会被赋值为id

> 上面提到的两个规则，Mybatis会先查找所有名称完全符合的成员变量进行赋值，之后才会按照构造器赋值，也即**规则一会覆盖规则二**

### 1.3.2流程分析

#### 1）读取配置文件

首先需要将配置好的xml读入，从其中获取有关于数据库连接的信息（用于创建Connection对象）以及mapper的相关信息

#### 2）创建SqlSessionFactory工厂

此处使用了构建者模式，通过调用SqlSessionFactoryBuilder对象的build方法即可创建出一个SqlSessionFactory

>建造者模式（Builder Pattern）使用多个简单的对象一步一步构建成一个复杂的对象。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。
>
>一个 Builder 类会一步一步构造最终的对象。该 Builder 类是独立于其他对象的。

#### 3）使用工厂创建SqlSession对象

这里使用了工厂模式，通过调用SqlSessionFactory的getSession方法创建出一个SqlSession对象

#### 4）使用SqlSession对象创建代理对象

这一步使用了动态代理的思想，由于此处提供了一个接口，Mybatis会创建一个实现了这个接口的代理对象，

##### （1）使用xml

如果使用xml来配置mapper，那么sql语句和resultType会一同作为namespace与id拼接处的key（其实也即方法的引用名，本例中为dao.UserDaoXML.findAllUserWithXML）所对应的值。Mybatis会根据这些信息创建出PreparedStatement对象并根据给定的resultType将查询到的数据逐条封装为POJO对象，本例中，由于返回的是一个list集合，所以会将所有的POJO对象放入到list中返回

##### （2）使用注解

使用注解的不同是需要使用@Select注解，并给注解的value属性赋值为sql语句，以String类型存入，且mapper中的属性使用的是class而不是resource。此外由于返回的值**不像xml中通过resultType指定**，所以必须要在接口方法的List返回值中**指明泛型的类型**，相反；**使用xml时，即使方法的类型指明泛型仍然需要制定resultType的值**

#### 5）使用代理对象执行方法

由于返回的代理对象实现了接口中的方法，所以只需要直接执行就可以获得结果

#### 6）关闭连接

由于返回的SqlSession对象在执行过方法后不会直接关闭Connection对象，所以需要调用close方法将Connection关闭

> 不论是使用xml还是注解来配置mapper信息，都需要一个用于配置数据库连接信息的xml文件，并在其中指明mapper要映射的对象


