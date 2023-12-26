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

<br />

## __Kafka Consumer Polling 동작 원리__

컨슈머는 메시지를 가져올때 주기적으로 브로커에게 __`poll()`__ 메소드를 통해서 데이터를 가져오고 있습니다.  

주기적으로 `polling` 하는곳을 먼저 찾아가 보겠습니다.  

### 1. __KafkaMessageListenerContainer.java__

    ![kafka-record-image-7](https://user-images.githubusercontent.com/28802545/292733327-134f61c8-e35c-487b-9db1-df0a6fd88445.png)

    <br />

    `KafkaMessageListenerContainer` 의 run 메소드를 살펴보면 무한루프를 돌면서 주기적으로 __`pollAndInvoke()`__ 를 이용해 주기적으로 메시지를 `polling` 하고 있습니다.

    <br />

    ![kafka-record-image-8](https://user-images.githubusercontent.com/28802545/292733486-72fa6b48-bfa6-4e59-b424-f2d9a83721b2.png)

    <br />

    그리고 `doPoll()` 메소드를 통해 `ConsumerRecord` 로 메시지를 읽어 가져옵니다.  
    그럼 한번 `doPoll` 메소드를 살펴보겠습니다.

    <br />

    ![kafka-record-image-9](https://user-images.githubusercontent.com/28802545/292735186-f99e7fc2-ae69-4ecb-a95f-a34e96ce7458.png)

    <br />

    - `isBatchListener`: 배치 리스너인지 여부를 반환
    - `subBatchperPartition`: 배치를 파티션별로 분할할지 여부를 반환, default is null

    저희는 배치 리스너도 아니고 `subBatchperPartition` 설정도 별도로 하지 않았기에 두 값 모두 false 이기에 else 로 빠지게 됩니다.

    <br />

    ![kafka-record-image-10](https://user-images.githubusercontent.com/28802545/292735646-23a9835b-a85c-4daf-affb-83582849ca75.png)

    <br />

    그리고 `pollConsumer()` 를 따라가보니 consumer 의 poll 메소드를 호출하네요. 계속 따라가 보겠습니다.

<br />

### 2. __KafkaConsumer.java__
    ![kafka-record-image-11](https://user-images.githubusercontent.com/28802545/292736244-34306cb1-5d7a-4f08-9441-a646137bfe2d.png)

    <br />

    해당 `poll()` 메소드에서는 지정한 타임아웃 시간만큼 루프를 돌면서 `pollForFetches(timer)` 메소드를 호출합니다.

    <br />    

    ![kafka-record-image-12](https://user-images.githubusercontent.com/28802545/292740262-e0fefd65-209a-4abb-a159-8eb4cf8561a5.png)

    - __sendFetches()__: Fetch 요청을 Queue 에 넣습니다. 이후 RequestFuture 타입을 반환하고 onSuccess 콜백 메서드에서 fetch 가 완료된 레코드들을 completedFetches 큐에 넣습니다.
    - __client.poll()__: __`ConsumerNetworkClient`__ 객체에서 fetch 요청을 실제 네트워크를 태워서 보냅니다.
    - __fetcher.collectFetch()__: completedFetches 큐에 레코드가 있으면 반환합니다.  
    __이때, `max.poll.records` 의 개수만큼 반환합니다.__

    `pollForFetches` 메소드에서는 다음과 같은 동작을 수행합니다.  
    이때, 실제 네트워크 요청을 하는 __`ConsumerNetworkClient`__ 객체는 데이터를 `bytes` 형태로 가져와 keyDeserializer / valueDeseializer 를 통해  
    역직렬화 후 completedFetches 에 데이터를 넣습니다.

    그럼 이제 __`fetcher.collectFetch()`__ 내부를 들여다보겠습니다.

<br />

### 3. __AbstractFetch.collectFetch__

    ![kafka-record-image-13](https://user-images.githubusercontent.com/28802545/292864791-dff30651-46d0-4386-98d1-b419699766fc.png)

    <br />

    `collectFetch()` 메소드를 간략하게 가져왔습니다.  
    recordsRemaining 는 `max.poll.records` 의 설정 값이고 그 아래를 보면 `recordsRemaining > 0` 의 경우 계속 루프를 돌며 레코드를 가져오는 것을 볼 수 있습니다.

    `Fetch<K, V nextFetch = fetchRecords(recordsRemaining)` 에서는 recordsRemaining 수 만큼 레코드를 가져오고 있습니다.

    <br />

    __CompletedFetch.fetchRecords__

    ![kafka-record-image-14](https://user-images.githubusercontent.com/28802545/292863070-e7460809-5a47-4a81-a6df-aef4ed7d0e7a.png)

    <br />

    다음을 보시면 `maxRecords` 만큼 for loop 를 수행하면서 records 를 넣고 offset 을 기록합니다.  
    만약 `max.poll.records` 값이 5이고 컨슘해야할 레코드들이 10개라면 5개만 가져오게 되는것이죠.

    이후 `recordsRemaining -= nextFetch.numRecords();` 는 가져온 레코드 수 만큼 recordsRemaining 를 차감하고 있습니다.  

    그래서 만약 recordsRemaining 이 5개이고 `fetchRecords(recordsRemaining)` 를 통해 5개의 레코드를 가져왔다면 차감되어 recordsRemaining 는 0이 되고 while 조건은 종료되게 됩니다.  

    반대로 `fetchRecords(recordsRemaining)` 으로 가져온 레코드의 개수가 recordsRemaining 보다 적다면 그 다음 읽어온 records 가 없기 때문에 아래 코드와 같이 루프가 종료되게 됩니다.  

    ```
    CompletedFetch<K, V> records = completedFetches.peek();
    if (records == null) break;
    ```
    
<hr />
<br />

## 결론

컨슈머의 __polling__ 과정을 따라가 보며 __`max.poll.records`__ 설정은 __`MessageListener`__ 가 한번에 처리할 수 있는 레코드의 개수가 아니라  
컨슈머가 브로커에게 메시지를 가져오기 위해 `polling` 요청으로 가져올 수 있는 레코드의 최대 개수라는것을 알았습니다.

컨슈머는 poll 요청으로 `max.poll.records` 개수만큼 레코드를 한번에 가져온 후  
내부적으로 리스너에게 단건 혹은 배치로 전달하게 됩니다.

문서에도 polling 시 가져오는 최대 개수라고 명시되어있지만 저는 그것을 그대로 리스너에게 전달하거나, 혹은 `max.poll.records` 설정이 동작하지 않나? 라는 잘못된 오해를 하고있었는데요.  
이번 기회를 통해 컨슈머의 내부 동작과정을 얕게나마 살펴본것이 많은 도움이 된것 같습니다.

감사합니다.

#### __Reference__

- https://kafka.apache.org/documentation/#consumerconfigs_max.poll.records
- https://yaboong.github.io/spring/2020/06/07/kafka-batch-consumer-unintended-listener-invoking/
- https://d2.naver.com/helloworld/0974525