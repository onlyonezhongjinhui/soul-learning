#### soul http client 注册流程

spring mvc应用端配置soul注册信息

```yaml
soul:
  http:
    adminUrl: http://localhost:9095
    port: 8188
    contextPath: /http
    appName: http
    full: false
```

soul-spring-boot-starter-client-springmvc启动的时候注册SpringMvcClientBeanPostProcessor后置处理器

```java
@Configuration
public class SoulSpringMvcClientConfiguration {
  
    /**
     * 注册SpringMvcClientBeanPostProcessor后置处理器
     * Spring http client bean post processor spring http client bean post processor.
     *
     * @param soulSpringMvcConfig the soul http config
     * @return the spring http client bean post processor
     */
    @Bean
    public SpringMvcClientBeanPostProcessor springHttpClientBeanPostProcessor(final SoulSpringMvcConfig soulSpringMvcConfig) {
        return new SpringMvcClientBeanPostProcessor(soulSpringMvcConfig);
    }
  
    /**
     * 读取应用端配置
     * Soul http config soul http config.
     *
     * @return the soul http config
     */
    @Bean
    @ConfigurationProperties(prefix = "soul.http")
    public SoulSpringMvcConfig soulHttpConfig() {
        return new SoulSpringMvcConfig();
    }
}
```

SpringMvcClientBeanPostProcessor后置处理器进行Bean扫描并提交注册任务到线程池

```java
public class SpringMvcClientBeanPostProcessor implements BeanPostProcessor {

    private final ThreadPoolExecutor executorService;

    private final String url;

    private final SoulSpringMvcConfig soulSpringMvcConfig;

    /**
     * 1、校验配置是否合法
     * 2、实例化一个线程池,后续用来多线程注册代理接口
     * Instantiates a new Soul client bean post processor.
     *
     * @param soulSpringMvcConfig the soul spring mvc config
     */
    public SpringMvcClientBeanPostProcessor(final SoulSpringMvcConfig soulSpringMvcConfig) {
        ValidateUtils.validate(soulSpringMvcConfig);
        this.soulSpringMvcConfig = soulSpringMvcConfig;
        url = soulSpringMvcConfig.getAdminUrl() + "/soul-client/springmvc-register";
        executorService = new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>());
    }

    /**
     * 1、扫描配置
     * 2、注册代理接口
     */
    @Override
    public Object postProcessAfterInitialization(@NonNull final Object bean, @NonNull final String beanName) throws BeansException {
        // isFull = true 表示代理整个应用，isFull = false 表示只代理一部分
        if (soulSpringMvcConfig.isFull()) {
            return bean;
        }
        // 扫描Controller注解
        Controller controller = AnnotationUtils.findAnnotation(bean.getClass(), Controller.class);
        // 扫描RestController注解
        RestController restController = AnnotationUtils.findAnnotation(bean.getClass(), RestController.class);
        // 扫描RequestMapping注解
        RequestMapping requestMapping = AnnotationUtils.findAnnotation(bean.getClass(), RequestMapping.class);
        if (controller != null || restController != null || requestMapping != null) {
            // 扫描xxController类上SoulSpringMvcClient注解
            SoulSpringMvcClient clazzAnnotation = AnnotationUtils.findAnnotation(bean.getClass(), SoulSpringMvcClient.class);
            String prePath = "";
            if (Objects.nonNull(clazzAnnotation)) {
                if (clazzAnnotation.path().indexOf("*") > 1) {
                    String finalPrePath = prePath;
                    // 提交注册任务到线程池,由线程池请求soul-admin进行注册
                    executorService.execute(() -> RegisterUtils.doRegister(buildJsonParams(clazzAnnotation, finalPrePath), url,
                            RpcTypeEnum.HTTP));
                    return bean;
                }
                prePath = clazzAnnotation.path();
            }
  
            final Method[] methods = ReflectionUtils.getUniqueDeclaredMethods(bean.getClass());
            for (Method method : methods) {
                // 扫描xxController方法上的SoulSpringMvcClient注解
                SoulSpringMvcClient soulSpringMvcClient = AnnotationUtils.findAnnotation(method, SoulSpringMvcClient.class);
                if (Objects.nonNull(soulSpringMvcClient)) {
                    String finalPrePath = prePath;
                    // 提交注册任务到线程池,由线程池请求soul-admin进行注册
                    executorService.execute(() -> RegisterUtils.doRegister(buildJsonParams(soulSpringMvcClient, finalPrePath), url,
                            RpcTypeEnum.HTTP));
                }
            }
        }
        return bean;
    }
}
```

最终委托给了RegisterUtils工具类通过OkHttpTools发送http请求注册到了soul-admin

```java
public final class RegisterUtils {

    private RegisterUtils() {
    }

    /**
     * call register api.
     *
     * @param json        request body
     * @param url         url
     * @param rpcTypeEnum rcp type
     */
    public static void doRegister(final String json, final String url, final RpcTypeEnum rpcTypeEnum) {
        try {
            String result = OkHttpTools.getInstance().post(url, json);
            if (AdminConstants.SUCCESS.equals(result)) {
                log.info("{} client register success: {} ", rpcTypeEnum.getName(), json);
            } else {
                log.error("{} client register error: {} ", rpcTypeEnum.getName(), json);
            }
        } catch (IOException e) {
            log.error("cannot register soul admin param, url: {}, request body: {}", url, json, e);
        }
    }

}
```

到此，http方式注册就算完成。