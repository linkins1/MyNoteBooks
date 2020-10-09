## 7.Jedis

### 7.1导入jar包

```xml
<!-- https://mvnrepository.com/artifact/redis.clients/jedis -->
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>3.3.0</version>
</dependency>
```

### 7.2使用

首先需要创建Jedis对象

```java
Jedis jedis = new Jedis("IP",PORT);
```

在创建出jedis对象后，通过jedis进行操作即可