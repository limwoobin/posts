#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/kafka-item-reader-example)

<br />

# __Spring Batch KafkaItemReader 란?__

안녕하세요. 이번에는 Spring Batch 의 ItemReader 중 하나인 __`KafkaItemReader`__ 에 대해 알아보겠습니다.  
__`KafkaItemReader`__ 은 __`Spring Batch`__ 에서 제공하는 __`ItemReader<T>`__ 를 구현하고 있으며  
__`KafkaConsumer`__ 를 이용해 카프카의 토픽, 파티션에 있는 데이터를 읽어들일 수 ItemReader 중 하나입니다.

토픽에 있는 데이터를 일괄로 처리가 필요한 경우에 __`KafkaItemReader`__ 를 사용하기 용이합니다.

<br />

## __KafkaItemReader 예시__

매일 오전 카프카의 __`DeadLetterTopic`__ 에 있는 데이터를 조회하고 가공해야 하는 작업이 있다고 가정하겠습니다.  
__`DeadLetterTopic`__ 의 데이터를 읽어들여서 로그로 출력하는 간단한 예제를 만들어보겠습니다.

<br />

```
- Java 17
- SpringBoot 3.2.4
- SpringBatch 5.1.1
```

<br />

__build.gradle__
```gradle
implementation 'org.springframework.boot:spring-boot-starter-batch'
implementation 'org.springframework.kafka:spring-kafka'
```

<br />

__application.yml__
```yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/batch
    username: root
    password: 1234
  kafka:
    consumer:
      bootstrap-servers: localhost:29092
    topics:
      dead-letter: dead-letter

```

<br />

__DeadLetterJobConfig.java__
```java
@Configuration
public class DeadLetterJobConfig {
  public static final String JOB_NAME = "deadLetterJob";

  private final Step deadLetterStep;

  public DeadLetterJobConfig(Step deadLetterStep) {
    this.deadLetterStep = deadLetterStep;
  }

  @Bean
  public Job deadLetterJob(final JobRepository jobRepository) {
    return new JobBuilder(JOB_NAME, jobRepository)
      .start(deadLetterStep)
      .build();
  }
}
```

<br />

__DeadLetterStepConfig.java__
```java
@Configuration
public class DeadLetterStepConfig {
  public static final String STEP_NAME = "deadLetterStep";

  private final DeadLetterItemReader deadLetterItemReader;
  private final DeadLetterItemWriter deadLetterItemWriter;

  public DeadLetterStepConfig(DeadLetterItemReader deadLetterItemReader,
                              DeadLetterItemWriter deadLetterItemWriter) {
    this.deadLetterItemReader = deadLetterItemReader;
    this.deadLetterItemWriter = deadLetterItemWriter;
  }

  @Bean
  public Step deadLetterStep(final JobRepository jobRepository,
                             final PlatformTransactionManager platformTransactionManager) {
    return new StepBuilder(STEP_NAME, jobRepository)
      .<String, String>chunk(5, platformTransactionManager)
      .reader(deadLetterItemReader.deadLetterKafkaItemReader())
      .writer(deadLetterItemWriter)
      .build();
  }
}
```

<br />

__DeadLetterItemReader.java__
```java
@Component
public class DeadLetterItemReader {
  private final KafkaProperties kafkaProperties;
  private final SslBundles sslBundles;

  public DeadLetterItemReader(KafkaProperties kafkaProperties, SslBundles sslBundles) {
    this.kafkaProperties = kafkaProperties;
    this.sslBundles = sslBundles;
  }

  @Value("${spring.kafka.topics.dead-letter}")
  private String topic;


  public KafkaItemReader<String, String> deadLetterKafkaItemReader() {
    Properties props = new Properties();
    props.putAll(kafkaProperties.buildConsumerProperties(sslBundles));
    props.put("group.id", "dlt-consumer-group");

    return new KafkaItemReaderBuilder<String, String>()
      .name("deadLetterKafkaItemReader")
      .topic(topic)
      .partitions(0)
      .consumerProperties(props)
      .pollTimeout(Duration.ofSeconds(5L))
      .partitionOffsets(new HashMap<>())
      .saveState(true)
      .build();
  }
}
```

<br />

__DeadLetterItemWriter.java__
```java
@Slf4j
@Component
public class DeadLetterItemWriter implements ItemWriter<String> {

  @Override
  public void write(Chunk<? extends String> items) {
    for (String item : items) {
      log.info("item: {}", item);
    }

    log.info("-----------------------------------");
  }
}
```

<br />

전체적인 Job 의 구성은 다음과 같습니다.  
여기서 이번의 주제인 __`KafkaItemReader`__ 에 대해 조금 더 자세히 살펴보겠습니다.

<br />

__DeadLetterReader.java__
```java
@Component
public class DeadLetterItemReader {
  private final KafkaProperties kafkaProperties;
  private final SslBundles sslBundles;

  public DeadLetterItemReader(KafkaProperties kafkaProperties, SslBundles sslBundles) {
    this.kafkaProperties = kafkaProperties;
    this.sslBundles = sslBundles;
  }

  @Value("${spring.kafka.topics.dead-letter}")
  private String topic;


  public KafkaItemReader<String, String> deadLetterKafkaItemReader() {
    Properties props = new Properties(); 
    props.putAll(kafkaProperties.buildConsumerProperties(sslBundles)); // (1)
    props.put("group.id", "dlt-consumer-group");

    return new KafkaItemReaderBuilder<String, String>()
      .name("deadLetterKafkaItemReader") // (2)
      .topic(topic) // (3)
      .partitions(0) // (4)
      .consumerProperties(props) // (5)
      .pollTimeout(Duration.ofSeconds(30L)) // (6)
      .partitionOffsets(new HashMap<>()) // (7)
      .saveState(true) // (8)
      .build();
  }
}
```
- __(1)__ `props.putAll(kafkaProperties.buildConsumerProperties(sslBundles));` : KafkaItemReader 내의 KafkaConsumer 에게 할당할 Consumer 속성을 정의합니다
- __(2)__ `name("deadLetterKafkaItemReader")` : Reader 의 인스턴스 이름을 정의합니다.
- __(3)__ `topic(topic)` : 읽어들일 토픽을 지정합니다.
- __(4)__ `partitions(0)` : 토픽의 파티션을 지정합니다. 매개변수는 가변인자로 되어있기에 여러 토픽을 지정할 수 있습니다.
  ex) `partitions(0, 1, 2)`
- __(5)__ `consumerProperties(props)` : 컨슈머의 속성을 정의합니다.
- __(6)__ `pollTimeout(Duration.ofSeconds(30L))` : 컨슈머의 polling 에 대한 타임아웃입니다. 기본값은 30s 입니다.
- __(7)__ `partitionOffsets(new HashMap<>())` : 파티션의 오프셋에 대한 설정입니다. 기본값은 오프셋을 0부터 읽어들입니다.  
빈 해시맵을 전달하면 컨슈머가 카프에 저장된 offset Id 부터 읽어들입니다.
- __(8)__ `saveState(false)` : Reader 의 상태를 저장하는 옵션입니다.  
Reader 가 실패한 지점을 알 수 있습니다. 예제에서는 false 로 사용하겠습니다.

__`KafkaItemReader`__ 에는 다음과 같은 설정들이 존재합니다. 여기서 두 설정에 조금 더 알아보겠습니다.

### __`pollTimeout()`__

__`pollTimeout()`__ 은 컨슈머의 polling 에 대한 타임아웃 설정이고, 기본값은 30s 입니다.  
이 상태로 read() 를 하게 되면 컨슈머는 30s 동안은 계속해서 데이터를 polling 하게 됩니다.

그렇기에 읽어들일 데이터에 대해 `fetch.min.bytes`, `fetch.max.wait.ms`, `max.poll.records` 와 `pollTimeout` 과 같은 컨슈머 조회에 영향을 주는 옵션들에 대해 고민이 필요합니다.

- __`fetch.min.bytes`__(기본값: 1bytes) : 한번에 가져올 수 있는 최소 사이즈로, 만약 가져오는 데이터가 지정한 사이즈보다 데이터가 누적될 때 까지 기다림
- __`fetch.max.wait.ms`__(기본값: 500ms) : fetch.min.bytes에 설정된 데이터보다 데이터 양이 적은 경우 요청에 응답을 기다리는 최대시간
- __`max.poll.records`__(기본값: 500개) : polling 시 최대로 가져올 수 있는 record 개수

<br />

### __`partitionOffsets()`__

__`partitionOffsets`__ 은 파티션의 오프셋에 대한 설정입니다.  

![kafka-item-reader-image-1](https://private-user-images.githubusercontent.com/28802545/323615298-ebda9416-b41d-4829-9f59-1e47bb90e920.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTM4ODI5ODMsIm5iZiI6MTcxMzg4MjY4MywicGF0aCI6Ii8yODgwMjU0NS8zMjM2MTUyOTgtZWJkYTk0MTYtYjQxZC00ODI5LTlmNTktMWU0N2JiOTBlOTIwLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA0MjMlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNDIzVDE0MzEyM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTczNTc5ZjAxZWYxMzIyNDRjYjQ3ZDlkYWJjMTE5NzYyYWU4NTBmMTdlYmExOThiZWRhMjBlMmUyYTgwMTVhMTAmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.prckmWgHiu30eNCtlwQC0tl2RhI7L7mR2Cjnr5z64dg)

##### __https://docs.spring.io/spring-batch/docs/current/api/org/springframework/batch/item/kafka/KafkaItemReader.html#setPartitionOffsets(java.util.Map)__

> <br />
> 이 매핑은 리더에게 각 파티션에서 읽기를 시작할 오프셋을 알려줍니다.  
> 이는 선택 사항이며, 기본값은 각 파티션의 오프셋 0부터 시작됩니다.  
> 빈 맵을 전달하면 판독기가 소비자 그룹 ID에 대해 Kafka에 저장된 오프셋에서 시작됩니다.
> 
> <br />

<br />

번역기를 돌려보니 다음과 같이 이야기하고있네요.  
그렇다면 KafkaItemReader 내부를 한번 살펴보겠습니다.

![kafka-item-reader-image-2](https://private-user-images.githubusercontent.com/28802545/323927544-bc935af3-3eb7-4ca8-ad4d-03ce39df2aa3.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTM4ODI5ODMsIm5iZiI6MTcxMzg4MjY4MywicGF0aCI6Ii8yODgwMjU0NS8zMjM5Mjc1NDQtYmM5MzVhZjMtM2ViNy00Y2E4LWFkNGQtMDNjZTM5ZGYyYWEzLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA0MjMlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNDIzVDE0MzEyM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTYyYjIzNWJmNTNmYzI2NDZjNWYxNmJmYjkxMDRjMTVjNjQ2YzVmNmI0NmIwYzhiMmJhMzAzZGE1Yjk3OTQzOWUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.mk3W01SfYCfHYT53BZ2HfRw7VbFQ_Zab9fPr_d_4u9k)

`open()` 메소드에서 partitionOffsets 설정이 null 인 경우에는 정말 각 파티션들의 offset 을 0으로 설정하는 것을 볼 수 있습니다.  
그렇기에 빈 맵을 전달하면 `if (this.partitionOffsets == null)`  
조건에 해당하지 않기에 컨슈머의 groupId 에 저장된 offset 정보를 기반으로 데이터를 읽을 수 있습니다.

++ 위 조건만 피하면 되기에 빈 맵을 전달하는 대신 인자를 대체할 수 있는 HashTable 을 전달해도 동일하게 동작합니다.

e.g) `partitionOffsets(new HashTable<>())`

<br />

## __KafkaItemReader 주의사항__

### Spring Batch 버전에 따른 offset 관리 이슈

__`SpringBatch 4.36`__ 이전의 버전에는 __`KafkaItemReader`__ 의 __`partitionOffsets`__ 옵션이 존재하지 않습니다.  
그래서 오프셋에 대한 관리를 할 수 없게되는 상황이 존재합니다.

그렇다고 저 오프셋관리를 위해 __`SpringBatch`__ 버전을 올리기에는 어떤 리스크가 존재할지 몰라 위험할 수 있습니다.
그런 경우 상위 버전의 __`KafkaItemReader`__ 의 코드를 가져와서 만들어 사용하는 방법을 고민해 볼 수 있습니다.

<br />

__CustomKafkaItemReader.java__
```java
public class CustomKafkaItemReader<K, V> extends AbstractItemStreamItemReader<V> {

  private static final String TOPIC_PARTITION_OFFSETS = "topic.partition.offsets";
  private static final long DEFAULT_POLL_TIMEOUT = 30L;
  private List<TopicPartition> topicPartitions;
  private Map<TopicPartition, Long> partitionOffsets;
  private KafkaConsumer<K, V> kafkaConsumer;
  private Properties consumerProperties;
  private Iterator<ConsumerRecord<K, V>> consumerRecords;
  private Duration pollTimeout = Duration.ofSeconds(DEFAULT_POLL_TIMEOUT);
  private boolean saveState = true;

  ...
}
```

<br />

### KafkaItemReader 실행환경에 따른 offset 최신화

배치 Job 을 여러번 실행해도 오프셋이 갱신되지 않는 이슈입니다.  
Job 을 어떻게 실행시키느냐에 따라 발생할 수 있는 문제인데요.

Job 을 실행할때마다 __KafkaItemReader__ 의 Bean 이 새로 로드된다면 문제가 없습니다.  
다만 어플리케이션에 실행된 상태에서 이미 Bean 이 띄워진 Reader 에 대해서는 Job을 여러번 실행하게 되어도 처음 실행 시점의 컨슈머의 current offset 을 계속 읽게됩니다.

컨트롤러를 이용해 jobLauncher 로 배치를 실행하는 경우를 예로 들 수 있습니다.

좀 더 자세하게 예시를 통해 알아보겠습니다.

![kafka-item-reader-image-3](https://private-user-images.githubusercontent.com/28802545/324235915-ae2419ad-189b-4b32-91b1-389a406267cf.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTM4ODI5ODMsIm5iZiI6MTcxMzg4MjY4MywicGF0aCI6Ii8yODgwMjU0NS8zMjQyMzU5MTUtYWUyNDE5YWQtMTg5Yi00YjMyLTkxYjEtMzg5YTQwNjI2N2NmLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA0MjMlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNDIzVDE0MzEyM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTYxNjU4MmNiMGVjZGNkZmIwM2VmMGYzODRlODc0OWEwYTJhYzdjOTQyMWFiNDAwODExM2YwZWI5ZmE4N2UzNzgmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.Ojpzxz9topUYppc2LMWSTp8D0sZdRn4t8mdBEKTwv2I)

처음 배치 실행 시 __KafkaItemReader__ 컨슈머의 current offset 은 11입니다.  
현재 offset 은 15까지 데이터가 존재하니 __KafkaItemReader__ 는 11부터 15까지의 데이터를 모두 가져오게 됩니다.  

![kafka-item-reader-image-2](https://private-user-images.githubusercontent.com/28802545/324236266-eda412c1-2066-4563-91db-6b4879c1c6c9.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTM4ODI5ODMsIm5iZiI6MTcxMzg4MjY4MywicGF0aCI6Ii8yODgwMjU0NS8zMjQyMzYyNjYtZWRhNDEyYzEtMjA2Ni00NTYzLTkxZGItNmI0ODc5YzFjNmM5LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA0MjMlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNDIzVDE0MzEyM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWRjNWMxZGY3ZGRhYjMwMzVhODgwNzc2ZGYzZWMwMThmMjYxNWU2MjM4MmRiYTlmMjliMzllYjljYWI5MWM4YjgmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.SCzg-arwuReABJ-UOIO-PCurIxZgg36qip9HN_MPeOg)

그리고 해당 컨슈머의 current offset 은 아래와 같이 16으로 변경됩니다.

이후 Job 을 다시 실행하게된다면 offset 을 어디서부터 읽어올까요?  
current offset 은 16번이니 16번부터 읽어올것으로 생각되지만 실제로는 처음 Bean 이 로드될 때 시점의 11번 offset 부터 읽게됩니다.

__왜 그럴까요?__

카프카 컨슈머의 current offset 은 Kafka Broker 내의 Coordinator 에서 관리됩니다.  
즉, 카프카 서버에서 관리된다고 볼 수 있습니다. 

그리고 __`KafkaItemReader`__ 의 컨슈머의 경우에는 처음 Bean 이 로드될때 카프카 서버로부터 __`current offset`__ 을 가져오게 됩니다.  
이후 Bean 이 새로 로드되지 않는 이상은 처음 읽어왔던 current offset 이 갱신되지 않는것입니다.

#### 해결방안

해결방안은 Job 실행시마다 Bean 을 새로 로드시켜주면 __`current offset`__ 을 실행시마다 갱신할 수 있습니다.  
바로 SpringBatch 의 LateBinding 을 이용하는건데요 __`@JobScope, @StepScope`__ 를 이용하여 해당 Job 에 대해 LateBinding 으로 처리하는 것입니다.


```java
@Bean
@JobScope // LateBinding
public Step deadLetterStep(final JobRepository jobRepository,
                            final PlatformTransactionManager platformTransactionManager) {
  return new StepBuilder(STEP_NAME, jobRepository)
    .<String, String>chunk(5, platformTransactionManager)
    .reader(deadLetterItemReader.deadLetterKafkaItemReader())
    .writer(deadLetterItemWriter)
    .build();
}
```

<br />

#### reference

- https://docs.spring.io/spring-batch/docs/current/api/org/springframework/batch/item/kafka/KafkaItemReader.html