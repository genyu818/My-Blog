# IOC容器详解

## 1.	概述

IOC—Inversion of Control，即“控制反转”，是Spring的核心思想，它不是什么技术，而是一种设计思想。在Java开发中，IOC意味着将你设计好的对象交给容器控制，而不是传统的在你的对象内部直接控制。

首先来使用Spring来实际操作创建一个对象(SpringBoot中，已经取消繁琐的XML，使用注解的方式来声明Bean，但是为了本次IOC的学习会使用XML和注解的方法来学习Bean的注册)

#### 1.1	实体类

```java
package com.springbootstudy.springbootstudy.entity;
import static java.lang.System.out;

public class User {
    public void add() {
        out.println("我加");
    }
}
```



#### 1.2	编写XML

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    
    <bean id="user" class="com.springbootstudy.springbootstudy.entity.User"></bean>
</beans>
```



#### 1.3	测试类

```java
import com.springbootstudy.springbootstudy.entity.User;
import org.junit.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class test {
    @Test
    public void testAdd() {
        ApplicationContext context = new ClassPathXmlApplicationContext("bean.xml");
        User user = context.getBean("user", User.class);
        user.add();
    }
}
```

可以看到对象已经被成功创建，并且调用了add方法

![image-20211013203857772](C:\Users\MSI-PC\AppData\Roaming\Typora\typora-user-images\image-20211013203857772.png)



## 2.	IOC底层原理

下面来大致的看一下IOC的实现原理，可分为三个部分

#### 2.1	XML解析(注解解析)

简单来说就是通过解析开发者编写的XML(例子中的Bean.xml)或者注解获取类。

```xml
<bean id="user" class="com.springbootstudy.springbootstudy.entity.User"></bean>
```

#### 2.2	工厂模式(使用代理创建对象)

将类的创建交给创建工厂来解耦。

```java
//	简写的工厂模式，不建议在开发中使用。可以看一下博主的设计模式中的工厂模式详解
public class UserFactory {
    public static User getUser() {
        String classValue = class属性值 // com.springbootstudy.springbootstudy.entity.User
        Class clazz  = Class.forName(classValue);
        return clazz.newInstance();
    }
}
```



### 3.	IOC容器

IOC容器的底层就是对象工厂，Spring提供IOC容器的两种方式

1. BeanFactory
   1. IOC容器的基本实现，是Spring的内部接口
   2. 懒加载：加载的时候不会创建对象，使用对象时才会创建
2. ApplicationContext
   1. BeanFactory接口的子接口，<font color = 'red'>提供更多更强大的功能(哪些功能)</font>提供更多更强大的功能，一般由开发人员使用
   2. 加载配置文件的时候进行对象创建
   3. ApplicationContext有实现类
      1. ClassPathXmlApplicationContext(xml文件写在src下)
      2. FileSystemXmlApplicationContext(xml文件写在磁盘中，需要使用全路径)

1. IOC操作Bean管理(基于注解)



### 4.	Bean管理

1. Spring创建对象
2. Spring属性注入



#### 4.1	基于XML的操作(了解即可)

1. 配置文件方式(XML)
   1. 基于XML创建对象
   2. 基于XML注入属性
      * DI：依赖注入
        * 使用set注入
          * 创建对象
          * XML中注入属性（<proeryty></property>）
          * 注入空值 <proeryty>null </property>
          * 注入特殊值
            * 转义 <proeryty value ="&lt;&gt<><<beijing "></property>
            * 特殊符号内容写进CDATA
        * 构造器注入（有参构造器） <contructer-args></contructer-args>
   3. 注入Bean
      1. 级联属性注入
         1. ref
         2. <proerty> <bean></bean></proerty> 
         
      2. 集合属性

         1. 数组

            ```xml
            <proeryty>
            	<array>
                    <value>aaa</value>
            		<value>aaa</value>
            	</array>
            </property>
            ```

         2. Map集合

            ```xml
            <proeryty>
            	<map>
                	<entrty key="a" value = "b"></entrty>
                    <entrty key="c" value = "d"></entrty>
            	</map>
            </property>
            ```

         3. 集合中设置对象的值

            ```xml
            <bean id="course1" class="course">
            	<proerty name="cname" value="Spring"></proerty>
            </bean>
            <bean id="course2" class="course">
            	<proerty name="cname" value="Spring"></proerty>
            </bean>
            
            <bean id="stu" class ="student">
                <property name="courseList">
                    <list>
                    	<ref bean="course1"></ref>
                        <ref bean="course2"></ref>
                    </list>
                </property>
            </bean>
            ```




#### 4.2	基于注解操作的操作

Spring提供一下注解创建Bean对象

1. @Component ： 通用的注解，可标注任意类为 `Spring` 组件。如果一个 Bean 不知道属于哪个层，可以使用`@Component` 注解标注。
2. @Repository ： 对应持久层即 Dao 层，主要用于数据库相关操作。
3. @Service ： 对应服务层，主要涉及一些复杂的逻辑，需要用到 Dao 层。
4. @Controller ： 对应 Spring MVC 控制层，主要用户接受用户请求并调用 Service 层返回数据给前端页面。



#### 4.3	自动装配

##### 4.3.1	@Autowired

1. 作用于属性

   当`@Autowired`注解使用在类的属性上时，Spring会自动调用属性的目标类的构造方法，并将最终的实例化对象赋予当前的属性。下面以电脑，键盘为例子，演示两者之间通过Spring完成自动注入：

   ```java
   @Component
   public class Keyboard {
   
       public void working(){
           System.out.println("keyboard is working...");
       }
   }
   ```

   

   ```java
   @Component
   public class Computer {
   
       //	如果没有@Autowired，keyboard为null
       @Autowired
       private Keyboard keyboard;
       
       public void working(){
           keyboard.working();
           System.out.println("computer is working...");
       }
   }
   
   ```

   最后，在main方法中测试上述定义的Keyboard组件是否成功注入到Computer类中，并输出相关的提示语句。在测试代码中，首先使用AnnotationConfigApplicationContext类获得应用上下文，然后通过类的类型获得注入到Spring容器中的bean组件。

   ```java
   @SpringBootApplication
   public class AutowiredAnnotationApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(AutowiredAnnotationApplication.class, args);
           AnnotationConfigApplicationContext context =
                   new AnnotationConfigApplicationContext(AppConfig.class);
   
           System.out.println("@Autowired Annotation on field.");
           Computer computer = context.getBean(Computer.class);
           computer.working();
   
           context.close();
   
       }
   }
   ```

   main()方法运行结束后，可在控制台观察到如下的输出语句：

   ```tex
   @Autowired Annotation on field.
   keyboard is working...
   computer is working...
   ```

   

2. 作用于Set方法

   当在Setter方法上使用@Autowired注解时，Spring会根据对象类型完成组件的自动装配工作。创建一个新的类MagicBook.java来演示此案例(其余的类沿用之前案例中的类):

   ```java
   @Component
   public class MagicBook {
   
       private Keyboard keyboard;
   
       @Autowired
       public void setKeyboard(Keyboard keyboard) {
           this.keyboard = keyboard;
       }
      
       public void working(){
           keyboard.working();
           System.out.println("MagicBook is working...");
       }
   }
   ```

   同样，在main()方法中编写对应的代码进行测试，并观察控制台输出信息。

   ```java
   System.out.println("@Autowired Annotation on setter method.");
   MagicBook magicBook = context.getBean(MagicBook.class);
   magicBook.working();
   ```

   ```tex
   @Autowired Annotation on setter method.
   screen is working...
   keyboard is working...
   mouse is working...
   MagicBook is working...
   ```

   Notes: 只有调用Set方法时才会进行装配

   

3. 作用于Constructor

除上述两种方式外，@Autowired还可以在构造器上使用，在此情况下，当实例化当前对象时，Spring将自动完成组件注入工作。新建一个XPhone.java类演示此案例，并在main()方法中编写测试代码，运行并观察控制台输出。

```java
@Component
public class XPhone {
    private final Keyboard keyboard;
    private final Mouse mouse;
    private final Screen screen;

    @Autowired
    public XPhone(Keyboard keyboard,Mouse mouse,Screen screen){
        this.keyboard = keyboard;
        this.mouse = mouse;
        this.screen = screen;
    }

    public void working(){
        keyboard.working();
        mouse.working();
        screen.working();
        System.out.println("XPhone is working...");
    }
}
```

```java
System.out.println("@Autowired Annotation on constructor.");
XPhone xPhone = context.getBean(XPhone.class);
xPhone.working();
```

```tex
@Autowired Annotation on constructor.
keyboard is working...
mouse is working...
screen is working...
XPhone is working...
```



##### 4.3.2	@Qualifier

默认情况下，`@Autowired` 按类型装配 **Spring Bean**。 如果容器中有多个相同类型的 **bean**，则框架将抛出 `NoUniqueBeanDefinitionException`，  以提示有多个满足条件的 **bean** 进行自动装配。程序无法正确做出判断使用哪一个。

```java
    @Component("fooFormatter")
    public class FooFormatter implements Formatter {
        public String format() {
            return "foo";
        }
    }

    @Component("barFormatter")
    public class BarFormatter implements Formatter {
        public String format() {
            return "bar";
        }
    }

    @Component
    public class FooService {
        @Autowired
        private Formatter formatter;
        
        //todo 
    }

```

如果我们尝试将 `FooService` 加载到我们的上下文中，**Spring** 框架将抛出 `NoUniqueBeanDefinitionException`。这是因为 **Spring** 不知道要注入哪个 **bean**。为了避免这个问题，我们使用@Qualifier注解，通过指定唯一名称即可解决该问题。

```java
@Component
public class FooService {
    @Autowired
    @Qualifier("fooFormatter")
    private Formatter formatter;

    //todo 
}
```



##### 4.3.3	@Primary

还有另一个名为 `@Primary` 的注解，我们也可以用来发生依赖注入的歧义时决定要注入哪个 **bean**。当存在多个相同类型的 **bean** 时，此注解定义了首选项。除非另有说明，否则将使用与 `@Primary` 注释关联的 **bean** 。 我们来看一个例子：

```java
@Component
@Primary
public class FooFormatter implements Formatter {
    public String format() {
        return "foo";
    }
}

@Component
public class BarFormatter implements Formatter {
    public String format() {
        return "bar";
    }
}
```



##### 4.3.4	@Resources



#### 4.3 Bean的生命周期和作用域

##### 4.3.1 作用域

在Spring中，开发者可以设置(通过Scope)创建的Bean是单例还是多实例(默认是**单例**)。

Scope属性值：

1. singleto：单例(加载Spring配置时就会创建单实例对象)
2. prototype：多实例(调用getBean方法时创建多实例对象，**懒加载**)
3. request
4. session

##### 4.3.2	生命周期

1. 通过构造器创建bean实例（无参构造器）

2. 为bean的属性设置值和对其他bean引用(依赖注入)

3. 检查Aware相关接口，将xxxAware注入给bean

   * Aware是一个标记性的超接口（顶级接口），指示了一个Bean有资格通过回调方法的形式获取Spring容器底层组件。
   * 说白了：只要实现了Aware子接口的Bean都能获取到一个Spring底层组件。

4. BeanPostProcessor ： postProcessBeforeInitialzation，当前正在初始化的bean对象会被传递进来，我们就可以对这个bean作任何处理。
   这个函数会先于InitialzationBean执行，因此称为前置处理。

5. 调用bean的初始化方法(需要配置初始化方法)

   ```xml
   <bean  ...  init-method="initMethod"></bean>
   ```

6. BeanPostProcessor ： postProcessAfterInitialzation，当前正在初始化的bean对象会被传递进来，我们就可以对这个bean作任何处理。
   这个函数会先晚于InitialzationBean执行，因此称为后置处理。

7. bean可以使用了(对象获取到了)

8. 当容器关闭时，调用bean的销毁方法(需要进行配置销毁方法)

   ```xml
   <bean  ...  destory-method="destoryMethod"></bean>
   ```

#### 

#### 4.4	Bean的循环依赖

