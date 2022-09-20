# MQ

## 一、消息队列分类

![](../pictures/Snipaste_2022-09-18_23-00-59.png)

**<font color=red>消息队列是一种将消息进行转发给多个用户的发布器</font>**

### 1.1 Rabbit MQ

特点：

可用性高，单机吞吐量一般，消息延迟低，消息可靠性高

### 1.2 Active MQ

特点：各方面都一般或者比较差

### 1.3 Rocket MQ

特点：

可用性高，单机吞吐量高，消息延迟低，消息可靠性高

### 1.4 Kafka

特点：

可用性高，单机吞吐量很高，消息延迟低，消息可靠性一般般

## 二、Rabbit MQ

### 2.1 交换器

* 定义：
  只负责消息的转发，如果路由转发失败，消息就会丢失
* 分类
  * FanoutExchange
    广播路由器，会将收到的消息发送给每一个绑定的队列
  * DirectExchange
    直接交换机，通过指定的routingKey将收到的消息发送给绑定队列
  * TopicExchange
    主题交换机，使用方法和直接交换机一致，不过它必须使用多个单词并通过符号 . 进行拼接，然后通过通配符的形式将收到的消息转发给匹配的队列（#代表任意个、*代表一个）

### 2.2 路由键（routingKey）

* 当使用DirectExchange或者TopicExchange时用来作为绑定关系的字符串

### 2.3 消息队列

* 负责从交换机中获取数据然后发给指定的消费者，如果有多个消费者同时从一个队列中获取数据，那么这多个消费者只能轮流获取队列中的部分数据，因为队列中的消息一旦被消费就不会保留。所以如果一个消费者消费了队列中的某个消息之后，其他消费者是无法获取的这个消息的。

### 2.4 消息的可靠性投递

* **<font color=red>确认回调函数</font>**

  ~~~java
  rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
              /**
               * 确认消息是否被交换机接收的回调函数
               * @param correlationData data to correlate publisher confirms.
               * @param b 交换机是否接收到消息
               * @param s 交换机接收消息失败的原因
               */
              @Override
              public void confirm(CorrelationData correlationData, boolean b, String s) {
                  System.out.println("发送消息成功。。。");
                  if (!b) {
                      System.err.println("消息接收失败，失败原因是" + s);
                  } else {
                      System.out.println("消息接收成功。。");
                  }
              }
          });
  ~~~

  **作用**：当生产者发送出消息之后，该函数就会自动执行，通过该函数中的boolean类型的参数就可以直到交换机接收消息是否成功。
  **使用步骤**：

  1. 在配置文件中开启确认回调函数。

     ~~~yml
     spring:
       rabbitmq:
         port: 5672
         username: root
         password: root
         host: 192.168.220.129
         publisher-confirm-type: correlated
     ~~~

  2. 通过rabbitTemplate对象调用回调函数。

* **<font color=red>返回回调函数</font>**

  ~~~java
  rabbitTemplate.setMandatory(true);
          rabbitTemplate.setReturnCallback((message, replyCode, replyText, exchange, routingKey) -> {
              System.out.println("return 返回消息");
              System.out.println("交换机: " + exchange + "接收到消费者的消息: " + new String(message.getBody())
                      + ", 但是未通过指定的路由键：" + routingKey + "发送给消费者的队列，错误码是" +
                      replyCode + ", 对应的错误信息为：" + replyText + "：" + routingKey);
  
          });
  ~~~

  **作用**：当消息从交换机发出之后，用来确定该消息是否被队列接收，<font color=red>如果队列接收到该消息，返回回调函数就不会执行，如果接收失败就会执行返回回调函数中的内容</font>
  **使用步骤**：

  1. 在配置文件中开启返回回调函数。

     ~~~yml
     spring:
       rabbitmq:
         port: 5672
         username: root
         password: root
         host: 192.168.220.129
         publisher-returns: true
     ~~~

  2. 将丢失的消息开启处理模式，默认情况下丢失的消息不会进行处理，只有通过rabbitTemplate对象进行api的调用才能开启处理

     ~~~java
     rabbitTemplate.setMandatory(true);
     ~~~

  3. 设置返回回调函数

  



