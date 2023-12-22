#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/kafka-record-example)

## __Kafka MessageListener 에서 max.poll.records 옵션의 동작__

이번에는 카프카 컨슈머의 __`Listener`__ 와 __`max.poll.records`__ 옵션의 관계, 그리고 제가 가지고 있던 오해에 대해 알아보겠습니다.

먼저 카프카 컨슈머의 구현체인 리스너는 크게 다음과 같이 나누어져 있습니다.
- __`MessageListener`__ : Record 를 1개씩 처리한다
- __`BatchMessageListener`__ : Record 를 다수를 한번에 처리한다

그리고 __`max.poll.records`__ 옵션은 다음과 같습니다.
- 컨슈머가 `polling` 시 최대로 가져갈 수 있는 record 개수 (defualt : 500개)


__그렇다면 `MessageListener` 로 컨슈머를 구현하고 `max.poll.records` 옵션이 10개라고 가정한다면 컨슈머는 데이터를 어떻게 읽어올까요?__  

__`MessageListener`__ 는 record 를 1개씩 단건으로 처리한다고 했는데요, 여러개를 처리할수 있는걸까요 혹은 __`max.poll.records`__ 설정이 무시되는걸까요?

이 글은 저 의문에서 시작되어 작성하게 되었습니다.

<hr>
<br>

## __카프카 컨슈머 예제 코드__

한번 예제코드를 통해 실제 컨슈머에서는 어떻게 동작하는지 확인해보겠습니다.

__application.yml__
```yml
spring:
  kafka:
    consumer:
      bootstrap-servers: localhost:29092
      group-id: record-test-group
    topics:
      record-test: record-test
```

<br />

__KafkaConsumerConfig.java__
```java
@EnableKafka
@Configuration
public class KafkaConsumerConfig {

  @Value("${spring.kafka.consumer.bootstrap-servers}")
  private String bootstrapServers;

  @Value("${spring.kafka.consumer.group-id}")
  private String groupId;

  @Bean
  public ConsumerFactory<String, String> consumerFactory() {
    Map<String, Object> config = consumerConfig();
    return new DefaultKafkaConsumerFactory<>(config);
  }

  @Bean
  public Map<String, Object> consumerConfig() {
    Map<String, Object> config = new HashMap<>();

    config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
    config.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
    config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
    config.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 10); // max.poll.records 설정
    config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

    return config;
  }

  @Bean
  public ConcurrentKafkaListenerContainerFactory<String, String> containerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());

    return factory;
  }
}
```

<br />

__RecordConsumer.java__
```java
@Component
@Slf4j
public class RecordConsumer {

  @KafkaListener(
    topics = "${spring.kafka.topics.record-test}",
    groupId = "${spring.kafka.consumer.group-id}",
    containerFactory = "containerFactory"
  )
  public void consume(@Payload String message, @Headers MessageHeaders messageHeaders) {
    log.info("message: {}", message);
    log.info("messageHeaders: {}", messageHeaders);
  }
}
```

<br />

__`max.poll.records`__ 는 10개로 설정하였습니다.  
그럼 이제 토픽에 메시지5개 정도를 쌓아두고 컨슈머를 실행하면 데이터를 어떻게 읽어오는지 확인해보겠습니다.

![kafka-record-image-1](https://user-images.githubusercontent.com/28802545/292464366-95a85cd6-6f24-4aa9-ac97-8354de90dbbb.png)

<br />

그럼 컨슈머를 실행해보겠습니다. 다음과 같이 __`max.poll.records`__ 가 10개로 잘 설정된것을 볼 수 있습니다.

![kafka-record-image-2](https://user-images.githubusercontent.com/28802545/292465176-19187cec-8c20-461a-8ed4-417695f7629e.png)

<br />

그리고 로그를 확인해보면 다음과 같이 단건으로 실행된것을 확인할 수 있습니다. 

![kafka-record-image-3](https://user-images.githubusercontent.com/28802545/292465202-da4b13fb-0466-4443-9dd6-6c9e0efd2223.png)

__`max.poll.records`__ 설정이 __`MessageListener`__ 에서는 동작하지 않는것일까요?  
그럼 리스너의 페이로드를 `List` 로 변경해서 받아보면 어떨까요?

이번엔 메시지 3개를 프로듀싱하고 다시 컨슈머를 실행해보겠습니다.

![kafka-record-image-4](https://user-images.githubusercontent.com/28802545/292467652-031d5859-8392-454e-9f61-3b941e0750a2.png)

![kafka-record-image-5](https://user-images.githubusercontent.com/28802545/292467676-b59fb30e-a250-4e84-a8ff-3cd1e634ef4c.png)

이번에도 역시 동일하게 단건으로 처리하고 있습니다.  
그렇다면 __`max.poll.records`__ 설정은 __`BatchListener`__ 에서만 유효한걸까요?  

이번에는 컨슈머에서 레코드를 __`polling()`__ 하는 과정을 한번 따라가보겠습니다.

## __Consumer Polling__

<!-- KafkaMessageListenerContainer run -> pollAndInvoke

1. KafkaConsumer.java -> poll method / 1158 line
2. Fetcher -> sendfetches method
3. KafkaConsumer 으ㅣ Fetch 데이터에 레코드 존재함 -->

#### __Reference__

- https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html#max-poll-records
- https://kafka.apache.org/documentation/#consumerconfigs_max.poll.records
- https://yaboong.github.io/spring/2020/06/07/kafka-batch-consumer-unintended-listener-invoking/