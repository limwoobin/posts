[**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/lock-example)

<br>

# **Redisson 라이브러리를 이용한 Distribute Lock 동시성 처리 (2)**

이번엔 앞에서 만들어놓은 `@DistributeLock` 어노테이션을 이용해 동시성을 처리하는 예제코드, 테스트 코드를 작성해보겠습니다.

<br>

## **Case1 - 쿠폰 발급 서비스**

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

**RedissonLockTest.java**

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

@DisplayName("Redisson Lock 테스트")
@SpringBootTest
class RedissonLockTest {

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
1. 쿠폰 100개가 준비되어있음
2. 사용자 100명이 동시에 쿠폰을 발급받기 위해 쿠폰 발급을 요청함
3. 정상적으로 남은 쿠폰 갯수가 0이 되어야 함
```

다음과 같이 테스트 코드가 정상적으로 통과된 것을 확인할 수 있습니다.

![redisson-example-image3](https://user-images.githubusercontent.com/28802545/194810213-0bbcacd9-93cc-4769-889b-c94a416f7596.png)

그렇다면 반대로 `@DistributeLock`을 지우고 Lock없이 동시성 로직을 수행하면 어떤 결과가 나올지 확인해보겠습니다.

![redisson-example-image4](https://user-images.githubusercontent.com/28802545/194810918-04414132-76fa-4a20-9263-7f1f98e0e95c.png)

`@DistributeLock` 을 지우고 `@Transactional`을 선언하였습니다.

![redisson-example-image5](https://user-images.githubusercontent.com/28802545/194811231-b13fde26-0854-46d0-beac-2e1fdc7e8604.png)

Lock없이 동시성 로직을 수행하니 사용자 100명이 쿠폰을 요청했지만 남은 쿠폰이 61개나 있습니다.  
쿠폰 차감이 제대로 되지 않은것을 확인 할 수 있습니다.

<br>

## **Reference**

https://www.inflearn.com/course/%EB%8F%99%EC%8B%9C%EC%84%B1%EC%9D%B4%EC%8A%88-%EC%9E%AC%EA%B3%A0%EC%8B%9C%EC%8A%A4%ED%85%9C
