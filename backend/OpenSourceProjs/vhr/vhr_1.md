## 1.数据表与POJO

### 1.1数据表关系

![p274](https://raw.githubusercontent.com/wiki/lenve/vhr/doc/p274.png)

### 1.2Role.class和role表

#### 1.2.1结构

##### 1）Role.class

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201018144941709.png" alt="image-20201018144941709" style="zoom:80%;" /> 

##### 2）role表

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201018144826821.png" alt="image-20201018144826821" style="zoom: 80%;" /> 

### 1.3Hr.class与hr表

#### 1.3.1 结构

##### 1）Hr.class

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201013165945305.png" alt="image-20201013165945305" style="zoom:80%;" /> 

##### 2）hr表

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-202010131327460515.png" alt="image-20201013132746055" style="zoom:80%;" /> 

比较两者可以发现，Hr类中的成员变量roles不在hr表中，需要通过hr_role表来关联

### 1.4 Menu.class与menu表

#### 1.4.1结构

##### 1）Menu.class

<img src="C:%5CUsers%5C123%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20201018145208652.png" alt="image-20201018145208652" style="zoom:80%;" /> 

##### 2）menu表

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201018145114965.png" alt="image-20201018145114965" style="zoom:80%;" /> 

### 1.5 hr_role表与menu_role表

#### 1.5.1 hr_role表

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201018145655898.png" alt="image-20201018145655898" style="zoom:80%;" /> 

#### 1.5.2 menu_role表

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201018145715012.png" alt="image-20201018145715012" style="zoom:80%;" /> 

### 1.6 新建Hr.class对象

Hr.class中存储的不仅是**hr表**中的用户信息，还存储了这个用户对应的角色信息，存储在**role表**中，想要得到一个**Hr对象就需要根据hr、hr_role和role三个表来连接查询得到一个hr**

**HrService**类中实现了**UserDetailsService**中的**loadUserByUsername**方法，函数体如下

```java
@Override
public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
    //根据用户名从hr表中查找到用户
    Hr hr = hrMapper.loadUserByUsername(username);
    if (hr == null) {
        throw new UsernameNotFoundException("用户名不存在!");
    }
    //根据查找到的用户的id去hr_roles表中找到r_id集合并以此从role表中找到对应的名称并返回为List<Role> roles
    hr.setRoles(hrMapper.getHrRolesById(hr.getId()));
    return hr;
}
```

其中hrMapper是**HrMapper**的对象，是Mybatis的一个接口mapper，接口中的loadUserByUsername对应的xml如下

```xml
</select>
  <select id="loadUserByUsername" resultMap="BaseResultMap">
  select * from hr where username=#{username}
</select>
```

```xml
<resultMap id="BaseResultMap" type="org.javaboy.vhr.model.Hr">
    <id column="id" property="id" jdbcType="INTEGER"/>
    <result column="name" property="name" jdbcType="VARCHAR"/>
    <result column="phone" property="phone" jdbcType="CHAR"/>
    <result column="telephone" property="telephone" jdbcType="VARCHAR"/>
    <result column="address" property="address" jdbcType="VARCHAR"/>
    <result column="enabled" property="enabled" jdbcType="BIT"/>
    <result column="username" property="username" jdbcType="VARCHAR"/>
    <result column="password" property="password" jdbcType="VARCHAR"/>
    <result column="userface" property="userface" jdbcType="VARCHAR"/>
    <result column="remark" property="remark" jdbcType="VARCHAR"/>
</resultMap>
```

这个方法即通过用户名从hr表中查找到对应的记录，如果hr为空则抛出异常，否则取出hr中的id属性，并调用`hr.setRoles(hrMapper.getHrRolesById(hr.getId()))`来查找该hr对应的roles集合并set给**roles**属性

getHrRolesById对应的xml如下

```xml
<select id="getHrRolesById" resultType="org.javaboy.vhr.model.Role">
  select r.* from role r,hr_role hrr where hrr.`rid`=r.`id` and hrr.`hrid`=#{id}
</select>
```

通过关联role和hr_role并限定hrid为给定的hr的id查找出该hr所有的roles条目，并封装为**Role**类