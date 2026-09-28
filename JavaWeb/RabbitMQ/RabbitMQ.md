# RabbitMQ

> 消息中间件：让业务通过消息协作，支持异步处理、解耦和削峰。

## 消息怎么走

```text
生产者 → Exchange（交换机）→ Queue（队列）→ Consumer（消费者）
                 ↑
       Binding 定义路由规则
```

- RabbitMQ 是独立运行的服务；引入 Java 依赖不等于安装 RabbitMQ。
- 生产者指定交换机和 Routing Key；交换机根据绑定规则将消息送到队列。
- 同一队列的多个消费者通常分担消息；多个业务都要收到同一个事件，应各自建立队列。

| 交换机 | 路由方式 |
| --- | --- |
| Direct | 路由键精确匹配 |
| Fanout | 发给所有绑定的队列 |
| Topic | 按模式匹配；`*` 匹配一个单词，`#` 匹配零个或多个单词，单词用 `.` 分隔 |
| Headers | 按消息头匹配 |

## RabbitConfig：声明消息走哪条路

放在启动类所在包的 `config` 子包下，确保默认组件扫描能找到。类名和包名是约定，不是固定要求。

- `application.yml`：配置 RabbitMQ 地址、端口、连接账号。
- `RabbitConfig`：声明交换机、队列和绑定，不负责发送或消费。
- `@Configuration`：标记 Spring 配置类。
- `@Bean`：把方法返回的对象交给 Spring 管理；默认 Bean 名是方法名。
- `public static final`：定义可通过类名访问的常量，集中管理资源名称。

```java
@Configuration
public class RabbitConfig {
    public static final String EXCHANGE = "learning.exchange";
    public static final String QUEUE = "learning.queue";
    public static final String ROUTING_KEY = "learning.hello";

    @Bean
    public DirectExchange learningExchange() {
        // 名称、持久化、自动删除
        return new DirectExchange(EXCHANGE, true, false);
    }

    @Bean
    public Queue learningQueue() {
        // 名称、持久化；此构造方法默认非独占、不自动删除
        return new Queue(QUEUE, true);
    }

    @Bean
    public Binding learningBinding() {
        return BindingBuilder.bind(learningQueue())
                .to(learningExchange())
                .with(ROUTING_KEY);
    }
}
```

`Queue` 使用 `org.springframework.amqp.core.Queue`，不是 `java.util.Queue`。

**容易忘的细节：**

- `new Queue(...)` 先创建 Java 描述对象；常规自动配置下，`RabbitAdmin` 在建立连接时向 RabbitMQ 声明资源。
- `learningQueue` 是 Spring Bean 名，`learning.queue` 才是 RabbitMQ 队列名。
- 默认 `@Configuration` 开启方法代理：上面调用 `learningQueue()` 会获取 Spring 管理的单例 Bean。也可以改为在绑定方法参数中注入队列和交换机。
- 同名且属性一致的声明可重复执行；修改已有队列的持久化等属性可能导致声明冲突。
- 持久化交换机、队列定义，不等于消息一定不丢失。

## 请求怎么变成消息

```text
Controller 接收参数 → Service 发送消息 → RabbitMQ → Consumer → Service 执行业务
```

发送端在 Service 中注入 `RabbitTemplate`，核心调用：

```java
rabbitTemplate.convertAndSend(
        RabbitConfig.EXCHANGE,    // 交换机
        RabbitConfig.ROUTING_KEY, // 路由键，不是队列名
        content                  // 消息内容；字符串可直接用于入门示例
);
```

接收端监听队列，再调用业务 Service：

```java
@RabbitListener(queues = RabbitConfig.QUEUE)
public void consume(String content) {
    businessService.handle(content);
}
```

发送对象时，需要配置匹配的消息序列化与反序列化方式。

## Consumer 不替代 Mapper

```text
Controller（HTTP 入口） ─┐
                        ├→ Service（业务）→ Mapper（CRUD）→ 数据库
Consumer（消息入口） ────┘
```

Consumer 不只用于通知，也可以触发创建、修改、删除等业务。是否经过 Consumer，取决于业务是否由消息触发；实际数据库操作仍由 Mapper 执行。

**Trip 示例：**

- 同步：创建 Trip 基础记录，数据库提交后返回 Trip ID。
- 异步：发布 Trip 创建事件，由 Consumer 调用 Service 生成行程建议、发送邮件。
- 若把创建本身放入队列，接口返回的是“任务已受理”，不代表 Trip 已创建，需要任务状态查询。
- Consumer 应调用实际执行业务的方法，不要再次调用发送同一任务的方法，避免循环发消息。

## 哪些业务值得入队

**看是否允许稍后完成，以及异步能解决什么问题，不按业务重要程度划分。**

| 通常直接执行 | 适合考虑队列 |
| --- | --- |
| 查询 Trip 详情、列表 | AI 生成行程、生成报告 |
| 修改标题、日期 | 批量导出文件 |
| 创建基础记录并立即返回 ID | 发送邮件、同步搜索数据 |

队列适合耗时任务、突发流量、多个服务独立处理事件。普通 CRUD 项目可以很少用，甚至不用；没有固定业务占比。

## 几个边界

- **发送方法返回 ≠ 消息可靠入队 ≠ 消费者处理成功。** 发布确认与消费者 ACK 是不同环节。
- Controller 不等待消费者完成；消费者执行与 HTTP 返回没有固定先后顺序。
- 默认监听确认模式下，方法正常结束后由 Spring 容器确认消息；吞掉业务异常可能被当成成功。
- 消息可能重复投递，业务需要幂等；重试应有限次，避免失败消息不断重新入队。
- 数据库提交与消息发送不会天然一起成功，需要可靠衔接。
- 队列让任务排队，不自动提升处理速度；长期生产快于消费仍会积压。
