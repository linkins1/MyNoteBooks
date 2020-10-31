## 6.邮件发送

### 6.1 邮件发送原理

#### 6.1.1 流程图

##### 1）浏览器发送邮件

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201026133902390.png" alt="image-20201026133902390" style="zoom:67%;" />

##### 2）javax.mail发送邮件

<img src="C:%5CUsers%5C123%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20201026133940010.png" alt="image-20201026133940010" style="zoom: 67%;" />

#### 6.1.2 SMTP（**Simple Mail Transfer Protocol**）

> 邮件发送协议

##### 1）定义

SMTP协议是一种用于发送邮件的协议，其具有如下特征

- 明文传输

  指不会对传输的内容进行加密

- 适用于传输ASCII码

  不适用于二进制传输

- TCP连接

  SMTP服务器之间会建立TCP连接

##### 2）执行流程

###### （1）连接建立

在发送端SMTP服务器收到用户要发送的邮件后，开始如下流程

- SMTP服务器会通过**25号端口**向接收方的SMTP服务器发出TCP连接请求
- 如果连接建立成功，接收方发送"220 Service ready"来应答
- 这时发送用户向发送端SMTP服务器发送**"HELO+用户的主机名"**请求
- 如果发送端SMTP服务器可以接收这次请求，则返回"250 OK"，否则返回"421 Service not available"

> 这里需要注意，建立TCP连接时**直接发生在发送端SMTP服务器和接收端SMTP上的，不经过任何中间服务器**

###### （2）传送邮件

- 发送用户传送**"MAIL FROM + 发送者邮箱号"**至发送端SMTP服务器
- 如果发送端SMTP服务器接收请求，那么返回**"250 Mail OK"**
- 发送端用户发送**"RCPT TO : 接收者邮箱号"**至接收端SMTP服务器
- 如果接收端SMTP服务器判断执行邮箱符合范围，则返回**"250 Mail OK"**，否则返回"550 No Such User Here"
- 接着发送用户发送**"DATA"**命令至发送端SMTP服务器，表示要开始发送邮件
- 如果发送端SMTP服务器可以发送，则返回**"354 End data with <CR><LF>.<CR><LF>"**，代表以`<CR><LF>.<CR><LF>`作为发送结束的标志
- 如果发送成功，则返回发送用户**"message successfully delivered to mail server"**

> 这里需要注意，一般发送用户使用浏览器发送邮件至发送端SMTP服务器的数据传输是使用HTTP协议，同样需要建立TCP连接

###### （3）连接释放

- 发送用户发送**"QUIT"**至发送端SMTP服务器
- 如果发送端SMTP服务器同意释放TCP连接，会返回**"221 Bye"**

##### 3）改进

###### （1）安全性

由于SMTP协议使用的是透明传输，**不会对传输的内容进行加密**且**没有身份认证**。所以提出SMTPS（SMTP over SSL）来保证安全的通信，这里**只是加入**身份校验环节，使用非对称加密，使用公私钥加密来保证数据传输的安全性，**但是**，并没有对数据本身进行加密

> TLS是SSL的升级版，目前常说TLS
>
> SMTPS和ESMTP的初衷一致，目前常说SMTPS

在采用加密后，由SMTPS发出的请求（实际是ESMTP规定的）为**"EHLO"**，且增加了**"AUTH"**用于校验数据

###### （2）数据格式

由于SMTP只被设计用来传输7位ASCII码，为了能够支持二进制等不同类型数据的传输，提出了**MIME(Multipurpose Internet Mail Extensions)**，增强了对传输内容的扩展，通过指定MIME中的内容类型和编码方式，即可将多种数据类型的数据编码为7位ASCII码，使得SMTP可以传输，这样就增强了SMTP对传输内容类型的支持，转换关系如下图所示

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201026125352128.png" alt="image-20201026125352128" style="zoom:67%;" />

#### 6.1.3 POP3与IMAP

> 邮件接收协议

两种协议都实现了接收用户通过POP3或IMAP客户端程序从接收方POP3或IMAP服务器上拉取邮件。

两者主要区别如下

| 功能                     | IMAP                                 | POP3                       |
| ------------------------ | ------------------------------------ | -------------------------- |
| 一封邮件被多处且多次访取 | 支持                                 | 不支持                     |
| 客户端和服务器同步       | 支持                                 | 不支持                     |
| 离线访问                 | 如果没有保存，则必须在线从服务器访问 | 自动离线保存，但只保存一次 |

### 6.2 rabbitmq与redis

> 引入rabbitmq可以将邮件发送服务和人事网页服务分隔开
>
> redis可以避免消息重复消费

#### 6.2.1 rabbitmq

> vhr中构造了一个任务队列，vhr_service向队列中添加消息，mail_server订阅队列的消息并消费且会进行手动确认

##### 1）消息确认机制

###### （1）消费者回传ack（[Delivery Acknowledgements](https://www.rabbitmq.com/confirms.html#consumer-acknowledgements)）

> 为了保证队列内的消息成功的被消费者消费

:one: delivery tag（交付标签）

当一个消费者被注册后，broker通过`basicDeliver`方法发送消息，此方法会携带一个**delivery tag**发送至消费者

delivery tag有如下特点

- 信道独立性

  每个信道中的delivery tag是独立的，对收到的消息进行ack时**必须保证**与收到的消息信道相同

  如果不在相同信道，会抛出"unknown delivery tag"协议异常

- 单调递增性

  每次增一

:two: 确认机制

- 自动确认（"fire-and-forget"）

  rabbitmq在消息发送出去之后就设定为ack

  换取了更大的吞吐量，可能导致消费者的消息积压

- 手动确认

  通过下面三个api来实现

  | api         | 含义                                                         |
  | ----------- | ------------------------------------------------------------ |
  | basicAck    | 回传ack，如果设定其中的`mutipul(boolean)`参数为true，那么可以一次确认多个消息，假设发送了5，6，7，8；只ack了8，那么会将前面几个一并确认 |
  | basicNack   | 回传nack，同样可以设定`mutipul(boolean)`，此外还可以设定`requeue(boolean)`为true让消息重新入队，在非并发条件下会回到原来位置，在并发条件下，会回到尽可能靠前的位置 |
  | basicReject | 回传nack                                                     |

  - **prefetch count**

    在设定为手动确认后，消费者需要考虑每次可以从rabbitmq获得的异步消息数应该为多少（QoS问题），如果发给消费者的消息太快太多，消费者会来不及ack，所以需要设定一个`预取值prefetch count`（通过`basicQoS`设定），来规定每个信道上允许的最多的没有被ack的消息数（也即消息发送窗口的大小），这样rabbitmq在发送消息时，如果窗已满，那么就不再发送。

    使用发送窗的情形可以考虑开启`mutipul(boolean)`，这样可以高效的清空发送窗

  - **消费幂等性**

    由于可能存在消费者消费后返回的ack因为网络问题没有送达rabbitmq，那么此时rabbitmq会将这些消息（rabbitmq中消息不会超时）重新入队，并设定`redeliver=true`，这样消费者就必须检查是否为重复消息来保证幂等性

    >***Q1***：**prefetch count可以修改吗**
    >
    >***A1***：在消息发送过程中，不可以修改`预取值prefetch count`，如果突然调小窗口，可能会导致"在空中"的消息不能被ack
    >
    >***Q2***：**prefetch count值一般取多大？**
    >
    >***A2***：为1时会导致rabbitmq阻塞时间过长，吞吐量下降，一般可取200-300
    >
    >***Q3***：**既然有redeliver标签，vhr为何还要使用redis来存储已消费的消息？**
    >
    >***A3***：因为可能存在rabbitmq发来的消息在网络中丢失，此时，rabbitmq因为没有收到ack会重传消息，这样对于消费者来说这条**"重传的"**的消息实际上从没被收到过，这是就需要redis来**辅助查询**，保证幂等性

  - **重复ack或错误ack**

    - 重复ack

      如果消费者对同一个delivery-tag重复ack，此时rabbitmq会抛出`PRECONDITION_FAILED - unknown delivery tag xxx`的异常

    - 错误ack

      如果ack到其他信道上，此时会导致其他信道上的状态混乱，会抛出`"unknown delivery tag"`

###### （2）发布者被确认

> 为了保证发出的消息成功入队

:one:**使信道事务化**

即让信道中的每一条消息都具备事务特性（发布、提交、回滚），这样会导致信道传送效率极差

:two: **消费者确认机制**

可以采用和消费端类似的思想，对每一条发布到rabbitmq的消息都做ack或nack，在确认机制下，不能使信道事务化

**publisher confirms**要求发布者和broker同时从1开始对消息进行编号（sn号），broker返回的ack消息中通过delivery-tag来传送sn号。broker也可以设定multipul属性为true

:three: **broker发送ack、nack和return**

- ack（`broker->exchange->queue`）

  在消息被成功的放入队列时，会返回一个ack，发布者将调用confirmcallback

- nack（`broker-❌>exchange`）

  在broker不能成功处理发来的消息时，会返回一个nack，发布者将调用confirmcallback

- return（`broker->exchange-❌->queue`）

  在消息不能被路由时，

  - 如果`mandatory=true`，那么会返回return，发布者将调用returncallback
  - 否则，返回空队列

:four: **持久化消息**

对于消息和信道都设定为`durable`时，消息会被持久化到磁盘上，这时，**ack会晚于持久化发出**，这样会导致确认的延迟，且会影响到发送者收到的确认的顺序，**根据返回的sn号作为是否重新发布的标准是不稳妥的！**

**更稳健**的方案如下

- 发布者创建唯一的`uuid`赋给CorrelationData
- CorrelationData会随ack一并返回，这时发布者取出其中的`uuid`来判断是否发布过此消息。

> ***Q***：**有了发布者确认机制是否可以保证所以消息都可以被投递到队列中？**
>
> ***A***：**不能！**如果要对`消息a`发送的确认是nack，但是由于nack要晚于持久化，如果持久化失败（宕机或重启），重启后且与发布者重新建立连接后，由于磁盘中不存在`消息a`，所以不会返回给发布者nack，那么发布者也不会重传`消息a`，就永远都消失了

#### 6.2.2 redis

##### 1）存储内容

redis中用于存储在消费者对消息进行过手动确认后，消息相关的id，以hash的形式存入，value为javaboy

##### 2）作用（消费幂等性）

这样做是为了保证队列中的消息**不会被重复消费**，由于消费者传给队列的ack有可能会丢失，但是rabbitmq中的消息只要没有收到确认就会一直保留在队列中。**所以**，在每次收到新的消息时都要从redis中查看是否存在对应的id，如果存在就向队列手动进行ack，让其将此消息从队列中删除；否则，就是第一次处理该消息。

> 幂等性即执行一次操作和执行多次操作的效果是相同的

### 6.3 邮件发送实现

#### 6.3.1 总体流程

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201027171236323.png" alt="image-20201027171236323" style="zoom:150%;" />

- 首先将emp调用convertAndSend发送出去，并将此emp对应的MailSendLog存入mysql
- 交给交换机后，交换机将其放入指定的队列
- mail_server从消息中取出msgId（emp的唯一标记，用于判断重复消费）和tag（信道中发送的消息的唯一标记，用于ack）
  - 如果msgId在redis中不存在
    - 将emp中的内容渲染为html并存为邮件内容
    - 对tag进行ack
    - 将此msgId以键值对的形式存入redis
  - 如果存在
    - 对tag进行nack

##### 1）配置

###### （1）rabbitmq相关配置类

:one: vhr_web

```Java
@Configuration
public class RabbitConfig {
    public final static Logger logger = LoggerFactory.getLogger(RabbitConfig.class);
    @Autowired
    CachingConnectionFactory cachingConnectionFactory;
    @Autowired
    MailSendLogService mailSendLogService;
    /**
    省略异常处理
    */
    @Bean
    Queue mailQueue() {
        return new Queue(MailConstants.MAIL_QUEUE_NAME, true);
    }

    @Bean
    DirectExchange mailExchange() {
        return new DirectExchange(MailConstants.MAIL_EXCHANGE_NAME, true, false);
    }

    @Bean
    Binding mailBinding() {
        return BindingBuilder.bind(mailQueue()).to(mailExchange()).with(MailConstants.MAIL_ROUTING_KEY_NAME);
    }

}
```

此处定义了交换机、队列，并将队列名称、交换机、routingKey绑定在一起

:two: mail_server

```java
@Bean
Queue queue() {
    return new Queue(MailConstants.MAIL_QUEUE_NAME);
}
```

要声明一个队列，队列名称和发送端相同，以避免接收端先启动时监听的队列为空（也即只会创建一个同名的队列）

##### 2）yml

:one: vhr_web

要配置其中的redis和rabbitmq的端口号和ip，其中rabbitmq端口必须为5672

:two: mail_server

```properties
server.port=8082
# SMTP 服务器 host  163邮箱的为 smtp.163.com 端口 465
spring.mail.host=smtp.163.com
spring.mail.protocol=smtps
spring.mail.default-encoding=UTF-8
# 发送端的用户邮箱密码
spring.mail.password=VREPZKPROFFRLYIT
# 发送端的用户邮箱名
spring.mail.username=****@163.com
spring.mail.port=587
spring.mail.properties.mail.stmp.socketFactory.class=javax.net.ssl.SSLSocketFactory
spring.mail.properties.mail.debug=true
spring.rabbitmq.username=root
spring.rabbitmq.password=****
spring.rabbitmq.host=****
##端口必须正确
spring.rabbitmq.port=5672
spring.rabbitmq.listener.direct.acknowledge-mode=manual
spring.rabbitmq.listener.simple.prefetch=100
spring.redis.host=****
spring.redis.port=6379
spring.redis.password=****
spring.redis.database=0
```

其中rabbitmq的端口必须为5672

且根据邮件发送原理，此处协议需要为smtps（开启SSL），对应的端口需要时587

> 465虽然被使用，但是不推荐（官方不认可）

##### 2）生产者/消费者api

###### （1）vhr_web（发送端）

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
        //将此次发送邮件的信息存储在mysql中
        mailSendLogService.insert(mailSendLog);
        //向消息队列发送消息Message，需要将emp对象转换为Message对象，Message对象的body是employee对象的字节流
        rabbitTemplate.convertAndSend(MailConstants.MAIL_EXCHANGE_NAME, MailConstants.MAIL_ROUTING_KEY_NAME, emp, new CorrelationData(msgId));
    }
    return result;
}
```

###### （2）mail_server（接收端）

```java
@Component
public class MailReceiver {

    public static final Logger logger = LoggerFactory.getLogger(MailReceiver.class);

    @Autowired
    JavaMailSender javaMailSender;
    @Autowired
    MailProperties mailProperties;
    @Autowired
    TemplateEngine templateEngine;
    @Autowired
    StringRedisTemplate redisTemplate;

    @RabbitListener(queues = MailConstants.MAIL_QUEUE_NAME)
    public void handler(Message message, Channel channel) throws IOException {
//        channel.basicQos(3);max number of unacknowledged deliveries that are permitted on a channel
        Employee employee = (Employee) message.getPayload();

        MessageHeaders headers = message.getHeaders();
        //判断是否为重传的消息，如果是则检查redis中查询是否有此消息，保证消费幂等性
        boolean redeliver_flag = (Boolean) headers.get("amqp_redelivered");
        //服务端发来的消息的标志
        Long tag = (Long) headers.get(AmqpHeaders.DELIVERY_TAG);
        //生产者放入消息中的uuid
        String msgId = (String) headers.get("spring_returned_message_correlation");
        if(redeliver_flag){
            if (redisTemplate.opsForHash().entries("mail_log").containsKey(msgId)) {
                //redis 中包含该 key，说明该消息已经被消费过
                logger.info(msgId + ":消息已经被消费");
                channel.basicAck(tag, false);//确认消息已消费
                return;
            }
        }
        //收到消息，发送邮件，helper用于辅助构造msg
        MimeMessage msg = javaMailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(msg);
        try {
            helper.setTo(employee.getEmail());
            helper.setFrom(mailProperties.getUsername());
            helper.setSubject("入职欢迎");
            helper.setSentDate(new Date());
            Context context = new Context();
            context.setVariable("name", employee.getName());
            context.setVariable("posName", employee.getPosition().getName());
            context.setVariable("joblevelName", employee.getJobLevel().getName());
            context.setVariable("departmentName", employee.getDepartment().getName());
            String mail = templateEngine.process("mail", context);
            helper.setText(mail, true);
            javaMailSender.send(msg);
            redisTemplate.opsForHash().put("mail_log", msgId, "javaboy");
            channel.basicAck(tag, false);
            logger.info(msgId + ":邮件发送成功");
        } catch (MailException | MessagingException e) {
            if (e.getMessage().contains("Invalid Address")) {
                channel.basicAck(tag, false);
                logger.error("邮件发送失败：地址无效" + e.getMessage());
            } else {
                channel.basicNack(tag, false, true);
                e.printStackTrace();
                logger.error("邮件发送失败：" + e.getMessage());
            }

        }
    }
}
```

通过`@RabbitListener(queues = MailConstants.MAIL_QUEUE_NAME)`注解即可监听对应的队列，并获得需要的参数（message和channel）

#### 6.3.2 **publisher confirms**（发布者可靠投递）

<img src="https://cdn.jsdelivr.net/gh/linkins1/MyNoteBooks/resources/imgs/vhr/image-20201028200848305.png" alt="image-20201028200848305" style="zoom:150%;" />

> **在application.yml中需要设定发布者确认模式**
>
> ```yml
>   rabbitmq:
> #    需要选定对publisher的确认机制 这里使用CorrelationData
>     publisher-confirm-type: correlated
> #   默认是none，还有一个是simple，采用waitForConfirms，这种方式是一直等待确认才继续发布消息
> ```

##### 1）定时检测

```java
@Component
public class MailSendTask {
    @Autowired
    MailSendLogService mailSendLogService;
    @Autowired
    RabbitTemplate rabbitTemplate;
    @Autowired
    EmployeeService employeeService;
    @Scheduled(cron = "0/10 * * * * ?")
    public void mailResendTask() {
        List<MailSendLog> logs = mailSendLogService.getMailSendLogsByStatus();
        if (logs == null || logs.size() == 0) {
            return;
        }
        logs.forEach(mailSendLog->{
            if (mailSendLog.getCount() >= 3) {
                mailSendLogService.updateMailSendLogStatus(mailSendLog.getMsgId(), 2);//直接设置该条消息发送失败
            }else{
                mailSendLogService.updateCount(mailSendLog.getMsgId(), new Date());
                Employee emp = employeeService.getEmployeeById(mailSendLog.getEmpId());
                rabbitTemplate.convertAndSend(MailConstants.MAIL_EXCHANGE_NAME, MailConstants.MAIL_ROUTING_KEY_NAME, emp, new CorrelationData(mailSendLog.getMsgId()));
            }
        });
    }
}
```

###### （1）设定定时

通过`@Scheduled`标签中的cron属性来指定定时执行的策略，cron中共有6个属性，从左到右意义如下

- second
- minute
- hour
- day of month
- month
- day of week

两个例子如下

- "0 * * * * MON-FRI" 代表在工作日的每分钟执行一次
- "0/10 * * * * ?"代表每10秒执行一次

> [在线cron生成](https://cron.qqe2.com/[)

###### （2）任务

- 查询数据库中日期大于系统时间（当前）且status为0（代表还在尝试）的条目
- 判断每个条目的count值
  - 如果count>=3，更新status为2，代表失败
  - 否则，取出其中的empId，从employee表中查询emp条目，再次调用rabbitTemplate重新发送，并将mysql中的对应条目的count++，并更新时间

> 注意要修改mysql的时区，可以通过修改配置文件和设定系统变量来完成
>
> - 配置文件
>
>   ```shell
>   default-time_zone = '+8:00'
>   ```
>
> - 修改全局变量
>
>   ```mysql
>   set global time_zone = '+8:00';  ##修改mysql全局时区为北京时间，即我们所在的东8区
>   set time_zone = '+8:00';  ##修改当前会话时区
>   flush privileges;  #立即生效
>   ```

##### 2）回调函数

```java
@Bean
RabbitTemplate rabbitTemplate() {
    RabbitTemplate rabbitTemplate = new RabbitTemplate(cachingConnectionFactory);
    rabbitTemplate.setConfirmCallback((data, ack, cause) -> {
        String msgId = data.getId();
        if (ack) {
            logger.info(msgId + ":消息发送成功");
            mailSendLogService.updateMailSendLogStatus(msgId, 1);//修改数据库中的记录，消息投递成功
        } else {
            logger.info(msgId + ":消息发送失败");
        }
    });
    rabbitTemplate.setReturnCallback((msg, repCode, repText, exchange, routingkey) -> {
        logger.info("消息发送失败");
    });
    return rabbitTemplate;
}
```

###### （1）ConfirmCallback

```java
rabbitTemplate.setConfirmCallback((data, ack, cause) -> {
    String msgId = data.getId();
    if (ack) {
        logger.info(msgId + ":消息发送成功");
        mailSendLogService.updateMailSendLogStatus(msgId, 1);//修改数据库中的记录，消息投递成功
    } else {
        logger.info(msgId + ":消息发送失败");
    }
});
```

判断返回的ack是否为true

- 如果为true则设定status=1
- 否则记录错误日志

###### （2）ReturnCallback

```java
rabbitTemplate.setReturnCallback((msg, repCode, repText, exchange, routingkey) -> {
    logger.info("消息发送失败");
});
```

如果返回return，当前没做任何处理，由于当前是采用定时检测重传的方式来解决发送端可靠投递，如果采用死信队列，只需要在return中将消息传入其中即可

#### 6.3.3 消费者确认

```java
@Component
public class MailReceiver {

    public static final Logger logger = LoggerFactory.getLogger(MailReceiver.class);

    @Autowired
    JavaMailSender javaMailSender;
    @Autowired
    MailProperties mailProperties;
    @Autowired
    TemplateEngine templateEngine;
    @Autowired
    StringRedisTemplate redisTemplate;

    @RabbitListener(queues = MailConstants.MAIL_QUEUE_NAME)
    public void handler(Message message, Channel channel) throws IOException {
//        channel.basicQos(3);max number of unacknowledged deliveries that are permitted on a channel
        Employee employee = (Employee) message.getPayload();

        MessageHeaders headers = message.getHeaders();
        //判断是否为重传的消息，如果是则检查redis中查询是否有此消息，保证消费幂等性
        boolean redeliver_flag = (Boolean) headers.get("amqp_redelivered");
        //服务端发来的消息的标志
        Long tag = (Long) headers.get(AmqpHeaders.DELIVERY_TAG);
        //生产者放入消息中的uuid
        String msgId = (String) headers.get("spring_returned_message_correlation");
        if(redeliver_flag){
            if (redisTemplate.opsForHash().entries("mail_log").containsKey(msgId)) {
                //redis 中包含该 key，说明该消息已经被消费过
                logger.info(msgId + ":消息已经被消费");
                channel.basicAck(tag, false);//确认消息已消费
                return;
            }
        }
        //收到消息，发送邮件，helper用于辅助构造msg
        MimeMessage msg = javaMailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(msg);
        try {
            helper.setTo(employee.getEmail());
            helper.setFrom(mailProperties.getUsername());
            helper.setSubject("入职欢迎");
            helper.setSentDate(new Date());
            Context context = new Context();
            context.setVariable("name", employee.getName());
            context.setVariable("posName", employee.getPosition().getName());
            context.setVariable("joblevelName", employee.getJobLevel().getName());
            context.setVariable("departmentName", employee.getDepartment().getName());
            String mail = templateEngine.process("mail", context);
            helper.setText(mail, true);
            javaMailSender.send(msg);
            redisTemplate.opsForHash().put("mail_log", msgId, "javaboy");
            channel.basicAck(tag, false);
            logger.info(msgId + ":邮件发送成功");
        } catch (MailException | MessagingException e) {
            if (e.getMessage().contains("Invalid Address")) {
                channel.basicAck(tag, false);
                logger.error("邮件发送失败：地址无效" + e.getMessage());
            } else {
                channel.basicNack(tag, false, true);
                e.printStackTrace();
                logger.error("邮件发送失败：" + e.getMessage());
            }

        }
    }
}
```

由于rabbitmq重传的消息中会添加一个字段`redeliver=true`，rabbitTemplate会将其转为`amqp_redelivered`并放入message中的headers中，**消费者需要在这个字段为true且redis中有对应记录的条件下进行已消费确认**。对于第一次消费的，会被放入redis

这里对于**邮件地址写错**导致的发送失败，异常信息中会包含"Invalid Address"字段，需要对这种消息也进行确认消费并添加`MailException`的catch，否则会一直重传此消息，陷入死循环，对于其他邮件发送错误，将消息重新入队后再次消费

### 6.4 附

##### 1）可靠投递方案

[RabbitMQ生产端消息可靠性投递方案分析](https://cloud.tencent.com/developer/article/1369500)

##### 2）redis中存储的数据

```powershell
123.57.16.192:6379> HGETALL mail_log
 1) "fd74df58-9383-4c03-9755-a1ba212bba14"
 2) "javaboy"
 3) "1a833978-9de9-4529-b81f-bc4164748236"
 4) "javaboy"
```

##### 3）邮件发送流程

```shell
DEBUG: getProvider() returning javax.mail.Provider[TRANSPORT,smtps,com.sun.mail.smtp.SMTPSSLTransport,Oracle]
DEBUG SMTP: useEhlo true, useAuth false
DEBUG SMTP: trying to connect to host "smtp.163.com", port 587, isSSL true
220 163.com Anti-spam GT for Coremail System (163com[20141201])
DEBUG SMTP: connected to host "smtp.163.com", port: 587
EHLO ******************.mshome.net
250-mail
250-PIPELINING
250-AUTH LOGIN PLAIN
250-AUTH=LOGIN PLAIN
250-coremail ******************
250-STARTTLS
250 8BITMIME
DEBUG SMTP: Found extension "PIPELINING", arg ""
DEBUG SMTP: Found extension "AUTH", arg "LOGIN PLAIN"
DEBUG SMTP: Found extension "AUTH=LOGIN", arg "PLAIN"
DEBUG SMTP: Found extension "coremail", arg "******************"
DEBUG SMTP: Found extension "STARTTLS", arg ""
DEBUG SMTP: Found extension "8BITMIME", arg ""
DEBUG SMTP: protocolConnect login, host=smtp.163.com, user=*****@163.com, password=<non-null>
DEBUG SMTP: Attempt to authenticate using mechanisms: LOGIN PLAIN DIGEST-MD5 NTLM XOAUTH2 
DEBUG SMTP: Using mechanism LOGIN
DEBUG SMTP: AUTH LOGIN command trace suppressed
DEBUG SMTP: AUTH LOGIN succeeded
DEBUG SMTP: use8bit false
MAIL FROM:<*****@163.com>
250 Mail OK
RCPT TO:<*****@126.com>
250 Mail OK
DEBUG SMTP: Verified Addresses
DEBUG SMTP:   *****@126.com
DATA
354 End data with <CR><LF>.<CR><LF>
Date: Mon, 26 Oct 2020 10:55:52 +0800 (CST)
From: *****@163.com
To: *****@126.com
Message-ID: <122197673.0.1603680952847@*****.mshome.net>
Subject: =?UTF-8?B?5YWl6IGM5qyi6L+O?=
MIME-Version: 1.0
Content-Type: text/html;charset=UTF-8
Content-Transfer-Encoding: quoted-printable

<!DOCTYPE html>
<html lang=3D"en">
<head>
    <meta charset=3D"UTF-8">
    <title>=E5=85=A5=E8=81=8C=E6=AC=A2=E8=BF=8E=E9=82=AE=E4=BB=B6</title>
</head>
<body>
<!--
 @=E4=BD=9C=E8=80=85 =E6=B1=9F=E5=8D=97=E4=B8=80=E7=82=B9=E9=9B=A8
 @=E5=85=AC=E4=BC=97=E5=8F=B7 =E6=B1=9F=E5=8D=97=E4=B8=80=E7=82=B9=E9=9B=A8
 @=E5=BE=AE=E4=BF=A1=E5=8F=B7 a_java_boy
 @GitHub https://github.com/lenve
 @=E5=8D=9A=E5=AE=A2 http://wangsong.blog.csdn.net
 @=E7=BD=91=E7=AB=99 http://www.javaboy.org=20
 @=E6=97=B6=E9=97=B4 2019-11-24 7:48
-->
=E6=AC=A2=E8=BF=8E <span>=E9=98=BF=E7=91=9F=E4=B8=9C</span> =E5=8A=A0=E5=85=
=A5 Java=E8=BE=BE=E6=91=A9=E9=99=A2 =E5=A4=A7=E5=AE=B6=E5=BA=AD=EF=BC=8C=E6=
=82=A8=E7=9A=84=E5=85=A5=E8=81=8C=E4=BF=A1=E6=81=AF=E5=A6=82=E4=B8=8B=EF=BC=
=9A
<table border=3D"1">
    <tr>
        <td>=E5=A7=93=E5=90=8D</td>
        <td>=E9=98=BF=E7=91=9F=E4=B8=9C</td>
    </tr>
    <tr>
        <td>=E8=81=8C=E4=BD=8D</td>
        <td>=E6=8A=80=E6=9C=AF=E6=80=BB=E7=9B=91</td>
    </tr>
    <tr>
        <td>=E8=81=8C=E7=A7=B0</td>
        <td>=E6=95=99=E6=8E=88</td>
    </tr>
    <tr>
        <td>=E9=83=A8=E9=97=A8</td>
        <td>=E5=8D=8E=E5=8D=97=E5=B8=82=E5=9C=BA=E9=83=A8</td>
    </tr>
</table>


<p>=E5=B8=8C=E6=9C=9B=E5=9C=A8=E6=9C=AA=E6=9D=A5=E7=9A=84=E6=97=A5=E5=AD=90=
=E9=87=8C=EF=BC=8C=E6=90=BA=E6=89=8B=E5=85=B1=E8=BF=9B=EF=BC=81</p>

</body>
</html>
.
250 Mail OK queued as smtp10,DsCowACnrVG3OpZfr3upSA--.17032S2 1603680952
DEBUG SMTP: message successfully delivered to mail server
QUIT
221 Bye
2020-10-26 10:55:52.917  INFO 4456 --- [ntContainer#0-1] o.j.mailserver.receiver.MailReceiver     : fa2d85e8-c002-42bb-a7d1-1bc3cf6a43a0:邮件发送成功
```