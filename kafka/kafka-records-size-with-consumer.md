#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/kafka-record-example)

## Kafka MessageListener 에서 max.poll.records 옵션의 동작

이번에는 카프카 컨슈머의 `Listener` 와 `max.poll.records` 옵션의 관계 그리고 제가 가지고 있던 오해에 대해 알아보겠습니다.

카프카 컨슈머의 구현체인 리스너는 크게 다음과 같이 나누어져 있습니다.
- `MessageListener` : Record 를 1개씩 처리한다
- `BatchMessageListener` : Record 를 다수를 한번에 처리한다

그리고 `max.poll.records` 옵션은 다음과 같습니다.
- 컨슈머가 `polling` 시 최대로 가져갈 수 있는 record 개수 (defualt - 500)


__그렇다면 `MessageListener` 로 컨슈머를 구현하고 `max.poll.records` 옵션이 10개라고 가정한다면 컨슈머는 데이터를 어떻게 읽어올까요?__  

`MessageListener` 는 record 를 단건으로 가져온다고 했는데 여러개를 가져오게 될까요, 아니면 `max.poll.records` 옵션이 무시되는걸까요?  

이 글은 저 의문에서부터 시작되어 작성하게 되었습니다.

<hr>

KafkaMessageListenerContainer run -> pollAndInvoke

1. KafkaConsumer.java -> poll method / 1158 line
2. Fetcher -> sendfetches method
3. KafkaConsumer 으ㅣ Fetch 데이터에 레코드 존재함

#### __Reference__

- https://docs.confluent.io/platform/current/installation/configuration/consumer-configs.html#max-poll-records
- https://kafka.apache.org/documentation/#consumerconfigs_max.poll.records
- https://yaboong.github.io/spring/2020/06/07/kafka-batch-consumer-unintended-listener-invoking/