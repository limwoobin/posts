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
그래서 멀티 인스턴스 환경에서도 공통된 락을 사용할 수 있는 분산 락을 고려하게 되었습니다.  

## 2. Redis의 Redisson 라이브러리 선정 이유

분산락 구현은 `Redis, Mysql, Zookeeper` 등을 이용하면 모두 구현할 수 있습니다.  
그 중 Redis를 선택한 이유는 우선적으로 팀에서 해당 기술스택을 사용중이어서 추가 인프라 구축이 필요 없다는 점이 컸습니다.  
Mysql도 사용중이었지만 락을 사용하기 위해 별도의 커넥션 풀을 관리해야 하고 락에 관련된 부하를 RDS에서 받는다는 점에서 Redis를 사용하는것이 더 효율적이라 생각되었습니다.  

`Redisson` 은 일반적으로 쓰이는 `Lettuce` 와 비교했을때 여러 차이가 있습니다.  그중 `Redisson`을 선택한 가장 큰 이유는 다음과 같습니다.

### 1. 락 획득 방식

`Lettuce`는 락 획득시 `setnx` 명령어를 이용해 지속적으로 Redis에게 요청을 보내는 스핀락 방식으로 동작합니다. 요청이 많을 수록 Redis는 큰 부하를 받게됩니다.  
이에 비해 `Redisson`은 Pub/Sub 방식을 이용하기에 락이 해제되면 락을 subscribe하는 클라이언트는 락이 해제되었다는 신호를 받고 락 획득을 시도하게 됩니다. `Lettuce`에 비해 Redis에 가하는 부하가 훨씬 덜어졌습니다.

### 2. 타임아웃 지정

`Lettuce`는 락 획득시 타임아웃을 지정할 수 없습니다. 만약 락 획득 이후 모종의 이유로 예외가 발생해 락을 해제하지 못하는 경우 다른 클라이언트들은 무한정 락 획득을 기다리는 `deadlock` 상태로 빠지게 됩니다.  
그렇게 되면 무한정으로 Redis에게 락 획득 요청을 보내게 되어 서비스가 다운되는 이슈로까지 번질 수 있습니다.  
이에 비해 `Redisson`은 명시적으로 락의 leaseTime을 지정할 수 있습니다. 어플리케이션에서 락을 해제하지 못하는 이슈가 발생하더라도 leaseTime이 지나면 락은 사라지기때문에 락을 subscribe하는 클라이언트는 락을 획득할 수 있습니다.

이외에도 많은 차이가 있지만 자세한 내용이 궁금하시다면 아래 링크를 참고해보시길 추천드립니다.
#### https://www.baeldung.com/redis-redisson


## 3. 분산락을 보다 손쉽게 사용할 수는 없을까?

분산락을 도입하며 보다 손쉽고 효율적으로 사용할 수 없을까? 라는 고민을 시작으로 몇가지 규칙을 만들었습니다.  

1. 분산락 처리 로직은 비즈니스 로직이 오염되지 않게 분리해서 사용한다.
2. waitTime, leaseTime을 커스텀하게 지정 가능하다.
3. 락의 name에 대해 사용자로부터 커스텀하게 받아 처리한다.
4. 추가 요구사항에 대해서 공통으로 관리한다.

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
/*
 * RedissonClient Configuration
 */
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
                rLock.unlock();   // (4)
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

1) 락의 이름으로 RLock 인스턴스를 가져온다.
2) 정의된 waitTime까지 획득을 시도한다, 정의된 leaseTime이 지나면 잠금을 해제한다.
3) DistributedLock 어노테이션이 선언된 메소드를 별도의 트랜잭션으로 실행한다.
4) 어느 상황이던 종료시 락을 해제한다.

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

`Spring Expression Language`를 사용하면 다음과 같이 Lock의 이름을 보다 자유롭게 전달할 수 있습니다.


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

동시성 예제로 자주 등장하는 재고 차감을 예로 들어보겠습니다.  
A라는 상품의 재고가 100개 존재합니다. 여러명의 작업자들이 동시에 해당 재고를 사용한다고 가정해보겠습니다.  
이 때, 락의 해제시점이 트랜잭션 커밋시점보다 빠르면 어떻게 동작할까요?


## 4. 테스트 시나리오를 검증해보자 

분산락은 다양한 경우에 쓰일 수 있습니다.  
쿠폰 차감과 같이 한정된 수량을 제한하는 경우에도 쓰일 수 있고 동시에 들어오는 중복된 데이터를 거르는 용도로도 사용할 수 있습니다.  
위 두가지 케이스에 대해 테스트 코드로 검증해보겠습니다.

#### 1. 쿠폰 차감 테스트 시나리오
`KURLY_001` 라는 쿠폰 100개를 고객들에게 이벤트로 발급한다고 가정해보겠습니다. 이때 100명의 고객들이 쿠폰을 받기 위해 쿠폰 발급 요청을 하게 됩니다.  

__Coupon.java__
```java
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Coupon {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    /*
     * 사용 가능한 재고수량
     */
    private Long availableStock;

    public Coupon(String name, Long availableStock) {
        this.name = name;
        this.availableStock = availableStock;
    }

    public void decrease() {
        validateStockCount();
        this.availableStock -= 1;
    }

    private void validateStockCount() {
        if (availableStock < 1) {
            throw new IllegalArgumentException();
        }
    }
}
```

__CouponDecreaseService.java__
```java
@Component
@RequiredArgsConstructor
public class CouponDecreaseService {
    private final CouponRepository couponRepository;

    @Transactional
    public void couponDecrease(Long couponId) {
        Coupon coupon = couponRepository.findById(couponId)
                .orElseThrow(IllegalArgumentException::new);

        coupon.decrease();
    }

    @DistributedLock(key = "#lockName")
    public void couponDecrease(String lockName, Long couponId) {
        Coupon coupon = couponRepository.findById(couponId)
                .orElseThrow(IllegalArgumentException::new);

        coupon.decrease();
    }
}
```
분산락이 있는 경우와 없는 경우 어떻게 동작하는지 확인해보기 위해 두개의 메소드를 선언했습니다.

테스트 코드로 다음 예제를 검증해보겠습니다.

__CouponDecreaseLockTest.java__
```java
@BeforeEach
void setUp() {
    coupon = new Coupon("KURLY_001", 100L);
    couponRepository.save(coupon);
}

/**
 * Feature: 쿠폰 차감 동시성 테스트
 * Background
 *     Given KURLY_001 라는 이름의 쿠폰 100장이 등록되어 있음
 * <p>
 * Scenario: 50장의 쿠폰을 100명의 사용자가 발급 요청함
 * <p>
 * Then 쿠폰 개수보다 많은 사용자들의 요청이 오더라도 쿠폰 개수는 0미만으로 차감되지 않아야 함
 */
@Test
void 쿠폰차감_분산락_적용_동시성100명_테스트() throws InterruptedException {
    int numberOfThreads = 100;
    ExecutorService executorService = Executors.newFixedThreadPool(32);
    CountDownLatch latch = new CountDownLatch(numberOfThreads);

    for (int i = 0; i < numberOfThreads; i++) {
        executorService.submit(() -> {
            try {
                // 분산락 적용 메소드 호출 (락의 key는 쿠폰의 name으로 설정)
                couponDecreaseService.couponDecrease(coupon.getName(), coupon.getId());
            } finally {
                latch.countDown();
            }
        });
    }

    latch.await();

    Coupon persistCoupon = couponRepository.findById(coupon.getId())
            .orElseThrow(IllegalArgumentException::new);

    assertThat(persistCoupon.getAvailableStock()).isZero();
    System.out.println("잔여 쿠폰 갯수 = " + persistCoupon.getAvailableStock());
}
```

![distributed-lock-image1](https://user-images.githubusercontent.com/28802545/227767148-9ffe9f23-ad3c-42a6-a2d2-fb0ab27f079f.png)

100장의 쿠폰에 대해 100명이 동시에 요청한 경우 정확하게 쿠폰이 100명 모두에게 발급된 것을 확인할 수 있습니다.  
만약 100명이 아닌 그 이상의 사용자가 발급을 요청하더라도 `validateStockCount` 에 의해서 발급에 실패하게 되겟죠??

그럼 분산락이 적용되지 않는 버전의 테스트 코드를 호출해보겠습니다.

__CouponDecreaseLockTest.java__
```java
@Test
void 쿠폰차감_분산락_미적용_동시성100명_테스트() throws InterruptedException {
    int numberOfThreads = 100;
    ExecutorService executorService = Executors.newFixedThreadPool(numberOfThreads);
    CountDownLatch latch = new CountDownLatch(numberOfThreads);

    for (int i = 0; i < numberOfThreads; i++) {
        executorService.submit(() -> {
            try {
                // 분산락 미적용 메소드 호출
                couponDecreaseService.couponDecrease(coupon.getId());
            } finally {
                latch.countDown();
            }
        });
    }

    latch.await();

    Coupon persistCoupon = couponRepository.findById(coupon.getId())
            .orElseThrow(IllegalArgumentException::new);

    assertThat(persistCoupon.getAvailableStock()).isZero();
    System.out.println("잔여 쿠폰 갯수 = " + persistCoupon.getAvailableStock());
}
```

![distributed-lock-image2](https://user-images.githubusercontent.com/28802545/227767760-c617af31-0550-4953-87b8-acc882f073b2.png)

100명이 동시에 발급을 요청했지만 100개의 쿠폰중 남은 쿠폰은 79개입니다.  
락이 없다보니 동시에 요청이 왔을때 각자 읽은 쿠폰의 잔여갯수가 다르기에 결국 데이터의 정합성이 깨져버렸습니다.

#### 2. 중복된 발주데이터 동시 수신
`KURLY_001` 라는 발주 데이터 10개가 서비스에 중복으로 수신되었다고 가정해보겠습니다.  
시스템 상 중복발주는 허용하지 않습니다.

__Purchase.java__
```java
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Purchase {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String code;

    public Purchase(String code) {
        this.code = code;
    }
}
```

__PurchaseRegisterService.java__
```java
@Service
@RequiredArgsConstructor
public class PurchaseRegisterService {
    private final PurchaseRepository purchaseRepository;

    @DistributedLock(key = "#lockName")
    public void register(String lockName, String code) {
        boolean existsPurchase = purchaseRepository.existsByCode(code);
        if (existsPurchase) {
            throw new IllegalArgumentException();
        }

        Purchase purchase = new Purchase(code);
        purchaseRepository.save(purchase);
    }

    @Transactional
    public void register(String code) {
        boolean existsPurchase = purchaseRepository.existsByCode(code);
        if (existsPurchase) {
            throw new IllegalArgumentException();
        }

        Purchase purchase = new Purchase(code);
        purchaseRepository.save(purchase);
    }
}
```
발주 등록시 중복발주에 대한 `validation logic`을 수행합니다.  
그리고 분산락이 있는 경우와 없는 경우 어떻게 동작하는지 확인해보기 위해 두개의 메소드를 선언했습니다.  

__PurchaseRegisterLockTest.java__
```java
/**
 * Feature: 발주 등록 동시성 테스트
 * <p>
 * Scenario: KURLY_001 라는 이름의 발주 10개가 동시에 등록 요청됨
 * <p>
 * Then 중복된 발주 10개가 동시에 들어오더라도 한 건만 정상 등록 되어야 함
 */
@Test
void 발주등록_분산락_적용_테스트() throws InterruptedException {
    String 발주_코드 = "KURLY_001";

    int numberOfThreads = 10;
    ExecutorService executorService = Executors.newFixedThreadPool(numberOfThreads);
    CountDownLatch latch = new CountDownLatch(numberOfThreads);

    for (int i = 0; i < numberOfThreads; i++) {
        executorService.submit(() -> {
            try {
                // 분산락 적용 메소드 호출
                purchaseRegisterService.register(발주_코드, 발주_코드);
            } finally {
                latch.countDown();
            }
        });
    }

    latch.await();

    Long totalCount = purchaseRepository.countByCode(발주_코드);

    System.out.println("등록된 발주 = " + totalCount);
    assertThat(totalCount).isOne();
}
```

![distributed-lock-image3](https://user-images.githubusercontent.com/28802545/227771387-addd7032-0d5c-4868-a898-742ee74c92c3.png)

분산락이 적용된 버전의 메소드에 대한 테스트 결과입니다.
10건의 요청이 들어와도 정상적으로 한 건의 발주만 등록된 것을 확인할 수 있습니다.

그럼 분산락 미적용 버전의 메소드를 확인해보겠습니다.

```java
@Test
void 발주등록_분산락_미적용_테스트() throws InterruptedException {
    String 발주_코드 = "KURLY_001";

    int numberOfThreads = 10;
    ExecutorService executorService = Executors.newFixedThreadPool(numberOfThreads);
    CountDownLatch latch = new CountDownLatch(numberOfThreads);

    for (int i = 0; i < numberOfThreads; i++) {
        executorService.submit(() -> {
            try {
                // 분산락 미적용 메소드 호출
                purchaseRegisterService.register(발주_코드);
            } finally {
                latch.countDown();
            }
        });
    }

    latch.await();

    Long totalCount = purchaseRepository.countByCode(발주_코드);

    System.out.println("등록된 발주 = " + totalCount);
    assertThat(totalCount).isOne();
}
```

![distributed-lock-image4](https://user-images.githubusercontent.com/28802545/227771505-e7fdbcc5-413b-470e-94a5-ccc43ecb5e9b.png)

메소드에 작성된 `validation logic`에 예외가 걸리지 않고 모두 등록되었습니다.(등록 개수는 테스트마다 그리고 테스트 환경의 connection pool size에 따라 다를 수 있습니다)  
이렇게 분산락의 유무에 따라 시스템이 어떻게 동작하는지 테스트 코드로 검증해보았습니다.

## 5. 마치며

여기까지 입고서비스팀에서 동시성 환경에서 분산락 컴포넌트를 사용하는 방법에 대해 소개해드렸습니다.
이렇게 한 층 더 수준높은 락 처리를 할 수 있게 되었고, 분산락 사용에 대해 생산성도 올라가고 핵심 로직과도 분리해 사용할 수 있어 가독성 측면에서도 훨씬 수월하게 사용할 수 있습니다.

지금까지 읽어주셔서 감사합니다~

### Reference
https://www.baeldung.com/redis-redisson