soul global plugin是做什么的呢？这个官方文档里面没有提及，其实是因为这个插件是个面向开发的这么一个插件。那么它具体有什么作用呢？

```java
public class GlobalPlugin implements SoulPlugin {
  
    private final SoulContextBuilder builder;
  
    /**
     * Instantiates a new Global plugin.
     *
     * @param builder the builder
     */
    public GlobalPlugin(final SoulContextBuilder builder) {
        this.builder = builder;
    }
  
    @Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        final ServerHttpRequest request = exchange.getRequest();
        final HttpHeaders headers = request.getHeaders();
        final String upgrade = headers.getFirst("Upgrade");
        SoulContext soulContext;
        if (StringUtils.isBlank(upgrade) || !"websocket".equals(upgrade)) {
            soulContext = builder.build(exchange);
        } else {
            final MultiValueMap<String, String> queryParams = request.getQueryParams();
            soulContext = transformMap(queryParams);
        }
        exchange.getAttributes().put(Constants.CONTEXT, soulContext);
        return chain.execute(exchange);
    }
  
    @Override
    public int getOrder() {
        return 0;
    }
  
    private SoulContext transformMap(final MultiValueMap<String, String> queryParams) {
        SoulContext soulContext = new SoulContext();
        soulContext.setModule(queryParams.getFirst(Constants.MODULE));
        soulContext.setMethod(queryParams.getFirst(Constants.METHOD));
        soulContext.setRpcType(queryParams.getFirst(Constants.RPC_TYPE));
        return soulContext;
    }
  
    @Override
    public String named() {
        return "global";
    }
}
```

从上面的代码可以看到，其实现了SoulPlugin接口，是个标准的插件，可以直接加到职责链中运行。做的事情就是构造一个上下文对象SoulContext。因为SoulContext的参数很多，所以使用了建造者模式，委托给了一个DefaultSoulContextBuilder去构造。

```java
public class DefaultSoulContextBuilder implements SoulContextBuilder {
  
    @Override
    public SoulContext build(final ServerWebExchange exchange) {
        final ServerHttpRequest request = exchange.getRequest();
        String path = request.getURI().getPath();
        MetaData metaData = MetaDataCache.getInstance().obtain(path);
        if (Objects.nonNull(metaData) && metaData.getEnabled()) {
            exchange.getAttributes().put(Constants.META_DATA, metaData);
        }
        return transform(request, metaData);
    }
  
    /**
     * ServerHttpRequest transform RequestDTO .
     *
     * @param request {@linkplain ServerHttpRequest}
     * @return RequestDTO request dto
     */
    private SoulContext transform(final ServerHttpRequest request, final MetaData metaData) {
        final String appKey = request.getHeaders().getFirst(Constants.APP_KEY);
        final String sign = request.getHeaders().getFirst(Constants.SIGN);
        final String timestamp = request.getHeaders().getFirst(Constants.TIMESTAMP);
        SoulContext soulContext = new SoulContext();
        String path = request.getURI().getPath();
        soulContext.setPath(path);
        if (Objects.nonNull(metaData) && metaData.getEnabled()) {
            if (RpcTypeEnum.SPRING_CLOUD.getName().equals(metaData.getRpcType())) {
                setSoulContextByHttp(soulContext, path);
                soulContext.setRpcType(metaData.getRpcType());
            } else if (RpcTypeEnum.DUBBO.getName().equals(metaData.getRpcType())) {
                setSoulContextByDubbo(soulContext, metaData);
            } else if (RpcTypeEnum.SOFA.getName().equals(metaData.getRpcType())) {
                setSoulContextBySofa(soulContext, metaData);
            } else if (RpcTypeEnum.TARS.getName().equals(metaData.getRpcType())) {
                setSoulContextByTars(soulContext, metaData);
            } else {
                setSoulContextByHttp(soulContext, path);
                soulContext.setRpcType(RpcTypeEnum.HTTP.getName());
            }
        } else {
            setSoulContextByHttp(soulContext, path);
            soulContext.setRpcType(RpcTypeEnum.HTTP.getName());
        }
        soulContext.setAppKey(appKey);
        soulContext.setSign(sign);
        soulContext.setTimestamp(timestamp);
        soulContext.setStartDateTime(LocalDateTime.now());
        Optional.ofNullable(request.getMethod()).ifPresent(httpMethod -> soulContext.setHttpMethod(httpMethod.name()));
        return soulContext;
    }
  
    private void setSoulContextByDubbo(final SoulContext soulContext, final MetaData metaData) {
        soulContext.setModule(metaData.getAppName());
        soulContext.setMethod(metaData.getServiceName());
        soulContext.setRpcType(metaData.getRpcType());
        soulContext.setContextPath(metaData.getContextPath());
    }

    private void setSoulContextBySofa(final SoulContext soulContext, final MetaData metaData) {
        soulContext.setModule(metaData.getAppName());
        soulContext.setMethod(metaData.getServiceName());
        soulContext.setRpcType(metaData.getRpcType());
        soulContext.setContextPath(metaData.getContextPath());
    }

    private void setSoulContextByTars(final SoulContext soulContext, final MetaData metaData) {
        soulContext.setModule(metaData.getServiceName());
        soulContext.setMethod(metaData.getMethodName());
        soulContext.setRpcType(metaData.getRpcType());
        soulContext.setContextPath(metaData.getContextPath());
    }
  
    private void setSoulContextByHttp(final SoulContext soulContext, final String path) {
        String contextPath = "/";
        String[] splitList = StringUtils.split(path, "/");
        if (splitList.length != 0) {
            contextPath = contextPath.concat(splitList[0]);
        }
        String realUrl = path.substring(contextPath.length());
        soulContext.setContextPath(contextPath);
        soulContext.setModule(contextPath);
        soulContext.setMethod(realUrl);
        soulContext.setRealUrl(realUrl);
    }
}
```

Global插件作为第一个运行的插件，会从请求ServerWebExchange对象中取出SoulContext所需要的信息存入SoulContext中，如果有元数据也一应放入。最终放入到ServerWebExchange属性中取名为context，往下透传。这是一种上下文设计模式。spring中各种context都是如此。

## 关于Global插件代码一点思考

1. 这段if-else逻辑可以优化掉

```java
    private SoulContext transform(final ServerHttpRequest request, final MetaData metaData) {
        final String appKey = request.getHeaders().getFirst(Constants.APP_KEY);
        final String sign = request.getHeaders().getFirst(Constants.SIGN);
        final String timestamp = request.getHeaders().getFirst(Constants.TIMESTAMP);
        SoulContext soulContext = new SoulContext();
        String path = request.getURI().getPath();
        soulContext.setPath(path);
        if (Objects.nonNull(metaData) && metaData.getEnabled()) {
            if (RpcTypeEnum.SPRING_CLOUD.getName().equals(metaData.getRpcType())) {
                setSoulContextByHttp(soulContext, path);
                soulContext.setRpcType(metaData.getRpcType());
            } else if (RpcTypeEnum.DUBBO.getName().equals(metaData.getRpcType())) {
                setSoulContextByDubbo(soulContext, metaData);
            } else if (RpcTypeEnum.SOFA.getName().equals(metaData.getRpcType())) {
                setSoulContextBySofa(soulContext, metaData);
            } else if (RpcTypeEnum.TARS.getName().equals(metaData.getRpcType())) {
                setSoulContextByTars(soulContext, metaData);
            } else {
                setSoulContextByHttp(soulContext, path);
                soulContext.setRpcType(RpcTypeEnum.HTTP.getName());
            }
        } else {
            setSoulContextByHttp(soulContext, path);
            soulContext.setRpcType(RpcTypeEnum.HTTP.getName());
        }
        soulContext.setAppKey(appKey);
        soulContext.setSign(sign);
        soulContext.setTimestamp(timestamp);
        soulContext.setStartDateTime(LocalDateTime.now());
        Optional.ofNullable(request.getMethod()).ifPresent(httpMethod -> soulContext.setHttpMethod(httpMethod.name()));
        return soulContext;
    }
```

2. SoulContext构造有点分散，是否都放入建造者中处理会集中一些