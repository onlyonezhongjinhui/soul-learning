## RateLimiter插件

限流插件，是网关对流量管控限制核心的实现。可以到接口级别，也可以到参数级别。

## 技术方案

采用redis令牌桶算法进行限流。

![limiting.png](assets/20210123223018-fa0q0ss-limiting.png)

## 什么是令牌桶算法?

令牌桶算法是一种限流算法，他与漏桶算法的实现是一种相反的实现。

漏桶算法是按照一定频率的速率进行漏水，然后对于我们的请求就可以想象成上边的水龙头。

![00831rSTly1gdnjduhuivj30cb08b74u.jpg](assets/20210123224741-eyjf3bv-00831rSTly1gdnjduhuivj30cb08b74u.jpg)

令牌桶算法则是定时的往桶中放入令牌，然后每次请求都会从令牌桶中获取一个令牌，如果桶中没有令牌，则拒绝请求或者阻塞直到令牌可以获得。

![00831rSTly1gdnjhiarxgj30bp06pwek.jpg](assets/20210123224827-hu1tklg-00831rSTly1gdnjhiarxgj30bp06pwek.jpg)

## 牛刀小试

* soul-examples-http中引入RateLimiter插件

```xml
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-ratelimiter</artifactId>
            <version>${soul.version}</version>
        </dependency>
```

* admin中开启RateLimiter插件

![微信截图20210123225355.png](assets/20210123225914-6aq1hcp-%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20210123225355.png)

* 配置限流选择器和规则，两个重要的蚕食capacity、rate。

**rate：是你允许用户每秒执行多少请求，而丢弃任何请求。这是令牌桶的填充速率。**

**capacity：是允许用户在一秒钟内执行的最大请求数。这是令牌桶可以保存的令牌数。**

![微信截图20210123225640.png](assets/20210123230000-t65v5sm-%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20210123225640.png)

用jmeter压测一番，看看配置是否好用

![微信截图20210123230854.png](assets/20210123230924-4raiu8r-%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20210123230854.png)

从图中可看到99.03%的错误率，且错误请求响应参数为

```json
{"code":429,"message":"You have been restricted, please try again later!","data":null}
```

看来配置确实生效了。那么容量调大点再压

![1113.png](assets/20210123231710-d6ewkti-1113.png)

![4441.png](assets/20210123231722-yah20mv-4441.png)

这下错误率直接降为26.49%了。就这么简单，一个限流的功能就整起来了，非常方便灵活，并且随时可以调整配置，这就是soul网关强大之处呀。