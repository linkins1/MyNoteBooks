## #3.异常处理

### #3.1 RespBean

#### #3.1.1 定义

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

    public static RespBean  error(String msg) {
        return new RespBean(500, msg, null);
    }

    public static RespBean error(String msg, Object obj) {
        return new RespBean(500, msg, obj);
    }

    private RespBean() {
    }

    private RespBean(Integer status, String msg, Object obj) {
        this.status = status;
        this.msg = msg;
        this.obj = obj;
    }
}
```

####  #3.1.2负责异常范围

##### 1）CRUD

在CRUD相关的Controller中的方法中，如果执行对应的service出现了异常，就会将异常信息包裹为`RespBean`对象并返回给前端，前端将信息以alert的形式返回在页面上

RespBean提供了几种静态方法（工厂模式），用来构造不同的异常对象，几乎囊括了所有可能出现的错误。

**要注意！**RespBean用于处理业务异常而不是代码异常！此外，由于这些异常是以`RespBean`对象的形式返回，由于每个需要返回异常信息的Controller中的方法的返回值都是`RespBean`，且Controller为`@RestController`，所以通过Jackson处理，在前端可以直接以JSON的形式拿到异常信息

##### 2）loginFilter

在登陆成功和失败时，会分别由`AbstractAuthenticationProcessingFilter`中的`AuthenticationSuccessHandler`和`AuthenticationFailureHandler`对象来做处理

由于这两个类中的onAuthenticationSuccess（用于认证成功时执行）和onAuthenticationFailure方法（用于捕获认证过滤器调用的认证方法所抛出的异常）都是`AbstractAuthenticationProcessingFilter`（常为`UsernamePasswordAuthenticationFilter`）放行时要执行的方法，只能采用从response对象获取输出流的形式来输出信息到前端。

vhr中采用两个lambda表达式来重写这两个方法，如下

###### （1）成功时

```java
//登录成功后执行下面的方法
loginFilter.setAuthenticationSuccessHandler((request, response, authentication) -> {
            //设定response的body格式为json
            response.setContentType("application/json;charset=utf-8");
            PrintWriter out = response.getWriter();
            Hr hr = (Hr) authentication.getPrincipal();
            hr.setPassword(null);
            RespBean ok = RespBean.ok("登录成功!", hr);
            //将对象写为JSON串
            String s = new ObjectMapper().writeValueAsString(ok);
            //将json串写到页面
            out.write(s);
            out.flush();
            out.close();
        }
);
```

###### （2）失败时

```java
loginFilter.setAuthenticationFailureHandler((request, response, exception) -> {
            response.setContentType("application/json;charset=utf-8");
            PrintWriter out = response.getWriter();
            //根据失败的不同原因返回不同的消息
            RespBean respBean = RespBean.error(exception.getMessage());
            if (exception instanceof LockedException) {
                respBean.setMsg("账户被锁定，请联系管理员!");
            } else if (exception instanceof CredentialsExpiredException) {
                respBean.setMsg("密码过期，请联系管理员!");
            } else if (exception instanceof AccountExpiredException) {
                respBean.setMsg("账户过期，请联系管理员!");
            } else if (exception instanceof DisabledException) {
                respBean.setMsg("账户被禁用，请联系管理员!");
            } else if (exception instanceof BadCredentialsException) {
                respBean.setMsg("用户名或者密码输入错误，请重新输入!");
            }
            out.write(new ObjectMapper().writeValueAsString(respBean));
            out.flush();
            out.close();
        }
);
```

> 捕获的异常还包括LoginFilter中重写的attemptAuthentication方法中抛出的异常，如下
>
> - ```java
>   if (!request.getMethod().equals("POST")) {
>       throw new AuthenticationServiceException(
>               "Authentication method not supported: " + request.getMethod());
>   }
>   ```
>
> - ```java
>   public void checkCode(HttpServletResponse resp, String code, String verify_code) {
>       if (code == null || verify_code == null || "".equals(code) || !verify_code.toLowerCase().equals(code.toLowerCase())) {
>           //验证码不正确
>           throw new AuthenticationServiceException("验证码不正确");
>       }
>   }
>   ```

### #3.2 全局异常处理

#### #3.2.1 SpringBoot中的全局异常处理相关注解

##### 1） @ControllerAdvice

> Specialization of `@Component` for classes that declare `@ExceptionHandler`,` @InitBinder`, or `@ModelAttribute` methods to be **shared across multiple @Controller classes**.

从上面注释可以看出，这个注解用于配合上面其他三个注解存在，加入某个类标记了` @ControllerAdvice`，那么所有的Controller就都可以共享这个bean。

下面先看`@ExceptionHandler`的定义，如下

> Annotation for **handling exceptions** in **specific handler classes** and/or **handler methods**.

这个注解用于接收被标记的类或者方法抛出异常e，并将此异常传入这个注解修饰的方法的参数中

##### 2）@RestControllerAdvice

这个注解用于标记在类上，其是`@ResponseBody`+`@ControllerAdvice`的结合体

下面，看vhr中如何使用

#### #3.2.2 vhr中的全局异常处理

##### 1）全局处理类

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(SQLException.class)
    public RespBean sqlException(SQLException e) {
        if (e instanceof SQLIntegrityConstraintViolationException) {
            return RespBean.error("该数据有关联数据，操作失败!");
        }
        return RespBean.error("数据库异常，操作失败!");
    }
}
```

首先这个类相关的bean会被放入容器中以便于所有的Controller共享

当有Controller抛出`SQLException.class`及其子类的异常时，这个全局异常处理对象就会将异常对象e拦截下俩，之后放入`@ExceptionHandler`修饰的方法的参数中，之后根据具体的异常类型将异常信息封装为`RespBean`对象后以JSON形式返回至前端（这也是为什么需要RestController的原因）

##### 2）例子

这里举删除角色信息的例子，由于role表中的主键是menu_role中的外键，如果menu_role中还有记录，那么就不允许删除role中对应的条目，如果删除会抛出下面的异常

```mysql
Error updating database.  Cause: java.sql.SQLIntegrityConstraintViolationException: Cannot delete or update a parent row: a foreign key constraint fails (`vhr`.`menu_role`, CONSTRAINT `menu_role_ibfk_2` FOREIGN KEY (`rid`) REFERENCES `role` (`id`))
```

这时上面定义的全局异常处理对象就发挥作用，将此`SQLIntegrityConstraintViolationException`对象拦截后注入到方法参数中，并返回对应的异常信息。

##### 3）演变

Springboot中全局异常的演变过程见此帖[Spring Boot处理全局异常](https://www.jianshu.com/p/12e1a752974d)
