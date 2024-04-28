
<br />

![kafka-redis-pubsub-1](https://private-user-images.githubusercontent.com/28802545/326169428-9629df9d-8538-4c98-90bb-ef93f05e90c7.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTQyMTEwNDcsIm5iZiI6MTcxNDIxMDc0NywicGF0aCI6Ii8yODgwMjU0NS8zMjYxNjk0MjgtOTYyOWRmOWQtODUzOC00Yzk4LTkwYmItZWY5M2YwNWU5MGM3LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA0MjclMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNDI3VDA5MzkwN1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTU4ZDI0NTkxZDJkMzUxNjJjN2RmMmY0MGI4YjcxNzkyMjc0ZDdkOTBhOGY1Y2UwNzhlY2UzNTRmYjFkMDZjMGQmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.Y3riYP5qpGt_jvjIeXoHl8d6cJQG0CaWR0gMsNxRaWc)

<br />

# __Kafka, Redis 의 Pub/Sub 방식의 차이__

이번에는 Kafka 와 Redis 의 __Pub/Sub__ 기능에 대해 서로 어떤 차이가 있는지 알아보려합니다.

그전에 우선 __Pub/Sub__ 기능이 무었인지 간단하게 살펴보겠습니다.

<br />

## Pub/Sub 이란?

__Pub/Sub__ 은 Publush / Subscribe 의 줄임말입니다. 생산자/소비자 패턴이라고도 불립니다.

이 패턴은 메시지 기반의 미들웨어로 메시지를 발행하는 __발행자(publisher)__ 와 메시지를 수신하는 __구독자(subscribe)__ 로 나누어집니다.  
그리고 중간에 __Topic/Event Channel__ 이 위치하게 됩니다.

이 패턴은 다음과 같은 특징을 가집니다.

- __발행자(publisher)__ 는 이 메시지를 누가 수신하는지 알지도 못하고 관심사가 아닙니다. 그저 __Topic/Event Channel__ 에 메시지를 적재합니다.  
- __구독자(subscribe)__ 또한 이 메시지를 누가 발행했는지 알지도 못하고 관심사가 아닙니다. __Topic/Event Channel__ 에 적재된 메시지를 가져오기만 하면 됩니다.
- 발행자와 구독자 서로의 관심사가 분리되어있기에 __느슨한 결합(Loose Coupling)__ 을 가집니다.
- 비동기식 메시지 패턴입니다.

<br />

Kafka, Redis 서로의 Pub/Sub 방식이 어떻게 다른지 알아보겠습니다.

<br />

## __Kafka Pub/Sub 의 특징__

![kafka-redis-pubsub-2](https://private-user-images.githubusercontent.com/28802545/326185000-e54711ac-4d0f-4c57-a5d9-1c7e3ac8b2c6.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTQyMjg3ODUsIm5iZiI6MTcxNDIyODQ4NSwicGF0aCI6Ii8yODgwMjU0NS8zMjYxODUwMDAtZTU0NzExYWMtNGQwZi00YzU3LWE1ZDktMWM3ZTNhYzhiMmM2LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA0MjclMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNDI3VDE0MzQ0NVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTdhNDM0Mzg4YjMxZGVhZTEzNGEzZWE0YjY3Y2Q4YzI5ZjY0YTAzNjA0ZWQ5ZjMzOGIyMmFhNzk0OWRhZjJjZDAmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.iRvBybLztVDSZYVlbp5r_zA5O5KW56Yi0dVkiXuDMDw)

__Kafka Pub/Sub__ 에서는 Publisher, Subscriber 라는 네이밍보다는 일반적으로 Producer, Consumer 라는 네이밍을 사용합니다.  
Publisher(Producer), Subscriber(Consumer) 이며 역할은 동일합니다.

__Kafka Pub/Sub__ 의 특징은 다음과 같습니다.

- __Consumer 는 주기적으로 Topic 에 메시지가 있는지 확인하며 메시지를 가져온다. (Polling)__
- __Consumer 가 메시지를 가져가도 Topic 의 메시지는 바로 삭제되지 않는다.  
보관 주기/용량이 넘어가면 삭제됨 `(retention.ms / retention.bytes`)__
- __Consumer는 메시지를 읽고 확인(acknowledge)할 수 있어, 메시지의 처리 상태를 추적할 수 있다.__
- Consumer 들은 하나의 Consumer Group 으로 관리된다.
- Consumer 는 각자의 offset 을 가지고 있다.  
offset 은 Consumer 가 메시지를 어디까지 읽었는지 기록한다.

<br />

## Redis Pub/Sub 의 특징

![kafka-redis-pubsub-3](https://private-user-images.githubusercontent.com/28802545/326249471-a498282d-e09b-4eb5-ab38-0ddb13b04a2a.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTQyOTk1NDAsIm5iZiI6MTcxNDI5OTI0MCwicGF0aCI6Ii8yODgwMjU0NS8zMjYyNDk0NzEtYTQ5ODI4MmQtZTA5Yi00ZWI1LWFiMzgtMGRkYjEzYjA0YTJhLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA0MjglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNDI4VDEwMTQwMFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTUwYzFlNDVkMDJiN2I2MThjZTRjNjE4OWY2NDZhMzkwMzViODdlNzYyNmFiNmQ3NmVmMWQ4YTQzNWEzODA0Y2ImWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.NdM9DpNoFqSkkLqhv0ZicOPu30G_rimEnetPy37C5AI)

__Redis Pub/Sub__ 에서는 채널(토픽) 을 통해 데이터를 전송합니다.  
여기서 채널은 Subscriber 가 채널을 구독하는 시점에 생성됩니다.

그리고 채널에 메시지를 발행하게 되면 채널을 구독하고 있는 모든 Subscriber 들에게 메시지가 전송됩니다.

컨텐츠 구독의 기능과 유사하다고 볼 수 있습니다.  
컨텐츠를 구독하는 경우에 구독자들은 새로운 컨텐츠가 등록되면 해당 컨텐츠에 대해 알림을 받게되는데요.  
이때, 컨텐츠 발행이 Publisher 가 되고 컨텐츠 구독자들은 Subscriber 가 되는것입니다. 

```shell
# subscriber
> subscribe {channel_name}
// 채널 생성

# publisher
> publish {channel_name} {message}
// 메시지 발행
```

<br />

__Redis Pub/Sub__ 의 특징은 다음과 같습니다.

- __메시지 발행 시 채널에 Subscriber 가 존재하지 않더라도 메시지는 발행되며 즉시 사라진다.  
(이벤트를 저장하지 않음)__
- __메시지 발행 후 Sucscriber 가 메시지를 정상 수신하였는지 확인하지 않는다.__
- __Subscriber 가 채널의 메시지를 가져가는 방식이 아닌 채널로부터 수신받는 방식이다.__
- Group 이라는 개념이 없기에 Subscriber 들은 각각 이벤트를 모두 수신한다.
- Subscriber 는 특정 패턴을 이용해 채널을 구독할 수 있다.

<br />

## Kafka, Redis Pub/Sub 차이

Kafka 와 Redis 의 Pub/Sub 방식의 차이에 대해 간단하게 정리해보겠습니다.

| id  | Kafka  | Redis |
| --- | ----- | ------ |
| 메시지 저장 여부   | 토픽에 메시지를 일정 주기동안 저장할 수 있음  | 메시지가 Subscriber 에게 전송되면서 채널에서 바로 삭제됨   |
| 메시지 수신 방식   | Consumer 가 polling 방식으로 Topic 에서 메시지를 가져감 | 채널에서 Subscriber 들에게 메시지를 push |
| 전송 보장 | ack 옵션에 따라 보장 가능  | 메시지를 수신했는지 따로 확인하지 않음 |
| 지연시간 | 메시지에 대해 ack 옵션에 따라 수신 확인 등의 작업이 있어 redis 보다는 비교적 느림(그래도 빠르다)  | In-Memory 기반으로 메시지 전송 후 수신 확인같은 별도의 작업이 없어 Kafka 보다 비교적 가볍고 빠름 |

<br />

### __메시지 수신 단위의 차이__

일반적으로 운영되는 서비스들은 배포, Scale-Out, 고가용성 등을 위해 인해 여러대의 인스턴스로 운영됩니다.

이때 이 인스턴스들이 특정 메시지 수신을 위해 Kafka 와 Redis 의 토픽(채널)을 수신하고 있다고 가정해보겠습니다.  
Kafka 의 경우 하나의 메시지가 발행되더라도 여러 인스턴스들은 같은 Consumer Group 으로 묶여있기에 하나의 인스턴스에서만 메시지를 수신합니다.  
Kafka 메시지를 Consumer Group 단위로 수신하기 때문입니다.

Redis 의 경우에는 각 Subscriber 단위로 메시지를 수신합니다.  
즉, 하나의 메시지가 발행되더라도 각 인스턴스들은 모두 메시지를 수신하게 됩니다.

그렇기에 이벤트를 전달하더라도 비즈니스의 성격 혹은 성능을 고려해서 선택해야합니다.  

예를들면 낮은 Latency 를 요구하면서 모든 인스턴스들에게 캐시 동기화같은 기능이 필요한 경우라면 __Redis Pub/Sub__ 이 적합할 수 있습니다.

혹은 의존성을 분리하고 특정 이벤트를 받아 중복없이 한건만 DB 에 저장해야하는 경우라면 __Kafka Pub/Sub__ 이 적합할 수 있습니다.  
__Redis Pub/Sub__ 으로 전달받을 경우 N 건이 저장되어 비즈니스 요구사항이 틀어지거나 경합이 발생할 수 있기 때문입니다.

<br />

#### __reference__

- https://inpa.tistory.com/entry/REDIS-%F0%9F%93%9A-PUBSUB-%EA%B8%B0%EB%8A%A5-%EC%86%8C%EA%B0%9C-%EC%B1%84%ED%8C%85-%EA%B5%AC%EB%8F%85-%EC%95%8C%EB%A6%BC
- https://aws.amazon.com/ko/compare/the-difference-between-kafka-and-redis/