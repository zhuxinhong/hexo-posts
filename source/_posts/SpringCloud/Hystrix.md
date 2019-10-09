---
title: Hystrix之hystrix-javanica
date: 2017-07-16
categories: SpringCloud 
tags: 
- Hystrix
- SpringCloud
---

##	什么是 Hystrix？

在分布式环境中，不可避免地会有一些服务依赖项会失败。Hystrix是一个库，它通过添加延迟容忍和容错逻辑来帮助控制这些分布式服务之间的交互。Hystrix通过隔离服务之间的访问点、阻止它们之间的级联故障，并提供备用选项，从而提高系统的整体弹性。

##	Hystrix 解决什么问题？

对于每个具有99.99％正常运行时间的30个服务。

>99.99^30 = 99.7％正常运行时间

>10亿次请求中的0.3％= 3,000,000次故障

>2+小时停机时间/月，即使所有依赖关系都有很好的正常运行时间。

即使所有依赖关系表现良好，即使0.01％的停机时间对数十个服务中的每一个服务的总体影响等同于每个月停机的潜在时间。

对于高容量流量，单个后端依赖关系故障可能导致所有服务器上资源在几秒钟内饱和。

##	Hystrix 工作原理

*	防止任何单个依赖关系使用容器（如Tomcat）全部线程。
*	减少负载并快速失败，而不是排队。
*	在可行的情况下提供回退以保护用户免受故障。
*	使用隔离技术（如隔板，泳道和断路器模式）来限制任何一个依赖的影响。
*	提供准实时的监控指标、报警。
*	提供低延迟的配置修改，并支持Hystrix大多数的动态属性更改，以便循环使用低延迟修改进行实时操作修改。
*	保护整个依赖客户端执行中的故障，而不仅仅是在网络流量中。

##	Hystrix 功能实现

*	通过HystrixCommand或HystrixObservableCommand包装请求，
实现在单独的线程中执行（这是命令模式的一种）。

*	超时调用比您定义的阈值长时间更长。提供一个默认配置，对于大多数依赖调用你可以通过”properties”来自定义它们的超时时间，以使它们的可用性达到99.5%或更高。

*	为每个依赖调用维护一个小的线程池（或信号量）; 如果它满了，那么依赖调用的请求将立即被拒绝，而不是排队等待。

*	处理成功，失败（由客户端抛出的异常），超时和线程拒绝。

*	跳闸断路器可以在一段时间内停止对特定服务的所有请求，如果服务的错误百分比通过阈值，可手动或自动停止。

*	当请求失败时执行回退逻辑，被拒绝，超时或短路。

*	准实时监控，运行时变更配置。

##	hystrix-javanica

使用完整的hystrix需要大量代码，可能需要花费大量时间写一个Hystrix命令。Javanica旨在通过引入注释来简化使用hystrix。Javanica通过在Spring中声明HystrixCommandAspect这个bean来实现，这里不再赘述AOP。

Javanica 通过 ConfigurationManager 来管理属性配置。

以下注解和代码效果等同。

```Java
    @HystrixCommand(commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "500")
        })
    public User getUserById(String id) {
        return userResource.getUserById(id);
    }
```

```Java
ConfigurationManager.getConfigInstance().setProperty("hystrix.command.getUserById.execution.isolation.thread.timeoutInMilliseconds", "500");
```


###	HystrixCommand 注解配置

####	隔离策略

*	ExecutionIsolationStrategy.THREAD

	在一个单独的线程上执行，并发请求受到线程池中线程数量的限制。推荐使用。
	
* 	ExecutionIsolationStrategy.SEMAPHORE

	它在调用线程上执行，并发请求受到信号量计数的限制。开销太大，适用于非网络调用。


####	HystrixCommand

参数|作用|默认值
:-:|:-:|:-:
groupKey|所属分组，一个group共用一个线程池|当前类名
commandKey|业务key|当前方法名
execution.isolation.strategy|隔离策略|ExecutionIsolationStrategy.THREAD
execution.isolation.thread.timeoutInMilliseconds|超时时间|1000ms
execution.timeout.enable|超时开关|true
execution.isolation.thread.interruptOnTimeout|超时线程中断|true，Thread模式有效
execution.isolation.thread.interruptOnCancel|取消线程中断|false，Thread模式有效
execution.isolation.semaphore.maxConcurrentRequests|信号量最大并发度|10，SEMAPHORE模式有效

####	Fallback

参数|作用|默认值
:-:|:-:|:-:
fallback.isolation.semaphore.maxConcurrentRequests|fallback最大并发度|10
fallback.enabled|fallback开关|true

####	Circuit Breaker

参数|作用|默认值
:-:|:-:|:-:
circuitBreaker.enabled|熔断器开关|true
circuitBreaker.requestVolumeThreshold|触发熔断的最小个数/10s|20
circuitBreaker.sleepWindowInMilliseconds|熔断多长时间后去尝试请求|5000ms
circuitBreaker.errorThresholdPercentage|触发熔断的失败百分比|50
circuitBreaker.forceOpen|强制打开|false，设为true会拒绝所有请求
circuitBreaker.forceClosed|强制关闭|false，设为true会允许所有请求通过

####	ThreadPool

参数|作用|默认值|实时修改
:-:|:-:|:-:|:-:
coreSize|线程池大小|10|支持
maximumSize|队列大小|10|支持
maxQueueSize|BlockingQueue的最大长度|-1使用SynchronousQueue，否则使用LinkedBlockingQueue|初始化时确定，不允许运行时修改
queueSizeRejectionThreshold|队列大小拒绝阈值|5|支持，即使maxQueueSize没有达到，也会出现拒绝，因为允许动态更改拒绝队列的大小
keepAliveTimeMinutes|线程释放的空闲时间|1|支持
allowMaximumSizeToDivergeFromCoreSize|maximumSize大于coreSize时，线程空闲时是否释放资源|false|支持

###	SpringMVC

```xml
	<aop:aspectj-autoproxy/>
    <bean id="hystrixAspect" class="com.netflix.hystrix.contrib.javanica.aop.aspectj.HystrixCommandAspect"></bean>
```

###	SpringBoot

```Java
	@Configuration
	public class HystrixConfiguration {
	
	  @Bean
	  public HystrixCommandAspect hystrixAspect() {
	    return new HystrixCommandAspect();
	  }
	
	}
```

###	Sync Execution

```Java
	@HystrixCommand(groupKey="UserGroup", commandKey = "GetUserByIdCommand")
	public User getUserById(String id) {
	    return userResource.getUserById(id);
	}
```

###	Async Execution

```Java
	@HystrixCommand
	public Future<User> getUserByIdAsync(final String id) {
	    return new AsyncResult<User>() {
	        @Override
	        public User invoke() {
	            return userResource.getUserById(id);
	        }
	    };
	}
```

###	Reactive Exection

```Java
	@HystrixCommand
	public Observable<User> getUserById(final String id) {
	    return Observable.create(new Observable.OnSubscribe<User>() {
	            @Override
	            public void call(Subscriber<? super User> observer) {
	                try {
	                    if (!observer.isUnsubscribed()) {
	                        observer.onNext(new User(id, name + id));
	                        observer.onCompleted();
	                    }
	                } catch (Exception e) {
	                    observer.onError(e);
	                }
	            }
	        });
	}
```

###	Fallback

```Java
	@HystrixCommand(fallbackMethod = "defaultUser")
	public User getUserById(String id) {
	    return userResource.getUserById(id);
	}
	
	private User defaultUser(String id) {
	    return new User("def", "def");
	}
```

*	Hystrix命令和fallback应该在同一个类中，并且具有相同的方法签名(失败执行异常的可选参数)。
*  	如果需要异常，在末尾添加Throwable，对访问修饰符无要求。
* 	fallback方法可以继续添加fallback。

### Fallback Exception

```Java
	@HystrixCommand(fallbackMethod = "fallback1")
	User getUserById(String id) {
	    throw new RuntimeException("getUserById command failed");
	}
	
	@HystrixCommand(fallbackMethod = "fallback2")
	User fallback1(String id, Throwable e) {
	    assert "getUserById command failed".equals(e.getMessage());
	    throw new RuntimeException("fallback1 failed");
	}
	
	@HystrixCommand(fallbackMethod = "fallback3")
	User fallback2(String id) {
	    throw new RuntimeException("fallback2 failed");
	}
	
	@HystrixCommand(fallbackMethod = "staticFallback")
	User fallback3(String id, Throwable e) {
	    assert "fallback2 failed".equals(e.getMessage());
	    throw new RuntimeException("fallback3 failed");
	}
	
	User staticFallback(String id, Throwable e) {
	    assert "fallback3 failed".equals(e.getMessage());
	    return new User("def", "def");
	}
	    
	// test
	@Test
	public void test() {
		assertEquals("def", getUserById("1").getName());
	}
```

###	Async/Sync fallback

*	sync command, sync fallback
* 	async command, sync fallback
*  	async command, async fallback

###	Default fallback

该特性允许为整个类或具体的命令定义默认的回退。如果你有一组具有完全相同的回滚逻辑的命令，那么仍然需要为每个命令定义一个fallback，因为fallback应该具有与命令相同的签名，如下所示:

```Java
	 public class Service {
	    @RequestMapping(value = "/test1")
	    @HystrixCommand(fallbackMethod = "fallback")
	    public APIResponse test1(String param1) {
	        // some codes here
	        return APIResponse.success("success");
	    }
	
	    @RequestMapping(value = "/test2")
	    @HystrixCommand(fallbackMethod = "fallback")
	    public APIResponse test2() {
	        // some codes here
	        return APIResponse.success("success");
	    }
	
	    @RequestMapping(value = "/test3")
	    @HystrixCommand(fallbackMethod = "fallback")
	    public APIResponse test3(ObjectRequest obj) {
	        // some codes here
	        return APIResponse.success("success");
	    }
	
	    private APIResponse fallback(String param1) {
	        return APIResponse.failed("Server is busy");
	    }
	
	    private APIResponse fallback() {
	        return APIResponse.failed("Server is busy");
	    }
	    
	    private APIResponse fallback(ObjectRequest obj) {
	        return APIResponse.failed("Server is busy");
	    }
	}
```

默认的fallback特性减少冗余方法：

```Java
	@DefaultProperties(defaultFallback = "fallback")
	public class Service {
	    @RequestMapping(value = "/test1")
	    @HystrixCommand
	    public APIResponse test1(String param1) {
	        // some codes here
	        return APIResponse.success("success");
	    }
	
	    @RequestMapping(value = "/test2")
	    @HystrixCommand
	    public APIResponse test2() {
	        // some codes here
	        return APIResponse.success("success");
	    }
	
	    @RequestMapping(value = "/test3")
	    @HystrixCommand
	    public APIResponse test3(ObjectRequest obj) {
	        // some codes here
	        return APIResponse.success("success");
	    }
	
	    private APIResponse fallback() {
	        return APIResponse.failed("Server is busy");
	    }
	}
```

默认的fallback不含任何参数，除了异常参数以获得执行异常，并且不应该抛出任何异常。以下是按下行优先级的顺序排列:

1.	@HystrixCommand 定义的 fallbackMethod。
2. 	@HystrixCommand 定义的 defaultFallback。
3. 	在 class 上 使用 @DefaultProperties 定义的 defaultFallback。

###	错误传播

@HystrixCommand 提供忽略指定异常不触发fallback的功能。

```Java
    @HystrixCommand(ignoreExceptions = {BadRequestException.class})
    public User getUserById(String id) {
        return userResource.getUserById(id);
    }
```	

如果 userResource.getUserById(id) 抛出一个 BadRequestException，然后这个异常会裹着 HystrixBadRequestException 再抛出来，不会触发fallback。