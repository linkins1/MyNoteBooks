## #5.日志

### #5.1 日志框架历史

#### #5.1.1 抽象与实现

##### #1）抽象

日志的框架的抽象层有如下几类

- slf4j（simple logging for java）
- jcl（jakarta commons logging）
- jboss-logging

这几种抽象中，目前主流的是slf4j

##### #2）实现

- **logback**
- jul
- log4j

#### #5.1.2 使用

##### #1）将slf4j门面适配到不同实现

在实际使用时，门面常用slf4j，但是**由于只有logback按照slf4j进行实现**，其他几类实现如果想使用slf4j作为门面，就需要进行适配，如下图所示

![click to enlarge](http://www.slf4j.org/images/concrete-bindings.png)

如图中所示，绿色部分即为需要引入的适配层

> ***Q：*****为何通常使用slf4j作为门面？**
>
> ***A：***由于slf4j提供的门面（接口）更易于使用，springboot中默认也使用slf4j作为门面

##### #2）将其他框架转换为slf4j

由于不同的框架/项目使用的日志门面不一定为slf4j，**为了统一使用slf4j进行日志记录**，所以需要将这些门面也转换为slf4j门面，需要将对原来的门面的依赖（exclusion标签）排除掉，再用slf4j提供的jar包来替换，这样就完成了将其他门面转化为slf4j的目的，需要引入的转换包如下

![click to enlarge](http://www.slf4j.org/images/legacy.png)

### #5.2 springboot中的**日志**

#### #5.2.1 默认规则

**springboot使用slf4j+logback的组合进行日志记录**

#### #5.2.2适配

由于默认规则的存在，springboot为了能将引入的其他框架/项目中使用的日志框架都合并为slf4j+logback的形式，就需要按照上面提到的方式来对依赖进行改造

在spring-boot-starter中引入了spring-boot-starter-logging，这个starter就进行了框架的以来的转换，实现了下面两个目的

- 将其他门面转换为slf4j
- 统一使用logback

#### #5.2.3 使用

##### #1）日志级别

日志级别按下面排列级别递增，选中了某种级别，会输出大于等于当前级别的所有日志

- TRACE
- DEBUG
- INFO
- WARN
- ERROR
- FATAL
- OFF

##### #2）配置

###### #（1）yml中配置

- logging.level.root

  用于指定级别

- logging.file.name

  用于指定文件名称和路径

- logging.file.path

  用于指定路径，如果指定了名称，此项会被忽略

- logging.pattern.console

  用于指定控制台输出的日志格式

- logging.pattern.file

  用于指定文件中的输出的日志格式

###### #（2）配置文件(.xml)

- logback.xml

  直接被slf4j识别，其中可以指定yml中包含的所有配置

- logback-spring.xml

  可以被springboot识别，可以对logback.xml进行profile的增强，可以指定profile下进行日志记录

##### #3）编码

###### #（1）获取logger

通过`LoggerFactory.getLogger`方法传入要记录的类的Class对象，即可获取一个针对当前类的日志记录器

###### #（2）记录日志

根据不同的日志级别，调用logger的不同级别对应的方法即可实现记录日志

```java
@Component
public class QueueCleaner {

    public static final Logger logger = LoggerFactory.getLogger(QueueCleaner.class);

    @RabbitListener(queues = MailConstants.MAIL_QUEUE_NAME)
    public void handler(Message message, Channel channel) throws IOException {
        /**
        省略
        */
        if (redeliver_flag) {
            if (redisTemplate.opsForHash().entries("mail_log").containsKey(msgId)) {
        /**
        省略
        */
                logger.info(msgId + ":消息已经被消费");
        /**
        省略
        */
                return;
            }
        }
        try {

            logger.info(msgId + ":邮件发送成功");
        } catch (MailException | MessagingException e) {
            if (e.getMessage().contains("Invalid Address")) {
                channel.basicAck(tag, false);
                logger.error("邮件发送失败：地址无效" + e.getMessage());
            } else {

                logger.error("邮件发送失败：" + e.getMessage());
                logger.error("交由死信消费者处理");
            }
        }
    }
}
```