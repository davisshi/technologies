Event-driven design with Spring Reactive Redis Pub/Sub model
============================================================

[![Ruchira Madhushan Rajapaksha](https://miro.medium.com/v2/resize:fill:88:88/1*r9mc_Jh4e8TY8D3ynKUGIA.jpeg)](https://medium.com/@maduz.ruchira?source=post_page-----64b152feca8a--------------------------------)
[![Javarevisited](https://miro.medium.com/v2/resize:fill:48:48/1*ceHirGi683U9Xn6tAlr0jQ.jpeg)](https://medium.com/javarevisited?source=post_page-----64b152feca8a--------------------------------)

[Ruchira Madhushan Rajapaksha](https://medium.com/@maduz.ruchira?source=post_page-----64b152feca8a--------------------------------)

![](https://miro.medium.com/v2/resize:fit:700/1*fOWMIT4fMZce6PH2_na4Rg.png)

Publisher Subscriber Pattern

In this article, I will talk about designing an event-driven architecture with the Reactive Redis Publish/Subscribe model.

Overview of the Reactive Redis
==============================

I have already discussed a detailed article about how to integrate Reactive Redis with Spring WebFlux. You can refer to it below.

[Building a Reactive Architecture with Spring WebFlux and Reactive Redis](https://blog.stackademic.com/building-a-reactive-architecture-with-spring-webflux-and-reactive-redis-009fe42b93f7?source=post_page-----64b152feca8a--------------------------------)
-----------------------------------------------------------------------
### [In this article, I will talk about how to design a reactive application with Spring WebFlux and Reactive Redis.](https://blog.stackademic.com/building-a-reactive-architecture-with-spring-webflux-and-reactive-redis-009fe42b93f7?source=post_page-----64b152feca8a--------------------------------)
[blog.stackademic.com](https://blog.stackademic.com/building-a-reactive-architecture-with-spring-webflux-and-reactive-redis-009fe42b93f7?source=post_page-----64b152feca8a--------------------------------)

You can refer to my other articles related to reactive architecture using reactive Redis down below.

[Reactive Event Streaming with Redis Streams in Spring Boot 3](https://blog.stackademic.com/reactive-event-streaming-with-redis-streams-in-spring-boot-3-4bf7dabf7196?source=post_page-----64b152feca8a--------------------------------)
------------------------------------------------------------
### [How to design a reactive architecture for real-time data streaming using Redis streams in Spring Boot 3.](https://blog.stackademic.com/reactive-event-streaming-with-redis-streams-in-spring-boot-3-4bf7dabf7196?source=post_page-----64b152feca8a--------------------------------)
[blog.stackademic.com](https://blog.stackademic.com/reactive-event-streaming-with-redis-streams-in-spring-boot-3-4bf7dabf7196?source=post_page-----64b152feca8a--------------------------------)

[How to handle a write-heavy workload effectively with Spring Reactive Redis](https://blog.stackademic.com/how-to-handle-a-write-heavy-workload-effectively-with-spring-reactive-redis-02e7c6acdf09?source=post_page-----64b152feca8a--------------------------------)
---------------------------------------------------------------------------
### [Building a Reactive Architecture to handle write-heavy workloads around Spring Reactive Redis](https://blog.stackademic.com/how-to-handle-a-write-heavy-workload-effectively-with-spring-reactive-redis-02e7c6acdf09?source=post_page-----64b152feca8a--------------------------------)
[blog.stackademic.com](https://blog.stackademic.com/how-to-handle-a-write-heavy-workload-effectively-with-spring-reactive-redis-02e7c6acdf09?source=post_page-----64b152feca8a--------------------------------)

Redis Publisher/Subscriber Model
================================

Pub/Sub is a messaging model that allows different components in a distributed system to communicate with one another. Publishers send messages to a topic, and subscribers receive messages from that topic, allowing publishers to send messages to subscribers while remaining anonymous. The Pub/Sub system ensures that the message reaches all subscribers who are interested in the topic.

One distinguishable aspect of the Redis Pub/Sub model is that messages published by the publishers to the channels will be pushed by Redis to all the subscribed clients. Subscribers receive the messages in the order that they are published.

![](https://miro.medium.com/v2/resize:fit:700/1*TdfNz9LBh49If16KoMESqg.png)

Redis Pub/Sub Model

Application Setup with Spring Boot and Reactive Redis
=====================================================

Project Technical Stack
-----------------------

1.  Spring Boot 3.2.0
2.  Java 21
3.  Redis
4.  Docker

Starting Up Redis Server and Redis Insight
------------------------------------------

We will use Docker to start Redis with Redis Insight. The port of our Redis server is 6379, and the port of Redis Insight is 8001.
```
docker run -d --name redis-stack -p 6379:6379 -p 8001:8001 redis/redis-stack:latest
```
We will then set up the application with Spring Initializer with the following dependencies in the `pom.xml` file.
```
<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>

<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-webflux</artifactId>
</dependency>

<dependency>
<groupId>org.projectlombok</groupId>
<artifactId>lombok</artifactId>
<optional>true</optional>
</dependency>

<dependency>
<groupId>io.projectreactor</groupId>
<artifactId>reactor-test</artifactId>
<scope>test</scope>
</dependency>

<dependency>
<groupId>org.springdoc</groupId>
<artifactId>springdoc-openapi-starter-webflux-ui</artifactId>
<version>2.3.0</version>
</dependency>
```
Setting Up Configurations
-------------------------

Configure the`application.yml` file for the SpringBoot application as follows: We will define the configurations related to Redis inside it.
```
redis:
host: localhost
port: 6379
```
Reactive Redis Pub/Sub Configuration
------------------------------------

`RedisMessageListenerContainer`acts as a message-listener container. It is used to receive messages from a Redis channel and drive the `MessageListener` instances that are injected into it.
```
@Configuration
@Slf4j
public class RedisConfig {

    @Value("${redis.host}")
    private String host;

    @Value("${redis.port}")
    private int port;

    @Bean
    @Primary
    public ReactiveRedisConnectionFactory reactiveRedisConnectionFactory() {
        return new LettuceConnectionFactory(host, port);
    }

    @Bean
    public ReactiveRedisOperations<String, Product> reactiveRedisOperations(ReactiveRedisConnectionFactory reactiveRedisConnectionFactory) {
        Jackson2JsonRedisSerializer<Product> serializer = new Jackson2JsonRedisSerializer<>(Product.class);

        RedisSerializationContext.RedisSerializationContextBuilder<String, Product> builder =
                RedisSerializationContext.newSerializationContext(new StringRedisSerializer());

        RedisSerializationContext<String, Product> context = builder.value(serializer).build();

        return new ReactiveRedisTemplate<>(reactiveRedisConnectionFactory, context);
    }

    @Bean
    public ReactiveRedisMessageListenerContainer messageListenerContainer(final ProductService productService, final ReactiveRedisConnectionFactory reactiveRedisConnectionFactory) {
        final ReactiveRedisMessageListenerContainer container = new ReactiveRedisMessageListenerContainer(reactiveRedisConnectionFactory);
        final ObjectMapper objectMapper = new ObjectMapper();
        container.receive(ChannelTopic.of("products"))
                .map(ReactiveSubscription.Message::getMessage)
                .map(message -> {
                    try {
                        return objectMapper.readValue(message, Product.class);
                    } catch (IOException e) {
                        return null;
                    }
                })
                .switchIfEmpty(Mono.error(new IllegalArgumentException()))
                .flatMap(productService::save)
                .subscribe(c-> log.info("Product Saved Successfully."));
        return container;
    }

}
```
Product Publisher Class
-----------------------

The Product Publisher class acts as the `Redis Publisher`, publishing to the desired channel.
```
@Component
@RequiredArgsConstructor
@Slf4j
public class ProductPublisher {

    private static final String REDIS_CHANNEL = "PRODUCT_CHANNEL";
    private final ReactiveRedisOperations<String, Product> reactiveRedisOperations;

    public Mono<String> publishProductMessage(final Product product) {
        this.reactiveRedisOperations.convertAndSend(REDIS_CHANNEL, product).subscribe(count -> {
            log.info("Product Published Successfully.");
        });
        return Mono.just("Published Message to Redis Channel");
    }

}
```
Product Service Class
---------------------
```
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ReactiveRedisOperations<String, Product> reactiveRedisOperations;
    private static final String REDIS_KEY = "PRODUCT:REDIS";

    public Flux<Product> findAll() {
        return this.reactiveRedisOperations.opsForList().range(REDIS_KEY, 0, -1);
    }

    public Mono<Long> save(final Product product) {
        final String id = UUID.randomUUID().toString().substring(0, 8);
        product.setId(id);
        return this.reactiveRedisOperations.opsForList().rightPush(REDIS_KEY, product);
    }

}
```
Product Controller Class
------------------------
```
@RestController
@RequestMapping("/product")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;
    private final ProductPublisher productPublisher;

    @GetMapping
    public Flux<Product> findAll(){
        return this.productService.findAll();
    }

    @PostMapping("/publish")
    public Mono<String> publishProduct(final Product product){
        return this.productPublisher.publishProductMessage(product);
    }

}
```
Testing the Application
=======================

We will use the Spring Doc Open API to publish product messages to the Redis Channel.

![](https://miro.medium.com/v2/resize:fit:700/1*NQO2OITjBBUgZxmwrB6j2g.png)

Publish Product Message to Redis Channel

Redis Insight Dashboard with Published Products
-----------------------------------------------

![](https://miro.medium.com/v2/resize:fit:700/1*KKfvBmkvgV0AD_s4NaswQg.png)

Published Products in Redis Database

Summary
=======

In this article, we have covered:

1.  Overview of Reactive Redis
2.  Redis Publisher Subscriber model
3.  How to start a Redis Server and Redis Insight with Docker
4.  Reactive Redis Pub/Sub Integration with Spring Boot.
5.  Testing the Rest Endpoints using Spring Doc Open API

Please feel free to share your feedback.

Thanks for reading.

[Spring Boot](https://medium.com/tag/spring-boot?source=post_page-----64b152feca8a---------------spring_boot-----------------)

[Redis](https://medium.com/tag/redis?source=post_page-----64b152feca8a---------------redis-----------------)

[Event Driven Architecture](https://medium.com/tag/event-driven-architecture?source=post_page-----64b152feca8a---------------event_driven_architecture-----------------)

[Reactive Programming](https://medium.com/tag/reactive-programming?source=post_page-----64b152feca8a---------------reactive_programming-----------------)

[Software Architecture](https://medium.com/tag/software-architecture?source=post_page-----64b152feca8a---------------software_architecture-----------------)

[![Ruchira Madhushan Rajapaksha](https://miro.medium.com/v2/resize:fill:144:144/1*r9mc_Jh4e8TY8D3ynKUGIA.jpeg)](https://medium.com/@maduz.ruchira?source=post_page-----64b152feca8a--------------------------------)
[![Javarevisited](https://miro.medium.com/v2/resize:fill:64:64/1*ceHirGi683U9Xn6tAlr0jQ.jpeg)](https://medium.com/javarevisited?source=post_page-----64b152feca8a--------------------------------)

[Written by Ruchira Madhushan Rajapaksha](https://medium.com/@maduz.ruchira?source=post_page-----64b152feca8a--------------------------------)
---------------------------------------

[203 Followers](https://medium.com/@maduz.ruchira/followers?source=post_page-----64b152feca8a--------------------------------)

-Writer for

[Javarevisited](https://medium.com/javarevisited?source=post_page-----64b152feca8a--------------------------------)

Tech Guy | Cloud Native Enthusiast | Sharing is Caring | Find me: <https://bento.me/ruchirarajapaksha> | Email: <ruchirarajapaksha.business@gmail.com>