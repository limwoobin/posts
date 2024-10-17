# Rate Limiter

처리율 제한 장치
- 클라이언트 또는 서버가 보내는 요청을 제한하기 위핸 장치 (트래픽 처리율 제한)

Why
- 디도스 공격에 의한 자원고갈
- 잘못된 이용패턴으로 인해 유발된 트래픽을 막아 서버 과부하 방지

알고리즘 종류
- Leaky Bucket
- Token Bucket
- Fixed Window
- Sliding Window log
- Sliding Window Counter

HTTP Header
- X-Ratelimit-Limit	: 클라이언트가 보낼 수 있는 API별 총 요청 한도.
- X-Ratelimit-Remaining	: 남은 API 요청 횟수. 이 횟수 만큼 추가 요청을 보낼 수 있다.
- X-Ratelimit-Retry-After : 다음 API 요청을 시도하기 전에 대기해야 하는 시간
- X-Ratelimit-Reset : 요청 최댓값이 재설정 될 때까지의 시간
- X-Daily-Requests-Left : API 요청에서 사용 가능한 남은 일일 요청 횟수

라이브러리
- bucket4j (토큰 버킷 알고리즘)
- guava (토큰 버킷 알고리즘)
- resilience4j
