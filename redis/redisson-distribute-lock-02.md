[**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/lock-example)

<br>

# **Redisson 라이브러리를 이용한 Distribute Lock 동시성 처리 (2)**

이번엔 앞에서 만들어놓은 `@DistributeLock` 어노테이션을 이용해 동시성을 처리하는 예제코드, 테스트 코드를 작성해보겠습니다.

<br>

## **Case1 - 쿠폰 발급 서비스**

```shell
테스트 시나리오는 다음과 같습니다.
1. 쿠폰 100개가 준비되어있음
2. 사용자 100명이 동시에 쿠폰을 발급받기 위해 쿠폰 발급을 요청함
3. 정상적으로 남은 쿠폰 갯수가 0이 되어야 함
```

## **쿠폰 발급 얘제코드**

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
이제 테스트 코드를 이용해 코드를 검증해보겠습니다.

<br>

## **Reference**

https://www.inflearn.com/course/%EB%8F%99%EC%8B%9C%EC%84%B1%EC%9D%B4%EC%8A%88-%EC%9E%AC%EA%B3%A0%EC%8B%9C%EC%8A%A4%ED%85%9C
