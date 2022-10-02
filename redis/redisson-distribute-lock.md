[**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/lock-example)

# **Redisson 라이브러리를 이용한 Distribute Lock 동시성 처리**

Redis를 통한 분산락을 이용해 동시성을 해결하는 방법에 대해 알아보고 예제코드를 함께 공유드리려 합니다.

<br>

## **분산 락(Distribute Lock) ?**

Lock: DB의 트랜잭션의 순차적 처리를 보장하기 위한 방법  
여러 서버에서 동기화된 처리를 하기 위해 Database, Redis와 같은 공통된 저장소를 이용한 방법  
**(공통된 저장소를 사용해 여러 서버에 대한 동기화된 처리가 가능함)**

<br>

## **Redisson 사용 이유**

Spring에서 제공하는 대표적인 redis 라이브러리로는 Lettuce가 있습니다.  
하지만 Lettuce는 다음과 같은 단점이 존재합니다.

```shell
1. Lock의 타임아웃이 지정되있지 않음
- 락 획득 후 모종의 이유로 어플리케이션이 종료된다거나 했을 경우엔 락은 해제되지 않음
그렇게 되면 타 프로세스들은 락이 해제되기만을 기다리는 무한정 대기상태로 빠지게됨, 이는 곧 시스템 장애로 이어짐

2. 스핀락으로 인해 Redis에 많은 부담을 가하게 됨
락을 획득하지 못한 경우 Redis에게 락을 획득하기 위해 계속 요청을 보내게 됨
```

Redisson은 다음과 같은 장점이 있습니다.

```shell
1. Lock에 타임아웃을 지정할 수 있음
- Redisson은 락 획득시도시 타임아웃을 명시하게 되어있습니다.
그래서 무한정 대기상태로 빠질 수 있는 위험이 없습니다.
2. pub/sub방식을 사용하므로 스핀락을 사용하지 않음
- 락이 해제되면 락을 subscribe하는 클라이언트들에게 락이 해제되었다는 신호를 보내게 됩니다.
그렇기에 락을 subscribe하는 클라이언트들은 더 이상 락을 획득해도 되냐고 redis로 요청을 보내지 않습니다.
```

<br>
<hr>

### **Redisson 라이브러리**

Spring에서 Redisson을 사용하기 위해선 아래의 의존성이 필요합니다.  
그리고 Redisson에서 제공하는 인터페이스와 사용법을 간략하게 소개하겠습니다.

build.gradle

```java
dependencies {
	// redisson
	implementation 'org.redisson:redisson-spring-boot-starter:3.17.4'
}
```

Redisson에서는 Lock을 사용하기 위해 `RLock` 이라는 인터페이스를 제공합니다.

락을 획득하기 위해서는 `tryLock`이라는 메소드를 이용합니다.

```java
boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException;
```

- **waitTime:** 락을 획득하기 위한 대기 시간
- **leaseTime:** 락을 임대하는 시간
- **unit:** 시간 단위

<br>

```java
RLock rLock = redissonClient.getLock(key); // (1)

try {
		boolean available = rLock.tryLock(5, 3, TimeUnit.SECONDS); // (2)
		if (!available) {	// (3)
			return false;
		}

		// 락 획득 후 수행 로직...
} catch (InterruptedException e) {
		e.printStackTrace();
		throw new InterruptedException();
} finally {
		rLock.unlock(); // (4)
}
```

```Shell
(1) - key 이름에 해당하는 RLock 인스턴스를 가져온다
(2) - Lock 획득을 시도한다 (성공: true / 실패: false)
(3) - 획득 실패시 Lock을 subscribe 하며 해제되길 기다린다
(4) - finally에서 Lock을 해제한다
```

<br>
<hr>

## **Redisson 분산락 annotation 기반으로 사용하기**

저는 Redisson 분산락 처리를 annotation 기반으로 작성해서 사용해보았습니다.  
annotation기반으로 분산락을 사용하려는 이유는 다음과 같습니다.

```shell
- 개발 효율성 향상
(annotatino을 이용해 분산락 처리를 보다 손쉼게 사용)
- 비지니스로직과 분산락 처리 로직의 관심사 분리
(각자의 역할만 담당하여 코드 가독성 증대)
- 코드 재사용성 향상
```

다음과 같은 이유를 고려해 분산락을 annotation기반으로 작성하는 것으로 선택하였습니다.

이제 Redisson 기반 분산락을 사용하기 위한 예제 코드를 소개해드리겠습니다.

**application.yml**

```properties
spring:
  redis:
    host: 127.0.0.1
    port: 6379
```

<br>

**DistributeLock.java**

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.util.concurrent.TimeUnit;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributeLock {
    String key(); // (1)

    TimeUnit timeUnit() default TimeUnit.SECONDS; // (2)

    long waitTime() default 5L; // (3)

    long leaseTime() default 3L; // (4)
}
```

`DistributeLock anntation` 입니다. key는 분산락의 락을 설정할 이름입니다.  
그렇기에 key는 어노테이션의 필수값으로 받고 있습니다.  
나머지 파라미터에 대해서는 클라이언트가 직접 선언해서 사용할 수 있게끔 작성했습니다.

(1) key: 락의 이름  
(2) timeUnit: 시간 단위(MILLISECONDS, SECONDS, MINUTE..)  
(3) waitTime: 락을 획득하기 위한 대기 시간  
(4) leaseTime: 락을 임대하는 시간

<br>

**RedissonConfig.java**

```java
import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RedissonConfig {
    @Value("${spring.redis.host}")
    private String redisHost;

    @Value("${spring.redis.port}")
    private int redisPort;

    private static final String REDISSON_HOST_PREFIX = "redis://";

    @Bean
    public RedissonClient redissonClient() {
        RedissonClient redisson = null;
        Config config = new Config();
        config.useSingleServer().setAddress(REDISSON_HOST_PREFIX + redisHost + ":" + redisPort);
        redisson = Redisson.create(config);
        return redisson;
    }
}
```

RedissonClient를 사용하기 위해 bean으로 등록합니다.

<br>

#### **reference**

https://hyperconnect.github.io/2019/11/15/redis-distributed-lock-1.html
