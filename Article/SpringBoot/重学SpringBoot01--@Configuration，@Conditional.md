# 重学Spring--@Configuration，@Conditional



### 1.	@Configuration

#### 1.1	定义

@Configuration用于定义配置类，可替换xml配置文件，被注解的类内部包含有一个或多个被@Bean注解的方法，这些方法将会被AnnotationConfigApplicationContext或AnnotationConfigWebApplicationContext类进行扫描，并用于构建bean定义，初始化Spring容器。



#### 1.2	基础使用

我们初始化两个实体类

```java
package com.springbootstudy.springbootstudy.entity;

import lombok.Data;

@Data
public class Person {
    private String name;

    private String sex;

    public Person(String name, String sex){
        this.name = name;
        this.sex = sex;
    }
}
```



```java
package com.springbootstudy.springbootstudy.entity;

import lombok.Data;

@Data
public class Pet {
    private String name;


    public Pet(String name) {
        this.name = name;

    }
}
```



此时我们创建一个MyConfig类，来管控Bean的注册

```java
package com.springbootstudy.springbootstudy.cofig;

import com.springbootstudy.springbootstudy.entity.Person;
import com.springbootstudy.springbootstudy.entity.Pet;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration(proxyBeanMethods = false)
public class Myconfig {
	
    //注册Person实例到Bean中
    @Bean("Jack")
    public Person person() {
        return new Person("Jack", 18);
    }
	//注册Pet实例到Bean中
    @Bean("Tom")
    public Pet pet() {
        return new Pet("wangwang");
    }
}
```



我们回到启动类，我们可以看到已经成功注册到Bean中

```java
package com.springbootstudy.springbootstudy;

import com.springbootstudy.springbootstudy.cofig.Myconfig;
import com.springbootstudy.springbootstudy.entity.Person;
import com.springbootstudy.springbootstudy.entity.Pet;
import org.springframework.boot.ConfigurableBootstrapContext;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;

@SpringBootApplication
public class SpringbootstudyApplication {

    public static void main(String[] args) {
		//返回IOC 容器
        ConfigurableApplicationContext run = SpringApplication.run(SpringbootstudyApplication.class, args);
		
        //从容器中获取组件
        Person Jack = run.getBean("Jack", Person.class);
        Pet Tom = run.getBean("Tom", Pet.class);

        System.out.println(Jack.getName());
        System.out.println(Tom.getName());
        
        //结果如下
      	//wang
		//wangwang
    }
}
```



#### 1.3 proxyBeanMethods属性

proxyBeanMethods属性默认为True，即开启代理对象。当我们使用代理对象的时候，调用它的方法，他会检测容器中是不是有了这样的组件，如果有，则不再新建组件，直接将已经有的组件返回。如果说没有的话，才会新建组件。这样保证了容器中的组件始终就保持单一性。不过这也有一个不好的地方，那就是每次都要检测，会降低速度。

```java
@SpringBootApplication
public class SpringbootstudyApplication {

    public static void main(String[] args) {

        ConfigurableApplicationContext run = SpringApplication.run(SpringbootstudyApplication.class, args);

        
        Myconfig bean = run.getBean(Myconfig.class);
        Pet tom1 =  bean.pet();
        Pet tom2 = bean.pet();
        
		// com.springbootstudy.springbootstudy.cofig.Myconfig$$EnhancerBySpringCGLIB$$5f881f0b@c3c4c1c 代理类
        System.out.println(bean);
		
        // true
        System.out.println(tom1 == tom2);
    }
}
```



如果设为false的话，即关闭代理对象，调用它的方法会去New一个新对象。

```java
@SpringBootApplication
public class SpringbootstudyApplication {

    public static void main(String[] args) {

        ConfigurableApplicationContext run = SpringApplication.run(SpringbootstudyApplication.class, args);


        Myconfig bean = run.getBean(Myconfig.class);
        Pet tom1 =  bean.pet();
        Pet tom2 = bean.pet();
		
        //com.springbootstudy.springbootstudy.cofig.Myconfig@e93f3d5    普通类
        System.out.println(bean);
		
        //False
        System.out.println(tom1 == tom2);
    }
}
```



proxyBeanMethods关键可以用来解决组建依赖的问题

我们在Persion类中加上一个属性Pet

```java
package com.springbootstudy.springbootstudy.entity;

import lombok.Data;

@Data
public class Person {
    private String name;

    private int age;

    private Pet pet;
    public Person(String name, int age){
        this.name = name;
        this.age = age;
    }
}

```



我们在MyConfig中注册Person组件时使用Set定义另一个组件pet

```java
package com.springbootstudy.springbootstudy.cofig;

import com.springbootstudy.springbootstudy.entity.Person;
import com.springbootstudy.springbootstudy.entity.Pet;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration(proxyBeanMethods = true)
public class Myconfig {

    @Bean("Jack")
    public Person person() {
        Person wang = new Person("wang", 18);
        wang.setPet(pet());
        return wang;
    }

    @Bean("Tom")
    public Pet pet() {
        return new Pet("wangwang");
    }
}
```



然后我们来验证当我们开启proxyBeanMethods为True时，组件Jack中的pet和我们定义另一个组件Tom是否一致

```java

import com.springbootstudy.springbootstudy.cofig.Myconfig;
import com.springbootstudy.springbootstudy.entity.Person;
import com.springbootstudy.springbootstudy.entity.Pet;
import org.springframework.boot.ConfigurableBootstrapContext;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;

@SpringBootApplication
public class SpringbootstudyApplication {

    public static void main(String[] args) {

        ConfigurableApplicationContext run = SpringApplication.run(SpringbootstudyApplication.class, args);

        Person Jack = run.getBean("Jack", Person.class);
        Pet JackPet = Jack.getPet();
        Pet Tom = run.getBean("Tom", Pet.class);

		//结果为 True
        System.out.println(JackPet == Tom);
    }
}

```





### 2.	@Conditional

条件装配：满足Conditional指定的条件，则进行组件注入

![img](https://cdn.nlark.com/yuque/0/2020/png/1354552/1602835786727-28b6f936-62f5-4fd6-a6c5-ae690bd1e31d.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_17%2Ctext_YXRndWlndS5jb20g5bCa56GF6LC3%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

