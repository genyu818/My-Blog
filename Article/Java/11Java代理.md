# CGLIB代理 & JDK动态代理

## 1.	什么是代理

今天我们来讲一下Java的动态代理，Java代理是Spring AOP中的核心部分，我们先来谈谈什么是代理？

最简单的从字面上来看，`代理`就是别人帮我来做这件事，举个例子，最近非常火的计量单位1<font color='red'>爽</font>事件，那么爽妹子肯定不是自己天天在算自己收几<font color='red'>爽</font>，它会雇一批专业的财会人员来，那这些财会人员就是帮爽妹子来代理收<font color='red'>爽</font>。放我我们程序来看，代理的目的就是用与原对象不相关的对象来实现增强原对象的功能(实在是太拗口了)。

再来举个例子，我们部门最近为Litho(负责光刻的部门，贼贼贼有钱)开发一套系统，我们老板为了统计效益，需要计算每天有多少人使用该系统，我又不想修改现在任何的业务代码，那么我就可以通过代理的方式来实现。

总结一下：从我个人自己的理解，代理是在一个对象的生命周期内，在这个不改动对象前提下来增强该类，为它增加功能，最重要的目的是<font color=red>解耦</font>，即不让与业务无关的代码写到类中。常用的比如：记录登录，日志写入等等。



## 2.	Java中的代理

Java中实现代理的方式主要有两种，一是基于JDK的动态代理，一种是CGLIB代理。 

JDK的动态代理主要是使`java.lang.reflect`(反射)来实现。CGLIB主要是通过CGLIB库实现，比JDK动态代理更加强大，也更加复杂。下面我来实际操作一下如何使用代理



### 2.1	JDK动态代理

注意JDK动态代理 只能作用于实现接口的类，这一点非常重要。

那么，我们直接从实现之前说的实现记录我们系统的登录记录开始。因为JDK动态代理只能作用于实现接口类，我们先定义一个接口`IuserDao`只有一个Login方法。

```java
package proxy.Entity;

public interface IUserDao
{
    public String login(String userName);
}
```



然后我们继承这个类来实现Login方法，这里很简单，如果userName是我的姓氏就可以登录成功。

```java
package proxy.Entity;

public class UserDao implements IUserDao{
    @Override
    public String login(String userName) {
        if (userName == "wang") {
            System.out.println("登录成功");
            return "SUCCESS";
        }
        return "ERROR";
    }
}
```



重点来了，我们来创建代理类，需要继承`InvocationHandler`并且重写`invoke`方法。我们就是成功登录后，我们将用户信息写入数据库，详细内容请看注释。

```java
package proxy.Entity;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class UserLoginRecordProxy implements InvocationHandler {
    //维护Object对象，可以理解为需要反射的类，我们代码就是UserDao，这里建议打断点看一下，
    private Object obj;
	
    //构造方法
    public UserLoginRecordProxy(Object obj){
        this.obj = obj;
    }


    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("----登陆前置代码提示--------");
        //反射
        Object target = method.invoke(obj, args);
        System.out.println("----登陆完成代码提示--------");
        String res = target.toString();
        if (res == "SUCCESS") {
            //获取反射执行结果
            //省略写入数据库的逻辑
            System.out.println("----在此记录用户登录信息，写入数据库--------");

        }
        return target;
    }
}
```



现在我们来测试一下我们的代码

```java
package proxy.Entity;

import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Proxy;


public class TestProxy {

    public static void main(String[] args) {
        /*
        参数：1.类加载器
             2.类实例
             3.自定义InvocationHandler
         */
        IUserDao proxyClass = (IUserDao) Proxy.newProxyInstance(IUserDao.class.getClassLoader(),
                new Class[]{IUserDao.class}, new UserLoginRecordProxy(new UserDao()));
        proxyClass.login("wang");
    }
}
```

测试结果成功

```bash
----登陆前置代码提示--------
登录成功
----登陆完成代码提示--------
----在此记录用户登录信息，写入数据库--------
```

