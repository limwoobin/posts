#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/redis-example)

<br>

## Spring 에서 Redis를 테스트 하는 방법

이번엔 Spring 에서 Redis를 테스트 하는 방법에 대해 알아보려 합니다.

- **Embedded Redis**
- **Test-Containers**

Redis를 테스트 한다면 다음과 같은 방법들을 이용해 테스트를 진행할 수 있습니다.

<br>

로컬pc에 직접 Redis를 띄워서 테스트 코드를 검증하는 경우도 많이 봤지만 그런 방식은 저는 추천드리지 않습니다.  
테스트 코드는 어느 환경에서든 동일하게 실행되어야 한다고 생각합니다.  
만약 로컬Pc에 Redis를 설치하여 테스트한다면 다음과 같은 문제점이 있습니다.

- 테스트를 수행하는 pc마다 Redis 인스턴스를 직접 구축해야 하는 번거로움이 있다.
- 로컬 Redis에 이미 저장된 값이 테스트를 오염시킬 수 있다. 이는 테스트를 수행하는 pc마다 결과가 다르게 나올 수 있다.  
즉, 일관적인 테스트가 보장되지 않는다.

위와 같은 이유로 해당 방법을 저는 개인적으로 지양하고 있습니다.

<hr>
<br>

**그럼 테스트를 하기 위한 코드를 먼저 작성해보겠습니다.**

## **Redis Test 예제 코드**

### 환경

- Spring Boot 2.7.3
- java 11
- junit 5

<br>

Product.java

```java
import lombok.Getter;
import org.springframework.data.annotation.Id;
import org.springframework.data.redis.core.RedisHash;

@Getter
@RedisHash("product")
public class Product {
    @Id
    private String id;

    private String name;

    private Long price;

    public Product(String id, String name, Long price) {
        this.id = id;
        this.name = name;
        this.price = price;
    }

    public void changePrice(Long price) {
        this.price = price;
    }
}
```

Spring Data JPA를 사용해 객체를 redis에 저장하기 위해 `@RedisHash` 를 사용했습니다.

<br>
<br>

ProductRepository.java

```java
import org.springframework.data.repository.CrudRepository;

public interface ProductRepository extends CrudRepository<Product, String> {
}
```

<br>

해당 Product 객체를 CRUD하는 테스트 코드를 작성해보겠습니다.

RedisCrudTest.java

```java
@DisplayName("Redis CRUD Test")
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class RedisCrudTest {

    @Autowired
    private ProductRepository productRepository;

    private Product product;

    @BeforeEach
    void setUp() {
        product = new Product("P0001", "테스트_상품", 20000L);
    }

    @AfterEach
    void teardown() {
        productRepository.deleteById(product.getId());
    }

    @Test
    @DisplayName("Redis 에 데이터를 저장하면 정상적으로 조회되어야 한다")
    void redis_save_test() {
        // given
        productRepository.save(product);

        // when
        Product persistProduct = productRepository.findById(product.getId())
                .orElseThrow(RuntimeException::new);

        // then
        assertThat(persistProduct.getId()).isEqualTo(product.getId());
        assertThat(persistProduct.getName()).isEqualTo(product.getName());
        assertThat(persistProduct.getPrice()).isEqualTo(product.getPrice());
    }

    @Test
    @DisplayName("Redis 에 데이터를 수정하면 정상적으로 수정되어야 한다")
    void redis_update_test() {
        // given
        productRepository.save(product);
        Product persistProduct = productRepository.findById(product.getId())
                .orElseThrow(RuntimeException::new);

        // when
        persistProduct.changePrice(35000L);
        productRepository.save(persistProduct);

        // then
        assertThat(persistProduct.getPrice()).isEqualTo(35000L);
    }

    @Test
    @DisplayName("Redis 에 데이터를 삭제하면 정상적으로 삭제되어야 한다")
    void redis_delete_test() {
        // given
        productRepository.save(product);

        // when
        productRepository.delete(product);
        Optional<Product> deletedProduct = productRepository.findById(product.getId());

        // then
        assertTrue(deletedProduct.isEmpty());
    }
}
```

## Reference

[https://www.baeldung.com/spring-boot-redis-testcontainers](https://www.baeldung.com/spring-boot-redis-testcontainers)
