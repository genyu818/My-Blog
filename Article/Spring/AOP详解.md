# Spring AOP详解

## 1.	什么是AOP

面向切面编程(Aspect-oriented Programming，俗称AOP)提供了一种面向对象编程(Object-oriented Programming，俗称OOP)的补充，面向对象编程最核心的单元是类(class)，然而面向切面编程最核心的单元是切面(Aspects)。与面向对象的顺序流程不同，AOP采用的是横向切面的方式，注入与主业务流程无关的功能，例如事务管理和日志管理。

通俗描述就是：不修改源代码的方式，增加新功能，比如当前我们有一个登录功能，当User登录时，要将登录信息写进一个历史表，但是写入历史表的功能和我们登录的功能并无关联，为了解耦可以使用AOP来实现。

![image-20211020200001229](D:\Blog\My-Blog\pic\image-20211020200001229.png)







## 2.	AOP的概念

在学习SpringAOP 之前，让我们先对AOP的几个基本术语有个大致的概念，这些概念不是很容易理解，比较抽象，可以知道有这么几个概念，下面一起来看一下：

- `切面(Aspect)`： Aspect 声明类似于 Java 中的类声明，事务管理是AOP一个最典型的应用。在AOP中，切面一般使用 `@Aspect` 注解来使用
- `连接点(Join Point)`: 被拦截到的点，因为Spring只支持方法类型的连接点，所以在Spring中连接点指的就是被拦截到的方法，实际上连接点还可以是字段或者构造器
- `通知(Advice):`在切面中(类)的某个连接点(方法出)采取的动作，会有四种不同的通知方式： **around(环绕通知)，before(前置通知)，after(后置通知)， exception(异常通知)，return(返回通知)**。许多AOP框架（包括Spring）将建议把通知作为为拦截器，并在连接点周围维护一系列拦截器。
- `切入点(Pointcut):`表示一组连接点，通知与切入点表达式有关，并在切入点匹配的任何连接点处运行(例如执行具有特定名称的方法)。**由切入点表达式匹配的连接点的概念是AOP的核心，Spring默认使用AspectJ切入点表达式语言。**
- `介绍(Introduction):` introduction可以为原有的对象增加新的属性和方法。例如，你可以使用introduction使bean实现IsModified接口，以简化缓存。
- `目标对象(Target Object):` 由一个或者多个切面代理的对象。也被称为"切面对象"。由于Spring AOP是使用运行时代理实现的，因此该对象始终是代理对象。
- `AOP代理(AOP proxy):` 由AOP框架创建的对象，在Spring框架中，AOP代理对象有两种：**JDK动态代理和CGLIB代理**
- `织入(Weaving):` 是指把增强应用到目标对象来创建新的代理对象的过程，它(例如 AspectJ 编译器)可以在编译时期，加载时期或者运行时期完成。与其他纯Java AOP框架一样，Spring AOP在运行时进行织入。

### 2.2  Spring AOP 中通知的分类

- 前置通知(@Before): 在目标方法被调用前调用通知功能；
- 后置通知(@After): 在目标方法被调用之后调用通知功能；
- 返回通知(@Afterreturning): 在目标方法成功执行之后调用通知功能；
- 异常通知(@AfterThrowing): 在目标方法抛出异常之后调用通知功能；
- 环绕通知(@Around): 把整个目标方法包裹起来，在**被调用前和调用之后分别调用通知功能**



## 2.	如何在Spring 中使用AOP

Spring框架中一般使用AspectJ实现AOP操作

### 2.1	什么是AspectJ

AspectJ不是Spring组成部分, 它是一个独立框架, 一般常把Spring和AspectJ一起使用, 进行AOP操作.

### 2.2	如何使用

#### 2.2.1	基于XML配置(不常使用,略)



#### 2.2.2	基于注解

首先先创建一个简单的登录功能

```java
//	LoginService接口
public interface LoginService {
    boolean login();
}

//	LoginService实现类
@Service
public class LoginServiceImpl implements LoginService {
    @Override
    public boolean login() {
        System.out.println("用户来登录了");
        return true;
    }
}

//	Login接口
public class LoginAOPController {
    @Autowired
    LoginService loginService;

    @GetMapping("/login")
    public void login() {
        loginService.login();
    }
}
```



* 引入AOP相关依赖

  ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-aop</artifactId>
  </dependency>
  ```

  

* 创建增强类

  * 使用@Aspect将当前类标识为一个切面供容器读取

  * 使用 @Pointcut指定连接点

  * 根据需要增强的实际添加章节2.2 中相应的注解

    

  ```java
  package com.springbootstudy.springbootstudy.Proxy;
  
  import org.aspectj.lang.ProceedingJoinPoint;
  import org.aspectj.lang.annotation.*;
  import org.springframework.stereotype.Component;
  
  @Component
  @Aspect
  public class LoginAspect {
  
      @Pointcut("execution(* com.springbootstudy.springbootstudy.service.LoginService.*(..))")
      public void loginPointCut(){
      }
  
      @Before("loginPointCut()")
      public void beforePointCut() {
          System.out.println("before login");
      }
  
      @After("loginPointCut()")
      public void afterPointCut() {
          System.out.println("after login");
      }
  
      @AfterReturning("loginPointCut()")
      public void afterReturningCut() {
          System.out.println("after returning login");
      }
  
      @AfterThrowing("loginPointCut()")
      public void afterThrowingCut() {
          System.out.println("after throwing login");
      }
  
      @Around("loginPointCut()")
      public boolean aroundCut(ProceedingJoinPoint joinPoint) throws Throwable {
          System.out.println("around login begin");
          //  获取增强方法中的参数，在本例中即获取 LoginService中的login参数
          Object[] args = joinPoint.getArgs();
  
          //  使用@Around注解移动要执行 切点的proceed方法，执行该方法才代表被增强方法执行了
          Boolean result = (Boolean) joinPoint.proceed(args);
  
          System.out.println("around login end");
          return  result;
      }
  }
  ```

  ```
  around login begin
  before login
  用户来登录了
  after returning login
  after login
  around login end
  ```

  

## 3.	AOP原理(动态代理)

### 3.1	JDK代理(基于接口)

#### 3.1.1	如何使用JDK代理

1. 创建接口

   来基于JDK动态代理实现上文所说在登录功能里添加写入登陆历史信息的功能

   ```java
   public interface LoginService {
       boolean login();
   }
   ```

2. 创建实现了

   ```java
   public class LoginServiceImpl implements LoginService {
       @Override
       public boolean login() {
           System.out.println("用户来登录了");
           return true;
       }
   }
   ```

3. 创建增强类类实现InvocationHandler并且重写invoke方法,在invoke方法中即可实现代理功能 

   ```java
   public class LoginProxy implements InvocationHandler {
    
       //  创建的是谁的代理对象，就把谁传进来。当前案例应是把 LoginServiceImpl传进来
       private Object object;
       public LoginProxy(Object obj) {
           this.object = obj;
       }
       @Override
       public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
           //  proxy   代理对象
           //  method  代理方法
           //  args    方法参数
   
           //  方法之前执行
           System.out.println("执行代理方法之前");
           //  执行方法
           Object res = method.invoke(object, args);
           System.out.println("将登录信息写入数据库中");
           //  方法之后执行
           System.out.println("执行代理方法之后");
           return res;
       }
   }
   ```

4. 测试

   ```java
   public class JDKProxy {
       public static void main(String[] args) {
           Class[] interfaces = {LoginService.class};
           LoginServiceImpl loginServiceImpl = new LoginServiceImpl();
           LoginService loginService = (LoginService) Proxy.newProxyInstance(JDKProxy.class.getClassLoader(), interfaces, new LoginProxy(loginServiceImpl));
           loginService.login();
       }
   }
   ```

   ```
   执行代理方法之前
   用户来登录了
   将登录信息写入数据库中
   执行代理方法之后
   ```

   



#### 3.2	CGLIB代理(基于对象)

##### 3.2.1	什么是CGLIB

CGLIB是一个功能强大，高性能的代码生成包。它为没有实现接口的类提供代理，为JDK的动态代理提供了很好的补充。通常可以使用Java的动态代理创建代理，但当要代理的类没有实现接口或者为了更好的性能，CGLIB是一个好的选择。



##### 3.2.2	如何使用CGLIB

1. 引入依赖

   ```xml
   <!-- https://mvnrepository.com/artifact/cglib/cglib -->
   <dependency>
       <groupId>cglib</groupId>
       <artifactId>cglib</artifactId>
   </dependency>
   ```

   

2. 创建实体类

   ```java
   public class Login {
       public boolean login(){
           System.out.println("登录了");
           return true;
       }
   }
   ```

3. 定义拦截器实现MethodInterceptor并且重写intercept方法

   ```java
   public class LoginInterceptor implements MethodInterceptor {
       @Override
       public Object intercept(Object obj, Method method, Object[] params, MethodProxy proxy) throws Throwable {
           System.out.println("login 前");
           //	利用反射执行需要被增强类中的方法
           Object result = proxy.invokeSuper(obj, params);
           System.out.println("login 后");
           return result;
       }
   }
   ```

4. 生成动态代理类

   ```java
   public class LoginCGLIBTest {
       public static void main(String[] args) {
           //	创建增强器
           Enhancer enhancer =new Enhancer();
           //	注入需要被增强的类
           enhancer.setSuperclass(Login.class);
           //	注入拦截器
           enhancer.setCallback(new LoginInterceptor());
           //	窗前被增强对象
           Login login = (Login) enhancer.create();
           login.login();
       }
   }
   ```

   ```
   login 前
   登录了
   login 后
   ```



##### 3.2.3	回调过滤器CallbackFilte 

再扩展一点点，比方说在AOP中我们经常碰到的一种复杂场景是：**我们想对类A的B方法使用一种拦截策略、类A的C方法使用另外一种拦截策略**。

在本例中，我们在Login类里再加一个方法，即通过账号获取密码，我们需要对两个方法采取两种拦截策略，测试就需要到回调过滤器。

1. 在被增强类中增加一个方法

   ```java
   public class Login {
       public boolean login(){
           System.out.println("登录了");
           return true;
       }
   
       public String getPassWord(String userID) {
           return "SMIT";
       }
   }
   ```

2. 再定义getPassWord方法的拦截器

   ```java
   public class GetPassWordInterceptor implements MethodInterceptor {
       @Override
       public Object intercept(Object obj, Method method, Object[] params, MethodProxy proxy) throws Throwable {
           System.out.println("getPassword 前");
           Object result = proxy.invokeSuper(obj, params);
           System.out.println("getPassword 后");
           return result;
       }
   
   }
   ```

3. 定义CallbackFilter类

   ```java
   public class LoginCallbackFilter implements CallbackFilter {
       @Override
       public int accept(Method method) {
           //	CallbackFilter的accept方法返回的数值表示的是顺序，顺序和setCallbacks里面Proxy的顺序是一致的
           //	可结合下面的动态嗲李磊来看
           if (method.getName().equals("getPassWord")) {
               System.out.println("getPassWord方法被调用");
               return 0;
           }
           if (method.getName().equals("login")){
               System.out.println("login方法被调用");
               return 1;
           }
           return 0;
       }
   }
   ```

4. 定义动态代理类

   ```java
   public class LoginCGLIBTest {
       public static void main(String[] args) {
           LoginInterceptor loginInterceptor = new LoginInterceptor();
           GetPassWordInterceptor getPassWordInterceptor =new GetPassWordInterceptor();
           Enhancer enhancer =new Enhancer();
           enhancer.setSuperclass(Login.class);
           //	方法名为"getPassWord"的方法返回的顺序为0，即使用Callback数组中的0位callback，即getPassWordInterceptor
           //	方法名不为"login"的方法返回的顺序为1，即使用Callback数组中的1位callback，loginInterceptor
           //	NoOp.INSTANCE，这表示一个空Callback，即如果不想对某个方法进行拦截，可以在LoginCallbackFilter中返回2
           enhancer.setCallbacks(new Callback[]{getPassWordInterceptor,loginInterceptor, NoOp.INSTANCE});
           enhancer.setCallbackFilter(new LoginCallbackFilter());
           Login login = (Login) enhancer.create();
           login.login();
           login.getPassWord("wang");
       }
   }
   ```

5. 结果

   ```
   getPassWord方法被调用
   login方法被调用
   getPassword 前
   getPassword 后
   login 前
   登录了
   login 后
   ```

   