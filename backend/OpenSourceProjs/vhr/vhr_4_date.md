## 3.日期格式转换

### 3.1 Servlet对请求的处理

#### 3.1.1 前端->后端

##### Content-Type

前端在发出**POST**请求时，根据Content-Type属性的不同，被提交的格式也不同，常用的格式有如下几个

###### （1）**application/x-www-form-urlencoded**

在使用表单提交少量数据时（大多数的Web表单都是），会采用这种方式设定请求体的编码，这种编码方式下，请求体会以一个**长字符串传输**，其格式如下

`k1=v1&k2=v2`

其中v可以是**字符串也可以是json**，一般情况下为字符串

- 字符串

  `name=lily&date=2019-01-01`

- json

  `name={"lastName"="lily"}&date=2019-01-01`

表单提交的数据**虽说**会放入请求体中以键值对的形式存在，但是在**tomcat**中，对于**application/x-www-form-urlencoded**类型的表单，会将请求体中的参数会被解析成parameters（具体的处理见[此文](https://hongjiang.info/http-application-x-www-form-urlencoded/)），这样**最终的效果就等同于提交了一个GET请求**，**url中携带了要传输的参数**，这些参数同样会被放入parameters中（一个ConcurrentHashMap）

这样，就可以**servlet**中可以通过`request.getParameter(key)`来获得，**那么在Spring MVC中，就可以利用`@RequestParam`也就可以拿到（这也是为什么`@RequestParam`不只可以用于`GET`请求）**

> **如果使用POST请求发送，`@RequestBody`同样可以获取**

###### （2）**multipart/form-data**

这种情况一般用在上传文件（二进制数据）时使用，其会将要上传的数据分割为多个MIME段，格式如下图所示（来自《图解HTTP》）

![image-20201021111149344](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201021111149344.png)

每一部分都会有固定的头来表示文件名称（name）和类型（Disposition），这种方式通过**增大MIME类型的payload的大小**，来实现减少头部信息，适用于大文件传输

>**application/x-www-form-urlencoded与multipart/form-data对比**
>
>由于前者对**每一个要传输的字符都需要用三个字符来描述**，如描述"名字 : lily"，那么名字字段就会被转化成`k1=名字`，这样很浪费资源，所以不适用于大文件传输
>
>但后者也不适用于小表单传输，因为引入了过多的实体头部信息和分界符，这时`键值对`是**更简练**的传输方式
>
>详情见[application/x-www-form-urlencoded or multipart/form-data?](https://stackoverflow.com/questions/4007969/application-x-www-form-urlencoded-or-multipart-form-data)

###### （3）application/json

当使用ajax发送如下POST请求时，服务端收到的请求体格式为json

```xml
<script>
    $(function () {
        $("#bt1").click(function () {
            $.ajax({
                url:"anno/test_json",
                contentType:"application/json;charset=UTF-8",
                data:'{"gender":"女","age":"13"}',
                type:"POST",
                success:function (data) {
                    //下面的name会显示undefined这是因为User类没有提供name的get方法
                    alert(data.name)
                    alert(data.pwd)
                },
                error : function() {
                    alert('Sorry, it is wrong!');
                }
            });
        });
    });
</script>
```

其中通过指定`contentType:"application/json;charset=UTF-8"`就可以完成传送json数据。请求体中存放的数据也正如ajax请求中的date部分所示`'{"gender":"女","age":"13"}'`

由于servlet只会将json类型的数据当作字符串处理，如果想解析和生成json，需要调用第三方库（如fastjson，jackson等）。**更重要的是！**，**servlet不会像**对待**application/x-www-form-urlencoded**类型的表单那样，将请求体中的数据解析到parameters中，只能从请求体中获取json

> **只能使用`@RequestBody`注解获取参数**

#### 3.1.2 后端 ->前端

##### Content-type

###### text/html

一般servlet进行响应时，都会通过`response.setContentType("text/html;charset=utf-8");`将响应体的类型设定为text/html，之后调用write方法将页面输出到浏览器上。

write方法一般输出字符串，字符串可以是json也可以是"html代码"，当想要输出json时同样需要调用第三方库

### 3.2 Spring MVC对日期格式化

日期格式化主要考虑三个点之间的传输分别是

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201021132939276.png" alt="image-20201021132939276" style="zoom:50%;" />

#### 3.2.1 前端->后台

##### 1）全局配置

###### （1）配置方式

:one: 自定义Converter并注册为Bean

这种方式适用于所有请求发来的情况，由于Spring MVC中没有给出String到Date的Converter，所以需要自定义类实现此接口，并重写其中的convert方法

```java
public class DateConverter implements Converter<String, Date> {
    SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
    @Override
    public Date convert(String source) {
        try {
            return sdf.parse(source);
        } catch (ParseException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

- **如果不使用Spring Boot，需要在xml中配置如下**

  ```xml
  <!--开启对springmvc框架注解的支持-->
  <mvc:annotation-driven conversion-service="conversionService" />
  
  <!-- 配置类型转换器 -->
  <bean id="conversionService"
     class="org.springframework.format.support.FormattingConversionServiceFac toryBean">
   <property name="converters">
       <set>
           <bean class="ControllerTest.utils.DateConverter" />
       </set>
   </property>
  </bean>
  ```

   这样才可以完成String到Date的类型转换，否则会抛出如下异常

  ```console
   org.springframework.web.servlet.handler.AbstractHandlerExceptionResolve r.logException Resolved  [org.springframework.web.method.annotation.MethodArgumentTypeMismatchExc eption: Failed to convert value of type 'java.lang.String' to required type 'java.util.Date'; nested exception is org.springframework.core.convert.ConversionFailedException: Failed to  convert from type [java.lang.String] to type [java.util.Date] for value  '2019-01-01'; nested exception is java.lang.IllegalArgumentException]
  ```

- **如果使用Spring Boot，则只需要在此类上注解@Component即可**

:two: 配置applicaton.yaml

如果使用Spring Boot可以按如下配置

```yaml
spring:
	jackson:
		date-format: yyyy-MM-dd HH:mm:ss
		time-zone: GMT+8
```

> 所有可以配置的属性在`JacksonProperties`中

:three: 配置Jackson的序列化机制

这样配置相当于自定义一个配置类，绕过了JacksonAutoConfiguration中使用configureDateFormat方法进行定制

```java
@Configuration
public class JacksonConfig {

    /** 默认日期时间格式 */
    public static final String DEFAULT_DATE_TIME_FORMAT = "yyyy-MM-dd HH:mm:ss";
    /** 默认日期格式 */
    public static final String DEFAULT_DATE_FORMAT = "yyyy-MM-dd";
    /** 默认时间格式 */
    public static final String DEFAULT_TIME_FORMAT = "HH:mm:ss";

    @Bean
    public MappingJackson2HttpMessageConverter mappingJackson2HttpMessageConverter() {
        MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();
        ObjectMapper objectMapper = new ObjectMapper();

        // 忽略json字符串中不识别的属性
        objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        // 忽略无法转换的对象 
        objectMapper.configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false);
        // PrettyPrinter 格式化输出
        objectMapper.configure(SerializationFeature.INDENT_OUTPUT, true);
        // NULL不参与序列化
        objectMapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);

        // 指定时区
        objectMapper.setTimeZone(TimeZone.getTimeZone("GMT+8:00"));
        // 日期类型字符串处理
        objectMapper.setDateFormat(new SimpleDateFormat(DEFAULT_DATE_TIME_FORMAT));

        // java8日期日期处理
        JavaTimeModule javaTimeModule = new JavaTimeModule();
        javaTimeModule.addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)));
        javaTimeModule.addSerializer(LocalDate.class, new LocalDateSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)));
        javaTimeModule.addSerializer(LocalTime.class, new LocalTimeSerializer(DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT)));
        javaTimeModule.addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)));
        javaTimeModule.addDeserializer(LocalDate.class, new LocalDateDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)));
        javaTimeModule.addDeserializer(LocalTime.class, new LocalTimeDeserializer(DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT)));
        objectMapper.registerModule(javaTimeModule);

        converter.setObjectMapper(objectMapper);
        return converter;
    }
}
```

> 方式:two: 和方式 :three: 都是使用`SimpleDateFormat`，不支持`LocalDateTime`，需要使用`DateTimeFormatter`，这时使用converter配置更好些，因为可以自定义**格式化器**

:four: @JsonFormat注解

只需要将`@JsonFormat(pattern = "yyyy-MM-dd",timezone = "GMT+8") `**修饰在POJO中Date相关的成员变量**上即可

###### （2）生效场景

:one: 

- 当**日期以字符串形式使用GET请求**发来时，Spring MVC会调用此converter将String类型转换为Date类型并注入到相应属性中
  - 如果是POJO，就注入到对应成员变量字段
  - 如果只是Date变量，则直接赋值
  
- 当**日期以字符串形式使用POST请求**发来时，Spring MVC和上一个效果一致

- 当**日期以JSON形式使用POST请求**发来时，Spring MVC需要导入第三方库，此处使用Jackson，那么JSON解析为POJO对象时，由Jackson中定义的规则来实现，**但是关于日期类型的转换Jackson是使用`com.fasterxml.jackson.databind.util.StdDateFormat`来完成，这个类的成员变量中只提供了很有限的几种策略，如下所示**

  - "yyyy-MM-dd'T'HH:mm:ss.SSSZ"；
  - "yyyy-MM-dd";
  - "EEE, dd MMM yyyy HH:mm:ss zzz";
  - long类型的时间戳

  **注意上面的几种策略仅限从`客户端到服务端`！！！**

**不支持从`服务端向客户端`返回JSON串时将Date属性格式化，这归因于Jackson的序列化机制**

:two: 和 :three:

- 凡是配置的date-format格式都可以使用
- 不支持LocalDateTime

:four: 客户端与服务器之间**双向的**POST-JSON都可以

**但是！不支持从客户端到服务器的非JSON的传输**

##### 2）局部配置

###### （1）配置方式

只需要将`@DateTimeFormat(pattern = "yyyy-MM-dd hh:mm:ss")`

> 如果定义了Converter那么注解会失效，也即全局配置覆盖局部配置

###### （2）生效场景

:one: 客户端到服务器方向的任何请求都可以包括

- POST--**application/x-www-form-urlencoded**
- POST-JSON
- GET

**但是！不支持服务器端到客户端！**

#### 3.2.2 后端->前端

在后端从数据库查到日期信息后，一般需要将POJO对象序列化为JSON字符串再输出到浏览器上，这时必须使用第三方库，**但是**，并不支持**Date类型的格式化**（指挥序列化为long型变量），有两种方式来解决

##### 全局配置

:one: 

由于Jackson在序列化POJO时没有定义如何格式化日期，所以可以通过修改序列化机制来完成

```java
/**
 * 日期序列化
 */
public class DateJsonSerializer extends JsonSerializer<Date> {
    @Override
    public void serialize(Date date, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException {
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
        jsonGenerator.writeString(dateFormat.format(date));
    }
}

/**
 * 日期反序列化
 */
public class DateJsonDeserializer extends JsonDeserializer<Date> {
    @Override
    public Date deserialize(JsonParser jsonParser, DeserializationContext deserializationContext) throws IOException {
        try {
            SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
            return dateFormat.parse(jsonParser.getText());
        } catch (ParseException e) {
            throw new RuntimeException(e);
        }
    }
}

/**
 * 使用方式
 */
@JsonSerialize(using = DateJsonSerializer.class)
@JsonDeserialize(using = DateJsonDeserializer.class)
private Date releaseDate;
```

设定好序列化器和反序列化器之后，只需配合`@JsonSerialize`和`sonDeserialize`两个注解使用即可

:two:  @JsonFormat注解

使用方法和前面相同，此注解支持双向绑定

### 3.3 例子

此例子用于验证Converter、@DateTimeFormat和@JsonFormat三者之间的区别

#### 3.3.1 POJO

```JAVA
public class DateUser {

    private String name;


    //@DateTimeFormat(pattern = "yyyy-MM-dd hh:mm:ss") //客户端-->服务器，什么提交都可以，但是服务器到客户端不行 全局会覆盖局部
   @JsonFormat(pattern = "yyyy-MM-dd hh:mm:ss",timezone = "GMT+8") //只能是json相关的转换 但是双向都可以
    private Date date;

    public Date getDate() {
        return date;
    }

    public void setDate(Date date) {
        this.date = date;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "DateUser{" +
                "date=" + date +
                ", name='" + name + '\'' +
                '}';
    }
}
```

#### 3.3.2 Converter

```java
public class DateConverter implements Converter<String, Date> {
    SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
    @Override
    public Date convert(String source) {
        try {
            return sdf.parse(source);
        } catch (ParseException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

#### 3.3.3 Controller

```java
@RequestMapping(path = "/dateformat/post1")
@ResponseBody
public DateUser DateTest(@RequestBody DateUser du) {
    System.out.println("dateformat/post1执行");
    System.out.println(du);
    return du;
}

@RequestMapping(path = "/dateformat/post2")
public void DateTest2(DateUser dateUser) {
    System.out.println("dateformat/post2执行");
    System.out.println(dateUser);
}
```

#### 3.3.4 jsp页面

```jsp
<script>
    $(function () {
        $("#bt2").click(function () {
            $.ajax({
                url:"anno/dateformat/post1",
                contentType:"application/json;charset=UTF-8",
                data:'{"name":"lily","date":"2019-01-01 12:02:23"}',
                type:"POST",
                success:function (data) {
                    //下面的name会显示undefined这是因为User类没有提供name的get方法
                    alert(data.name)
                    alert(data.date)
                },
                error : function() {
                    alert('Sorry, it is wrong!');
                }
            });
        });
    });
</script>
```

```jsp
<h4>测试17--日期转换测试1</h4>
<div style="background-color: moccasin; background-size: auto">
    <button id="bt2">发送DateUser</button>
</div>
<h4>测试18--日期转换测试2</h4>
<div style="background-color: moccasin; background-size: auto">
    <form action="anno/dateformat/post2" method="post">
        输入用户名： <input type="text" name="name">
        输入日期：<input type="text" name="date">
        <input type="submit" value="发送用户信息">
    </form>
</div>
```

### 3.4 vhr中的配置

采用了`Converter`（非JSON的客到服）+`@JsonFormat`（JSON相关）注解的方式，这样就涵盖了所有的情况

`Converter`配置如下

```java
@Component
public class DateConverter implements Converter<String, Date> {
    SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");

    @Override
    public Date convert(String source) {
        try {
            return sdf.parse(source);
        } catch (ParseException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

`POJO`类上都添加了`@JsonFormat`注解

### **要注意！vhr中配置`Converter`是为了高级检索时解析`date数组`而存在的！对于以JSON形式符合Jackson规范的，`即使不配置@JsonFormat也可以正常解析！`**

### 3.5 总结

- Converter

  适用于客户端->服务端的非JSON的任何传递，依赖于Jackson对json的预处理

- @DateTimeFormat

  适用于客户端->服务端的任何传递

- @JsonFormat

  Jackson的注解，可以应用于序列化和反序列化JSON时对日期参数格式化

### 附注

#### #1

关于`application/x-www-form-urlencoded` 和 `multipart/form-data`的区别参见[此帖](https://stackoverflow.com/questions/4007969/application-x-www-form-urlencoded-or-multipart-form-data)

#### #2

[servlet到SpringMVC的演化](https://juejin.im/post/6844903570681135117)

#### #3

[servlet与Tomcat关系](https://blog.csdn.net/qq_19782019/article/details/80292110)

一句话总结

servlet是Java编写的，用于简化HTTP请求处理和响应的工具接口

Tomcat是servlet等的实现（容器），作为HTTP请求的第一站和服务端发出响应的最后一站，负责解析请求和返回响应，并创建和调用对应servlet(通过web.xml中的配置或者@WebServlet注解查询)来执行service方法（其中根据请求方法不同，执行doGet和doPost方法）

#### #4 

[Tomcat创建servlet的流程](https://blog.csdn.net/chenmixuexi_/article/details/73614504)

[Tomcat与servlet的关系](https://objcoding.com/2019/05/30/tomcat-architecture/)

#### #5

[Jackson序列化-“默认的”时间格式化类—StdDateFormat解析](https://www.jianshu.com/p/307ad48978d6)
