## #6.SpringBoot-缓存

> 需要引入下面依赖以及要使用的Nosql的依赖开启Springboot对缓存的支持
>
> ```xml
> <dependency>
>     <groupId>org.springframework.boot</groupId>
>     <artifactId>spring-boot-starter-cache</artifactId>
> </dependency>
> ```

### #6.1 JSR-107规范

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201031125514123152.png" alt="image-20201031125514152" style="zoom:67%;" />

#### #6.1.1CachingProvider

`CachingProvider`用于创建和管理`CachingManager`的生命周期

#### #6.1.2CachingManager

`CachingManager`用于创建和配置`Cache`

#### #6.1.3Cache

`Cache`中存储了多对键值对，每对键值对都有一个expiry属性用于标记这个键值对的过期时间

### #6.2 Springboot中的缓存

> springboot对JSR-107做了简化，仅保留了manage和cache两个级别

#### #6.2.1 自动缓存配置

:one: 不使用Nosql

##### #1）spring-boot-autoconfigure-2.3.5.RELEASE.jar!\META-INF\spring.factories

在`org.springframework.boot.autoconfigure.EnableAutoConfiguration`包含的条目中，有`org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration`条目，此类中会从备选的缓存配置中选出合适的，备选的有如下几项

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201031145718502.png" alt="image-20201031145718502" style="zoom: 80%;" />

当没有引入EhCache等Nosql时，会使用`SimpleCacheConfiguration`

##### #2）SimpleCacheConfiguration

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingBean(CacheManager.class)
@Conditional(CacheCondition.class)
class SimpleCacheConfiguration {

   @Bean
   ConcurrentMapCacheManager cacheManager(CacheProperties cacheProperties,
         CacheManagerCustomizers cacheManagerCustomizers) {
      ConcurrentMapCacheManager cacheManager = new ConcurrentMapCacheManager();
      List<String> cacheNames = cacheProperties.getCacheNames();
      if (!cacheNames.isEmpty()) {
         cacheManager.setCacheNames(cacheNames);
      }
      return cacheManagerCustomizers.customize(cacheManager);
   }

}
```

###### #（1）生效条件

此类会在没有配置CacheManager对象且配置了CacheCondition对象时才会配置

###### #（2）组件配置

此类会配置一个`ConcurrentMapCacheManager`对象，如果在yml中指定了cacheNames属性，还会注册一个`Cache`对象，令其名称为指定的cacheName

:two: 使用Nosql

> 此处引入Redis

如果引入了`spring-boot-starter-data-redis`，会调用redis的自动配置类

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {

   @Bean
   @ConditionalOnMissingBean(name = "redisTemplate")
   public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory)
         throws UnknownHostException {
      RedisTemplate<Object, Object> template = new RedisTemplate<>();
      template.setConnectionFactory(redisConnectionFactory);
      return template;
   }

   @Bean
   @ConditionalOnMissingBean
   public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory)
         throws UnknownHostException {
      StringRedisTemplate template = new StringRedisTemplate();
      template.setConnectionFactory(redisConnectionFactory);
      return template;
   }

}
```

###### #（1）生效条件

此类会在配置了RedisOperations组件时才会配置

###### #（2）组件配置

- RedisTemplate

  在没有配置redisTemplate时，会创建一个redisTemplate组件，那么也就意味着如果在配置类中自定义了redisTemplate对象就不会再创建这个组件

  > 在自定义时，要注意设定redisConnectionFactory

- StringRedisTemplate

  与RedisTemplate逻辑相同

由于引入了RedisAutoConfiguration，所以会调用`RedisCacheConfiguration`配置类进行配置

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisConnectionFactory.class)
@AutoConfigureAfter(RedisAutoConfiguration.class)
@ConditionalOnBean(RedisConnectionFactory.class)
@ConditionalOnMissingBean(CacheManager.class)
@Conditional(CacheCondition.class)
class RedisCacheConfiguration {

   @Bean
   RedisCacheManager cacheManager(CacheProperties cacheProperties, CacheManagerCustomizers cacheManagerCustomizers,
         ObjectProvider<org.springframework.data.redis.cache.RedisCacheConfiguration> redisCacheConfiguration,
         ObjectProvider<RedisCacheManagerBuilderCustomizer> redisCacheManagerBuilderCustomizers,
         RedisConnectionFactory redisConnectionFactory, ResourceLoader resourceLoader) {
      RedisCacheManagerBuilder builder = RedisCacheManager.builder(redisConnectionFactory).cacheDefaults(
            determineConfiguration(cacheProperties, redisCacheConfiguration, resourceLoader.getClassLoader()));
      List<String> cacheNames = cacheProperties.getCacheNames();
      if (!cacheNames.isEmpty()) {
         builder.initialCacheNames(new LinkedHashSet<>(cacheNames));
      }
      redisCacheManagerBuilderCustomizers.orderedStream().forEach((customizer) -> customizer.customize(builder));
      return cacheManagerCustomizers.customize(builder.build());
   }

   private org.springframework.data.redis.cache.RedisCacheConfiguration determineConfiguration(
         CacheProperties cacheProperties,
         ObjectProvider<org.springframework.data.redis.cache.RedisCacheConfiguration> redisCacheConfiguration,
         ClassLoader classLoader) {
      return redisCacheConfiguration.getIfAvailable(() -> createConfiguration(cacheProperties, classLoader));
   }

   private org.springframework.data.redis.cache.RedisCacheConfiguration createConfiguration(
         CacheProperties cacheProperties, ClassLoader classLoader) {
      Redis redisProperties = cacheProperties.getRedis();
      org.springframework.data.redis.cache.RedisCacheConfiguration config = org.springframework.data.redis.cache.RedisCacheConfiguration
            .defaultCacheConfig();
      config = config.serializeValuesWith(
            SerializationPair.fromSerializer(new JdkSerializationRedisSerializer(classLoader)));
      if (redisProperties.getTimeToLive() != null) {
         config = config.entryTtl(redisProperties.getTimeToLive());
      }
      if (redisProperties.getKeyPrefix() != null) {
         config = config.prefixCacheNameWith(redisProperties.getKeyPrefix());
      }
      if (!redisProperties.isCacheNullValues()) {
         config = config.disableCachingNullValues();
      }
      if (!redisProperties.isUseKeyPrefix()) {
         config = config.disableKeyPrefix();
      }
      return config;
   }

}
```

###### #（1）生效条件

此类会在引入了RedisConnectionFactory类、RedisConnectionFactory组件、没有配置CacheManager时，会在RedisAutoConfiguration自动配置之后进行配置

###### #（2）组件配置

- RedisCacheManager

  类似于ConcurrentCacheManager，用于管理redis中的缓存

Redis中的不同的`Cache`对象会以不同的前缀开始，前缀也即该对象配置的名称，其中存放的所有的key都会带有Cache对象的名称作为前缀

#### #6.2.2 缓存注解

##### #1）@EnableCache

标注在main方法的类上，开启Springboot对缓存的支持

##### #2）@Cacheable

###### #（1）功能

标记在方法上，在方法执行前从缓存中查找是否有想要的结果，如果有则直接从缓存中返回；否则执行方法并将返回值放入缓存中

###### #（2）属性

通用属性

##### #3）@CacheEvict

###### #（1）功能

清空缓存

###### #（2）属性

除了通用属性，还可以指定`allEntries`来表示清空整个缓存，默认为false，此时只清除键（前缀）相关的缓存，在使用此属性时，`key`属性无效

##### #4）@CachePut

###### #（1）功能

每次在方法调用之后将结果写入缓存

###### #（2）属性

通用属性

上面几个注解的**通用属性**有

- `cacheNames`用于指明要存入的缓存的名称，反映在redis中也即key使用的前缀名称
- `key`用于给定要存入的数据的key的值，一般使用el表达式从方法参数中获得
- `keyGenerator`用于指定key的生成器，在指定生成器时就不需要指定key
- `condition`表示在满足某些条件时放入缓存
- `unless`表示在不满足某些条件时放入缓存
- `sync`表示是否异步的放入缓存，如果开启则不能使用`unless`属性

##### #5）@Caching

其中可以指定多个@Cacheable、@CacheEvict和@CachePut注解规则

### #6.3vhr中相关使用

#### #PRE - 自定义

##### #1）自定义keyGenerator

> 默认使用SimpleKeyGenerator

```java
@Configuration
public class KeyGeneratorConfig {
    @Bean
    public KeyGenerator keyGenerator() {
        return (target, method, params) -> method.getName() + "[" + Arrays.toString(params) + "]";
    }
}
```

##### #2）自定义序列化

```java
@Configuration
public class RedisTemplateConfig {
    @Autowired
    RedisConnectionFactory redisConnectionFactory;

    @Bean
    public RedisTemplate<Object, Object> redisTemplate() {
        RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
        GenericJackson2JsonRedisSerializer serializer = new GenericJackson2JsonRedisSerializer();
        redisTemplate.setDefaultSerializer(serializer);
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        return redisTemplate;
    }
}
```

这里通过自定义RedisTemplate对象让每个RedisCacheManager都使用这个对象和redis存取数据，由于默认使用JDK的序列化策略，这里指定为JackSon的序列化策略，将对象序列化为JSON字符串

#### #6.3.1 菜单缓存

##### Service方法

```java
@Cacheable(keyGenerator = "keyGenerator",cacheNames = "menus")
//    Default is "", meaning all method parameters are considered as a key,
//    unless a custom keyGenerator has been configured.
public List<Menu> getAllMenusWithRole(){
    System.out.println("向redis中放入菜单缓存");
    return menuMapper.getAllMenusWithRole();
}
```

#### #6.3.2 员工表缓存

```java
@Cacheable(cacheNames = "emps", key = "'page:'+#page+'-'+#size+'items'")
public RespPageBean getEmployeeByPage(Integer page, Integer size, Employee employee, Date[] beginDateScope) {
//        System.out.println("从mysql中查询并放入redis");
    //mysql中起始索引从0开始
    //这两个参数的作用相当于limit page（offset） size（size）
    if (page != null && size != null) {
        page = (page - 1) * size;
    }
    List<Employee> data = employeeMapper.getEmployeeByPage(page, size, employee, beginDateScope);
    Long total = employeeMapper.getTotal(employee, beginDateScope);
    RespPageBean bean = new RespPageBean();
    bean.setData(data);
    bean.setTotal(total);
    return bean;
}
```

此处需要注意给定key时需要以页号为key来存储，因为每次请求来的数据都只是一个页面的信息