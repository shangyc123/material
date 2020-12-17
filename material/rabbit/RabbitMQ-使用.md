# Maven-RabbitMQ使用

## 创建Maven工程

添加依赖

```
<dependency>
     <groupId>com.rabbitmq</groupId>
     <artifactId>amqp-client</artifactId>
     <version>5.5.1</version>
</dependency>
```

## 生产者

```java
package com.qf.rabbitmq;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RabbitProvider {

    //队列名称
    private static final String QUEUE_NAME = "simple_queue";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();

        connectionFactory.setHost("10.0.0.181");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("vh1");
        connectionFactory.setUsername("test1");
        connectionFactory.setPassword("123456");

        Connection connection = connectionFactory.newConnection();

        Channel channel = connection.createChannel();
        //声明队列
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);

        String message = "今天天气好晴朗";
        //发送消息
        channel.basicPublish("",QUEUE_NAME,null,message.getBytes());
        System.out.println("发送消息成功");

    }
}

```

## 消费者

```java
package com.qf.rabbitmq;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RabbitConsumer {

    private static final String QUEUE_NAME = "simple_queue";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();

        connectionFactory.setHost("10.0.0.181");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("vh1");
        connectionFactory.setUsername("test1");
        connectionFactory.setPassword("123456");

        Connection connection = connectionFactory.newConnection();

        Channel channel = connection.createChannel();

        channel.queueDeclare(QUEUE_NAME,false,false,false,null);

        //处理消息
        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println(" [x] Received '" + message + "'");
        };
        //监听对列
        channel.basicConsume(QUEUE_NAME, true, deliverCallback, consumerTag -> { });

        
    }

}

```

### 非Lambda

```java
package com.qf.rabbitmq;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RabbitConsumer {

    private static final String QUEUE_NAME = "simple_queue";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();

        connectionFactory.setHost("10.0.0.181");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("vh1");
        connectionFactory.setUsername("test1");
        connectionFactory.setPassword("123456");

        Connection connection = connectionFactory.newConnection();

        Channel channel = connection.createChannel();

        channel.queueDeclare(QUEUE_NAME,false,false,false,null);

        //处理消息
        Consumer consumer = new DefaultConsumer(channel){
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String message = new String(body,"utf-8");
                System.out.println("接收到的消息："+message);
            }
        };
        //监听队列
        channel.basicConsume(QUEUE_NAME,true,consumer);


    }

}

```



# 消息队列的收发模式

## 简单队列模式

默认情况下，是简单队列模式，两个消费者交替接收消息

## work模式——能者多劳

一次只处理一个消息，待消息处理完后手动回复

### 改变消费者

```java
package com.qf.rabbitmq;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RabbitConsumer1 {

    private static final String QUEUE_NAME = "simple_queue";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();

        connectionFactory.setHost("10.0.0.181");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("vh1");
        connectionFactory.setUsername("test1");
        connectionFactory.setPassword("123456");

        Connection connection = connectionFactory.newConnection();

        Channel channel = connection.createChannel();
        channel.basicQos(1);//一次只处理一个消息,避免消息堆积

        channel.queueDeclare(QUEUE_NAME,false,false,false,null);

        //处理消息
        Consumer consumer = new DefaultConsumer(channel){
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {

                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                String message = new String(body,"utf-8");
                System.out.println("消费者1接收到的消息："+message);
                //手动回复 ：处理完成
                channel.basicAck(envelope.getDeliveryTag(),false);
            }
        };
        //监听队列
        channel.basicConsume(QUEUE_NAME,false,consumer);//不自动回复


    }

}

```

## 发布订阅模式

将消息发送给交换机，交换机绑定多个队列，每个队列都能得到该消息。

### 改变生成者

```java
package com.qf.rabbitmq;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RabbitProvider {

    private static final String QUEUE_NAME = "simple_queue";

    private static final String EXCHANGE_NAME="fanout_exchange";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();

        connectionFactory.setHost("10.0.0.181");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("vh1");
        connectionFactory.setUsername("test1");
        connectionFactory.setPassword("123456");

        Connection connection = connectionFactory.newConnection();

        Channel channel = connection.createChannel();

        //声明交换机
        channel.exchangeDeclare(EXCHANGE_NAME,"fanout");

        for (int i = 1; i <= 100; i++) {
            String message = "message"+i;
//            channel.basicPublish("",QUEUE_NAME,null,message.getBytes());
            channel.basicPublish(EXCHANGE_NAME,"",null,message.getBytes());
            System.out.println("发送第"+i+"个消息成功");
        }
        
    }

}
```

### 改变消费者

```java
package com.qf.rabbitmq;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RabbitConsumer1 {

    //定义队列名字
    private static final String QUEUE_NAME = "fanout_queue1";
    //定义交换机名字
    private static final String EXCHANGE_NAME="fanout_exchange";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();

        connectionFactory.setHost("10.0.0.181");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("vh1");
        connectionFactory.setUsername("test1");
        connectionFactory.setPassword("123456");

        Connection connection = connectionFactory.newConnection();

        Channel channel = connection.createChannel();
        channel.basicQos(1);//一次只处理一个消息,避免消息堆积
        //声明队列
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        //将队列绑定在交换机上
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"");

        //处理消息
        Consumer consumer = new DefaultConsumer(channel){
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {

                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                String message = new String(body,"utf-8");
                System.out.println("消费者1接收到的消息："+message);
                //手动回复 ：处理完成
                channel.basicAck(envelope.getDeliveryTag(),false);
            }
        };
        //监听队列
        channel.basicConsume(QUEUE_NAME,false,consumer);//不自动回复


    }

}

```



## 路由模式

修改交换机类型为direct

### 生产者

```java
package com.qf.rabbitmq;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RabbitProvider {


    private static final String EXCHANGE_NAME="direct_exchange";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();

        connectionFactory.setHost("10.0.0.181");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("vh1");
        connectionFactory.setUsername("test1");
        connectionFactory.setPassword("123456");

        Connection connection = connectionFactory.newConnection();

        Channel channel = connection.createChannel();

        //声明交换机-- 改变交换机类型为“direct”
        channel.exchangeDeclare(EXCHANGE_NAME,"direct");
        //指明路由key
        channel.basicPublish(EXCHANGE_NAME,"nba",null,"NBA Message".getBytes());
        channel.basicPublish(EXCHANGE_NAME,"cba",null,"cba Message".getBytes());
        System.out.println("发送消息成功");
    }

}

```

### 消费者

```java
package com.qf.rabbitmq;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RabbitConsumer1 {

    //定义队列名字
    private static final String QUEUE_NAME = "direct_queue1";
    //定义交换机名字
    private static final String EXCHANGE_NAME="direct_exchange";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();

        connectionFactory.setHost("10.0.0.181");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("vh1");
        connectionFactory.setUsername("test1");
        connectionFactory.setPassword("123456");

        Connection connection = connectionFactory.newConnection();

        Channel channel = connection.createChannel();
        channel.basicQos(1);//一次只处理一个消息,避免消息堆积
        //声明队列
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        //将队列绑定在交换机上,并指明路由key
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"nba");
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"cba");
        //处理消息
        Consumer consumer = new DefaultConsumer(channel){
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {

                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                String message = new String(body,"utf-8");
                System.out.println("消费者1接收到的消息："+message);
                //手动回复 ：处理完成
                channel.basicAck(envelope.getDeliveryTag(),false);
            }
        };
        //监听队列
        channel.basicConsume(QUEUE_NAME,false,consumer);//不自动回复
    }
}

```

##通配符模式

修改交换机类型为topic

### 生产者

```java
package com.qf.rabbitmq;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RabbitProvider {


    private static final String EXCHANGE_NAME="topic_exchange";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();

        connectionFactory.setHost("10.0.0.181");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("vh1");
        connectionFactory.setUsername("test1");
        connectionFactory.setPassword("123456");

        Connection connection = connectionFactory.newConnection();

        Channel channel = connection.createChannel();

        //声明交换机-- 改变交换机类型为“direct”
        channel.exchangeDeclare(EXCHANGE_NAME,"topic");
        //指明路由key
        channel.basicPublish(EXCHANGE_NAME,"product.add",null,"添加商品".getBytes());
        channel.basicPublish(EXCHANGE_NAME,"product.del",null,"删除商品".getBytes());
        channel.basicPublish(EXCHANGE_NAME,"product.del.other",null,"删除商品!!".getBytes());
        System.out.println("发送消息成功");


    }

}
```

### 消费者1

```java
package com.qf.rabbitmq;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RabbitConsumer1 {

    //定义队列名字
    private static final String QUEUE_NAME = "topic_queue1";
    //定义交换机名字
    private static final String EXCHANGE_NAME="topic_exchange";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();

        connectionFactory.setHost("10.0.0.181");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("vh1");
        connectionFactory.setUsername("test1");
        connectionFactory.setPassword("123456");

        Connection connection = connectionFactory.newConnection();

        Channel channel = connection.createChannel();
        channel.basicQos(1);//一次只处理一个消息,避免消息堆积
        //声明队列
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        //将队列绑定在交换机上
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"product.*");
        //处理消息
        Consumer consumer = new DefaultConsumer(channel){
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {

                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                String message = new String(body,"utf-8");
                System.out.println("消费者1接收到的消息："+message);
                //手动回复 ：处理完成
                channel.basicAck(envelope.getDeliveryTag(),false);
            }
        };
        //监听队列
        channel.basicConsume(QUEUE_NAME,false,consumer);//不自动回复


    }

}

```

### 消费者2

```java
package com.qf.rabbitmq;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RabbitConsumer2 {

    private static final String QUEUE_NAME = "topic_queue2";
    private static final String EXCHANGE_NAME="topic_exchange";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();

        connectionFactory.setHost("10.0.0.181");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("vh1");
        connectionFactory.setUsername("test1");
        connectionFactory.setPassword("123456");

        Connection connection = connectionFactory.newConnection();

        Channel channel = connection.createChannel();
        channel.basicQos(1);//一次只处理一个消息,避免消息堆积

        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"product.#");

        //处理消息
        Consumer consumer = new DefaultConsumer(channel){
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String message = new String(body,"utf-8");
                System.out.println("消费者2接收到的消息："+message);
                //手动回复 ：处理完成
                channel.basicAck(envelope.getDeliveryTag(),false);
            }
        };
        //监听队列
        channel.basicConsume(QUEUE_NAME,false,consumer);


    }

}

```



# SpringBoot-RabbitMQ 使用

## 生产者

创建一个名为 `spring-boot-amqp-provider` 的生产者项目

### application.yml

```yaml
spring:
  application:
    name: spring-boot-amqp
  rabbitmq:
    host: 192.168.75.133
    port: 5672
    username: rabbit
    password: 123456
```

### 创建队列配置

```java
package com.duo.spring.boot.amqp.config;

import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * 队列配置
 * <p>Title: RabbitMQConfiguration</p>
 * <p>Description: </p>
 */
@Configuration
public class RabbitMQConfiguration {
    @Bean
    public Queue queue() {
        return new Queue("helloRabbit");
    }
}
```

### 创建消息提供者

```java
package com.duo.spring.boot.amqp.provider;

import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.Date;

/**
 * 提供者
 * <p>Title: HelloRabbitProvider</p>
 * <p>Description: </p>
 */
@Component
public class HelloRabbitProvider {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void send() {
        String context = "hello" + new Date();
        System.out.println("Provider: " + context);
        amqpTemplate.convertAndSend("helloRabbit", context);
    }
}
```

### 创建测试用例

```java
package com.duo.spring.boot.amqp.test;

import com.duo.spring.boot.amqp.Application;
import com.duo.spring.boot.amqp.provider.HelloRabbitProvider;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
public class AmqpTest {
    @Autowired
    private HelloRabbitProvider helloRabbitProvider;

    @Test
    public void testSender() {
        for (int i = 0; i < 10; i++) {
            helloRabbitProvider.send();
        }
    }
}
```

## 消费者

创建一个名为 `spring-boot-amqp-consumer` 的消费者项目

### application.yml

```yaml
spring:
  application:
    name: spring-boot-amqp-consumer
  rabbitmq:
    host: 192.168.75.133
    port: 5672
    username: rabbit
    password: 123456
```

### 创建消息消费者

```java
package com.duo.spring.boot.amqp.consumer;

import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
public class HelloRabbitConsumer {
    @RabbitListener(queues = "helloRabbit")
    public void process(String message) {
        System.out.println("Consumer: " + message);
    }
}
```

# 交换机模式

## 编辑配置文件

```java
package com.qf.rabbitmqdemo.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.FanoutExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitConfiguration {

    @Bean
    public Queue queue(){
        return new Queue("simple_queue");
    }


    @Bean
    public FanoutExchange getFanoutExchange(){
        return  new FanoutExchange("fanout_exchange_new");
    }

    @Bean
    public Binding getBinding(Queue getQueue,FanoutExchange getFanoutExchange){
        return BindingBuilder.bind(getQueue).to(getFanoutExchange);
    }


}

```



## 生产者

```java
package com.qf.rabbitmqdemo;

import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class ExchangeProvider {

    @Autowired
    private RabbitTemplate rabbitTemplate;


    public void send(){
        rabbitTemplate.convertAndSend("fanout_exchange_new","","交换机模式消息");
    }


}

```

# 通配符模式

## 修改配置文件

```java
    @Bean
    public TopicExchange getTopicExchange(){
        return new TopicExchange("topic_exchange_new");
    }

    @Bean
    public Binding getBinding2(Queue getQueue,TopicExchange getTopicExchange){
        return BindingBuilder.bind(getQueue).to(getTopicExchange).with("product.*");
    }

```

## 生产者

```java
   public void send(){
        rabbitTemplate.convertAndSend("topic_exchange_new","product.add","topic模式消息");
    }
```



