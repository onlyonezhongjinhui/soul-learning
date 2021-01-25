上一篇我们已经了解了`RateLimiter`插件的大致原理和使用方法。今天我们再从微观层面再来探个究竟。

请求来了以后由`SoulWebHandler`进行进行处理，把封装请求信息的对象ServerWebExchange放入

`DefaultSoulPluginChain`职责链中按照插件排序逐一进行处理。

```java
    public Mono<Void> handle(@NonNull final ServerWebExchange exchange) {
        MetricsTrackerFacade.getInstance().counterInc(MetricsLabelEnum.REQUEST_TOTAL.getName());
        Optional<HistogramMetricsTrackerDelegate> startTimer = MetricsTrackerFacade.getInstance().histogramStartTimer(MetricsLabelEnum.REQUEST_LATENCY.getName());
        return new DefaultSoulPluginChain(plugins).execute(exchange).subscribeOn(scheduler)
                .doOnSuccess(t -> startTimer.ifPresent(time -> MetricsTrackerFacade.getInstance().histogramObserveDuration(time)));
    }
```

先是执行插件抽象基类`AbstractSoulPlugin`的execute方法做统一的一些处理。校验插件是否配置、插件是否开启、选择器是否配置、选择器是否开启、选取匹配的选择器、规则是否配置、选取匹配的规则。不过指定一提的是，这里设计得很棒，选择器、规则都是通过SPI的方式进行加载，可直接扩展。一切准备就绪，最核心的地方来了。选取好的选择器数据、规则数据传给子类`RateLimiterPlugin`，执行核心方法doExecute。

```java
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        final String handle = rule.getHandle();
        final RateLimiterHandle limiterHandle = GsonUtils.getInstance().fromJson(handle, RateLimiterHandle.class);
        return redisRateLimiter.isAllowed(rule.getId(), limiterHandle.getReplenishRate(), limiterHandle.getBurstCapacity())
                .flatMap(response -> {
                    if (!response.isAllowed()) {
                        exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
                        Object error = SoulResultWrap.error(SoulResultEnum.TOO_MANY_REQUESTS.getCode(), SoulResultEnum.TOO_MANY_REQUESTS.getMsg(), null);
                        return WebFluxResultUtils.result(exchange, error);
                    }
                    return chain.execute(exchange);
                });
    }
```

还记得上一篇文章说到`rateLimiter`插件的两个信息配置参数：

**rate：是你允许用户每秒执行多少请求，而丢弃任何请求。这是令牌桶的填充速率。**

capacity：是允许用户在一秒钟内执行的最大请求数。这是令牌桶可以保存的令牌数。

这里从json反序列化出来放到了RateLimiterHandle对象中进行使用。我看到，这两个参数直接传给了`RedisRateLimiter`的isAllowed方法。如果调用后返回信息`RateLimiterResponse`的allowed为true，那么就表示获取到了令牌，请求可以通过，继续执行下一个插件，否则中断处理直接响应结果。

那么isAllowed方法做了什么呢？继续往下探究。

```java
    public Mono<RateLimiterResponse> isAllowed(final String id, final double replenishRate, final double burstCapacity) {
        if (!this.initialized.get()) {
            throw new IllegalStateException("RedisRateLimiter is not initialized");
        }
        List<String> keys = getKeys(id);
        List<String> scriptArgs = Arrays.asList(replenishRate + "", burstCapacity + "", Instant.now().getEpochSecond() + "", "1");
        Flux<List<Long>> resultFlux = Singleton.INST.get(ReactiveRedisTemplate.class).execute(this.script, keys, scriptArgs);
        return resultFlux.onErrorResume(throwable -> Flux.just(Arrays.asList(1L, -1L)))
                .reduce(new ArrayList<Long>(), (longs, l) -> {
                    longs.addAll(l);
                    return longs;
                }).map(results -> {
                    boolean allowed = results.get(0) == 1L;
                    Long tokensLeft = results.get(1);
                    RateLimiterResponse rateLimiterResponse = new RateLimiterResponse(allowed, tokensLeft);
                    log.info("RateLimiter response:{}", rateLimiterResponse.toString());
                    return rateLimiterResponse;
                }).doOnError(throwable -> log.error("Error determining if user allowed from redis:{}", throwable.getMessage()));
    }
```

构造一个redis keys数组：

```java
    private static List<String> getKeys(final String id) {
        String prefix = "request_rate_limiter.{" + id;
        String tokenKey = prefix + "}.tokens";
        String timestampKey = prefix + "}.timestamp";
        return Arrays.asList(tokenKey, timestampKey);
    }
```

redis脚本参数数组:

```java
 List<String> scriptArgs = Arrays.asList(replenishRate + "", burstCapacity + "", Instant.now().getEpochSecond() + "", "1");
```

调用初始化的时候加载好的lua脚本：

```java
    private RedisScript<List<Long>> redisScript() {
        DefaultRedisScript redisScript = new DefaultRedisScript<>();
        redisScript.setScriptSource(new ResourceScriptSource(new ClassPathResource("/META-INF/scripts/request_rate_limiter.lua")));
        redisScript.setResultType(List.class);
        return redisScript;
    }
```

最后把脚本的执行结果放入`RateLimiterResponse`返回。

由此看到，`RateLimiter`插件使用Redis实现了令牌桶算法，这就天然支持了分布式。

```lua
local tokens_key = KEYS[1]
local timestamp_key = KEYS[2]
--redis.log(redis.LOG_WARNING, "tokens_key " .. tokens_key)

local rate = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local fill_time = capacity/rate
local ttl = math.floor(fill_time*2)

--redis.log(redis.LOG_WARNING, "rate " .. ARGV[1])
--redis.log(redis.LOG_WARNING, "capacity " .. ARGV[2])
--redis.log(redis.LOG_WARNING, "now " .. ARGV[3])
--redis.log(redis.LOG_WARNING, "requested " .. ARGV[4])
--redis.log(redis.LOG_WARNING, "filltime " .. fill_time)
--redis.log(redis.LOG_WARNING, "ttl " .. ttl)

local last_tokens = tonumber(redis.call("get", tokens_key))
if last_tokens == nil then
  last_tokens = capacity
end
--redis.log(redis.LOG_WARNING, "last_tokens " .. last_tokens)

local last_refreshed = tonumber(redis.call("get", timestamp_key))
if last_refreshed == nil then
  last_refreshed = 0
end
--redis.log(redis.LOG_WARNING, "last_refreshed " .. last_refreshed)

local delta = math.max(0, now-last_refreshed)
local filled_tokens = math.min(capacity, last_tokens+(delta*rate))
local allowed = filled_tokens >= requested
local new_tokens = filled_tokens
local allowed_num = 0
if allowed then
  new_tokens = filled_tokens - requested
  allowed_num = 1
end

--redis.log(redis.LOG_WARNING, "delta " .. delta)
--redis.log(redis.LOG_WARNING, "filled_tokens " .. filled_tokens)
--redis.log(redis.LOG_WARNING, "allowed_num " .. allowed_num)
--redis.log(redis.LOG_WARNING, "new_tokens " .. new_tokens)

redis.call("setex", tokens_key, ttl, new_tokens)
redis.call("setex", timestamp_key, ttl, now)

return { allowed_num, new_tokens }
```

至于这个令牌桶的具体脚本实现逻辑，我们且看下回分解。