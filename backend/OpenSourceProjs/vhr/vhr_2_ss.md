## 2.登录流程

### 2.1更新验证码

每点击一次验证码，就会发出如下的**GET请求**

```shell
2020-10-14 16:28:00.237 DEBUG 17392 --- [nio-8081-exec-1] o.s.web.servlet.DispatcherServlet        : GET "/verifyCode?time=Wed%20Oct%2014%202020%2016:28:00%20GMT+0800%20(%E4%B8%AD%E5%9B%BD%E6%A0%87%E5%87%86%E6%97%B6%E9%97%B4)", parameters={masked}
```

此请求会被映射到下面的Controller

```java
@GetMapping("/verifyCode")
public void verifyCode(HttpServletRequest request, HttpServletResponse resp) throws IOException {
    VerificationCode code = new VerificationCode();
    BufferedImage image = code.getImage();
    String text = code.getText();
    HttpSession session = request.getSession(true);
    session.setAttribute("verify_code", text);
    VerificationCode.output(image,resp.getOutputStream());
}
```

其中新建一个VerificationCode对象并获取一个image流，并将image中的验证码取出存为text，并在session中设定一个键值对为`{verify_code:text}`，最终将image流写到页面上，每刷新一次验证码，session中也会随着更新验证码

在用户发来登陆请求时，会验证这个text和用户输入的text是否一致达到验证码匹配的效果

> VerificationCode源码解读参加[验证码]()

### 2.2输入信息并登陆（认证）

### *<u>Pre --- Spring Security中的登录流程</u>*

> 参见[官方文档](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-form-custom)

#### 2.2.1Spring Security中的过滤器

##### 1）DelegatingFilterProxy

由于Servlet容器中注册的filters**并不会感知到**Spring中的beans，为了让两者连接起来，SS中按照servlet容器的规则创建了一个filter名为DelegatingFilterProxy，这个filter会将实际任务交由Spring中的beans来完成，这样就实现了将beans按照filter的顺序和生命周期来运行。

DelegatingFilterProxy在执行链条中的位置如下所示

<img src="https://docs.spring.io/spring-security/site/docs/current/reference/html5/images/servlet/architecture/delegatingfilterproxy.png" alt="delegatingfilterproxy" style="zoom:80%;" />

如上图所示，此代理Filter会从ApplicationContext中查询Bean Filter0来完成工作

##### 2）FilterChainProxy

FilterChainProxy是一个Bean，封装在DelegatingFilterProxy中。其是一个特殊的Bean Filter，会将一系列任务代理给SecurityFilterChain中的多个Filters（可以有多个FilterChainProxys）。关系图如下所示

<img src="https://docs.spring.io/spring-security/site/docs/current/reference/html5/images/servlet/architecture/filterchainproxy.png" alt="filterchainproxy" style="zoom:80%;" />

FilterChainProxy主要功能如下

:one: 管理SS中所有Filters执行的入口

:two: 可以清空SecurityContext来避免内存泄漏

:three: 通过设定HttpFireWall来避免恶意攻击

:four: 可以决定哪个SercurityFilterChain用于处理此次请求（不仅仅依据URL，可以是Request中的任意参数）

:five: 当多个SercurityFilterChain都匹配当前请求时，只有第一个匹配的Chain会被触发

##### 3）SecurityFilterChain

SecurityFilterChain是一系列Bean Filters构成的Bean Filter Chain，链条中的SecurityFilters的相对位置十分重要，决定了各个过滤器在过滤流程中的位置。

此外，由于可以注册多个Chains，所以Chains的相对位置也很重要，比如下面的例子，当发来/api/message请求时，只会匹配第一个Chain

<img src="https://docs.spring.io/spring-security/site/docs/current/reference/html5/images/servlet/architecture/multi-securityfilterchain.png" alt="multi securityfilterchain" style="zoom:80%;" />

##### 4）Security Filters

SS中的Filters的执行顺序大抵如下

- ChannelProcessingFilter

- WebAsyncManagerIntegrationFilter

- SecurityContextPersistenceFilter

- HeaderWriterFilter

- CorsFilter

- CsrfFilter

- LogoutFilter

- OAuth2AuthorizationRequestRedirectFilter

- Saml2WebSsoAuthenticationRequestFilter

- X509AuthenticationFilter

- AbstractPreAuthenticatedProcessingFilter

- CasAuthenticationFilter

- OAuth2LoginAuthenticationFilter

- Saml2WebSsoAuthenticationFilter

- [`UsernamePasswordAuthenticationFilter`](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-usernamepasswordauthenticationfilter)（继承于AbstractAuthenticationProcessingFilter）

- OpenIDAuthenticationFilter

- DefaultLoginPageGeneratingFilter

- DefaultLogoutPageGeneratingFilter

- ConcurrentSessionFilter

- [`DigestAuthenticationFilter`](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-digest)

- BearerTokenAuthenticationFilter

- [`BasicAuthenticationFilter`](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-basic)

- RequestCacheAwareFilter

- SecurityContextHolderAwareRequestFilter

- JaasApiIntegrationFilter

- RememberMeAuthenticationFilter

- AnonymousAuthenticationFilter

- OAuth2AuthorizationCodeGrantFilter

- SessionManagementFilter

- [`ExceptionTranslationFilter`](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-exceptiontranslationfilter)

  这个Filter用于**将抛出的**[`AccessDeniedException`](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/access/AccessDeniedException.html) 和 [`AuthenticationException`](https://docs.spring.io/spring-security/site/docs/current/api//org/springframework/security/core/AuthenticationException.html)异常**转化为HTTP 响应**，

  一般来说，其处理的逻辑顺序如下

  <img src="https://docs.spring.io/spring-security/site/docs/current/reference/html5/images/servlet/architecture/exceptiontranslationfilter.png" alt="exceptiontranslationfilter" style="zoom:50%;" />

  通过上图可以看出，在没有进行认证之前，如果访问被拒绝，就会抛出AccessDeniedException异常，并响应给客户端，否则开始进行认证流程。具体过程如下

  - 首先， `ExceptionTranslationFilter` 触发`FilterChain.doFilter(request, response)` 继续正常执行后续的代码，执行中
  - 如果用户没有被认证或者抛出了`AuthenticationException`, 那么开始认证
    - 清空 [SecurityContextHolder](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-securitycontextholder) 
    - `HttpServletRequest`被缓存到[`RequestCache`](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/web/savedrequest/RequestCache.html). 当用户认证成功时，`RequestCache`用于获取原有请求消息
    - `AuthenticationEntryPoint` 用于从客户端请求认证信息（密码）. 比如, 他会重定向到登陆页面，并发一个`WWW-Authenticate` 头
  - 否则，如果doFilter的执行过程中再次抛出了`AccessDeniedException`，那么请求被拒绝。`AccessDeniedHandler` 用于处理这个异常

- [`FilterSecurityInterceptor`](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authorization-filtersecurityinterceptor)（继承于AbstractSecurityInterceptor）

- SwitchUserFilter

#### 2.2.2 SS中的认证流程

在前面的`ExceptionTranslationFilter`执行中如果没有认证或者抛出`AuthenticationException`，就会触发认证流程

##### 1）相关组件及认证机制

- [SecurityContextHolder](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-securitycontextholder) - 用于存储已经认证的用户信息，默认会以ThreadLocal的形式存储（线程独有），其中包含的信息在失效时，FilterChainProxy会确保这些信息会被清除。如果可以保证多个线程安全访问信息，可以使用`SecurityContextHolder.INHERITABLETHREADLOCAL`机制，可以通过调用此类的静态方法实现配置

- [SecurityContext](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-securitycontext) - 从`SecurityContextHolder`中获取，其中存储了当前被认证用户的`Authentication`对象。为了避免多线程竞争的问题，常常在Holder中创建多个Contexts而不是直接从Holder中获取已有的Context

- [Authentication](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-authentication) - 作为`AuthenticationManager`的输入，存储了用户提供的待认证的信息或者当前存储在`SecurityContext`中的认证过的用户信息，**不同认证机制的Authentication实现也不同**，如果采用用户名密码认证机制，则会创建一个`UsernamePasswordAuthenticationToken`对象

  上面三者关系如下所示

  <img src="https://docs.spring.io/spring-security/site/docs/current/reference/html5/images/servlet/authentication/architecture/securitycontextholder.png" alt="securitycontextholder" style="zoom:67%;" />

  其中包含的信息如下

  - `principal` - 用户信息. 当使用用户名密码认证时，这个对象是[`UserDetails`](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-userdetails)的一个实例，**如果要自定义用户类，需要实现这个接口**
  - `credentials` - 一般为密码. 为了确保不被泄露，一般在认证结束后会被清除
  - `authorities` - 保存了多个 [`GrantedAuthority`](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-granted-authority)实例，一般为多个roles（基于角色的权限访问控制，只有固定角色可以访问对应资源。如果是基于用户名密码的机制，通常由[`UserDetailsService`](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-userdetailsservice)从数据库或内存中完成加载，不会保存在本地）或者scopes（基于资源的访问控制，拥有某个权限标识，可以分配给多个角色）

- [GrantedAuthority](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-granted-authority) - 根据`Authentication` 中的用户信息，查询到的被授予的权限(即roles, scopes, etc.)存储在这里。通过getAuthorities回去

- [AuthenticationManager](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-authenticationmanager) - 定义了SS中的filters如何进行认证，其**返回的`Authentication`对象**会被存储在SecurityContextHolder中。

- [ProviderManager](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-providermanager) - `AuthenticationManager`的最常用实现。其会将认证任务代理给一系列`AuthenticationProvider`对象来执行，每一个`AuthenticationProvider`对象**独立的完成不同类型的**认证（如用户名密码认证）。如下所示

  <img src="https://docs.spring.io/spring-security/site/docs/current/reference/html5/images/servlet/authentication/architecture/providermanager.png" alt="providermanager" style="zoom: 50%;" />

  每一个ProviderManagers会共享一个父`AuthenticationManager`(一般为ProviderManager)，当自己的Providers都不能完成认证时，会交由父Manager来处理，这也是为什么不同的SecurityFilterChain会有一部分认证流程是相同的，如下图所示

  <img src="https://docs.spring.io/spring-security/site/docs/current/reference/html5/images/servlet/authentication/architecture/providermanagers-parent.png" alt="providermanagers parent" style="zoom:50%;" />

  ProviderManager在认证结束后，会将credentials清除，如果想要保存在缓存中支持多次相同的认证，可以通过让`ProviderManager`中的`eraseCredentialsAfterAuthentication`属性失效来实现。

- [AuthenticationProvider](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-authenticationprovider) - `ProviderManager`用其来完成特定类型的认证流程，如果采用用户名密码认证机制，一般为使用 [`DaoAuthenticationProvider`](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-daoauthenticationprovider) 

- Request Credentials with `AuthenticationEntryPoint` - 当请求时没有发送用户名密码时，此组件用于向客户端请求认证信息 (i.e. redirecting to a log in page, sending a `WWW-Authenticate` response, etc.)

- [AbstractAuthenticationProcessingFilter](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-abstractprocessingfilter) - 用于完成认证的基础Filter. 提供了认证过程中各个组件是如何相互配合的逻辑，在可以获取到可用的认证信息时，开始执行认证，流程如下

  <img src="https://docs.spring.io/spring-security/site/docs/current/reference/html5/images/servlet/authentication/architecture/abstractauthenticationprocessingfilter.png" alt="abstractauthenticationprocessingfilter" style="zoom:80%;" />

  - 首先，`AbstractAuthenticationProcessingFilter`会根据用户提交的`HttpServletRequest`中创建出一个`Authentication`对象
  - 之后，此对象交由`AuthenticationManager`进行认证
  - 如果认证失败
    - The [SecurityContextHolder](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authentication-securitycontextholder)被清除
    - `RememberMeServices.loginFail` 被触发，如果没有开启"记住我"则不会有操作
    - `AuthenticationFailureHandler` 被触发
  - 如果认证成功
    - `SessionAuthenticationStrategy`会被提醒有一个新的登录
    - `Authentication`被保存进`SecurityContextHolder`中，之后`SecurityContextPersistenceFilter`会将`SecurityContext` 保存进`HttpSession`
    - `RememberMeServices.loginSuccess`被触发，如果没有开启"记住我"则不会有操作
    - `ApplicationEventPublisher`将 `InteractiveAuthenticationSuccessEvent`发布出去
    - `AuthenticationSuccessHandler` 被触发

##### 2）用户名密码认证机制

采用这种方式认证需要解决两个数据源的问题

- 提交的带验证的认证信息，有三种形式
  - 表单提交认证
  - 基本HTTP认证
  - 摘要认证（不推荐❌）

- 存储的被参考的认证信息
  - 存储在内存中`In-Memory Authentication`
  - 存储在关系型数据库中，利用JDBC查询
  - 利用`UserDetailService`定制数据
  - 利用LDAP（Light Directory Access Portocol）型（目录）数据库存储

这里主要解释基于**查询关系型数据库**的**采用表单登录**的**用户名密码认证机制**

###### （1）表单提交

执行流程如下图所示

![loginurlauthenticationentrypoint](https://docs.spring.io/spring-security/site/docs/current/reference/html5/images/servlet/authentication/unpwd/loginurlauthenticationentrypoint.png)

- 首先用户会在没有认证的情况下请求一个需要认证的`/private`资源

- [`FilterSecurityInterceptor`](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authorization-filtersecurityinterceptor) 会将此请求拒绝，并抛出`AccessDeniedException`

- 基于`ExceptionTranslationFilter`的逻辑，由于用户没有认证，那么会调用`AuthenticationEntryPoint` （通常为`LoginUrlAuthenticationEntryPoint`）来请求客户端提交认证信息，浏览器会被重定向到登录界面

  ***SS会提供一个默认的登录页面，可以自定义该页面，并提供对应的Controller***

当用户名和密码提交后，`UsernamePasswordAuthenticationFilter` （继承于 `AbstractAuthenticationProcessingFilter`）开始执行认证，流程如下

<img src="https://docs.spring.io/spring-security/site/docs/current/reference/html5/images/servlet/authentication/unpwd/usernamepasswordauthenticationfilter.png" alt="usernamepasswordauthenticationfilter" style="zoom: 67%;" />

上面流程中，在AuthenticationManager执行认证过程中，会需要查询存储在数据库中的认证信息，下面小节介绍如何查询

###### （2）查询认证信息

- **JdbcDaoImpl**

  这个是SS官方提供的实现了`UserDetailsService`的用于查询用户名密码的实现类。`JdbcUserDetailsManager` 继承了`JdbcDaoImpl`，并提供了对于UserDetails（用于获取权限信息）的管理

- **使用Mybatis查询**

  使用Mybatis之前，需要先了解SS中定义的一些接口规范

  - #### UserDetails

    这个对象通过`UserDetailsService`返回，`DaoAuthenticationProvider`在验证了UserDetails后会返回一个`Authentication`对象，其中的principal属性就是UserDetails接口的实现类

  - #### UserDetailsService

    `DaoAuthenticationProvider`调用`UserDetailsService`来从数据库中获取存储的用户信息（是UserDetails的实现类对象），如果想要定制`UserDetailsService`只需要将自定义的实现类上添加`@Bean`标签即可。

    > 这里需要注意，这个@Bean标签只有在`DaoAuthenticationProvider`的bean生成之前且没有注入到`AuthenticationManagerBuilder`之前才可以生效。
    >
    > **`AuthenticationManagerBuilder`是用于构建`AuthenticationManager`（此处对应的实现类为`ProviderManager`）的类，其中存储了要使用的AuthenticationProviders和UserDetailsServices以及所有AuthenticationManager共享的父AuthenticationManager对象**
    >
    > 
    >
    > **所以如果想使自定义的Bean生效。不仅需要配置@Bean标签，还需要调用AuthenticationManagerBuilder的userDetailsService方法将自定义的实现类传递给defaultUserDetailsService成员变量**，**这样AuthenticationManagerBuilder才会使用传给他的参数去构建ProviderManager**
    >
    > 
    >
    > **而AuthenticationManagerBuilder的配置又可以在WebSecurityConfigurerAdapter中通过configure函数完成，所以可以自定义类继承与WebSecurityConfigurerAdapter并完成重写**

- #### PasswordEncoder

  由于存储在数据库中的用户信息的密码不会以明文的形式存在，所以需要提供一个密码编码器，将密码编为密文存储在数据库中。

  如果想要定制编码器，只需要在配置类下添加@Bean标签并返回指定的PasswordEncoder具体实现类对象即可

  - #### DaoAuthenticationProvider

    这个类是`AuthenticationProvider`的实现类，其会利用UserDetailService（用于查询用户信息）和密码编码器（对密文解密）得到存储在数据库中的用户信息的解密版本（`UserDetails对象`）并于用户提交的认证信息（`UsernamePasswordAuthenticationToken对象`）比较。

    如果认证失败，中间会抛出异常可能为`UsernameNotFoundException`或者`AuthenticationException`

    如果认证成功，会将返回一个新的`UsernamePasswordAuthenticationToken对象`，创建时还会设定其中的details属性为`WebAuthenticationDetails`对象，其中存储了登录者的**远程地址和sessionId**

#### **至此SS中登录流程就结束了**

### *<u>Main --- vhr中的自定义登录</u>*

vhr中采用了用户名密码的机制，通过SS的登录流程可以看出，我们可以干预如下几个点

:one: 密码的编码器PasswordEncoder

:two: UserDetails的实现类，也即存储在数据库中的用户信息（其中包括用户名、密码、权限等），在认证成功后会作为`UsernamePasswordAuthenticationToken`的principal字段存在

:three: UserDetailSevice的实现类，可以改为自定义数据库查询方式（使用Mybatis）

:four: 由于在认证信息提交后，**认证的入口是UsernamePasswordAuthenticationFilter，为了支持验证码和自定义异常，可以继承此类，重写其方法并将此Filter替换为自定义的Filter**

#### 2.2.3 自定义PasswordEncoder

可以在任意配置类下添加下面代码即可指定密码编码器

```java
@Bean
PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

#### 2.2.4 自定义UserDetails实现类

```java
public class Hr implements UserDetails {
    private Integer id;

    private String name;

    private String phone;

    private String telephone;

    private String address;

    private Boolean enabled;

    private String username;

    private String password;

    private String userface;

    private String remark;
    private List<Role> roles;
    /**
    * 省去getter和setter
    */

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    @JsonIgnore
    //通过org.javaboy.vhr.service.HrService.loadUserByUsername方法查找到hr的信息后，List<Role>就会为Hr的一个成员变量，
    //将此集合用SimpleGrantedAuthority包装
    public Collection<? extends GrantedAuthority> getAuthorities() {
        List<SimpleGrantedAuthority> authorities = new ArrayList<>(roles.size());
        for (Role role : roles) {
            authorities.add(new SimpleGrantedAuthority(role.getName()));
        }
        return authorities;
    }
    /**
    * 省去getter和setter
    */
}
```

此类是用于封装从数据库中根据用户名查询得到的用户信息和相关认证信息（密码和角色）

#### 2.2.5 自定义UserDetailSevices

```java
@Service
public class HrService implements UserDetailsService {
    @Autowired
    HrMapper hrMapper;
    @Autowired
    HrRoleMapper hrRoleMapper;

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
    /**
    * 省略其他部分
    */
}
```

这里重写了`loadUserByUsername`方法，此方法的调用栈如下所示

- `DaoAuthenticationProvider#retrieveUser`中调用`loadUserByUsername`

- `AbstractUserDetailsAuthenticationProvider#authenticate`中调用`retrieveUser`

- `ProviderManager#authenticate`中调用上一个`authenticate`

- `UsernamePasswordAuthenticationFilter#attemptAuthentication`中调用上一个`authenticate`

  **这个Filter也即认证的开始**

#### 2.2.6 自定义UsernamePasswordAuthenticationFilter

上面提到，UsernamePasswordAuthenticationFilter是获取到客户端提交认证信息后的入口，其中的attemptAuthentication方法是这个类的核心，代码如下

```java
public Authentication attemptAuthentication(HttpServletRequest request,
      HttpServletResponse response) throws AuthenticationException {
   if (postOnly && !request.getMethod().equals("POST")) {
      throw new AuthenticationServiceException(
            "Authentication method not supported: " + request.getMethod());
   }

   String username = obtainUsername(request);
   String password = obtainPassword(request);

   if (username == null) {
      username = "";
   }

   if (password == null) {
      password = "";
   }

   username = username.trim();

   UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
         username, password);

   // Allow subclasses to set the "details" property
   setDetails(request, authRequest);

   return this.getAuthenticationManager().authenticate(authRequest);
}
```

可见，其主要操作是将request封装为UserDetails。

由于vhr中要**增加验证码的校验环节**，并**设定UsernamePasswordAuthenticationToken中的principal属性为Hr对象**

所以自定义了`LoginFilter`继承于UsernamePasswordAuthenticationFilter，并重写了其中的方法，代码如下

```java
public class LoginFilter extends UsernamePasswordAuthenticationFilter {
    @Autowired
    SessionRegistry sessionRegistry;
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
        //原本的执行顺序为WebAsyncManagerIntegrationFilter（使得SecurityContext可以被异步访问）-->SecurityContextPersistenceFilter（将此次的请求和即将返回的响应存储在SecurityContext中）
        //-->HeaderWriterFilter-->LogoutFilter-->UsernamePasswordAuthenticationFilter
        //由于LoginFilter继承了此类且重写了其中的方法，因而执行下面的方法

        //前端发来的doLogin的POST请求，url在SecurityConfig中指定为doLogin
        if (!request.getMethod().equals("POST")) {
            throw new AuthenticationServiceException(
                    "Authentication method not supported: " + request.getMethod());
        }
        String verify_code = (String) request.getSession().getAttribute("verify_code");
        //如果requets包含Json类型的数据，则需要将其转换为Map集合，从中获取到请求信息后，将其封装为UsernamePasswordAuthenticationToken类型的认证信息
        if (request.getContentType().contains(MediaType.APPLICATION_JSON_VALUE) || request.getContentType().contains(MediaType.APPLICATION_JSON_UTF8_VALUE)) {
            Map<String, String> loginData = new HashMap<>();
            try {
                loginData = new ObjectMapper().readValue(request.getInputStream(), Map.class);
            } catch (IOException e) {
            }finally {
                String code = loginData.get("code");
                checkCode(response, code, verify_code);
            }
            String username = loginData.get(getUsernameParameter());
            String password = loginData.get(getPasswordParameter());
            if (username == null) {
                username = "";
            }
            if (password == null) {
                password = "";
            }
            username = username.trim();
            UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
                    username, password);
            setDetails(request, authRequest);
            Hr principal = new Hr();
            principal.setUsername(username);
            //把此次请求的sessionId和UserDetail注册到此次会话中
            sessionRegistry.registerNewSession(request.getSession(true).getId(), principal);
            //之后利用认证信息调用DaoAuthenticationProvider开始认证
            return this.getAuthenticationManager().authenticate(authRequest);
        } else {
            checkCode(response, request.getParameter("code"), verify_code);
            return super.attemptAuthentication(request, response);
        }
    }

    public void checkCode(HttpServletResponse resp, String code, String verify_code) {
        if (code == null || verify_code == null || "".equals(code) || !verify_code.toLowerCase().equals(code.toLowerCase())) {
            //验证码不正确
            throw new AuthenticationServiceException("验证码不正确");
        }
    }
}
```

其余部分和原有方法的逻辑类似。

之后需要**在配置类中将此Filter生成为Bean**，且需要对此LoginFilter的一些属性进行设置，如下

```java
@Bean
LoginFilter loginFilter() throws Exception {
    LoginFilter loginFilter = new LoginFilter();
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
    //登录失败后执行下面的方法
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
    //设定认证管理器（其中指定了认证提供者，此处使用DaoAuthenticationProvider--retrieveUser和additionalCheck及其父类--authenticate方法）
    loginFilter.setAuthenticationManager(authenticationManagerBean());
    //指定过滤的url
    loginFilter.setFilterProcessesUrl("/doLogin");
    ConcurrentSessionControlAuthenticationStrategy sessionStrategy = new ConcurrentSessionControlAuthenticationStrategy(sessionRegistry());
    //设定每个用户登录认证后和服务端最多保留一个session
    sessionStrategy.setMaximumSessions(1);
    //将上面的session策略注册给LoginFilter
    loginFilter.setSessionAuthenticationStrategy(sessionStrategy);
    return loginFilter;
}
```

**由于默认的`UsernamePasswordAuthenticationFilter`已经被注册，所以需要调用HttpSecurity的addFilterAt方法，将默认的filter替换为上面的bean，如下**

`http.addFilterAt(loginFilter(), UsernamePasswordAuthenticationFilter.class);`

#### 2.2.7 登录成功后

认证成功后会调用`DaoAuthenticationProvider#createSuccessAuthentication`生成一个新的`UsernamePasswordAuthenticationToken对象`，设定其中的属性值如下

- 将principal设定为从数据库中查询得到的Hr对象（其中包含了authorities）
- credentials中存储的密码会被清除
- 将details设定为`WebAuthenticationDetails`对象，其中存储了登录者的**远程地址和sessionId**
- **从Hr中获取authorities并赋给UsernamePasswordAuthenticationToken的父类AbstractAuthenticationToken中的authorities属性**（关键！）

***<u>这里额外关注一下authorities的赋值</u>***，在创建Token时，会先调用DaoAuthenticationProvider的createSuccessAuthentication方法如下

```java
protected Authentication createSuccessAuthentication(Object principal,
      Authentication authentication, UserDetails user) {
   boolean upgradeEncoding = this.userDetailsPasswordService != null
         && this.passwordEncoder.upgradeEncoding(user.getPassword());
   if (upgradeEncoding) {
      String presentedPassword = authentication.getCredentials().toString();
      String newPassword = this.passwordEncoder.encode(presentedPassword);
      user = this.userDetailsPasswordService.updatePassword(user, newPassword);
   }
   return super.createSuccessAuthentication(principal, authentication, user);
}
```

传入的三个参数

- principal 刚刚认证通过的**Hr对象**（没有强转为字符串的话，就是user）
- 之前的Token，主要为了从中获取credentials和details
- 刚刚认证通过的**Hr对象**

这里又会调用父方法`AbstractUserDetailsAuthenticationProvider#createSuccessAuthentication`，如下

```java
protected Authentication createSuccessAuthentication(Object principal,
      Authentication authentication, UserDetails user) {
   // Ensure we return the original credentials the user supplied,
   // so subsequent attempts are successful even with encoded passwords.
   // Also ensure we return the original getDetails(), so that future
   // authentication events after cache expiry contain the details
   UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(
         principal, authentication.getCredentials(),
         authoritiesMapper.mapAuthorities(user.getAuthorities()));
   result.setDetails(authentication.getDetails());

   return result;
}
```

使用user也即Hr的getAuthorities方法时，会将`List<Role> roles`集合转化为`List<SimpleGrantedAuthority> authorities`集合返回

其中要注意`authoritiesMapper.mapAuthorities(user.getAuthorities())`这一句，`authoritiesMapper`是AbstractUserDetailsAuthenticationProvider的成员变量，此处会调用`SimpleAuthorityMapper`的`mapAuthorities`方法，将从Hr中获取到的权限信息转存为一个`HashSet`并返回，这个`HashSet`作为`UsernamePasswordAuthenticationToken`的构造器参数传入，**最终**，**赋值给父类的`authorities`属性！**

> **这里有两个注意点**
>
> :one: `private final Collection<GrantedAuthority> authorities`仍为SimpleGrantedAuthority类型的集合
>
> :two: 调用mapAuthorities方法时，会使用mapAuthority方法来**检查**传入的权限字符串是否是**以`ROLE_`为前缀**，所以这要求存储在数据库中的角色信息必须**也是以`ROLE_`为前缀**
>
> :three: 在存储权限信息时，之所以选择`SimpleGrantedAuthority`是因为SS中的AuthenticationProvider默认使用这个类，此外这个类可以很好的将字符串和权限进行转换，这样可以保证调用get方法获取权限时不返回[null](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#authz-authorities)

**这个对象会被存入HttpSession和SecurityContext中，用于之后的授权操作**(获取Token中的authorities属性)

**下面为登录时封装了用户提交的认证信息的Token对象（右）和新创建的Token（左）的对比图，注意观察其中的principal属性**

![image-20201014164013774](https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201014164013774.png)