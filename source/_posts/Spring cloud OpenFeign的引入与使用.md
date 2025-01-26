---
title: Spring cloud OpenFeign的引入与使用
tags:
  - java
  - gateway
categories:
  - 技术集
thumbnail: "../images/yellow_scene.jpg"

---
------


### 1、基本概念

[Feign](https://github.com/OpenFeign/feign) is a declarative web service client. It makes writing web service clients easier. To use Feign create an interface and annotate it. It has pluggable annotation support including Feign annotations and JAX-RS annotations. Feign also supports pluggable encoders and decoders. Spring Cloud adds support for Spring MVC annotations and for using the same `HttpMessageConverters` used by default in Spring Web. Spring Cloud integrates Eureka, **Spring Cloud CircuitBreaker**, as well as **Spring Cloud LoadBalancer** to provide a load-balanced http client when using Feign.

### 2、openfeign基本使用

#### 2.1 Example spring boot app

```java
@SpringBootApplication
@EnableFeignClients
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}

}
```

#### 2.2 StoreClient.java

```java
@FeignClient("stores")
public interface StoreClient {
	@RequestMapping(method = RequestMethod.GET, value = "/stores")
	List<Store> getStores();

	@GetMapping("/stores")
	Page<Store> getStores(Pageable pageable);

	@PostMapping(value = "/stores/{storeId}", consumes = "application/json",
				params = "mode=upsert")
	Store update(@PathVariable("storeId") Long storeId, Store store);

	@DeleteMapping("/stores/{storeId:\\d+}")
	void delete(@PathVariable Long storeId);
}
```

#### 2.3 注解参数以及使用说明

```java
public @interface FeignClient {

    /**
     * The name of the service with optional protocol prefix. Synonym for {@link #name()
     * name}. A name must be specified for all clients, whether or not a url is provided.
     * Can be specified as property key, eg: ${propertyKey}.
     * @return the name of the service with optional protocol prefix
     * 注册中心服务名
     */
    @AliasFor("name")
    String value() default "";

    /**
     * This will be used as the bean name instead of name if present, but will not be used
     * as a service id.
     * @return bean name instead of name if present
     * 同一个client,使用不同配置
     */
    String contextId() default "";

    /**
     * @return The service id with optional protocol prefix. Synonym for {@link #value()
     * value}.
     * 注册中心服务名
     */
    @AliasFor("value")
    String name() default "";

    /**
     * @return an absolute URL or resolvable hostname (the protocol is optional).
     * url 配置了则不会再取服务名
     */
    String url() default "";

    /**
     * @return whether 404s should be decoded instead of throwing FeignExceptions
     * 接口404是解码返回还是直接抛出异常
     */
    boolean decode404() default false;

    /**
     * A custom configuration class for the feign client. Can contain override
     * <code>@Bean</code> definition for the pieces that make up the client, for instance
     * {@link feign.codec.Decoder}, {@link feign.codec.Encoder}, {@link feign.Contract}.
     *
     * @see FeignClientsConfiguration for the defaults
     * @return list of configurations for feign client
     * 自定义feign 的一些参数（请求参数，编码方式）
     */
    Class<?>[] configuration() default {};

    /**
     * Define a fallback factory for the specified Feign client interface. The fallback
     * factory must produce instances of fallback classes that implement the interface
     * annotated by {@link FeignClient}. The fallback factory must be a valid spring bean.
     *
     * @see FallbackFactory for details.
     * @return fallback factory for the specified Feign client interface
     * 配置熔断的具体实现
     */
    Class<?> fallbackFactory() default void.class;

    /**
     * @return path prefix to be used by all method-level mappings.
     * 路径统一前缀
     */
    String path() default "";

    /**
     * @return whether to mark the feign proxy as a primary bean. Defaults to true.
     * 解决bean定义冲突
     */
    boolean primary() default true;

}
```

#### 2.4  其他配置

```
spring:
	cloud:
		openfeign:
			client:
				config:
					feignName:
                        url: http://remote-service.com
						connectTimeout: 5000
						readTimeout: 5000
						loggerLevel: full
						errorDecoder: com.example.SimpleErrorDecoder
						retryer: com.example.SimpleRetryer
						defaultQueryParameters:
							query: queryValue
						defaultRequestHeaders:
							header: headerValue
						requestInterceptors:
							- com.example.FooRequestInterceptor
							- com.example.BarRequestInterceptor
						dismiss404: false
						encoder: com.example.SimpleEncoder
						decoder: com.example.SimpleDecoder
						contract: com.example.SimpleContract
						micrometer.enabled: false
```

#### 2.5 缓存支持

```java
public interface DemoClient {

	@GetMapping("/demo/{filterParam}")
    @Cacheable(cacheNames = "demo-cache", key = "#keyParam")
	String demoEndpoint(String keyParam, @PathVariable String filterParam);
}
```

#### 2.6 @RequestMapping支持

```
@FeignClient("demo")
public interface DemoTemplate {

        @PostMapping(value = "/stores/{storeId}", params = "mode=upsert")
        Store update(@PathVariable("storeId") Long storeId, Store store);
}
```

#### 2.7 `@RefreshScope` Support

在此路径下开启  `spring.cloud.openfeign.client.config.{feignName}.url`

```
spring.cloud.openfeign.client.refresh-enabled=true
```

#### 2.8 上下文传播

```java
/**
 * 使用自定义的contextPropagator 来传播上下文边界
 */
@Configuration
public class Resilience4jConfig {

    @Resource
    Resilience4jBulkheadProvider resilience4jBulkheadProvider;

    @PostConstruct
    public void concurrentThreadContextStrategy() {
        ThreadPoolBulkheadConfig threadPoolBulkheadConfig = ThreadPoolBulkheadConfig.custom().contextPropagator(new CustomInheritContextPropagator()).build();
        resilience4jBulkheadProvider.configureDefault(id -> new Resilience4jBulkheadConfigurationBuilder()
                .bulkheadConfig(BulkheadConfig.ofDefaults()).threadPoolBulkheadConfig(threadPoolBulkheadConfig)
                .build());
    }

}
public class CustomInheritContextPropagator implements ContextPropagator<SecurityContext> {
    @NotNull
    @Override
    public Supplier<Optional<SecurityContext>> retrieve() {
        return () -> Optional.of(SecurityContextHolder.getContext());
    }


    @NotNull
    @Override
    public Consumer<Optional<SecurityContext>> copy() {
        return optional -> {
            optional.ifPresent(SecurityContextHolder::setContext);
        };
    }

    @NotNull
    @Override
    public Consumer<Optional<SecurityContext>> clear() {
        return optional -> {
            optional.ifPresent(s -> SecurityContextHolder.clearContext());
        };
    }
}
```

2.9  feign logger 支持

```
<dependency>
    <groupId>com.gigacloudtech.b2b</groupId>
    <artifactId>feign-client</artifactId>
    <version>1.0.6</version>
</dependency>
```

![image-20241225102126533](C:\Users\SHENFEI\AppData\Roaming\Typora\typora-user-images\image-20241225102126533.png)



### 3、Spring Cloud LoadBalancer组件整合

#### 3.1 依赖

```xml
   <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>
```

####  3.2 默认算法（轮询）

The `ReactiveLoadBalancer` implementation that is used by default is `RoundRobinLoadBalancer`. To switch to a different implementation, either for selected services or all of them, you can use the [custom LoadBalancer configurations mechanism](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html#custom-loadbalancer-configuration).

| 算法            | 描述               | 优缺点                     |
| --------------- | ------------------ | -------------------------- |
| **Round Robin** | 轮流选择服务器实例 | 简单但可能导致负载不均匀   |
| **Random**      | 随机选择服务器实例 | 简单，但可能导致负载不均匀 |

实例选择算法

| 算法                          | 描述                                     | 优缺点                                       |
| ----------------------------- | ---------------------------------------- | -------------------------------------------- |
| **same-instance-preference**  | 优先选择之前选中的实例                   | 简单，但可能导致负载不均匀                   |
| **RequestBasedStickySession** | 指定instance_id                          | 简单，但可能导致负载不均匀                   |
| **Weighted**                  | 根据权重选择服务器实例，权重越大请求越多 | 适合性能差异较大的情况                       |
| **Hint-Based**                | 基于hint的路由选择                       | 简单                                         |
| **zone-based**                | 根据区域（Zone）优先选择本地实例         | 提高跨区域的容错性，但可能导致跨区域请求延迟 |

#### 3.3  使用，配置实例发现策略以及负载均衡算法

```java
public class LoadBalancerConfig {
    @Bean
    ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(Environment environment,
                                                            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RoundRobinLoadBalancer(loadBalancerClientFactory
                .getLazyProvider(name, ServiceInstanceListSupplier.class),
                name);
    }

    @Bean
    public ServiceInstanceListSupplier discoveryClientServiceInstanceListSupplier(ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
                .withBlockingDiscoveryClient()
                .build(context);
    }
    
}

```

整合openfeign 统一使用

```java
@LoadBalancerClient(name = "b2b-demo-api", configuration = LoadBalancerConfig.class)
@FeignClient(name = "b2b-demo-api", fallbackFactory = B2BApiFeignFallBack.class,path = "b2b-demo")
public interface B2BApiFeign {
    @GetMapping(value = "/demo/testMysql" , headers = "X-api-key=11223344")
    String demo(@RequestParam("name")String name , @RequestParam("password")String password);

    @PostMapping(value = "/demo/testPost" , headers = "X-api-key=11223344")
    String testPost(@RequestBody Map<String , String> map);
}

```



### 3、resilience4J Circuit Breakers

   核心模块

1. resilience4j-circuitbreaker: 熔断器
2. resilience4j-ratelimiter: 请求量限制
3. resilience4j-bulkhead: 隔离板
4. resilience4j-retry: 自动重试

#### 3.1 circuitBreaker 模块

```yml
feign:
  circuitbreaker:
    enabled: true
  client:
    config:
      default:
        #连接超时时间
        connectTimeout: 180000
        #读取超时时间
        readTimeout: 600000
        loggerLevel: FULL
```

##### 3.1.1  配置参数, 与 openfeign 无缝集成

```yml
resilience4j.circuitbreaker:
  instances:
    backendA:
      registerHealthIndicator: true
      slidingWindowSize: 100
    b2bDemoApi:
      registerHealthIndicator: true
      slidingWindowSize: 5
      permittedNumberOfCallsInHalfOpenState: 2
      slidingWindowType: count_based
      minimumNumberOfCalls: 10
      waitDurationInOpenState: 10s
      failureRateThreshold: 50
      eventConsumerBufferSize: 10
      recordExceptions:
        - java.lang.Throwable
      ignoreExceptions:
        - java.lang.IllegalArgumentException
    #      recordFailurePredicate: io.github.robwin.exception.RecordFailurePredicate
  configs:
      default:
        registerHealthIndicator: true
        slidingWindowSize: 5
        permittedNumberOfCallsInHalfOpenState: 5
        slidingWindowType: count_based
        minimumNumberOfCalls: 10
        waitDurationInOpenState: 10s
        failureRateThreshold: 50
        eventConsumerBufferSize: 10
        recordExceptions:
          - java.lang.Throwable
        ignoreExceptions:
          - java.lang.IllegalArgumentException
```

##### 3.1.2 方法级别使用

```java
@Slf4j
@Service
public class TestService {
    @CircuitBreaker(name = "b2bDemoApi",fallbackMethod = "circuitbreakerfallback")
    public String circuitbreaker() {
        log.info("*********进入方法*********");
        int i= RandomUtil.randomInt(0,2);
        if(i<10){
            throw new RuntimeException("error");
        }
        log.info("*********离开方法*********");
        return "";
    }
    public String circuitbreakerfallback(Exception e){
        return "服务器繁忙，请稍后重试";
    }
}
```



#### 3.2 retry模块

##### 3.2.1 配置

```yml
# retry
resilience4j.retry:
  configs:
    default:
      maxAttempts: 3
      waitDuration: 1000
      retryExceptions:
        - java.lang.Throwable
      ignoreExceptions:
        - java.lang.IllegalArgumentException
  instances:
    backendA:
      baseConfig: default
    backendB:
      baseConfig: default
```

##### 3.2.2 代码

```java
    /**
     * 重试机制
     * @return String
     */
    @Retry(name = "b2bDemoApi",fallbackMethod = "retryfallback")
    public String retry(){
        log.info("********进入方法*********");
        int i= RandomUtil.randomInt(0,2);
        if(i<10){
            throw new RuntimeException("retry error");
        }
        log.info("********离开方法*********");
        return "";
    }
    public String retryfallback(Exception e){
        return "重试 服务器繁忙，请稍后重试";
    }
```

#### 3.3  bulkHead 模块

```
    <dependency>
            <groupId>io.github.resilience4j</groupId>
            <artifactId>resilience4j-bulkhead</artifactId>
    </dependency>
```

```java
    @Bulkhead(name = "backendB", fallbackMethod = "bulkheadfallback")
    @GetMapping("/doBulkHead")
    public String doBulkHead() throws InterruptedException {

        TimeUnit.SECONDS.sleep(1);
        String circuitbreaker = testService.bulkHeader();
        log.info("result , {}" , circuitbreaker);
        return circuitbreaker;
    }

    public String bulkheadfallback(Throwable e){
        throw new RuntimeException(e);
    }

```

```yml
# 隔板配置
resilience4j.bulkhead:
  configs:
    default:
      maxConcurrentCalls: 2
  instances:
    backendA:
      maxConcurrentCalls: 10
    backendB:
      # 限流时，请求的最大等待时间
      maxWaitDuration: 10ms
      # 最大并发
      maxConcurrentCalls: 2
```

#### 3.4 ratelimit 模块

```
        <dependency>
            <groupId>io.github.resilience4j</groupId>
            <artifactId>resilience4j-ratelimiter</artifactId>
        </dependency>
```

```java
    @RateLimiter(name = "backendA", fallbackMethod = "rateLimitfallback")
    @GetMapping("/doRateLimit")
    public String doRateLimit() throws InterruptedException {
        String circuitbreaker = testService.bulkHeader();
        log.info("result , {}" , "success");
        return "success";
    }

    public String rateLimitfallback(Throwable e){
        throw new RuntimeException(e);
    }
```

```yaml
resilience4j.ratelimiter:
  configs:
    default:
      registerHealthIndicator: false
      limitForPeriod: 2
      limitRefreshPeriod: 10s
      timeoutDuration: 0
      eventConsumerBufferSize: 1
  instances:
    backendA:
      baseConfig: default
    backendB:
      limitForPeriod: 6
      limitRefreshPeriod: 500ms
      timeoutDuration: 3s
```

