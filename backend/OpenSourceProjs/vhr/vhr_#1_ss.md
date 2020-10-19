## #1.定制FilterSecurityInterceptor

> *本部分回答了下面几个问题*
>
> ***Q1***：RoleVoter起作用了吗？
>
> **A1：**没有，因为自定义的CustomUrlDecisionManager中重写了support和decide方法，是用自己定义的逻辑实现的鉴权，并没有使用RoleVoter等AccessDecisionVoter对象。之所以保留**ROLE_**的前缀（使用SimpleGrantedAuthority类必须遵守的规范），是为了符合Spring Security中的规范。
>
> 
>
> ***Q2***：如何定制FilterSecurityInterceptor中使用的组件？（如`FilterInvocationSecurityMetadataSource`，`AccessDecisionManager`等）
>
> **A2：**通过使用set方法设定，可以通过添加自定义的postProcessor，并在其中的postProcess方法中调用set方法
>
> 
>
> ***Q3***：Spring Security中没有@Bean标签是如何将组件注入到容器中？
>
> **A3：**通过`AutowireBeanFactoryObjectPostProcessor`中的postProcess方法实现手动注入

### #1.1 SS对Java对象增强

Spring Security中为了简化配置并没有使用@Bean标签或xml配置的方式向IoC容器中注入bean，而是直接new出来的Java对象，为了能够让new出来的Java对象可以实现**自动注入**并**遵循Bean的生命周期**，Spring Security中提供了如下方案，主要围绕objectPostProcessor展开

#### #1.1.1 objectPostProcessor

这个接口的定义如下

```java
/**
 * Allows initialization of Objects. Typically this is used to call the {@link Aware}
 * methods, {@link InitializingBean#afterPropertiesSet()}, and ensure that
 * {@link DisposableBean#destroy()} has been invoked.
 *
 * @param <T> the bound of the types of Objects this {@link ObjectPostProcessor} supports.
 *
 * @author Rob Winch
 * @since 3.2
 */
public interface ObjectPostProcessor<T> {

   /**
    * Initialize the object possibly returning a modified instance that should be used
    * instead.
    *
    * @param object the object to initialize
    * @return the initialized version of the object
    */
   <O extends T> O postProcess(O object);
}
```

注释中提到"postProcess方法主要用于初始化对象，并可能返回一个**修改过的对象**"。这里指的**修改过的对象**其实就是提供了Spring支持的Bean对象

此接口有两个重要的实现类

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201016141536664.png" alt="image-20201016141536664" style="zoom:67%;" />

下面分别讲解这两个类

##### 1）AutowireBeanFactoryObjectPostProcessor

这里主要关注重写的postProcess方法

```java
public <T> T postProcess(T object) {
   if (object == null) {
      return null;
   }
   T result = null;
   try {
      result = (T) this.autowireBeanFactory.initializeBean(object,
            object.toString());
   }
   catch (RuntimeException e) {
      Class<?> type = object.getClass();
      throw new RuntimeException(
            "Could not postProcess " + object + " of type " + type, e);
   }
   this.autowireBeanFactory.autowireBean(object);
   if (result instanceof DisposableBean) {
      this.disposableBeans.add((DisposableBean) result);
   }
   if (result instanceof SmartInitializingSingleton) {
      this.smartSingletons.add((SmartInitializingSingleton) result);
   }
   return result;
}
```

此方法是调用AutowireCapableBeanFactory的多个方法，将object对象初始化为T类型的Bean对象，将此对象进行初始化（initializeBean）、自动注入（autowireBean）、设定Bean生命周期。

对于SS中的每一个new出来的raw bean，都会调用这个方法实现**改动**（初始化、注入、生命周期），**这样**，**就完成了new出现的Java对象也可以支持Spring中Bean对象功能的要求了。**

**那么何时进行调用呢？**见下面的类CompositeObjectPostProcessor

>**由于开启Spring Security的支持需要添加注解@EnableWebSecurity，此注解会将AutowireBeanFactoryObjectPostProcessor注入到IoC容器中**

##### 2）CompositeObjectPostProcessor

介绍此类前，先介绍SS中的每个组件（各类Provider、Filter、Interceptor）是如何被创建并配置的

- 调用对应的**XXXConfigurer**中的configure方法，方法执行中如下
  - 新建要配置的对象
  - 配置相关属性
  - 执行postProcess方法
- 由于**XXXConfigurer**均继承于**SecurityConfigurerAdapter**，在执行postProcess方法时，都是执行的**SecurityConfigurerAdapter**中的postProcess方法，执行时是调用**CompositeObjectPostProcessor**中的postProcess方法

那么我们的关注点就在此类中的postProcess方法了，方法体如下

```java
/**
 * An {@link ObjectPostProcessor} that delegates work to numerous
 * {@link ObjectPostProcessor} implementations.
 *
 * @author Rob Winch
 */
private static final class CompositeObjectPostProcessor implements
      ObjectPostProcessor<Object> {
   private List<ObjectPostProcessor<?>> postProcessors = new ArrayList<>();

   @SuppressWarnings({ "rawtypes", "unchecked" })
   public Object postProcess(Object object) {
      for (ObjectPostProcessor opp : postProcessors) {
         Class<?> oppClass = opp.getClass();
         Class<?> oppType = GenericTypeResolver.resolveTypeArgument(oppClass,
               ObjectPostProcessor.class);
         if (oppType == null || oppType.isAssignableFrom(object.getClass())) {
            object = opp.postProcess(object);
         }
      }
      return object;
   }

   /**
    * Adds an {@link ObjectPostProcessor} to use
    * @param objectPostProcessor the {@link ObjectPostProcessor} to add
    * @return true if the {@link ObjectPostProcessor} was added, else false
    */
   private boolean addObjectPostProcessor(
         ObjectPostProcessor<?> objectPostProcessor) {
      boolean result = this.postProcessors.add(objectPostProcessor);
      postProcessors.sort(AnnotationAwareOrderComparator.INSTANCE);
      return result;
   }
}
```

下面关注**一个属性和两个方法**

###### 1）属性

postProcessors这个属性包含了许多ObjectPostProcess的实现类对象

###### 2）方法

:one: postProcess

会根据当前对象持有的postProcessors，遍历其中的每一个postProcessor，先检查是否可以对当前对象调用，之后调用postProcess方法

:two: addObjectPostProcessor

用于为当前的对象添加更多的postProcessor

**通过上面，可以看出SS是借助此类代理完成了多个postProcess方法，而每个组件配置时postProcessors中至少会包含AutowireBeanFactoryObjectPostProcessor的对象，这样才能保证刚刚new出来的对象能够被自动注入等。**

### #1.1.2 Insights

从上面的SS的组件配置中可以发现，**有关于SS相关组件的创建和配置都是围绕着postProcess方法展开的**，这也是我们可以SS组件属性的入手点，我们要做的就是

- 想办法获取到创建对应组件的XXXConfigurer，并通过addObjectPostProcessor方法添加自定义的postProcessor
- 在自定义的postProcess方法中定义好如何对新建的组件的属性进行修改。

下面就开始对**FilterSecurityInterceptor**这个用于认证和鉴权的组件属性进行定制

### #1.2 FilterSecurityInterceptor的默认配置

`FilterSecurityInterceptor`通过`AbstractInterceptUrlConfigurer`的configure方法完成创建和配置，方法体如下

```java
public void configure(H http) throws Exception {
   FilterInvocationSecurityMetadataSource metadataSource = createMetadataSource(http);
   if (metadataSource == null) {
      return;
   }
   FilterSecurityInterceptor securityInterceptor = createFilterSecurityInterceptor(
         http, metadataSource, http.getSharedObject(AuthenticationManager.class));
   if (filterSecurityInterceptorOncePerRequest != null) {
      securityInterceptor
            .setObserveOncePerRequest(filterSecurityInterceptorOncePerRequest);
   }
   securityInterceptor = postProcess(securityInterceptor);
   http.addFilter(securityInterceptor);
   http.setSharedObject(FilterSecurityInterceptor.class, securityInterceptor);
}
```

其中先调用createFilterSecurityInterceptor，方法体如下

```java
private FilterSecurityInterceptor createFilterSecurityInterceptor(H http,
      FilterInvocationSecurityMetadataSource metadataSource,
      AuthenticationManager authenticationManager) throws Exception {
   FilterSecurityInterceptor securityInterceptor = new FilterSecurityInterceptor();
   securityInterceptor.setSecurityMetadataSource(metadataSource);
   securityInterceptor.setAccessDecisionManager(getAccessDecisionManager(http));
   securityInterceptor.setAuthenticationManager(authenticationManager);
   securityInterceptor.afterPropertiesSet();
   return securityInterceptor;
}
```

上面方法中设定了三个重要的属性

:one: `FilterInvocationSecurityMetadataSource`决定了如何查询FilterInvocation中的属性（getAttributes方法）

:two: `AccessDecisionManager`决定了如何授权包括鉴权者（Voter）和鉴权逻辑（decide方法）

:three: `AuthenticationManager`决定了如何进行认证（authenticate方法，其中可以指定如何获取用户权限信息（retrieveUser方法）以及使用的密码编码器（PasswordEncoder））

之后执行postProcess方法，是调用`SecurityConfigurerAdapter.CompositeObjectPostProcessor`的postProcess方法，此类也是代理给多个`ObjectPostProcessor`的实现类去完成postProcess方法

当执行postProcess时，postProcessors中必定会出现一个`AutowireBeanFactoryObjectPostProcessor`，这个类中重写的postProcess方法如下

```java
public <T> T postProcess(T object) {
   if (object == null) {
      return null;
   }
   T result = null;
   try {
      result = (T) this.autowireBeanFactory.initializeBean(object,
            object.toString());
   }
   catch (RuntimeException e) {
      Class<?> type = object.getClass();
      throw new RuntimeException(
            "Could not postProcess " + object + " of type " + type, e);
   }
   this.autowireBeanFactory.autowireBean(object);
   if (result instanceof DisposableBean) {
      this.disposableBeans.add((DisposableBean) result);
   }
   if (result instanceof SmartInitializingSingleton) {
      this.smartSingletons.add((SmartInitializingSingleton) result);
   }
   return result;
}
```

这个方法会将传入的对象通过手动注入的方法注册为bean并放入IoC容器中。这个postProcessor对Spring Security中的每一个new出现的Java对象都有效。

上面部分即为FilterSecurityInterceptor的默认配置

### #1.3 修改默认配置

#### #1.3.1 入手点

**由上面的部分可以看出，对FilterSecurityInterceptor定制可以从下面几点入手**，

:one: setSecurityMetadataSource

可以调用此方法设定自定义的类，自定义类必须实现FilterInvocationSecurityMetadataSource，可以重写其中的getAttributes方法

:two: setAccessDecisionManager

可以调用此方法设定自定义的类，自定义类必须实现AccessDecisionManager，可以重写其中的decide方法（影响鉴权逻辑）和两个supports方法（影响鉴权Voter）

**那么如何才能调用这两个set方法实现覆盖呢？**

#### #1.3.2 如何调用set方法

##### 1）添加ObjectPostProcessor的自定义实现类

从上面我们可以看出，可以通过添加postProcess实现类的方式来对代码逻辑进行修改，要注意，由于每一个被new出来的对象至少有一个postProcessor是`AutowireBeanFactoryObjectPostProcessor`，我们自定义的会被按顺序放入`List<ObjectPostProcessor<?>> postProcessors`中，所以，**一定是先注入到IoC容器中，之后再对其中的Bean做修改**，那么自定义的实现类中的postProcess方法中就可以调用上面两个set方法实现自定义

##### 2）调用addObjectPostProcessor方法

获取这个类的目的是为了调用其内部类中的add方法来添加postProcessor。

又由于Spring Security中使用的是`WebSecurityConfigurerAdapter`来完成登陆的认证和鉴权的相关配置，此类中的一个configure的重载方法如下所示

```java
protected void configure(HttpSecurity http) throws Exception {
   logger.debug("Using default configure(HttpSecurity). If subclassed this will potentially override subclass configure(HttpSecurity).");

   http
      .authorizeRequests()
         .anyRequest().authenticated()
         .and()
      .formLogin().and()
      .httpBasic();
}
```

**注意到**，此方法中通过`http.authorizeRequests()`获得了`ExpressionInterceptUrlRegistry`对象，此类中定义了withObjectPostProcessor方法，如下

```java
public ExpressionInterceptUrlRegistry withObjectPostProcessor(
      ObjectPostProcessor<?> objectPostProcessor) {
   addObjectPostProcessor(objectPostProcessor);
   return this;
}
```

此方法中的addObjectPostProcessor方法是其外部类`ExpressionUrlAuthorizationConfigurer`继承于`SecurityConfigurerAdapter	`的，

**最终是调用`CompositeObjectPostProcessor`中的addObjectPostProcessor方法！！！**（找到了）

**那么**，**可以通过定义`WebSecurityConfigurerAdapter`的子类，并重写其中的configure方法，并在其中调用withObjectPostProcessor即可实现添加自定义postProcessor**，在其中的postProcess方法中可以调用前面提到的set方法来实现定制。

##### 3）定制postProcess方法

可以直接定义一个WebSecurityConfigurerAdapter的子类，重写上面提到的configure方法，并添加postProcessor即可

```java
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
            .withObjectPostProcessor(new ObjectPostProcessor<FilterSecurityInterceptor>() {
                @Override
                public <O extends FilterSecurityInterceptor> O postProcess(O object) {
                    //设定鉴权时使用的角色管理者
                    object.setAccessDecisionManager(customUrlDecisionManager);
                    //设定鉴权时使用下面类中的getAttributes方法来查询FilterInvocation中的信息
                object.setSecurityMetadataSource(customFilterInvocationSecurityMetadataSource);
                    return object;
                }
            })
            .and()
            ......
}
```

### 至此，定制FilterSecurityInterceptor就完成了！

