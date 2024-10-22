RabbitMQ官方网站：https://www.rabbitmq.com/

版本：3.12.14

我们从RabbitMQ发送消息的过程来了解。


# 一. AMQP

AMQP，高级消息队列协议（Advanced Message Queuing Protocol）。
基于此协议的客户端与消息中间件可传递消息，并不受产品、开发语言等条件的限制。

AMQP中增加了 Exchange 和 Binding 的角色。生产者把消息发布到 Exchange 上，消息最终到达队列并被消费者接收，而 Binding 决定交换器的消息应该发送到哪个队列。


AMQP 1.0 支持的JDK版本是11以上。0.9.1 支持JDK8以上。

> 版本对应关系参考：https://www.rabbitmq.com/client-libraries/java-versions


![image](assets/image-20241022102029-vznnsf4.png)
















# 二. RabbitMQ 架构图

![image](assets/image-20241017211702-ibhm4tu.png)


Message：消息包括 Headers（Exchange_Name、RoutingKey、Other Properties） 和 Body。
Headers 中保存了消息路由到的交换机以及和队列绑定相关的路由键以及其他的一些属性，比如 ttl-message 消息过期时间属性等。
Body 中保存的要发送的消息内容。

Channel：消息发送的通道。采用NIO设计，提供多路复用机制来降低性能开销。

Exchange：交换器。
用来接收生产者的消息，并将消息路由到目的队列中。
如果路由不到队列，则会直接丢弃（如果配置了死信队列则路由到队列中）、直接丢弃。取决于交换器的 mandatory 属性。

```markdown
1. 当发布的消息无法路由到任何队列，并且发布者将强制消息属性设置为 false（默认值），消息将会被丢弃或者重新发布到备用交换器（比如死信队列）。`mandatory = false`
2. 当发布的消息无法路由到任何队列，并且发布者将强制消息属性设置为 true，消息将会被返回。发布者必须设置返回的消息处理逻辑。`mandatory = true`
```

BindingKey：交换器与队列通过 BindingKey 建立绑定关系。

RoutingKey：生产者发送消息时一般会指定一个RoutingKey，用来指定消息的路由规则。

Queue：消息队列载体。消息会被路由到Queue队列中，队列中的消息会被 Consumer 消费，采用轮询的方式，不会出现一条消息被重复消费的情况。

Consumer：消费者。提供消息确认机制 MessageKnowledgement ，通过 autoAck 参数控制。
autoAck=true，消息发送出去就被认为消费成功。当TCP连接、channel 关闭时会出现消息丢失。
autoAck=false，需要用户手动进行确认，可以保证消息的可靠投递，会降低系统吞吐量。

Broker：消息队列服务器。

vHost：虚拟主机。一个broker可以有多个vhost，用来区分不同的权限。


交换器枚举 BuiltinExchangeType

```java
public enum BuiltinExchangeType {
    DIRECT("direct"), // 直连交换器
    FANOUT("fanout"), // 扇形交换器
    TOPIC("topic"), // 主题交换器
    HEADERS("headers"); // 头信息交换器
}
// 	Default("default) 默认交换器。
```


# 三. RabbitMQ API

API接口使用 AMQP 0.9.1 协议模型。

`com.rabbitmq.client` 为接口的顶级包。

> 参考连接：https://www.rabbitmq.com/client-libraries/java-api-guide


## 3.1 Channel

提供大部分操作，协议方法。

```java
// 使用前导入依赖
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;

// 与MQ建立连接
ConnectionFactory factory = new ConnectionFactory();
// "guest"/"guest" by default, limited to localhost connections
factory.setUsername(userName); // default guest
factory.setPassword(password); // default guest
factory.setVirtualHost(virtualHost); // /
factory.setHost(hostName); // localhost
factory.setPort(portNumber); // 5672端口用于常规连接，5671用于TLS连接

// 如果创建连接之前没有分配这些属性，则使用默认值。
Connection conn = factory.newConnection();
```


使用URIs方式

```java
ConnectionFactory factory = new ConnectionFactory();
factory.setUri("amqp://userName:password@hostName:portNumber/virtualHost");
Connection conn = factory.newConnection();
```


## 3.2 Connection

发送者和Broker建立消息发送连接。





## 3.3 ConnectionFactory

构造 Connection 实例。




## 3.4 Consumer

消息使用者。




## 3.5 DefaultConsumer

消费者常用的基类。




## 3.6 BasicProperties

消息属性（元数据）。




## 3.7 BasicProperties.Builder

BasicProperties 的构建器。



# 四、Spring AMQP

## 1. 配置文件

```yml
# 更新为自己项目的信息
spring:
  rabbitmq:
    host: "localhost"
    port: 5672
    username: "admin"
    password: "secret"

# 或者使用 addoress 形式配置
spring:
  rabbitmq:
    addresses: "amqp://admin:secret@localhost"
```


`ConnectionFactory`；通过定义 `ConnectionFactoryCustomizer` 来







































---


# 思考：使用MQ的同时会带来哪一些问题。

使用MQ消息队列需要考虑到后续问题出现的处理方式：消息丢失、消息重复，要保证数据的一致性。

上述情况的发生跟哪一个组件有关系？
消息发送方：只是将消息发送到Broker中存储。
Broker：存储发送的消息，将消息路由到消费者队列中。
消费者：消费消息。

消息丢失 --> 发送方发送消息时有消息没有到达Broker、Broker路由消息时没有将消息路由到消费者队列

消息重复消费 --> 发送方向Broker重复发送了一条消息、Broker重复发送了一条消息到消费者队列

## 1. 消息丢失


## 2. 重复消费



## 3. 数据一致性