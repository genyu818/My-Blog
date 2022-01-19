# Seata

## 1.	简介

### 1.1 什么是Seata

Seata 是一款开源的分布式事务解决方案，致力于提供高性能和简单易用的分布式事务服务。Seata 将为用户提供了 AT、TCC、SAGA 和 XA 事务模式，为用户打造一站式的分布式解决方案。

![image](D:\Blog\My-Blog\pic\145942191-7a2d469f-94c8-4cd2-8c7e-46ad75683636.png)



### 1.2	Seata术语

#### TC (Transaction Coordinator) - 事务协调者

维护全局和分支事务的状态，驱动全局事务提交或回滚。

#### TM (Transaction Manager) - 事务管理器

定义全局事务的范围：开始全局事务、提交或回滚全局事务。

#### RM (Resource Manager) - 资源管理器

管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。



### 1.3	TCC模式

1. TM(**可以理解为程序中开启事务注解的点**)向TCC申请开启一个全局事务，并生成一个全局唯一的XID
2. XID在微服务的调用链路的上下文中传播
3. RM(**可以理解为连接DB的点，一个微服务可能连接多个DB**)向TC注册分支事务，将其纳入XID的全局事务管理范围
4. TM向TC发起针对XID的全局提交或者回滚决议
5. TC调度XID下管辖范围的全部分支事务完成提交或回滚请求

![image-20220105192050037](D:\Blog\My-Blog\pic\image-20220105192050037.png)



## 2.	下载Seata

1. 前往 https://seata.io/zh-cn/blog/download.html，下载

2. 解压

3. 修改**seata\seata-server-1.4.2\conf\file.conf**

   ```conf
   ## 修改DB这一段
   db {
       ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp)/HikariDataSource(hikari) etc.
       datasource = "druid"
       ## mysql/oracle/postgresql/h2/oceanbase etc.
       dbType = "mysql"
       driverClassName = "com.mysql.jdbc.Driver"
       ## if using mysql to store the data, recommend add rewriteBatchedStatements=true in jdbc connection param
       url = "jdbc:mysql://127.0.0.1:3306/seata?rewriteBatchedStatements=true"
       ## 改成自己的账号密码
       user = "root"
       password = "your password"
       minConn = 5
       maxConn = 100
       globalTable = "global_table"
       branchTable = "branch_table"
       lockTable = "lock_table"
       queryLimit = 100
       maxWait = 5000
     }
   ```

4. 修改**seata\seata-server-1.4.2\conf\registry.conf**

   ```
    //修改为nacos
    type = "nacos"
   
     nacos {
       application = "seata-server"
       serverAddr = "127.0.0.1:8848"
       group = "SEATA_GROUP"
       namespace = ""
       cluster = "default"
       username = ""
       password = ""
     }
   ```

5. 启动Nacos

6. 启动 ``seata\seata-server-1.4.2\bin\seata-server.bat``

7. 打开nacos 服务管理，看到``seata-server``服务注册成功

   ![image-20220105202624904](D:\Blog\My-Blog\pic\image-20220105202624904.png)



## 3.	实战Seata(订单系统)

用户购买商品的业务逻辑。整个业务逻辑由3个微服务提供支持：

- 仓储服务：对给定的商品扣除仓储数量。
- 订单服务：根据采购需求创建订单。
- 帐户服务：从用户帐户中扣除余额。

**用户下订单应该库存减少的同时账户余额也减少(缺一不可，事务)**

下订单 -> 减库存 -> 扣余额 -> 改订单

![image-20220105202743001](D:\Blog\My-Blog\pic\image-20220105202743001.png)



### 3.1	创建DB

创建订单表

```sql
CREATE DATABASE seata_order;

USE seata_order;
CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  `ext` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
CREATE TABLE t_order(
    id BIGINT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY ,
    user_id BIGINT(11) DEFAULT NULL COMMENT '用户id',
    product_id BIGINT(11) DEFAULT NULL COMMENT '产品id',
    count INT(11) DEFAULT NULL COMMENT '数量',
    money DECIMAL(11,0) DEFAULT NULL COMMENT '金额',
    status INT(1) DEFAULT NULL COMMENT '订单状态：0创建中，1已完结'
)ENGINE=InnoDB AUTO_INCREMENT=7 CHARSET=utf8;
```



创建库存表

```sql
CREATE DATABASE seata_storage;
USE seata_storage;
CREATE TABLE t_storage(
    id BIGINT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY ,
    product_id BIGINT(11) DEFAULT NULL COMMENT '产品id',
    total INT(11) DEFAULT NULL COMMENT '总库存',
    used INT(11) DEFAULT NULL COMMENT '已用库存',
    residue INT(11) DEFAULT NULL COMMENT '剩余库存'
)ENGINE=InnoDB AUTO_INCREMENT=7 CHARSET=utf8;
INSERT INTO t_storage(id, product_id, total, used, residue) VALUES(1,1,100,0,100);
CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  `ext` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
```



创建账户表

```sql
CREATE DATABASE seata_account;

CREATE TABLE t_account(
    id BIGINT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY ,
    user_id BIGINT(11) DEFAULT NULL COMMENT '用户id',
    total DECIMAL(10,0) DEFAULT NULL COMMENT '总额度',
    used DECIMAL(10,0) DEFAULT NULL COMMENT '已用额度',
    residue DECIMAL(10,0) DEFAULT 0 COMMENT '剩余可用额度'
)ENGINE=InnoDB AUTO_INCREMENT=7 CHARSET=utf8;
INSERT INTO t_account(id, user_id, total, used, residue) VALUES(1,1,1000,0,1000);

CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  `ext` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

```



### 3.2 	创建订单工程

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
   
       <artifactId>CloudSeataOrder80</artifactId>
   
       <dependencies>
           <!--eureka-server-->
           <dependency>
               <groupId>com.alibaba.cloud</groupId>
               <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
           </dependency>
           <dependency>
               <groupId>com.alibaba.cloud</groupId>
               <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
               <exclusions>
                   <exclusion>
                       <groupId>io.seata</groupId>
                       <artifactId>seata-all</artifactId>
                   </exclusion>
               </exclusions>
           </dependency>
           <dependency>
               <groupId>io.seata</groupId>
               <artifactId>seata-all</artifactId>
               <version>1.4.2</version>
           </dependency>
           <dependency>
               <groupId>org.springframework.cloud</groupId>
               <artifactId>spring-cloud-starter-openfeign</artifactId>
               <version>2.2.1.RELEASE</version>
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
           <dependency>
               <groupId>org.mybatis.spring.boot</groupId>
               <artifactId>mybatis-spring-boot-starter</artifactId>
           </dependency>
           <dependency>
               <groupId>com.alibaba</groupId>
               <artifactId>druid-spring-boot-starter</artifactId>
               <version>1.1.16</version>
           </dependency>
           <dependency>
               <groupId>mysql</groupId>
               <artifactId>mysql-connector-java</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-jdbc</artifactId>
           </dependency>
       </dependencies>
   </project>
   ```

3. 在resources下创建``registry.conf``（将``seata\seata-server-1.4.2\conf\registry.conf``复制过来）

   ```
   registry {
     # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
     type = "nacos"
   
     nacos {
       application = "seata-server"
       serverAddr = "127.0.0.1:8848"
       group = "SEATA_GROUP"
       namespace = ""
       cluster = "default"
       username = ""
       password = ""
     }
     eureka {
       serviceUrl = "http://localhost:8761/eureka"
       application = "default"
       weight = "1"
     }
     redis {
       serverAddr = "localhost:6379"
       db = 0
       password = ""
       cluster = "default"
       timeout = 0
     }
     zk {
       cluster = "default"
       serverAddr = "127.0.0.1:2181"
       sessionTimeout = 6000
       connectTimeout = 2000
       username = ""
       password = ""
     }
     consul {
       cluster = "default"
       serverAddr = "127.0.0.1:8500"
       aclToken = ""
     }
     etcd3 {
       cluster = "default"
       serverAddr = "http://localhost:2379"
     }
     sofa {
       serverAddr = "127.0.0.1:9603"
       application = "default"
       region = "DEFAULT_ZONE"
       datacenter = "DefaultDataCenter"
       cluster = "default"
       group = "SEATA_GROUP"
       addressWaitTime = "3000"
     }
     file {
       name = "file.conf"
     }
   }
   
   config {
     # file、nacos 、apollo、zk、consul、etcd3
     type = "file"
   
     nacos {
       serverAddr = "127.0.0.1:8848"
       namespace = ""
       group = "SEATA_GROUP"
       username = ""
       password = ""
       dataId = "seataServer.properties"
     }
     consul {
       serverAddr = "127.0.0.1:8500"
       aclToken = ""
     }
     apollo {
       appId = "seata-server"
       ## apolloConfigService will cover apolloMeta
       apolloMeta = "http://192.168.1.204:8801"
       apolloConfigService = "http://192.168.1.204:8080"
       namespace = "application"
       apolloAccesskeySecret = ""
       cluster = "seata"
     }
     zk {
       serverAddr = "127.0.0.1:2181"
       sessionTimeout = 6000
       connectTimeout = 2000
       username = ""
       password = ""
       nodePath = "/seata/seata.properties"
     }
     etcd3 {
       serverAddr = "http://localhost:2379"
     }
     file {
       name = "file.conf"
     }
   }
   ```

4. 在resources下创建``file.conf``（将``seata\seata-server-1.4.2\conf\file.conf``复制过来）

   主要要加上`service`中的内容

   ```
   ## transaction log store, only used in seata-server
   store {
     ## store mode: file、db、redis
     mode = "file"
     ## rsa decryption public key
     publicKey = ""
     ## file store property
     file {
       ## store location dir
       dir = "sessionStore"
       # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
       maxBranchSessionSize = 16384
       # globe session size , if exceeded throws exceptions
       maxGlobalSessionSize = 512
       # file buffer size , if exceeded allocate new buffer
       fileWriteBufferCacheSize = 16384
       # when recover batch read size
       sessionReloadReadSize = 100
       # async, sync
       flushDiskMode = async
     }
   service {
     #vgroup->rgroup
     vgroup_mapping.my_test_tx_group = "default"
     #only support single node
     default.grouplist = "127.0.0.1:8091"
     #degrade current not support
     enableDegrade = false
     #disable
     disable = false
     #unit ms,s,m,h,d represents milliseconds, seconds, minutes, hours, days, default permanent
     max.commit.retry.timeout = "-1"
     max.rollback.retry.timeout = "-1"
   }
     ## database store property
     db {
       ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp)/HikariDataSource(hikari) etc.
       datasource = "druid"
       ## mysql/oracle/postgresql/h2/oceanbase etc.
       dbType = "mysql"
       driverClassName = "com.mysql.jdbc.Driver"
       ## if using mysql to store the data, recommend add rewriteBatchedStatements=true in jdbc connection param
       url = "jdbc:mysql://127.0.0.1:3306/seata?rewriteBatchedStatements=true"
       user = "root"
       password = "why818818818"
       minConn = 5
       maxConn = 100
       globalTable = "global_table"
       branchTable = "branch_table"
       lockTable = "lock_table"
       queryLimit = 100
       maxWait = 5000
     }
   
     ## redis store property
     redis {
       ## redis mode: single、sentinel
       mode = "single"
       ## single mode property
       single {
         host = "127.0.0.1"
         port = "6379"
       }
       ## sentinel mode property
       sentinel {
         masterName = ""
         ## such as "10.28.235.65:26379,10.28.235.65:26380,10.28.235.65:26381"
         sentinelHosts = ""
       }
       password = ""
       database = "0"
       minConn = 1
       maxConn = 10
       maxTotal = 100
       queryLimit = 100
     }
   }
   ```

5. 创建yaml

   ```yaml
   server:
     port: 80
   spring:
     application:
       name: seata-order-service
     cloud:
       alibaba:
         seata:
           tx-service-group: my_test_tx_group
       nacos:
         discovery:
           server-addr: localhost:8848
     datasource:
       driver-class-name: com.mysql.jdbc.Driver
       url: jdbc:mysql://127.0.0.1:3306/seata_order
       username: root
       password: why818818818
   
   feign:
     hystrix:
   ```

6. 创建Order实体类

   ```java
   package com.why.springcloud.alibaba.entity;
   
   import lombok.AllArgsConstructor;
   import lombok.Data;
   import lombok.NoArgsConstructor;
   
   import java.math.BigDecimal;
   
   @Data
   @AllArgsConstructor
   @NoArgsConstructor
   public class Order {
       private Long id;
       private Long userID;
       private  Long productID;
       private  Integer count;
       private BigDecimal money;
       private Integer status;
   }
   ```

7. 创建OrderMapper

   ```java
   package com.why.springcloud.alibaba.mapper;
   
   import com.why.springcloud.alibaba.entity.Order;
   import org.apache.ibatis.annotations.*;
   
   @Mapper
   public interface OrderMapper {
   
       @Insert("INSERT INTO t_order (id, user_id, product_id, count, status) VALUES (#{id}," +
               " #{userID}, #{productID},#{count},#{money},0)")
       @SelectKey(statement = "SELECT UNIX_TIMESTAMP(NOW())", keyColumn = "id", keyProperty = "id", resultType = Long.class, before = true)
       void createOrder(Order order);
   
       @Update("UPDATE t_order SET status = 1" +
               " where user_ud = #{userID} and status = #{status}")
       void updateOrder(@Param("userID") Long userID, @Param("status") Integer status);
   }
   ```

8. 创建OrderService，AccountService，StorageService

   ```java
   package com.why.springcloud.alibaba.service;
   
   import com.why.springcloud.alibaba.entity.Order;
   public interface OrderService {
       void create(Order order);
   }
   ```

   ```java
   package com.why.springcloud.alibaba.service;
   
   import org.springframework.cloud.openfeign.FeignClient;
   import org.springframework.context.annotation.Bean;
   import org.springframework.stereotype.Component;
   import org.springframework.stereotype.Service;
   import org.springframework.web.bind.annotation.PostMapping;
   import org.springframework.web.bind.annotation.RequestParam;
   
   import java.math.BigDecimal;
   
   @FeignClient(value = "seata-account-service")
   //@Component
   public interface AccountService {
       @PostMapping(value = "/account/decrease")
       void decrease(@RequestParam("userID") Long userID, @RequestParam("money") BigDecimal money);
   }
   ```

   ```java
   package com.why.springcloud.alibaba.service;
   
   import org.springframework.cloud.openfeign.FeignClient;
   import org.springframework.context.annotation.Bean;
   import org.springframework.stereotype.Component;
   import org.springframework.stereotype.Service;
   import org.springframework.web.bind.annotation.PostMapping;
   import org.springframework.web.bind.annotation.RequestParam;
   
   //@Component
   @FeignClient(value = "seata-storage-service")
   public interface StorageService {
   
       @PostMapping(value = "/storage/decrease")
       void decrease(@RequestParam("productID") Long productID, @RequestParam("count") Integer count);
   }
   ```

9. 创建OrderServiceImpl(OrderService实现类)

   ```java
   package com.why.springcloud.alibaba.service.impl;
   
   import com.why.springcloud.alibaba.entity.Order;
   import com.why.springcloud.alibaba.mapper.OrderMapper;
   import com.why.springcloud.alibaba.service.AccountService;
   import com.why.springcloud.alibaba.service.OrderService;
   import com.why.springcloud.alibaba.service.StorageService;
   import lombok.extern.slf4j.Slf4j;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.stereotype.Service;
   
   import javax.annotation.Resource;
   
   @Slf4j
   @Service
   public class OrderServiceImpl implements OrderService {
       @Resource
       private OrderMapper orderMapper;
   
       @Resource
       private StorageService storageService;
   
       @Resource
       private AccountService accountService;
   
       @Override
       public void create(Order order) {
           log.info("开始新建订单");
           orderMapper.createOrder(order);
   
           log.info("库存减少");
           storageService.decrease(order.getProductID(), order.getCount());
   
           log.info("money减少");
           accountService.decrease(order.getUserID(), order.getMoney());
   
           orderMapper.updateOrder(order.getUserID(),0);
           log.info("下订单结束");
       }
   }
   ```

10. 创建启动类

    ```java
    package com.why.springcloud.alibaba;
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
    import org.springframework.cloud.openfeign.EnableFeignClients;
    
    @SpringBootApplication
    @EnableDiscoveryClient
    @EnableFeignClients
    public class SeataOrder80 {
        public static void main(String[] args) {
            SpringApplication.run(SeataOrder80.class, args);
        }
    }
    ```

11. 启动



### 3.3	创建库存订单