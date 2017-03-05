---
title: 一个简单的 dubbo 服务
date: 2017-03-05 20:34
categories: [技术]
tags: [dubbo, maven]
---

这篇文章将简单的介绍如何写一个 dubbo 服务。dubbo 的准备工作请参照[前文](/2017/03/intro-dubbo/)。

![dubbo 架构](http://wulfric.qiniudn.com/dubbo/dubbo-architecture.png "dubbo 架构")

从此图可知，当注册中心和监控已经工作起来之后，我们需要写的就是消费方和服务方的代码了。一个简单的一对一提供服务的代码结构如下[^1]：

```shell
~/IdeaProjects/dubbox/dubbo-demo master
> ls
dubbo-demo-api      dubbo-demo-provider pom.xml
dubbo-demo-consumer dubbo-demo.iml
```

其中api 部分定义了 provider 和 consumer 都需要用到的接口，这样 provider 实现这些接口，consumer 就可以像本地调用一样来调用这些接口了。

## 接口定义与步骤

该 dubbo 服务将实现获取权限数组的功能，即调用`PermissionService.getPermissions`获取权限数组。在本文的例子中，我们将实现 1) 直接返回字符串的数组；2) 返回 java 对象（POJO） 的  json/xml 序列化的结果。

```shell
# 直接返回字符串的数组
['permission_1', 'permission_2', 'permission_3']
# 返回 json/xml 序列化的结果
[
	{
      "id": 1,
      "name": "x1_permission",
	},
	{
      "id": 2,
      "name": "x2_permission",
	}
]
```

### 协议分类

dubbo 的服务方和提供方通信支持多种协议，默认的是 dubbo 协议。官方给出的协议建议为：

| 协议      | 优势                                       | 劣势                         |
| ------- | ---------------------------------------- | -------------------------- |
| Dubbo   | 采用NIO复用单一长连接，并使用线程池并发处理请求，减少握手和加大并发效率，性能较好（推荐使用） | 在大文件传输时，单一连接会成为瓶颈          |
| Rmi     | 可与原生RMI互操作，基于TCP协议                       | 偶尔会连接失败，需重建Stub            |
| Hessian | 可与原生Hessian互操作，基于HTTP协议                  | 需hessian.jar支持，http短连接的开销大 |

除此之外，当当还通过扩展 dubbo 得到 dubbox，支持了 HTTP 的 Rest 接口。关于 Rest 的优缺点参见 dubbox 的[介绍](https://dangdangdotcom.github.io/dubbox/rest.html)。本文使用的就是 dubbox。

### 步骤

我们将循序渐进的实现一个获取权限数组的服务。

1. [x] 通过 dubbo 协议实现获取权限数组的服务
2. [ ] 通过 rest 规范实现获取权限数组的服务
3. [ ] 通过 rest 规范实现获取权限 POJO 序列化结果的服务
4. [ ] 将提供方以服务的形式部署到服务器

## dubbo 协议简例

创建 maven 项目 dubbo-test，编辑 pom 文件，并仿照官方示例，在 maven 项目下分别创建 3 个 module：api, provider 和 consumer。

```xml
<!-- dubbo test 的 pom 文件直接用默认生成的即可 -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>dubbotest</groupId>
    <artifactId>dubbotest</artifactId>
    <version>1.0-SNAPSHOT</version>
</project>
```

### 创建 api module

通过 IDE 创建 api 子模块（或者手动创建文件夹），并创建 api.PermissionService 类：

```java
package api;
import java.util.List;
public interface PermissionService {
    List<String> getPermissions(Long id);
}
```

文件夹结构如下：

```
.
├── api
│   ├── pom.xml
│   └── src
│       ├── main
│       │   ├── java
│       │   │   └── api
│       │   │       └── PermissionService.java
│       │   └── resources
│       └── test...
├── pom.xml
└── src...
```

### 创建 provider module

用同样的方式创建子模块，并编辑 pom 文件，引入 dubbo 等必要的依赖包：

```xml
...
    <dependencies>
        <!-- 引入 api，在 provider 中实现 api 定义的接口 -->
        <dependency>
            <groupId>dubbotest</groupId>
            <artifactId>api</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
        <!-- 引入 dubbo -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.8.4</version>
        </dependency>
        <!-- 引入 log4j -->
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.17</version>
        </dependency>
        <!-- 引入 zookeeper client -->
        <dependency>
            <groupId>com.github.sgroschupf</groupId>
            <artifactId>zkclient</artifactId>
            <version>0.1</version>
        </dependency>
    </dependencies>
...
```

阿里的Dubbo框架已经集成了 Zookeeper、Spring 等框架的依赖，但是有一个例外就是zkclient，如果没有引用将会抛出如下异常信息：

```
Exception in thread "main" java.lang.NoClassDefFoundError: org/I0Itec/zkclient/exception/ZkNoNodeException
...
```

创建 provider.DemoProvider 类：

```java
package provider;
import java.io.IOException;
public class DemoProvider {
    public static void main(String[] args) throws IOException {
      	// 如果 spring 配置文件的位置是默认的，则可以直接这样启动服务
        com.alibaba.dubbo.container.Main.main(args);
    }
}
```

如果 spring 配置文件在默认的 resources/META-INF/spring 下，则可以直接这样启动服务，否则需要声明指定其位置：

```java
// 建议使用上一种方法
package provider;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import java.io.IOException;
public class DemoProvider {
    public static void main(String[] args) throws IOException {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("classpath*:META-INF/spring/*.xml");
        System.out.println(context.getDisplayName() + ": here");
        context.start();
        System.out.println("服务已经启动...");
        System.in.read();
    }
}
```

创建 api 中定义 api.PermissionService 接口的实现类 provider.PermissionServiceImpl：

```java
package provider;
import api.PermissionService;
import java.util.ArrayList;
import java.util.List;
public class PermissionServiceImpl implements PermissionService {
    public List<String> getPermissions(final Long id) {
      	// 该函数应该实现 getPermissions 的业务逻辑，这里简单返回一个 list
        List<String> permissions = new ArrayList<String>() {{
            add(String.format("Permission_%d", id - 1));
            add(String.format("Permission_%d", id));
            add(String.format("Permission_%d", id + 1));
        }};
        return permissions;
    }
}
```

创建 spring 配置文件，重点是 dubbo 相关服务的配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo
       http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
    <!--定义了提供方应用信息，用于计算依赖关系；在 dubbo-admin 或 dubbo-monitor 会显示这个名字，方便辨识-->
    <dubbo:application name="demotest-provider" owner="programmer" organization="dubbox"/>
    <!--使用 zookeeper 注册中心暴露服务，注意要先开启 zookeeper-->
    <dubbo:registry address="zookeeper://localhost:2181"/>
    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880" />
    <!--使用 dubbo 协议实现定义好的 api.PermissionService 接口-->
    <dubbo:service interface="api.PermissionService" ref="permissionService" protocol="dubbo" />
    <!--具体实现该接口的 bean-->
    <bean id="permissionService" class="provider.PermissionServiceImpl"/>
</beans>
```

最后，在 resources 目录下创建 log4j 的配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE log4j:configuration SYSTEM "log4j.dtd">
<log4j:configuration xmlns:log4j="http://jakarta.apache.org/log4j/" debug="false">
    <appender name="CONSOLE" class="org.apache.log4j.ConsoleAppender">
        <layout class="org.apache.log4j.PatternLayout">
            <param name="ConversionPattern" value="[%d{dd/MM/yy hh:mm:ss:sss z}] %t %5p %c{2}: %m%n" />
        </layout>
    </appender>
    <root>
        <level value="INFO" />
        <appender-ref ref="CONSOLE" />
    </root>
</log4j:configuration>
```

文件创建完毕，接下来在根目录下执行`mvn clean package`，然后执行 provider.DemoProvider 的 main 函数，即可启动该服务了。查看 monitor/admin，看我们定义的 demotest-provider 这个名称的提供者是否出现在服务列表中；或者在 console 输出中发现如下信息，即表示成功启动：

```
...
[05/03/17 11:56:43:043 CST] main  INFO container.Main:  [DUBBO] Dubbo SpringContainer started!, dubbo version: 2.8.4, current host: 127.0.0.1
[2017-03-05 23:56:43] Dubbo service server started!
```

provider 的文件结构如下：

```
.
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── provider
│   │   │       ├── DemoProvider.java
│   │   │       └── PermissionServiceImpl.java
│   │   └── resources
│   │       ├── META-INF
│   │       │   └── spring
│   │       │       └── dubbotest-provider.xml
│   │       └── log4j.xml
│   └── test...
└── target...
```

### 创建 consumer module

consumer 子模块的创建和 provider 很像。pom 文件：

```xml
    <dependencies>
        <dependency>
            <!-- 引入 api，在 consumer 中调用 api 定义的接口 -->
            <groupId>dubbotest</groupId>
            <artifactId>api</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.8.4</version>
        </dependency>
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.17</version>
        </dependency>
        <dependency>
            <groupId>com.github.sgroschupf</groupId>
            <artifactId>zkclient</artifactId>
            <version>0.1</version>
        </dependency>
    </dependencies>
```

创建 spring 配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
    <dubbo:application name="demotest-consumer" owner="programmer" organization="dubbox"/>
  	<!--向 zookeeper 订阅 provider 的地址，由 zookeeper 定时推送-->
    <dubbo:registry address="zookeeper://localhost:2181"/>
    <!--使用 dubbo 协议调用定义好的 api.PermissionService 接口-->
    <dubbo:reference id="permissionService" interface="api.PermissionService"/>
</beans>
```

创建 consumer.DemoConsumer 类：

```java
package consumer;
import api.PermissionService;
import org.springframework.context.support.ClassPathXmlApplicationContext;
public class DemoConsumer {
    public static void main(String[] args) {
        //测试常规服务
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("classpath*:META-INF/spring/*.xml");
        context.start();
        PermissionService permissionService = context.getBean(PermissionService.class);
        System.out.println(permissionService.getPermissions(1L));
    }
}
```

log4j 配置相同。

执行`mvn clean package`，然后执行 consumer.DemoConsumer 的 main 函数，即可启动该服务了。查看 monitor/admin，看我们定义的 demotest-consumer 这个名称的提供者是否出现在服务列表中；或者在 console 输出中发现如下信息，即表示成功启动（输出的数组即为调用的结果）：

```
...
[Permission_0, Permission_1, Permission_2]
```

可见，在 consumer 中调用 provider 的实现，代码上看起来和本地调用一样，即 provider 相对于 consumer 来说是透明的。

<!--## dubbo rest 接口

rest 协议

注释不错，前面介绍不错：http://blog.csdn.net/u012049463/article/details/12161923

主要参考，代码和较为清晰的目录结构：http://www.cnblogs.com/yjmyzz/p/dubbox-demo.html

还有些依赖关系需要

dubbo 协议和 rest 协议的区别，rest 协议需要@Path



api, provider, consumer 关系

-->



[^1]: 该段示例来自 dubbox 的源码