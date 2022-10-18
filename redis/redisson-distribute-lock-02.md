[**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/lock-example)

<br>

# **Redisson 라이브러리를 이용한 Distribute Lock 동시성 처리 (2)**

이번엔 앞에서 만들어놓은 `@DistributeLock` 어노테이션을 이용해 동시성을 처리하는 예제코드, 테스트 코드를 작성해보겠습니다.  
동시성에 대한 테스트 코드는 멀티스레드를 이용해 작성 하겠습니다.

앞선 글을 아직 읽지 안았다면 **Redisson 라이브러리를 이용한 Distribute Lock 동시성 처리 (1)** 을 먼저 보고오시는것을 추천드립니다.  
https://devoong2.tistory.com/entry/Spring-Redisson-%EB%9D%BC%EC%9D%B4%EB%B8%8C%EB%9F%AC%EB%A6%AC%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-Distribute-Lock-%EB%8F%99%EC%8B%9C%EC%84%B1-%EC%B2%98%EB%A6%AC-1

<br>

## **Case1 - 쿠폰 차감 서비스**

여러명의 사용자가 쿠폰을 동시에 발급받으면 요청한 사용자 수 만큼  
쿠폰을 차감하는 기능을 동시성을 고려한 예제코드로 보여드리겠습니다.

<br>

**Coupon.java**

```java
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Getter
public class Coupon {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private Long availableStock;

    public Coupon(String name, Long availableStock) {
        this.name = name;
        this.availableStock = availableStock;
    }

    public static Coupon of(String name, Long availableStock) {
        return new Coupon(name, availableStock);
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

<br>

**CouponRepository.java**

```java
import org.springframework.data.jpa.repository.JpaRepository;

public interface CouponRepository extends JpaRepository<Coupon, Long> {
}
```

<br>

**CouponService.java**

```java
import com.example.lockexample.redisson.dto.CouponRequest;
import com.example.lockexample.redisson.dto.CouponResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class CouponService {
    private final CouponDecreaseService couponDecreaseService;

    private static final String COUPON_KEY_PREFIX = "COUPON_";

    public void decrease(Long couponId) {
        String key = COUPON_KEY_PREFIX + couponId;
        couponDecreaseService.couponDecrease(key, couponId);
    }
}
```

<br>

**CouponDecreaseService.java**

```java
import com.example.lockexample.redisson.aop.DistributeLock;
import com.example.lockexample.redisson.domain.Coupon;
import com.example.lockexample.redisson.domain.CouponRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
public class CouponDecreaseService {
    private final CouponRepository couponRepository;

    @DistributeLock(key = "#key")
    public void couponDecrease(String key, Long couponId) {
        Coupon coupon = couponRepository.findById(couponId)
                .orElseThrow(IllegalArgumentException::new);

        coupon.decrease();
    }
}
```

다음은 쿠폰 차감에 대한 예제 코드입니다.  
`CouponService`에서 `CouponDecreaseService`의 `decrease` 메소드에게  
lock 을 잡기 위한 key를 전달해 `@Distribute` 어노테이션에서 해당 락을 잡게 됩니다.

<br>

이제 테스트 코드를 이용해 코드를 검증해보겠습니다.

Redis 테스트 환경은 `TestContainer`를 이용하겠습니다.  
`TestContainer`가 뭔지 잘 모르신다면 아래의 포스팅을 참고해주세요.  
https://devoong2.tistory.com/entry/Springboot-Redis-%ED%85%8C%EC%8A%A4%ED%8A%B8-%ED%99%98%EA%B2%BD-%EA%B5%AC%EC%B6%95%ED%95%98%EA%B8%B0-Embedded-Redis-TestContainer

<br>

**CouponDecreaseLockTest.java**

```java
import com.example.lockexample.redisson.application.CouponService;
import com.example.lockexample.redisson.domain.Coupon;
import com.example.lockexample.redisson.domain.CouponRepository;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.concurrent.*;

import static org.assertj.core.api.Assertions.assertThat;

@DisplayName("Redisson Lock 쿠폰 차감 테스트")
@SpringBootTest
class CouponDecreaseLockTest {

    @Autowired
    private CouponService couponService;

    @Autowired
    private CouponRepository couponRepository;

    private Coupon coupon;

    @BeforeEach
    void setUp() {
        coupon = new Coupon("C0001", 100L);
        couponRepository.save(coupon);
    }

    @AfterEach
    void teardown() {
        couponRepository.deleteAll();
    }

    @Test
    void 쿠폰차감_동시성100명_테스트() throws InterruptedException {
        int numberOfThreads = 100;
        ExecutorService executorService = Executors.newFixedThreadPool(32);
        CountDownLatch latch = new CountDownLatch(numberOfThreads);

        for (int i=0; i<numberOfThreads; i++) {
            executorService.submit(() -> {
                try {
                    couponService.decrease(coupon.getId());
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await();

        Coupon persistCoupon = couponRepository.findById(coupon.getId())
                .orElseThrow(IllegalArgumentException::new);

        assertThat(persistCoupon.getAvailableStock()).isZero();
    }
}
```

해당 테스트 코드에 대한 시나리오는 다음과 같습니다.

```shell
1. 쿠폰 100개가 준비되어있다
2. 사용자 100명이 동시에 쿠폰을 발급받기 위해 쿠폰 발급을 요청한다
3. 정상적으로 남은 쿠폰 갯수가 0이 되어야 한다
```

다음과 같이 테스트 코드가 정상적으로 통과된 것을 확인할 수 있습니다.

![redisson-example-image3](https://user-images.githubusercontent.com/28802545/194810213-0bbcacd9-93cc-4769-889b-c94a416f7596.png)

<br>

### **동시성 처리가 없다면?**

그렇다면 반대로 `@DistributeLock`을 지우고 Lock없이 동시성 로직을 수행하면 어떤 결과가 나올지 확인해보겠습니다.

![redisson-example-image4](https://user-images.githubusercontent.com/28802545/194810918-04414132-76fa-4a20-9263-7f1f98e0e95c.png)

`@DistributeLock` 을 지우고 `@Transactional`을 선언하였습니다.

![redisson-example-image5](https://user-images.githubusercontent.com/28802545/194811231-b13fde26-0854-46d0-beac-2e1fdc7e8604.png)

Lock없이 동시성 로직을 수행하니 사용자 100명이 쿠폰을 요청했지만 남은 쿠폰이 61개나 있습니다.  
(다만 이 부분은 테스트 수행시마다 약간은 달라질 수 있습니다.)  
저희는 쿠폰 차감이 제대로 되지 않은것을 확인 할 수 있습니다.

<br>

## **Case2 - 쿠폰 등록 서비스**

쿠폰 등록시 DB에는 같은 이름의 쿠폰을 등록할 수 없다는 제약이 있다고 가정하겠습니다.  
(일반적으론 DB의 Unique index를 이용할 수도 있지만 예제인 만큼 가볍게 봐주시면 감사하겠습니다.)  
해당 기능은 동시에 같은 이름의 쿠폰등록요청이 여러건 와도 단 한개만 등록되어야 합니다.  
예제코드를 작성해보겠습니다.

<br>

**CouponService.java**

```java
import com.example.lockexample.redisson.dto.CouponRequest;
import com.example.lockexample.redisson.dto.CouponResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class CouponService {
    private final CouponRegisterService couponRegisterService;

    private static final String COUPON_KEY_PREFIX = "COUPON_";

    public CouponResponse registerCoupon(CouponRequest couponRequest) {
        String key = COUPON_KEY_PREFIX + couponRequest.getName();
        return couponRegisterService.register(key, couponRequest);
    }
}
```

<br>

__CouponRegisterService.java__

```java
import com.example.lockexample.redisson.aop.DistributeLock;
import com.example.lockexample.redisson.domain.Coupon;
import com.example.lockexample.redisson.domain.CouponRepository;
import com.example.lockexample.redisson.dto.CouponRequest;
import com.example.lockexample.redisson.dto.CouponResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

@Component
@RequiredArgsConstructor
public class CouponRegisterService {

    private final CouponRepository couponRepository;

    @DistributeLock(key = "#key")
    public CouponResponse register(final String key, CouponRequest request) {
        validateAlreadyExist(request);

        Coupon coupon = request.toCoupon();
        couponRepository.save(coupon);
        return CouponResponse.toResponse(coupon);
    }

    private void validateAlreadyExist(CouponRequest request) {
        couponRepository.findByName(request.getName()).ifPresent(x -> {
            throw new IllegalArgumentException();
        });
    }
}
```

쿠폰 등록시 해당 이름의 쿠폰이 이미 등록되어있는지 유효성 검사를 진행합니다.  
그리고 쿠폰 등록시에는 쿠폰 이름, 쿠폰Prefix로 이루어진 이름을 key로 Lock을 잡아  
같은 이름의 쿠폰을 등록하려는 경우 해당 Lock을 사용해 하나의 요청만 접근하게 하여 동시성을 처리하고 있습니다.

쿠폰 등록에 대해 테스트 코드를 작성해보겠습니다.

<br>

__CouponRegisterLockTest.java__

```java
import com.example.lockexample.redisson.application.CouponService;
import com.example.lockexample.redisson.domain.CouponRepository;
import com.example.lockexample.redisson.dto.CouponRequest;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.concurrent.*;

import static org.junit.jupiter.api.Assertions.assertEquals;

@DisplayName("Redisson Lock 쿠폰 등록 테스트")
@SpringBootTest
class CouponRegisterLockTest {

    @Autowired
    private CouponService couponService;

    @Autowired
    private CouponRepository couponRepository;

    @Test
    void 같은이름의_쿠폰이_여러개_등록될수_없음() throws InterruptedException {
        CouponRequest couponRequest = new CouponRequest("NEW001", 10L);

        int numberOfThreads = 100;
        ExecutorService executorService = Executors.newFixedThreadPool(32);
        CountDownLatch latch = new CountDownLatch(numberOfThreads);

        for (int i=0; i<numberOfThreads; i++) {
            executorService.submit(() -> {
                try {
                    couponService.registerCoupon(couponRequest);
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await();

        Long totalCount = couponRepository.countByName("NEW001");
        assertThat(totalCount).isOne();
    }
}
```

해당 테스트 코드에 대한 시나리오는 다음과 같습니다.

```shell
1. "NEW001" 이라는 이름을 가진 쿠폰을 준비한다
2. 사용자 100명이 동시에 "NEW001" 쿠폰을 등록요청한다
3. 정상적으로 등록된 "NEW001" 쿠폰 갯수는 단 하나이어야 한다
```

다음과 같이 정상적으로 하나만 등록되어 테스트에 통과되는 것을 확인할 수 있습니다.

![redisson-example-image6](https://user-images.githubusercontent.com/28802545/194820444-afd24d18-bc53-43d3-a888-60eac055b65f.png)

<br>

### **동시성 처리가 없다면?**

반대로 쿠폰 등록시 `@DistributeLock`을 지우고 Lock없이 동시성 로직을 수행하면 어떻게 될지 확인해보겠습니다.

![redisson-example-image8](https://user-images.githubusercontent.com/28802545/194823214-dd48ce30-bf0c-474e-ad6e-c3e0dfee9279.png)

테스트 시나리오는 위와 동일합니다.

![redisson-example-image7](https://user-images.githubusercontent.com/28802545/194822999-a8ec7eca-b063-4ffe-94aa-84ba7f804fb1.png)

같은 이름의 쿠폰이 등록될 수 없도록 유효성 로직이 있음에도 불구하고  
**`NEW001`** 이라는 이름의 쿠폰이 32개나 등록된 것을 확인할 수 있습니다.  
(개수는 테스트마다 달라질 수 있습니다.)

<br>

이렇게 쿠폰 차감, 등록과 같이 동시성 처리가 필요한 기능들에 대해  
`Redisson Distribute Lock`을 이용해 Custom Annotation인 `@DistributeLock`을 만들어 동시성 처리를 해보았습니다.

감사합니다.

<br>

## **Reference**

https://www.inflearn.com/course/%EB%8F%99%EC%8B%9C%EC%84%B1%EC%9D%B4%EC%8A%88-%EC%9E%AC%EA%B3%A0%EC%8B%9C%EC%8A%A4%ED%85%9C
