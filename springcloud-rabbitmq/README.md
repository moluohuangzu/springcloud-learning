# RabbitMQ 实现消息总线

在介绍基本概念之前直接先在主机上安装一下 RabbitMQ。

rabbitMQ 是一个在AMQP协议标准基础上完整的，可服用的企业消息系统。它遵循Mozilla Public License开源协议，采用 Erlang 实现的工业级的消息队列(MQ)服务器，Rabbit MQ 是建立在 Erlang OTP平台上。所以，在安装 RabbitMQ 之前先装 ERL 环境。

访问 [Erlang官网](http://www.erlang.org) 在下载页面选择适合自己操作系统的安装文件。

![erl-download.png](images/erl-download.png)

下载后直接安装，并在环境变量中配置变量。环境变量要指到 bin 目录，配置好后再命令行中输入 `erl -version`，能输出版本号即 Erl 环境已经配置完成。

![erl-version.png](images/erl-version.png)

在进入 RabbitMQ官网 下载 MQ，找到对应的系统环境直接下载安装 next、next and next......

![rabbitmq-info.png](images/rabbitmq-info.png)
![rabbitmq-download.png](images/rabbitmq-download.png)

安装完成后，就能在服务中看到 rabbitMQ 服务

![rabbitmq-server.png](images/rabbitmq-server.png)

接着进入 rabbitMQ 安装目录下的 `sbin` 文件夹，打开控制台执行 `rabbitmqctl status` 查看参数信息

![install-success.png](images/install-success.png)

一切正常后就开启 web 管理后台，在命令行中继续输入 `rabbitmq-plugins enable rabbitmq_management` 命令

![web-plugins-manegement.png](images/web-plugins-manegement.png)

重启服务，访问 `http://localhost:15672` 就能看到管理后台，默认账号密码为 `guest`。

现在试着在用户管理中新建一个用户：

![rabbitMQ-adduser.png](images/rabbitMQ-adduser.png)

rabbitMQ 的安装到此结束

---

创建工程，并在 pom 依赖中引入 `spring-boot-starter-amqp` 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

配置文件内容如下：

```properties
spring.application.name=rabbitmq-hello

spring.rabbitmq.host=localhost
spring.rabbitmq.username=springcloud
spring.rabbitmq.password=123456
spring.rabbitmq.port=5672
```

- `spring.rabbitmq.host`：rabbitMQ 主机地址
- `spring.rabbitmq.username`：rabbitMQ 账号
- `spring.rabbitmq.password`：rabbitMQ 密码
- `spring.rabbitmq.port`：rabbitMQ 端口

启动类不需要做任何改变。创建一个消息生成这类 `Sender`

```java
@Component
public class Sender {

	private static final Logger LOGGER = LoggerFactory.getLogger(Sender.class);

	@Resource
	private AmqpTemplate amqpTemplate;

	public void send() {
		String context = "Hello " + new Date();
		LOGGER.info("<<== The Producer Generates a Message: {}", context);
		amqpTemplate.convertAndSend("hello", context);
	}
}
```

在消息生成这类中注入 `AmqpTemplate` 接口来实现消息的发送。在生成者中，我们产生一个字符串，并发送至 hello 的队列中。

接着创建消费者类 `Receiver`

```java
@Component
@RabbitListener(queues = "hello")
public class Receiver {

	private static final Logger LOGGER = LoggerFactory.getLogger(Receiver.class);

	@RabbitHandler
	public void process(String context) {
		LOGGER.info("Receiver: {}", context);
	}
}
```

在消费者类上加上 `@RabbitListener` 注解并制定消息队列，在消费体中使用 `@RabbitHandler` 注解来指定对消息的处理方法。所以，该消费者实现了对 hello 队列的消费。

再创建 `RabbitConfig` 配置类，用来配置队列、交换器、路由等信息。

```java
@Configuration
public class RabbitConfig {

	@Bean
	public Queue helloQueue() {
		return new Queue("hello");
	}
}
```

最后再创建测试单元，用来调用消息生产者。

```java

@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringcloudBusRabbitmqApplicationTests {

	@Resource
	private Sender sender;

	@Test
	public void contextLoads() {
		sender.send();
	}

}
```

完成准备工作之后，就来启动应用主类。在启动时发现包错，错误信息如下：

![rabbitmq-authority.png](images/rabbitmq-authority.png)

原因是在配置文件中配置的用户没有分配权限，在刚才创建的 `springcloud` 账号除了创建外没有分配任何权限，可以看到信息面板：

![rabbitmq-user-no-authority.png.png](images/rabbitmq-user-no-authority.png.png)

只需要给该用户分配权限即可！

![rabbitmq-user-set-authority.png](images/rabbitmq-user-set-authority.png)

现在再来重启服务，会看到 rabbitMQ 的连接信息：

![rabbitq-connect-info.png](images/rabbitq-connect-info.png)

运行测试类，从控制台打印的日志中可以看到生成了一条数据：

![send-message.png](images/send-message.png)

再来看下应用主类控制台，成功的消费了这条消息！

![receiver-message.png](images/receiver-message.png)

---

在前面展示了 rabbitMQ 消息总线的 生产者与消费者的一个简单小应用。

rabbitMQ 是实现了高级消息队列协议的开源消息代理软件，也称为面向消息的中间件。rabbitMQ 是用高可用、可伸缩而闻名的 Erlang 语言编写而成的，其集群和故障转移是构建在开放电信平台框架上的。

**rabbitMQ 的基本概念** ：

- `Broker`：消息队列服务器的实体。是一个中间件应用，负责接收生产者的消息，然后将消息发送至消费者或其他的 Broker。
- `Exchange`：消息交换机。是消息第一个到达的地方，消息通过它指定的路由规则分发到不同的消息队列中去。
- `Queue`：消息队列。消息通过发送和路由之后最终到达的地方。到达 Queue 的消息即进入逻辑上等待消费的状态。每个消息都会被发送到一个或多个队列。
- `Binding`：绑定。它的作用就是将 Exchange 和 Queue 安装路由规则绑定起来，也就是两者之间的虚拟连接。
- `Routing Key`：路由关键字。Exchange 根据这个关键字进行消息投递。
- `Virtual host`：虚拟主机。是对 Broker 的虚拟划分，将消费者、生产者和他们的依赖的 AMQP 相关结构进行隔离，一般是为了安全考虑。比如：可以在一个 Broker 中设置多个虚拟主机，对不同用户进行权限的分离。
- `Connection`：连接。代表生产者、消费者、Broker之间进行通信的物理网络。
- `Channel`：消息通道。用于连接生产者、消费者的逻辑结构。在客户端的每一个连接里可建立多个 Channel，每个 Channel 代表一个会话任务，通过 Channel 可以隔离同一连接中的不同交互内容。
- `Producer`：消息生产者。
- `Consumer`：消息消费者。

**消息投递到队列大致过程** ：

1. 客户端连接带消息队列服务器，打开一个 Channel。
2. 客户端生成一个 Exchange，并设置相关属性。
3. 客户端声明一个 Queue，并设置相关属性。
4. 客户端使用 Routing Key，在 Exchange 和 Queue 之间建立好绑定关系。
5. 客户端投递消息到 Exchange。
6. Exchange 接收到消息后，根据消息的 Key 和已经谁知的 Binding 进行消息路由。将消息投递到一个或多个 Queue 里。
   - Direct 交换机：完全根据 Key 进行投递。比如：绑定时设置了 Routing Key 为 abc，那么客户端提交的信息，只有设置了 Key 为 abe 的才会被投递到队列。
   - Topic 交换机：对 Key 进行模糊匹配后进行投递，可以使用符号 `#` 匹配一个或多个词，符号 `*` 匹配正好有个词。比如 `abc.#` 匹配 `abc.def.ghi`， `abc.*` 只匹配 `abc.def`。
   - Fanout 交换机：不需要设置 Key，它采用广播的模式，一个消息进来时被投递到与该交换机绑定的所有队列。
   
---

在前面的的工程中有介绍 SpringCloud Config 的动态更新。我们在远程仓库的配置中心中配置信息，当配置中心的配置信息更新时我们需要调用 `/actuator/refresh` POST 端点去更新各个客户端的配置。那这里就来看下如何通过 RabbitMQ 的消息总线去更新一个服务的各个实例的配置信息！

这里不需要创建任何工程，只需要将 [服务注册中心](../springcloud-eureka) 启动起来，然后将 [springcloud-config-client-eureka](../springcloud-config-client-eureka) 和 [springcloud-config-server-eureka](../springcloud-config-server-eureka) 工程各拷贝一份，分别命名为 [springcloud-config-client-eureka-bus](../springcloud-config-client-eureka-bus)(Config Eureka Client) 和 [springcloud-config-server-eureka-bus](../springcloud-config-server-eureka-bus)(Config Eureka Server)。

在这两个工程中分别引入 `spring-cloud-starter-bus-amqp` 和 `spring-boot-starter-actuator` 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

> **注意：** spring-boot-starter-actuator 依赖也是必须的，用于提供刷新端点。

在 Config Server 的配置文件中在之前的基础上增加如下配置信息：

```properties
#暴露所有端点
management.endpoints.web.exposure.include=*
## RabbitMQ
spring.rabbitmq.port=5672
spring.rabbitmq.host=localhost
spring.rabbitmq.username=springcloud
spring.rabbitmq.password=123456
```

而在 Config Client 中的配置文件中只需要增加 `management.endpoints.web.exposure.include=*` 配置即可！

现在启动 [springcloud-config-server-eureka-bus](../springcloud-config-server-eureka-bus)(7001) 和 [springcloud-config-client-eureka-bus](../springcloud-config-client-eureka-bus)。其中 springcloud-config-client-eureka-bus 启动不同端口的两个实例，如(7002,7003)。

在注册中心中看到三个服务如下三个服务：

![eureka-server.png](images/eureka-server.png)

现在通过工具或浏览器访问 `http://localhost:7001/actuator`，会看到 Config Server 暴露的端点如下：

```json
{
    "_links": {
        "self": {
            "href": "http://localhost:7001/actuator",
            "templated": false
        },
        "archaius": {
            "href": "http://localhost:7001/actuator/archaius",
            "templated": false
        },
        "auditevents": {
            "href": "http://localhost:7001/actuator/auditevents",
            "templated": false
        },
        "beans": {
            "href": "http://localhost:7001/actuator/beans",
            "templated": false
        },
        "health": {
            "href": "http://localhost:7001/actuator/health",
            "templated": false
        },
        "conditions": {
            "href": "http://localhost:7001/actuator/conditions",
            "templated": false
        },
        "configprops": {
            "href": "http://localhost:7001/actuator/configprops",
            "templated": false
        },
        "env": {
            "href": "http://localhost:7001/actuator/env",
            "templated": false
        },
        "env-toMatch": {
            "href": "http://localhost:7001/actuator/env/{toMatch}",
            "templated": true
        },
        "info": {
            "href": "http://localhost:7001/actuator/info",
            "templated": false
        },
        "loggers": {
            "href": "http://localhost:7001/actuator/loggers",
            "templated": false
        },
        "loggers-name": {
            "href": "http://localhost:7001/actuator/loggers/{name}",
            "templated": true
        },
        "heapdump": {
            "href": "http://localhost:7001/actuator/heapdump",
            "templated": false
        },
        "threaddump": {
            "href": "http://localhost:7001/actuator/threaddump",
            "templated": false
        },
        "metrics-requiredMetricName": {
            "href": "http://localhost:7001/actuator/metrics/{requiredMetricName}",
            "templated": true
        },
        "metrics": {
            "href": "http://localhost:7001/actuator/metrics",
            "templated": false
        },
        "scheduledtasks": {
            "href": "http://localhost:7001/actuator/scheduledtasks",
            "templated": false
        },
        "httptrace": {
            "href": "http://localhost:7001/actuator/httptrace",
            "templated": false
        },
        "mappings": {
            "href": "http://localhost:7001/actuator/mappings",
            "templated": false
        },
        "refresh": {
            "href": "http://localhost:7001/actuator/refresh",
            "templated": false
        },
        "bus-env-destination": {
            "href": "http://localhost:7001/actuator/bus-env/{destination}",
            "templated": true
        },
        "bus-env": {
            "href": "http://localhost:7001/actuator/bus-env",
            "templated": false
        },
        "bus-refresh-destination": {
            "href": "http://localhost:7001/actuator/bus-refresh/{destination}",
            "templated": true
        },
        "bus-refresh": {
            "href": "http://localhost:7001/actuator/bus-refresh",
            "templated": false
        },
        "features": {
            "href": "http://localhost:7001/actuator/features",
            "templated": false
        },
        "service-registry": {
            "href": "http://localhost:7001/actuator/service-registry",
            "templated": false
        },
        "bindings-name": {
            "href": "http://localhost:7001/actuator/bindings/{name}",
            "templated": true
        },
        "bindings": {
            "href": "http://localhost:7001/actuator/bindings",
            "templated": false
        },
        "channels": {
            "href": "http://localhost:7001/actuator/channels",
            "templated": false
        }
    }
}
```

这里主要使用 `bus-refresh` 和 `bus-refresh-destination` 端点用于刷新配置，这两个端点都是 POST请求。

现在分别访问 `http://localhost:7002/from` 和 `http://localhost:7003/from`，会看到成功返回配置文件中的 from 属性值：`git-dev-1.0`。

现在，修改配置中心 from 属性值为 `git-dev-2.0`，并请求 `http://localhost:7002/actuator/refresh` 或 `http://localhost:7003/actuator/refresh`。在次请求两个客户端信息就会发现两个客户端返回的数据是 `git-dev-2.0`。成功刷新配置信息！
同样的，当我们请求 `http://localhost:7001/actuator/refresh` 时也是能够刷新配置！

>**注意：** 有一点需要注意，如果发现配置属性值并没有被更新但是在请求刷新节点时在控制台输出了刷新信息，需要看下是否在需要刷新配置属性的类上增加了 `@RefreshScope` 注解。只要是需要动态更新的配置都需要在类上增加该注解！

现在我们再来看下为什么不管是请求客户端刷新端点还是请求服务端刷新节点都能刷新配置信息。

首先，来看下当请求客户端时的示意图：

![rabbitmq-client-pic.png](images/rabbitmq-client-pic.png)

当我们请求客户端的刷新端点时，客户端就向消息总线发送（生产）一条消息到消息总线上。消息总线在收到该消息之后就会分发消息至同一服务的不同实例，客户端收到消息后并消费消息。各个客户端都会向服务端请求配置信息！

现在，再来看下当请求服务端时的示意图：

![rabbitmq-server-pic.png](images/rabbitmq-server-pic.png)

当请求服务端的刷新端点是，同样的会向消息总线发送消息。与客户端不同的是，这次的消息总线是向所有的客户端发送消息（不同服务的不同实例）。接着所有的客户端实例都会消费这个消息，请求刷新配置端点信息。

可以看到，请求服务端与请求客户端的刷新端点之间的唯一区别就是：请求客户端是刷新同一服务的不同实例配置信息。而请求服务端则是刷新不同服务的所有实例配置信息！这是唯一的区别！

某些场景下（例如灰度发布），我们可能只想刷新部分微服务的配置，此时可通过 `/actuator/bus-refresh/{destination}` 端点的 destination 参数来定位要刷新的应用程序。例如：`/actuator/bus-refresh/config:7002`，这样消息总线上的微服务实例就会根据 destination 参数的值来判断是否需要要刷新。其中，`/actuator/bus-refresh/config:7002` 指的是各个微服务的 ApplicationContext ID，也可以说是 `${spring.application.name}:${server.port}`。destination 参数也可以用来定位特定的微服务。例如：`/actuator/bus-refresh/config:**`，这样就可以触发 config 微服务所有实例的配置刷新。

# Kafka 实现消息总线

现在对 RabbitMQ 已经有了基本的了解，现在看下使用 Kafka 实现消息总线。在实践之前同样的还是先安装 Kafka 环境！

首先要进入 Apache 官网在zookeeper下载页面 [zookeeper](https://www.apache.org/dyn/closer.cgi/zookeeper/) 下载。

![zookeeper-download.png](images/zookeeper-download.png)

下载完成后进行解压，配置环境变量

![zookeeper-enviro-path.png](images/zookeeper-enviro-path.png)

在命令终端中输入 `zkServer` 能看到如下信息，即表示已经 OK 了

![zkServer.png](images/zkServer.png)

在来下载，同样的进入 [Kafka 下载页面](http://kafka.apache.org/downloads) 下载

![kafka-download.png](images/kafka-download.png)

![kafka-enviro-path.png](images/kafka-enviro-path.png)

这里对 kafka 的安装目录下的各个文件不做讲解，直接在 bin 的同级目录中打开控制台 输入命令: `.\bin\windows\kafka-server-start.bat .\config\server.properties` 启动服务

在启动时会打印会打印许多配置信息！到此已经基本完成。现在我们可以来创建一个生产者和一个消费者。在bin目录下（或bin的windows文件夹下）打开终端控制台创建一个 Topic 为 test 的生产者：`kafka-console-producer.bat --broker-list localhost:9092 --topic test`，
在打开一个控制台创建一个消费 test 的消费者：` kafka-console-consumer.bat --zookeeper localhost:2181 --topic test --from-beginning`

>**注意：** 这里消费者的 zookeeper 端口是 2181 原因是zookeeper 的启动端口配置的是 2181，可以在 zookeeper 的配置文件中自行更改。

可以测试一下在生成者中输入消息是，在消费者的控制台中能看到消费的信息！

![kafka-pro-con.png](images/kafka-pro-con.png)

这里对 Kafka 的概念讲一下：

- Broker：Kafka 集群包含一个或多个服务器，这些服务器被称为 Broker。
- Topic：逻辑上同 RabbitMQ 的 Queue 队列相似，每条发布到 Broker 集群的消息都必须有一个 Topic。
- Partition：Partition 是物理概念上的分区，为了提供系统的吞吐率，在物理上每个 Topic 会分成一个或多个 Partition，每个Partition对应一个文件夹（存储对应分区的消息内容和索引文件）
- Producer：消息生产者，负责生产消息并发送到 Broker。
- Consumer：消息消费者，向 Kafka 读取消息并处理的客户端。
- Consumer Group：每个 Consumer 属于一个特定的组（可为每个 Consumer 指定属于一个组，若不指定则属于默认组），组可以用来实现一条消息被组内多个成员消费等功能。

**整个 Spring Cloud Bus**

这里直接在 [springcloud-config-server-eureka-bus](../springcloud-config-server-eureka-bus) 和 [springcloud-config-client-eureka-bus](../springcloud-config-client-eureka-bus) 工程中进行修改即可！将 pom 依赖中的 `spring-cloud-starter-bus-amqp` 依赖注释掉
然后将 `spring-cloud-starter-bus-kafka` 依赖引入即可！

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-kafka</artifactId>
</dependency>
```

启动服务注册中心 [springcloud-eureka](../springcloud-eureka) 并将启动两个工程，其中客户端启动两个实例！

在控制台中能可以看到，服务连接到 Kafka 中，并使用了名为 springCloudBus 的 Topic！

![console-info.png](images/console-info.png)

在命令终端输入 `kafka-topics.bat --list --zookeeper localhost:2181` 看到 Topic 集合中确实多了一个名为 springCloudBus 的 Topic：

![topic-list.png](images/topic-list.png)

访问 `127.0.0.1:7002/from` 或 `127.0.0.1:7003/from` 可以看到返回的信息是 `git-dev-1.0` ，现在修改配置仓库中的from 属性值 为 git-dev-2.0。访问 `/actuator/bus-refresh` 端点刷新配置，再次访问 `/from` 端点，可以看到返回的信息是 `git-dev-2.0`。说明成功更新！

刷新端点操作与 RabbitMQ 完全一样，这里不做多余解释。只需要在依赖中引入 kafka 依赖即可！

在服务启动时，在控制台中看到创建了一个名为 springCloudBus 的 Topic。当刷新端点时，我们可以在控制台中启动对该 Topic 的监控观察：
```
# 在命令终端中输入
kafka-console-consumer.bat --zookeeper localhost:2181 --topic springCloudBus
```

访问刷新端点是就会打印如下信息：
```json
// http://localhost:7001/actuator/bus-refresh 端点
[
    {
        "destinationService": "**",
        "id": "a937bbf0-9d9c-498c-9d3f-0412bb5d9302",
        "originService": "config-server:7001:cb9717cd6cda905cf5c8afe58612059d",
        "timestamp": 1532746023505,
        "type": "RefreshRemoteApplicationEvent"
    },
    {
        "ackDestinationService": "**",
        "ackId": "a937bbf0-9d9c-498c-9d3f-0412bb5d9302",
        "destinationService": "**",
        "event": "org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent",
        "id": "f919728f-c14f-48fd-8ec4-8ee935c8caf5",
        "originService": "config-server:7001:cb9717cd6cda905cf5c8afe58612059d",
        "timestamp": 1532746023508,
        "type": "AckRemoteApplicationEvent"
    },
    {
        "ackDestinationService": "**",
        "ackId": "a937bbf0-9d9c-498c-9d3f-0412bb5d9302",
        "destinationService": "**",
        "event": "org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent",
        "id": "a4ed3f6a-a0df-46a3-b7b1-356a590bfbb7",
        "originService": "config:7003:f516c9c40f621d674abd898dbb5339be",
        "timestamp": 1532746028330,
        "type": "AckRemoteApplicationEvent"
    },
    {
        "ackDestinationService": "**",
        "ackId": "a937bbf0-9d9c-498c-9d3f-0412bb5d9302",
        "destinationService": "**",
        "event": "org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent",
        "id": "f728f3ec-cb84-405c-985c-02ee0ea28499",
        "originService": "config:7002:863a4796937ce3f13dfd500d828ac835",
        "timestamp": 1532746028841,
        "type": "AckRemoteApplicationEvent"
    }
]
```

```json
// http://localhost:7002/actuator/bus-refresh/config:7003 端点
[
    {
        "destinationService": "config:7003:**",
        "id": "1675a3ec-56c7-4738-85c0-9b03e3f9eb67",
        "originService": "config:7002:863a4796937ce3f13dfd500d828ac835",
        "timestamp": 1532746200095,
        "type": "RefreshRemoteApplicationEvent"
    },
    {
        "ackDestinationService": "config:7003:**",
        "ackId": "1675a3ec-56c7-4738-85c0-9b03e3f9eb67",
        "destinationService": "**",
        "event": "org.springframework.cloud.bus.event.RefreshRemoteApplicationEvent",
        "id": "ac2037f4-e35c-4ea1-9b54-3110d908e949",
        "originService": "config:7003:f516c9c40f621d674abd898dbb5339be",
        "timestamp": 1532746205406,
        "type": "AckRemoteApplicationEvent"
    }
]
```

下面，来相信说下信息内容：

* `RefreshRemoteApplicationEvent` 和 `AckRemoteApplicationEvent` 共有属性
  + type：消息的事件类型。
    - `RefreshRemoteApplicationEvent` 是用来刷新配置的事件
    - `AckRemoteApplicationEvent` 是相应消息已经正确解释的告知消息事件
  + timestamp：消息的时间戳。
  + destinationService：消息的目标服务实例。
  + id：消息的唯一标识。
* `AckRemoteApplicationEvent` 特有属性
  + ackId：Ack 消息对应的消息来源。从上面的信息中可以看到 type 为 `AckRemoteApplicationEvent` 类型的 ackId 对应 type 为 `RefreshRemoteApplicationEvent` 类型的id。
  + ackDestinationService：ack 消息的目标服务实例。`**` 表示消息总线上的所有所有实例都接收到 ack 消息。`config:7003:**` 则表示只有服务为 Config 端口为 7003 的实例才接受到消息。
  + event：ack 消息的来源事件。