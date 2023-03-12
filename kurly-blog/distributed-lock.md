---
layout  : post
title   : 풀필먼트 입고 서비스팀에서 분산락을 사용하는 방법 - Spring Redisson
summary : "어노테이션 기반으로 분산락을 사용하는 방법에 대해 소개합니다."
date    : 2023-03-05 00:00:00 +0900
author  : 임우빈
github  : limwoobin
public  : true
comment : true
---
* TOC
{:toc}

## 1. 들어가며

안녕하세요. 컬리 풀필먼트 프로덕트에서 입고서비스를 개발하고 있는 임우빈입니다.  

풀필먼트 입고서비스에서는 다양한 동시성 문제들을 맞닥드리고 있는데요. 이를 해결하기 위해 시행착오를 겪었던 경험에 대해 공유드리려고 합니다.

RMS에는 여러 동시성 문제를 가지고 있었습니다.
- 카프카로 동시에 들어오는 중복된 발주를 수신하는 경우
- 검수/검품 이슈 등록시 더블 클릭, 네트워크 이슈로 인해 같은 리퀘스트가 여러개 동시에 들어오는 경우
- 이동출고시 여러 작업자가 CTA를 동시에 클릭하여 잘못된 재고 트랜잭션이 생성되는 경우

등등 이외에도 다양한 경우의 동시성 이슈가 서비스에 존재했습니다.

해당 기능에 대해 Application에서의 예외처리는 존재했지만 보다 확실하게 동시성 이슈를 처리할 방법이 필요했습니다.  
그래서 멀티 인스턴스 환경에서도 공통된 락을 사용할 수 있는 `Remote Data Source`를 고려하게 되었습니다.  

## 2. Redis의 Redisson 라이브러리를 도입한 이유

어떤 `Remote Data Source`를 사용할지에 대해 먼저 고민해보았습니다.  
우선적으로 팀에서 현재 사용중인 기술스택인 `mysql`과 `redis`를 선택지로 놓았습니다.

## 3. 분산락을 보다 손쉽게 사용할 수는 없을까?

분산락을 도입하며 보다 손쉽고 커스텀하게 사용할 수 없을까? 라는 고민을 시작으로 몇가지 규칙을 만들었습니다.  

1. 분산락 처리 로직은 비즈니스 로직이 오염되지 않게 분리해서 사용한다.
2. waitTime, leaseTime을 커스텀하게 지정 가능하다.
3. 락의 key name에 대해 사용자로부터 커스텀하게 받아 처리한다.
4. 락을 획득하지 못한 경우의 처리에 대해 커스텀하게 설정할 수 있다.

다음과 같은 규칙을 충족하기 위해 어노테이션 기반으로 AOP를 이용해 분산락 컴포넌트를 만들었습니다.  
입고서비스에서 분산락 컴포넌트를 사용하는 방법은 다음과 같습니다.

__build.gradle__
```java
dependencies {
    // redisson
    implementation 'org.redisson:redisson-spring-boot-starter:3.18.0'
}
```
Redisson 라이브러리를 사용하기 위해 의존성을 추가합니다.


__RedissonConfig.java__
```java
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

`RedissonClient`를 사용하기 위해 Config 설정을 빈으로 등록합니다.


__DistributedLock.java__
```java
/**
 * Redisson Distributed Lock annotation
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributedLock {

    /**
     * 락의 이름
     */
    String key();

    /**
     * 락의 시간 단위
     */
    TimeUnit timeUnit() default TimeUnit.SECONDS;

    /**
     * 락을 기다리는 시간 (default - 5s)
     * 락 획득을 위해 waitTime 만큼 대기한다
     */
    long waitTime() default 5L;

    /**
     * 락 임대 시간 (default - 3s)
     * 락을 획득한 이후 leaseTime 이 지나면 락을 해제한다
     */
    long leaseTime() default 3L;
}
```

`DistributedLock` 어노테이션의 파라미터는 key는 필수값, 나머지 값들은 커스텀하게 설정할 수 있도록 작성했습니다.  


__DistributedLockAop.java__
```java
/**
 * @DistributedLock 선언 시 수행되는 Aop class
 */
@Aspect
@Component
@RequiredArgsConstructor
@Sl4j
public class DistributedLockAop {
    private static final String REDISSON_LOCK_PREFIX = "LOCK:";

    private final RedissonClient redissonClient;
    private final AopForTransaction aopForTransaction;

    @Around("@annotation(com.kurly.rms.aop.DistributedLock)")
    public Object lock(final ProceedingJoinPoint joinPoint) throws Throwable {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        DistributedLock distributedLock = method.getAnnotation(DistributedLock.class);

        String key = REDISSON_LOCK_PREFIX + CustomSpringELParser.getDynamicValue(signature.getParameterNames(), joinPoint.getArgs(), distributedLock.key());
        RLock rLock = redissonClient.getLock(key);  // (1)

        try {
            boolean available = rLock.tryLock(distributedLock.waitTime(), distributedLock.leaseTime(), distributedLock.timeUnit());  // (2)
            if (!available) {
                return false;
            }

            return aopForTransaction.proceed(joinPoint);  // (3)
        } catch (InterruptedException e) {
            throw new InterruptedException();
        } finally {
            try {
                rLock.unlock();
            } catch (IllegalMonitorStateException e) {
                log.info("Redisson Lock Already UnLock {} {}",
                        kv("serviceName", method.getName()),
                        kv("key", key)
                );
            }
        }
    }
}
```

다음은 `@DistributedLock` 어노테이션 선언시 수행되는 aop 클래스입니다.  
`@DistributedLock` 어노테이션의 파라미터 값을 가져와 분산락 획득 시도 그리고 어노테이션이 선언된 메소드를 실행합니다.

1) asd 
2) asd
3) asd

여기서 주의해서 볼 부분은 `CustomSpringELParser` 와 `AopForTransaction` 클래스입니다.
이 클래스들은 분산락 컴포넌트에서 어떤 역할을 맡고 있을까요??


__CustomSpringELParser.java__
```java
/**
 * Spring Expression Language Parser
 */
public class CustomSpringELParser {
    private CustomSpringELParser() {
    }

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

`CustomSpringELParser` 는 전달받은 Lock의 이름을 `Spring Expression Language` 로 파싱하여 읽어옵니다.  

```java
// (1)
@DistributedLock(key = "#lockName")
public void shipment(String lockName) {
    ...
}

// (2)
@DistributedLock(key = "#model.getName().concat('-').concat(#model.getShipmentOrderNumber())")
public void shipment(ShipmentModel model) {
    ...
}


ShipmentModel.java
public class ShipmentModel {
    private String name;
    private String shipmentNumber;

    public String getName() {
        return name;
    }

    public String getShipmentNumber() {
        return shipmentNumber;
    }

    ...
}
```

`SpringEL`을 사용하면 다음과 같이 Lock의 이름을 보다 자유롭게 전달할 수 있습니다.


__AopForTransaction.java__
```java
/**
 * AOP에서 트랜잭션 분리를 위한 클래스
 */
@Component
public class AopForTransaction {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public Object proceed(final ProceedingJoinPoint joinPoint) throws Throwable {
        return joinPoint.proceed();
    }
}
```

`@DistributedLock` 이 선언된 메소드는 `Propagation.REQUIRES_NEW` 옵션을 주어 부모 트랜잭션의 유무에 관계없이 별도의 트랜잭션으로 동작하게끔 설정했습니다.  
그리고 반드시 트랜잭션 커밋 이후 락이 해제되게끔 처리했습니다.  

왜 트랜잭션이 커밋되고 난 이후 락이 해제되어야 할까요?? 
> 바로 동시성 환경에서 데이터의 정합성을 보장하기 위해서 입니다.


다음과 같은 예시를 들어보겠습니다.  
A지번의 재고를 B지번으로 옮기기 위해 우선 A지번의 재고를 차감한다고 가정해보겠습니다.

- A지번에는 10개의 재고가 존재한다.
- 재고를 옮기는 사용자는 여러명이다.


## 4. 테스트 시나리오를 검증해보자 

테스트 시나리오1
재고를 이동하기 위해 차감

테스트 시나리오2
동일한 발주코드를 n건 이상 동시에 수신해도 한개만 등록