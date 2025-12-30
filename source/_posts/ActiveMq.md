---
uuid: 0e16785d-5970-4e4b-8ce8-1f441b566d21
title: ActiveMq
date: 2021-05-07 11:24
tags: mq
categories: 
---

<!--more-->

# ActiveMq

## 安装

### 直接安装方式

1.  选择对应的版本下载 <http://activemq.apache.org/components/classic/download/>
2.  配置外网访问，需要将conf/jetty.xml 的ip修改为0.0.0.0
3.  到bin目录下启动 ./active start

## 配置

pom引入坐标

```xml
        <dependency>
            <groupId>org.apache.activemq</groupId>
            <artifactId>activemq-all</artifactId>
            <version>5.16.0</version>
        </dependency>
```

## 使用

```java
public class ActiveMain {
    private static final String ACTIVE_URL = "tcp://139.199.106.104:61616";
    private static final String QUEUE_NAME = "queue01";
    public static void main(String[] args) throws JMSException, IOException {
        //1、创建工厂,默认密码是admin/admin
        ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(ACTIVE_URL);
        //2、通过工厂获取connection
        Connection connection = connectionFactory.createConnection();
        connection.start();
        //3、获取session 第一个参数：是否开启事务   第二参数：签收
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        //4、创建目的地  队列或则主题
        Queue queue = session.createQueue(QUEUE_NAME);

        //5、消费者监听消息
        MessageConsumer consumer = session.createConsumer(queue);
        consumer.setMessageListener((message -> {
            TextMessage textMessage = (TextMessage) message;
            try {
                System.out.println(textMessage.getText());
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }));

        //5、创建生成者  发送消息
        MessageProducer producer = session.createProducer(queue);
        TextMessage textMessage = session.createTextMessage("this is a second message");
        producer.send(textMessage);



        //最后、发送结束 释放资源
        producer.close();
        session.close();
        connection.close();
    }
}
```

### queue

消息生产者生产消息发送到queue中，然后消息消费者从queue中取出并且消费消息。  
消息被消费以后，queue中不再有存储，所以消息消费者不可能消费到已经被消费的消息。  
Queue支持存在多个消费者，但是对一个消息而言，只会有一个消费者可以消费、其它的则不能消费此消息了。  
当消费者不存在时，消息会一直保存，直到有消费消费

### topic

消息生产者（发布）将消息发布到topic中，同时有多个消息消费者（订阅）消费该消息。  
和点对点方式不同，发布到topic的消息会被所有订阅者消费。  
当生产者发布消息，不管是否有消费者。都不会保存消息

## 消费者

### receive方法接受消息

```java
//在接收到消息前会一直阻塞
Message receive = consumer.receive();
//在接收到消息前会一直阻塞1s，还没有收到消息就结束等待
Message receive = consumer.receive(1000);
```

### 消息监听器

```java

        consumer.setMessageListener((message -> {
            TextMessage textMessage = (TextMessage) message;
            try {
                System.out.println(textMessage.getText());
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }));

```

## 整合spring boot

### 依赖配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-activemq</artifactId>
</dependency>
<!--消息队列连接池-->
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-pool</artifactId>
    <version>5.15.0</version>
</dependency>
```

### 属性配置

```properties
# failover:(tcp://localhost:61616,tcp://localhost:61617)
# tcp://localhost:61616
spring.activemq.broker-url=tcp://localhost:61616
#true 表示使用内置的MQ，false则连接服务器
spring.activemq.in-memory=false
#true表示使用连接池；false时，每发送一条数据创建一个连接
spring.activemq.pool.enabled=true
#连接池最大连接数
spring.activemq.pool.max-connections=10
#空闲的连接过期时间，默认为30秒
spring.activemq.pool.idle-timeout=30000
#强制的连接过期时间，与idleTimeout的区别在于：idleTimeout是在连接空闲一段时间失效，而expiryTimeout不管当前连接的情况，只要达到指定时间就失效。默认为0，never
spring.activemq.pool.expiry-timeout=0
```

### 启动类\(provider\)，consumer同样

```java
@SpringBootApplication
@EnableJms //启动消息队列
public class ProviderApplication {
	public static void main(String[] args) {
		SpringApplication.run(ProviderApplication.class, args);
	}
}
```

### 生产者

```java
import javax.jms.Queue;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jms.core.JmsMessagingTemplate;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
 
/*
 * @author uv
 * @date 2018/9/15 14:54
 *
 */
@RestController
public class ProviderController {
 
    //注入存放消息的队列，用于下列方法一
    @Autowired
    private Queue queue;
 
    //注入springboot封装的工具类
    @Autowired
    private JmsMessagingTemplate jmsMessagingTemplate;
 
    @RequestMapping("send")
    public void send(String name) {
        //方法一：添加消息到消息队列
        jmsMessagingTemplate.convertAndSend(queue, name);
        //方法二：这种方式不需要手动创建queue，系统会自行创建名为test的队列
        //jmsMessagingTemplate.convertAndSend("test", name);
    }
}

```

### 消费者

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jms.annotation.JmsListener;
import org.springframework.jms.core.JmsMessagingTemplate;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Component;
 
/*
 * @author uv
 * @date 2018/9/15 18:36
 *
 */
@Component
public class ConsumerService {
 
    @Autowired
    private JmsMessagingTemplate jmsMessagingTemplate;
 
    // 使用JmsListener配置消费者监听的队列，其中name是接收到的消息
    @JmsListener(destination = "ActiveMQQueue")
    // SendTo 会将此方法返回的数据, 写入到 OutQueue 中去.
    @SendTo("SQueue")
    public String handleMessage(String name) {
        System.out.println("成功接受Name" + name);
        return "成功接受Name" + name;
    }
}
```