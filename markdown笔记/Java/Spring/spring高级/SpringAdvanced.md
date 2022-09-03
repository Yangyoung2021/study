# Spring 高级

## 1、IOC容器分类

### 1.1 BeanFactory

* 它是真正的Spring的IOC容器，它作为IOC容器的顶层接口，其他的容器实现功能都是“组合”了它的功能
* 它表面上只提供了getBean()方法，但实际上bean的控制反转、依赖注入以及生命周期的各种功能都是由它提供的
  ![1659627646964](C:\Users\19816\AppData\Roaming\Typora\typora-user-images\1659627646964.png)

### 1.2 ApplicationContext容器

* 它的功能主要来自四个接口，分别是MessageSource、ResoucePatternReasolver、ApplicationEventPublisher、EnvironmentCapable
  ![1659627898133](C:\Users\19816\AppData\Roaming\Typora\typora-user-images\1659628315971.png)

* MessageSource是用来进行国际化的功能，是用来将浏览器传过来的地区（时区）信息将语言转换成对应的语言的功能
  ![1659628020614](C:\Users\19816\AppData\Roaming\Typora\typora-user-images\1659628020614.png)

* ResoucePatternReasolver是一个资源通配符解析器，用来解析资源路径中的通配符

  ~~~java
  // 使用classpath*来匹配所有的类路径下的匹配文件
  Resource[] resources = ctx.getResources("classpath*:META-INF/spring.factories");
  
  for (Resource resource : resources) {
      System.out.println(resource);
  }
  ~~~

* ApplicationEventPublisher是一个应用事件发布器，用来发布事件

  ~~~java
  
  /**
   * 事件发布器，当完成某项功能以后将消息发送，功能类似消息队列
   */
  @Component
  public class PublisherDemo01 extends ApplicationEvent {
  
  
      public PublisherDemo01(Object source) {
          super(source);
      }
  }
  ~~~

  ~~~java
  /**
   * 事件监听器，一种监听方法对应一种发布，发布的是什么对象就要使用什么对象来接收信息
   */
  @Component
  @Slf4j
  public class ComponentDemo01 {
  
      @EventListener
      public void listenerRegister(PublisherDemo01 publisherDemo01) {
          System.out.println(publisherDemo01);
          System.out.println("发送短信");
      }
  }
  ~~~

  ~~~java
  /**
   * 事件源对象，当事件源做了某些事情之后需要发布，就注入一个事件发布器来发布，然后将自己设置为发布源
   */
  @Component
  @Slf4j
  public class ComponentDemo2 {
  
      @Autowired
      private ApplicationEventPublisher context;
  
      public void register() {
          System.out.println("用户注册");
          context.publishEvent(new PublisherDemo01(this));
      }
  }
  ~~~

  ​		

  ​		<font color=red size=5px>通过这种方式可以实现事件发布与监听的解耦，事件的发布者不需要清楚到底是谁接收了它发布的信息，它只作为一个发送消息的工具使用，专注于自己的业务逻辑，而接收器可以通过传递过来的参数得到这个事件的发布者，知道这个事件的来龙去脉，发布器则可以设置发布这种事件都需要做的一个共性的事情，让这类相同的事情完全由发布器来完成，接收者只需要关注自己需要做的独特的业务。</font>

  

* EnvironmentCapable是一个用来获取环境相关的配置的接口，主要用来获取配置信息

  ~~~java
  // 通过getEnviroment()方法获取具体属性，而该方法就是通过实现EnvironmentCapable获取的
  System.out.println(ctx.getEnvironment().getProperty("java_home"));
  System.out.println(ctx.getEnvironment().getProperty("server.port"));
  ~~~

  

## 2、容器实现

### 2.1 BeanFactory的重要实现

**<font color=red size=4>DefaultListableBeanFactory</font>**

* 它不会主动调用BeanFactory的后处理器
* 它不会主动调用Bean的后处理器
* 它其中的单例对象默认是延迟加载的（第一次用到的时候再去创建）
* 它不会解析BeanFactory和#{}、${}
* Bean后处理器会有排序的逻辑

辨析：

​	@Resource注解和@Autowired注解的注入方式差别（二者同时使用默认按照属性名称匹配）

* Autowired注解默认按照类型注入，如果匹配到0个且注解的属性required为true就报错，匹配到多个或者就会按照需要注入的属性的名称去查找（先类型后名称）
* Resource注解匹配到单个直接进行注入，多个默认是通过注解中的name属性进行匹配，如果name没有指定也会根据需要注入的属性名称进行匹配

~~~java
package com.yang.beanFactory;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.beans.factory.support.AbstractBeanDefinition;
import org.springframework.beans.factory.support.BeanDefinitionBuilder;
import org.springframework.beans.factory.support.DefaultListableBeanFactory;
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.context.annotation.AnnotationConfigUtils;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.core.RedisOperations;
import org.springframework.stereotype.Component;


@ConditionalOnBean(RedisOperations.class)
public class DefaultListableBeanFactoryDemo {


    public static void main(String[] args) {
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();

        // 想要将需要的对象放到容器中，需要将对象先通过工具类注册成BeanDefinition对象
        AbstractBeanDefinition demo01 = 														BeanDefinitionBuilder.genericBeanDefinition(Demo01.class).
            setScope("singleton").getBeanDefinition();
        // 将BeanDefinition对象注册到容器中，但此时不会解析注解，而且此时还只是给你注册了名字，没有真			正成为一个Bean对象
        beanFactory.registerBeanDefinition("demo01", demo01);

        // 使用工具类给beanFactory注册后处理器
        AnnotationConfigUtils.registerAnnotationConfigProcessors(beanFactory);

        // 拿到BeanFactory后处理器对beanFactory进行处理，是对使用@Configuration和@Component注解			中的@Bean注解定义的要添加的bean进行解析
        // 并不能直接解析@Configuration和@Component注解将其转换成对象
       																							beanFactory.getBeansOfType(BeanFactoryPostProcessor.class)
           .values().forEach(beanFactoryPostProcessor -> {
            System.out.println(beanFactoryPostProcessor.getClass().getName());
            beanFactoryPostProcessor.postProcessBeanFactory(beanFactory);
        });

        // 拿到Bean后处理器对注解进行解析，包含两个后处理器对象，
        // AutowiredAnnotationBeanPostProcessor(解析@Autowired)和									CommonAnnotationBeanPostProcessor(解析@Resource)
       		beanFactory.getBeansOfType(BeanPostProcessor.class)
			.values().forEach(beanPostProcessor -> {
            System.out.println(beanPostProcessor.getClass().getName());
            beanFactory.addBeanPostProcessor(beanPostProcessor);
        });


        String[] names = beanFactory.getBeanDefinitionNames();

        for (String name : names) {
            System.out.println(name);
        }
        
        System.out.println("-------------------");

        Demo01 bean01 = beanFactory.getBean(Demo01.class);

        bean01.testDemo02();

        System.out.println(beanFactory.getBean(Demo02.class).getDemo03());


    }


    @Configuration
    static class Demo01 {

        @Bean
        public Demo03 demo03() {
            return new Demo03();
        }

        @Bean
        public Demo02 demo02() {
            return new Demo02();
        }

        @Autowired
        private Demo02 demo02;

        public void testDemo02() {
            System.out.println(demo02);
        }


    }

    @Component
    static class Demo02 {

        public Demo02() {
            System.out.println("构造Demo02.。。");
        }

        @Autowired
        private Demo03 demo03;

        public Demo03 getDemo03() {
            return demo03;
        }


    }

    static class Demo03 {

    }
}

~~~

### 2.2 ApplicationContext的实现

#### 2.2.1 ClassPathXmlApplicationContext

* 通过类路径（resources）的XML配置文件创建IOC容器
* 通过Bean标签实现Bean的定义

#### 2.2.2 FileSystemXmlApplicationContext

* 可以直接扫描当前设备上的XML文件来作为源创建IOC容器

**注意：**

​		前两个都是通过创建好一个DefaultListableBeanFactory，然后通过一个能够解析Xml文件的类来将配置文件中的Bean解析成为一个BeanDefinition对象放到创建好的容器中。并且这两个创建的默认的容器中也是没有BeanFactory和Bean的后置处理器的，打开方式是在配置文件中引入命名空间然后使用context标签。

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans
        xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
       http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
">

    <context:annotation-config/>

    <bean id="bean1" class="com.yang.beanFactory.DefaultListableBeanFactoryDemo.Bean1"/>

</beans>
~~~



~~~java
package com.yang.beanFactory;

import org.springframework.beans.factory.support.DefaultListableBeanFactory;
import org.springframework.beans.factory.xml.XmlBeanDefinitionReader;

public class ApplicationContextImpls {


    public static void main(String[] args) {
        // 创建IOC容器
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        // 创建能够直接将Xml文件中定义好的Bean对象解析成BeanDefinition对象的解析器，将IOC容器作为接收的对象
        XmlBeanDefinitionReader xmlBeanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
        // 指定Xml配置文件的位置让解析器去读取
        xmlBeanDefinitionReader.loadBeanDefinitions("applicationContext.xml");
        // 获取IOC容器中的Bean
        String[] beanDefinitionNames = beanFactory.getBeanDefinitionNames();
        // 打印
        for (String beanDefinitionName : beanDefinitionNames) {
            System.out.println(beanDefinitionName);
        }

    }
}

~~~

#### 2.2.3 AnnotationConfigApplicationContext

* 这个对象是使用配置类的方法进行Bean的注册，在获取容器时需要传入配置类的字节码文件
* 这个容器创建的时候就会默认添加BeanFactory和Bean的后置处理器
* 多用于非Web环境

~~~java
private static void AnnotationApplicationContext() {
    AnnotationConfigApplicationContext context = new 									     						AnnotationConfigApplicationContext(Config.class);
    String[] beanDefinitionNames = context.getBeanDefinitionNames();

    for (String beanDefinitionName : beanDefinitionNames) {
        System.out.println(beanDefinitionName);
    }
}
~~~

#### 2.2.4 AnnotationConfigServletWebServerApplicationContext

* 这是一个用于Web环境的容器，其配置方法和普通的注解配置方法一样，只不过需要配置一些必要的bean对象用于web服务

~~~java
private static void WebAnnotationApplicationContext() {
        AnnotationConfigServletWebServerApplicationContext webServerApplicationContext =
                new AnnotationConfigServletWebServerApplicationContext(WebConfig.class);

        String[] names = webServerApplicationContext.getBeanDefinitionNames();

        for (String name : names) {
            System.out.println(name);
        }
    }


    @Configuration
    static class WebConfig {
        /**
         * 配置Tomcat服务器
         * @return Tomcat网络服务器工厂
         */
        @Bean
        public ServletWebServerFactory servletWebServerFactory() {
            return new TomcatServletWebServerFactory();
        }

        /**
         * 配置前端控制器
         * @return 前端控制器实现
         */
        @Bean
        public DispatcherServlet dispatcherServlet() {
            return new DispatcherServlet();
        }

        /**
         * 注册前端控制器并设置服务器要拦截的请求
         * @param dispatcherServlet 前端控制器
         * @return 注册前端控制器的Bean
         */
        @Bean
        public DispatcherServletRegistrationBean 													dispatcherServletRegistrationBean(DispatcherServlet dispatcherServlet) {
            return new DispatcherServletRegistrationBean(dispatcherServlet, "/");
        }

        /**
         * 测试接口
         * @return null
         */
        @Bean("/hello")
        public Controller controller1() {
            return (request, response) -> {
                response.getWriter().write("Hello ....");
                return null;
            };
        }
    }

~~~

