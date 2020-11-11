## 6.Mybatis与存储过程

### 6.1 使用方法

Mybatis想要使用存储过程，首先需要在数据库中创建好存储过程。下面举一个vhr中的例子

#### 6.1.1 创建存储过程

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

上面定义了三个入参，两个出参

#### 6.1.2 定义mapper接口

```java
void addDep(Department dep);
```

其中`Department`的定义如下

```java
package org.javaboy.vhr.model;

import java.io.Serializable;
import java.util.ArrayList;
import java.util.List;
import java.util.Objects;

public class Department implements Serializable {
    private Integer id;

    private String name;

    private Integer parentId;

    public Department() {
    }

    public Department(String name) {

        this.name = name;
    }

    private Integer result;

    public Integer getResult() {
        return result;
    }

    public void setResult(Integer result) {
        this.result = result;
    }
    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }
}
```

#### 6.1.3 定义mappers.xml

```xml
<select id="addDep" statementType="CALLABLE">
  call addDep(#{name,mode=IN,jdbcType=VARCHAR},#{parentId,mode=IN,jdbcType=INTEGER},#{enabled,mode=IN,jdbcType=BOOLEAN},#{result,mode=OUT,jdbcType=INTEGER},#{id,mode=OUT,jdbcType=INTEGER})
</select>
```

其中需要注意如下几点

##### 1）statementType

必须为`CALLABLE`

##### 2）mode

使用`call`调用存储过程时，参数与存储过程中定义的顺序和类型必须一致，存储过程中为IN的mode必须是IN；同理，OUT的也必须为OUT

##### 3）jdbcType

同样必须和存储过程中定义的匹配

### 6.2 执行效果

当上面存储过程执行时，会将传入的`dep`对象中的参数传入存储过程中，**并且将执行结果注入到`dep`对象**

**这样后端就可以拿到完整的`dep`对象的信息**