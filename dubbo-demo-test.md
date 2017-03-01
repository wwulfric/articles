## 写一个简单的 dubbo 服务

http://dubbo.io/Administrator+Guide-zh.htm#AdministratorGuide-zh-%E7%A4%BA%E4%BE%8B%E6%8F%90%E4%BE%9B%E8%80%85%E5%AE%89%E8%A3%85

注释不错，前面介绍不错：http://blog.csdn.net/u012049463/article/details/12161923

主要参考，代码和较为清晰的目录结构：http://www.cnblogs.com/yjmyzz/p/dubbox-demo.html

zkclient

http://www.codingjabber.com/Article/397.html：阿里的Dubbo框架已经集成了Zookeeper、Spring等框架所以无须再添加这些框架的引用，但是有一个例外就是zkclient，如果没有引用将会抛出如下异常信息

```
Exception in thread "main" java.lang.NoClassDefFoundError: org/I0Itec/zkclient/exception/ZkNoNodeException
```

还有些依赖关系需要

dubbo 协议和 rest 协议的区别，rest 协议需要@Path



api, provider, consumer 关系

配置 log4j