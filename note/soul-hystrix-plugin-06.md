前面几篇都聊了Hystrix的基本原理，今天回归主题，看看soul是如何使用Hystrix的。

首先使用Hystrix就要定义Command，嘿嘿，一看，果然soul里面定义了Command。

```java
public interface Command {
    /**
     * wrap fetch Observable in {@link HystrixCommand} and {@link HystrixCommandOnThread}.
     *
     * @return {@code Observable<R>} that executes and calls back with the result of command execution
     *         or a fallback if the command fails for any reason.
     */
    Observable<Void> fetchObservable();

    /**
     * whether the 'circuit-breaker' is open.
     *
     * @return boolean
     */
    boolean isCircuitBreakerOpen();

    /**
     * generate a error when some error occurs.
     *
     * @param exchange  the exchange
     * @param exception exception instance
     * @return error which be wrapped by {@link SoulResultWrap}
     */
    default Object generateError(ServerWebExchange exchange, Throwable exception) {
        Object error;
        if (exception instanceof HystrixRuntimeException) {
            HystrixRuntimeException e = (HystrixRuntimeException) exception;
            if (e.getFailureType() == HystrixRuntimeException.FailureType.TIMEOUT) {
                exchange.getResponse().setStatusCode(HttpStatus.GATEWAY_TIMEOUT);
                error = SoulResultWrap.error(SoulResultEnum.SERVICE_TIMEOUT.getCode(), SoulResultEnum.SERVICE_TIMEOUT.getMsg(), null);
            } else {
                exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
                error = SoulResultWrap.error(SoulResultEnum.SERVICE_RESULT_ERROR.getCode(), SoulResultEnum.SERVICE_RESULT_ERROR.getMsg(), null);
            }
        } else if (exception instanceof HystrixTimeoutException) {
            exchange.getResponse().setStatusCode(HttpStatus.GATEWAY_TIMEOUT);
            error = SoulResultWrap.error(SoulResultEnum.SERVICE_TIMEOUT.getCode(), SoulResultEnum.SERVICE_TIMEOUT.getMsg(), null);
        } else {
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            error = SoulResultWrap.error(SoulResultEnum.SERVICE_RESULT_ERROR.getCode(), SoulResultEnum.SERVICE_RESULT_ERROR.getMsg(), null);
        }
        return error;
    }

    /**
     * do fall back when some error occurs on hystrix execute.
     * @param exchange {@link ServerWebExchange}
     * @param exception {@link Throwable}
     * @return {@code Mono<Void>} to indicate when request processing is complete.
     */
    default Mono<Void> doFallback(ServerWebExchange exchange, Throwable exception) {
        if (Objects.isNull(getCallBackUri())) {
            Object error;
            error = generateError(exchange, exception);
            return WebFluxResultUtils.result(exchange, error);
        }
        DispatcherHandler dispatcherHandler =
            SpringBeanUtils.getInstance().getBean(DispatcherHandler.class);
        ServerHttpRequest request = exchange.getRequest().mutate().uri(getCallBackUri()).build();
        ServerWebExchange mutated = exchange.mutate().request(request).build();
        return dispatcherHandler.handle(mutated);
    }

    /**
     * get call back uri.
     * @return when some error occurs in hystrix invoke it will forward to this
     */
    URI getCallBackUri();

}
```

soul定义了一个Command接口，定义了fetchObservable()、isCircuitBreakerOpen()、getCallBackUri()三个接口，generateError()、doFallback()两个默认方法。fetchObservable()用来包装command的执行。isCircuitBreakerOpen()用来判断熔断器是否打开。getCallBackUri()用来获取快速失败回退降级的url。generateError()判断了失败的类型返回一个响应。doFallback()方法则是调用远程降级回退接口获取响应。

HystrixCommand、HystrixCommandOnThread两个Command都实现了Command接口，并分别继承了Hystrix的HystrixObservableCommand、HystrixCommand。

```java
public class HystrixCommandOnThread extends HystrixCommand<Mono<Void>> implements Command {
  
    private final ServerWebExchange exchange;

    private final SoulPluginChain chain;

    private final URI callBackUri;
  
    /**
     * Instantiates a new Http command.
     *
     * @param setter      the setter
     * @param exchange    the exchange
     * @param chain       the chain
     * @param callBackUri the call back uri
     */
    public HystrixCommandOnThread(final HystrixCommand.Setter setter,
                          final ServerWebExchange exchange,
                          final SoulPluginChain chain,
                          final String callBackUri) {
        super(setter);
        this.exchange = exchange;
        this.chain = chain;
        this.callBackUri = UriUtils.createUri(callBackUri);
    }

    @Override
    protected Mono<Void> run() {
        RxReactiveStreams.toObservable(chain.execute(exchange)).toBlocking().subscribe();
        return Mono.empty();
    }

    @Override
    protected Mono<Void> getFallback() {
        if (isFailedExecution()) {
            log.error("hystrix execute have error: ", getExecutionException());
        }
        final Throwable exception = getExecutionException();
        return doFallback(exchange, exception);
    }

    @Override
    public Observable<Void> fetchObservable() {
        return RxReactiveStreams.toObservable(this.execute());
    }

    @Override
    public URI getCallBackUri() {
        return callBackUri;
    }
}
```

HystrixCommand、HystrixCommandOnThread的区别就是HystrixObservableCommand、HystrixCommand两者的区别。HystrixCommandOnThread类Hystrix会从线程池中取一个线程以非阻塞方式执行run()，调用线程不必等待run()；HystrixCommand类将以调用线程堵塞执行construct()，调用线程需等待construct()执行完才能继续往下走。

找到了Command命令，那么接下来应该看看是如何配置Command的。

```java
public class HystrixBuilder {

    /**
     * this is build HystrixObservableCommand.Setter.
     *
     * @param hystrixHandle {@linkplain HystrixHandle}
     * @return {@linkplain HystrixObservableCommand.Setter}
     */
    public static HystrixObservableCommand.Setter build(final HystrixHandle hystrixHandle) {
        initHystrixHandleOnRequire(hystrixHandle);
        HystrixCommandGroupKey groupKey = HystrixCommandGroupKey.Factory.asKey(hystrixHandle.getGroupKey());
        HystrixCommandKey commandKey = HystrixCommandKey.Factory.asKey(hystrixHandle.getCommandKey());
        HystrixCommandProperties.Setter propertiesSetter =
                HystrixCommandProperties.Setter()
                        .withExecutionTimeoutInMilliseconds((int) hystrixHandle.getTimeout())
                        .withCircuitBreakerEnabled(true)
                        .withExecutionIsolationStrategy(HystrixCommandProperties.ExecutionIsolationStrategy.SEMAPHORE)
                        .withExecutionIsolationSemaphoreMaxConcurrentRequests(hystrixHandle.getMaxConcurrentRequests())
                        .withCircuitBreakerErrorThresholdPercentage(hystrixHandle.getErrorThresholdPercentage())
                        .withCircuitBreakerRequestVolumeThreshold(hystrixHandle.getRequestVolumeThreshold())
                        .withCircuitBreakerSleepWindowInMilliseconds(hystrixHandle.getSleepWindowInMilliseconds());
        return HystrixObservableCommand.Setter
                .withGroupKey(groupKey)
                .andCommandKey(commandKey)
                .andCommandPropertiesDefaults(propertiesSetter);
    }

    /**
     * this is build HystrixCommand.Setter.
     * @param hystrixHandle {@linkplain HystrixHandle}
     * @return {@linkplain HystrixCommand.Setter}
     */
    public static HystrixCommand.Setter buildForHystrixCommand(final HystrixHandle hystrixHandle) {
        initHystrixHandleOnRequire(hystrixHandle);
        HystrixCommandGroupKey groupKey = HystrixCommandGroupKey.Factory.asKey(hystrixHandle.getGroupKey());
        HystrixCommandKey commandKey = HystrixCommandKey.Factory.asKey(hystrixHandle.getCommandKey());
        HystrixCommandProperties.Setter propertiesSetter =
                HystrixCommandProperties.Setter()
                        .withExecutionTimeoutInMilliseconds((int) hystrixHandle.getTimeout())
                        .withCircuitBreakerEnabled(true)
                        .withCircuitBreakerErrorThresholdPercentage(hystrixHandle.getErrorThresholdPercentage())
                        .withCircuitBreakerRequestVolumeThreshold(hystrixHandle.getRequestVolumeThreshold())
                        .withCircuitBreakerSleepWindowInMilliseconds(hystrixHandle.getSleepWindowInMilliseconds());
        HystrixThreadPoolConfig hystrixThreadPoolConfig = hystrixHandle.getHystrixThreadPoolConfig();
        HystrixThreadPoolProperties.Setter threadPoolPropertiesSetter =
                HystrixThreadPoolProperties.Setter()
                        .withCoreSize(hystrixThreadPoolConfig.getCoreSize())
                        .withMaximumSize(hystrixThreadPoolConfig.getMaximumSize())
                        .withMaxQueueSize(hystrixThreadPoolConfig.getMaxQueueSize())
                        .withKeepAliveTimeMinutes(hystrixThreadPoolConfig.getKeepAliveTimeMinutes())
                        .withAllowMaximumSizeToDivergeFromCoreSize(true);
        return HystrixCommand.Setter
                .withGroupKey(groupKey)
                .andCommandKey(commandKey)
                .andCommandPropertiesDefaults(propertiesSetter)
                .andThreadPoolPropertiesDefaults(threadPoolPropertiesSetter);
    }

    private static void initHystrixHandleOnRequire(final HystrixHandle hystrixHandle) {
        if (hystrixHandle.getMaxConcurrentRequests() == 0) {
            hystrixHandle.setMaxConcurrentRequests(Constants.MAX_CONCURRENT_REQUESTS);
        }
        if (hystrixHandle.getErrorThresholdPercentage() == 0) {
            hystrixHandle.setErrorThresholdPercentage(Constants.ERROR_THRESHOLD_PERCENTAGE);
        }
        if (hystrixHandle.getRequestVolumeThreshold() == 0) {
            hystrixHandle.setRequestVolumeThreshold(Constants.REQUEST_VOLUME_THRESHOLD);
        }
        if (hystrixHandle.getSleepWindowInMilliseconds() == 0) {
            hystrixHandle.setSleepWindowInMilliseconds(Constants.SLEEP_WINDOW_INMILLISECONDS);
        }
        if (Objects.isNull(hystrixHandle.getHystrixThreadPoolConfig())) {
            hystrixHandle.setHystrixThreadPoolConfig(new HystrixThreadPoolConfig());
        }
        HystrixThreadPoolConfig hystrixThreadPoolConfig = hystrixHandle.getHystrixThreadPoolConfig();
        if (hystrixThreadPoolConfig.getCoreSize() == 0) {
            hystrixThreadPoolConfig.setCoreSize(Constants.HYSTRIX_THREAD_POOL_CORE_SIZE);
        }
        if (hystrixThreadPoolConfig.getMaximumSize() == 0) {
            hystrixThreadPoolConfig.setMaximumSize(Constants.HYSTRIX_THREAD_POOL_MAX_SIZE);
        }
        if (hystrixThreadPoolConfig.getMaxQueueSize() == 0) {
            hystrixThreadPoolConfig.setMaxQueueSize(Constants.HYSTRIX_THREAD_POOL_QUEUE_SIZE);
        }
    }
}
```

因为Command的配置很多，所以此处使用了建造者模式通过建造者进行配置。好了，前面学到的两种资源隔离模式来了。两个build方法分别构建了线程隔离模式、信号量隔离模式。而其它的配置全部来自admin同步过来的数据配置。

万事俱备只欠东风，Hystrix的东西都齐活了，那就等上帝之手把它串联运用起来了。HystrixPlugin就是上帝之手。所有的逻辑都在doExecute中进行串联。

```java
public class HystrixPlugin extends AbstractSoulPlugin {

    @Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        final HystrixHandle hystrixHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), HystrixHandle.class);
        if (StringUtils.isBlank(hystrixHandle.getGroupKey())) {
            hystrixHandle.setGroupKey(Objects.requireNonNull(soulContext).getModule());
        }
        if (StringUtils.isBlank(hystrixHandle.getCommandKey())) {
            hystrixHandle.setCommandKey(Objects.requireNonNull(soulContext).getMethod());
        }
        Command command = fetchCommand(hystrixHandle, exchange, chain);
        return Mono.create(s -> {
            Subscription sub = command.fetchObservable().subscribe(s::success,
                    s::error, s::success);
            s.onCancel(sub::unsubscribe);
            if (command.isCircuitBreakerOpen()) {
                log.error("hystrix execute have circuitBreaker is Open! groupKey:{},commandKey:{}", hystrixHandle.getGroupKey(), hystrixHandle.getCommandKey());
            }
        }).doOnError(throwable -> {
            log.error("hystrix execute exception:", throwable);
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.ERROR.getName());
            chain.execute(exchange);
        }).then();
    }

    private Command fetchCommand(final HystrixHandle hystrixHandle, final ServerWebExchange exchange, final SoulPluginChain chain) {
        if (hystrixHandle.getExecutionIsolationStrategy() == HystrixIsolationModeEnum.SEMAPHORE.getCode()) {
            return new HystrixCommand(HystrixBuilder.build(hystrixHandle),
                exchange, chain, hystrixHandle.getCallBackUri());
        }
        return new HystrixCommandOnThread(HystrixBuilder.buildForHystrixCommand(hystrixHandle),
            exchange, chain, hystrixHandle.getCallBackUri());
    }

    @Override
    public String named() {
        return PluginEnum.HYSTRIX.getName();
    }

    @Override
    public int getOrder() {
        return PluginEnum.HYSTRIX.getCode();
    }

}
```

doExcuete中根据配置创建一个基于线程隔离还是信号量隔离的Command，执行command的fetchObservable()方法。这样就把Hystrix使用起来了。

## 问题

```java
default Mono<Void> doFallback(ServerWebExchange exchange, Throwable exception) {
        if (Objects.isNull(getCallBackUri())) {
            Object error;
            error = generateError(exchange, exception);
            return WebFluxResultUtils.result(exchange, error);
        }
        DispatcherHandler dispatcherHandler =
            SpringBeanUtils.getInstance().getBean(DispatcherHandler.class);
        ServerHttpRequest request = exchange.getRequest().mutate().uri(getCallBackUri()).build();
        ServerWebExchange mutated = exchange.mutate().request(request).build();
        return dispatcherHandler.handle(mutated);
    }
```

doFallback是否应该包装在Command中执行，防止降级服务也跨了引起问题。