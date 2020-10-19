## 3. 授权流程

### 3.1 Spring Security中的认证

#### 3.1.1 认证架构

##### 1）SimpleGrantedAuthority

由于在认证成功后，会在session中保存认证成功的UsernamePasswordAuthenticationToken对象，主要有如下三个

- principal
- credentials
- authorities

其中的authorities属性会被`AccessDecisionManager`用于权限判断的参考，通过Token中的**getAuthorities**方法获得。

由于在创建认证成功的UsernamePasswordAuthenticationToken对象时，使用的是SimpleGrantedAuthority来封装权限，因而获取到的权限也是这个类的对象，**在实际比较时会将其中存储的字符串返回并用于比较**

##### 2）Pre-Invocation Handling

在进行方法触发和请求处理时，都会先进行一次**预处理**（也即鉴权），由`AbstractSecurityInterceptor`通过调用`AccessDecisionManager`中的方法来实现。其中有三个重要的方法

```java
//用于判读要触发方法或者获取资源的Invocation可否继续执行
void decide(Authentication authentication, Object secureObject,
    Collection<ConfigAttribute> attrs) throws AccessDeniedException;
//检查是否支持这种权限信息
boolean supports(ConfigAttribute attribute);
//查看配置的AccessDecisionManager是否支持鉴权
boolean supports(Class clazz);
```

##### 基于投票的AccessDecisionManager的实现

![access decision voting](https://docs.spring.io/spring-security/site/docs/current/reference/html5/images/access-decision-voting.png)

使用这种机制将调用一系列AccessDecisionVoter的实现类对象来完成鉴权，有如下三个方法

```java
//投票后返回ACCESS_ABSTAIN（弃权）, ACCESS_DENIED（拒绝） 或者 ACCESS_GRANTED（同意）
int vote(Authentication authentication, Object object, Collection<ConfigAttribute> attrs);

boolean supports(ConfigAttribute attribute);

boolean supports(Class clazz);
```

AccessDecisionManager的实现**根据投票机制**不同可以分为三类

- ConsensusBased

  比较除了弃权者的所有票数，得出最终结果，当所有人弃权，有额外的条件判断

- AffirmativeBased

  只要有>=1个支持票，就同意，当所有人弃权，有额外的条件判断

- UnanimousBased

  只要有反对票，就拒绝，当所有人弃权，有额外的条件判断


上面的三个实现类可以看作是**三种"投票委员会"**，每一个都包含了一群AccessDecisionVoter（**"投票员"**）,AccessDecisionVoter**按照投票标准**分为下面三类

###### （1）**RoleVoter**（与SimpleAuthorityMapper和SimpleGrantedAuthority搭配使用）

这是最常用的投票器，其会将ConfigAttribute当作简单的权限名称（role）。其只会对以**"Role_"**前缀开头的ConfigAttribute做判断，如果前缀都匹配，后缀也匹配，支持，如果后缀不匹配不支持，如果没有前缀匹配则弃权

###### （2）**AuthenticatedVoter**

用于区分匿名、认证过的和记住我的用户，当设定`IS_AUTHENTICATED_ANONYMOUSLY`参数来允许匿名登录时，将由此Voter来处理

###### （3）**Custom Voters**

自定义投票器

##### 3） After Invocation Handling

![after invocation](https://docs.spring.io/spring-security/site/docs/current/reference/html5/images/after-invocation.png)

在鉴权通过后，需要开始调用`secureObject`中指定的方法或者请求，执行后会返回一个object，如果想对此对象进行修改后再返回，就需要通过`AfterInvocationManager`调用`AbstractSecurityInterceptor#afterInvocation`来完成 

##### 4）Hierarchical Roles

如果使用到的roles存在上下级的关系，且**下级可以访问的要保证上级也能访问**，为此可以使用`RoleHierarchyVoter`，常规的配置如下

```xml
<bean id="roleVoter" class="org.springframework.security.access.vote.RoleHierarchyVoter">
    <constructor-arg ref="roleHierarchy" />
</bean>
<bean id="roleHierarchy"
        class="org.springframework.security.access.hierarchicalroles.RoleHierarchyImpl">
    <property name="hierarchy">
        <value>
            ROLE_ADMIN > ROLE_STAFF
            ROLE_STAFF > ROLE_USER
            ROLE_USER > ROLE_GUEST
        </value>
    </property>
</bean>
```

上面的value中指定了ROLE之间的上下级关系

#### 3.1.2 使用FilterSecurityInterceptor对HttpServletRequest授权

##### 1）Pre

根据上一节中给出的SS中filter的执行顺序，由于使用的是**用户名密码机制**，在认证成功的情况下，会直接跳到`FilterSecurityInterceptor`过滤器，其是`AbstractSecurityInterceptor`的子类

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
- [`FilterSecurityInterceptor`](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-authorization-filtersecurityinterceptor)（继承于AbstractSecurityInterceptor）
- SwitchUserFilter

##### 2）FilterSecurityInterceptor授权流程

![filtersecurityinterceptor](https://docs.spring.io/spring-security/site/docs/current/reference/html5/images/servlet/authorization/filtersecurityinterceptor.png)

- `FilterSecurityInterceptor`从`SecurityContextHolder`中获取`Authentication对象`（用户名密码下也即UsernamePasswordAthenticationToken）
- `FilterSecurityInterceptor`根据传入的`HttpServletRequest`, `HttpServletResponse`,和`FilterChain`对象创建[`FilterInvocation`](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/web/FilterInvocation.html)对象（作为Security Object）
- 将刚创建好的`FilterInvocation`对象传给`SecurityMetadataSource`，后者从数据库中查出请求资源和权限之间的对应关系（getAttributes方法）后返回一个`ConfigAttribute`集合
- 最终将`Authentication对象`,`FilterInvocation对象`和刚查出的`ConfigAttribute`集合传给`AccessDecisionManager`开始进行决策
  - 如果鉴权失败，会抛出`AccessDeniedException` ，在这个例子中，会由[`ExceptionTranslationFilter`](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-exceptiontranslationfilter) 来处理异常
  - 如果鉴权成功，`FilterSecurityInterceptor`**调用FilterChain继续执行剩下的Filters**

默认情况下，**授权必须发生在认证之后**。

> 可以通过下面这段来手动配置授权逻辑
>
> ```java
> protected void configure(HttpSecurity http) throws Exception {
>     http
>         // ...
>         .authorizeRequests(authorize -> authorize                                  
>             .mvcMatchers("/resources/**", "/signup", "/about").permitAll()         
>             .mvcMatchers("/admin/**").hasRole("ADMIN")                             
>             .mvcMatchers("/db/**").access("hasRole('ADMIN') and hasRole('DBA')")   
>             .anyRequest().denyAll()                                                
>         );
> }
> ```

#### 3.1.3 基于表达式的访问控制

使用Spring的EL表达式来提供额外的鉴权逻辑

#### 3.1.4Secure Object的实现

为了保证方法能被安全调用，提供了Spring AOP和AspectJ两种方式来解决

##### 1）联合AOP(MethodInvocation)的Security Interceptor

使用`MethodSecurityInterceptor`，其中包含`MethodSecurityMetadataSource`对象（用于获取访问资源需要的权限信息），和`AccessDecisionManager`对象（用于设定鉴权对象）。此类的对象用于保证方法被安全的访问，得益于Spring AOP的特点，这个对象中的`MethodSecurityMetadataSource`对象可以被共享

常见的xml参见[官方文档](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#el-access-web)

##### 2）AspectJ Security Interceptor

使用此种方式，`AspectJSecurityInterceptor`会被AspectJ编译器编入字节码中，在编译期完成AOP织入

#### 3.1.5方法安全性

参见[官方文档](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#jc-method)

### 3.2 vhr中的授权

#### 3.2.1 发出请求&开始鉴权

##### 1）doFilter

vhr项目中，在认证成功后，会立马请求`/system/config/menu`，用于显示页面的菜单信息。根据SS中的授权流程，首先将来到`FilterSecurityInterceptor#doFilter`方法，此方法中，会创建新的FilterInvocation（将request和response对象与filterChain包装在一起），可以看到原有的过滤器链和额外的过滤器链（Spring Security添加的，通过getFilters(fwRequest)获得），如下图所示

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201015110910138.png" alt="image-20201015110910138" style="zoom: 200%;" />

##### 2）invoke

之后将传入fi调用invoke方法，代码如下

```java
public void invoke(FilterInvocation fi) throws IOException, ServletException {
   if ((fi.getRequest() != null)
         && (fi.getRequest().getAttribute(FILTER_APPLIED) != null)
         && observeOncePerRequest) {
      // filter already applied to this request and user wants us to observe
      // once-per-request handling, so don't re-do security checking
      fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
   }
   else {
      // first time this request being called, so perform security checking
      if (fi.getRequest() != null && observeOncePerRequest) {
         fi.getRequest().setAttribute(FILTER_APPLIED, Boolean.TRUE);
      }

      InterceptorStatusToken token = super.beforeInvocation(fi);

      try {
         fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
      }
      finally {
         super.finallyInvocation(token);
      }

      super.afterInvocation(token, null);
   }
}
```

首先会判断是否是第一次请求（如果设定observeOncePerRequest为true，那么需要通过查看发来的请求中是否有FILTER_APPLIED参数，从Cookie中获得）

- 如果是，则需要给客户端发送一个Cookie包含这个参数，之后
  - 调用beforeInvocation方法
  - 如果鉴权成功，继续执行FilterChain剩下的过滤器
  - 执行finallyInvocation
  - 执行afterInvocation
- 如果不是，则代表授权过，
  - 如果要求observeOncePerRequest，则直接放行
  - 否则，每次都需要重复一遍安全检查

下面按照第一次访问继续追踪

##### 3）beforeInvocation

接下来调用父类`AbstractSecurityInterceptor#beforeInvocation`方法，代码如下

```java
protected InterceptorStatusToken beforeInvocation(Object object) {
   Assert.notNull(object, "Object was null");
   final boolean debug = logger.isDebugEnabled();

   if (!getSecureObjectClass().isAssignableFrom(object.getClass())) {
      throw new IllegalArgumentException(
            "Security invocation attempted for object "
                  + object.getClass().getName()
                  + " but AbstractSecurityInterceptor only configured to support secure objects of type: "
                  + getSecureObjectClass());
   }

   Collection<ConfigAttribute> attributes = this.obtainSecurityMetadataSource()
         .getAttributes(object);

   if (attributes == null || attributes.isEmpty()) {
      if (rejectPublicInvocations) {
         throw new IllegalArgumentException(
               "Secure object invocation "
                     + object
                     + " was denied as public invocations are not allowed via this interceptor. "
                     + "This indicates a configuration error because the "
                     + "rejectPublicInvocations property is set to 'true'");
      }

      if (debug) {
         logger.debug("Public object - authentication not attempted");
      }

      publishEvent(new PublicInvocationEvent(object));

      return null; // no further work post-invocation
   }

   if (debug) {
      logger.debug("Secure object: " + object + "; Attributes: " + attributes);
   }

   if (SecurityContextHolder.getContext().getAuthentication() == null) {
      credentialsNotFound(messages.getMessage(
            "AbstractSecurityInterceptor.authenticationNotFound",
            "An Authentication object was not found in the SecurityContext"),
            object, attributes);
   }

    //如果设定了需要额外认证，则会再触发一次认证操作
   Authentication authenticated = authenticateIfRequired();

   // Attempt authorization
   try {
      this.accessDecisionManager.decide(authenticated, object, attributes);
   }
   catch (AccessDeniedException accessDeniedException) {
      publishEvent(new AuthorizationFailureEvent(object, attributes, authenticated,
            accessDeniedException));

      throw accessDeniedException;
   }

   if (debug) {
      logger.debug("Authorization successful");
   }

   if (publishAuthorizationSuccess) {
      publishEvent(new AuthorizedEvent(object, attributes, authenticated));
   }

   // Attempt to run as a different user
   Authentication runAs = this.runAsManager.buildRunAs(authenticated, object,
         attributes);

   if (runAs == null) {
      if (debug) {
         logger.debug("RunAsManager did not change Authentication object");
      }

      // no further work post-invocation
      return new InterceptorStatusToken(SecurityContextHolder.getContext(), false,
            attributes, object);
   }
   else {
      if (debug) {
         logger.debug("Switching to RunAs Authentication: " + runAs);
      }

      SecurityContext origCtx = SecurityContextHolder.getContext();
      SecurityContextHolder.setContext(SecurityContextHolder.createEmptyContext());
      SecurityContextHolder.getContext().setAuthentication(runAs);

      // need to revert to token.Authenticated post-invocation
      return new InterceptorStatusToken(origCtx, true, attributes, object);
   }
}
```

关注其中的下面几个部分

###### （1）getAttributes(object)

这里将调用`SecurityMetadataSource#getAttributes`方法，这里会调用vhr中自定义的`CustomFilterInvocationSecurityMetadataSource`类中重写的方法，重写的方法体如下

```java
public Collection<ConfigAttribute> getAttributes(Object object) throws IllegalArgumentException {
    String requestUrl = ((FilterInvocation) object).getRequestUrl();
    List<Menu> menus = menuService.getAllMenusWithRole();
    for (Menu menu : menus) {
        if (antPathMatcher.match(menu.getUrl(), requestUrl)) {
            List<Role> roles = menu.getRoles();
            String[] str = new String[roles.size()];
            for (int i = 0; i < roles.size(); i++) {
                str[i] = roles.get(i).getName();
            }
            // public static List<ConfigAttribute> createList(String... attributeNames) {
            //    Assert.notNull(attributeNames, "You must supply an array of attribute names");
            //    List<ConfigAttribute> attributes = new ArrayList<>(
            //          attributeNames.length);
            //
            //    for (String attribute : attributeNames) {
            //       attributes.add(new SecurityConfig(attribute.trim()));
            //    }
            //
            //    return attributes;
            // }
            //由上面代码可以看出，返回的是类型为SecurityConfig的list集合
            return SecurityConfig.createList(str);
        }
    }
    return SecurityConfig.createList("ROLE_LOGIN");
}
```

通过调用menuService的方法获得全部的资源（url）和权限的对应信息存为`menus`集合，之后从传入的fi对象中取出请求的url，之后遍历刚刚查到的`menus`集合，查找url匹配的条目（menu对象），将此条目得到后，将条目中存储的权限信息转换为字符串数组再转换为`SecurityConfig`（ConfigAttribute的子类）数组，并将其返回

如果请求的url没有找到对应的条目，则返回一个`ROLE_LOGIN`的ConfigAttribute数组

> 需要注意到，默认的`DefaultFilterInvocationSecurityMetadataSource`中会存在一个`Map<RequestMatcher, Collection<ConfigAttribute>> requestMap`对象，其中RequestMatcher用于匹配HttpServletRequest，其对应的ConfigAttribute集合是这个请求需要的权限。
>
> 由于**vhr中并没有存储这样一个Map集合对象**，为了能够实现请求资源和对应权限的匹配，采用从数据库中查询并利用`AntPathMatcher`对象进行手动url匹配，并返回对应的权限信息

###### （2）decide(authenticated, object, attributes)

这里会调用`AccessDecisionManager#decide`方法，开始进行认证，由于vhr中自定义了`CustomUrlDecisionManager`并重写了其中的三个关键的方法，所以会执行重写的方法，代码如下

```java
public void decide(Authentication authentication, Object object, Collection<ConfigAttribute> configAttributes) throws AccessDeniedException, InsufficientAuthenticationException {
    for (ConfigAttribute configAttribute : configAttributes) {
        //由于CustomFilterInvocationSecurityMetadataSource类中返回的是SecurityConfig的集合
        //调用getAttribute方法也即返回每一个SecurityConfig中存储的角色字符串信息
        String needRole = configAttribute.getAttribute();
        //如果当前的请求需要的角色是LOGIN，那么进行下面判断
        if ("ROLE_LOGIN".equals(needRole)) {
            if (authentication instanceof AnonymousAuthenticationToken) {
                throw new AccessDeniedException("尚未登录，请登录!");
            }else {
                return;
            }
        }
        Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
        for (GrantedAuthority authority : authorities) {
            if (authority.getAuthority().equals(needRole)) {
                return;
            }
        }
    }
    throw new AccessDeniedException("权限不足，请联系管理员!");
}
```

这里会遍历刚刚查询得到的访问url需要的权限信息（ConfigAttribute数组），将每一项（ConfigAttribute）转换为字符串，之后进行比较，

- 如果是`ROLE_LOGIN`，那么代表没有找到此次访问需要的权限信息
  - 如果是匿名登陆，则会抛出AccessDeniedException异常，交由[`ExceptionTranslationFilter`](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#servlet-exceptiontranslationfilter)处理
  - 否则，则代表是认证过的用户，则不做授权，直接返回
- 否则，将从之前认证过的Token中取出其拥有的权限（此处为SimpleGrantedAuthority），如果找到匹配，则代表授权成功，直接返回


> support方法默认返回**true**

###### （3）返回对象

最终，会返回`InterceptorStatusToken`对象，其中封装了传入的三个参数（Token，ConfigAttribute，FilterInvocation）

##### 4）fi.getChain().doFilter

如果授权成功，将继续执行过滤器链，最后会分别调用finallyInvocation和afterInvocation

#### 3.2.2 配置自定义类

上面使用了两个自定义类`CustomFilterInvocationSecurityMetadataSource`和`CustomUrlDecisionManager`

但是仅仅用`@Component`标记并不能使两个类生效，这是因为`FilterSecurityInterceptor`在最初配置时），会采用默认的两个Bean，分别是`DefaultFilterInvocationSecurityMetadataSource`和`AffirmativeBased`，所以需要替换掉原来默认的Bean。

> 具体的自定义方式参见我的另一篇文章[定制FilterSecurityInterceptor](/backend/OpenSourceProjs/vhr/vhr_%231_ss.md)
