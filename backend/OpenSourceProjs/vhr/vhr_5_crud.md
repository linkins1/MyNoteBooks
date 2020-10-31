
## 5.CRUD总结

### PRE

#### Controller 负责映射请求，并调用Service处理请求

#### Service 负责对请求处理，并调用Dao执行查询

#### Dao（Mappers）执行查询

### 5.1 Employee相关

> 下面与持久层相关（Mybatis）的部分，只选取重要的mappers.xml配置放出

#### 5.1.1 认证成功后返回页面

##### 1）Controller

```java
@RestController
@RequestMapping("/system/config")
public class SystemConfigController {
    @Autowired
    MenuService menuService;
    @GetMapping("/menu")
    public List<Menu> getMenusByHrId() {
        return menuService.getMenusByHrId();
    }
}
```

请求会映射到这个Controller上，调用`menuService.getMenusByHrId()`

##### 2）Service

```java
public List<Menu> getMenusByHrId() {
    //根据登陆后认证成功的Authentication中包含的Hr的id信息去查找Menus
    return menuMapper.getMenusByHrId(((Hr) SecurityContextHolder.getContext().getAuthentication().getPrincipal()).getId());
}
```

#### 5.1.2 查看员工资料

点击**“基本资料”**会发出如下请求

- `GET "/employee/basic/?page=1&size=10&name=", parameters={masked}`
- `GET "/employee/basic/nations", parameters={}`
- `GET "/employee/basic/joblevels", parameters={}`
- `GET "/employee/basic/deps", parameters={}`
- `GET "/employee/basic/politicsstatus", parameters={}`
- `GET "/employee/basic/positions", parameters={}`
- `GET "/employee/basic/positions", parameters={}`

> #### 总体分发流程如下
>
> - 上面的所有请求都由`org.springframework.web.servlet.DispatcherServlet`来处理
> - 交由`RequestMappingHandlerMapping`来将请求映射到对应的Controller的方法上
> - 由于是RestController，最终返回值由`RequestResponseBodyMethodProcessor#handleReturnValue`输出到页面上

##### 1）Controller

```java
@RestController
@RequestMapping("/employee/basic")
public class EmpBasicController {    
    @Autowired
    EmployeeService employeeService;
    @Autowired
    NationService nationService;
    @Autowired
    PoliticsstatusService politicsstatusService;
    @Autowired
    JobLevelService jobLevelService;
    @Autowired
    PositionService positionService;
    @Autowired
    DepartmentService departmentService;
    ...
}
```

**此处只举例`@GetMapping("/")`对应的方法，对应Controller方法如下**

```java
public RespPageBean getEmployeeByPage(@RequestParam(defaultValue = "1") Integer page, @RequestParam(defaultValue = "10") Integer size, Employee employee, Date[] beginDateScope) {
    return employeeService.getEmployeeByPage(page, size, employee,beginDateScope);
}
```

这个方法中，如果没有设定其中的参数，所以均取默认值

###### （1）基本查询

只支持员工名查询，因为发出的GET请求都是如下所示

`/employee/basic/?page=1&size=10&name` `=` `设定值`

在基本搜索框中输入不是姓名的信息，都将查询为空。

这里发出的GET请求映射到上面的Controller方法会按参数顺序赋值，也即只会将Employee的name属性进行赋值，其他均为空

###### （2）高级查询

支持多种类型查询，如下几种请求都是

- `/employee/basic/?page=1&size=10&nationId=1&posId=29`

- `/employee/basic/?page=1&size=10&politicId=1&nationId=1&posId=29&beginDateScope=2015-10-24,2020-11-16`

Spring MVC会按照参数的顺序和名称将其分别赋值到Employee对象和Date数组中

其中在解析Date数组时，会将传入的时间字符串调用[上一节]()提到的DateConverter进行转换，执行顺序如下

- `org.springframework.core.convert.support.StringToArrayConverter#convert`

- `org.springframework.core.convert.support.GenericConversionService#convert(java.lang.Object, org.springframework.core.convert.TypeDescriptor, org.springframework.core.convert.TypeDescriptor)`

  其方法体如下

  ```java
  public Object convert(@Nullable Object source, @Nullable TypeDescriptor sourceType, TypeDescriptor targetType) {
     Assert.notNull(targetType, "Target type to convert to cannot be null");
     if (sourceType == null) {
        Assert.isTrue(source == null, "Source must be [null] if source type == [null]");
        return handleResult(null, targetType, convertNullSource(null, targetType));
     }
     if (source != null && !sourceType.getObjectType().isInstance(source)) {
        throw new IllegalArgumentException("Source to convert from must be an instance of [" +
              sourceType + "]; instead it was a [" + source.getClass().getName() + "]");
     }
     GenericConverter converter = getConverter(sourceType, targetType);
     if (converter != null) {
        Object result = ConversionUtils.invokeConverter(converter, source, sourceType, targetType);
        return handleResult(sourceType, targetType, result);
     }
     return handleConverterNotFound(source, sourceType, targetType);
  }
  ```

  - 其中会先执行`GenericConversionService#getConverter`方法，获取到的就是 DateConverter对象

  - 之后执行`org.springframework.core.convert.support.ConversionUtils#invokeConverter`开始进行类型转换

##### 2）Service

```java
public RespPageBean getEmployeeByPage(Integer page, Integer size, Employee employee, Date[] beginDateScope) {
    //mysql中起始索引从0开始
    //这两个参数用于limit page（offset） size（size）
    if (page != null && size != null) {
        page = (page - 1) * size;
    }
    List<Employee> data = employeeMapper.getEmployeeByPage(page, size, employee, beginDateScope);
    Long total = employeeMapper.getTotal(employee, beginDateScope);
    RespPageBean bean = new RespPageBean();
    //设定当前分页的数据
    bean.setData(data);
    //设定所有查询到的数据
    bean.setTotal(total);
    return bean;
}
```

##### 3）Dao(Mapper)

- 接口

```java
List<Employee> getEmployeeByPage(@Param("page") Integer page, @Param("size") Integer size, @Param("emp") Employee employee,@Param("beginDateScope") Date[] beginDateScope);
```

由于方法存在多个参数，所以使用了@Param注解

- xml

```xml
<select id="getEmployeeByPage" resultMap="AllEmployeeInfo">
    select e.*,p.`id` as pid,p.`name` as pname,n.`id` as nid,n.`name` as nname,d.`id` as did,d.`name` as
    dname,j.`id` as jid,j.`name` as jname,pos.`id` as posid,pos.`name` as posname from employee e,nation
    n,politicsstatus p,department d,joblevel j,position pos where e.`nationId`=n.`id` and e.`politicId`=p.`id` and
    e.`departmentId`=d.`id` and e.`jobLevelId`=j.`id` and e.`posId`=pos.`id`
--         在没指定emp时，不启用下面的查询条件
    <if test="emp.name !=null and emp.name!=''">
        and e.name like concat('%',#{emp.name},'%')
    </if>
    <if test="emp.politicId !=null">
        and e.politicId =#{emp.politicId}
    </if>
    <if test="emp.nationId !=null">
        and e.nationId =#{emp.nationId}
    </if>
    <if test="emp.departmentId !=null">
        and e.departmentId =#{emp.departmentId}
    </if>
    <if test="emp.jobLevelId !=null">
        and e.jobLevelId =#{emp.jobLevelId}
    </if>
    <if test="emp.engageForm !=null and emp.engageForm!=''">
        and e.engageForm =#{emp.engageForm}
    </if>
    <if test="emp.posId !=null">
        and e.posId =#{emp.posId}
    </if>
    <if test="beginDateScope !=null">
        and e.beginDate between #{beginDateScope[0]} and #{beginDateScope[1]}
    </if>
--         默认情况下，是查询page=0,size=10
    <if test="page !=null and size!=null">
        limit #{page},#{size}
    </if>
</select>
```

##### 4）RespPageBean

上面的Controller方法返回的是这个类用于封装分页查询的信息，其中

- data属性是根据指定的page和size查询到的结果
- total则是根据除了这两个条件查询得到的全部结果集，用于返回给前端做分页显示

```java
public class RespPageBean {
    private Long total;
    private List<?> data;

    public Long getTotal() {
        return total;
    }

    public void setTotal(Long total) {
        this.total = total;
    }

    public List<?> getData() {
        return data;
    }

    public void setData(List<?> data) {
        this.data = data;
    }
}
```

其中的`List<?>`可以更好的支持分页查询多种类型的信息，此例中返回的是`List<Employee>`类型的信息

#### 5.1.3 增加一个员工

> 通过点击"添加用户"，填写信息后提交，会发出一个POST请求，如下
>
> `POST /employee/basic/`
>
> 请求体中为JSON数据，封装了表单信息

##### 1）Controller

```java
@PostMapping("/")
public RespBean addEmp(@RequestBody Employee employee) {
    if (employeeService.addEmp(employee) == 1) {
        return RespBean.ok("添加成功!");
    }
    return RespBean.error("添加失败!");
}
```

要注意，此处由于前端返回的JSON中日期格式为"yyyy-MM-dd"，这符合Jackson中默认的配置，因而即使不添加`@JsonFormat`也可以解析成功。

##### 2）Service

```java
public Integer addEmp(Employee employee) {
    Date beginContract = employee.getBeginContract();
    Date endContract = employee.getEndContract();
    double month = (Double.parseDouble(yearFormat.format(endContract)) - Double.parseDouble(yearFormat.format(beginContract))) * 12 + (Double.parseDouble(monthFormat.format(endContract)) - Double.parseDouble(monthFormat.format(beginContract)));
    employee.setContractTerm(Double.parseDouble(decimalFormat.format(month / 12)));
    int result = employeeMapper.insertSelective(employee);
    if (result == 1) {
        Employee emp = employeeMapper.getEmployeeById(employee.getId());
        //生成消息的唯一id
        String msgId = UUID.randomUUID().toString();
        MailSendLog mailSendLog = new MailSendLog();
        mailSendLog.setMsgId(msgId);
        mailSendLog.setCreateTime(new Date());
        mailSendLog.setExchange(MailConstants.MAIL_EXCHANGE_NAME);
        mailSendLog.setRouteKey(MailConstants.MAIL_ROUTING_KEY_NAME);
        mailSendLog.setEmpId(emp.getId());
        mailSendLog.setTryTime(new Date(System.currentTimeMillis() + 1000 * 60 * MailConstants.MSG_TIMEOUT));
        mailSendLogService.insert(mailSendLog);
        rabbitTemplate.convertAndSend(MailConstants.MAIL_EXCHANGE_NAME, MailConstants.MAIL_ROUTING_KEY_NAME, emp, new CorrelationData(msgId));
    }
    return result;
}
```

上面方法体中首先计算出合同期（ContractTerm）并将其设定给employee对象，之后调用`insertSelective`函数完成插入，插入成功后，会先获取出刚刚插入的emp对象，之后向emp中存储的邮箱地址通过rabbitmq发送出去，邮箱发送见[发送邮件](./vhr_6_mail)部分，此处仅介绍插入的mapper

##### 3）Dao

```xml
<insert id="insertSelective" parameterType="org.javaboy.vhr.model.Employee" useGeneratedKeys="true"
        keyProperty="id">
    insert into employee
    <trim prefix="(" suffix=")" suffixOverrides=",">
        <if test="id != null">
            id,
        </if>
        -- 省略...
    </trim>
    <trim prefix="values (" suffix=")" suffixOverrides=",">
        <if test="id != null">
            #{id,jdbcType=INTEGER},
        </if>
        -- 省略...
    </trim>
</insert>
```

此处需要注意以下几点

- 设定`useGeneratedKeys="true"`可以实现把数据库中自增的**主键id**封装给输入的参数，也即此处的emp对象
- 通过`<trim>`标签来构建一个括号以及其中的内容
- 在values中取值时也需要注明数据类型，如id对应需要指明`jdbcType=INTEGER`

在添加完成之后，前端会立即发出`GET /employee/basic/?page=1&size=10&name=`请求来刷新页面，这样前端就拿到了**后端传来的员工信息（带id！）**，这是以后删除的标准

####  5.1.4 修改一个员工

> 点击编辑按钮，修改信息后提交会发出一个`PUT /employee/basic/`请求，请求体中包含了要修改的信息

逻辑是按照id去更新条目

在成功后同样会发出`GET请求`用于刷新页面

#### 5.1.5 删除一个员工

> 点击删除按钮，会发出一个`DELETE /employee/basic/id号`请求

由于发送的请求中使用id作为占位符，对应的Controller取出占位符后根据id删除对应员工

在成功后同样会发出`GET请求`用于刷新页面

---

### 5.2 系统管理相关

#### 5.2.1 基础信息设置

#### :one: 部门管理

##### 1）添加部门

> 添加部门会发出`POST /system/basic/department`请求
>
> 请求体格式如下
>
> `{"name":"部门名称","parentId":上级部门编号}`

###### （1）Controller

```java
@RestController
@RequestMapping("/system/basic/department")
public class DepartmentController {
    @Autowired
    DepartmentService departmentService;
    @PostMapping("/")
    public RespBean addDep(@RequestBody Department dep) {
        departmentService.addDep(dep);
        if (dep.getResult() == 1) {
            return RespBean.ok("添加成功", dep);
        }
        return RespBean.error("添加失败");
    }
}
```

会映射到addDep方法上，将name和parentId映射到Department上

###### （2）Service

```java
public void addDep(Department dep) {
    dep.setEnabled(true);
    departmentMapper.addDep(dep);
}
```

###### （3）Dao

这里会调用xml中的存储过程

```xml
<select id="addDep" statementType="CALLABLE">
    call addDep(#{name,mode=IN,jdbcType=VARCHAR},#{parentId,mode=IN,jdbcType=INTEGER},#{enabled,mode=IN,jdbcType=BOOLEAN},#{result,mode=OUT,jdbcType=INTEGER},#{id,mode=OUT,jdbcType=INTEGER})
</select>
```

**这里需要注意字符集编码不匹配可能导致存储过程调用失败！**

存储过程如下

```mysql
create
    definer = root@localhost procedure addDep(IN depName varchar(32) character set utf8, IN parentId int, IN enabled tinyint(1),
                                              OUT result int, OUT result2 int)
begin
  declare did int;
  declare pDepPath varchar(64);
  insert into department set name=depName,parentId=parentId,enabled=enabled;
  select row_count() into result;
  select last_insert_id() into did;
  set result2=did;
  select depPath into pDepPath from department where id=parentId;
  update department set depPath=concat(pDepPath,'.',did) where id=did;
  update department set isParent=true where id=parentId;
end;
```

上面所示的depName，如果是中文，需要保证下面几处的字符集匹配

| 位置     | 配置方式                                                     | 附注                                                         |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| IDEA     | 在settings中设定File Encodings为UTF-8 并设定Transparent native-to-ascii conversion✔ | 设定最后一项是为了Properties文件中存储的信息转为ascii编码，这样才可以保证被正确的读取 |
| mysql    | 设定mysqld/client/default设定为utf-8                         | 常见的[配置文件](####mysql配置文件)                          |
| 存储过程 | 如果上两项设定后仍然乱码，需要对可能传入中文字符的字段设定   | 如`IN depName varchar(32) character set utf8`                |

关于Mybatis中存储过程的调用，参见[Mybatis笔记](/backend/mybatis/mybatis_1?id=_6mybatis与存储过程.md)

当存储过程完成后，会继续执行Controller中的方法，返回一个`RespBean`对象

###### （4）RespBean

```java
public class RespBean {
    private Integer status;
    private String msg;
    private Object obj;

    public static RespBean build() {
        return new RespBean();
    }

    public static RespBean ok(String msg) {
        return new RespBean(200, msg, null);
    }

    public static RespBean ok(String msg, Object obj) {
        return new RespBean(200, msg, obj);
    }

    public static RespBean error(String msg) {
        return new RespBean(500, msg, null);
    }

    public static RespBean error(String msg, Object obj) {
        return new RespBean(500, msg, obj);
    }
}
```

当执行成功时，会返回`RespBean.ok("添加成功", dep);`，其中封装了注入好信息（OUT参数）的`dep`对象，这样前端就可以拿到dep中存储的id信息（执行完存储过程才能得到），**这样才能实现根据id删除某个部门的功能**

##### 2）删除部门

> 删除部门会发出`DELETE /system/basic/department/102`请求

###### （1）Controller

与上面的Controller类一致，方法如下

```java
@DeleteMapping("/{id}")
public RespBean deleteDepById(@PathVariable Integer id) {
    Department dep = new Department();
    dep.setId(id);
    departmentService.deleteDepById(dep);
    if (dep.getResult() == -2) {
        return RespBean.error("该部门下有子部门，删除失败");
    } else if (dep.getResult() == -1) {
        return RespBean.error("该部门下有员工，删除失败");
    } else if (dep.getResult() == 1) {
        return RespBean.ok("删除成功");
    }
    return RespBean.error("删除失败");
}
```

传来的占位符会被注入给id，利用此id去删除对应的部门

###### （2）Service

```java
public void deleteDepById(Department dep) {
    departmentMapper.deleteDepById(dep);
}
```

###### （3）Dao

这里会调用Mysql中的存储过程，如下

```mysql
create
    definer = root@localhost procedure deleteDep(IN did int, OUT result int)
begin
  declare ecount int;
  declare pid int;
  declare pcount int;
  declare a int;
  select count(*) into a from department where id=did and isParent=false;
  if a=0 then set result=-2;
  else
  select count(*) into ecount from employee where departmentId=did;
  if ecount>0 then set result=-1;
  else
  select parentId into pid from department where id=did;
  delete from department where id=did and isParent=false;
  select row_count() into result;
  select count(*) into pcount from department where parentId=pid;
  if pcount=0 then update department set isParent=false where id=pid;
  end if;
  end if;
  end if;
end;
```

- `select count(*) into a from department where id=did and isParent=false;`

  用于判断该部门是不是父部门

- `select count(*) into ecount from employee where departmentId=did;`

  用于判断该部门下是否有员工

- `if pcount=0 then update department set isParent=false where id=pid;`

  在对应部门被删掉后，如果其父部门没有子部门，则可以设定为子部门

> 判断Update或Delete影响的行数用**`row_count()`函数**进行判断，这里需要注意，如果Update前后的值一样，row_count则为0

###### （4）RespBean

此处会根据返回的result去判断删除情况，并返回不同的**RespBean**对象

#### :two: 职位管理

使用到的Controller如下，也是一个RestController

```java
@RestController
@RequestMapping("/system/basic/pos")
public class PositionController {
}
```

##### 1）添加职位

> 输入职位后点击添加按钮会发出`POST /system/basic/pos/`请求，请求体中包含了职位的名称

###### （1）Controller

```java
@PostMapping("/")
public RespBean addPosition(@RequestBody Position position) {
    if (positionService.addPosition(position) == 1) {
        return RespBean.ok("添加成功!");
    }
    return RespBean.error("添加失败!");
}
```

将请求的参数注入到Position对象

###### （2）Service

```java
public Integer addPosition(Position position) {
    position.setEnabled(true);
    position.setCreateDate(new Date());
    return positionMapper.insertSelective(position);
}
```

其中会开启此职位有效，并设定创建日期

###### （3）Dao

mapper中的逻辑和employee中的一致，但是并没有使用`useGeneratedKeys="true"`标签

和添加员工一样，成功后会立即发出`GET /system/basic/pos/`请求来刷新页面，这样前端就可以拿到**职位的信息（带id！）**

##### 2）修改职位

> 发出`PUT /system/basic/pos/`请求，同样是根据id去更新信息

在成功后同样会发出`GET请求`用于刷新页面

##### 3）删除职位

> 如果是删除单个，会发出`DELETE /system/basic/pos/?ids=42&`请求
>
> 如果是批量删除，会发出`DELETE /system/basic/pos/?ids=42&ids=41&`请求

:a: 删除一个

###### （1）Controller

```java
@DeleteMapping("/{id}")
public RespBean deletePositionById(@PathVariable Integer id) {
    if (positionService.deletePositionById(id) == 1) {
        return RespBean.ok("删除成功!");
    }
    return RespBean.error("删除失败!");
}
```

###### （2）Service

```java
public Integer deletePositionById(Integer id) {
    return positionMapper.deleteByPrimaryKey(id);
}
```

###### （3）Dao

```xml
<delete id="deleteByPrimaryKey" parameterType="java.lang.Integer" >
  delete from position
  where id = #{id,jdbcType=INTEGER}
</delete>
```

根据id删除元素

:b: 批量删除

###### （1）Controller

```java
@DeleteMapping("/")
public RespBean deletePositionsByIds(Integer[] ids) {
    if (positionService.deletePositionsByIds(ids) == ids.length) {
        return RespBean.ok("删除成功!");
    }
    return RespBean.error("删除失败!");
}
```

###### （2）Service

```java
public Integer deletePositionsByIds(Integer[] ids) {
    return positionMapper.deletePositionsByIds(ids);
}
```

###### （3）Dao

```xml
<delete id="deletePositionsByIds">
    delete from position where id in
    <foreach collection="ids" item="id" separator="," open="(" close=")">
        #{id}
    </foreach>
</delete>
```

其中使用`foreach`标签来递归输入的数组（使用`Collections`标签），将其构造成`(1,2,3,4)`的样式后删除

#### :three: 职称管理

> 与职位管理逻辑一致

#### :four: 权限组管理

##### 1）添加角色

> 会发出`POST /system/basic/permiss/role`请求，请求体如下
>
> `{"name":"英文名","nameZh":"中文名"}`

###### （1）Controller

```java
@RestController
@RequestMapping("/system/basic/permiss")
public class PermissController {
    @PostMapping("/role")
    public RespBean addRole(@RequestBody Role role) {
        if (roleService.addRole(role) == 1) {
            return RespBean.ok("添加成功!");
        }
        return RespBean.error("添加失败!");
    }   
}
```

Role中包含的信息只有角色的中英文名，角色和资源之间的对应关系还需要通过menu_role表来指明

###### （2）Service

```java
public Integer addRole(Role role) {
    if (!role.getName().startsWith("ROLE_")) {
        role.setName("ROLE_" + role.getName());
    }
    return roleMapper.insert(role);
}
```

设定**"Role_"**前缀以满足SimpleGrantedAuthority的要求

###### （3）Dao

```xml
<insert id="insert" parameterType="org.javaboy.vhr.model.Role" >
  insert into role (id, name, nameZh
    )
  values (#{id,jdbcType=INTEGER}, #{name,jdbcType=VARCHAR}, #{nameZh,jdbcType=VARCHAR}
    )
</insert>
```

##### 2）修改角色（添加角色的权限）

> 由于上一步添加角色后，并没有在menu_role中添加其可以访问的资源的映射信息，所以其权限信息是空的
>
> 当选择好权限信息并点击确认修改后，会发出如下请求
>
> `PUT /system/basic/permiss/?rid=22&mids=7&mids=8`
>
> 其中**`rid为角色的id`，`mids是资源的id`**

###### （1）Controller

```java
@PutMapping("/")
public RespBean updateMenuRole(Integer rid, Integer[] mids) {
    if (menuService.updateMenuRole(rid, mids)) {
        return RespBean.ok("更新成功!");
    }
    return RespBean.error("更新失败!");
}
```

###### （2）Service

```java
@Transactional
public boolean updateMenuRole(Integer rid, Integer[] mids) {
    menuRoleMapper.deleteByRid(rid);
    if (mids == null || mids.length == 0) {
        return true;
    }
    Integer result = menuRoleMapper.insertRecord(rid, mids);
    return result==mids.length;
}
```

这个方法是开启事务的，如果失败会进行回滚。

**首先会删除请求信息中rid相关的所有条目，之后将mids数组中的信息插入其中**

###### （3）Dao

- 删除

```xml
<delete id="deleteByRid">
  delete from menu_role where rid=#{rid}
</delete>
```

- 添加

```xml
<insert id="insertRecord">
  insert into menu_role (mid,rid) values
  <foreach collection="mids" separator="," item="mid">
    (#{mid},#{rid})
  </foreach>
</insert>
```

##### 3）删除角色

###### （1）Controller

```java
@DeleteMapping("/role/{rid}")
public RespBean deleteRoleById(@PathVariable Integer rid) {
    if (roleService.deleteRoleById(rid) == 1) {
        return RespBean.ok("删除成功!");
    }
    return RespBean.error("删除失败!");
}
```

###### （2）Service

```java
public Integer deleteRoleById(Integer rid) {
    return roleMapper.deleteByPrimaryKey(rid);
}
```

###### （3）Dao

```java
<delete id="deleteByPrimaryKey" parameterType="java.lang.Integer" >
  delete from role
  where id = #{id,jdbcType=INTEGER}
</delete>
```

这里删除的逻辑很简单，**但是**，此处需要额外考虑一下异常情况（主外键关联删不掉），具体参见[异常处理]()









### 附

:one:

#### mysql配置文件

```shell
[mysqld]
server-id=10
## 复制过滤：也就是指定哪个数据库不用同步（mysql库一般不同步）
binlog-ignore-db=mysql
## 开启二进制日志功能，可以随便取，最好有含义（关键就是这里了）
log-bin=5-7-mysql-bin
## 为每个session 分配的内存，在事务过程中用来存储二进制日志的缓存
binlog_cache_size=1M
## 主从复制的格式（mixed,statement,row，默认格式是statement）
binlog_format=mixed
## 二进制日志自动删除/过期的天数。默认值为0，表示不自动删除。
expire_logs_days=7
## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
slave_skip_errors=1062
read-only=0
character_set_server=utf8

[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
```



















