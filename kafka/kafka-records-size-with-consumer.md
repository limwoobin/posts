#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/kafka-record-example)

## Kafka MessageListener 에서 max.poll.records 옵션의 동작

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

KafkaMessageListenerContainer run -> pollAndInvoke

1. KafkaConsumer.java -> poll method / 1158 line
2. Fetcher -> sendfetches method
3. KafkaConsumer 으ㅣ Fetch 데이터에 레코드 존재함

#### __Reference__

- https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html#max-poll-records
- https://kafka.apache.org/documentation/#consumerconfigs_max.poll.records
- https://yaboong.github.io/spring/2020/06/07/kafka-batch-consumer-unintended-listener-invoking/