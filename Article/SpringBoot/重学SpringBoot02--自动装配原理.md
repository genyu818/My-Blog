# SpringBoot自动装配原理

我们首先创建一个SpringBoot WEB应用，我们已经知道在包下有一个启动类

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
        SpringApplication.run(SpringbootstudyApplication.class, args);
    }
}

```

以上简单几行代码就是我们Spring Boot启动类的所有内容，唯一比较特别的就是一个注解 ``@SpringBootApplication``，我们点进去看一下。

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
```

以上就是整个注解的核心内容，我们一一来解释



## 1.	@SpringBootConfiguration

我们来点进来看一下这个注解的内容

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
@Indexed
public @interface SpringBootConfiguration {
    @AliasFor(
        annotation = Configuration.class
    )
    boolean proxyBeanMethods() default true;
}
```

可以看到@SpringBootConfiguration注解没什么特别，就是声明这个一个注解类



## 2.	@ComponentScan

这个注解咱们都是比较熟悉的，无非就是自动扫描并加载符合条件的Bean到容器中，这个注解会默认扫描声明类所在的包开始扫描。

这个注解一共包含以下几个属性：

```
basePackages：指定多个包名进行扫描
basePackageClasses：对指定的类和接口所属的包进行扫
excludeFilters：指定不扫描的过滤器
includeFilters：指定扫描的过滤器
lazyInit：是否对注册扫描的bean设置为懒加载
nameGenerator：为扫描到的bean自动命名
resourcePattern：控制可用于扫描的类文件
scopedProxy：指定代理是否应该被扫描
scopeResolver：指定扫描bean的范围
useDefaultFilters：是否开启对@Component，@Repository，@Service，@Controller的类进行检测
```



## 3.	@EnableAutoConfiguration

接下来就是我们的重头戏，`@EnableAutoConfiguration`注解我们用英语直译，就是开启自动配置

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}

```



该注解下主要有两个注解我们来分别看一下

#### 3.1 	@AutoConfigurationPackage

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import({Registrar.class})
public @interface AutoConfigurationPackage {
    String[] basePackages() default {};

    Class<?>[] basePackageClasses() default {};
}


//该注解主要作用就是Import Registrar.class这个组件，我们继续看一下这个组件的内容

//该组件下有两个方法
 static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {
        Registrar() {
        }
		
        public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
            AutoConfigurationPackages.register(registry, (String[])(new 			AutoConfigurationPackages.PackageImports(metadata)).getPackageNames().toArray(new String[0]));
        }

        public Set<Object> determineImports(AnnotationMetadata metadata) {
            return Collections.singleton(new AutoConfigurationPackages.PackageImports(metadata));
        }
    }
```



我们来打一下断点看一下这个Registrar方法，可以看到断点只进到这个registerBeanDefinitions函数中，

来看一下参数metadata，可以看到这个参数 即是我们的启动类，即可得知注解是落在了我们的启动类中。

```metadata : com.springbootstudy.springbootstudy.SpringbootstudyApplication```



根据下面源码分析可知，registerBeanDefinitions方法就是扫描我们主包下的所有组件并且注册

```java
public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
            AutoConfigurationPackages.register(registry, (String[])(new AutoConfigurationPackages.PackageImports(metadata)).getPackageNames().toArray(new String[0]));
        }

//	使用idea自动的evaluate
//	(new AutoConfigurationPackages.PackageImports(metadata)).getPackageNames() = com.springbootstudy.springbootstudy
//	我们看到该方法即，把我们包下的所有组件封装成一个数组并且注册


//	我们点进 AutoConfigurationPackages.register(),看到该方法就是把我们刚才封装成Array的所有包，注册进beanDefinition中
public static void register(BeanDefinitionRegistry registry, String... packageNames) {
    if (registry.containsBeanDefinition(BEAN)) {
            AutoConfigurationPackages.BasePackagesBeanDefinition beanDefinition = 				(AutoConfigurationPackages.BasePackagesBeanDefinition)registry.getBeanDefinition(BEAN);
            beanDefinition.addBasePackages(packageNames);
        } 
    else {
            registry.registerBeanDefinition(BEAN, new AutoConfigurationPackages.BasePackagesBeanDefinition(packageNames));
        }

    }
```



#### 3.2	@Import({AutoConfigurationImportSelector.class})

点进来翻到selectImports这个方法，经过一下源码，我们得知该注解就是引入我们项目中所有``META-INF/spring.factories``下的组件。

```java
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!this.isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        } else {
            //
            AutoConfigurationImportSelector.AutoConfigurationEntry autoConfigurationEntry = this.getAutoConfigurationEntry(annotationMetadata);
            // 最终将AutoConfigurationEntry的所有配置封装成数组返回，即我们只需要搞懂getAutoConfigurationEntry这个方法即可
            return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
        }
    }

//	看一下getAutoConfigurationEntry这个方法
protected AutoConfigurationImportSelector.AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
    // annotationMetadata : com.springbootstudy.springbootstudy.SpringbootstudyApplication
        if (!this.isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        } else {
            AnnotationAttributes attributes = this.getAttributes(annotationMetadata);
            // 获得我们包下的所有组件共131个
            List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);
            // 对刚才获取的注解去重
            configurations = this.removeDuplicates(configurations);
            
            //	检查我们标注的不想要注册的注解
            Set<String> exclusions = this.getExclusions(annotationMetadata, attributes);
            this.checkExcludedClasses(configurations, exclusions);
            configurations.removeAll(exclusions);
            configurations = this.getConfigurationClassFilter().filter(configurations);
            this.fireAutoConfigurationImportEvents(configurations, exclusions);
            
            //	返回我们所有的自动配置组件
            return new AutoConfigurationImportSelector.AutoConfigurationEntry(configurations, exclusions);
        }
    }

//	List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);获取所有自动注册的组件
//	打一下断点看一下getCandidateConfigurations方法

protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    //	利用工厂模式加载所有的组件
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
        Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
        return configurations;
    }

//	继续看一下这个方法  SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());

//	这个方法太长我们只截取核心，核心就是加载META-INF/spring.factories组件
private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader){
    
            try {
                Enumeration urls = classLoader.getResources("META-INF/spring.factories");
                ......
                    ....
                    ....
}

```



我们来验证一下，我们看到我们引入的JAR包中有很多都含有这个文件``META-INF/spring.factories``



![image-20210916201842818](..\spring,factories)

其中最核心的就是``spring-autoconfigure``中的spring.factories，我们来打开看一下。

内容太多(``共有132个AutoConfiguration``)我们只截取一部分，我们可以看到里边包含了，近乎所有我们可以使用的JAR包。

```xml
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.LifecycleAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration,\
org.springframework.boot.autoconfigure.dao.PersistenceExceptionTranslationAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraReactiveDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraReactiveRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseReactiveDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseReactiveRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ReactiveElasticsearchRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ReactiveElasticsearchRestClientAutoConfiguration,\
org.springframework.boot.autoconfigure.data.jdbc.JdbcRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration,\
...
...

```



#### 3.3 Spring Boot按需加载

如我们刚才的分析，SpringBoot中共有131个AutoConfiguration，难道我们每次启动SpringBoot的项目都加载了这个131个组件吗？

这就要说到我们的按需加载，我们来随便点进一个``org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration``，从名字就可得知，这是一个Spring的消息队列组件，

我们点进去之后很神奇的看到一个``@ConditionalOnClass({RabbitTemplate.class, Channel.class})``，

那就代表我们必须要导入org.springframework.amqp.rabbit.core.RabbitTemplate;，该组件才能生效。

```java
import org.springframework.amqp.rabbit.core.RabbitTemplate;

@ConditionalOnClass({RabbitTemplate.class, Channel.class})
@EnableConfigurationProperties({RabbitProperties.class})
@Import({RabbitAnnotationDrivenConfiguration.class})
public class RabbitAutoConfiguration {
```



感兴趣的朋友可以多翻翻几个AutoConfiguration，可以发现几何所有的都会有@Conditional注解，这表明自动装配时，SpringBoot会注册所有组件，但是会按需进行加载。



## 4.	总结

SpringBoot的自动装配主要依赖于@EnableAutoConfiguration，该注解下又有两个注解

* @AutoConfigurationPackage：注册我们启动类所有包下的组件进beanDefinition中
* @Import({AutoConfigurationImportSelector.class})： 自动注册``spring-autoconfigure``中的spring.factories下的100多个组件，但是会按需加载

