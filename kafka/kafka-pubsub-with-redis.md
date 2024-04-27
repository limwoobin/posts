
<br />

![kafka-redis-pubsub-1](https://private-user-images.githubusercontent.com/28802545/326169428-9629df9d-8538-4c98-90bb-ef93f05e90c7.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTQyMTEwNDcsIm5iZiI6MTcxNDIxMDc0NywicGF0aCI6Ii8yODgwMjU0NS8zMjYxNjk0MjgtOTYyOWRmOWQtODUzOC00Yzk4LTkwYmItZWY5M2YwNWU5MGM3LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA0MjclMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNDI3VDA5MzkwN1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTU4ZDI0NTkxZDJkMzUxNjJjN2RmMmY0MGI4YjcxNzkyMjc0ZDdkOTBhOGY1Y2UwNzhlY2UzNTRmYjFkMDZjMGQmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.Y3riYP5qpGt_jvjIeXoHl8d6cJQG0CaWR0gMsNxRaWc)

# Kafka, Redis 의 Pub/Sub 방식의 차이

이번에는 Kafka 와 Redis 의 __Pub/Sub__ 기능에 대해 서로 어떤 차이가 있는지 알아보려합니다.

<br />

## Pub/Sub 이란?

그전에 우선 __Pub/Sub__ 기능이 무었인지 먼저 알아야 할텐데요.  
__Pub/Sub__ 은 Publush / Subscribe 의 줄임말입니다. 즉 생산자/소비자 패턴이라고도 불립니다.

이 패턴은 메시지 기반의 미들웨어로 메시지를 발행하는 __발행자(publisher)__ 와 메시지를 수신하는 __구독자(subscribe)__ 로 나누어집니다.  
그리고 이 중간에 __Topic/Event Channel__ 이 위치하게 됩니다.

이 패턴은 다음과 같은 특징을 가집니다.

- __발행자(publisher)__ 는 이 메시지를 누가 수신하는지 알지도 못하고 관심사가 아닙니다. 그저 __Topic/Event Channel__ 에 메시지를 적재합니다.  
- __구독자(subscribe)__ 또한 이 메시지를 누가 발행했는지 알지도 못하고 관심사가 아닙니다. __Topic/Event Channel__ 에 적재된 메시지를 가져오기만 하면 됩니다.
- 발행자와 구독자 서로의 관심사가 분리되어있기에 __느슨한 결합(Loose Coupling)__ 을 가집니다.
- 비동기식 메시지 패턴입니다.

그럼 이제 Kafka,Redis 서로의 Pub/Sub 방식이 어떻게 다른지 알아보겠습니다.

<br />

## Kafka Pub/Sub 의 특징

<br />

## Redis Pub/Sub 의 특징

<br />

## Kafka, Redis Pub/Sub 차이 정리

<br />

#### __reference__