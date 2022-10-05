# Spring 高级

## 1、IOC容器分类

### 1.1 BeanFactory

* 它是真正的Spring的IOC容器，它作为IOC容器的顶层接口，其他的容器实现功能都是“组合”了它的功能
* 它表面上只提供了getBean()方法，但实际上bean的控制反转、依赖注入以及生命周期的各种功能都是由它提供的
  ![1659627646964](..\pictures\1659627646964.png)

### 1.2 ApplicationContext容器

* 它的功能主要来自四个接口，分别是MessageSource、ResoucePatternReasolver、ApplicationEventPublisher、EnvironmentCapable
  ![1659627898133](..\pictures\1659628315971.png)

* MessageSource是用来进行国际化的功能，是用来将浏览器传过来的地区（时区）信息将语言转换成对应的语言的功能
  ![1659628020614](..\pictures\1659628020614.png)

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

* 它内部没有默认实现的后处理器，需要通过工具类进行添加
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

## 3. bean的生命周期及其后处理器

### 3.1 bean的生命周期

- 实例化 Instantiation

  实例化就是创建一个bean对象的过程，但是此时该bean中没有需要进行注入的属性，分为两种情况-----使用BeanFactory进行实例化和ApplicationContext对象进行实例化
  使用前者进行实例化需要等到该对象第一次被使用的时候才会进行，采用的是懒加载模式，而使用后者就会在容器创建的时候就进行实例化

- 属性填充 Populate
  给bean中填充其需要注入的属性，例如@Value、@Autowired注解等需要进行填充的属性

- 初始化 Initialization
  将bean所需要的条件全部满足，然后投入使用

- 销毁 Destruction
  单例通常是在spring容器关闭进行bean的销毁

**除了bean本身的生命周期之外，spring通过bean的后处理器进行了一系列的加强**

~~~java
// 实例化之前执行，返回的对象如果不为空就会替代原来的bean对象
default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
    return null;
}

// 实例化之后执行，返回值代表是否进入属性填充阶段
default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
    return true;
}

// 初始化之前执行，返回的对象如果不为空就会替代原来的bean对象
@Nullable
default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    return bean;
}

// 初始化之后执行，返回的对象如果不为空就会替代原来的bean对象（代理增强）
@Nullable
default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    return bean;
}
~~~

### 3.2 Bean后处理器相关

**<font color=red size=4>1、执行时机</font>**

* AutowiredAnnotationBeanPostProcessor
  解析@Autowired、@Value注解的后处理器，所以执行时机在属性填充阶段
* CommonAnnotationBeanPostProcessor
  解析@Resource、@PostConstruct、@PreDestroy注解，所以在属性填充和实例化后（初始化前）以及销毁方法之前都会执行
* ConfigurationPropertiesBindingPostProcessor
  解析@ConfigurationProperties注解，在初始化之前进行解析

**<font color=red size=4>2、Aware、ApplicationContextAware、InitializingBean接口</font>**

* 作用
  分别能获取注册的bean的名称，注册的容器以及调用属性填充之后的方法（初始化之前），和使用后处理器的方式不一样的是他们可以在任何情况下生效，只要在spring 的容器中，而使用后处理器还需要额外添加后处理器的bean对象进行相关注解的解析

<font color=red size=4>**3、@Autowired注解失效的情况**</font>

* 正常情况下一个bean的创建步骤
  * 执行BeanFactory的后处理器 
  *  注册执行Bean的后处理器
  * 创建bean对象以及初始化
    1. 解析@Value以及@Autowired注解
    2. 初始化扩展（如@PostConstruct））
    3. 执行Aware即InitializeBean接口
    4. 创建成功
* 如果一个bean中需要创建一个beanFactory后处理器放进容器中，就会导致该bean的对象创建和初始化会领先于两个后处理器的创建，最后导致扩展到后处理器功能失效，因为在创建对象以及初始化的时候没有后处理器，所以无法正确解析这些注解。解决方法就是使用Aware和InitializeBean接口进行这些增强的处理，这样就让这些功能不在依赖前两步的后处理器

### 3.3 bean的初始化和销毁方法

**初始化方法**（顺序按照写的顺序）

* 通过实现Instantiation接口重写的postProcessBeforeInitialization方法
* 通过在指定的bean中添加PostConstruct注解的方法
* 通过实现InitializingBean接口的重写的afterPropertiesSet方法
* 在通过@Bean注解创建的对象时在注解中添加initMethod属性的方法

**销毁方法**（顺序按照写的顺序）

* 通过在指定的bean中添加PreDestroy注解的方法
* 通过实现DisposableBean接口的重写的destroy方法
* 在通过@Bean注解创建的对象时在注解中添加destroyMethod属性的方法

### 3.4 scope类型

* 类型分类
  * singleton
  * prototype
  * request
  * session
  * application

* scope是用来标明当前bean对象在spring中存在的范围，总共分为5种，但是在单例对象进行其他类型对象时会产生失效，导致每次获取的其他scope类型的bean都是同一个对象。原因是单例对象指挥初始化一次，也只会进行一次属性填充，所以正常情况下每次获取的属性对象都是同一个。解决方法就是通过在获取属性对象时不是直接返回该对象而是通过其他方法每次将获取的对象进行刷新。
  * 在注入其他对象时添加一个@Lazy注解，通过代理生成对象
  * 在多例对象的@Scope注解上添加一个proxyMode=TARGET_ClASS代表使用CGLIB进行动态代理，不能使用INTERFACES值，因为使用创建的对象没有实现接口，所以无法创建实现类
  * 注入该多例对象的工厂ObjectFactory<TargetClass>，每次获取时调用工厂的getObject（）方法
  * 注入上下文对象，每次获取时调用getBean（）方法