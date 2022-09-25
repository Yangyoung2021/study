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
  负责消息的接收和转发，如果路由（routingKey不匹配）转发失败，消息就会丢失（fanout交换机除外）
* 分类
  * FanoutExchange
    广播路由器，会将收到的消息发送给每一个绑定的队列
  * DirectExchange
    直接交换机，通过指定的routingKey将收到的消息发送给绑定队列
  * TopicExchange
    主题交换机，使用方法和直接交换机一致，不过它必须使用多个单词并通过符号 . 进行拼接，然后通过通配符的形式将收到的消息转发给匹配的队列（#代表任意个、*代表一个）

### 2.2 路由键（routingKey）

* 用来作为消息发送端（生产者）和接收消息的交换机进行绑定关系的字符串，即指定哪些交换机能接收到消息

### 2.3 消息队列

* 负责从交换机中获取数据然后发给指定的消费者，如果有多个消费者同时从一个队列中获取数据，那么这多个消费者只能按照声明的顺序轮流获取队列中的数据，因为队列中的消息一旦被消费就不会保留。所以如果一个消费者消费了队列中的某个消息之后，其他消费者是无法获取的这个消息的。

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
         System.out.println("交换机: " + exchange + "接收到消费者的消息: " + new String( message.getBody()) + ", 但是未通过指定的路由键：" + routingKey + "发送给消费者的队列，错误码是" + replyCode + ", 对应的错误信息为：" + replyText + "：" + routingKey);
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

### 2.5 消费者端的确认机制

* <font color=red size=4>消费端使用Ack机制</font>进行确认消息的签收，签收方式有3种
  
  * none 自动签收 
    默认方式，消费端接收消息之后自动进行确认，确认后Broker就会将消息进行销毁。
  * manual 手动签收 
    首先需要在程序中注册一个<font color=red>监听器容器</font>，接着将连接的参数通过<font color=red>连接工厂</font>设置到监听器容器，然后通过<font color=red>监听器容器</font>设置确定机制为手动确认。最后将设置好接收消息的队列和消息监听器进行绑定，就能实现手动签收。
    可以通过自己配置一个消息监听器（实现MessageListener接口），但是想要进行自动签收（通过Channel对象）就需要实现MessageListener的一个子接口ChannelAwareMessageListener然后实现它的onMessage方法，通过其中的Channel对象调用basicAck方法进行手动签收。
* auto 根据签收异常的不同进行不同的逻辑操作
  
* 手动监听实现示例1（自定义消息监听器）

  * 配置连接参数

    ~~~yml
    spring:
      rabbitmq:
        port: 5672
        username: root
        password: root
        host: 192.168.220.129
        listener:
          direct:
            prefetch: 1
        virtual-host: "/"
    ~~~

  * 注册连接工厂

    ~~~java
    /**
     * 注册连接工厂用来连接rabbitMQ主机
     * @return 监听工厂
     */
    @Bean
    public ConnectionFactory connectionFactory(RabbitProperties properties) {
        CachingConnectionFactory factory = new CachingConnectionFactory();
        factory.setUsername(properties.getUsername());
        factory.setPassword(properties.getPassword());
        factory.setHost(properties.getHost());
        factory.setPort(properties.getPort());
        factory.setVirtualHost(properties.getVirtualHost());
        return factory;
    }
    ~~~

  * 配置消费队列

    ~~~java
    @Bean
    public FanoutExchange fanoutExchange1() {
        return new FanoutExchange("com.yang");
    }
    ~~~

  * 自定义消息监听器

    ```java
    /**
     * 设置通用的消息签收机制，默认自动签收
     */
    @Component
    public class MessageAck implements ChannelAwareMessageListener {
        
        @Override
        public void onMessage(Message message, Channel channel) throws Exception {
            // 获取message标签
            long deliveryTag = message.getMessageProperties().getDeliveryTag();
            Thread.sleep(1000L);
            System.out.println("消费端进行签收操作");
            try {
                // 获取消息
                System.out.println(new String(message.getBody()));
                // 第一个参数是消息的标签，第二个参数表示是否全部签收
                channel.basicAck(deliveryTag, true);
                System.out.println("消息签收成功。。。");
            } catch (Exception e) {
                // 第一个参数是获取消息的标签，第二个是确定全部签收，第三个签收失败是否将让broker重新发送消息
                channel.basicNack(deliveryTag, true, true);
            }
        }
    }
    ```

    

  * 将消费队列、连接工厂、自定义的消息监听器与消息监听器容器进行绑定并开启手动确认模式

    ~~~java
    /**
     * 注册消息监听器容器，开启手动确认消息机制
     * @return 息监听器容器
     */
    @Bean
    public MessageListenerContainer messageListenerContainer(
          ConnectionFactory factory, MessageListener messageAck) {
        SimpleMessageListenerContainer container = new 							                                  SimpleMessageListenerContainer();
        container.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        container.setConnectionFactory(factory);
        container.setQueueNames("simple.queue");
        container.setMessageListener(messageAck);
        return container;
    }
    ~~~

* 手动监听实现示例2（使用RabbitTemplate消息监听器）

  ~~~java
  /**
   * 注册消息监听器容器，开启手动确认消息机制
   * @return 息监听器容器
   */
  @Bean
  public DirectRabbitListenerContainerFactory messageListenerContainer(
        ConnectionFactory factory) {
      DirectRabbitListenerContainerFactory container = new 							                                  DirectRabbitListenerContainerFactory();
      container.setAcknowledgeMode(AcknowledgeMode.MANUAL);
      container.setConnectionFactory(factory);
      return container;
  }
  
  // 消费消息
  @RabbitListener(containerFactory = "messageListenerContainer",
                  bindings = @QueueBinding(
                      value = @Queue(name = "yang.queue1"),
                      exchange = @Exchange(name = "com.yang.direct")
      ))
  public void getFanoutMessage1(String message, 
                                @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag, 	                                 Channel channel ) throws IOException {
      try {
          System.out.println("yang.queue1收到消息" + message);
          channel.basicAck(deliveryTag, true);
      } catch (Exception e) {
          channel.basicNack(deliveryTag, true, true);
      }
  }
  ~~~


### 2.6 消费端限流(手动确认)QOS

* 消费端可以通过监听器容器进行限流的操作，每次只从broker中获取设置好的prefetch的数量，只有等消费端手动确认消息接收过后才能进行下一次的获取，从而达到保护消费端的目的

* 实现步骤

  * 在配置监听器容器的时候调用setPrefetchCount方法进行设置

    ~~~java
    // Tell the broker how many messages to send to each consumer in a single request.
    // Often this can be set quite high to improve throughput.
    container.setPrefetchCount(1000);
    ~~~

  * 如果消费端没有设置当前消息为签收状态，那消息就不会在broker里被销毁，当出现异常还能设置broker进行重发

### 2.7 TTL（Time To Live）

* 定义
  过期时间设置，可以通过给队列或者具体的消息进行设置，如果队列中的消息在定义好的时间内没有被取走，也会在队列中被删除，<font size=4 color=red>需要注意的是，消息过期之后不会立即从队列中移除，而是当消息到达队列尾部的时候进行判断消息是否过期，如果过期再进行移除</font>。
  
* 实现案例（整个队列的过期）

  ~~~java
  // 全注解配置
  @RabbitListener(bindings = @QueueBinding(
      exchange = @Exchange(
          name = "yang.direct",
          type = ExchangeTypes.DIRECT
              ),
      value = @Queue(
          name = "yang.direct.queue3",
          arguments = @Argument(
              name = "x-message-ttl",
              value = "10000",
              // 此处一定要定义为整型，默认为字符串
              type = "java.lang.Integer"
          )
      ),
      key = "direct3"
  
  ))
  public void getDirectMsg2(String msg) {
      log.info("direct3 get {}",msg);
  }
  // 显式bean配置
  @Bean
  public Queue topicQueue1() {
      return QueueBuilder.durable("yang.topic.queue1").withArgument("x-message-ttl", 10000).build();
  }
  
  @Bean
  public TopicExchange topicExchange1() {
      return new TopicExchange("yang.topic");
  }
  
  @Bean
  public Binding bindingNode1(Queue topicQueue1, TopicExchange topicExchange) {
      return BindingBuilder.bind(topicQueue1).to(topicExchange).with("topic1");
  }
  ~~~

* 单个消息过期

  ~~~java
  // 创建一个消息后处理器对单个消息进行过期设置
  MessagePostProcessor messagePostProcessor = msg -> {
              msg.getMessageProperties().setExpiration("1000");
              return msg;
          };
  // 在发送消息的api最后添加一个消息后处理器进行消息过期设置
  rabbitTemplate.convertAndSend("yang.topic", "topic1",
                                "topic msg", messagePostProcessor);
  ~~~

### 2.8 死信队列

* Dead Message定义
  1. 接收消息的队列满了之后接收的消息会成为死信
  2. 消息被消费者拒绝签收之后（basicNack/basicReject），并且将消息重回队列设置为false之后，该消息成为死信
  3. 当某个队列中的消息的ttl到期之后会成为死信

* 队列绑定死信交换机
  1. 给队列设置x-dead-letter-exchange（设置死信交换机名称）
  2. 给队列设置x-dead-letter-routing-key（设置死信交换机和死信队列的路由键）

* 流程
  ![](../pictures/Snipaste_2022-09-25_19-36-36.png)

* 配置bean实现

  ~~~java
  /**
   * 需要注意，如果配置的死信队列和交换机是direct类型的，那么正常交换机的路由键（routingKey）和死信
   * 队列的路由键以及配置x-dead-letter-routing-key相同，否则无法保证死信队列接收到消息
   */
  
  // 配置正常交换机
  @Bean
  public DirectExchange deadMsgDirectTest() {
      return new DirectExchange("test.dead.direct");
  }
  // 配置死信交换机
  @Bean
  public DirectExchange deadMsgDirect() {
      return new DirectExchange("dead.msg.direct");
  }
  // 配置正常队列
  @Bean
  public Queue deadMsgQueueTest() {
      Map<String, Object> map = new HashMap<>();
      map.put("x-message-ttl", 10000);
      map.put("x-max-length", 5);
      map.put("x-dead-letter-exchange", "dead.msg.direct");
      map.put("x-dead-letter-routing-key", "dead.test.abc");
      return QueueBuilder.durable("test.dead.queue").withArguments(map).build();
  }
  // 配置死信队列
  @Bean
  public Queue deadMsgQueue() {
      Map<String, Object> map = new HashMap<>();
      map.put("x-message-ttl", 10000);
      return QueueBuilder.durable("dead.msg.config.queue").withArguments(map).build();
  }
  // 绑定正常队列和交换机
  @Bean
  public Binding bindingNode2(Queue deadMsgQueueTest, DirectExchange deadMsgDirectTest) {
      return BindingBuilder.bind(deadMsgQueueTest).to(deadMsgDirectTest).with("dead.test.abc");
  }
  // 绑定死信队列和交换机
  @Bean
  public Binding bindingNode3(Queue deadMsgQueue, DirectExchange deadMsgDirect) {
      return BindingBuilder.bind(deadMsgQueue).to(deadMsgDirect).with("dead.test.abc");
  }
  ~~~

* @RebbitListener注解实现

  ~~~java
  // 配置正常队列和交换机
  @RabbitListener(
      ackMode = "MANUAL",
      bindings = @QueueBinding(
          exchange = @Exchange(
              name = "dead.test.exchange",
              type = ExchangeTypes.TOPIC
                      ),
          value = @Queue(
              name = "dead.test.queue",
              arguments = {@Argument(
                  name = "x-max-length",
                  value = "5",
                  type = "java.lang.Integer"
              ), @Argument(
                  name = "x-message-ttl",
                  value = "10000",
                  type = "java.lang.Integer"
              ), @Argument(
                  name = "x-dead-letter-exchange",
                  value = "dead.msg.exchange"
              ), @Argument(
                  name = "x-dead-letter-routing-key",
                  value = "dead.msg.#"
              )}
          ),
          key = "dead.test.#"
      ))
  public void getDeadTestMsg(String msg, @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag, Channel channel) throws IOException {
      log.info("dead test get {}", msg);
      channel.basicNack(deliveryTag, true, false);
  }
  // 配置死信队列和交换机
  @RabbitListener(bindings = @QueueBinding(
      exchange = @Exchange(
          name = "dead.msg.exchange",
          type = ExchangeTypes.TOPIC
              ),
      value = @Queue(
          name = "dead.msg.queue",
          arguments = @Argument(
              name = "x-message-ttl",
              value = "10000",
              type = "java.lang.Integer"
          )
      ),
      key = "dead.#"
  ))
  public void getDeadMsg(String msg) {
      log.info("dead queue get {}", msg);
  }
  ~~~

  