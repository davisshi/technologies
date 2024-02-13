Resilient Application Development with Reactor Kafka and Spring WebFlux
=======================================================================

[![Ruchira Madhushan Rajapaksha](https://miro.medium.com/v2/resize:fill:88:88/1*r9mc_Jh4e8TY8D3ynKUGIA.jpeg)](https://medium.com/@maduz.ruchira?source=post_page-----5c8e8badd6a0--------------------------------)
[![Stackademic](https://miro.medium.com/v2/resize:fill:48:48/1*U-kjsW7IZUobnoy1gAp1UQ.png)](https://blog.stackademic.com/?source=post_page-----5c8e8badd6a0--------------------------------)

[Ruchira Madhushan Rajapaksha](https://medium.com/@maduz.ruchira?source=post_page-----5c8e8badd6a0--------------------------------)

![](https://miro.medium.com/v2/resize:fit:700/1*SqeaJEAAYrBPcJvEcTCnBg.png)

What is Kafka?
==============

Kafka is a distributed system comprising `servers` and `clients` that communicate via a high-performance [TCP network protocol](https://kafka.apache.org/protocol.html). We can deploy it on bare-metal hardware, virtual machines, and containers in on-premise and cloud environments.

Kafka provides three key capabilities:

-   `Publish and subscribe` to streams of events, like a message queue.
-   `Storage` Messages can be consumed asynchronously. Data is replicated for failure tolerance by Kafka and is written in a scalable disk structure. Producers can hold off until written acknowledgements.
-   `Stream processing` Complex aggregations or joins of input streams onto an output stream of processed data are made possible by the Kafka Streams API.
-   `Messaging`: replacement for a more traditional message broker to decouple processing from data producers.
-   `Website activity tracking:` tracking pipeline as a set of real-time publish-subscribe feeds.
-   `Metrics:` Aggregating statistics from distributed applications to produce centralized feeds of operational data.
-   `Log Aggregation:` Event data is transformed into a stream of messages as an alternative to file-based log aggregation.
-   `Stream Processing:` processing pipelines where data is consumed from topics and then aggregated, enriched, or otherwise transformed into new topics for further consumption or follow-up processing.
-   `Event Sourcing:` State changes are logged as a time-ordered sequence of records.
-   `Commit Log:` As an external commit log for a distributed system.

Kafka Cluster Setup
-------------------

For this Demo, I will use [Confluent Kafka](https://www.confluent.io/) to set up my Kafka Cluster. With Confluent, you can set up a Kafka Cluster on the Cloud or on-premise easily. Refer to the below [guide](https://docs.confluent.io/cloud/current/get-started/confluent-cloud-basics.html) on how to set up Confluent Kafka Cluster on the cloud.

Kafka Application Setup with SpringBoot
=======================================

We will set up the application with Spring Initializer with the following dependencies in the `pom.xml` file.
```xml
<dependency>
<groupId>org.springframework.kafka</groupId>
<artifactId>spring-kafka</artifactId>
</dependency>

<dependency>
<groupId>io.projectreactor.kafka</groupId>
<artifactId>reactor-kafka</artifactId>
</dependency>

<dependency>
<groupId>org.springframework.boot</groupId>
<artifactId>spring-boot-starter-webflux</artifactId>
</dependency>

<dependency>
<groupId>io.cloudevents</groupId>
<artifactId>cloudevents-kafka</artifactId>
</dependency>

<dependency>
<groupId>io.cloudevents</groupId>
<artifactId>cloudevents-json-jackson</artifactId>
<version>2.4.2</version>
</dependency>

<dependency>
<groupId>io.cloudevents</groupId>
<artifactId>cloudevents-spring</artifactId>
<version>2.4.2</version>
</dependency>
```
Kafka Configuration
-------------------

The configurations related to the Confluent are defined inside the `application.yml` inside the resource folder.
```yml
CONFLUENT:
URL: CONFLUENT_URL_OF_KAFKA_CLUSTER
USERNAME: USERNAME
PASSWORD: PASSWORD
SCHEMA_REG_URL: SCHEMA_REG_URL
SCHEMA_REG_USER: SCHEMA_REG_URL

KAFKA_GROUP_ID: EVENT_GROUP
KAFKA_TOPIC_NAME: EVENT_GROUP
```
Kafka Basic Configurations
--------------------------

The Basic Configuration class defines the Configurations common to both `Kafka Consumer` and `Kafka Producer`.
```config
@Configuration
public class KafkaBasicConfig {
@Value("${CONFLUENT.URL}")
private String bootstrapServer;
@Value("${CONFLUENT.USERNAME}")
private String userName;
@Value("${CONFLUENT.PASSWORD}")
private String password;
@Value("${CONFLUENT.SCHEMA_REG_URL}")
private String schemaRegistryUrl;
@Value("${CONFLUENT.SCHEMA_REG_USER}")
private String schemaRegistryUser;

    protected Map<String, Object> getBasicConfig() {
        Map<String, Object> config = new HashMap<>();
        config.put("bootstrap.servers", this.bootstrapServer);
        config.put("ssl.endpoint.identification.algorithm", "https");
        config.put("security.protocol", "SASL_SSL");
        config.put("sasl.mechanism", "PLAIN");
        config.put("sasl.jaas.config", String.format("org.apache.kafka.common.security.plain.PlainLoginModule required username="%s" password="%s";", this.userName, this.password));
        config.put("basic.auth.credentials.source", "USER_INFO");
        config.put("schema.registry.basic.auth.user.info", this.schemaRegistryUser);
        config.put("schema.registry.url", this.schemaRegistryUrl);
        return config;
    }

    protected Properties getBasicConfigProperty() {
        Properties config = new Properties();
        config.put("bootstrap.servers", this.bootstrapServer);
        config.put("ssl.endpoint.identification.algorithm", "https");
        config.put("security.protocol", "SASL_SSL");
        config.put("sasl.mechanism", "PLAIN");
        config.put("sasl.jaas.config", String.format("org.apache.kafka.common.security.plain.PlainLoginModule required username=\"%s" password="%s";", this.userName, this.password));
        config.put("basic.auth.credentials.source", "USER_INFO");
        config.put("schema.registry.basic.auth.user.info", this.schemaRegistryUser);
        config.put("schema.registry.url", this.schemaRegistryUrl);
        return config;
    }
}
```
Kafka Producer Configurations
-----------------------------

The configuration class defines the properties to be used by the `Kafka producer,` after which it uses them to instantiate a `KafkaSender` and `ReactiveKafkaProducerTemplate` that can publish events.
```configuration
@Configuration
public class KafkaPublisherConfig extends KafkaBasicConfig {

    private SenderOptions<String, CloudEvent> getSenderOptions() {
        Map<String, Object> props = getBasicConfig();
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, CloudEventSerializer.class);
        props.put(CloudEventSerializer.ENCODING_CONFIG, Encoding.STRUCTURED);
        props.put(CloudEventSerializer.EVENT_FORMAT_CONFIG, JsonFormat.CONTENT_TYPE);

        return SenderOptions.create(props);
    }

    @Bean
    public KafkaSender<String, CloudEvent> kafkaSender() {
        return KafkaSender.create(getSenderOptions());
    }

    @Bean
    public ReactiveKafkaProducerTemplate<String, CloudEvent> reactiveKafkaProducerTemplate() {
        return new ReactiveKafkaProducerTemplate<>(getSenderOptions());
    }
}
```
Kafka Consumer Configurations
-----------------------------

The configuration class defines the properties to be used by the `Kafka producer,` after which it uses them to instantiate a `KafkaReceiver` and `ReactiveKafkaConsumerTemplate` that is used to consume events.
```configuration
@Configuration
public class KafkaReceiverConfig extends KafkaBasicConfig {

    @Value("${KAFKA_GROUP_ID}")
    public String KafkaGroupId;

    @Value("${KAFKA_TOPIC_NAME}")
    public String kafkaTopicName;

    public ReceiverOptions<String, CloudEvent> getReceiverOptions() {
        Map<String, Object> basicConfig = getBasicConfig();

        basicConfig.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "latest");
        basicConfig.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        basicConfig.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        basicConfig.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        basicConfig.put(ConsumerConfig.GROUP_ID_CONFIG, KafkaGroupId);

        basicConfig.put(ErrorHandlingDeserializer.KEY_DESERIALIZER_CLASS, StringDeserializer.class);
        basicConfig.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, CloudEventDeserializer.class);

        ReceiverOptions<String, CloudEvent> basicReceiverOptions = ReceiverOptions.create(basicConfig);

        return basicReceiverOptions
                .subscription(Collections.singletonList(kafkaTopicName));
    }

    @Bean
    public KafkaReceiver<String, CloudEvent> kafkaReceiver() {
        return KafkaReceiver.create(getReceiverOptions());
    }

    @Bean
    public ReactiveKafkaConsumerTemplate<String, CloudEvent> reactiveKafkaConsumerTemplate() {
        return new ReactiveKafkaConsumerTemplate<>(getReceiverOptions());
    }

}
```
The setup is quite typical, however, one notable feature is the use of Spring Kafka's `ErrorHandlingDeserializer`. Although Spring Kafka is not a dependency of `Reactor Kafka` which is a completely independent project, we find these deserializers to be very useful in ensuring that the consumer can handle and recover from deserialization errors, which would otherwise cause it to effectively get stuck at the faulty record. Implementing equivalent, specialized deserializers for delegation that are also equipped with error-handling features could be a substitute strategy that would get rid of the need for the Spring Kafka library.

Kafka Producer Pipeline
=======================

Here We will publish a Cloud Event Object using `ReactiveKafkaProducerTemplate.` For a piece of detailed information about cloud events, refer to the [link](https://cloudevents.io/) here.
```
@Service
@RequiredArgsConstructor
@Slf4j
public class PublisherService {

    @Value("${KAFKA_TOPIC_NAME}")
    public String kafkaTopicName;
    private final CloudEventUtil<Event> cloudEventUtil;
    private final ReactiveKafkaProducerTemplate<String, CloudEvent> kafkaProducerTemplate;

    public void publish(String key) {

        final Event event = Event.builder().eventName("Kafka Event").build();
        final CloudEvent cloudEvent = cloudEventUtil.pojoCloudEvent(event, UUID.randomUUID().toString());

        kafkaProducerTemplate.send(SenderRecord.create(new ProducerRecord<>(
                        kafkaTopicName,
                        key, cloudEvent), new Object()))
                .doOnError(error -> log.info("unable to send message due to: {}", error.getMessage())).subscribe(record -> {
                    RecordMetadata metadata = record.recordMetadata();
                    log.info("send message with partition: {} offset: {}", metadata.partition(), metadata.offset());
                });
    }
}
```
Kafka Consumer Pipeline
=======================

In Reactor Kafka, the abstraction of choice on Kafka Consumer is an inbound Flux where all events received from Kafka are published by the framework. This Flux is created by calling one of the `receive`, `receiveAtmostOnce`, `receiveAutoAck`, `receiveExactlyOnce` methods on the `ReactiveKafkaProducerTemplate`.

In our implementation, the Flux is created and subscribed after the dependency injection is done to perform any initialization, In other words, the subscription should continue to be active as long as the service is available.
```
public Disposable onNEventReceived() {
    return reactiveKafkaConsumerTemplate
            .receive()
            .doOnError(error -> {
            log.error("Error receiving event, Will retry", error);
            })
            .retryWhen(Retry.fixedDelay(Long.MAX_VALUE, Duration.ZERO.withSeconds(5)))
            .doOnNext(consumerRecord -> log.info("received key={}, value={} from topic={}, offset={}",
            consumerRecord.key(), consumerRecord.value(), consumerRecord.topic(), consumerRecord.offset())
            )
            .concatMap(this::handleEvent)
            .doOnNext(event -> {
            log.info("successfully consumed {}={}", Event.class.getSimpleName(), event);
            }).subscribe(record-> {
            record.receiverOffset().commit();
            });
}
```
Error Handling on the Consumer Side
===================================

The first crucial point to remember is that with a `reactive publisher`, an error is a `terminal signal` that results in the subscription being terminated. With regard to physical reactive Kafka consumers, the implication is that any mistake that is thrown anywhere along the pipeline will force the consumer to effectively shut down. Sadly, this also implies that the service instance will keep running even though it is not actually consuming any Kafka events. To address this problem, we add a retry mechanism using the `retryWhen` operator, which makes sure that failures are caught and that, in the event of one, the upstream publisher is re-subscribed and the Kafka consumer is re-created.

This retry will only catch problems delivered by the `Kafka consumer` itself because it is positioned just after the source publisher in the reactive pipeline and before the actual event processing. The retry strategy also stipulates a practically endless number of retries at a 5-second interval. This is because launching the service instance without a functioning Kafka consumer inside is useless, thus the best course of action is to keep trying until one of two events occurs:

`Transient Errors`: The consumer will ultimately re-subscribe and begin effectively consuming events.

`Non Transient Errors`: The error log that was printed before the retry will cause an alert, prompting a human operator to intervene and look into the problem. (explain the subtleties of retriable and interval alerts)

Event Handling Mechanism and Acknowledgement
============================================

When an event is received, the application must process it before acknowledging it. At least once delivery semantics are provided by this sequencing; the pipeline may alter if at-most-once or exactly-once semantics are used instead. According to the pattern we suggest, event processing is delegated to a different method called handleEvent, which always returns the ReceiverRecord that the subscriber uses to acknowledge the offset. However, the operator we select to refer to this approach has a significant influence on the consumer's behaviour.

Here we are using the `concatMap` operator as it creates and subscribes to the inner publishers sequentially. This is extremely important in scenarios where the events must be processed in the exact order in which they were read from the partition.

Proposed Architecture for Resilient Event Handling
==================================================

All of the logic involved in handling a single event is included in the `handleEvent` function. The `ReceiverRecord` containing the Kafka event is often the actual input parameter to this function, and it is also the record that is anticipated to be returned when the event has been processed successfully or unsuccessfully.
```java
public Mono<ReceiverRecord<String, CloudEvent>> handleEvent(ReceiverRecord<String, CloudEvent> record) {
        return Mono.just(record).<CloudEvent>handle((result, sink) -> {
                    List<DeserializationException> exceptionList = getExceptionList(result);
                    if (!CollectionUtils.isEmpty(exceptionList)) {
                        logError(exceptionList);
                    } else {
                        if (Objects.nonNull(result.value())) {
                            sink.next(result.value());
                        }
                    }
                }).flatMap(this::processRecord)
                .doOnError(error -> {
                    log.error("Error Processing event: key {}", record.key(), error);
                }).onErrorResume(error-> Mono.empty())
                .then(Mono.just(record));
    }
```
The first step is to look for any `key/value` `deserialization` problems, assuming the pipeline has been appropriately assembled and subscribed to. Any such error will be placed as a header on the `ReceiverRecord` since the consumer is set to use Spring Kafka's `ErrorHandlingDeserializer` and in those circumstances, the actual record value will be null.

When `deserialization` fails, we simply report the error and don't publish the event to the sink, thus discarding it. We utilize the handle operator to make sure that we only process events for which deserialization was successful. We send the event value downstream for processing if there are no deserialization errors.

The next stage is where we actually process the event.

Following the exhaustion of all retries, the `onErrorResume` operator is designed to handle any subscription-time problems, which often originate from the event processing component. The operation continues once the error is swallowed by `onErrorResume` the current snippet, which prints an error log. Similar to before, depending on the needs, alternative error-handling techniques could be used in this situation.

Summary
=======

In this article, we have covered:

1.  Overview of Kafka and typical use cases.
2.  Kafka Cluster Setup with Confluent Kafka.
3.  Kafka Producer Consumer Configurations for reactive application.
4.  Kafka Producer Pipeline and Consumer Pipeline.
5.  Resilient Event Handling Architecture for Reactive Kafka.

The GitHub repo for the above article can be found [here](https://github.com/LordMaduz/reactor-kafka-spring-webflux).

Please feel free to share your feedback.

[Visit to find more of my articles on Medium.](https://medium.com/@maduz.ruchira)

Thanks for reading.

Stackademic
===========

*Thank you for reading until the end. Before you go:*

-   *Please consider **clapping** and **following** the writer! 👏*
-   *Follow us on *[*Twitter(X)*](https://twitter.com/stackademichq)*, *[*LinkedIn*](https://www.linkedin.com/company/stackademic)*, and *[*YouTube*](https://www.youtube.com/c/stackademic)*.*
-   *Visit *[*Stackademic.com*](http://stackademic.com/)* to find out more about how we are democratizing free programming education around the world.*

[Reactor](https://medium.com/tag/reactor?source=post_page-----5c8e8badd6a0---------------reactor-----------------)

[Spring Boot](https://medium.com/tag/spring-boot?source=post_page-----5c8e8badd6a0---------------spring_boot-----------------)

[Event Driven Architecture](https://medium.com/tag/event-driven-architecture?source=post_page-----5c8e8badd6a0---------------event_driven_architecture-----------------)

[Reactive Programming](https://medium.com/tag/reactive-programming?source=post_page-----5c8e8badd6a0---------------reactive_programming-----------------)

[Kafka](https://medium.com/tag/kafka?source=post_page-----5c8e8badd6a0---------------kafka-----------------)


[![Ruchira Madhushan Rajapaksha](https://miro.medium.com/v2/resize:fill:144:144/1*r9mc_Jh4e8TY8D3ynKUGIA.jpeg)](https://medium.com/@maduz.ruchira?source=post_page-----5c8e8badd6a0--------------------------------)
[![Stackademic](https://miro.medium.com/v2/resize:fill:64:64/1*U-kjsW7IZUobnoy1gAp1UQ.png)](https://blog.stackademic.com/?source=post_page-----5c8e8badd6a0--------------------------------)

[Written by Ruchira Madhushan Rajapaksha](https://medium.com/@maduz.ruchira?source=post_page-----5c8e8badd6a0--------------------------------)
---------------------------------------


[203 Followers](https://medium.com/@maduz.ruchira/followers?source=post_page-----5c8e8badd6a0--------------------------------)

-Writer for

[Stackademic](https://blog.stackademic.com/?source=post_page-----5c8e8badd6a0--------------------------------)

Tech Guy | Cloud Native Enthusiast | Sharing is Caring | Find me: <https://bento.me/ruchirarajapaksha> | Email: <ruchirarajapaksha.business@gmail.com>
