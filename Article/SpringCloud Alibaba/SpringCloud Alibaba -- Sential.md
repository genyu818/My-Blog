Sentinel

## 1.	简介

### 1.1	什么是 Sentinel

随着微服务的流行，服务和服务之间的稳定性变得越来越重要。Sentinel 以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性。

Sential具有以下特征:

- **丰富的应用场景**：Sentinel 承接了阿里巴巴近 10 年的双十一大促流量的核心场景，例如秒杀（即突发流量控制在系统容量可以承受的范围）、消息削峰填谷、集群流量控制、实时熔断下游不可用应用等。
- **完备的实时监控**：Sentinel 同时提供实时的监控功能。您可以在控制台中看到接入应用的单台机器秒级数据，甚至 500 台以下规模的集群的汇总运行情况。
- **广泛的开源生态**：Sentinel 提供开箱即用的与其它开源框架/库的整合模块，例如与 Spring Cloud、Apache Dubbo、gRPC、Quarkus 的整合。您只需要引入相应的依赖并进行简单的配置即可快速地接入 Sentinel。同时 Sentinel 提供 Java/Go/C++ 等多语言的原生实现。
- **完善的 SPI 扩展机制**：Sentinel 提供简单易用、完善的 SPI 扩展接口。您可以通过实现扩展接口来快速地定制逻辑。例如定制规则管理、适配动态数据源等。

![Sentinel-features-overview](D:\Blog\My-Blog\pic\50505538-2c484880-0aaf-11e9-9ffc-cbaaef20be2b.png)



### 1.2	下载Sentinel

1. 打开链接，下载[sentinel-dashboard-1.8.3.jar](https://github.com/alibaba/Sentinel/releases/download/1.8.3/sentinel-dashboard-1.8.3.jar)

2. 打开cmd，输入``java -jar sentinel-dashboard-1.8.2.jar``

3. 打开浏览器，输入http://localhost:8080/

4. 输入账号密码(默认都是sentinel)

5. 看到以下界面为下载成功

   ![image-20220103160527161](D:\Blog\My-Blog\pic\image-20220103160527161.png)



## 2.	Sential流控(限流)

### 2.1	建立初始项目

1. 建立Maven Project

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
   
       <artifactId>CloudAlibabaSentinel8401</artifactId>
   
       <dependencies>
           <!--eureka-server-->
           <dependency>
               <groupId>com.alibaba.cloud</groupId>
               <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
           </dependency>
           <dependency>
               <groupId>com.alibaba.cloud</groupId>
               <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
           </dependency>
           <dependency>
               <groupId>com.alibaba.csp</groupId>
               <artifactId>sentinel-datasource-nacos</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-openfeign</artifactId>
           </dependency>
           <!-- 引入自己定义的api通用包，可以使用Payment支付Entity -->
           <dependency>
               <groupId>org.why.springcouldstudy1</groupId>
               <artifactId>cloud-api-commons</artifactId>
               <version>${project.version}</version>
           </dependency>
           <!--boot web actuator-->
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
           </dependency>
   
           <!--一般通用配置-->
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

3. 建立yaml

   ```yaml
   server:
     port: 8401
   spring:
     application:
       name: cloud-alibaba-sentinel-service
     cloud:
       nacos:
         discovery:
           server-addr: 127.0.0.1:8848
       sentinel:
         transport:
           dashboard: 127.0.0.1:8080
           port: 8719
   
   management:
     endpoints:
       web:
         exposure:
           include: '*'
   ```

4. 建立启动类

   ```java
   package com.why.springcloud.alibaba;
   
   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
   
   @SpringBootApplication
   @EnableDiscoveryClient
   public class SentinelService8401 {
       public static void main(String[] args) {
           SpringApplication.run(SentinelService8401.class, args);
       }
   
   }
   ```

5. 建立Controller

   ```java
   package com.why.springcloud.alibaba.controller;
   
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.RestController;
   
   @RestController
   public class FlowLimintController {
       @GetMapping("/testb")
       public String testA() {
           return "---------a";
       }
   
       @GetMapping("/testB")
       public String testB() throws InterruptedException {
           TimeUnit.MILLISECONDS.sleep(1000);
           return "---------b";
       }
   }
   ```

6. 启动项目

7. 打开Sentinel Dashboard，打开簇点链路，可以看到`testA`和`testB`已经被注册到Sentinel中

   ![image-20220103161718189](D:\Blog\My-Blog\pic\image-20220103161718189.png)



### 2.2	QPS直接失败

1. 打开簇点链路页面，选择`/testA`接口，点击流控按钮

   ![image-20220103162145926](D:\Blog\My-Blog\pic\image-20220103162145926.png)

2. 选择阈值类型为QPS，阈值为1

   ![](D:\Blog\My-Blog\pic\image-20220103162239996.png)

3. 那么此时，代表我们的/testA接口，1S内只允许一个请求

4. 测试

   1. 每秒请求一次，有正常相应

      ![image-20220103162820166](D:\Blog\My-Blog\pic\image-20220103162820166.png)

   2. 一秒内连续点刷新几次发现被Sentinel限流了

      ![image-20220103162923526](D:\Blog\My-Blog\pic\image-20220103162923526.png)

### 2.3	线程数直接失败

1. 打开簇点链路页面，选择`/testB`接口，点击流控按钮

2. 选择阈值类型为并发线程数，阈值为1

   ![image-20220103163502932](C:\Users\MSI-PC\AppData\Roaming\Typora\typora-user-images\image-20220103163502932.png)

   

3. 那么此时，代表我们的`/testB`接口，1S内只允许接受一个线程的请求。(`/testB`已经将线程sleep了1s)。

4. 测试

   1. 注释掉``/testB``中的sleep代码，不停的去请求``/testB``，没有被限流

   2. 打开``/testB``中的sleep代码，不停的去请求``/testB``，发现被限流。(sleep过程中，有新的请求会再次开启一个线程，超过一个线程就会被限流)

      ![image-20220103164557446](D:\Blog\My-Blog\pic\image-20220103164557446.png)

### 2.4	关联

举例：当与A关联的资源B带到阈值时，就限流A自己

1. 打开簇点链路页面，选择`/testA`接口，点击流控按钮

2. 打开高级选项，点击关联，关联资源选择`/testB` 。**(即访问`/testB`的QPS超过1，`/testA`就挂了)**

   ![image-20220103165008936](D:\Blog\My-Blog\pic\image-20220103165008936.png)

3. 使用Postman并发模拟访问`/testB`

   ![image-20220103165501848](D:\Blog\My-Blog\pic\image-20220103165501848.png)

4. 此时去访问`/testA`，发现已经被限流了

   ![image-20220103165626348](D:\Blog\My-Blog\pic\image-20220103165626348.png)

### 2.5	预热

什么是预热：Warm Up（`RuleConstant.CONTROL_BEHAVIOR_WARM_UP`）方式，即预热/冷启动方式。当系统长期处于低水位的情况下，当流量突然增加时，直接把系统拉升到高水位可能瞬间把系统压垮。通过"冷启动"，让通过的流量缓慢增加，在一定时间内逐渐增加到阈值上限，给冷系统一个预热的时间，避免冷系统被压垮。默认冷加载因子为3。针对秒杀系统，怕一开始就把系统杀死，先给一个预热。

1. 假设现在对`/testA`设置Warm Up，预热时长为5，单机阈值设为10。那么刚启动时的QPS阈值为**(单机阈值 / 冷加载因子) 10 / 3 = 3**，5S中之后QPS阈值从3变成10。

![image-20220103170516546](D:\Blog\My-Blog\pic\image-20220103170516546.png)

2. 测试，打开浏览器请求http://localhost:8401/testA

   1. 刚开始不停的刷新发现会报出限流

   2. 5S之后刷新不会在出现限流(手动刷新很难会达到10次/1S，如果使用jmeter进行压测QPS超过10，依旧会被流控)

      ![image-20220103170744461](D:\Blog\My-Blog\pic\image-20220103170744461.png)

### 2.6	排队等待

匀速排队（`RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER`）方式会严格控制请求通过的间隔时间，也即是让请求以均匀的速度通过，对应的是漏桶算法。

换句话说就是，当请求超过QPS时，不会直接拒绝，而是加入队列中，等待处理。

这种方式主要用于处理间隔性突发的流量，例如消息队列。想象一下这样的场景，在某一秒有大量的请求到来，而接下来的几秒则处于空闲状态，我们希望系统能够在接下来的空闲期间逐渐处理这些请求，而不是在第一秒直接拒绝多余的请求。

1. 打开簇点链路页面，选择`/testA`接口，点击流控按钮，流控为高级等待，超时时间为20s。

   ![image-20220104191840559](D:\Blog\My-Blog\pic\image-20220104191840559.png)

2. 修改`/testA`的代码，每次请求时打印出当前线程

   ```java
   public class FlowLimintController {
       @GetMapping("/testA")
       public String testA() {
           log.info(Thread.currentThread().getName() + "\t" + "testA");
           return "---------a";
       }
   ```

3. 使用Postman对`/testA`进行迭代调用，请求时间间隔为0.5s

   ![image-20220104192117314](D:\Blog\My-Blog\pic\image-20220104192117314.png)

4. 测试，我们可以看到，即使我们在Postman中，每隔0.5s请求一次，但仍被限流到1s接受一个请求

   ![image-20220104193124194](D:\Blog\My-Blog\pic\image-20220104193124194.png)

## 3.	Sentinel降级(熔断)

除了流量控制以外，对调用链路中不稳定的资源进行熔断降级也是保障高可用的重要措施之一。一个服务常常会调用别的模块，可能是另外的一个远程服务、数据库，或者第三方 API 等。例如，支付的时候，可能需要远程调用银联提供的 API；查询某个商品的价格，可能需要进行数据库查询。然而，这个被依赖服务的稳定性是不能保证的。如果依赖的服务出现了不稳定的情况，请求的响应时间变长，那么调用服务的方法的响应时间也会变长，线程会产生堆积，最终可能耗尽业务自身的线程池，服务本身也变得不可用。

现代微服务架构都是分布式的，由非常多的服务组成。不同服务之间相互调用，组成复杂的调用链路。以上的问题在链路调用中会产生放大的效果。复杂链路上的某一环不稳定，就可能会层层级联，最终导致整个链路都不可用。因此我们需要对不稳定的**弱依赖服务调用**进行熔断降级，暂时切断不稳定调用，避免局部不稳定因素导致整体的雪崩。熔断降级作为保护自身的手段，通常在客户端（调用端）进行配置。



即当调用链路中某个资源处于不稳定的状态时(例如响应时间过长或异常比例升高)，对这个资源进行限制，让请求快速失败，避免发生级联错误。





### 3.1	RT(慢调用比例)

需要设置允许的慢调用 RT（即最大的响应时间），请求的响应时间大于该值则统计为慢调用。当单位统计时长（`statIntervalMs`）内请求数目大于设置的最小请求数目，并且慢调用的比例大于阈值，则接下来的熔断时长内请求会自动被熔断。经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求响应时间小于设置的慢调用 RT 则结束熔断，若大于设置的慢调用 RT 则会再次被熔断。

1. Controller中增加一个新的接口`/testRT`

   ```java
   @GetMapping("/testRT")
       public String testRT() throws InterruptedException {
           TimeUnit.SECONDS.sleep(1);
           log.info(Thread.currentThread().getName() + "\t" + "test RT");
           return "---------test RT";
       }
   ```

2. 打开簇点链路页面，选择`/testRT`接口，点击熔断按钮，做出如下设置

   **(解释一下，以下设置的意义就是 1S之内请求数目大于5个，并且慢比例大于3个，则接下来请求会被熔断1s)**

   ![image-20220104195716833](D:\Blog\My-Blog\pic\image-20220104195716833.png)

3. 设置JMeter，添加线程组1s请求10次

   ![image-20220104195932372](D:\Blog\My-Blog\pic\image-20220104195932372.png)

4. 测试

   1. 此时通过浏览器请求`/testRT`发现异常

      ![image-20220104200046571](D:\Blog\My-Blog\pic\image-20220104200046571.png)

   2. 等待熔断时间过(1s之后)发现请求正常

      ![image-20220104200143669](C:\Users\MSI-PC\AppData\Roaming\Typora\typora-user-images\image-20220104200143669.png)

### 3.2	异常比例

异常比例 (`ERROR_RATIO`)：当单位统计时长（`statIntervalMs`）内请求数目大于设置的最小请求数目，并且异常的比例大于阈值，则接下来的熔断时长内请求会自动被熔断。经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求成功完成（没有错误）则结束熔断，否则会再次被熔断。异常比率的阈值范围是 `[0.0, 1.0]`，代表 0% - 100%。

1. Controller中增加一个新的接口`/testRT`

   ```java
       @GetMapping("/testExceptionPercent")
       public String testExceptionPercent() {
           // 错误
           int age = 1 / 0;
           log.info(Thread.currentThread().getName() + "\t" + "testA");
           return "---------test ExceptionPercent";
       }
   
   ```

2. 打开簇点链路页面，选择`/testExceptionPercent`接口，点击熔断按钮，做出如下设置

   **(解释一下，以下设置的意义就是 1S之内请求数目大于5个，并且错误比例大于50%，则接下来请求会被熔断5s)**

   ![image-20220104200941295](D:\Blog\My-Blog\pic\image-20220104200941295.png)

3. 设置JMeter，添加线程组1s请求10次

   ![image-20220104200704277](D:\Blog\My-Blog\pic\image-20220104200704277.png)

4. 测试

   1. 此时通过浏览器请求`/testExceptionPercent`发现被熔断

      ![image-20220104200838879](D:\Blog\My-Blog\pic\image-20220104200838879.png)

   2. 等待熔断时间过(5s之后)发现请求正常(报错)

      ![image-20220104200915937](D:\Blog\My-Blog\pic\image-20220104200915937.png)



### 3.3	异常数

略(和异常比例非常像，甚至更简单)

## 4.	热点Key

何为热点？热点即经常访问的数据。很多时候我们希望统计某个热点数据中访问频次最高的 Top K 数据，并对其访问进行限制。比如：

- 商品 ID 为参数，统计一段时间内最常购买的商品 ID 并进行限制
- 用户 ID 为参数，针对一段时间内频繁访问的用户 ID 进行限制

热点参数限流会统计传入参数中的热点参数，并根据配置的限流阈值与模式，对包含热点参数的资源调用进行限流。热点参数限流可以看做是一种特殊的流量控制，仅对包含热点参数的资源调用生效。

1. Controller中增加一个新的接口`/testHostKey`

   ```java
   @GetMapping("/testHostKey")
   //	预热一下，后面会详细说。 代表当testHostKey方法出错时，会有一个兜底方法deal_testHotKey
   @SentinelResource(value = "testHostKey", blockHandler = "deal_testHotKey")
   public String testHostKey(@RequestParam(value = "p1", required = false) String p1,
                             @RequestParam(value = "p2", required = true) String p2) {
       return "---------test hot key";
   }
   
   public String deal_testHotKey(String p1, String p2, BlockException blockException) {
       return "deal with hot key";
   }
   ```

2. 打开簇点链路页面，选择`/testHostKey`接口，点击热点按钮，做出如下设置

   **（PS：一定要注意，资源名使用的是 @SentinelResource中的(testHostKey)）**

   ![image-20220104205808968](D:\Blog\My-Blog\pic\image-20220104205808968.png)

3. 测试

   1. 1s请求一次都是正常

   2. 1s内连续请求，发现返回了兜底方法

      ![image-20220104205946258](D:\Blog\My-Blog\pic\image-20220104205946258.png)

## 4.	@SentinelResource



### 4.1	只配置fallback

兜底方法

1. Controller中增加一个新的接口`/testFallback`

   ```java
   @GetMapping("/testFallback")
   @SentinelResource(value = "testFallback", fallback = "handlerFallback")
   public String testFallback(@RequestParam(value = "id", required = false) Integer id) throws IllegalAccessException {
       if (id == 5) {
           throw new IllegalAccessException("非法参数异常");
       }
       return "---------test Fallback";
   }
   
   //	主要handlerFallback方法一定要和testFallback一样的方法参数
   public String handlerFallback(@RequestParam(value = "id", required = false) Integer id, Throwable e) {
       return "请求错误" + e.getMessage();
   }
   ```

2. 测试(请求http://localhost:8401/testFallback?id=5)

   1. 转到自定义异常(兜底方法)

      ![image-20220104214556029](D:\Blog\My-Blog\pic\image-20220104214556029.png)

   2. 注释掉*@SentinelResource(value = "testFallback", fallback = "handlerFallback")*，到正常错误页面

      ![image-20220104214350196](D:\Blog\My-Blog\pic\image-20220104214350196.png)

### 4.2	只配置blockHandler

blockHandler只负责Sentinel控制台(即在Sentinel管理页面配置的流控，降级，热点等问题)，不负责Java内部错误例如**IllegalAccessException**等。

### 4.3	配置fallback和blockHandler

1. Controller中修改接口`/testFallback`

   ```java
   @GetMapping("/testFallback")
   @SentinelResource(value = "testFallback", fallback = "handlerFallback",  blockHandler = "dealBlock")
   public String testFallback(@RequestParam(value = "id", required = false) Integer id) throws IllegalAccessException {
       if (id == 5) {
           throw new IllegalAccessException("非法参数异常");
       }
       return "---------test Fallback";
   }
   
   public String handlerFallback(@RequestParam(value = "id", required = false) Integer id, Throwable e) {
       return "请求错误" + e.getMessage();
   }
   public String dealBlock(@RequestParam(value = "id", required = false) Integer id, BlockException blockException) {
       return "deal with block";
   }
   ```

2. 打开簇点链路页面，选择`/testFallback`接口，配置1QPS的限流

3. 测试

   1. 快速请求http://localhost:8401/testFallback?id=1( > 1QPS)，出现dealBlock(限流)

      ![image-20220104215648736](D:\Blog\My-Blog\pic\image-20220104215648736.png)

   2. 正常请求http://localhost:8401/testFallback?id=5 (<1QPS)，进入到handlerFallback(fallback)

      ![image-20220104215812919](D:\Blog\My-Blog\pic\image-20220104215812919.png)

   3. 快速请求http://localhost:8401/testFallback?id=5( > 1QPS)，出现dealBlock(限流)

      ![image-20220104215850450](D:\Blog\My-Blog\pic\image-20220104215850450.png)

### 4.4	配置exceptionsToignore

```java
// 可通过 exceptionsToIgnore忽略异常
@SentinelResource(value = "testFallback", fallback = "handlerFallback",  blockHandler = "dealBlock", exceptionsToIgnore = {IllegalAccessException.class})
```

