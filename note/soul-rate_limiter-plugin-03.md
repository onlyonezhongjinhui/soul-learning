上一篇了解了`RateLimiter`插件的执行流程。讲到soul基于redis实现的令牌桶，是通过lua脚本实现。这篇我们就再细谈令牌桶算法。

## 令牌桶算法由来

令牌桶算法最初来源于计算机网络。在网络传输数据时，为了防止网络拥塞，需限制流出网络的流量，使流量以比较均匀的速度向外发送。令牌桶算法就实现了这个功能，可控制发送到网络上数据的数目，并允许突发数据的发送。令牌桶算法是网络流量整形（Traffic Shaping）和速率限制（Rate Limiting）中最常使用的一种算法。典型情况下，令牌桶算法用来控制发送到网络上的数据的数目，并允许突发数据的发送。大小固定的令牌桶可自行以恒定的速率源源不断地产生令牌。如果令牌不被消耗，或者被消耗的速度小于产生的速度，令牌就会不断地增多，直到把桶填满。后面再产生的令牌就会从桶中溢出。最后桶中可以保存的最大令牌数永远不会超过桶的大小。传送到令牌桶的数据包需要消耗令牌。不同大小的数据包，消耗的令牌数量不一样。令牌桶这种控制机制基于令牌桶中是否存在令牌来指示什么时候可以发送流量。令牌桶中的每一个令牌都代表一个字节。如果令牌桶中存在令牌，则允许发送流量；而如果令牌桶中不存在令牌，则不允许发送流量。因此，如果突发门限被合理地配置并且令牌桶中有足够的令牌，那么流量就可以以峰值速率发送。

## 令牌桶算详细描述

首先令牌桶算法涉及到两个最重要的参数填充速率（rate）、令牌桶最大容量（capacity）。假如rate=r，capacity=c：

1. 每隔1/c秒就会产生一个令牌放入令牌桶中，也就是每秒放入c个令牌。
2. 如果消费令牌的速率小于产生令牌的速率，当令牌桶中的令牌满了，也就是数量到达了c，那么新产生的令牌将会被丢弃。
3. 不同的数据包需要消耗不同数量的令牌。如果有n个字节的数据包到达时，就从令牌桶中删除n个令牌。
4. 如果令牌桶中的令牌数量小于n，不会删除令牌,该数据包先缓存或者丢弃。
5. 当有突发流量来了，最多能处理c个字节。

## 令牌桶实现要点

1.记录最大令牌桶容量、当前剩余令牌数量、下次可填充令牌的时间。

2.响应完一个请求之后并计算下次可填充令牌的时间，如果下一个请求在这个时间之前则需要等待。如果设置rate为1，那么要处理这个请求就得在一秒之后了。

3.支持突发流量的处理，且最大突发流量为令牌桶的最大容量。如果rate设置为1，capacity为10，如果10秒内没有消耗令牌，则如果有突发流量到来，则一下子可以获取到10个令牌。

## 令牌生产方式

1.通过定时任务填充。此方式有明显的劣势，定时任务需要消耗系统资源，如果需要维护很多令牌桶，则消耗巨大。

2.延迟生成。在每次获取令牌的时候进行一次计算，计算下次可填充令牌时间到当前时间的间隔内可产生的令牌数量，并更令牌桶。

## `RateLimiter`具体实现

```lua
// 令牌桶key
local tokens_key = KEYS[1]
// 可填充令牌时间key
local timestamp_key = KEYS[2]
// 填充速率
local rate = tonumber(ARGV[1])
// 令牌桶最大容量
local capacity = tonumber(ARGV[2])
// 请求令牌时间
local now = tonumber(ARGV[3])
// 请求令牌数量
local requested = tonumber(ARGV[4])
// 填充满整个令牌桶所需要的时间
local fill_time = capacity/rate
// 令牌可累计的时长
local ttl = math.floor(fill_time*2)
// 查询出令牌桶当前剩余的令牌数量，没有则为0
local last_tokens = tonumber(redis.call("get", tokens_key))
if last_tokens == nil then
  last_tokens = capacity
end
// 查询可填充令牌时间，没有则为0
local last_refreshed = tonumber(redis.call("get", timestamp_key))
if last_refreshed == nil then
  last_refreshed = 0
end
// 计算可填充令牌时间到当前时间的间隔，如果本次请求在可填充令牌时间之前，则为0
local delta = math.max(0, now-last_refreshed)
// 计算出令牌数量=剩余令牌+间隔时间内可生成的令牌数量，不能超过令牌桶最大容量
local filled_tokens = math.min(capacity, last_tokens+(delta*rate))
// 判断当前请求是否令牌数量是否足够
local allowed = filled_tokens >= requested
local new_tokens = filled_tokens
local allowed_num = 0
// 如果足够则删除请求的令牌数量
if allowed then
  new_tokens = filled_tokens - requested
  allowed_num = 1
end

// 更新令牌桶
redis.call("setex", tokens_key, ttl, new_tokens)
redis.call("setex", timestamp_key, ttl, now)
// 响应令牌请求结果，0表示未获得令牌，1表示成功获得令牌
return { allowed_num, new_tokens }
```

## 总结

1.使用lua操作redis实现令牌桶算法，利用redis单线程操作内存的特性式使其具有原子性保持高性能。

2.rate、capacity、nextFreeTicketMillis是整个算法的核心。尤其nextFreeTicketMillis比较拗口。