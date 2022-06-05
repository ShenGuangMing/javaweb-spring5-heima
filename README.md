# 介绍
代码仓库地址：[https://gitee.com/CandyWall/spring-source-study](https://gitee.com/CandyWall/spring-source-study)
跟着**黑马满一航老师的spring高级49讲**做的学习笔记，本笔记跟视频内容的项目名称和代码略有不同，我将49讲的代码每一讲的代码都拆成了独立的springboot项目，并且项目名称尽量做到了见名知意，都是基于我自己的考量，代码都已经过运行验证过的，仅供参考。

视频教程地址：[https://www.bilibili.com/video/BV1P44y1N7QG](https://www.bilibili.com/video/BV1P44y1N7QG)

<font color="red">注：</font>

​	<font color="red">1. 每一讲对应一个二级标题，每一个三级标题是使用子项目名称命名的，和我代码仓库的项目是一一对应的；</font>
​	<font color="red">2. 代码里面用到了lombok插件来简化了Bean中的get()、set()方法，以及日志的记录的时候用了lombok的@Slf4j注解。</font>

**笔记中如有不正确的地方，欢迎在评论区指正，非常感谢！！！**

每个子项目对应的视频链接以及一些重要内容的笔记

## 第一讲 `BeanFactory`与`ApplicationContext`的区别与联系

### spring_01_beanfactory_applicationcontext_differences_connections

[p1 000-Spring高级49讲-导学](https://www.bilibili.com/video/BV1P44y1N7QG?p=1)

[p2 001-第一讲-BeanFactory与ApplicationContext_1](https://www.bilibili.com/video/BV1P44y1N7QG?p=2)

测试代码：

```java
@SpringBootApplication
@Slf4j
public class A01Application {
    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(A01Application.class, args);
        // class org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext
        System.out.println(context.getClass());
    }
}
```

到底什么是`BeanFactory`
* 它是`ApplicationContext`的父接口

  鼠标选中`ConfigurableApplicationContext`，按`Ctrl + Shift + U`或者`Ctrl + Alt + U`打开类图，可以看到`ApplicationContext`的有个父接口是`BeanFactory`

  ![image-20220323144102451](http://jp88.top/image/image-20220323144102451.png)

* 它才是 `Spring` 的核心容器，主要的 `ApplicationContext` 实现都 [组合]了他的功能

  打印`context.getClass()`，可以看到SpringBoot的启动程序返回的`ConfigurableApplicationContext`的具体的实现类是`AnnotationConfigServletWebServerApplicationContext`

  ```java
  ConfigurableApplicationContext context = SpringApplication.run(A01Application.class, args);
  // org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext
  System.out.println(context.getClass());
  ```

  按图索骥，`AnnotationConfigServletWebServerApplicationContext`又间接继承了`GenericApplicationContext`，在这个类里面可以找到`beanFactory`作为成员变量出现。

  ![image-20220404140925587](http://jp88.top/image/image-20220404140925587.png)
  
  ![image-20220404141152288](http://jp88.top/image/image-20220404141152288.png)

[p3 002-第一讲-BeanFactory功能](https://www.bilibili.com/video/BV1P44y1N7QG?p=3)

`BeanFactory`接口中的方法

![image-20220323145908937](http://jp88.top/image/image-20220323145908937.png)

查看`springboot`默认的`ConfigurableApplicationContext`类中的`BeanFactory`的实际类型

```java
ConfigurableApplicationContext context = SpringApplication.run(A01Application.class, args);
// org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext
ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
// 查看实际类型
// class org.springframework.beans.factory.support.DefaultListableBeanFactory
System.out.println(beanFactory.getClass());
```

从打印结果可以了解到实际类型为`DefaultListableBeanFactory`，所以这里以`BeanFactory`的一个实现类`DefaultListableBeanFactory`作为出发点，进行分析。

它的类图如下：

![image-20220323150712761](http://jp88.top/image/image-20220323150712761.png)

这里我们暂且不细看`DefaultListableBeanFactory`，先看`DefaultListableBeanFactory`的父类`DefaultSingletonBeanFactory`，先选中它，然后按`F12`，可以跳转到对应的源码，可以看到有个私有的成员变量`singletonObjects`

![image-20220323150929451](http://jp88.top/image/image-20220323150929451.png)

这里通过反射的方法来获取该成员变量，进行分析

>  先补充一下反射获取某个类的成员变量的步骤：
>
> 获取成员变量，步骤如下：
>
> 1. 获取Class对象
>
> 2. 获取构造方法
>
> 3. 通过构造方法，创建对象
>
> 4. 获取指定的成员变量（私有成员变量，通过**setAccessible**(boolean flag)方法暴力访问）
>
> 5. 通过方法，给指定对象的指定成员变量赋值或者获取值
>
>    public void set(Object obj, Object value)
>
>    ​       在指定对象obj中，将此 Field 对象表示的成员变量设置为指定的新值
>
> ​        public Object get(Object obj)
>
> ​           	返回指定对象obj中，此 Field 对象表示的成员变量的值

代码如下：

```java
Field singletonObjects = DefaultSingletonBeanRegistry.class.getDeclaredField("singletonObjects");
// 设置私有变量可以被访问
singletonObjects.setAccessible(true);
ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
Map<String, Object> map = (Map<String, Object>) singletonObjects.get(beanFactory);
// 查看实际类型
// class org.springframework.beans.factory.support.DefaultListableBeanFactory
System.out.println(beanFactory.getClass());
map.entrySet().stream().filter(entry -> entry.getKey().startsWith("component")).forEach(System.out::println);
```

这里`singletonObjects.get(beanFactory)`为什么要传一个`ConfigurableListableBeanFactory`的变量进去呢？打印了这个`beanFactory`的实际类型为`DefaultListableBeanFactory`，查看其类图，可以了解到该类也实现了`DefaultSingletonBeanRegistry`接口，所以这里反射获取某个类的成员变量的`get()`方法中可以作为参数传进来。

![image-20220324003636506](http://jp88.top/image/image-20220324003636506.png)

[p4 003-第一讲-ApplicationContext功能1](https://www.bilibili.com/video/BV1P44y1N7QG?p=4)

`ApplicationContext` 比 `BeanFactory` 多点啥？

多实现了四个接口：

* `MessageSource`: 国际化功能，支持多种语言
* `ResourcePatternResolver`: 通配符匹配资源路径
* `EnvironmentCapable`:  环境信息，系统环境变量，`*.properties`、`*.application.yml`等配置文件中的值
* `ApplicationEventPublisher`: 发布事件对象

![image-20220324115647260](http://jp88.top/image/image-20220324115647260.png)

1. `MessageSource`

   在`resources`目录下创建四个文件`messages.propertes`、`messages_en.properties`、`messages_ja.properties`、`messages_zh.properties`，然后分别在四个文件里面定义同名的`key`，比如在`message_en.properties`中定义`hi=hello`，在`messages_ja.propertes`中定义`hi=こんにちは`，在`messages_zh`中定义`hi=你好`，这样在代码中就可以根据这个**`key hi`**和不同的**语言类型**获取不同的`value`了。

   ```java
   System.out.println(context.getMessage("hi", null, Locale.CHINA));
   System.out.println(context.getMessage("hi", null, Locale.ENGLISH));
   System.out.println(context.getMessage("hi", null, Locale.JAPANESE));
   ```

   运行结果如下：

   ![image-20220324181409040](http://jp88.top/image/image-20220324181409040.png)

[p5 004-第一讲-ApplicationContext功能2,3](https://www.bilibili.com/video/BV1P44y1N7QG?p=5)

2. `ResourcePatternResolver`

   例1：获取类路径下的`messages`开头的配置文件

   ```java
   Resource[] resources = context.getResources("classpath:messages*.properties");
   for (Resource resource : resources) {
       System.out.println(resource);
   }
   ```

   ![image-20220324182456169](http://jp88.top/image/image-20220324182456169.png)

   例2：获取`spring`相关`jar`包中的`spring.factories`配置文件

   ```java
   resources = context.getResources("classpath*:META-INF/spring.factories");
   for (Resource resource : resources) {
       System.out.println(resource);
   }
   ```

   ![image-20220324183236048](http://jp88.top/image/image-20220324183236048.png)

3. `EnvironmentCapable`

   获取系统环境变量中的`java_home`和项目的`application.yml`中的`server.port`属性

   ```java
   System.out.println(context.getEnvironment().getProperty("java_home"));
   System.out.println(context.getEnvironment().getProperty("server.port"));
   ```

   ![image-20220324191740825](http://jp88.top/image/image-20220324191740825.png)

[p6 005-第一讲-ApplicationContext功能4](https://www.bilibili.com/video/BV1P44y1N7QG?p=6)

4. `ApplicationEventPublisher`

   定义一个**用户注册事件**类，继承自`ApplicationEvent`类

   ```java
   public class UserRegisteredEvent extends ApplicationEvent {
       public UserRegisteredEvent(Object source) {
           super(source);
       }
   }
   ```

   再定义一个**监听器**类，用于监听用户注册事件，类头上需要加`@Component`注解，将该类交给`spring`管理，定义一个处理事件的方法，参数类型为**用户注册事**件类的对象，方法头上需要加上`@EventListener`注解

   ```java
   @Component
   @Slf4j
   public class UserRegisteredListener {
       @EventListener
       public void userRegist(UserRegisteredEvent event) {
           System.out.println("UserRegisteredEvent...");
           log.debug("{}", event);
       }
   }
   ```

   接着再定义一个**用户服务**类，里面有个`register(String username, String password)`方法可以完成用户的注册，注册完毕后发布一下**用户注册完毕事件**。

   ```java
   @Component
   @Slf4j
   public class UserService {
       @Autowired
       private ApplicationEventPublisher context;
       public void register(String username, String password) {
           log.debug("新用户注册，账号：" + username + "，密码：" + password);
           context.publishEvent(new UserRegisteredEvent(this));
       }
   }
   ```

   最后在`Springboot`启动类中调用一下`UserService`里面的`register()`方法注册一个新用户，`UserRegisteredListener`中就能处理这个用户注册完毕的事件，实现了`UserService`类和`UserRegisteredListener`类的解耦。

   ```java
   UserService userService = context.getBean(UserService.class);
   userService.register("张三", "123456");
   ```

   ![image-20220324210306704](http://jp88.top/image/image-20220324210306704.png)

[p7 006-第一讲-小结](https://www.bilibili.com/video/BV1P44y1N7QG?p=7)

## 第二讲 `BeanFactory` 和 `ApplicationContext` 类的重要实现类

### spring_02_01_beanfactory_impl

[p8 007-第二讲-BeanFactory实现](https://www.bilibili.com/video/BV1P44y1N7QG?p=8)

[p9 008-第二讲-BeanFactory实现](https://www.bilibili.com/video/BV1P44y1N7QG?p=9)

[p10 009-第二讲-BeanFactory实现-后处理器排序](https://www.bilibili.com/video/BV1P44y1N7QG?p=10)

`DefaultListableBeanFactory`

接着第一讲中的内容，执行以下代码，可以了解到

```java
ConfigurableApplicationContext context = SpringApplication.run(Application.class, args);
// org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext
ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
// 查看实际类型
// class org.springframework.beans.factory.support.DefaultListableBeanFactory
System.out.println(beanFactory.getClass());
```

`ConfigurableApplicationContext`类内部组合的`BeanFactory`实际类型为`DefaultListableBeanFactory`，`spring`底层创建实体类就是依赖于这个类，所以它是`BeanFactory`接口最重要的一个实现类，下面使用这个类，模拟一下`spring`使用`DefaultListableBeanFactory`类创建其他实体类对象的过程。

测试代码如下：

```java
package top.jacktgq.spring_02_beanfactory_impl;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.beans.factory.support.AbstractBeanDefinition;
import org.springframework.beans.factory.support.BeanDefinitionBuilder;
import org.springframework.beans.factory.support.DefaultListableBeanFactory;
import org.springframework.context.annotation.AnnotationConfigUtils;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.annotation.Resource;
import java.util.ArrayList;
import java.util.stream.Collectors;

/**
 * @Author CandyWall
 * @Date 2022/3/24--21:20
 * @Description
 */
public class TestBeanFactory {
    public static void main(String[] args) {
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        // bean 的定义（即bean的一些描述信息，包含class：bean是哪个类，scope：单例还是多例，初始化、销毁方法等）
        AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition(Config.class).setScope("singleton").getBeanDefinition();
        beanFactory.registerBeanDefinition("config", beanDefinition);
        // 给 BeanFactory添加一些常用的后处理器，让它具备解析@Configuration、@Bean等注解的能力
        AnnotationConfigUtils.registerAnnotationConfigProcessors(beanFactory);
        // 从bean工厂中取出BeanFactory的后处理器，并且执行这些后处理器
        // BeanFactory 后处理器主要功能，补充了一些 bean 的定义
        beanFactory.getBeansOfType(BeanFactoryPostProcessor.class).values().forEach(beanFactoryPostProcessor -> {
            System.out.println(beanFactoryPostProcessor);
            beanFactoryPostProcessor.postProcessBeanFactory(beanFactory);
        });
        // 打印BeanFactory中Bean
        for (String name : beanFactory.getBeanDefinitionNames()) {
            System.out.println(name);
        }

        // 从BeanFactory中取出Bean1，然后再从Bean1中取出它依赖的Bean2
        // 可以看到结果为null，所以@Autowired注解并没有被解析
        // Bean1 bean1 = beanFactory.getBean(Bean1.class);
        // System.out.println(bean1.getBean2());

        // 要想@Autowired、@Resource等注解被解析，还要添加Bean的后处理器，可以针对Bean的生命周期的各个阶段提供扩展
        // 从bean工厂中取出Bean的后处理器，并且执行这些后处理器
        // BeanFactory 后处理器主要功能，补充了一些 bean 的定义
        // beanFactory.getBeansOfType(BeanPostProcessor.class).values().forEach(beanFactory::addBeanPostProcessor);
        // beanFactory.addBeanPostProcessors(beanFactory.getBeansOfType(BeanPostProcessor.class).values());
        // 改变Bean后处理器加入BeanFactory的顺序
        // 写法1：
        // ArrayList<BeanPostProcessor> list = new ArrayList<>(beanFactory.getBeansOfType(BeanPostProcessor.class).values());
        // Collections.reverse(list);
        // beanFactory.addBeanPostProcessors(list);
        // 写法2：
        beanFactory.addBeanPostProcessors(beanFactory.getBeansOfType(BeanPostProcessor.class).values().stream().sorted(beanFactory.getDependencyComparator()).collect(Collectors.toCollection(ArrayList::new)));

        // 准备好所有单例，get()前就把对象初始化好
        beanFactory.preInstantiateSingletons();
        System.out.println(">>>>>>>>>>>>>>>>>>>>>>>>>>>>");
        Bean1 bean1 = beanFactory.getBean(Bean1.class);
        System.out.println(bean1.getBean2());

        /**
         * 学到了什么：
         *      a. beanFactory 不会做的事
         *         1. 不会主动调用BeanFactory的后处理器
         *         2. 不会主动添加Bean的后处理器
         *         3. 不会主动初始化单例
         *         4. 不会解析BeanFactory，还不会解析 ${}, #{}
         *
         *      b. Bean后处理器会有排序的逻辑
         */
        System.out.println(bean1.getInter());
    }

    @Configuration
    static class Config {
        @Bean
        public Bean1 bean1() {
            return new Bean1();
        }

        @Bean
        public Bean2 bean2() {
            return new Bean2();
        }

        @Bean
        public Bean3 bean3() {
            return new Bean3();
        }

        @Bean
        public Bean4 bean4() {
            return new Bean4();
        }
    }

    @Slf4j
    static class Bean1 {
        @Autowired
        private Bean2 bean2;

        public Bean2 getBean2() {
            return bean2;
        }

        @Autowired
        @Resource(name = "bean4")
        private Inter bean3;

        public Inter getInter() {
            return bean3;
        }

        public Bean1() {
            log.debug("构造 Bean1()");
        }
    }

    @Slf4j
    static class Bean2 {
        public Bean2() {
            log.debug("构造 Bean2()");
        }
    }

    interface Inter {

    }

    @Slf4j
    static class Bean3 implements Inter {
        public Bean3() {
            log.debug("构造 Bean3()");
        }
    }

    @Slf4j
    static class Bean4 implements Inter {
        public Bean4() {
            log.debug("构造 Bean4()");
        }
    }
}
```

总结：

* beanFactory 不会做的事

  * 不会主动调用BeanFactory的后处理器

  * 不会主动添加Bean的后处理器
  * 不会主动初始化单例
  * 不会解析BeanFactory，还不会解析 ${}, #{}

* Bean后处理器会有排序的逻辑

  先定义一个接口Inter，再定义两个Bean，名称分别为Bean3和Bean4，都继承Inter，接着在Config中通过@Bean注解将Bean3和Bean4都加进Bean工厂中，然后在Bean1中定义一个Inter对象，通过@Autowired注解将实现类注入进来。

  ```java
  @Configuration
  static class Config {
      @Bean
      public Bean1 bean1() {
          return new Bean1();
      }
  
      @Bean
      public Bean2 bean2() {
          return new Bean2();
      }
  
      @Bean
      public Bean3 bean3() {
          return new Bean3();
      }
  
      @Bean
      public Bean4 bean4() {
          return new Bean4();
      }
  }
  
  @Slf4j
  static class Bean1 {
      @Autowired
      private Bean2 bean2;
  
      public Bean2 getBean2() {
          return bean2;
      }
  
      @Autowired
      @Resource(name = "bean4")
      private Inter bean3;
  
      public Inter getInter() {
          return bean3;
      }
  
      public Bean1() {
          log.debug("构造 Bean1()");
      }
  }
  
  @Slf4j
  static class Bean2 {
      public Bean2() {
          log.debug("构造 Bean2()");
      }
  }
  
  interface Inter {
  
  }
  
  @Slf4j
  static class Bean3 implements Inter {
      public Bean3() {
          log.debug("构造 Bean3()");
      }
  }
  
  @Slf4j
  static class Bean4 implements Inter {
      public Bean4() {
          log.debug("构造 Bean4()");
      }
  }
  ```

  如果把以`Inter`接口声明的变量名定义为`inter`，`@Autowired`注解首先会**根据名称(byName)**进行匹配，没有匹配上，于是又会**根据类型(byType)**进行匹配，发现Bean3和Bean4都实现了Inter接口，会报无法自动装配的错误。

  ![image-20220325121422536](http://jp88.top/image/image-20220325121422536.png)

  所以为了避免这种错误，以`Inter`接口声明的变量名只能为`bean3`或者`bean4`，这里把以`Inter`接口声明的变量名定义为`bean3`，然后就不报错了，`@Autowired`会通过`byName`的方式进行匹配。

  ![image-20220325121717437](http://jp88.top/image/image-20220325121717437.png)

  在main方法中去获取Inter，然后打印，可以看到注入的是Bean3

  ![image-20220325133334762](http://jp88.top/image/image-20220325133334762.png)

  如果此时在`private Inter bean3;`上面再加上`@Resource(name = "bean4")`注解，然后再打印结果，结果还是`bean3`，为什么呢？我们先看一下加入`BeanFactory`的Bean后处理器的顺序，解析`@Autowired`注解的后处理器`internalAutowiredAnnotationProcessor`的顺序排在解析`@Resource`注解的后处理器`internalCommonAnnotationProcessor`的前面，所以`internalAutowiredAnnotationProcessor`会被`BeanFactory`先启用，故`@Autowired`注解先被解析了。

  ![image-20220325134154838](http://jp88.top/image/image-20220325134154838.png)

  如果想要让`@Resource`注解先被解析呢，这就需要让后处理器`internalCommonAnnotationProcessor`比`internalAutowiredAnnotationProcessor`先加入`BeanFactory`，代码如下：

  ```java
  // 改变Bean后处理器加入BeanFactory的顺序
          beanFactory.addBeanPostProcessors(beanFactory.getBeansOfType(BeanPostProcessor.class).values().stream().sorted(beanFactory.getDependencyComparator()).collect(Collectors.toCollection(ArrayList::new)));
  ```

  这样一来注入的结果就是`Bean4`，`@Resource(name = "bean4")`注解被先解析了

  ![image-20220325141300707](http://jp88.top/image/image-20220325141300707.png)

  通过`AnnotationConfigUtils`给`beanFactory`添加一些后处理的时候会默认设置比较器，可以对`BeanPostProcessor`进行排序，排序的依据是`BeanPostProcessor`内部的`order`属性，其中`internalAutowiredAnnotationProcessor`的order属性的值为`Ordered.LOWEST_PRECEDENCE - 2`，`internalCommonAnnotationProcessor`的`order`属性的值为`Ordered.LOWEST_PRECEDENCE - 3`。

  ![image-20220325143844005](http://jp88.top/image/image-20220325143844005.png)

  从打印结果来看，`internalAutowiredAnnotationProcessor:2147483645`，`internalCommonAnnotationProcessor:2147483644`，`internalCommonAnnotationProcessor`的`order`值更小，所以排序的时候会排在前面

  ![image-20220325144433436](http://jp88.top/image/image-20220325144433436.png)

* `BeanFactory`本身功能只是将定义好的`BeanDefinition`加进来，而`BeanFactory`的后处理器`BeanFactoryPostProcessor`补充了一些`Bean`的定义，可以解析`@Configuration`、`@Bean`等注解，将这些被注解修饰的`Bean`也加进`BeanFactory`。`@Configuration`和`@Bean`注解的解析过程的源码可以看`AnnotationConfigUtils`和`ConfigurationClassPostProcessor`

  ![image-20220325005027558](http://jp88.top/image/image-20220325005027558.png)

  ![image-20220325005135527](http://jp88.top/image/image-20220325005135527.png)

* 要想`@Autowired`、`@Resource`等注解被解析，还要添加`Bean`的后处理器`BeanPostProcessor`，可以针对`Bean`的生命周期的各个阶段提供扩展。

* BeanFactory中的对象都是懒加载的，如果不去调用get()方法获取的话，就不会初始化，如果想要让对象在get()之前就创建好，需要调用`beanFactory.preInstantiateSingletons()`方法。

* 教程弹幕中有人问：为啥`@Bean`和`@Configration`注解不需要建立联系就能使用？

  * 建立联系了啊，上面也获取了`BeanFactory`的后置处理器，然后`foreach`循环就是建立`BeanFactory`的后置处理器和BeanFactory的联系。
  * 另外`@Configuration`加不加，`Config`类中的`@Bea`n注解都会被解析，`@Configuration`是用于`spring`类扫描的时候用的，加了这个注解的类被扫描到了就会被放进`Bean`工厂

### spring_02_02_applicationcontext_impl

[p11 010-第二讲-ApplicationContext实现1,2](https://www.bilibili.com/video/BV1P44y1N7QG?p=11)

[p12 011-第二讲-ApplicationContext实现3](https://www.bilibili.com/video/BV1P44y1N7QG?p=12)

[p13 012-第二讲-ApplicationContext实现4](https://www.bilibili.com/video/BV1P44y1N7QG?p=13)

四个重要的`ApplicationContext`接口的实现类

![image-20220325174005606](http://jp88.top/image/image-20220325174005606.png)

* `ClassPathXmlApplicationContext`:
* `FileSystemXmlApplicationContext`:
* `AnnotationConfigApplicationContext`:
* `AnnotationConfigServletWebServerApplication`:

相关测试代码如下：

```java
@Slf4j
public class TestApplicationContext {
    @Test
    // ⬇️1.最为经典的容器，基于classpath 下 xml 格式的配置文件来创建
    public void testClassPathXmlApplicationContext() {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring_bean.xml");
        for (String name : context.getBeanDefinitionNames()) {
            System.out.println(name);
        }
        System.out.println(context.getBean(Bean2.class).getBean1());
    }

    @Test
    // ⬇️2.基于磁盘路径下 xml 格式的配置文件来创建
    public void testFileSystemXmlApplicationContext() {
        // 可以用绝对路径或者相对路径
        // FileSystemXmlApplicationContext context = new FileSystemXmlApplicationContext("D:\\ideacode\\spring-source-study\\spring_02_02_applicationcontext_impl\\src\\main\\resources\\spring_bean.xml");
        FileSystemXmlApplicationContext context = new FileSystemXmlApplicationContext("src\\main\\resources\\spring_bean.xml");
        for (String name : context.getBeanDefinitionNames()) {
            System.out.println(name);
        }
        System.out.println(context.getBean(Bean2.class).getBean1());
    }

    @Test
    // ⬇️模拟一下ClassPathXmlApplicationContext和FileSystemXmlApplicationContext底层的一些操作
    public void testMockClassPathAndFileSystemXmlApplicationContext() {
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        System.out.println("读取之前");
        for (String name : beanFactory.getBeanDefinitionNames()) {
            System.out.println(name);
        }
        System.out.println("读取之后");
        XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(beanFactory);
        // reader.loadBeanDefinitions("spring_bean.xml");
        // reader.loadBeanDefinitions(new ClassPathResource("spring_bean.xml"));
        reader.loadBeanDefinitions(new FileSystemResource("src\\main\\resources\\spring_bean.xml"));
        for (String name : beanFactory.getBeanDefinitionNames()) {
            System.out.println(name);
        }
    }

    @Test
    // ⬇️3.较为经典的容器，基于java配置类来创建
    public void testAnnotationConfigApplicationContext() {
        // 会自动加上5个后处理器
        // org.springframework.context.annotation.internalConfigurationAnnotationProcessor
        // org.springframework.context.annotation.internalAutowiredAnnotationProcessor
        // org.springframework.context.annotation.internalCommonAnnotationProcessor
        // org.springframework.context.event.internalEventListenerProcessor
        // org.springframework.context.event.internalEventListenerFactory
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(Config.class);
        for (String name : context.getBeanDefinitionNames()) {
            System.out.println(name);
        }
        System.out.println(context.getBean(Bean2.class).getBean1());
    }

    @Test
    // ⬇️4.较为经典的容器，基于java配置类来创建，并且还可以用于web环境
    // 模拟了 springboot web项目内嵌Tomcat的工作原理
    public void testAnnotationConfigServletWebServerApplicationContext() throws IOException {
        AnnotationConfigServletWebServerApplicationContext context = new AnnotationConfigServletWebServerApplicationContext(WebConfig.class);
        // 防止程序终止
        System.in.read();
    }
}

@Configuration
class WebConfig {
    @Bean
    // 1. WebServer工厂
    public ServletWebServerFactory servletWebServerFactory() {
        return new TomcatServletWebServerFactory();
    }

    @Bean
    // 2. web项目必备的DispatcherServlet
    public DispatcherServlet dispatcherServlet() {
        return new DispatcherServlet();
    }

    @Bean
    // 3. 将DispatcherServlet注册到WebServer上
    public DispatcherServletRegistrationBean dispatcherServletRegistrationBean(DispatcherServlet dispatcherServlet) {
        return new DispatcherServletRegistrationBean(dispatcherServlet, "/");
    }

    @Bean("/hello")
    public Controller controller1() {
        return (request, response) -> {
            response.getWriter().println("hello");
            return null;
        };
    }
}

// 单元测试的过程中如果要解析一些Spring注解，比如@Configuration的时候不要把相关类定义到写单元测试类的内部类，会读取不到
@Configuration
class Config {
    @Bean
    public Bean1 bean1() {
        return new Bean1();
    }

    @Bean
    public Bean2 bean2(Bean1 bean1) {
        Bean2 bean2 = new Bean2();
        bean2.setBean1(bean1);
        return bean2;
    }
}

class Bean1 {

}

class Bean2 {
    private Bean1 bean1;

    public Bean1 getBean1() {
        return bean1;
    }

    public void setBean1(Bean1 bean1) {
        this.bean1 = bean1;
    }
}
```

`spring_bean.xml`如下：

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">
    <!--
        把5个后处理器加进来
            等价于：AnnotationConfigUtils.registerAnnotationConfigProcessors(beanFactory);
    -->
    <context:annotation-config />
    <bean id="bean1" class="top.jacktgq.Bean1" />
    <bean id="bean2" class="top.jacktgq.Bean2">
        <property name="bean1" ref="bean1"/>
    </bean>
</beans>
```

## 第三讲 `Bean`的生命周期和模板方法设计模式

### spring_03_bean_lifecycle

[p14 013-第三讲-bean生命周期](https://www.bilibili.com/video/BV1P44y1N7QG?p=14)

`springboot`项目启动类

```java
@SpringBootApplication
public class BeanLifeCycleApplication {
    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(BeanLifeCycleApplication.class, args);
        context.close();
    }
}
```

定义一个`LifeCycleBean`，加上`@Component`注解，再编写一些方法，给这些方法加上`Bean`的生命周期过程中的注解

```java
@Component
@Slf4j
public class LifeCycleBean {
    public LifeCycleBean() {
        log.debug("构造");
    }

    @Autowired
    public void autowire(@Value("${JAVA_HOME}") String name) {
        log.debug("依赖注入：{}", name);
    }

    @PostConstruct
    public void init() {
        log.debug("初始化");
    }

    @PreDestroy
    public void destroy() {
        log.debug("销毁");
    }
}
```

编写自定义`Bean`的后处理器，需要实现`InstantiationAwareBeanPostProcessor`和`DestructionAwareBeanPostProcessor`接口，并加上`@Component`注解，对`lifeCycleBean`的生命周期过程进行扩展。

```java
@Slf4j
@Component
public class MyBeanPostProcessor implements InstantiationAwareBeanPostProcessor, DestructionAwareBeanPostProcessor {
    @Override
    // 实例化前（即调用构造方法前）执行的方法
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        if (beanName.equals("lifeCycleBean"))
            log.debug("<<<<<<<<<<< 实例化前执行，如@PreDestroy");
        // 返回null保持原有对象不变，返回不为null，会替换掉原有对象
        return null;
    }

    @Override
    // 实例化后执行的方法
    public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        if (beanName.equals("lifeCycleBean")) {
            log.debug("<<<<<<<<<<< 实例化后执行，这里如果返回 false 会跳过依赖注入阶段");
            // return false;
        }

        return true;
    }

    @Override
    // 依赖注入阶段执行的方法
    public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {
        if (beanName.equals("lifeCycleBean"))
            log.debug("<<<<<<<<<<< 依赖注入阶段执行，如@Autowired、@Value、@Resource");
        return pvs;
    }

    @Override
    // 销毁前执行的方法
    public void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException {
        if(beanName.equals("lifeCycleBean"))
            log.debug("<<<<<<<<<<<销毁之前执行");
    }

    @Override
    // 初始化之前执行的方法
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if(beanName.equals("lifeCycleBean"))
            log.debug("<<<<<<<<<<< 初始化之前执行，这里返回的对象会替换掉原本的bean，如 @PostConstruct、@ConfigurationProperties");
        return bean;
    }

    @Override
    // 初始化之后执行的方法
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if(beanName.equals("lifeCycleBean"))
            log.debug("<<<<<<<<<<< 初始化之后执行，这里返回的对象会替换掉原本的bean，如 代理增强");
        return bean;
    }
}
```

运行结果如下：

![image-20220326140553213](http://jp88.top/image/image-20220326140553213.png)

[p15 014-第三讲-模板方法](https://www.bilibili.com/video/BV1P44y1N7QG?p=15)

```java
public class TestMethodTemplatePattern {
    public static void main(String[] args) {
        MyBeanFactory beanFactory = new MyBeanFactory();
        beanFactory.addBeanPostProcessor(bean -> System.out.println("解析 @Autowired"));
        beanFactory.addBeanPostProcessor(bean -> System.out.println("解析 @Resource"));
        beanFactory.getBean();
    }

    static class MyBeanFactory {
        public Object getBean() {
            Object bean = new Object();
            System.out.println("构造：" + bean);
            System.out.println("依赖注入：" + bean);
            for (BeanPostProcessor processor : processors) {
                processor.inject(bean);
            }
            System.out.println("初始化：" + bean);
            return bean;
        }

        private List<BeanPostProcessor> processors = new ArrayList<>();

        public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) {
            processors.add(beanPostProcessor);
        }
    }

    interface BeanPostProcessor {
        void inject(Object bean);
    }
}
```

## 第四讲 常见`Bean`后处理器以及`@Autowired`注解被解析的详细过程

### spring_04_beanpostprocessor

[p16 015-第四讲-常见bean后处理器1,2](https://www.bilibili.com/video/BV1P44y1N7QG?p=16)

[p17 016-第四讲-常见bean后处理器3](https://www.bilibili.com/video/BV1P44y1N7QG?p=17)

定义三个`Bean`，名称分别为`Bean1`，`Bean2`，`Bean3`，其中`Bean1`中依赖了`Bean2`和`Bean3`，`Bean2`通过`@Autowired`的注解注入，`Bean3`通过`@Resource`注解注入，再通过`@Value`注解注入一个`Java`的环境变量`JAVA_HOME`的值。最后定义两个方法`init()`和`destroy()`，分别加上`@PostConstruct`和`@PreDestroy`注解。

```java
@Slf4j
public class Bean1 {
    private Bean2 bean2;

    @Autowired
    public void setBean2(Bean2 bean2) {
        log.debug("@Autowired 生效：{}", bean2);
        this.bean2 = bean2;
    }

    @Autowired
    public void setJava_home(@Value("${JAVA_HOME}") String java_home) {
        log.debug("@Value 生效：{}", java_home);
        this.java_home = java_home;
    }

    private Bean3 bean3;

    @Resource
    public void setBean3(Bean3 bean3) {
        log.debug("@Resource 生效：{}", bean3);
        this.bean3 = bean3;
    }

    private String java_home;

    @PostConstruct
    public void init() {
        log.debug("@PostConstruct 生效：{}");
    }

    @PreDestroy
    public void destroy() {
        log.debug("@PreDestroy 生效：{}");
    }

    @Override
    public String toString() {
        return "Bean1{" +
                "bean2=" + bean2 +
                ", bean3=" + bean3 +
                ", java_home='" + java_home + '\'' +
                '}';
    }
}

public class Bean2 {

}

public class Bean3 {

}

@ConfigurationProperties(prefix = "java")
@Slf4j
public class Bean4 {
    private String home;
    private String version;
}
```

这里用`GenericApplicationContext` 来探究一下`@Autowired`、`@Value`、`@Resource`、`@PostConstruct`、`@PreDestroy`以及`springboot`项目中的`@ConfigurationProperties`这些注解分别是由哪个后处理器来解析的。

注：<font color="red">`GenericApplicationContext` 是一个【干净】的容器，默认不会添加任何后处理器，方便做测试，这里用`DefaultListableBeanFactory`也可以完成测试，只是会比使用`GenericApplicationContext`麻烦一些。</font>

测试代码如下：

```java
@Slf4j
public class TestBeanPostProcessor {
    @Test
    public void testBeanPostProcessor() throws Exception {
        // ⬇️GenericApplicationContext 是一个【干净】的容器，默认不会添加任何后处理器，方便做测试
        // 这里用DefaultListableBeanFactory也可以完成测试，只是会比使用GenericApplicationContext麻烦一些
        GenericApplicationContext context = new GenericApplicationContext();

        // 用原始方法注册三个Bean
        context.registerBean("bean1", Bean1.class);
        context.registerBean("bean2", Bean2.class);
        context.registerBean("bean3", Bean3.class);
        context.registerBean("bean4", Bean4.class);

        // 设置解析 @Value 注解的解析器
        context.getDefaultListableBeanFactory().setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
        // 添加解析 @Autowired 和 @Value 注解的后处理器
        context.registerBean(AutowiredAnnotationBeanPostProcessor.class);
        // 添加解析 @Resource、@PostConstruct、@PreDestroy 注解的后处理器
        context.registerBean(CommonAnnotationBeanPostProcessor.class);
        // 添加解析 @ConfigurationProperties注解的后处理器
        // ConfigurationPropertiesBindingPostProcessor后处理器不能像上面几种后处理器那样用context直接注册上去
        // context.registerBean(ConfigurationPropertiesBindingPostProcessor.class);
        // 需要反着来注册一下
        ConfigurationPropertiesBindingPostProcessor.register(context.getDefaultListableBeanFactory());


        // ⬇️初始化容器
        context.refresh();
        System.out.println(context.getBean(Bean4.class));

        // ⬇️销毁容器
        context.close();
    }
}
```

运行结果如下：

![image-20220327233426083](http://jp88.top/image/image-20220327233426083.png)

经过测试和运行结果的比对：

* `@Autowired`注解对应的后处理器是`AutowiredAnnotationBeanPostProcessor`；
* `@Value`注解需要配合`@Autowired`注解一起使用，所以也用到了`AutowiredAnnotationBeanPostProcessor`后处理器，然后`@Value`注解还需要再用到`ContextAnnotationAutowireCandidateResolver`解析器，否则会报错；
* `@Resource`、`@PostConstruct`、`@PreDestroy`注解对应的后处理器是`CommonAnnotationBeanPostProcessor`；
* `@ConfigurationProperties`注解对应的后处理器是`ConfigurationPropertiesBindingPostProcessor`。

[p18 017-第四讲-@Autowired bean后处理器执行分析](https://www.bilibili.com/video/BV1P44y1N7QG?p=18)

[p19 018-第四讲-@Autowired bean后处理器执行分析](https://www.bilibili.com/video/BV1P44y1N7QG?p=19)

本案例测试代码紧接着上面，这里对`Bean1`中加了`@Autowired`注解的属性注入`Bean2`、方法注入`Bean3`以及方法注入环境变量`JAVA_HOME`的过程进行分析。

`@Autowired`注解解析用到的后处理器是`AutowiredAnnotationBeanPostProcessor`

* 这个后处理器就是通过调用`postProcessProperties(PropertyValues pvs, Object bean, String beanName)`完成注解的解析和注入的功能
* 这个方法中又调用了一个私有的方法`findAutowiringMetadata(beanName, bean.getClass(), pvs)`，其返回值`InjectionMetadata`中封装了被`@Autowired`注解修饰的属性和方法
* 然后会调用`InjectionMetadata.inject(bean1, "bean1", null)`进行依赖注入
* 由于`InjectionMetadata.inject(bean1, "bean1", null)`的源码调用链过长，摘出主要调用过程进行说明：
* 成员变量注入，`InjectionMetadata`注入`Bean3`的过程：
  * `InjectionMetadata`会把`Bean1`中加了`@Autowired`注解的属性的`BeanName`先拿到，这里拿到的`BeanName`就是 `bean3`，然后再通过反射拿到这个属性，`Field bean3Field = Bean1.class.getDeclaredField("bean3");`
  * 将这个属性封装成一个`DependencyDescriptor`对象，再去调用`Bean3 bean3Value = (Bean3) beanFactory.doResolveDependency(dd1, null, null, null);`拿到`bean3Value`
  * 最后把值赋给这个属性`bean3Field.set(bean1, bean3Value);`
* 方法参数注入，`InjectionMetadata`注入`Bean2`的过程：
  * `InjectionMetadata`会把`Bean1`中加了`@Autowired`注解的方法的`MethodName`先拿到，这里拿到的`MethodName`就是 `setBean2`，然后再通过反射拿到这个方法，`Method setBean2 = Bean1.class.getDeclaredMethod("setBean2", Bean2.class);`
  * 将这个属性封装成一个`DependencyDescriptor`对象，再去调用`Bean2 bean2Value = (Bean2) beanFactory.doResolveDependency(dd2, "bean2", null, null);`拿到`bean2Value`
  * 最后调用方法`setBean2.invoke(bean1, bean2Value)`，给方法参数赋值。
* 方法参数注入，参数类型为String类型，且加上了@Value注解，`InjectionMetadata`注入环境变量`JAVA_HOME`的过程：
  * `InjectionMetadata`会把`Bean1`中加了`@Autowired`注解的方法的`MethodName`先拿到，这里拿到的`MethodName`就是 `setJava_home`，然后再通过反射拿到这个方法，`Method setJava_home = Bean1.class.getDeclaredMethod("setJava_home", String.class);`
  * 将这个属性封装成一个`DependencyDescriptor`对象，再去调用`String java_home = (String) beanFactory.doResolveDependency(dd3, null, null, null);`拿到`java_home` 
  * 最后调用方法`setJava_home.invoke(bean1, java_home);`，给方法参数赋值。

全部测试代码如下：

```java
@Slf4j
public class TestBeanPostProcessors {
    @Test
    public void testAutowiredAnnotationBeanPostProcessor() throws Throwable {
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        // 这里为了省事就不使用 beanFactory.registerBeanDefinition()方法去添加类的描述信息了
        // 直接使用 beanFactory.registerSingleton可以直接将Bean的单例对象注入进去，
        // 后面调用beanFactory.getBean()方法的时候就不会去根据Bean的定义去创建Bean的实例了，
        // 也不会有懒加载和依赖注入的初始化过程了。
        beanFactory.registerSingleton("bean2", new Bean2());
        beanFactory.registerSingleton("bean3", new Bean3());
        // 设置@Autowired注解的解析器
        beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
        // 设置解析 @Value 注解中的 ${} 表达式的解析器
        beanFactory.addEmbeddedValueResolver(new StandardEnvironment()::resolvePlaceholders);

        // 1. 查找哪些属性、方法加了 @Autowired，这称之为InjectionMetadata
        // 创建后处理器
        AutowiredAnnotationBeanPostProcessor processor = new AutowiredAnnotationBeanPostProcessor();
        // 后处理器在解析@Autowired和@Value的时候需要用到其他Bean，
        // 而BeanFactory提供了需要的Bean，所以需要把BeanFactory传给这个后处理器
        processor.setBeanFactory(beanFactory);
        // 创建Bean1
        Bean1 bean1 = new Bean1();
        System.out.println(bean1);
        // 解析@Autowired和@Value注解，执行依赖注入
        // PropertyValues pvs: 给注解的属性注入给定的值，这里不需要手动给定，传null即可
        // processor.postProcessProperties(null, bean1, "bean1");
        // postProcessProperties()方法底层原理探究
        // 通过查看源码得知 postProcessProperties()方法中调用了一个私有的方法findAutowiringMetadata(beanName, bean.getClass(), pvs); 会返回一个InjectionMetadata的对象，然后会调用InjectionMetadata.inject(bean1, "bean1", null)进行依赖注入
        // 通过反射调用一下
        /*Method findAutowiringMetadata = AutowiredAnnotationBeanPostProcessor.class.getDeclaredMethod("findAutowiringMetadata",String.class, Class.class, PropertyValues.class);
        findAutowiringMetadata.setAccessible(true);
        // 获取Bean1上加了@Value @Autowired注解的成员变量和方法参数信息
        InjectionMetadata metadata = (InjectionMetadata) findAutowiringMetadata.invoke(processor, "bean1", Bean1.class, null);
        System.out.println(metadata);

        // 2. 调用 InjectionMetaData 来进行依赖注入，注入时按类型查找值
        metadata.inject(bean1, "bean1", null);
        System.out.println(bean1);*/
        
        // 3. 如何去Bean工厂里面按类型查找值
        // 由于InjectionMetadata.inject(bean1, "bean1", null)的源码调用链过长，摘出主要调用过程进行演示

        // 3.1 @Autowired加在成员变量上，InjectionMetatadata给Bean1注入Bean3的过程
        // 通过InjectionMetadata把Bean1加了@Autowired注解的属性的BeanName先拿到，这里假设拿到的BeanName就是 bean3
        // 通过BeanName反射获取到这个属性，
        Field bean3Field = Bean1.class.getDeclaredField("bean3");
        // 设置私有属性可以被访问
        bean3Field.setAccessible(true);
        // 将这个属性封装成一个DependencyDescriptor对象
        DependencyDescriptor dd1 = new DependencyDescriptor(bean3Field, false);
        // 再执行beanFactory的doResolveDependency
        Bean3 bean3Value = (Bean3) beanFactory.doResolveDependency(dd1, null, null, null);
        System.out.println(bean3Value);
        // 给Bean1的成员bean3赋值
        bean3Field.set(bean1, bean3Value);
        System.out.println(bean1);

        // 3.2 @Autowired加在方法上，InjectionMetatadata给Bean1注入Bean2的过程
        Method setBean2 = Bean1.class.getDeclaredMethod("setBean2", Bean2.class);
        DependencyDescriptor dd2 = new DependencyDescriptor(new MethodParameter(setBean2, 0), true);
        Bean2 bean2Value = (Bean2) beanFactory.doResolveDependency(dd2, "bean2", null, null);
        System.out.println(bean2Value);
        // 给Bean1的setBean2()方法的参数赋值
        setBean2.invoke(bean1, bean2Value);
        System.out.println(bean1);

        // 3.3 @Autowired加在方法上，方法参数为String类型，加了@Value，
        // InjectionMetadata给Bean1注入环境变量JAVA_HOME属性的值
        Method setJava_home = Bean1.class.getDeclaredMethod("setJava_home", String.class);
        DependencyDescriptor dd3 = new DependencyDescriptor(new MethodParameter(setJava_home, 0), true);
        String java_home = (String) beanFactory.doResolveDependency(dd3, null, null, null);
        System.out.println(java_home);
        setJava_home.invoke(bean1, java_home);
        System.out.println(bean1);
    }
}
```

运行结果如下：

![image-20220329115607946](http://jp88.top/image/image-20220329115607946.png)

## 第五讲 常见Bean工厂后处理器以及模拟实现组件扫描

### spring_05_beanfactorypostprocessor

[p20 019-第五讲-常见工厂后处理器](https://www.bilibili.com/video/BV1P44y1N7QG?p=20)

定义`Bean1`、`Bean2`、`Mapper1`、`Mapper2`、`Config` 5个类，其中`Bean2`上面加上`@Component`和`@ComponentScan`注解，`Config`上加`@Component`注解，`Config`中通过`@Bean`注解定义`Bean1`，`Mapper1`、`Mapper2`上加`@Mapper`注解，类的定义参考下图，由于涉及的类比较多，具体代码可以去我的代码仓库获取。

![image-20220329142407664](http://jp88.top/image/image-20220329142407664.png)

这里用`GenericApplicationContext` 来探究一下`@Component`、`@ComponentScan`、`@Bean`、`@MapperScan`这些注解分别是由哪个后处理器来解析的。

测试代码如下：

```java
@Slf4j
public class TestBeanFactoryPostProcessors {
    @Test
    public void testBeanPostProcessors() throws IOException {
        // ⬇️GenericApplicationContext 是一个【干净】的容器，默认不会添加任何后处理器，方便做测试
        GenericApplicationContext context = new GenericApplicationContext();

        context.registerBean("config", Config.class);
        // 添加Bean工厂后处理器ConfigurationClassPostProcessor
        // 解析@ComponentScan、@Bean、@Import、@ImportResource注解
        context.registerBean(ConfigurationClassPostProcessor.class);
        // 添加Bean工厂后处理器MapperScannerConfigurer，解析@MapperScan注解
        context.registerBean(MapperScannerConfigurer.class, beanDefinition -> {
            // 指定扫描的包名
            beanDefinition.getPropertyValues().add("basePackage", "top.jacktgq.mapper");
        });

        // ⬇️初始化容器
        context.refresh();

        for (String name : context.getBeanDefinitionNames()) {
            System.out.println(name);
        }

        // ⬇️销毁容器
        context.close();
    }
}
```

经过测试和运行结果的比对：

* `@Component`、`@Bean`对应的`Bean`工厂后处理器是`ConfigurationClassPostProcessor`；
* `@MapperScan`对应的`Bean`工厂后处理器是`MapperScannerConfigurer`。

[p21 020-第五讲-工厂后处理器模拟实现-组件扫描](https://www.bilibili.com/video/BV1P44y1N7QG?p=21)

[p22 021-第五讲-工厂后处理器模拟实现-组件扫描](https://www.bilibili.com/video/BV1P44y1N7QG?p=22)

自定义组件扫描`Bean`工厂后处理器`CandyComponentScanPostProcessor`来解析`@Component`注解，代码如下：

```java
public class CandyAtComponentScanPostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanFactory) throws BeansException {
        try {
            ComponentScan componentScan = AnnotationUtils.findAnnotation(Config.class, ComponentScan.class);
            if (componentScan != null) {
                for (String basePage : componentScan.basePackages()) {
                    System.out.println(basePage);
                    // top.jacktgq.component -> classpath*:com/jacktgq/component/**/*.class
                    String path = "classpath*:" + basePage.replace('.', '/') + "/**/*.class";
                    // System.out.println(path);
                    CachingMetadataReaderFactory factory = new CachingMetadataReaderFactory();
                    AnnotationBeanNameGenerator generator = new AnnotationBeanNameGenerator();
                    for (Resource resource : new PathMatchingResourcePatternResolver().getResources(path)) {
                        // System.out.println(resource);
                        // 查看对应的类上是否有@Component注解
                        // System.out.println("分隔符>>>>>>>>>>>>>>>>>");
                        MetadataReader reader = factory.getMetadataReader(resource);
                        String className = reader.getClassMetadata().getClassName();
                        // System.out.println("类名：" + className);
                        String name = Component.class.getName();
                        // System.out.println("是否加了 @Component注解：" + reader.getAnnotationMetadata().hasAnnotation(name));
                        AnnotationMetadata annotationMetadata = reader.getAnnotationMetadata();
                        // System.out.println("是否加了@Component的派生注解：" + annotationMetadata.hasMetaAnnotation(name));

                        // 如果直接或者间接加了@Component注解
                        if (annotationMetadata.hasAnnotation(Component.class.getName()) || annotationMetadata.hasMetaAnnotation(name)) {
                            // 创建Bean的定义
                            AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition(className).getBeanDefinition();
                            
                            String beanName = generator.generateBeanName(beanDefinition, beanFactory);
                            // 将Bean定义加入工厂
                            beanFactory.registerBeanDefinition(beanName, beanDefinition);
                        }
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    // context.refresh()中会回调该方法
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
        
    }
}
```

测试代码：

```java
@Slf4j
public class TestBeanFactoryPostProcessors {
    @Test
    // 模拟实现组件扫描
    public void testMockComponentScan() throws Exception {
        // ⬇️GenericApplicationContext 是一个【干净】的容器，默认不会添加任何后处理器，方便做测试
        GenericApplicationContext context = new GenericApplicationContext();

        context.registerBean("config", Config.class);
        // 把自定义组件扫描Bean工厂后处理器加进来
        context.registerBean(CandyComponentScanPostProcessor.class);

        // ⬇️初始化容器
        context.refresh();

        for (String name : context.getBeanDefinitionNames()) {
            System.out.println(name);
        }

        // ⬇️销毁容器
        context.close();
    }
}
```

运行结果：

![image-20220329192201153](http://jp88.top/image/image-20220329192201153.png)

[p23 022-第五讲-工厂后处理器模拟实现-@Bean](https://www.bilibili.com/video/BV1P44y1N7QG?p=23)

自定义`Bean`工厂后处理器`CandyAtBeanPostProcessor`来解析`@Bean`注解，代码如下：

```java
public class CandyAtBeanPostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanFactory) throws BeansException {
        try {
            CachingMetadataReaderFactory factory = new CachingMetadataReaderFactory();
            MetadataReader reader = factory.getMetadataReader(new ClassPathResource("top/jacktgq/Config.class"));
            Set<MethodMetadata> annotatedMethods = reader.getAnnotationMetadata().getAnnotatedMethods(Bean.class.getName());
            for (MethodMetadata annotatedMethod : annotatedMethods) {
                System.out.println(annotatedMethod);
                String initMethod = annotatedMethod.getAnnotationAttributes(Bean.class.getName()).get("initMethod").toString();

                // 这里不需要指定类名了，因为最终的BeanDefinition是Config类中加了@Bean属性的方法的返回值的类型的定义。
                BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
                builder.setFactoryMethodOnBean(annotatedMethod.getMethodName(), "config");
                builder.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_CONSTRUCTOR);
                if (initMethod.length() > 0) {
                    builder.setInitMethodName(initMethod);
                }

                AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
                beanFactory.registerBeanDefinition(annotatedMethod.getMethodName(), beanDefinition);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {

    }
}
```

注：自定义解析 `@ComponentScan` 和 后面的 `@Bean` 注解的`Bean`工厂后处理器，实现`BeanDefinitionRegistryPostProcessor`接口而不是`BeanFactoryPostProcessor`接口，这么做的原因是：

* 实现`BeanFactoryPostProcessor`，`postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory)`方法，参数`configurableListableBeanFactory`工厂调用不了`registerBeanDefinition()`方法，需要做强制转换，转成`DefaultableBeanFactory`类型，才能调用`registerBeanDefinition()`方法。
* 实现`BeanDefinitionRegistryPostProcessor`，它里面除了`postProcessBeanFactory()`方法，还有一个`postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanFactory)`，参数`beanFactory`可以直接调用`registerBeanDefinition()`方法，避免了`ConfigurableListableBeanFactory`向`DefaultableBeanFactory`的强制转换。  

测试代码：

```java
public class TestBeanFactoryPostProcessors {
    @Test
    // 模拟实现@Bean注解的解析
    public void testMockAtBeanAnnotation() throws Exception {
        // ⬇️GenericApplicationContext 是一个【干净】的容器，默认不会添加任何后处理器，方便做测试
        GenericApplicationContext context = new GenericApplicationContext();

        context.registerBean("config", Config.class);
        context.registerBean(CandyAtBeanPostProcessor.class);

        // ⬇️初始化容器
        context.refresh();

        for (String name : context.getBeanDefinitionNames()) {
            System.out.println(name);
        }

        // ⬇️销毁容器
        context.close();
    }
}
```

运行结果：

![image-20220329224604315](http://jp88.top/image/image-20220329224604315.png)

[p25 024-第五讲-工厂后处理器模拟实现-Mapper](https://www.bilibili.com/video/BV1P44y1N7QG?p=25)

自定义`Bean`工厂后处理器`CandyAtMapperPostProcessor`来解析`@Mapper`注解，代码如下：

```java
public class CandyAtMapperPostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanFactory) throws BeansException {
        try {
            PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
            Resource[] resources = resolver.getResources("classpath:top/jacktgq/mapper/**/*.class");
            // Bean的名字生成器
            AnnotationBeanNameGenerator generator = new AnnotationBeanNameGenerator();
            CachingMetadataReaderFactory factory = new CachingMetadataReaderFactory();
            for (Resource resource : resources) {
                MetadataReader reader = factory.getMetadataReader(resource);
                ClassMetadata classMetadata = reader.getClassMetadata();
                if (classMetadata.isInterface()) {
                    AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition(MapperFactoryBean.class).addConstructorArgValue(classMetadata.getClassName()).setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE).getBeanDefinition();
                    // 这里不能使用名字生成器和MapperFactoryBean的BeanDefinition作为参数直接生成名字，
                    // 这样会导致多个相同的类型的对象因为名字一样产生覆盖的问题
                    // 解决办法 这里参考Spring源码的做法
                    // 用@Mapper注解修饰的接口的BeanDefinition作为参数生成名字
                    AbstractBeanDefinition bd = BeanDefinitionBuilder.genericBeanDefinition(classMetadata.getClassName()).getBeanDefinition();
                    String beanName = generator.generateBeanName(bd, beanFactory);
                    beanFactory.registerBeanDefinition(beanName, beanDefinition);
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {

    }
}
```

测试代码：

```java
@Slf4j
public class TestBeanFactoryPostProcessors {
    @Test
    // 模拟实现@Mapper注解的解析
    public void testMockAtMapperAnnotation() throws Exception {
        // ⬇️GenericApplicationContext 是一个【干净】的容器，默认不会添加任何后处理器，方便做测试
        GenericApplicationContext context = new GenericApplicationContext();

        context.registerBean("config", Config.class);
        // 先解析@Bean注解，把SqlSessionFactory加到Bean工厂里面
        context.registerBean(CandyAtBeanPostProcessor.class);
        // 解析Mapper接口
        context.registerBean(CandyAtMapperPostProcessor.class);

        // ⬇️初始化容器
        context.refresh();

        for (String name : context.getBeanDefinitionNames()) {
            System.out.println(name);
        }

        // ⬇️销毁容器
        context.close();
    }
}
```

运行结果：

![image-20220330012845575](http://jp88.top/image/image-20220330012845575.png)

## 第六讲 Aware和InitializingBean接口以及@Autowired注解失效分析

### spring_06_aware_initializingbean

[p26 025-第六讲-Aware与InitializingBean接口](https://www.bilibili.com/video/BV1P44y1N7QG?p=26)

`Aware` 接口用于注入一些与容器相关信息，例如：

​	a. `BeanNameAware` 注入 `Bean` 的名字

​	b. `BeanFactoryAware` 注入 `BeanFactory` 容器

​	c. `ApplicationContextAware` 注入 `ApplicationContext` 容器

​	d. `EmbeddedValueResolverAware` 注入 解析器，解析`${}`

定义一个`MyBean`类，实现`BeanNameAware`、`ApplicationContextAware`和`InitializingBean`接口并实现其方法，再定义两个方法，其中一个加@Autowired注解，注入ApplicationContext容器，另一个加@PostConstruct注解，具体代码如下：

```java
public class MyBean implements BeanNameAware, ApplicationContextAware, InitializingBean {
    @Override
    public void setBeanName(String name) {
        log.debug("当前bean：" + this + "，实现 BeanNameAware 调用的方法，名字叫：" + name);
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        log.debug("当前bean：" + this + "，实现 ApplicationContextAware 调用的方法，容器叫：" + applicationContext);
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        log.debug("当前bean：" + this + "，实现 InitializingBean 调用的方法，初始化");
    }

    @Autowired
    public void aaa(ApplicationContext applicationContext) {
        log.debug("当前bean：" + this +"，使用 @Autowired 容器是：" + applicationContext);
    }

    @PostConstruct
    public void init() {
        log.debug("当前bean：" + this + "，使用 @PostConstruct 初始化");
    }
}
```

测试代码：

```java
public class TestAwareAndInitializingBean {
    @Test
    public void testAware1() throws Exception {
        GenericApplicationContext context = new GenericApplicationContext();
        context.registerBean("myBean", MyBean.class);

        context.refresh();

        context.close();
    }
}
```

运行结果：

![image-20220330173209568](http://jp88.top/image/image-20220330173209568.png)

加了`@Autowired`和`@PostConstruct`注解的方法并没有被执行，而`Aware`和`InitializingBean`接口方法都被执行了。

修改测试代码，把解析`@Autowired`和`@PostConstruct`注解的`Bean`后处理加进来，然后再运行一下

```java
public class TestAwareAndInitializingBean {
    @Test
    public void testAware1() throws Exception {
        GenericApplicationContext context = new GenericApplicationContext();
        context.registerBean("myBean", MyBean.class);
        // 解析 @Autowired 注解的Bean后处理器
        context.registerBean(AutowiredAnnotationBeanPostProcessor.class);
        // 解析 @PostConstruct 注解的Bean后处理器
        context.registerBean(CommonAnnotationBeanPostProcessor.class);

        context.refresh();

        context.close();
    }
}
```

运行结果：

![image-20220330173807848](http://jp88.top/image/image-20220330173807848.png)

可以看到这下都执行了

有人可能会问：`b`、`c`、`d`的功能用 `@Autowired`注解就能实现啊，为啥还要用 `Aware` 接口呢？
`InititalizingBean` 接口可以用 `@PostConstruct`注解实现，为啥还要用`InititalizingBean`呢？
简单地说：

* `@Autowired` 和`@PostConstruct`注解的解析需要用到 `Bean` 后处理器，属于扩展功能，而 `Aware` 接口属于内置功能，不加任何扩展，`Spring`就能识别；

* 某些情况下，扩展功能会失效，而内置功能不会失效

  [p27 026-第六讲-@Autowired失效分析](https://www.bilibili.com/video/BV1P44y1N7QG?p=27)

  * 例1：比如没有把解析`@Autowired`和`@PostStruct`注解的`Bean`的后处理器加到`Bean`工厂中，你会发现用 `Aware` 注入 `ApplicationContext` 成功， 而 `@Autowired` 注入 `ApplicationContext` 失败

  * 例2：定义两个`Java Config`类（类上加`@Configuration`注解），名字分别叫`MyConfig1`和`MyConfig2`，都实现注入`ApplicationContext`容器和初始化功能，`MyConfig1`用`@Autowired`和`@PostConstruct`注解实现，`MyConfig2`用实现`Aware`和`InitializingBean`接口的方式实现，另外，两个`Config`类中都通过`@Bean`注解的方式注入一个`BeanFactoryPostProcessor`，代码如下：

    `MyConfig1`:

    ```java
    @Slf4j
    public class MyConfig1 {
        @Autowired
        public void setApplicationContext(ApplicationContext applicationContext) {
            log.debug("注入 ApplicationContext");
        }
    
        @PostConstruct
        public void init() {
            log.debug("初始化");
        }
    
        @Bean
        public BeanFactoryPostProcessor processor1() {
            return beanFactory -> {
                log.debug("执行 processor1");
            };
        }
    }
    ```

    测试代码：

    ```java
    @Slf4j
    public class TestAwareAndInitializingBean {
        @Test
        public void testAware_MyConfig1() {
            GenericApplicationContext context = new GenericApplicationContext();
            // MyConfig1没有加上@
            context.registerBean("myConfig1", MyConfig1.class);
            // 解析 @Autowired 注解的Bean后处理器
            context.registerBean(AutowiredAnnotationBeanPostProcessor.class);
            // 解析 @PostConstruct 注解的Bean后处理器
            context.registerBean(CommonAnnotationBeanPostProcessor.class);
            // 解析@ComponentScan、@Bean、@Import、@ImportResource注解的后处理器
            // 这个后处理器不加出不来效果
            context.registerBean(ConfigurationClassPostProcessor.class);
    
            // 1. 添加beanfactory后处理器；2. 添加bean后处理器；3. 初始化单例。
            context.refresh();
    
            context.close();
    }
    ```

    运行结果：

    ![image-20220331090139965](http://jp88.top/image/image-20220331090139965.png)

    `MyConfig2`:

    ```java
    @Slf4j
    public class MyConfig2 implements ApplicationContextAware, InitializingBean {
        @Override
        public void setApplicationContext(ApplicationContext applicationContext) {
            log.debug("注入 ApplicationContext");
        }
    
        @Override
        public void afterPropertiesSet() throws Exception {
            log.debug("初始化");
        }
    
        @Bean
        public BeanFactoryPostProcessor processor1() {
            return beanFactory -> {
                log.debug("执行 processor1");
            };
        }
    }
    ```

    测试代码：

    ```java
    @Slf4j
    public class TestAwareAndInitializingBean {
        @Test
        public void testAutowiredAndInitializingBean_MyConfig2() {
            GenericApplicationContext context = new GenericApplicationContext();
            context.registerBean("myConfig2", MyConfig2.class);
    
            // 1. 添加beanfactory后处理器；2. 添加bean后处理器；3. 初始化单例。
            context.refresh();
    
            context.close();
        }
    }
    ```

    运行结果：

    ![image-20220331090537999](http://jp88.top/image/image-20220331090537999.png)

    Java配置类在添加了 `bean` 工厂后处理器后，你会发现用传统接口方式的注入和初始化依然成功，而 `@Autowired` 和 `@PostConstruct` 的注入和初始化失败。

    那是什么原因导致的呢？

    配置类 `@Autowired` 注解失效分析

    * Java 配置类不包含 `BeanFactoryPostProcessor` 的情况

      ```mermaid
      sequenceDiagram
      participant ac as ApplicationContext
      participant bfpp as BeanFactoryPostProcessor
      participant bpp as BeanPostProcessor
      participant config as Java配置类
      ac ->> bfpp : 1. 执行 BeanFactoryPostProcessor
      ac ->> bpp : 2. 注册 BeanPostProcessor
      ac ->> +config : 3. 创建和初始化
      bpp ->> config : 3.1 依赖注入扩展（如 @Value 和 @Autowired）
      bpp ->> config : 3.2 初始化扩展（如 @PostConstruct)
      ac ->> config : 3.3 执行 Aware 及 InitializingBean
      config -->> -ac : 3.4 创建成功
      ```

      

    * `Java` 配置类包含 `BeanFactoryPostProcessor` 的情况

      根据上面的时序图可以得知，正常情况下，`BeanFactoryPostProcessor`会在`Java`配置类初始化之前执行，而Java配置类里面却定义了一个`BeanFactoryPostProcessor`，要创建其中的 `BeanFactoryPostProcessor` ，必须提前创建 `Java` 配置类，这样`BeanFactoryPostProcessor`就会在Java配置类初始化后执行了，而此时的 `BeanPostProcessor` 还未准备好，导致 `@Autowired` 等注解失效。

      ```mermaid
      sequenceDiagram 
      participant ac as ApplicationContext
      participant bfpp as BeanFactoryPostProcessor
      participant bpp as BeanPostProcessor
      participant config as Java配置类
      ac ->> +config : 3. 创建和初始化
      ac ->> config : 3.1 执行 Aware 及 InitializingBean
      config -->> -ac : 3.2 创建成功
      
      ac ->> bfpp : 1. 执行 BeanFactoryPostProcessor
      ac ->> bpp : 2. 注册 BeanPostProcessor
      ```

总结：

* `Aware` 接口提供了一种【内置】 的注入手段，可以注入 `BeanFactory`，`ApplicationContext`；
* `InitializingBean` 接口提供了一种 【内置】 的初始化手段；
* 内置的注入和初始化不收扩展功能的影响，总会被执行，因此 `spring` 框架内部的类常用它们。

## 第七讲 Bean的初始化与销毁

### spring_07_init_destroy

[p28 027-第七讲-初始化与销毁](https://www.bilibili.com/video/BV1P44y1N7QG?p=28)

定义`Bean1`类，实现`InitializingBean`接口和对应的接口方法`afterPropertiesSet()`，再定义`init1()`方法，在方法上加`@PostConstruct`注解，最后定义`init3()`

```java
@Slf4j
public class Bean1 implements InitializingBean {
    @PostConstruct
    public void init1() {
        log.debug("初始化1，@PostConstruct");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        log.debug("初始化2，InitializingBean接口");
    }

    public void init3() {
        log.debug("初始化3，@Bean的initMethod");
    }
}
```

定义`Bean2`类，实现`DisposableBean`接口和对应的接口方法`destroy()`，再定义`destroy1()`方法，在方法上加`@PreDestroy`注解，最后定义`init3()`

```java
@Slf4j
public class Bean2 implements DisposableBean {
    @PreDestroy
    public void destroy1() {
        log.debug("销毁1，@PreDestory");
    }

    @Override
    public void destroy() throws Exception {
        log.debug("销毁2，DisposableBean接口");
    }

    public void destroy3() {
        log.debug("销毁3，@Bean的destroyMethod");
    }
}
```

定义`Config`类，类上加`@Configuration`注解，类中通过`@Bean`注解把`Bean1`和`Bean2`加到`Bean`工厂中，分别在`@Bean`注解中指定`initMethod = "init3"`,`destroyMethod = "destroy"`

```java
@Configuration
public class Config {
    @Bean(initMethod = "init3")
    public Bean1 bean1() {
        return new Bean1();
    }

    @Bean(destroyMethod = "destroy3")
    public Bean2 bean2() {
        return new Bean2();
    }
}
```

编写测试代码，观察三个初始化方法和三个销毁方法的执行顺序

```java
@Slf4j
public class TestInitAndDestroy {
    @Test
    public void testInitAndDestroy() throws Exception {
        // ⬇️GenericApplicationContext 是一个【干净】的容器，这里只是为了看初始化步骤，就不用springboot启动类进行演示了
        GenericApplicationContext context = new GenericApplicationContext();
        context.registerBean("config", Config.class);
        // 解析@PostConstruct注解的bean后处理器
        context.registerBean(CommonAnnotationBeanPostProcessor.class);
        // 解析@Configuration、@Component、@Bean注解的bean工厂后处理器
        context.registerBean(ConfigurationClassPostProcessor.class);

        context.refresh();
        context.close();
    }
}
```

运行结果：

![image-20220401013108042](http://jp88.top/image/image-20220401013108042.png)

可以看到，spring提供了多种初始化和销毁手段

* 对于`init`，三个初始化方法的执行顺序是

  `@PostConstruct` -> `InitializingBean`接口 -> `@Bean`的`initMethod`

* 对于`destory`, 三个销毁方法的执行顺序是

  `@PreDestroy` -> `DisposableBean`接口 -> `@Bean`的`destroy`

## 第八讲 Scope类型、注意事项、销毁和失效分析

### spring_08_scope

[p29 028-第八讲-Scope](https://www.bilibili.com/video/BV1P44y1N7QG?p=29)

`spring`的`scope`类型：

* `singleton`：单例
* `prototype`：多例
* `request`：`web`请求
* `session`：`web`的会话
* `application`：`web`的`ServletContext`

测试`scope`类型中的`request`、`session`、`application`

定义**`BeanForRequest`**类，加上`@Component`和`@Scope`注解，指定`Scope`类型为**`request`**，在类型中定义`destroy()`方法，方法上加`@PreDestory`注解，代码如下：

```java
@Slf4j
@Scope("request")
@Component
public class BeanForRequest {
    @PreDestroy
    public void destory() {
        log.debug("destroy");
    }
}
```

定义**`BeanForSession`**类，加上`@Component`和`@Scope`注解，指定`Scope`类型为**`session`**，在类型中定义`destroy()`方法，方法上加`@PreDestory`注解，代码如下：

```java
@Slf4j
@Scope("request")
@Component
public class BeanForRequest {
    @PreDestroy
    public void destory() {
        log.debug("destroy");
    }
}
```

定义**`BeanForApplication`**类，加上`@Component`和`@Scope`注解，指定`Scope`类型为**`application`**，在类型中定义`destroy()`方法，方法上加`@PreDestory`注解，代码如下：

```java
@Slf4j
@Scope("request")
@Component
public class BeanForRequest {
    @PreDestroy
    public void destory() {
        log.debug("destroy");
    }
}
```

编写一个`MyController`类，加上`@RestController`注解，在该类中通过`@Autowired`注解注入`BeanForRequest`、`BeanForSession`和`BeanForApplication`的实例，需要注意，这里还需要加`@Lazy`注解（至于原因后面会解释），否则会导致`@Scope`域失效，再定义一个方法`tes()`，加上`@GetMapping`注解，用于响应一个`http`请求，在`test()`方法中，打印`beanForRequest`、`beanForSession`和`beanForApplication`，代码如下：

```java
@RestController
public class MyController {
    @Lazy
    @Autowired
    private BeanForRequest beanForRequest;

    @Lazy
    @Autowired
    private BeanForSession beanForSession;

    @Lazy
    @Autowired
    private BeanForApplication beanForApplication;

    @GetMapping(value = "/test", produces = "text/html")
    public String test(HttpServletRequest request, HttpSession session) {
        // ServletContext sc = request.getServletContext();
        String result = "<ul>" +
                        "<li>request scope: " +  beanForRequest + "</li>" +
                        "<li>session scope: " +  beanForSession + "</li>" +
                        "<li>application scope: " +  beanForApplication + "</li>" +
                        "</ul>";
        return result;
    }
}
```

`springboot`启动类

```java
@Slf4j
@SpringBootApplication
public class ScopeApplicationContext {
    public static void main(String[] args) throws InterruptedException {
        testRequest_Session_Application_Scope();
    }

    // 演示 request, session, application作用域
    private static void testRequest_Session_Application_Scope() throws InterruptedException {
        ConfigurableApplicationContext context = SpringApplication.run(ScopeApplicationContext.class);
    }
}
```

启动后，用谷歌浏览器访问 <http://localhost:8080/test>，

浏览器运行结果如下：

![image-20220401222141258](http://jp88.top/image/image-20220401222141258.png)

再刷新一下当前页，查看运行结果：

![image-20220401222154488](http://jp88.top/image/image-20220401222154488.png)

控制台运行结果如下：

![image-20220401214139467](http://jp88.top/image/image-20220401214139467.png)

可以看到两次刷新只有`BeanForRequest`对象发生了改变，这是由于`scope`为`request`类型的对象，会在请求结束后销毁，再来一次请求就会重新创建，请求结束后又会销毁。

接下来我们换个`Edge`浏览器访问 <http://localhost:8080/test>，对比两个浏览器的显示结果：

![image-20220401222256653](http://jp88.top/image/image-20220401222256653.png)

可以看到这回除了`BeanForRequest`对象不同，`BeanForSession`对象也不同了，这是因为开一个新的浏览器会创建一个新的会话，所以`BeanForSession`对象也不同了。

继续进行测试，在`application.properties`配置一个属性`server.servlet.session.timeout=10s`，这个属性的默认值为`30`分钟，这样`10s`没有操作浏览器的话就会销毁对应`session`，不过经过测试这个这个属性最少为`1`分钟，低于1分钟一律按照1分钟算。具体原理看这篇博客：<https://www.jianshu.com/p/9d91cca74082>，里面进行了源码级别的分析。

设置好之后，重启项目， 然后去浏览器访问，1分钟后控制台会打印`session`被销毁，如下图所示：

![image-20220402144856449](http://jp88.top/image/image-20220402144856449.png)

那什么时候`scope`为`application`的对象`BeanForApplication`会销毁呢？按理说应该是在`SpringBoot`程序结束，也即内置的`Tomcat`服务器停止的时候调用，但是经过测试：无论是在控制台停止`SpringBoot`项目，还是调用`ApplicationContext`的`close()`方法，都没有调用`BeanForApplication`的销毁方法，有知道什么方法可以让它调用的，请评论区告知，谢谢！！！

[p30 029-第八讲-Scope失效解决1,2](https://www.bilibili.com/video/BV1P44y1N7QG?p=30)

[p31 030-第八讲-Scope失效解决3,4](https://www.bilibili.com/video/BV1P44y1N7QG?p=31)

定义1个单例类`SingletonBean`，指定它的`Scope`为`singleton`，定义`5`个多例类，`PrototypeBean`、`PrototypeBean1`、`PrototypeBean2`、`PrototypeBean3`、`PrototypeBean4`，将这`5`个多例类的对象注入到`SingletonBean`的`5`个属性中，其中`PrototypeBean`用来演示多例对象注入到单例对象中`Scope`失效的情况，其他四个类的对象用来演示四种解决`Scope`失效的方法。相关类的定义如下：

`SingletonBean`

```java
@Scope("singleton")
@Component
public class SingletonBean {
    @Autowired
    private PrototypeBean prototypeBean;

    @Lazy
    @Autowired
    private PrototypeBean1 prototypeBean1;

    @Autowired
    private PrototypeBean2 prototypeBean2;

    @Autowired
    private ObjectFactory<PrototypeBean3> factory;

    @Autowired
    private ApplicationContext context;

    public PrototypeBean getPrototypeBean() {
        return prototypeBean;
    }

    public PrototypeBean1 getPrototypeBean1() {
        return prototypeBean1;
    }

    public PrototypeBean2 getPrototypeBean2() {
        return prototypeBean2;
    }

    public PrototypeBean3 getPrototypeBean3() {
        return factory.getObject();
    }

    public PrototypeBean4 getPrototypeBean4() {
        return context.getBean(PrototypeBean4.class);
    }
}
```

`PrototypeBean`

```java
@Scope("prototype")
@Component
public class PrototypeBean {
}
```

`PrototypeBean1`

```java
@Scope("prototype")
@Component
public class PrototypeBean1 {
}
```

`PrototypeBean2`

```java
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
@Component
public class PrototypeBean2 {
}
```

`PrototypeBean3`

```java
@Scope("prototype")
@Component
public class PrototypeBean3 {
}
```

`PrototypeBean4`

```java
@Scope("prototype")
@Component
public class PrototypeBean4 {
}
```

测试代码：

```java
@Slf4j
@SpringBootApplication
public class ScopeApplicationContext {
    public static void main(String[] args) throws Exception {
        testSingletonPrototypeInvalidAndSolve();
    }

    // 演示单例中注入多例失效的情况，以及解决失效问题的方法
    private static void testSingletonPrototypeInvalidAndSolve() {
        ConfigurableApplicationContext context = SpringApplication.run(ScopeApplicationContext.class);
        SingletonBean singletonBean = context.getBean(SingletonBean.class);
        // 单例中注入多例失效的情况
        log.debug("{}", singletonBean.getPrototypeBean().getClass());
        log.debug("{}", singletonBean.getPrototypeBean());
        log.debug("{}", singletonBean.getPrototypeBean());
        log.debug("{}", singletonBean.getPrototypeBean());
        System.out.println("------------------------------------");

        // 解决方法1：在SingletonBean的PrototypeBean1属性上加@Lazy注解
        log.debug("{}", singletonBean.getPrototypeBean1().getClass());
        log.debug("{}", singletonBean.getPrototypeBean1());
        log.debug("{}", singletonBean.getPrototypeBean1());
        log.debug("{}", singletonBean.getPrototypeBean1());
        System.out.println("------------------------------------");

        // 解决方法2：在PrototypeBean2的类上的@Scope注解多配置一个属性，如，@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
        log.debug("{}", singletonBean.getPrototypeBean2().getClass());
        log.debug("{}", singletonBean.getPrototypeBean2());
        log.debug("{}", singletonBean.getPrototypeBean2());
        log.debug("{}", singletonBean.getPrototypeBean2());
        System.out.println("------------------------------------");

        // 解决方法3：使用ObjectFactory<PrototypeBean3>工厂类，在每次调用getProtypeBean3()方法中返回factory.getObject()
        log.debug("{}", singletonBean.getPrototypeBean3().getClass());
        log.debug("{}", singletonBean.getPrototypeBean3());
        log.debug("{}", singletonBean.getPrototypeBean3());
        log.debug("{}", singletonBean.getPrototypeBean3());
        System.out.println("------------------------------------");

        // 解决方法4：在SingletonBean中注入一个ApplicationContext，使用context.getBean(PrototypeBean4.class)获取对应的多例
        log.debug("{}", singletonBean.getPrototypeBean4().getClass());
        log.debug("{}", singletonBean.getPrototypeBean4());
        log.debug("{}", singletonBean.getPrototypeBean4());
        log.debug("{}", singletonBean.getPrototypeBean4());
        System.out.println("------------------------------------");

    }
}
```

运行结果如下：

![image-20220406141812580](http://jp88.top/image/image-20220406141812580.png)

## 第九讲 aop之ajc增强

`aop`是`spring`框架中非常重要的功能，其主要实现通常情况下是动态代理，但是这个说法并不全面，还有另外两种实现：

* `ajc`编译器
* `agent`类加载

### spring_09_aop_ajc

[p32 031-第九讲-aop之ajc增强](https://www.bilibili.com/video/BV1P44y1N7QG?p=32)

先看`aop`的第一种实现`ajc`编译器代码增强，这是一种编译时的代码增强。

新建一个普通的maven项目

* 添加依赖

  使用`ajc`编译器进行代码增强，首先需要在`pom.xml`文件中加入`ajc`编译器插件依赖

  ```xml
  <build>
      <plugins>
          <plugin>
              <groupId>org.codehaus.mojo</groupId>
              <artifactId>aspectj-maven-plugin</artifactId>
              <version>1.11</version>
              <configuration>
                  <complianceLevel>1.8</complianceLevel>
                  <source>8</source>
                  <target>8</target>
                  <showWeaveInfo>true</showWeaveInfo>
                  <verbose>true</verbose>
                  <Xlint>ignore</Xlint>
                  <encoding>UTF-8</encoding>
              </configuration>
              <executions>
                  <execution>
                      <goals>
                          <!-- use this goal to weave all your main classes -->
                          <goal>compile</goal>
                          <!-- use this goal to weave all your test classes -->
                          <goal>test-compile</goal>
                      </goals>
                  </execution>
              </executions>
          </plugin>
      </plugins>
  </build>
  ```

  加入`aspectjweaver`的依赖

  ```xml
  <dependency>
      <groupId>org.aspectj</groupId>
      <artifactId>aspectjweaver</artifactId>
      <version>1.9.7</version>
  </dependency>
  ```

  加入日志和单元测试的依赖

  ```xml
  <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
      <version>1.7.36</version>
  </dependency>
  <dependency>
      <groupId>ch.qos.logback</groupId>
      <artifactId>logback-classic</artifactId>
      <version>1.2.10</version>
  </dependency>
  <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.13.2</version>
  </dependency>
  ```

* 需要增强的类`MyService`

  ```java
  public class MyService {
      private static final Logger log = LoggerFactory.getLogger(MyAspect.class);
      public void foo() {
          log.debug("foo()");
      }
  }
  ```

* 切面类`MyAspect`，编写`execution`表达式，对`MyService`类的`foo()`方法进行增强

  ```java
  @Aspect // ⬅️注意此切面并未被 Spring 管理，本项目pom文件中根本没有引入spring的相关类
  public class MyAspect {
      private static final Logger log = LoggerFactory.getLogger(MyAspect.class);
      @Before("execution(* top.jacktgq.service.MyService.foo())")
      public void before() {
          log.debug("before()");
      }
  }
  ```

* 测试代码

  ```java
  public class Aop_Aspectj_Test {
      @Test
      public void testAopAjc() {
          new MyService().foo();
      }
  }
  ```

* 编译项目，这里需要使用`maven`来编译，打开`idea`中的`maven`面板，点击`compile`

  ![image-20220410183011468](http://jp88.top/image/image-20220410183011468.png)

  然后再运行测试代码，可以看到创建`MyService`对象并调用`foo()`方法会先执行切面类中的`before()`方法

  ![image-20220410183108780](http://jp88.top/image/image-20220410183108780.png)

  注：

  * 有些小伙伴可能会遇到问题：明明按照一样的步骤来操作，可是运行以后代码并没有增强。<font color="red">这是由于`idea`中在执行代码之前会默认编译一遍代码，这本来是正常的，可是，如果使用`maven`来编译代码，会在执行代码前将`maven`编译的代码覆盖，这就会导致`maven`的`ajc`编译器增强的代码被覆盖，所以会看不到最终的运行效果。</font>

  * 解决办法：在设置中将自动构建项目的选项勾上，就不会出现多次编译覆盖的问题了。

    ![image-20220410183955137](http://jp88.top/image/image-20220410183955137.png)

总结：

* 可以看到没有引入任何跟`spring`框架相关的包，`MyService`类是通过直接`new()`的方式获得的，所以也就不存在使用了动态代理的说法了

* 打开编译后的`MyService.class`文件，双击以后idea会反编译该字节码文件，可以看到`foo()`方法体的开头加了一行代码，这就是增强的代码，这是`ajc`编译器在编译`MyService`类的时候为我们添加的代码，这是一种编译时的增强。

  ![image-20220410184651224](http://jp88.top/image/image-20220410184651224.png)

## 第十讲 aop之agent增强

### spring_10_aop_agent

[p33 032-第十讲-aop之agent增强](https://www.bilibili.com/video/BV1P44y1N7QG?p=33)

现在来看`aop`的另外一种实现`agent`增强，这是一种类加载时的代码增强。

* 新建一个普通的maven项目

  * 加入`aspectjweaver`的依赖

    ```xml
    <dependency>
        <groupId>org.aspectj</groupId>
        <artifactId>aspectjweaver</artifactId>
        <version>1.9.7</version>
    </dependency>
    ```

    加入日志和单元测试的依赖

    ```xml
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>1.7.36</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.2.10</version>
    </dependency>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.13.2</version>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.22</version>
    </dependency>
    ```

  * 需要增强的类`MyService`

    ```java
    @Slf4j
    public class MyService {
        public void foo() {
            log.debug("foo()");
            bar();
        }
        public void bar() {
            log.debug("bar()");
        }
    }
    ```

  * 切面类`MyAspect`，编写`execution`表达式，对`MyService`类的`foo()`方法进行增强

    ```java
    @Aspect // ⬅️注意此切面并未被 Spring 管理，本项目pom文件中根本没有引入spring的相关类
    @Slf4j
    public class MyAspect {
        @Before("execution(* top.jacktgq.service.MyService.*())")
        public void before() {
            log.debug("before()");
        }
    }
    ```

  * 测试代码

    ```java
    public class Aop_agent_Test {
        @Test
        public void testAopAgent() throws Exception {
            MyService myService = new MyService();
            myService.foo();
            System.in.read();
        }
    }
    ```

    运行时需要在 VM options 里加入 `-javaagent:D:\同步空间\repository\org\aspectj\aspectjweaver\1.9.7\aspectjweaver-1.9.7.jar`把其中 `D:\同步空间\repository` 改为你自己 `maven` 仓库起始地址

    ![image-20220410210057854](http://jp88.top/image/image-20220410210057854.png)

    注：还需要在`resources/META-INF`目录下建一个`aop.xml`配置文件，内容如下，`aspectj`会自动扫描到这个配置文件，不加这个配置文件不会出效果。

    ![image-20220410210354881](http://jp88.top/image/image-20220410210354881.png)

    ```xml
    <aspectj>
        <aspects>
            <aspect name="top.jacktgq.aop.MyAspect"/>
            <weaver options="-verbose -showWeaveInfo">
                <include within="top.jacktgq.service.MyService"/>
                <include within="top.jacktgq.aop.MyAspect"/>
            </weaver>
        </aspects>
    </aspectj>
    ```

    运行测试代码，可以看到创建`MyService`对象并调用`foo()`方法会先执行切面类中的`before()`方法

    ![image-20220410210458098](http://jp88.top/image/image-20220410210458098.png)

    注：

    * 有些小伙伴可能会遇到问题：明明按照一样的步骤来操作，可是运行以后代码并没有增强。<font color="red">这是由于`idea`中在执行代码之前会默认编译一遍代码，这本来是正常的，可是，如果使用`maven`来编译代码，会在执行代码前将`maven`编译的代码覆盖，这就会导致`maven`的`ajc`编译器增强的代码被覆盖，所以会看不到最终的运行效果。</font>

    * 解决办法：在设置中将自动构建项目的选项勾上，就不会出现多次编译覆盖的问题了。

      ![image-20220410183955137](http://jp88.top/image/image-20220410183955137.png)

  总结：

  * 可以看到没有引入任何跟`spring`框架相关的包，`MyService`类是通过直接`new()`的方式获得的，所以也就不存在使用了动态代理的说法了

  * 打开编译后的`MyService.class`文件，双击以后idea会反编译该字节码文件，可以看到`foo()`方法体中并没有添加多余的代码，所以就不是编译时增强了，而是类加载的时候增强的，这里可以借助阿里巴巴的Arthas工具，下载地址：<https://arthas.aliyun.com/doc/en/download.html>，解压以后进入到arthas的bin目录下，启动黑窗口，输入`java -jar .\arthas-boot.jar`，在输出的`java`进程列表里面找到我们要连接的进程，输入对应进程的序号，我这里是`4`，连接上以后会打印`ARTHAS`的`logo`

    ![image-20220410235629087](http://jp88.top/image/image-20220410235629087.png)

    再输入`jad top.jacktgq.service.MyService`反编译内存中的`MyService`类

    ![image-20220410235616607](http://jp88.top/image/image-20220410235616607.png)

    可以看到`foo()`和`bar()`方法体的第一行都加了一行代码，这就说明通过添加虚拟机参数`-javaagent`的方式可以在类加载的时候对代码进行增强。 

## 第十一讲 aop之proxy增强-jdk和cglib

### spring_11_aop_proxy_jdk_cglib

[p34 033-第十一讲-aop之proxy增强-jdk](https://www.bilibili.com/video/BV1P44y1N7QG?p=34)

测试代码

```java
public class AopJdkProxyTest {
    @Test
    public void testJdkProxy() {
        // jdk的动态代理，只能针对接口代理
        // 目标对象
        Target target = new Target();
        // 用来加载在运行期间动态生成的字节码
        ClassLoader loader = AopJdkProxyTest.class.getClassLoader();
        Foo fooProxy = (Foo) Proxy.newProxyInstance(loader, new Class[]{Foo.class}, (proxy, method, args) -> {
            System.out.println("before...");
            Object result = method.invoke(target, args);
            System.out.println("after...");
            return result;  // 让代理也返回目标方法执行的结果
        });

        fooProxy.foo();
    }
}

interface Foo {
    void foo();
}

@Slf4j
final class Target implements Foo {
    public void foo() {
        log.debug("target foo");
    }
}
```

运行结果

![image-20220411083443041](http://jp88.top/image/image-20220411083443041.png)

`jdk`动态代理总结：

1. 代理对象和目标对象是兄弟关系，都实现了`Foo`接口，代理对象类型不能强转成目标对象类型；
2. 目标类定义的时候可以加`final`修饰。

[p35 034-第十一讲-aop之proxy增强-cglib](https://www.bilibili.com/video/BV1P44y1N7QG?p=35)

```java
public class AopCglibProxyTest {
    @Test
    public void testCglibProxy1() {
        // 目标对象
        Target target = new Target();
        Foo fooProxy = (Foo) Enhancer.create(Target.class, (MethodInterceptor) (obj, method, args, proxy) -> {
            System.out.println("before...");
            Object result = method.invoke(target, args); // 用方法反射调用目标
            System.out.println("after...");
            return result;
        });
        System.out.println(fooProxy.getClass());

        fooProxy.foo();
    }

    @Test
    public void testCglibProxy2() {
        // 目标对象
        Target target = new Target();
        Foo fooProxy = (Foo) Enhancer.create(Target.class, (MethodInterceptor) (obj, method, args, proxy) -> {
            System.out.println("before...");
            // proxy 它可以避免反射调用
            Object result = proxy.invoke(target, args); // 需要传目标类
            System.out.println("after...");
            return result;
        });
        System.out.println(fooProxy.getClass());

        fooProxy.foo();
    }

    @Test
    public void testCglibProxy3() {
        // 目标对象
        Foo fooProxy = (Foo) Enhancer.create(Target.class, (MethodInterceptor) (obj, method, args, proxy) -> {
            System.out.println("before...");
            // proxy 它可以避免反射调用
            Object result = proxy.invokeSuper(obj, args); // 不需要目标类，需要代理自己
            System.out.println("after...");
            return result;
        });
        System.out.println(fooProxy.getClass());

        fooProxy.foo();
    }
}

interface Foo {
    void foo();
}

@Slf4j
class Target implements Foo {
    public void foo() {
        log.debug("target foo");
    }
}
```

运行结果

![image-20220411085629883](http://jp88.top/image/image-20220411085629883.png)

`cglib`动态代理总结：

1. `MethodInterceptor`的`intercept()`方法的第2个参数是`method`，可以通过反射对目标方法进行调用

   ```java
   Object result = method.invoke(target, args); // 用方法反射调用目标
   ```

2. 第4个参数`proxy`，可以不用反射就能对目标方法进行调用；

   ```java
   Object result = proxy.invoke(target, args); // 需要传目标类 （spring用的是这种）
   // 或者
   Object result = proxy.invokeSuper(obj, args); // 不需要目标类，需要代理自己
   ```

3. 代理类不需要实现接口；

4. 代理对象和目标对象是父子关系，代理类继承于目标类；

5. 目标类定义的时候不能加`final`修饰，否则代理类就无法继承目标类了，会报`java.lang.IllegalArgumentException: Cannot subclass final class top.jacktgq.proxy.cglib.Target`异常；

6. 目标类方法定义的时候不能加`final`修饰，否则代理类继承目标类以后就不能重写目标类的方法了。

## 第十二讲 jdk代理原理

### spring_12_aop_proxy_jdk_cglib_principle

[p36 035-第十二讲-jdk代理原理](https://www.bilibili.com/video/BV1P44y1N7QG?p=36)

[p37 036-第十二讲-jdk代理原理](https://www.bilibili.com/video/BV1P44y1N7QG?p=37)

[p38 037-第十二讲-jdk代理源码](https://www.bilibili.com/video/BV1P44y1N7QG?p=38)

为了更好地探究`jdk`动态代理原理，先用代码显式地模拟一下这个过程。

先定义一个`Foo`接口，里面有一个`foo()`方法，再定义一个`Target`类来实现这个接口，代码如下所示：

```java
public interface Foo {
    void foo();
}

@Slf4j
public final class Target implements Foo {
    public void foo() {
        log.debug("target foo");
    }
}

public class $Proxy0 implements Foo {
    @Override
    public void foo() {
        // 1. 功能增强
        System.out.println("before...");
        // 2. 调用目标
        new Target().foo();
    }
}

public class Main {
    public static void main(String[] args) {
        Foo proxy = new $Proxy0();
        proxy.foo();
    }
}
```

接下来对`Target`类中的`foo()`方法进行增强

1. 首先想到的是，再定义一个类也同样地实现一下`Foo`接口，然后在`foo()`方法中编写增强代码，接着再`new`一个`Target`对象，调用它的`foo()`方法，代码如下所示：

   ```java
   public class $Proxy0 implements Foo {
       @Override
       public void foo() {
           // 1. 功能增强
           System.out.println("before...");
           // 2. 调用目标
           new Target().foo();
       }
   }
   
   // 测试运行
   public class Main {
       public static void main(String[] args) {
           Foo proxy = new $Proxy0();
           proxy.foo();
       }
   }
   ```

2. 上面的代码把功能增强的代码和调用目标的代码都固定在了代理类的内部，不太灵活。因此可以通过定义一个`InvocationHandler`接口的方式来将这部分代码解耦出来，代码如下：

   ```java
   public interface InvocationHandler {
       void invoke();
   }
   
   
   public interface Foo {
       void foo();
   }
   
   @Slf4j
   public final class Target implements Foo {
       public void foo() {
           System.out.println("target foo");
       }
   }
   
   public class $Proxy0 implements Foo {
       private InvocationHandler h;
   
       public $Proxy0(InvocationHandler h) {
           this.h = h;
       }
   
       @Override
       public void foo() {
           h.invoke();
       }
   }
   
   public class Main {
       public static void main(String[] args) {
           Foo proxy = new $Proxy0(new InvocationHandler() {
               @Override
               public void invoke() {
                   // 1. 功能增强
                   System.out.println("before...");
                   // 2. 调用目标
                   new Target().foo();
               }
           });
           proxy.foo();
       }
   }
   ```

3. 第2个版本的代码虽然将功能增强的代码和调用目标的代码通过接口的方式独立出来了，但还是有问题，如果此时接口中新增了一个方法`bar()`，`Target`类和`$Proxy0`类中都要实现`bar()`方法，那么调用`proxy`的`foo()`和`bar()`方法都将间接调用目标对象的`foo()`方法，因为在`InvocationHandler`的`invoke()`方法中调用的是`target.foo()`方法，代码如下：

   ```java
   public interface InvocationHandler {
       void invoke();
   }
   
   public interface Foo {
       void foo();
       void bar();
   }
   
   @Slf4j
   public final class Target implements Foo {
       public void foo() {
           System.out.println("target foo");
       }
   
       @Override
       public void bar() {
           log.debug("target bar");
       }
   }
   
   public class $Proxy0 implements Foo {
       private InvocationHandler h;
   
       public $Proxy0(InvocationHandler h) {
           this.h = h;
       }
   
       @Override
       public void foo() {
           h.invoke();
       }
   
       @Override
       public void bar() {
           h.invoke();
       }
   }
   
   public class Main {
       public static void main(String[] args) {
           Foo proxy = new $Proxy0(new InvocationHandler() {
               @Override
               public void invoke() {
                   // 1. 功能增强
                   System.out.println("before...");
                   // 2. 调用目标
                   new Target().foo();
               }
           });
           proxy.foo();
           proxy.bar();
       }
   }
   ```

   改进方法是，代理类中调用方法的时候，通过反射把接口中对应的方法`Method`对象作为参数传给`InvocationHandler`，代码如下：

   ```java
   public interface InvocationHandler {
       void invoke(Method method, Object[] args) throws Throwable;
   }
   
   public interface Foo {
       void foo();
       void bar();
   }
   
   public final class Target implements Foo {
       public void foo() {
           System.out.println("target foo");
       }
   
       @Override
       public void bar() {
           System.out.println("target bar");
       }
   }
   
   public class $Proxy0 implements Foo {
       private InvocationHandler h;
   
       public $Proxy0(InvocationHandler h) {
           this.h = h;
       }
   
       @Override
       public void foo() {
           try {
               Method foo = Foo.class.getDeclaredMethod("foo");
               h.invoke(foo, new Object[0]);
           } catch (Throwable e) {
               e.printStackTrace();
           }
       }
   
       @Override
       public void bar() {
           try {
               Method bar = Foo.class.getDeclaredMethod("bar");
               h.invoke(bar, new Object[0]);
           } catch (Throwable e) {
               e.printStackTrace();
           }
       }
   }
   
   public class Main {
       public static void main(String[] args) {
           Foo proxy = new $Proxy0(new InvocationHandler() {
               @Override
               public void invoke(Method method, Object[] args) throws InvocationTargetException, IllegalAccessException {
                   // 1. 功能增强
                   System.out.println("before...");
                   // 2. 调用目标
                   method.invoke(new Target(), args);
               }
           });
           proxy.foo();
           proxy.bar();
       }
   }
   ```

4. 第3个版本的代码其实已经离jdk动态代理生成的代码很相近了，为了更好地学习底层，更近一步，修改`Foo`接口的中`bar()`方法，使其具有`int`类型的返回值，因此`InvocationHandler`的`invoke()`方法也得有返回值，同时将代理对象本身作为第一个参数，具体代码如下：

   ```java
   public interface InvocationHandler {
       Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
   }
   
   public interface Foo {
       void foo();
       int bar();
   }
   
   public final class Target implements Foo {
       public void foo() {
           System.out.println("target foo");
       }
   
       @Override
       public int bar() {
           System.out.println("target bar");
           return 1;
       }
   }
   
   public class $Proxy0 implements Foo {
       static Method foo;
       static Method bar;
   
       static {
           try {
               foo = Foo.class.getDeclaredMethod("foo");
               bar = Foo.class.getDeclaredMethod("bar");
           } catch (NoSuchMethodException e) {
               throw new NoSuchMethodError(e.getMessage());
           }
       }
   
       private InvocationHandler h;
   
       public $Proxy0(InvocationHandler h) {
           this.h = h;
       }
   
       @Override
       public void foo() {
           try {
   
               h.invoke(this, foo, new Object[0]);
           } catch (RuntimeException | Error e) {
               throw e;
           } catch (Throwable e) {
               throw new UndeclaredThrowableException(e);
           }
       }
   
       @Override
       public int bar() {
           try {
               return (int) h.invoke(this, bar, new Object[0]);
           } catch (RuntimeException | Error e) {
               throw e;
           } catch (Throwable e) {
               throw new UndeclaredThrowableException(e);
           }
       }
   }
   
   public class Main {
       public static void main(String[] args) {
           Foo proxy = new $Proxy0(new InvocationHandler() {
               @Override
               public Object invoke(Object proxy, Method method, Object[] args) throws InvocationTargetException, IllegalAccessException {
                   // 1. 功能增强
                   System.out.println("before...");
                   // 2. 调用目标
                   return method.invoke(new Target(), args);
               }
           });
           proxy.foo();
           System.out.println("bar()方法返回值：" + proxy.bar());
       }
   }
   ```

5. 到这里跟jdk的动态代理只有些微差距了，`jdk`的动态代码会让代理类再继承一个`Proxy`类，里面定义了一个`InvocationHandler`接口的对象，代理类中会通过`super(h)`调用父类Proxy的构造，这里建议结合视频教程理解。

[p39 038-第十二讲-jdk代理字节码生成](https://www.bilibili.com/video/BV1P44y1N7QG?p=39)

[p40 039-第十二讲-jdk反射优化](https://www.bilibili.com/video/BV1P44y1N7QG?p=40)



[p41 040-第十三讲-cglib代理原理](https://www.bilibili.com/video/BV1P44y1N7QG?p=41)

[p42 041-第十三讲-cglib代理原理-MethodProxy](https://www.bilibili.com/video/BV1P44y1N7QG?p=42)

[p43 042-第十四讲-MethodProxy原理](https://www.bilibili.com/video/BV1P44y1N7QG?p=43)

[p44 043-第十四讲-MethodProxy原理](https://www.bilibili.com/video/BV1P44y1N7QG?p=44)

[p45 044-第十五讲-Spring选择代理](https://www.bilibili.com/video/BV1P44y1N7QG?p=45)

[p46 045-第十五讲-Spring选择代理](https://www.bilibili.com/video/BV1P44y1N7QG?p=46)

[p47 046-第十五讲-Spring选择代理](https://www.bilibili.com/video/BV1P44y1N7QG?p=47)

[p48 047-第十六讲-切点匹配](https://www.bilibili.com/video/BV1P44y1N7QG?p=48)

[p49 048-第十六讲-切点匹配](https://www.bilibili.com/video/BV1P44y1N7QG?p=49)

[p50 049-第十七讲-Advisor与@Aspect](https://www.bilibili.com/video/BV1P44y1N7QG?p=50)

[p51 050-第十七讲-findEligibleAdvisors](https://www.bilibili.com/video/BV1P44y1N7QG?p=51)

[p52 051-第十七讲-wrapIfNecessary](https://www.bilibili.com/video/BV1P44y1N7QG?p=52)

[p53 052-第十七讲-代理创建时机](https://www.bilibili.com/video/BV1P44y1N7QG?p=53)

[p54 053-第十七讲-吐槽@Order](https://www.bilibili.com/video/BV1P44y1N7QG?p=54)

[p55 054-第十七讲-高级切面转低级切面](https://www.bilibili.com/video/BV1P44y1N7QG?p=55)

[p56 055-第十八讲-统一转换为环绕通知](https://www.bilibili.com/video/BV1P44y1N7QG?p=56)

[p57 056-第十八讲-统一转换为环绕通知](https://www.bilibili.com/video/BV1P44y1N7QG?p=57)

[p58 057-第十八讲-适配器模式](https://www.bilibili.com/video/BV1P44y1N7QG?p=58)

[p59 058-第十八讲-调用链执行](https://www.bilibili.com/video/BV1P44y1N7QG?p=59)

[p60 059-第十八讲-模拟实现调用链](https://www.bilibili.com/video/BV1P44y1N7QG?p=60)

[p61 060-第十八讲-模拟实现调用链-责任链模式](https://www.bilibili.com/video/BV1P44y1N7QG?p=61)

[p62 061-第十九讲-动态通知调用](https://www.bilibili.com/video/BV1P44y1N7QG?p=62)

[p63 062-第十九讲-动态通知调用](https://www.bilibili.com/video/BV1P44y1N7QG?p=63)

[p64 063-第廿讲-DispatcherServlet初始化时机](https://www.bilibili.com/video/BV1P44y1N7QG?p=64)

[p65 064-第廿讲-DispatcherServlet初始化时机](https://www.bilibili.com/video/BV1P44y1N7QG?p=65)

[p66 065-第廿讲-DispatcherServlet初始化执行的操作](https://www.bilibili.com/video/BV1P44y1N7QG?p=66)

[p67 066-第廿讲-RequestMappingHandlerMapping](https://www.bilibili.com/video/BV1P44y1N7QG?p=67)

[p68 067-第廿讲-RequestMappingHandlerAdapter](https://www.bilibili.com/video/BV1P44y1N7QG?p=68)

[p69 068-第廿讲-RequestMappingHandlerAdapter-参数和返回值解析器](https://www.bilibili.com/video/BV1P44y1N7QG?p=69)

[p70 069-第廿讲-RequestMappingHandlerAdapter-自定义参数解析器](https://www.bilibili.com/video/BV1P44y1N7QG?p=70)

[p71 070-第廿讲-RequestMappingHandlerAdapter-自定义返回值解析器](https://www.bilibili.com/video/BV1P44y1N7QG?p=71)

[p72 071-第廿一讲-参数解析器-准备](https://www.bilibili.com/video/BV1P44y1N7QG?p=72)

[p73 072-第廿一讲-参数解析器-准备](https://www.bilibili.com/video/BV1P44y1N7QG?p=73)

[p74 073-第廿一讲-参数解析器-@RequestParam 0-4](https://www.bilibili.com/video/BV1P44y1N7QG?p=74)

[p75 074-第廿一讲-参数解析器-组合模式](https://www.bilibili.com/video/BV1P44y1N7QG?p=75)

[p76 075-第廿一讲-参数解析器 5-9](https://www.bilibili.com/video/BV1P44y1N7QG?p=76)

[p77 076-第廿一讲-参数解析器 10-12](https://www.bilibili.com/video/BV1P44y1N7QG?p=77)

[p78 077-第廿二讲-获取参数名](https://www.bilibili.com/video/BV1P44y1N7QG?p=78)

[p79 078-第廿二讲-获取参数名](https://www.bilibili.com/video/BV1P44y1N7QG?p=79)

[p80 079-第廿三讲-两套底层转换接口](https://www.bilibili.com/video/BV1P44y1N7QG?p=80)

[p81 080-第廿三讲-一套高层转换接口](https://www.bilibili.com/video/BV1P44y1N7QG?p=81)

[p82 081-第廿三讲-类型转换与数据绑定演示](https://www.bilibili.com/video/BV1P44y1N7QG?p=82)

[p83 082-第廿三讲-web环境下数据绑定演示](https://www.bilibili.com/video/BV1P44y1N7QG?p=83)

[p84 083-第廿三讲-绑定器工厂](https://www.bilibili.com/video/BV1P44y1N7QG?p=84)

[p85 084-第廿三讲-绑定器工厂-@InitBinder扩展](https://www.bilibili.com/video/BV1P44y1N7QG?p=85)

[p86 085-第廿三讲-绑定器工厂-ConversionService扩展](https://www.bilibili.com/video/BV1P44y1N7QG?p=86)

[p87 086-第廿三讲-绑定器工厂-默认ConversionService](https://www.bilibili.com/video/BV1P44y1N7QG?p=87)

[p88 087-第廿三讲-加餐-如何获取泛型参数](https://www.bilibili.com/video/BV1P44y1N7QG?p=88)

[p89 088-第廿四讲-@ControllerAdvice-@InitBinder](https://www.bilibili.com/video/BV1P44y1N7QG?p=89)

[p90 089-第廿四讲-@ControllerAdvice-@InitBinder](https://www.bilibili.com/video/BV1P44y1N7QG?p=90)

[p91 090-第廿五讲-控制器方法执行流程](https://www.bilibili.com/video/BV1P44y1N7QG?p=91)

[p92 091-第廿五讲-控制器方法执行流程](https://www.bilibili.com/video/BV1P44y1N7QG?p=92)

[p93 092-第廿五讲-控制器方法执行流程-代码](https://www.bilibili.com/video/BV1P44y1N7QG?p=93)

[p94 093-第廿六讲-@ControllerAdvice-@ModelAttribute](https://www.bilibili.com/video/BV1P44y1N7QG?p=94)

[p95 094-第廿七讲-返回值处理器](https://www.bilibili.com/video/BV1P44y1N7QG?p=95)

[p96 095-第廿七讲-返回值处理器-1](https://www.bilibili.com/video/BV1P44y1N7QG?p=96)

[p97 096-第廿七讲-返回值处理器-2-4](https://www.bilibili.com/video/BV1P44y1N7QG?p=97)

[p98 097-第廿七讲-返回值处理器-5-7](https://www.bilibili.com/video/BV1P44y1N7QG?p=98)

[p99 098-第廿八讲-MessageConverter](https://www.bilibili.com/video/BV1P44y1N7QG?p=99)

[p100 099-第廿八讲-MessageConverter](https://www.bilibili.com/video/BV1P44y1N7QG?p=100)

[p101 100-第廿九讲-@ControllerAdvice-ResponseBodyAdvice](https://www.bilibili.com/video/BV1P44y1N7QG?p=101)

[p102 101-第廿九讲-@ControllerAdvice-ResponseBodyAdvice](https://www.bilibili.com/video/BV1P44y1N7QG?p=102)

[p103 102-第卅讲-异常处理](https://www.bilibili.com/video/BV1P44y1N7QG?p=103)

[p104 103-第卅讲-异常处理](https://www.bilibili.com/video/BV1P44y1N7QG?p=104)

[p105 104-第卅一讲-@ControllerAdvice-@ExceptionHandler](https://www.bilibili.com/video/BV1P44y1N7QG?p=105)

[p106 105-第卅二讲-tomcat异常处理](https://www.bilibili.com/video/BV1P44y1N7QG?p=106)

[p107 106-第卅二讲-tomcat异常处理-自定义错误地址](https://www.bilibili.com/video/BV1P44y1N7QG?p=107)

[p108 107-第卅二讲-tomcat异常处理-BasicErrorController](https://www.bilibili.com/video/BV1P44y1N7QG?p=108)

[p109 108-第卅二讲-tomcat异常处理-BasicErrorController](https://www.bilibili.com/video/BV1P44y1N7QG?p=109)

[p110 109-第卅三讲-HandlerMapping与HandlerAdapter-1](https://www.bilibili.com/video/BV1P44y1N7QG?p=110)

[p111 110-第卅三讲-HandlerMapping与HandlerAdapter-自定义](https://www.bilibili.com/video/BV1P44y1N7QG?p=111)

[p112 111-第卅四讲-HandlerMapping与HandlerAdapter-2](https://www.bilibili.com/video/BV1P44y1N7QG?p=112)

[p113 112-第卅五讲-HandlerMapping与HandlerAdapter-3](https://www.bilibili.com/video/BV1P44y1N7QG?p=113)

[p114 113-第卅五讲-HandlerMapping与HandlerAdapter-3-优化](https://www.bilibili.com/video/BV1P44y1N7QG?p=114)

[p115 114-第卅五讲-HandlerMapping与HandlerAdapter-3-优化](https://www.bilibili.com/video/BV1P44y1N7QG?p=115)

[p116 115-第卅五讲-HandlerMapping与HandlerAdapter-4-欢迎页](https://www.bilibili.com/video/BV1P44y1N7QG?p=116)

[p117 116-第卅五讲-HandlerMapping与HandlerAdapter-总结](https://www.bilibili.com/video/BV1P44y1N7QG?p=117)

[p118 117-第卅六讲-MVC执行流程](https://www.bilibili.com/video/BV1P44y1N7QG?p=118)

[p119 118-第卅六讲-MVC执行流程](https://www.bilibili.com/video/BV1P44y1N7QG?p=119)

[p120 119-第卅七讲-构建boot骨架项目](https://www.bilibili.com/video/BV1P44y1N7QG?p=120)

[p121 120-第卅八讲-构建boot war项目](https://www.bilibili.com/video/BV1P44y1N7QG?p=121)

[p122 121-第卅八讲-构建boot war项目-用外置tomcat测试](https://www.bilibili.com/video/BV1P44y1N7QG?p=122)

[p123 122-第卅八讲-构建boot war项目-用内嵌tomcat测试](https://www.bilibili.com/video/BV1P44y1N7QG?p=123)

[p124 123-第卅九讲-boot执行流程-构造](https://www.bilibili.com/video/BV1P44y1N7QG?p=124)

[p125 124-第卅九讲-boot执行流程-构造-1](https://www.bilibili.com/video/BV1P44y1N7QG?p=125)

[p126 125-第卅九讲-boot执行流程-构造-2](https://www.bilibili.com/video/BV1P44y1N7QG?p=126)

[p127 126-第卅九讲-boot执行流程-构造-3](https://www.bilibili.com/video/BV1P44y1N7QG?p=127)

[p128 127-第卅九讲-boot执行流程-构造-4-5](https://www.bilibili.com/video/BV1P44y1N7QG?p=128)

[p129 128-第卅九讲-boot执行流程-run-1](https://www.bilibili.com/video/BV1P44y1N7QG?p=129)

[p130 129-第卅九讲-boot执行流程-run-1](https://www.bilibili.com/video/BV1P44y1N7QG?p=130)

[p131 130-第卅九讲-boot执行流程-run-8-11](https://www.bilibili.com/video/BV1P44y1N7QG?p=131)

[p132 131-第卅九讲-boot执行流程-run-2,12](https://www.bilibili.com/video/BV1P44y1N7QG?p=132)

[p133 132-第卅九讲-boot执行流程-run-3](https://www.bilibili.com/video/BV1P44y1N7QG?p=133)

[p134 133-第卅九讲-boot执行流程-run-4](https://www.bilibili.com/video/BV1P44y1N7QG?p=134)

[p135 134-第卅九讲-boot执行流程-run-5](https://www.bilibili.com/video/BV1P44y1N7QG?p=135)

[p136 135-第卅九讲-boot执行流程-run-5](https://www.bilibili.com/video/BV1P44y1N7QG?p=136)

[p137 136-第卅九讲-boot执行流程-run-6](https://www.bilibili.com/video/BV1P44y1N7QG?p=137)

[p138 137-第卅九讲-boot执行流程-run-7](https://www.bilibili.com/video/BV1P44y1N7QG?p=138)

[p139 138-第卅九讲-boot执行流程-小结](https://www.bilibili.com/video/BV1P44y1N7QG?p=139)

[p140 139-第卌讲-Tomcat重要组件](https://www.bilibili.com/video/BV1P44y1N7QG?p=140)

[p141 140-第卌讲-内嵌Tomcat](https://www.bilibili.com/video/BV1P44y1N7QG?p=141)

[p142 141-第卌讲-内嵌Tomcat与Spring整合](https://www.bilibili.com/video/BV1P44y1N7QG?p=142)

[p143 142-第卌一讲-自动配置类原理](https://www.bilibili.com/video/BV1P44y1N7QG?p=143)

[p144 143-第卌一讲-自动配置类原理](https://www.bilibili.com/video/BV1P44y1N7QG?p=144)

[p145 144-第卌一讲-AopAutoConfiguration](https://www.bilibili.com/video/BV1P44y1N7QG?p=145)

[p146 145-第卌一讲-AopAutoConfiguration](https://www.bilibili.com/video/BV1P44y1N7QG?p=146)

[p147 146-第卌一讲-自动配置类2-4概述](https://www.bilibili.com/video/BV1P44y1N7QG?p=147)

[p148 147-第卌一讲-自动配置类2-DataSource](https://www.bilibili.com/video/BV1P44y1N7QG?p=148)

[p149 148-第卌一讲-自动配置类3-MyBatis](https://www.bilibili.com/video/BV1P44y1N7QG?p=149)

[p150 149-第卌一讲-自动配置类3-mapper扫描](https://www.bilibili.com/video/BV1P44y1N7QG?p=150)

[p151 150-第卌一讲-自动配置类4-事务](https://www.bilibili.com/video/BV1P44y1N7QG?p=151)

[p152 151-第卌一讲-自动配置类5-MVC](https://www.bilibili.com/video/BV1P44y1N7QG?p=152)

[p153 152-第卌一讲-自定义自动配置类](https://www.bilibili.com/video/BV1P44y1N7QG?p=153)

[p154 153-第卌二讲-条件装配底层1](https://www.bilibili.com/video/BV1P44y1N7QG?p=154)

[p155 154-第卌二讲-条件装配底层2](https://www.bilibili.com/video/BV1P44y1N7QG?p=155)

[p156 155-第卌三讲-FactoryBean](https://www.bilibili.com/video/BV1P44y1N7QG?p=156)

[p157 156-第卌四讲-@Indexed](https://www.bilibili.com/video/BV1P44y1N7QG?p=157)

[p158 157-第卌五讲-Spring代理的特点](https://www.bilibili.com/video/BV1P44y1N7QG?p=158)

[p159 158-第卌五讲-Spring代理的特点](https://www.bilibili.com/video/BV1P44y1N7QG?p=159)

[p160 159-第卌六讲-@Value注入底层1](https://www.bilibili.com/video/BV1P44y1N7QG?p=160)

[p161 160-第卌六讲-@Value注入底层2](https://www.bilibili.com/video/BV1P44y1N7QG?p=161)

[p162 161-第卌七讲-@Autowired注入底层-doResolveDependency外1](https://www.bilibili.com/video/BV1P44y1N7QG?p=162)

[p163 162-第卌七讲-@Autowired注入底层-doResolveDependency外2](https://www.bilibili.com/video/BV1P44y1N7QG?p=163)

[p164 163-第卌七讲-@Autowired注入底层-doResolveDependency内1](https://www.bilibili.com/video/BV1P44y1N7QG?p=164)

[p165 164-第卌七讲-@Autowired注入底层-doResolveDependency内2](https://www.bilibili.com/video/BV1P44y1N7QG?p=165)

[p166 165-第卌七讲-@Autowired注入底层-doResolveDependency内3](https://www.bilibili.com/video/BV1P44y1N7QG?p=166)

[p167 166-第卌七讲-@Autowired注入底层-doResolveDependency内4](https://www.bilibili.com/video/BV1P44y1N7QG?p=167)

[p168 167-第卌八讲-事件监听器1](https://www.bilibili.com/video/BV1P44y1N7QG?p=168)

[p169 168-第卌八讲-事件监听器2](https://www.bilibili.com/video/BV1P44y1N7QG?p=169)

[p170 169-第卌八讲-事件监听器3](https://www.bilibili.com/video/BV1P44y1N7QG?p=170)

[p171 170-第卌八讲-事件监听器4](https://www.bilibili.com/video/BV1P44y1N7QG?p=171)

[p172 171-第卌八讲-事件监听器5](https://www.bilibili.com/video/BV1P44y1N7QG?p=172)

[p173 172-第卌九讲-事件发布器1](https://www.bilibili.com/video/BV1P44y1N7QG?p=173)

[p174 173-第卌九讲-事件发](https://www.bilibili.com/video/BV1P44y1N7QG?p=174)