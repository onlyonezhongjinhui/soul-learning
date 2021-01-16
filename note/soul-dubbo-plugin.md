#### dubbo插件的作用是什么？

dubbo插件是将`http协议` 转换成`dubbo协议` 的插件,也是网关实现dubbo泛化调用的关键。

#### 什么是泛化调用？

泛化接口调用方式主要用于客户端没有 API 接口及模型类元的情况，参数及返回值中的所有 POJO 均用 Map 表示，通常用于框架集成，比如：实现一个通用的服务测试框架，可通过 GenericService 调用所有服务实现。

#### 为什么使用泛化调用？

一般使用dubbo，provider端需要暴露出接口和方法，consumer端要十分明确服务使用的接口定义和方法定义（还有入参返参类型等等信息，常常还需要基于provider端提供的API），两端才能正常通信调用。然而存在一种使用场景，调用方并不关心要调用的接口的详细定义，它只关注我要调用哪个方法，需要传什么参数，我能接收到什么返回结果即可，这样可以大大降低consumer端和provider端的耦合性。所以为了应对以上的需求，dubbo提供了泛化调用，也就是在consumer只知道一个接口全限定名以及入参和返参的情况下，就可以调用provider端的调用，而不需要传统的接口定义这些繁杂的结构。

#### dubbo插件如何工作？

soul网关把http请求参数转换为泛化参数后根据元数据进行dubbo泛化调用

```
@Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        // 获取请求体数据
        String body = exchange.getAttribute(Constants.DUBBO_PARAMS);
        SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        // 获取dubbo元数据
        MetaData metaData = exchange.getAttribute(Constants.META_DATA);
        if (!checkMetaData(metaData)) {
            assert metaData != null;
            log.error(" path is :{}, meta data have error.... {}", soulContext.getPath(), metaData.toString());
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            Object error = SoulResultWrap.error(SoulResultEnum.META_DATA_ERROR.getCode(), SoulResultEnum.META_DATA_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        // 如果又参数类型而无请求体参数则报错
        if (StringUtils.isNoneBlank(metaData.getParameterTypes()) && StringUtils.isBlank(body)) {
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            Object error = SoulResultWrap.error(SoulResultEnum.DUBBO_HAVE_BODY_PARAM.getCode(), SoulResultEnum.DUBBO_HAVE_BODY_PARAM.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        // dubbo泛化调用
        final Mono<Object> result = dubboProxyService.genericInvoker(body, metaData, exchange);
        return result.then(chain.execute(exchange));
    }
```

```
public Mono<Object> genericInvoker(final String body, final MetaData metaData, final ServerWebExchange exchange) throws SoulException {
        // issue(https://github.com/dromara/soul/issues/471), add dubbo tag route
        // 如果存在tag标签则放入
        String dubboTagRouteFromHttpHeaders = exchange.getRequest().getHeaders().getFirst(Constants.DUBBO_TAG_ROUTE);
        if (StringUtils.isNotBlank(dubboTagRouteFromHttpHeaders)) {
            RpcContext.getContext().setAttachment(CommonConstants.TAG_KEY, dubboTagRouteFromHttpHeaders);
        }
        // 缓存中获取服务引用
        ReferenceConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
        if (Objects.isNull(reference) || StringUtils.isEmpty(reference.getInterface())) {
            ApplicationConfigCache.getInstance().invalidate(metaData.getPath());
            reference = ApplicationConfigCache.getInstance().initRef(metaData);
        }
        GenericService genericService = reference.get();
        // json参数转化为dubbo泛化参数
        Pair<String[], Object[]> pair;
        if (ParamCheckUtils.dubboBodyIsEmpty(body)) {
            pair = new ImmutablePair<>(new String[]{}, new Object[]{});
        } else {
            pair = dubboParamResolveService.buildParameter(body, metaData.getParameterTypes());
        }
        // 真正调用dubbo服务
        CompletableFuture<Object> future = genericService.$invokeAsync(metaData.getMethodName(), pair.getLeft(), pair.getRight());
        return Mono.fromFuture(future.thenApply(ret -> {
            if (Objects.isNull(ret)) {
                ret = Constants.DUBBO_RPC_RESULT_EMPTY;
            }
            exchange.getAttributes().put(Constants.DUBBO_RPC_RESULT, ret);
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
            return ret;
        })).onErrorMap(exception -> exception instanceof GenericException ? new SoulException(((GenericException) exception).getExceptionMessage()) : new SoulException(exception));
    }
```