# __Hikari CP 옵션과 설정 방법에 알아보자__

Hikari CP 는 Spring Boot 2.0.0 버전 이상부터 디폴트로 설정된 Connection Pool 입니다.  
이번엔 Hikari CP 의 설정 옵션들은 어떤것들이 있는지, 설정시 고민해야할 부분은 어떤것들이 있을지 한번 알아보려 합니다.

# __Hikari CP 의 옵션__

__application.yml__
```java
spring: 
    datasource: 
    	hikari: 
            driver-class-name: com.mysql.cj.jdbc.Driver 
            jdbc-url: jdbc:mysql://{url}:{port}/{schema} 
            maximum-pool-size: 10
            minimum-idle: 10
            max-lifetime: 10000
            idle-timeout: 5000
            connection-timeout: 30000
```

### __maximumPoolSize__ (default: 10)

- 커넥션 풀의 최대 크기를 의미합니다.

### __connectionTimeout__ (default: 30s)

- 커넥션 풀로부터 커넥션을 획득하기 위해 대기하는 시간입니다.

### __maxLifetime__ (default: 30m)

- 커넥션 풀에서 살아있을 수 있는 최대 시간입니다.
- 사용중인 커넥션은 해당 시간과 상관없이 제거되지 않습니다.

### __keepaliveTime__ (default: 60s)

- 커넥션이 살아있는지 확인하는 주기입니다.
- 30,000ms 보다 작게 설정할 수 없습니다.

### __minimumIdle__ (default: 10)

- 커넥션 풀에서 유지할 최소 유휴 커넥션 개수입니다.
- 기본값은 maximumPoolSize 와 동일한 10개입니다.

### __idleTimeout__

- 유휴 커넥션(idle)이 풀에서 제거되기 전까지 대가하는 시간입니다.
- 유휴 커넥션은 minimumIdle 을 초과한 커넥션에 대해서만 제거됩니다.
- 커넥션의 최대 수명(maxLifeTime) 이 idleTimeout 보다 짧으면, 커넥션은 idleTimeout 전에 닫힙니다.  
그렇기에 maxLifetime > idleTimeout 이 일반적인 설정입니다.

<hr />
<br />

# __권장되는 설정__

### __minimumIdle__

- 일반적으로는 maximumPoolSize 의 사이즈와 동일하게 유지하는것을 권장합니다.
- 트래픽 증가시 응답속도에 대해 항상 동일하게 맞춰 커넥션을 살아있도록 유지하는것이 유리하기 때문입니다.

### __connectionTimeout__

- 해당 설정은 커넥션을 얻기까지 대기하게 됩니다. 하지만 기본값 30초로 설정하게 되는 경우  
사용자는 요청 후 응답을 반환하기 까지 30초 이상이 소요되게 됩니다.
- 대규모 시스템의 경우 이는 사용자 경험에 좋지 않고 톰캣 스레드풀에도 요청이 계속 쌓여 시스템 장애로까지 전파될 수 있습니다.
- 그래서 3초 미만의 짧은 설정을 주어 커넥션이 부족한 경우 빠르게 예외를 응답하는것이 유리할 수 있습니다.  

### __maximumPoolSize__

아래 HikariCP 문서를 참고하면 다음과 같이 이야기합니다.

- 단순히 poolSize 를 키운다고 성능이 빨라지지는 않습니다
	- 실제 하나의 코어에서는 하나의 작업만 진행하기 때문입니다, 그래서 poolSize 가 커지면 오히려 Context-Switching 으로 인해 성능이 저하될 수 있습니다.
	- 그래서 커넥션풀 사이즈를 CPU 코어에 맞게 조정하면 성능 향상을 볼 수 있습니다.
- 코어수와 poolSize 를 동일하게 맞추더라도 디스크와 네트워크가 변수로 작용할 수 있습니다.
	- 디스크를 탐색하거나 네트워크 통신과정에서 blocking 이 발생할 수 있습니다.

__`connections = (core count * 2) + effective_spindle_count`__

- 그래서 PostgreSQL 에서는 위의 설정을 권장합니다.
- core count: CPU 코어 개수
- effectivespindle_count: 하드 디스크의 개수 (spindle은 DB 서버가 관리할 수 있는 동시 I/O 요청 수를 의미함)
- 예시로 하나의 하드디스크가 있는 4-core i7 CPU 에서는 다음과 같이 설정됩니다.
	- 9 = (4 * 2) + 1 

https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing

<hr />
<br />

# 설정시 주의사항들

### __Pool Locking 현상__

Pool Locking 은 하나의 스레드에서 커넥션을 획득하고 이후 중첩해서 추가적인 커넥션을 획득하려 할때 커넥션 풀이 고갈되어 커넥션을 획득하지 못하고 대기하는 현상을 이야기합니다.  
즉, 하나의 스레드에서 두개 이상의 커넥션을 획득하려다 경합이 발생하게 됩니다.

이 경우에는 다음과 같은 설정을 권장하고 있습니다.

Connection Pool Size = Tn * (Cn - 1) + 1
- Tn: 전체 Thread 개수
- Cn: 하나의 요청에서 동시에 필요한 Connection 개수

이는 DeadLock 을 피하기 위한 최소한의 Pool Size 를 의미합니다.  
서비스 특성에 맞게 성능테스트를 통하여 위의 최소 공식을 만족하는 위 설정에 + N 개 정도로 
최적의 poolSize 를 찾는것이 좋지 않을까 싶습니다.

##### __reference__

- https://github.com/brettwooldridge/HikariCP/wiki/MySQL-Configuration
- https://github.com/brettwooldridge/HikariCP#rocket-initialization
- https://techblog.woowahan.com/2663/