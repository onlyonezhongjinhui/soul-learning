## soul nacos方式同步数据原理

soul nacos方式同步数据原理和zookeeper基本一致，都是利用中间件自身的监听机制。`soul-admin` 在启动的时候，会将数据全量写入 `nacos`，后续数据发生变更时，会增量更新 到`nacos` 。而启动的时候会从nacos中读取全量数据保存到本地并开启监听nacos中数据的变化。一旦`soul-admin`中数据有变化并更新到`nacos`中，`soul-web`即可收到`nacos`通知然后把变化的数据刷新到本地缓存。

## 详解

NacosSyncDataService类实现了数据同步统一接口SyncDataService，作用就是在构造方法中调用start方法从`nacos`中读取全量数据并增添监听器监听数据变化并通知订阅者缓存和刷新数据。

```java
public class NacosSyncDataService extends NacosCacheHandler implements AutoCloseable, SyncDataService {

    /**
     * Instantiates a new Nacos sync data service.
     *
     * @param configService         the config service
     * @param pluginDataSubscriber the plugin data subscriber
     * @param metaDataSubscribers   the meta data subscribers
     * @param authDataSubscribers   the auth data subscribers
     */
    public NacosSyncDataService(final ConfigService configService, final PluginDataSubscriber pluginDataSubscriber,
                                final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {

        super(configService, pluginDataSubscriber, metaDataSubscribers, authDataSubscribers);
        start();
    }

    /**
     * Start.
     */
    public void start() {
        watcherData(PLUGIN_DATA_ID, this::updatePluginMap);
        watcherData(SELECTOR_DATA_ID, this::updateSelectorMap);
        watcherData(RULE_DATA_ID, this::updateRuleMap);
        watcherData(META_DATA_ID, this::updateMetaDataMap);
        watcherData(AUTH_DATA_ID, this::updateAuthMap);
    }

    @Override
    public void close() {
        LISTENERS.forEach((dataId, lss) -> {
            lss.forEach(listener -> getConfigService().removeListener(dataId, GROUP, listener));
            lss.clear();
        });
        LISTENERS.clear();
    }
}

```

而实际操作都在其父类NacosCacheHandler中完成。watcherData负责添加监听器。并注册收到通知时候执行OnChange接口的change方法。具体就是在change方法中调用updatePluginMap、updateSelectorMap、updateRuleMap、updateMetaDataMap、updateAuthMap。具体操作就是通知订阅者先删除本地数据，再缓存新数据。

```java
public class NacosCacheHandler {

    protected static final String GROUP = "DEFAULT_GROUP";

    protected static final String PLUGIN_DATA_ID = "soul.plugin.json";

    protected static final String SELECTOR_DATA_ID = "soul.selector.json";

    protected static final String RULE_DATA_ID = "soul.rule.json";

    protected static final String AUTH_DATA_ID = "soul.auth.json";

    protected static final String META_DATA_ID = "soul.meta.json";

    protected static final Map<String, List<Listener>> LISTENERS = Maps.newConcurrentMap();

    @Getter
    private final ConfigService configService;

    private final PluginDataSubscriber pluginDataSubscriber;

    private final List<MetaDataSubscriber> metaDataSubscribers;

    private final List<AuthDataSubscriber> authDataSubscribers;

    public NacosCacheHandler(final ConfigService configService, final PluginDataSubscriber pluginDataSubscriber,
                             final List<MetaDataSubscriber> metaDataSubscribers,
                             final List<AuthDataSubscriber> authDataSubscribers) {
        this.configService = configService;
        this.pluginDataSubscriber = pluginDataSubscriber;
        this.metaDataSubscribers = metaDataSubscribers;
        this.authDataSubscribers = authDataSubscribers;
    }

    protected void updatePluginMap(final String configInfo) {
        try {
            // Fix bug #656(https://github.com/dromara/soul/issues/656)
            List<PluginData> pluginDataList = new ArrayList<>(GsonUtils.getInstance().toObjectMap(configInfo, PluginData.class).values());
            pluginDataList.forEach(pluginData -> Optional.ofNullable(pluginDataSubscriber).ifPresent(subscriber -> {
                subscriber.unSubscribe(pluginData);
                subscriber.onSubscribe(pluginData);
            }));
        } catch (JsonParseException e) {
            log.error("sync plugin data have error:", e);
        }
    }

    protected void updateSelectorMap(final String configInfo) {
        try {
            List<SelectorData> selectorDataList = GsonUtils.getInstance().toObjectMapList(configInfo, SelectorData.class).values().stream().flatMap(Collection::stream).collect(Collectors.toList());
            selectorDataList.forEach(selectorData -> Optional.ofNullable(pluginDataSubscriber).ifPresent(subscriber -> {
                subscriber.unSelectorSubscribe(selectorData);
                subscriber.onSelectorSubscribe(selectorData);
            }));
        } catch (JsonParseException e) {
            log.error("sync selector data have error:", e);
        }
    }

    protected void updateRuleMap(final String configInfo) {
        try {
            List<RuleData> ruleDataList = GsonUtils.getInstance().toObjectMapList(configInfo, RuleData.class).values()
                    .stream().flatMap(Collection::stream)
                    .collect(Collectors.toList());
            ruleDataList.forEach(ruleData -> Optional.ofNullable(pluginDataSubscriber).ifPresent(subscriber -> {
                subscriber.unRuleSubscribe(ruleData);
                subscriber.onRuleSubscribe(ruleData);
            }));
        } catch (JsonParseException e) {
            log.error("sync rule data have error:", e);
        }
    }

    protected void updateMetaDataMap(final String configInfo) {
        try {
            List<MetaData> metaDataList = new ArrayList<>(GsonUtils.getInstance().toObjectMap(configInfo, MetaData.class).values());
            metaDataList.forEach(metaData -> metaDataSubscribers.forEach(subscriber -> {
                subscriber.unSubscribe(metaData);
                subscriber.onSubscribe(metaData);
            }));
        } catch (JsonParseException e) {
            log.error("sync meta data have error:", e);
        }
    }

    protected void updateAuthMap(final String configInfo) {
        try {
            List<AppAuthData> appAuthDataList = new ArrayList<>(GsonUtils.getInstance().toObjectMap(configInfo, AppAuthData.class).values());
            appAuthDataList.forEach(appAuthData -> authDataSubscribers.forEach(subscriber -> {
                subscriber.unSubscribe(appAuthData);
                subscriber.onSubscribe(appAuthData);
            }));
        } catch (JsonParseException e) {
            log.error("sync auth data have error:", e);
        }
    }

    @SneakyThrows
    private String getConfigAndSignListener(final String dataId, final Listener listener) {
        return configService.getConfigAndSignListener(dataId, GROUP, 6000, listener);
    }

    protected void watcherData(final String dataId, final OnChange oc) {
        Listener listener = new Listener() {
            @Override
            public void receiveConfigInfo(final String configInfo) {
                oc.change(configInfo);
            }

            @Override
            public Executor getExecutor() {
                return null;
            }
        };
        oc.change(getConfigAndSignListener(dataId, listener));
        LISTENERS.getOrDefault(dataId, new ArrayList<>()).add(listener);
    }

    protected interface OnChange {

        void change(String changeData);
    }
}
```

至于对`nacos`的操作全是通过ConfigService进行。这是`nacos`的api。`nacos`这个监听机制是通过http长轮询的方式，原理和`soul`的http长轮询方式一模一样。