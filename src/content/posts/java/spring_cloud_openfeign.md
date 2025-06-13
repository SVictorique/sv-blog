---
title: Spring Cloud OpenFeign学习笔记
published: 2025-06-13 13:41:00
description: ''
tags: [分布式,Java,Spring,SpringCloud]
category: 学习笔记
draft: false
---
# Introduce
[官方文档](https://docs.spring.io/spring-cloud-openfeign/reference/index.html)

>Feign is a declarative web service client. It makes writing web service clients easier. To use Feign create an interface and annotate it. It has pluggable annotation support including Feign annotations and JAX-RS annotations. Feign also supports pluggable encoders and decoders. Spring Cloud adds support for Spring MVC annotations and for using the same HttpMessageConverters used by default in Spring Web. Spring Cloud integrates Eureka, Spring Cloud CircuitBreaker, as well as Spring Cloud LoadBalancer to provide a load-balanced http client when using Feign.
# 引入
添加dependency`org.springframework.cloud:spring-cloud-starter-openfeign`
```java
@SpringBootApplication
@EnableFeignClients
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}

}

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
# 配置
可以通过`configuration`属性执行配置类。
```kotlin
@FeignClient(name = "stores", configuration = FooConfiguration.class)
public interface StoreClient {
	//..
}
```
默认的Bean包括：
- Decoder
- Encoder
- Logger
- MicrometerObservationCapability
- MicrometerCapability
- CachingCapability
- Contract
- Feign.Builder
- Client

也可以直接将配置写在`application.yaml`配置文件中：
```yaml
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
						responseInterceptor: com.example.BazResponseInterceptor
						dismiss404: false
						encoder: com.example.SimpleEncoder
						decoder: com.example.SimpleDecoder
						contract: com.example.SimpleContract
						capabilities:
							- com.example.FooCapability
							- com.example.BarCapability
						queryMapEncoder: com.example.SimpleQueryMapEncoder
						micrometer.enabled: false
```
# 超时处理
- connectTimeout
- readTimeout
# 手动创建Feign Client
```java
@Import(FeignClientsConfiguration.class)
class FooController {

	private FooClient fooClient;

	private FooClient adminClient;

	@Autowired
	public FooController(Client client, Encoder encoder, Decoder decoder, Contract contract, MicrometerObservationCapability micrometerObservationCapability) {
		this.fooClient = Feign.builder().client(client)
				.encoder(encoder)
				.decoder(decoder)
				.contract(contract)
				.addCapability(micrometerObservationCapability)
				.requestInterceptor(new BasicAuthRequestInterceptor("user", "user"))
				.target(FooClient.class, "https://PROD-SVC");

		this.adminClient = Feign.builder().client(client)
				.encoder(encoder)
				.decoder(decoder)
				.contract(contract)
				.addCapability(micrometerObservationCapability)
				.requestInterceptor(new BasicAuthRequestInterceptor("admin", "admin"))
				.target(FooClient.class, "https://PROD-SVC");
	}
}
```
# 压缩
可以设置对request和response开启GZIP压缩
```property
spring.cloud.openfeign.compression.request.enabled=true
spring.cloud.openfeign.compression.response.enabled=true

spring.cloud.openfeign.compression.request.mime-types=text/xml,application/xml,application/json
spring.cloud.openfeign.compression.request.min-request-size=2048
```
# Micrometer
开启Micrometer监控，需要满足一下条件：
- feign-micrometer在classpath上
- ObservationRegistry的Bean存在
- 开启micrometer配置
  - spring.cloud.openfeign.micrometer.enabled=true (for all clients)
  - spring.cloud.openfeign.client.config.feignName.micrometer.enabled=true (for a single client)
# 缓存
使用`@EnableCaching`注解会开启缓存，使得接口上的`@Cache*`注解生效。
```java
public interface DemoClient {

	@GetMapping("/demo/{filterParam}")
    @Cacheable(cacheNames = "demo-cache", key = "#keyParam")
	String demoEndpoint(String keyParam, @PathVariable String filterParam);
}
```
# @RequestMapping支持
```java
@FeignClient("demo")
public interface DemoTemplate {

        @PostMapping(value = "/stores/{storeId}", params = "mode=upsert")
        Store update(@PathVariable("storeId") Long storeId, Store store);
}
```
# @QueryMap支持
```java
// Params.java
public class Params {
	private String param1;
	private String param2;

	// [Getters and setters omitted for brevity]
}

@FeignClient("demo")
public interface DemoTemplate {

	@GetMapping(path = "/demo")
	String demoEndpoint(@SpringQueryMap Params params);
}
```
# 指定url
- `@FeignClient(name="testClient", url="http://localhost:8081")`
- `spring.cloud.openfeign.client.config.testClient.url=http://localhost:8081`
# 完整配置
[参考](https://docs.spring.io/spring-cloud-openfeign/reference/configprops.html)
