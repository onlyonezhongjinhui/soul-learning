上一篇了解了Hystrix的熔断机制，今天我们再看看回退和降级

## 回退降级

降级，通常指务高峰期，为了保证核心服务正常运行，需要停掉一些不太重要的业务，或者某些服务不可用时，执行备用逻辑从故障服务中快速失败或快速返回，以保障主体业务不受影响。Hystrix提供的降级主要是为了容错，保证当前服务不受依赖服务故障的影响，从而提高服务的健壮性。要支持回退或降级处理，可以重写HystrixCommand的getFallBack方法或HystrixObservableCommand的resumeWithFallback方法。

## Hystrix何时会触发降级

* 执行construct()或run()抛出异常
* 熔断器打开导致命令短路
* 命令的线程池和队列或信号量的容量超额，命令被拒绝
* 命令执行超时

## 降级回退方式

### Fail Fast 快速失败

快速失败是最普通的命令执行方法，命令没有重写降级逻辑。 如果命令执行发生任何类型的故障，它将直接抛出异常。

### Fail Silent 无声失败

指在降级方法中通过返回null，空Map，空List或其他类似的响应来完成

### Fallback: Static

指在降级方法中返回静态默认值。 这不会导致服务以“无声失败”的方式被删除，而是导致默认行为发生。如：应用根据命令执行返回true / false执行相应逻辑，但命令执行失败，则默认为true

### Fallback: Stubbed

当命令返回一个包含多个字段的复合对象时，适合以Stubbed 的方式回退

### Fallback: Cache via Network

有时，如果调用依赖服务失败，可以从缓存服务（如redis）中查询旧数据版本。由于又会发起远程调用，所以建议重新封装一个Command，使用不同的ThreadPoolKey，与主线程池进行隔离。

### Primary + Secondary with Fallback

有时系统具有两种行为- 主要和次要，或主要和故障转移。主要和次要逻辑涉及到不同的网络调用和业务逻辑，所以需要将主次逻辑封装在不同的Command中，使用线程池进行隔离。为了实现主从逻辑切换，可以将主次command封装在外观HystrixCommand的run方法中，并结合配置中心设置的开关切换主从逻辑。由于主次逻辑都是经过线程池隔离的HystrixCommand，因此外观HystrixCommand可以使用信号量隔离，而没有必要使用线程池隔离引入不必要的开销

主次模型的使用场景还是很多的。如当系统升级新功能时，如果新版本的功能出现问题，通过开关控制降级调用旧版本的功能