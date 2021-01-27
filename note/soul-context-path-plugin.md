网关context_path插件，根据文档说法，是用来对目标服务调用的时候，重写请求路径的contextPath。

我们就来试试吧。

## context_path插件设置

1. 网关引入context_path插件

```xml
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-context-path</artifactId>
            <version>${project.version}</version>
        </dependency>
```

2.启动网关后在admin中开启context_path插件

![微信截图20210128004129.png](assets/20210128004213-o7s9vqe-%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20210128004129.png)

3.配置插件选择器和规则

![微信截图20210128004646.png](assets/20210128004721-o38rcag-%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20210128004646.png)![微信截图20210128004656.png](assets/20210128004721-kt8iuf9-%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20210128004656.png)

4.尝试请求订单接口

![微信截图20210128004754.png](assets/20210128004815-oypa6xh-%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20210128004754.png)

从图中可看到，已经请求成功，插件生效了。

## context_path插件使用场景

当匹配到请求后，设置自定义的contextPath，那么就会根据请求的Url截取自定义的contextPath获取真正的Url。大白话就是真实请求的路径=请求路径删除前缀contextPath。例如请求路径为/soul/http/order， 配置的contextPath为/soul/http，那么真正请求的url为/order。可以对真实服务做一个隐藏、也可以用来替换老接口。