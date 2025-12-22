# 									RabbitMQ说明文档

## 1.引入依赖

```xml
 <!-- RabbitMQ 消息队列 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

## 2.配置连接信息

```yaml
spring:
    rabbitmq:
        host: 127.0.0.1
        port: 5672
        username: guest
        password: guest
        virtual_host: /
```

## 3.工作模式配置

### 1)直连工作模式(一个消费者绑定到一个队列)

```java
@Configuration
public class RabbitmqZLConfig {

    public static final String QUEUE_KEY = "springboot_rabbitmq_zl";

    //直连
    @Bean
    public Queue queue() {
        return new Queue(QUEUE_KEY, true);
    }

}

----------------------------------------------------------------------------------------------------------
//发送消息
@RestController
public class RabbitController {

    @Autowired
    private MsgSender msgSender;

    /**
     * 直连
     */
    @GetMapping(value = "/sendMsg", produces = "application/json;charset=utf-8")
    public void sendMsg() {
        msgSender.send();
    }
}

----------------------------------------------------------------------------------------------------------
@Component
@Slf4j
public class MsgSender {
    @Autowired
    private AmqpTemplate rabbitTemplate;

    public void send() {
        String msgContent = "Hello World ~";
        log.info("（直连）生产者发送消息 : " + msgContent);
        this.rabbitTemplate.convertAndSend(RabbitmqZLConfig.QUEUE_KEY, msgContent);
    }
}

----------------------------------------------------------------------------------------------------------
//接收消息
@Slf4j
@Component
public class MsgReceiver {

    @RabbitListener(queues = RabbitmqZLConfig.QUEUE_KEY)
    public void process(String msg) {
        log.info("消费者1接收消息 : " + msg);
	}
}
```

### 2）工作队列模式(让**多个消费者绑定到一个队列，共同消费队列中的消息**。)

```java
@Configuration
public class RabbitmqWorkConfig {

    public static final String QUEUE_WORK = "springboot_rabbitmq_work";

    //工作队列
    @Bean
    public Queue workQueue() {
        return new Queue(QUEUE_WORK, true);
    }

}

----------------------------------------------------------------------------------------------------------
//发送消息
@RestController
public class RabbitController {
    @Autowired
    private MsgSender msgSender;
    /*
      工作队列
     */
    @GetMapping(value = "/sendMsg2", produces = "application/json;charset=utf-8")
    public void sendMsg2() {
        msgSender.send2();
    }
}

----------------------------------------------------------------------------------------------------------
@Component
@Slf4j
public class MsgSender {
    @Autowired
    private AmqpTemplate rabbitTemplate;

    public void send2() {
        String msgContent = "Hello World ~";
        log.info("(工作队列)生产者发送消息 : " + msgContent);
        for (int i = 0; i < 5; i++) {
            this.rabbitTemplate.convertAndSend(RabbitmqWorkConfig.QUEUE_WORK, msgContent);
        }
    }
}


----------------------------------------------------------------------------------------------------------
/**
多个消费者监听同一个队列，默认为轮询接收消息
*/
public class MsgReceiver {

    @RabbitListener(queues = RabbitmqWorkConfig.QUEUE_WORK)
    public void process2(String msg) {
        log.info("消费者2接收消息 : " + msg);
    }

    @RabbitListener(queues = RabbitmqWorkConfig.QUEUE_WORK)
    public void process3(String msg) {
        log.info("消费者3接收消息 : " + msg);
    }
}
```

### 3）发布订阅（广播）

```java
package com.wzs.springboottest.config.rabbitmq;


import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.FanoutExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitmqFanoutConfig {

    public static final String QUEUE_FANOUT1 = "springboot_rabbitmq_fanout_queue";

    public static final String QUEUE_FANOUT2 = "springboot_rabbitmq_fanout2_queue";

    public static final  String EXCHANGE_FANOUT1 = "springboot_rabbitmq_exchange_fanout1";


    //广播队列
    @Bean
    public Queue fanoutQueue1() {
        return new Queue(QUEUE_FANOUT1, true);
    }

    @Bean
    public Queue fanoutQueue2() {
        return new Queue(QUEUE_FANOUT2, true);
    }
    //交换机
    @Bean
    FanoutExchange fanoutExchange1(){
        FanoutExchange fanoutExchange = new FanoutExchange(EXCHANGE_FANOUT1);
        return fanoutExchange;
    }
    //绑定
    @Bean
    Binding bindingQueue1(){
        Binding binding = BindingBuilder.bind(fanoutQueue1()).to(fanoutExchange1());
        return binding;
    }

    @Bean
    Binding bindingQueue2(){
        Binding binding = BindingBuilder.bind(fanoutQueue2()).to(fanoutExchange1());
        return binding;
    }

}
-------------------------------------------------------------------------------------------------------
@RestController
public class RabbitController { 
    @Autowired
    private MsgSender msgSender;
    /*
    广播队列
   */
    @GetMapping(value = "/sendMsg3", produces = "application/json;charset=utf-8")
    public void sendMsg3() {
        msgSender.send3();
    }
}
-------------------------------------------------------------------------------------------------------
/**
	生产者向【广播交换机】发送消息，而不向队列发送消息，交换机会将消息发送到绑定了该交换机的所有队列都发送消息
*/
@Component
@Slf4j
public class MsgSender {
    @Autowired
    private AmqpTemplate rabbitTemplate;
    public void send3() {
        String msgContent = "Hello World ~";
        log.info("(广播)生产者发送消息 : " + msgContent);
        for (int i = 0; i < 5; i++) {
            this.rabbitTemplate.convertAndSend(RabbitmqFanoutConfig.EXCHANGE_FANOUT1,"", msgContent);
        }
    }
}
-------------------------------------------------------------------------------------------------------
@Slf4j
@Component
public class MsgReceiver {
     @RabbitListener(
            bindings=@QueueBinding(
                    value = @Queue(RabbitmqFanoutConfig.QUEUE_FANOUT1),
                    exchange = @Exchange(name = RabbitmqFanoutConfig.EXCHANGE_FANOUT1,type = ExchangeTypes.FANOUT))
    )
    public void process4(String msg) {
        log.info("消费者4接收消息 : " + msg);
    }

    @RabbitListener(
            bindings=@QueueBinding(
                    value = @Queue(RabbitmqFanoutConfig.QUEUE_FANOUT2),
                    exchange = @Exchange(name = RabbitmqFanoutConfig.EXCHANGE_FANOUT1,type = ExchangeTypes.FANOUT))
    )
    public void process5(String msg) {
        log.info("消费者5接收消息 : " + msg);
    }
}
```

### 4)路由key

```java
package com.wzs.springboottest.config.rabbitmq;


import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitmqRoutingConfig {

    public static final String QUEUE_ROUTING1 = "queue_routing1";
    public static final String QUEUE_ROUTING2 = "queue_routing2";
    public static final String QUEUE_ROUTING3 = "queue_routing3";
    public static final String QUEUE_ROUTING4 = "queue_routing4";
    public static final String EXCHANGE_ROUTING = "exchange_routing";

    @Bean
    public Queue routingQueue1(){
        Queue queue = new Queue(RabbitmqRoutingConfig.QUEUE_ROUTING1);
        return queue;
    }

    @Bean
    public Queue routingQueue2(){
        Queue queue = new Queue(RabbitmqRoutingConfig.QUEUE_ROUTING2);
        return queue;
    }

    @Bean
    public Queue routingQueue3(){
        Queue queue = new Queue(RabbitmqRoutingConfig.QUEUE_ROUTING3);
        return queue;
    }

    @Bean
    public Queue routingQueue4(){
        Queue queue = new Queue(RabbitmqRoutingConfig.QUEUE_ROUTING4);
        return queue;
    }

    @Bean
    public DirectExchange directExchange(){
        DirectExchange directExchange = new DirectExchange(RabbitmqRoutingConfig.EXCHANGE_ROUTING);
        return directExchange;
    }
	/**
		将队列与交换机进行绑定，并指定路由key
	*/
    @Bean
    public Binding bindingRoutingQueue1(){
        Binding binding = BindingBuilder.bind(routingQueue1()).to(directExchange()).with("info");
        return binding;
    }

    @Bean
    public Binding bindingRoutingQueue2(){
        Binding binding = BindingBuilder.bind(routingQueue2()).to(directExchange()).with("warm");
        return binding;
    }

    @Bean
    public Binding bindingRoutingQueue3(){
        Binding binding = BindingBuilder.bind(routingQueue3()).to(directExchange()).with("debug");
        return binding;
    }

    @Bean
    public Binding bindingRoutingQueue4(){
        Binding binding = BindingBuilder.bind(routingQueue4()).to(directExchange()).with("error");
        return binding;
    }
}
------------------------------------------------------------------------------------------------------------
 @RestController
public class RabbitController {
    @Autowired
    private MsgSender msgSender;
    /*
    路由key
  */
    @GetMapping(value = "/sendMsg4", produces = "application/json;charset=utf-8")
    public void sendMsg4() {
        msgSender.send4();
    }
}
------------------------------------------------------------------------------------------------------------
  @Component
@Slf4j
public class MsgSender {
    @Autowired
    private AmqpTemplate rabbitTemplate;
    public void send4() {
        String msgContent = "Hello World ~";
        log.info("(路由)生产者发送消息 : " + msgContent);
        this.rabbitTemplate.convertAndSend(RabbitmqRoutingConfig.EXCHANGE_ROUTING,"info", msgContent);
        this.rabbitTemplate.convertAndSend(RabbitmqRoutingConfig.EXCHANGE_ROUTING,"warm", msgContent);
        this.rabbitTemplate.convertAndSend(RabbitmqRoutingConfig.EXCHANGE_ROUTING,"debug", msgContent);
        this.rabbitTemplate.convertAndSend(RabbitmqRoutingConfig.EXCHANGE_ROUTING,"error", msgContent);

    }
}
------------------------------------------------------------------------------------------------------------
@Slf4j
@Component
public class MsgReceiver {    
      @RabbitListener(
            bindings=@QueueBinding(
                    value = @Queue(RabbitmqRoutingConfig.QUEUE_ROUTING1),
                    key={"info"},
                    exchange = @Exchange(name = RabbitmqRoutingConfig.EXCHANGE_ROUTING,type = ExchangeTypes.DIRECT)
                   )
    )
    public void process6(String msg) {
        log.info("消费者6接收消息info : " + msg);
    }

    @RabbitListener(
            bindings=@QueueBinding(
                    value = @Queue(RabbitmqRoutingConfig.QUEUE_ROUTING2),
                    key={"warm"},
                    exchange = @Exchange(name = RabbitmqRoutingConfig.EXCHANGE_ROUTING,type = ExchangeTypes.DIRECT)
                    )
    )
    public void process7(String msg) {
        log.info("消费者7接收消息warm : " + msg);
    }

    @RabbitListener(
            bindings=@QueueBinding(
                    value = @Queue(RabbitmqRoutingConfig.QUEUE_ROUTING3),
                    key={"debug"},
                    exchange = @Exchange(name = RabbitmqRoutingConfig.EXCHANGE_ROUTING,type = ExchangeTypes.DIRECT)
                    )
    )
    public void process8(String msg) {
        log.info("消费者8接收消息debug : " + msg);
    }

    @RabbitListener(
            bindings=@QueueBinding(
                    value = @Queue(RabbitmqRoutingConfig.QUEUE_ROUTING4),
                    key={"error"},
                    exchange = @Exchange(name = RabbitmqRoutingConfig.EXCHANGE_ROUTING,type = ExchangeTypes.DIRECT)
                    )
    )
    public void process9(String msg) {
        log.info("消费者9接收消息error : " + msg);
    }
}
```

### 5)通配符（topic）

***可以匹配多个层级，#只能匹配一个层级**

```java
package com.wzs.springboottest.config.rabbitmq;

import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;


@Configuration
public class RabbitmqTopicConfig {
    public static final String QUEUE_TOPIC1 = "queue_topic1";
    public static final String QUEUE_TOPIC2 = "queue_topic2";
    public static final String EXCHANGE_TOPIC = "exchange_topic";

    @Bean
    public Queue topicQueue1(){
        Queue queue = new Queue(RabbitmqTopicConfig.QUEUE_TOPIC1);
        return queue;
    }

    @Bean
    public Queue topicQueue2(){
        Queue queue = new Queue(RabbitmqTopicConfig.QUEUE_TOPIC2);
        return queue;
    }


    @Bean
    public TopicExchange topicExchange(){
        TopicExchange topicExchange = new TopicExchange(RabbitmqTopicConfig.EXCHANGE_TOPIC);
        return topicExchange;
    }
	/**
		将队列和交换机进行绑定，并绑定通配符
	*/
    @Bean
    public Binding bindingTopicQueue1(){
        Binding binding = BindingBuilder.bind(topicQueue1()).to(topicExchange()).with("user.#");
        return binding;
    }

    @Bean
    public Binding bindingTopicQueue2(){
        Binding binding = BindingBuilder.bind(topicQueue2()).to(topicExchange()).with("user.*");
        return binding;
    }

}

------------------------------------------------------------------------------------------------------------
   @RestController
public class RabbitController {

    @Autowired
    private MsgSender msgSender;
    /*
   路由key
 */
    @GetMapping(value = "/sendMsg5", produces = "application/json;charset=utf-8")
    public void sendMsg5() {
        msgSender.send5();
    }
}

------------------------------------------------------------------------------------------------------------
@Component
@Slf4j
public class MsgSender {
    @Autowired
    private AmqpTemplate rabbitTemplate;
    public void send5() {
        String msgContent = "Hello World ~";
        log.info("(通配符)生产者发送消息 : " + msgContent);
        this.rabbitTemplate.convertAndSend(RabbitmqTopicConfig.EXCHANGE_TOPIC,"user.a", msgContent);
        this.rabbitTemplate.convertAndSend(RabbitmqTopicConfig.EXCHANGE_TOPIC,"user.a.b", msgContent);

    }
}
------------------------------------------------------------------------------------------------------------
@Slf4j
@Component
public class MsgReceiver {
        @RabbitListener(
            bindings=@QueueBinding(
                    value = @Queue(RabbitmqTopicConfig.QUEUE_TOPIC1),
                    key={"user.#"},
                    exchange = @Exchange(name = RabbitmqTopicConfig.EXCHANGE_TOPIC,type = ExchangeTypes.TOPIC)
            )
    )
    public void process10(String msg) {
        log.info("消费者10接收消息: " + msg);
    }

    @RabbitListener(
            bindings=@QueueBinding(
                    value = @Queue(RabbitmqTopicConfig.QUEUE_TOPIC2),
                    key={"user.*"},
                    exchange = @Exchange(name = RabbitmqTopicConfig.EXCHANGE_TOPIC,type = ExchangeTypes.TOPIC)
            )
    )
    public void process11(String msg) {
        log.info("消费者11接收消息: " + msg);
    }
}
```

## 4.保证消息的可靠性

`不可靠性的原因：`

​	`一、生产端：生产者将消息发送到交换机失败交换机将消息发送到队列失败`

​	`二、MQ:rabbitmq宕机`

​	`三、消费端：消费者获取消息后处理消息失败`

### 1.生产者端：confirm和return模式

1)confirm：【生产者】将【消息】发送到【交换机】

```
触发时机：
	【生产者】将【消息】发送到【交换机】成功或失败，都触发
```

配置文件配置如下：

```xml
publisher-confirm-type: correlated
```

2）return：【交换机】将【消息】发送到【队列】

```
触发时机：
	【交换机】将【消息】发送到【队列】失败时，触发
```

配置文件配置如下：

```xml
publisher-returns: true 
```



```java
    @Autowired
    private RabbitTemplate amqpTemplate;

    /**
     * 当消息发送到交换机，confirm方法会被调用
     * 当交换机将消息发送到队列，且发送到队列失败时，reurn方法会被调用。
     * 四种情况：
     * ①消息推送到server，但是在server里找不到交换机 ：推送消息找不到对应的交换器，会执行ConfirmCallback方法
     * ②消息推送到server，找到交换机了，但是没找到队列：找不到队列，执行ReturnsCallback方法
     * ③消息推送到sever，交换机和队列啥都没找到：找不到交换机，会执行ConfirmCallback方法
     * ④消息推送成功
     */
    @GetMapping(value = "/test01")
    public void test01(){
        amqpTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            @Override
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                if(ack){
                    System.out.println("消息->交换机【成功】");
                }else {
                    System.out.println("消息->交换机【失败】，原因：" + cause);
                }
            }
        });
        amqpTemplate.setReturnsCallback(new RabbitTemplate.ReturnsCallback() {
            @Override
            public void returnedMessage(ReturnedMessage returnedMessage) {
                System.out.println("交换机推送消息->队列【失败】，原因：" + returnedMessage.getMessage());
            }
        });
        //①消息推送到server，但是在server里找不到交换机 ：推送消息找不到对应的交换器，会执行ConfirmCallback方法
        //amqpTemplate.convertAndSend("ccc","zl","我是一条消息");
        //②消息推送到server，找到交换机了，但是没找到队列：找不到队列，执行ReturnsCallback方法
        //amqpTemplate.convertAndSend("ccc","我是一条消息");
        //③消息推送到sever，交换机和队列啥都没找到：找不到交换机，会执行ConfirmCallback方法
        //amqpTemplate.convertAndSend("ccc","ccc","我是一条消息");
        //④消息推送成功
        amqpTemplate.convertAndSend("zl","我是一条消息");
    }
```

https://www.cnblogs.com/liweixml/p/14662355.html

###   2.消费者端：手动ack

配置信息如下：

```
listener:
   direct:
      acknowledge-mode: manual #手动签收
```

```java
@Bean("ack")
    public Queue queue(){
        return new Queue("ack");
    }

    @RabbitListener(queues = {"ack"})
    public void msgResolve(String msg, Message message, Channel channel) throws IOException {
        System.out.println("消息：【" + msg + "】正在被处理");
        //确认拒绝消息
        channel.basicNack(deliveryTag,false,true);
        //确认消息被消费
        //channel.basicAck(deliveryTag,false);
    }
```

### 3.rabbitmq实现持久化

## 6.避免消息被重复消费

幂等性、消息去重，手动ack....

https://blog.csdn.net/weixin_42069404/article/details/140898901



## 7.保证消息顺序消费

消息按顺序发送到同一个队列，且这个队列只有一个消费者


https://blog.csdn.net/weixin_45498465/article/details/105708875
