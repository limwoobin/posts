# Redis를 이용한 Distribute Lock 동시성 처리


- Redisson 사용 이유 (Lettuce와의 차이)

- annotation 으로 처리한 이유

- 분산락 주의 사항
1. waitTime, leaseTime 사용
2. propagation new 사용 이유
3. db connection pool 주의사항

- annotation 예제 코드

- 분산락 동시성 테스트 코드
