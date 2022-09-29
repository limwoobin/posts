# Redis를 이용한 Distribute Lock 동시성 처리


- Redisson 사용 이유 (Lettuce와의 차이)

- annotation 으로 처리한 이유

- 분산락 주의 사항
1. waitTime, leaseTime 사용
2. propagation new 사용 이유
3. db connection pool 주의사항

- annotation 예제 코드

- 분산락 동시성 테스트 코드

Stock 재고차감 
시나리오1
1. 재고100개 생성
2. 동시성 thread 100개 생성
3. thread당 재고 1개씩 차감 실행
4. 재고 0개 확인


db connection pool 주의사항
트랜잭션 내부에서 새로운 트랜잭션을 여는 경우 cp 가 부족하면 cp dead lock을 유발하는 포인트가 될 수 있음
이 부분을 어떻게 풀어나가야하는지??

- facade pattern
- 트랜잭션 단위를 작업별로 짧게 가져
