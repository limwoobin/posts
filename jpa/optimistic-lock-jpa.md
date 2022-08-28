#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/lock-example)

<br />

# **낙관적 락과 비관적 락의 차이점**

이번엔 **낙관적 락(Optimistic Lock)** 을 이용해 동시성 처리를 하는 방법에 대해 알아보려 합니다.

그전에 **낙관적 락(Optimistic Lock)** 과 **비관적 락(Pessimistic Lock)** 의 간략한 차이점에 대해 먼저 설명드리겠습니다

<br />

### **낙관적 락(Optimistic Lock)**

충돌이 발생하지 않을 것이라 가정하고 Lock을 거는 방식

- 트랜잭션을 commit 하는 시점에 충돌을 알 수 있음
- DB Level 에서 동시성을 처리하는것이 아닌 Application Level 에서 처리

<br />

### **비관적 락(Pessimistic Lock)**

충돌이 발생할것이라 가정하고 우선 DB에 Lock을 거는 방식 (`select for update`)

- 데이터를 수정하는 즉시 충돌을 알 수 있음
- DB Level 동시성을 처리

<br />

# **낙관적 락(Optimistic Lock) 이란?**

JPA에서의 낙관적 락을 처리하는 방법은 **@Version Annotation** 을 이용해 처리할 수 있습니다.  
이 **@Version**은 버전 관리용 필드를 추가해 트랜잭션 내에서 처음 조회되었을때의 버전과 이후 수정 후 커밋될때의 버전을 비교합니다.

## **@Version Annotation**

JPA에서 version 속성을 정의할때 지켜야하는 몇가지 규칙이 있습니다.

- 각 Entity Class에는 @Version 속성이 하나만 있어야 한다
- 여러 테이블에 매핑된 Entity의 경우 기본 테이블에 배치되어야 한다
- 버전에 타입은 `int , Integer , long , Long , short , Short , java.sql.Timestamp` 중 하나여야 한다

이 field 의 **값 혹은 시간**이 처음 조회될 때의 버전과 commit될때의 버전이 서로 다르다면 이는 충돌이 발생한 것으로 판단하고 예외를 발생시킵니다.

재고를 차감하는 예를 들어보겠습니다.  
치킨A라는 재고는 현재 단 한개가 남아 있습니다.

```shell
[transaction-1] : 치킨A의 재고를 확인 / 치킨A 재고: 1개, version: 1
[transaction-2] : 치킨A의 재고를 확인 / 치킨A 재고: 1개, version: 1

-- 이때 두 트랜잭션 중 transaction-1 가 먼저 완료되었다고 가정해보겠습니다.

[transaction-1] : 치킨A를 구매 / 치킨A 재고: 0개, version: 2 로 업데이트하고 커밋
[transaction-2] : 치킨A를 구매 / 치킨A 재고: 0개, version: 2 로 업데이트하고 커밋하려는데 version이 처음 조회했던 1이 아니라 [transaction-1]에서 2로 변경되어 현재 조회한 버전과 다르므로 업데이트 실패
```

```sql
update stock
set
	availableStock = ?,
	version = 2
where
	id = ?
	and version = 1
```

위와 같은 쿼리가 발생하지만 해당 재고의 version은 `transaction-1` 으로 인해 이미 2로 증가된 상태입니다. 이때 처음 조회했던 version값인 1을 전달하게 되니 업데이트할 대상을 찾지 못해 예외가 발생합니다.

## **낙관적 락에서의 예외 종류**

- `javax.persistence.OptimisticLockException (JPA)`
- `org.hibernate.StaleObjectStateException (Hibernate)`
- `org.springframework.orm.ObjectOptimisticLockingFailureException (Spring)`

Spring 기반의 JPA에서 낙관적락을 사용하게 되면 충돌시 Hibernate에서 `StaleStateException` 을 발생시킵니다. 그리고 Spring에서 이 에외를 `OptimisticLockingFailureException` 로 감싸서 응답하게 됩니다. 그래서 OptimisticLockingFailureException을 예외로 잡아 충돌이 발생했는지 알 수 있습니다.

![optimistic-lock](https://user-images.githubusercontent.com/28802545/187056391-533f3dfe-d29a-48ab-aa1a-164834eeaee6.png)

위 이미지와 같이 예외로 `OptimisticLockingFailureException`을 확인할 수 있습니다.  
그리고 예외의 원인항목인 cause을 살펴보면 `StaleStateException`을 확인할 수 있습니다.

그렇다면 이 과정을 코드예제로 한번 보겠습니다.

# **코드 예제**

Stock.java

```java
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Version;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;

@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Getter
public class Stock {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private Long availableStock;

    @Version
    private Long version;

    public Stock(String name, Long availableStock) {
        this.name = name;
        this.availableStock = availableStock;
    }

    public static Stock createStock(String name, Long availableStock) {
        return new Stock(name, availableStock);
    }

    public void decrease(Long pickingCount) {
        validateStockCount(pickingCount);
        availableStock -= pickingCount;
    }

    private void validateStockCount(Long pickingCount) {
        if (pickingCount > availableStock) {
            throw new IllegalArgumentException();
        }
    }
}
```

StockService.java

```java
import com.example.lockexample.domain.Stock;
import com.example.lockexample.domain.StockRepository;
import com.example.lockexample.ui.StockRequest;
import com.example.lockexample.ui.StockResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class StockService {
    private final StockRepository stockRepository;

    @Transactional
    public StockResponse createStock(StockRequest stockRequest) {
        Stock stock = stockRequest.toStock();
        stockRepository.save(stock);
        return StockResponse.toResponse(stock);
    }

    @Transactional
    public void decrease(Long stockId, Long pickingCount) {
        Stock stock = stockRepository.findById(stockId)
            .orElseThrow(IllegalStateException::new);

        stock.decrease(pickingCount);
    }
}
```

StockRepository.java

```java
import org.springframework.data.jpa.repository.JpaRepository;

public interface StockRepository extends JpaRepository<Stock, Long> {

}

```

다음과 같이 Stock Entity 내의 @Version 으로 version field 를 선언해서 테스트를 해보겠습니다.  
동시성을 테스트하고 코드로 검증하기 위해서는 직접 멀티스레드를 이용한 테스트를 구현해야 합니다.

테스트 시나리오는 다음과 같습니다.

```
1. 불닭볶음면 재고를 한개 생성한다.
2. 생성된 재고에 재고1개를 차감하는 요청 세 개를 동시에 보낸다.
3. 세 개의 요청이 동시에 재고를 차감하다 버전 충돌이 발생해 OptimisticLockingFailureException을 발생한다.
```

StockOptimisticLockTest.java

```java
import static org.junit.jupiter.api.Assertions.assertTrue;

import com.example.lockexample.application.StockService;
import com.example.lockexample.domain.Stock;
import com.example.lockexample.domain.StockRepository;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.dao.OptimisticLockingFailureException;

@DisplayName("낙관적락 재고 선점 테스트")
@SpringBootTest
class StockOptimisticLockTest {

    @Autowired
    StockService stockService;

    @Autowired
    StockRepository stockRepository;

    @Test
    void 낙관적락_재고_선점_테스트() throws InterruptedException {
        Stock savedStock = 재고_1개_생성();
        int numberOfThreads = 3;

        ExecutorService executorService = Executors.newFixedThreadPool(numberOfThreads);

        Future<?> future = executorService.submit(
            () -> stockService.decrease(savedStock.getId(), 1L));
        Future<?> future2 = executorService.submit(
            () -> stockService.decrease(savedStock.getId(), 1L));
        Future<?> future3 = executorService.submit(
            () -> stockService.decrease(savedStock.getId(), 1L));

        Exception result = new Exception();

        try {
            future.get();
            future2.get();
            future3.get();
        } catch (ExecutionException e) {
            result = (Exception) e.getCause();
        }

        assertTrue(result instanceof OptimisticLockingFailureException);
    }

    Stock 재고_1개_생성() {
        Stock stock = Stock.createStock("불닭볶음면", 1L);
        stockRepository.save(stock);
        return stock;
    }
}
```

![optimistic-lock](https://user-images.githubusercontent.com/28802545/187021259-f4219d0f-967b-497f-a180-fb622456c86b.png)

테스트 결과를 보면 정상적으로 `OptimisticLockingFailureException` 이 발생하여 테스트가 정상적으로 통과된것을 확인할 수 있습니다.

<br>
감사합니다

<br>
<br>

## **reference**

<a>https://www.baeldung.com/jpa-optimistic-locking</a>
