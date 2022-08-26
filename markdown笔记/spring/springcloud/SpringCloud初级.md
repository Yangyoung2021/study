# SpringCloud初级

## 1、服务的注册和配置

### 1.1 服务的注册

#### **1.1.1 Eureka服务注册中心**

* 依赖

  * 服务端依赖

    ~~~xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
    ~~~

  * 客户端依赖

    ~~~xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    ~~~

* yml配置文件

  * 服务器端

    ~~~yml
    # 服务运行端口号
    server:
      port: 10086
    
    # 服务名称
    spring:
      application:
        name: eurekaserver
        
    # 将自己注册到eureka服务器中（为了集群中数据同步）
    eureka:
      client:
        fetch-registry: false # 默认不拉取注册信息，防止启动报错
        service-url: http://localhost:10086/eureka #作为客户端访问服务器端的地址
    ~~~

  * 客户端（服务消费者）

    ~~~yml
    # eureka 注册
    eureka:
      client:
        service-url:
          defaultZone: http://localhost:10086/eureka
    # 注册中心中每个实例都需要一个名字，所以必须给每个服务创建一个名字
    spring:
      application:
        name: orderservice
    # 配置userService的负载均衡规则
    userService:
      ribbon:
        NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule # 负载均衡规则实现，默认原始轮询
    
    # 配置ribbon的加载规则
    ribbon:
      eager-load:
        clients: "userService"
        enabled: true
    ~~~

* 编码

  * 服务器端

    ~~~java
    package com.yang;
    
    
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;
    
    @SpringBootApplication
    // 需要在启动类上开启Eureka的自动装配
    @EnableEurekaServer
    public class EurekaApplication {
    
        public static void main(String[] args) {
            SpringApplication.run(EurekaApplication.class, args);
        }
        
    }
    
    ~~~

  * 客户端（服务消费者）

    ~~~java
    package cn.itcast.order;
    
    
    import com.netflix.loadbalancer.IRule;
    import com.netflix.loadbalancer.RandomRule;
    import org.mybatis.spring.annotation.MapperScan;
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    import org.springframework.cloud.client.loadbalancer.LoadBalanced;
    import org.springframework.context.annotation.Bean;
    import org.springframework.web.client.RestTemplate;
    
    @MapperScan("cn.itcast.order.mapper")
    @SpringBootApplication
    public class OrderApplication {
    
        public static void main(String[] args) {
            SpringApplication.run(OrderApplication.class, args);
        }
    
    	/**
    	 * 开启负载均衡
    	 */
        @Bean
        @LoadBalanced
        public RestTemplate restTemplate() {
            return new RestTemplate();
        }
    
        /**
         * 这样配置的方案是基于全局的配置，就是说不管访问哪个服务都是随机的，具体的服务配置在yml		 * 文件中可以配置
         * @return 负载均衡访问规则实现对象
         */
        @Bean
        public IRule randomRule() {
            return new RandomRule();
        }
    
    }
    ~~~

#### 1.1.2 **Nacos服务注册中心**

* 依赖
  父工程依赖

  ~~~xml
  <!-- nacos的管理依赖 -->
  <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-alibaba-dependencies</artifactId>
      <version>2.2.5.RELEASE</version>
      <type>pom</type>
      <scope>import</scope>
  </dependency>
  ~~~

  子工程

  ~~~xml
  <!--nacos发现依赖-->
  <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
  </dependency>
  ~~~

* yml配置文件

  * 服务提供者

    ~~~yml
    cloud:
      nacos:
        server-addr: localhost:8848 # nacos 地址
        discovery:
          cluster-name: HZ # 集群名称
          namespace: d386bb28-5654-4380-af25-b4752de8e370 # 命名空间，环境隔离
    ~~~

  * 服务消费者

    ```yml
    cloud:
      nacos:
        server-addr: localhost:8848 # nacos 地址
        discovery:
          cluster-name: HZ # 配置集群名称
          namespace: d386bb28-5654-4380-af25-b4752de8e370
          ephemeral: false # 设置为非临时实例
    # 配置userService的负载均衡规则
    userService:
      ribbon:
        NFLoadBalancerRuleClassName: com.alibaba.cloud.nacos.ribbon.NacosRule # Nacos特有的负载均衡规则实现
    ```

* Nacos配置
  nacos启动命令

  ~~~powershell
  startup.cmd -m standalone
  ~~~

  浏览器访问 localhost:8848/nacos

### 1.2 Nacos服务配置中心

* 依赖

  * 父工程

    ~~~xml
    <!-- nacos的管理依赖 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-alibaba-dependencies</artifactId>
        <version>2.2.5.RELEASE</version>
        <type>pom</type>
        <scope>import</scope>
    </dependency>
    ~~~

  * 子工程

    ~~~xml
    <!-- nacos的配置依赖 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    </dependency>
    ~~~

* yml配置文件
  由于需要在启动服务之前将配置中心的配置项拉取，所以需要创建一个比yml配置文件优先级高的配置文件，所以需要另外加一个bootstrap.yml文件用来与配置中心的ID匹配。

  ~~~yml
  spring:
    application:
      name: userService # 服务名称
    profiles:
      active: dev # 当前环境
    cloud:
      nacos:
        server-addr: localhost:8848 # nacos地址
        config:
          file-extension: yaml # 配置文件类型
  
  ~~~

  * 配置中心
    ![1659970148810](C:\Users\19816\AppData\Roaming\Typora\typora-user-images\1659970148810.png)

* 后台获取配置信息的两种方式

  * 方式一
    使用@Value(${target.value})注解直接获取相对应的配置属性，需要注意，想要达到热更新的效果需要在<font color="red">当前的Controller接口上添加一个@RefreshScope注解</font>

    ~~~java
    package cn.itcast.user.web;
    
    import cn.itcast.user.pojo.User;
    import cn.itcast.user.service.UserService;
    import lombok.extern.slf4j.Slf4j;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.beans.factory.annotation.Value;
    import org.springframework.cloud.context.config.annotation.RefreshScope;
    import org.springframework.web.bind.annotation.*;
    
    import java.time.LocalDateTime;
    import java.time.format.DateTimeFormatter;
    
    @Slf4j
    @RestController
    @RequestMapping("/user")
    @RefreshScope
    public class UserController {
    
        @Autowired
        private UserService userService;
    
    
        @Value("${pattern.dateformat}")
        private String dateformat;
    
        /**
         * 路径： /user/110
         *
         * @param id 用户id
         * @return 用户
         */
        @GetMapping("/{id}")
        public User queryById(@PathVariable("id") Long id) {
            return userService.queryById(id);
        }
    
        @GetMapping("/now")
        public String now() {
            return LocalDateTime.now().format(DateTimeFormatter.ofPattern(dateformat));
        }
    }
    
    ~~~

  * 方式二
    使用@ConfigurationProperties注解然后通过属性值prefix与定义的前缀相同，需要添加另外两个重要注解

    ~~~java
    package cn.itcast.user.pojo;
    
    import lombok.Data;
    import org.springframework.boot.context.properties.ConfigurationProperties;
    import org.springframework.stereotype.Component;
    
    @Data
    @Component
    @ConfigurationProperties(prefix = "pattern")
    public class PatternProperties {
        private String dateformat;
    }
    
    ~~~

    

### 1.3 Feign 后台发http请求

#### 1.3.1 简单请求发送

* 依赖

  ~~~xml
  <!-- 引入feign的基础依赖 -->
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-openfeign</artifactId>
  </dependency>
  ~~~

* 启动类上开启自动配置
  <font color="red">@EnableFeignClients</font>

* 提供映射对象接口

  ~~~java
  package cn.itcast.order.clients;
  
  
  import cn.itcast.order.pojo.User;
  import org.springframework.cloud.openfeign.FeignClient;
  import org.springframework.web.bind.annotation.GetMapping;
  import org.springframework.web.bind.annotation.PathVariable;
  
  // 该注解能将对象注册到IOC容器中，name代表需要访问的服务名称
  @FeignClient(name = "userService")
  public interface UserClient {
  
      /**
       * 与服务提供者的接口对应的Api
       * @param id 用户id
       * @return 请求获取的用户对象
       */
      @GetMapping("/user/{id}")
      User getUserById(@PathVariable("id") Long id);
  
  }
  
  ~~~

* 请求发送模板

  ~~~java
  package cn.itcast.order.service;
  
  import cn.itcast.order.clients.UserClient;
  import cn.itcast.order.mapper.OrderMapper;
  import cn.itcast.order.pojo.Order;
  import cn.itcast.order.pojo.User;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.stereotype.Service;
  
  @Service
  public class OrderService {
  
      @Autowired
      private OrderMapper orderMapper;
  
      @Autowired
      private UserClient userClient;
  
      public Order queryOrderById(Long orderId) {
          // 1.查询订单
          Order order = orderMapper.findById(orderId);
          // 2.根据用户id查询用户信息
          User user = userClient.getUserById(order.getUserId());
          // 3.将用户信息设置到订单中
          order.setUser(user);
          // 4.返回
          return order;
      }
  }
  
  ~~~

#### 1.3.2 标准请求发送

* 引入多线程模式的openhttp依赖

  ~~~xml
  <!-- openfeign依赖 -->
  <dependency>
      <groupId>io.github.openfeign</groupId>
      <artifactId>feign-httpclient</artifactId>
  </dependency>
  ~~~

* 日志配置以及连接池配置

  ~~~yml
  # 单个服务日志设置
  feign:
    client:
      config:
        # 要调用服务的名称
        user-center:
          loggerLevel: BASIC
          
  # 全局服务日志设置
  feign:
    client:
      config:
        default:
          loggerLevel: BASIC # 日志等级
    httpclient:
    connection-timeout: 200 # 最大等待时间ms
    enabled: true # 是否开启连接池
    max-connections: 200 # 最大连接数
    max-connections-per-route: 50 # 每个路径的最大连接数
  ~~~

  日志实现方式二----向IOC容器中注入一个Bean，默认的日志是NONE

  ```
  package cn.itcast.order.config;
  
  
  import feign.Logger;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  
  
  @Configuration
  public class DefaultFeignClientConfiguration {
  
      @Bean
      public Logger.Level feignLogLevel() {
          return Logger.Level.BASIC;
      }
  }
  
  ```

  实现全局或者单个服务的配置的方式取决于是将该类注册到启动类上还是单个服务@EnableFeignClients(defaultConfiguration = "DefaultFeignClientConfiguration")    全局

  @FeignClient(name = "userService", configuration = "DefaultFeignClientConfiguration")    单个服务

* 实现FeignClient和服务提供者接口统一的方式

  * 方式一 ---》 实现相同接口（不推荐）

    ![1659974208810](C:\Users\19816\AppData\Roaming\Typora\typora-user-images\1659974208810.png)

  * 方式二 ----》 将统一接口打包作为依赖引入
    ![1659974119502](C:\Users\19816\AppData\Roaming\Typora\typora-user-images\1659974119502.png)

## 2、服务的路由、权限、限流（gateway）

### 2.1 路由功能

* 导依赖
  需要导入nacos的依赖才能获取服务列表进行负载均衡，如果使用其他的也可以，但是需要导入对应依赖

  ~~~xml
  <!-- nacos发现依赖 -->
  <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
  </dependency>
  
  <!-- gateway网关依赖 -->
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-gateway</artifactId>
  </dependency>
  ~~~

* yml配置文件

  ~~~yml
  server:
    port: 10010
  spring:
    application:
      name: gateway #服务名称，因为需要注册到nacos中心，所以一定要名字
    cloud:
      nacos:
        server-addr: localhost:8848 # 配置nacos地址，获取微服务的信息进行负载均衡并将自己注册到nacos
      gateway:
        routes: # 路由设置
          - id: user-service # 唯一标识
            uri: lb://userService # 服务名称，可以直接使用服务名称，但是当前名称可以采用负载均衡
            predicates:   # 断言工厂，可添加多个断言，此处也是进行权限认证的重点
              - Path=/user/** # 路径断言，符合路径则进行路由，更多直接访问官网
            filters: # 添加请求头过滤器，能直接将配置信息添加到请求头
              - AddRequestHeader=name, zhangsan
          - id: order-service
            uri: lb://orderService # 服务名称
            predicates:
              - Path=/order/**
              
  	  # 默认的配置过滤器，全局生效并且最先执行
        default-filters:
        - AddRequestParameter=red, blue
  ~~~

### 2.2 跨域问题

* 如果不配置跨域，访问会发生未认证或者401、403
  ![1660449738125](C:\Users\19816\AppData\Roaming\Typora\typora-user-images\1660449738125.png)

* 配置跨域

  * 方法一（配置文件方法）

    ~~~yml
    spring:
      cloud:
        gateway:
          globalcors:
            cors-configurations:
              '[/**]':
                allowedOrigins: "https://docs.spring.io"
                allowedMethods:
                - GET
                - POST
                - PUT
                - DELETE
                - OPTION
    ~~~

  * 方法二（IOC容器注入Bean）

    ~~~java
    @Configuration
    public class MyCorsConfiguration {
    
        @Bean
        public CorsWebFilter corsWebFilter(){
            UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    
            CorsConfiguration corsConfiguration = new CorsConfiguration();
            //1.配置跨域
            //允许哪种请求头跨域
            corsConfiguration.addAllowedHeader("*");
            //允许哪种方法类型跨域 get post delete put
            corsConfiguration.addAllowedMethod("*");
            // 允许哪些请求源跨域
            corsConfiguration.addAllowedOrigin("*");
            // 是否携带cookie跨域
            corsConfiguration.setAllowCredentials(true);
    
            //允许跨域的路径
            source.registerCorsConfiguration("/**",corsConfiguration);
            return new CorsWebFilter(source);
        }
    }
    
    ~~~

    