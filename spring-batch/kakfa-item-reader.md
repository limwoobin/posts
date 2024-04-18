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
  @JobScope
  public Step deadLetterStep(final JobRepository jobRepository,
                             final PlatformTransactionManager platformTransactionManager) {
    return new StepBuilder(STEP_NAME, jobRepository)
      .<String, String>chunk(2, platformTransactionManager)
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
  > ex) partitions(0, 1, 2)
- __(5)__ `consumerProperties(props)` : 컨슈머의 속성을 정의합니다.
- __(6)__ `pollTimeout(Duration.ofSeconds(30L))` : 컨슈머의 polling 에 대한 타임아웃입니다. 기본값은 30s 입니다.
- __(7)__ `partitionOffsets(new HashMap<>())` : 파티션의 오프셋에 대한 설정입니다. 기본값은 오프셋을 0부터 읽어들입니다.  
빈 해시맵을 전달하면 컨슈머가 카프에 저장된 offset Id 부터 읽어들입니다.
- __(8)__ `saveState(false)` : Reader 의 상태를 저장하는 옵션입니다.  
Reader 가 실패한 지점을 알 수 있습니다. 예제에서는 false 로 사용하겠습니다.

__`KafkaItemReader`__ 에는 다음과 같은 설정들이 존재합니다. 여기서 각 설정에 대해 주의해야할 부분들을 이야기해보겠습니다.

<br />

## __KafkaItemReader ㅋㅋ__

<br />

#### reference

- https://docs.spring.io/spring-batch/docs/current/api/org/springframework/batch/item/kafka/KafkaItemReader.html