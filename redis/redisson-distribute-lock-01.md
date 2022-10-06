[**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/lock-example)

<br>

# **[Spring] Redisson 라이브러리를 이용한 Distribute Lock 동시성 처리(1)**

Redis를 통한 분산락을 이용해 동시성을 해결하는 방법에 대해 알아보고 이를 적용한 방법에 대해 예제코드를 함께 공유드리려 합니다.

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

Redisson은 Lettuce에 비해 다음과 같은 장점이 있습니다.

```shell
1. Lock에 타임아웃을 지정할 수 있음
- Redisson은 락 획득시도시 타임아웃을 명시하게 되어있습니다.
그래서 무한정 대기상태로 빠질 수 있는 위험이 없습니다.
2. pub/sub방식을 사용하므로 스핀락을 사용하지 않음
- 락이 해제되면 락을 subscribe하는 클라이언트들에게 락이 해제되었다는 신호를 보내게 됩니다.
그렇기에 락을 subscribe하는 클라이언트들은 더 이상 락을 획득해도 되냐고 redis로 요청을 보내지 않습니다.
```

<hr>

### **Redisson 라이브러리**


Spring에서 Redisson을 사용하기 위해선 아래의 의존성이 필요합니다.  
그리고 Redisson에서 제공하는 인터페이스와 사용법을 간략하게 소개하겠습니다.

__build.gradle__

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
(1): key 이름에 해당하는 RLock 인스턴스를 가져온다
(2): Lock 획득을 시도한다 (성공: true / 실패: false)
(3): 획득 실패시 Lock을 subscribe 하며 해제되길 기다린다
(4): finally에서 Lock을 해제한다
```

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

**DistributeLockAop.java**

```java
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;

@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class DistributeLockAop {
    private static final String REDISSON_KEY_PREFIX = "RLOCK_";

    private final RedissonClient redissonClient;
    private final AopForTransaction aopForTransaction;


    @Around("@annotation(com.example.lockexample.redisson.aop.DistributeLock)")
    public Object lock(final ProceedingJoinPoint joinPoint) throws Throwable {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        DistributeLock distributeLock = method.getAnnotation(DistributeLock.class);	 // (1)

        String key = REDISSON_KEY_PREFIX + CustomSpringELParser.getDynamicValue(signature.getParameterNames(), joinPoint.getArgs(), distributeLock.key());	// (2)

        RLock rLock = redissonClient.getLock(key);	// (3)

        try {
            boolean available = rLock.tryLock(distributeLock.waitTime(), distributeLock.leaseTime(), distributeLock.timeUnit());	// (4)
            if (!available) {
                return false;
            }

            log.info("get lock success {}" , key);
            return aopForTransaction.proceed(joinPoint);	// (5)
        } catch (Exception e) {
            Thread.currentThread().interrupt();
            throw new InterruptedException();
        } finally {
            rLock.unlock();	// (6)
        }
    }
}
```

`@DistributeLock`을 선언한 메소드를 호출했을때 실행되는 aop 클래스입니다.

(1) : `@DistributeLock annotation`을 가져옴  
(2) : `@DistributeLock`에 전달한 key를 가져오기 위해 `SpringEL` 표현식을 파싱  
(3) : Redisson에 해당 락의 RLock 인터페이스를 가져옴  
(4) : `tryLock method`를 이용해 Lock 획득을 시도 (획득 실패시 Lock이 해제 될 때까지 subscribe)  
(5) : `@DistributeLock`이 선언된 메소드의 로직 수행(별도 트랜잭션으로 분리)  
(6) : 종료 혹은 예외 발생시 finally에서 Lock을 해제함

여기서 주의깊게 볼 부분은 **(5)** 의 `aopForTransaction.proceed(joinPoint);` 입니다.  
락을 획득/해제는 트랜잭션의 단위보다 크게 이루어져야 합니다.  
즉, 동시성 처리를 하기 위해서는 락을 획득 이후 트랜잭션이 시작되어야 하고 트랜잭션이 커밋되고 난 이후 락이 해제되어야 합니다.

<br>

**다음 예제를 보겠습니다.**

![redisson-example-image1](https://user-images.githubusercontent.com/28802545/193533591-e5367c6e-7def-44b2-bb0a-4bfa20318961.png)

사용자1, 2가 동시에 재고가100인 상품을 차감하려고 시도합니다.  
사용자1은 트랜잭션이 커밋되기전 Lock을 해제합니다. 사용자2는 해당 로직에 접근하여 커밋되기전의 재고수량인 100을 읽게 됩니다.  
사용자1, 사용자2가 서로 한번씩 재고차감을 시도했지만 재고는 정상적으로 2개가 차감되지 않고 1개만 차감된 상태로 저장됩니다.

그럼 Lock의 해제를 트랜잭션 커밋 이후로 변경해서 보겠습니다.

![redisson-example-image2](https://user-images.githubusercontent.com/28802545/193536783-dc94ab98-b82b-4f84-b6ef-128265bdd31f.png)

사용자1이 재고차감을 완료한 이후 Lock을 해제했음으로 사용자2는 재고 차감시 커밋된 수량 99를 읽고 재고차감을 시도합니다.  
그렇게되면 재고는 정상적으로 차감되게 됩니다.

<br>

**AopForTransaction.java**

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Component
public class AopForTransaction {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public Object proceed(final ProceedingJoinPoint joinPoint) throws Throwable {
        return joinPoint.proceed();
    }
}
```

`@Transactional`은 프록시 기반으로 동작하기에 Aop내에서 트랜잭션을 별도로 가져가기 위해 클래스를 분리했습니다.  
이 클래스는 `@DistributeLock`가 선언된 메소드의 로직을 수행합니다.  
부모트랜잭션의 유무와 관계없이 동시성에 대한 처리는 별도의 트랜잭션으로 동작해야 하기에 `@Transactional`의 전파옵션은 `propagation = Propagation.REQUIRES_NEW`로 선언했습니다.

하지만 부모트랜잭션 내에서 `@Transactional(propagation = Propagation.REQUIRES_NEW)`을 이용해 전파옵션을 따로 가져가는것을 추천드리지는 않습니다.  
모든 가용할 수 있는 `connection pool`이 해당 로직으로 접근하게 된다면 `connection pool dead lock`이 발생할 여지가 있습니다.  
새로운 트랜잭션을 얻어 이후 로직을 수행하기 때문에 가용 가능한 `connection pool` 이 없다면 모든 스레드들은 반한될 `connection pool`을 기다리게 됩니다.  
스레드들이 새로운 트랜잭션을 얻으려 대기하기 때문에 반환가능한 트랜잭션이 없어 `connection pool dead lock` 이 발생하게 됩니다.  
그렇기에 facade와 같이 객체를 감싸 트랜잭션을 짧은 단위로 가져가거나 해당 서비스의 트래픽에 알맞는 `connection pool size`를 설정하는것이 필요합니다.

<br>

**CustomSpringELParser.java**

```java
import org.springframework.expression.ExpressionParser;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;

public class CustomSpringELParser {
    public static Object getDynamicValue(String[] parameterNames, Object[] args, String key) {
        ExpressionParser parser = new SpelExpressionParser();
        StandardEvaluationContext context = new StandardEvaluationContext();

        for (int i = 0; i < parameterNames.length; i++) {
            context.setVariable(parameterNames[i], args[i]);
        }

        return parser.parseExpression(key).getValue(context, Object.class);
    }
}
```

`@DistributeLock` 사용시 key를 SpringExpression으로 전달하고 이를 파싱하는 util클래스입니다.

```java
@DistributeLock(key = "#key")
public void doAnything(final String key) {
	// ...
}
```

Aop클래스인 `DistributeLockAop.java`에서 해당 key를 전달받아 사용할 수 있습니다.  
다음 글에서는 Redisson 분산락을 이용해 실제 서비스에서 겪을 수 있는 동시성 문제에 대해 해결하고 테스트 코드로 검증하는 과정을 공유해드리겠습니다.  

감사합니다.


<br>

## **Reference**

https://hyperconnect.github.io/2019/11/15/redis-distributed-lock-1.html
https://github.com/redisson/redisson