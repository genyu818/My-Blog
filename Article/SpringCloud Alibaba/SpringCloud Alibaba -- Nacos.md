# Nacos

## 1.	简介

### 1.1	什么是 Nacos

简单来说Nacos集成了Eureka(服务调用)，Config(服务配置)，Bus(服务总线)，Hystrix(服务降级，熔断，限流等功能)。用于构建简单易用的，服务相关的工具集，包括服务发现、配置管理、服务元数据存储、推送、一致性及元数据管理等。兼容与包括[Spring Cloud](https://github.com/alibaba/spring-cloud-alibaba)、[Kubernetes](https://github.com/kubernetes/kubernetes)、[Dubbo](https://github.com/apache/dubbo)等开源生态做无缝的融合与支持，同时给这些生态带来很多面向生产时需要的优秀特性。

![image-20220102201407725](C:\Users\MSI-PC\AppData\Roaming\Typora\typora-user-images\image-20220102201407725.png)

本文主要面向 [Spring Cloud](https://spring.io/projects/spring-cloud) 的使用者，通过两个示例来介绍如何使用 Nacos 来实现分布式环境下的配置管理和服务注册发现。

关于 Nacos Spring Cloud 的详细文档请参看：[Nacos Config](https://github.com/spring-cloud-incubator/spring-cloud-alibaba/wiki/Nacos-config) 和 [Nacos Discovery](https://github.com/spring-cloud-incubator/spring-cloud-alibaba/wiki/Nacos-discovery)。

- 通过 Nacos Server 和 spring-cloud-starter-alibaba-nacos-config 实现配置的动态变更。
- 通过 Nacos Server 和 spring-cloud-starter-alibaba-nacos-discovery 实现服务的注册与发现。



### 1.2	下载Nacos

您需要先下载 Nacos 并启动 Nacos server。

1. 前往https://github.com/alibaba/nacos/releases，下载需要的版本。

2. 解压

3. 进入 ``nacos-server-1.4.2\nacos\bin``

4. 打开cmd，输入startup.cmd

5. 看到以下画面为成功

   ![image-20220102201929552](D:\Blog\My-Blog\pic\image-20220102201929552.png)

6. 打开浏览器输入http://localhost:8848/nacos/#/login 

7. 输入账号密码(都是Nacos)，看到一下页面，即是Nacos管理页面

   ![image-20220102202149097](D:\Blog\My-Blog\pic\image-20220102202149097.png)





### 1.3	创建父项目

1. 建立Maven Project，命名为SpringCloud

2. 修改POM

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <modelVersion>4.0.0</modelVersion>
   
       <groupId>com.why.study.springcloud</groupId>
       <artifactId>SpringCloud</artifactId>
       <packaging>pom</packaging>
       <version>1.0-SNAPSHOT</version>
       <properties>
           <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
           <maven.compiler.source>1.8</maven.compiler.source>
           <maven.compiler.target>1.8</maven.compiler.target>
           <junit.version>4.12</junit.version>
           <log4j.version>1.2.17</log4j.version>
           <lombok.version>1.16.18</lombok.version>
           <mysql.version>5.1.47</mysql.version>
           <druid.version>1.1.16</druid.version>
           <mybatis.spring.boot.version>1.3.0</mybatis.spring.boot.version>
       </properties>
   
       <dependencyManagement>
           <dependencies>
               <!--spring boot 2.2.2-->
               <dependency>
                   <groupId>com.alibaba.cloud</groupId>
                   <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                   <version>2.1.0.RELEASE</version>
               </dependency>
               <dependency>
                   <groupId>org.springframework.boot</groupId>
                   <artifactId>spring-boot-dependencies</artifactId>
                   <version>2.2.2.RELEASE</version>
                   <type>pom</type>
                   <scope>import</scope>
               </dependency>
               <dependency>
                   <groupId>org.springframework.cloud</groupId>
                   <artifactId>spring-cloud-dependencies</artifactId>
                   <type>pom</type>
                   <scope>import</scope>
               </dependency>
               <!--spring cloud alibaba 2.1.0.RELEASE-->
               <dependency>
                   <groupId>com.alibaba.cloud</groupId>
                   <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                   <version>2.1.0.RELEASE</version>
                   <type>pom</type>
                   <scope>import</scope>
               </dependency>
               <dependency>
                   <groupId>mysql</groupId>
                   <artifactId>mysql-connector-java</artifactId>
                   <version>${mysql.version}</version>
               </dependency>
               <dependency>
                   <groupId>com.alibaba</groupId>
                   <artifactId>druid</artifactId>
                   <version>${druid.version}</version>
               </dependency>
               <dependency>
                   <groupId>org.mybatis.spring.boot</groupId>
                   <artifactId>mybatis-spring-boot-starter</artifactId>
                   <version>${mybatis.spring.boot.version}</version>
               </dependency>
               <dependency>
                   <groupId>junit</groupId>
                   <artifactId>junit</artifactId>
                   <version>${junit.version}</version>
               </dependency>
               <dependency>
                   <groupId>log4j</groupId>
                   <artifactId>log4j</artifactId>
                   <version>${log4j.version}</version>
               </dependency>
               <dependency>
                   <groupId>org.projectlombok</groupId>
                   <artifactId>lombok</artifactId>
                   <version>${lombok.version}</version>
                   <optional>true</optional>
               </dependency>
           </dependencies>
       </dependencyManagement>
   
       <build>
           <plugins>
               <plugin>
                   <groupId>org.springframework.boot</groupId>
                   <artifactId>spring-boot-maven-plugin</artifactId>
                   <version>2.2.2.RELEASE</version>
   
                   <configuration>
                       <fork>true</fork>
                       <addResources>true</addResources>
                   </configuration>
               </plugin>
           </plugins>
       </build>
   
   </project>
   ```

   

   

## 2.	服务注册中心

Nacos服务注册中心提供了自动配置以及其他 Spring 编程模型的习惯用法为 Spring Boot 应用程序在服务注册与发现方面提供和 Nacos 的无缝集成。 通过一些简单的注解，您可以快速来注册一个服务，并使用经过双十一考验的 Nacos 组件来作为大规模分布式系统的服务注册中心。

### 2.1	创建Provider

1. 在父Project下创建Maven Project

2. 修改POM

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <parent>
           <artifactId>SpringCloud</artifactId>
           <groupId>com.why.study.springcloud</groupId>
           <version>1.0-SNAPSHOT</version>
       </parent>
       <modelVersion>4.0.0</modelVersion>
   
       <artifactId>CloudAlibabaProvider9000</artifactId>
   
       <dependencies>
           <dependency>
               <groupId>com.alibaba.cloud</groupId>
               <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
           </dependency>
           <dependency>
               <groupId>com.alibaba.cloud</groupId>
               <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
           </dependency>
           <dependency>
               <groupId>org.why.springcouldstudy1</groupId>
               <artifactId>cloud-api-commons</artifactId>
               <version>${project.version}</version>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-devtools</artifactId>
               <scope>runtime</scope>
               <optional>true</optional>
           </dependency>
           <dependency>
               <groupId>org.projectlombok</groupId>
               <artifactId>lombok</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-test</artifactId>
               <scope>test</scope>
           </dependency>
           <dependency>
               <groupId>junit</groupId>
               <artifactId>junit</artifactId>
           </dependency>
       </dependencies>
   
   </project>
   ```

3. 创建application.yml

   ```yaml
   server:
     port: 9000
   spring:
     application:
       # 服务名要唯一
       name: cloud-alibaba-provider
     cloud:
       nacos:
         discovery:
           # nacos地址
           server-addr: 127.0.0.1:8848
   ```
   
4. 创建主类

   ```java
   package com.why.springcloud.alibaba;
   
   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
   
   @SpringBootApplication
   // 服务注册注解
   @EnableDiscoveryClient
   public class Provider9000 {
       public static void main(String[] args) {
           SpringApplication.run(Provider9000.class, args);
       }
   }
   ```

5. 创建Controller

   ```java
   package com.why.springcloud.alibaba.controller;
   
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.PathVariable;
   import org.springframework.web.bind.annotation.RestController;
   
   @RestController
   public class PaymentController {
       @GetMapping(value = "/payment/nacos/{id}")
       public String getPayment(@PathVariable("id") Integer id){
           return "nacons registry, server port " ;
       }
   
       @GetMapping(value = "/testA")
       public String testA() {
           return "testA";
       }
       @GetMapping(value = "/testB")
       public String testB() {
           return "testB";
       }
   }
   ```

6. 启动主类

7. 打开Nacos管理页面，点开服务列表发现，此时cloud-alibaba-provider已经注册上了

   ![image-20220102203949989](D:\Blog\My-Blog\pic\image-20220102203949989.png)



### 2.2	创建Costumer

1. 在父Project下创建Maven Project

2. 创建POM

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <parent>
           <artifactId>SpringCloud</artifactId>
           <groupId>com.why.study.springcloud</groupId>
           <version>1.0-SNAPSHOT</version>
       </parent>
       <modelVersion>4.0.0</modelVersion>
   
       <artifactId>CloudAlibabaCostomer80</artifactId>
   
       <dependencies>
           <dependency>
               <groupId>com.alibaba.cloud</groupId>
               <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
           </dependency>
           <dependency>
               <groupId>org.why.springcouldstudy1</groupId>
               <artifactId>cloud-api-commons</artifactId>
               <version>${project.version}</version>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-devtools</artifactId>
               <scope>runtime</scope>
               <optional>true</optional>
           </dependency>
           <dependency>
               <groupId>org.projectlombok</groupId>
               <artifactId>lombok</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-test</artifactId>
               <scope>test</scope>
           </dependency>
           <dependency>
               <groupId>junit</groupId>
               <artifactId>junit</artifactId>
           </dependency>
       </dependencies>
   
   </project>
   ```

3. 创建application.yml

   ```yaml
   server:
     port: 80
   spring:
     application:
       name: cloud-alibaba-customer
     cloud:
       nacos:
         discovery:
           server-addr: 127.0.0.1:8848
   ```

4. 创建主类

   ```java
   package com.why.springcloud.alibaba;
   
   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
   
   @SpringBootApplication
   @EnableDiscoveryClient
   public class Customer80 {
       public static void main(String[] args) {
           SpringApplication.run(Customer80.class, args);
       }
   }
   ```

5. 创建RestFul Template

   ```java
   package com.why.springcloud.alibaba.config;
   
   import org.springframework.cloud.client.loadbalancer.LoadBalanced;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.web.client.RestTemplate;
   
   @Configuration
   public class ApplicationContext {
       @Bean
       //	负载均衡
       @LoadBalanced
       public RestTemplate getRestTemplate() {
           return new RestTemplate();
       }
   }
   ```

6. 创建Controller

   ```java
   package com.why.springcloud.alibaba.controller;
   
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.beans.factory.annotation.Value;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.Mapping;
   import org.springframework.web.bind.annotation.PathVariable;
   import org.springframework.web.bind.annotation.RestController;
   import org.springframework.web.client.RestTemplate;
   
   @RestController
   public class CustomerController {
       @Autowired
       private RestTemplate restTemplate;
   
       String serviceURL = "http://cloud-alibaba-provider";
   
       @GetMapping(value = "/customer/payment/nacos/{id}")
       public String paymentInfo(@PathVariable("id") Integer id) {
           return restTemplate.getForObject(serviceURL + "/payment/nacos/" + id, String.class);
       }
   }
   ```

7. 启动主类

8. 查看Nacos管理网站，此时cloud-alibaba-customer已经注册上了

   ![image-20220102204928821](D:\Blog\My-Blog\pic\image-20220102204928821.png)

9. 测试，打开浏览器，输入http://localhost/customer/payment/nacos/13

   ![image-20220102205431209](D:\Blog\My-Blog\pic\image-20220102205431209.png)

## 3.	配置中心

Nacos 提供用于存储配置和其他元数据的 key/value 存储，为分布式系统中的外部化配置提供服务器端和客户端支持。使用 Spring Cloud Alibaba Nacos Config，您可以在 Nacos Server 集中管理你 Spring Cloud 应用的外部属性配置。

Spring Cloud Alibaba Nacos Config 是 Config Server 和 Client 的替代方案，客户端和服务器上的概念与 Spring Environment 和 PropertySource 有着一致的抽象，在特殊的 bootstrap 阶段，配置被加载到 Spring 环境中。当应用程序通过部署管道从开发到测试再到生产时，您可以管理这些环境之间的配置，并确保应用程序具有迁移时需要运行的所有内容。

### 3.1	创建Nacos Config中心

1. 在父Project下创建Maven Project

2. 创建POM

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <parent>
           <artifactId>SpringCloud</artifactId>
           <groupId>com.why.study.springcloud</groupId>
           <version>1.0-SNAPSHOT</version>
       </parent>
       <modelVersion>4.0.0</modelVersion>
   
       <artifactId>CloudAlibabaConfigClient8003</artifactId>
   
       <dependencies>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-dependencies</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-bootstrap</artifactId>
               <version>3.0.3</version>
           </dependency>
           <dependency>
               <groupId>com.alibaba.cloud</groupId>
               <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
           </dependency>
           <dependency>
               <groupId>com.alibaba.cloud</groupId>
               <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
           </dependency>
           <dependency>
               <groupId>org.why.springcouldstudy1</groupId>
               <artifactId>cloud-api-commons</artifactId>
               <version>${project.version}</version>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-devtools</artifactId>
               <scope>runtime</scope>
               <optional>true</optional>
           </dependency>
           <dependency>
               <groupId>org.projectlombok</groupId>
               <artifactId>lombok</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-test</artifactId>
               <scope>test</scope>
           </dependency>
           <dependency>
               <groupId>junit</groupId>
               <artifactId>junit</artifactId>
           </dependency>
       </dependencies>
   
   </project>
   ```

3. 创建bootstrap.properties

   ```properties
   server.port=8003
   spring.profiles.active=dev
   spring.cloud.nacos.config.server-addr=127.0.0.1:8848
   spring.cloud.nacos.discovery.server-addr=127.0.0.1:8848
   spring.application.name=nacos-cofig-client
   ```

4. 创建主类

   ```java
   package com.why.springcloud.alibaba;
   
   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
   
   @SpringBootApplication
   @EnableDiscoveryClient
   public class ConfigClient8003 {
       public static void main(String[] args) {
           SpringApplication.run(ConfigClient8003.class, args);
       }
   }
   ```

5. 打开Nacos管理Web，

   1. 打开配置列表

   2. 点击右侧加号

      ![image-20220102210606944](D:\Blog\My-Blog\pic\image-20220102210606944.png)

   3. 基于**bootstrap.properties 配置文件来配置Nacos Server 地址**, name-profiles.active.properties

      1. 如我们的bootstrap.properties，我们在Nacos管理系统上注册中心命名就应该是 ``nacos-cofig-client-dev.properties``

         ![image-20220102211027027](D:\Blog\My-Blog\pic\image-20220102211027027.png)

6. 创建Controller

   ```java
   package com.why.springcloud.alibaba.controller;
   
   import org.springframework.beans.factory.annotation.Value;
   import org.springframework.cloud.context.config.annotation.RefreshScope;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.RequestMapping;
   import org.springframework.web.bind.annotation.RestController;
   
   @RestController
   @RequestMapping("/config")
   @RefreshScope
   public class ConfigClientController {
       @Value("${version}")
       private int version;
   
       @RequestMapping("/get")
       public int get() {
           return version;
       }
   }
   ```

7. 启动主类

8. 请求http://localhost:8003/config/get，可以看到我们已经读取到在Nacos配置中心中的Version 2.

![image-20220102211534254](D:\Blog\My-Blog\pic\image-20220102211534254.png)

## 4.	Nacos集群 + 持久化配置

参照官网

https://nacos.io/zh-cn/docs/cluster-mode-quick-start.html