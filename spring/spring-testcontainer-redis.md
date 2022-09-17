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

<br>

## **Embedded Redis**

build.gradle

```java
//embedded-redis
implementation group: 'it.ozimov', name: 'embedded-redis', version: '0.7.1'
```

EmbeddedRedisConfig.java

```java
package com.example.redisexample;

import org.junit.jupiter.api.DisplayName;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import redis.embedded.RedisServer;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import java.io.IOException;

@DisplayName("Embedded Redis 설정")
@Profile("test")
@Configuration
public class EmbeddedRedisConfig {
   private RedisServer redisServer;

   public EmbeddedRedisConfig(@Value("${spring.redis.port}") int port) throws IOException {
       this.redisServer = new RedisServer(port);
   }

   @PostConstruct
   public void startRedis() {
       this.redisServer.start();
   }

   @PreDestroy
   public void stopRedis() {
       this.redisServer.stop();
   }
}
```

다음은 it.ozimov 라이브러리를 이용한 Embedded Redis 설정 방법입니다.  
테스트 코드를 실행해보겠습니다.

![redis-test-image1](https://user-images.githubusercontent.com/28802545/190381829-1a508329-dc73-4791-857b-6b8678ed4e8f.png)

정상적으로 테스트가 통과한것을 확인할 수 있습니다.

<br>

하지만 `Embedded Redis`에도 문제가 있습니다. `Embedded Redis`는 테스트시 새로운 스프링 컨텍스트가 생성되면 `Embedded Redis`를 새로 띄우게 됩니다.  
이 과정에서 `Redis`의 기본포트인 6379가 이미 사용중이기 때문에 `Embedded Redis`를 띄우지 못하고 테스트 역시 실패되는 것입니다.

<br>

위에서 작성한 `RedisCrudTest.java` 를 복사하여 `RedisCrudTest2.java` 를 하나 생성하겠습니다.

```java
RedisCrudTest.java

@DisplayName("Redis CRUD Test")
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class RedisCrudTest {

-------------------------------------------------

RedisCrudTest2.java

@DisplayName("Redis CRUD Test2")
@SpringBootTest
class RedisCrudTest2 {
```

`@SpringBootTest` 설정을 다르게 주어 스프링 컨텍스트가 새로 띄워지게 의도했습니다. 그리고 테스트를 실행해보겠습니다.

![redis-test-image2](https://user-images.githubusercontent.com/28802545/190388272-c1948c54-ed8b-4899-9afc-bdd2f989e7cd.png)

다음과 같이 `Embedded Redis` 가 실행되지 못해 실패로 띄워지는것을 확인 할 수 있습니다.

<br>

그리고 한가지 문제가 또 있습니다.  
`Embedded Redis` 라이브러리는 지속적인 업데이트나 개선이 이뤄지고 있지 않습니다.  
[it.ozimov](https://github.com/ozimov/embedded-redis) 의 최근 커밋은 2020년 6월 입니다.  
다른 라이브러리인 [kstyrc](https://github.com/kstyrc/embedded-redis) 는 더 이전인 2018년 9월이 마지막인것을 알 수 있습니다.

<br>
<hr>

## **TestContainer**

TestContainer는 테스트 환경에서 도커 컨테이너를 실행할 수 있는 라이브러리 입니다.  
Testcontainer 를 이용하면 테스트시에 좀 더 production환경에 가까운 테스트를 할 수 있습니다.  
ex) `일반적인 h2 db 대신 mysql container를 이용한 테스트`

<br>

그리고 도커만 설치되어 있다면 별도의 환경구축이 필요하지 않습니다. 다만, 컨테이너를 생성,삭제하는 과정이 있기 때문에
테스트가 느려진다는 단점이 존재합니다.

이제 TestContainer를 이용하여 Redis Container 를 구축해보겠습니다.

<br>

### **Redis Container 구축하기**

build.gradle

```java
// test-containers
testImplementation group: 'org.testcontainers', name: 'testcontainers', version: '1.17.2'
```

<br>

RedisTestContainers.java

```java
import org.junit.jupiter.api.DisplayName;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.utility.DockerImageName;

@DisplayName("Redis Test Containers")
@ActiveProfile("test")
@Configuration
public class RedisTestContainers {

    private static final String REDIS_DOCKER_IMAGE = "redis:5.0.3-alpine";

    static {    // (1)
        GenericContainer<?> REDIS_CONTAINER =
            new GenericContainer<>(DockerImageName.parse(REDIS_DOCKER_IMAGE))
                .withExposedPorts(6379)
                .withReuse(true);

        REDIS_CONTAINER.start();    // (2)

        // (3)
        System.setProperty("spring.redis.host", REDIS_CONTAINER.getHost());
        System.setProperty("spring.redis.port", REDIS_CONTAINER.getMappedPort(6379).toString());
    }
}
```

- (1) `redis:5.0.3-alpine` 라는 이미지에 새 컨테이너를 생성한다.
- (2) Redis Container 를 실행한다.
  - withExposedPorts: container의 port를 6379로 연다.
  - withReuse: 해당 컨테이너를 재사용한다.
- (3) RedisContainer 와 연결하기 위해 host, port를 매핑한다.

다음과 같이 설정하면 테스트 실행시 도커로 Redis Container를 띄울 수 있습니다.

<br>

이제 테스트 코드를 실행해보겠습니다.

![redis-test-image3](https://user-images.githubusercontent.com/28802545/190628972-c8af0b8e-6399-48cb-916b-ecf041c1757e.png)

테스트를 실행하면 도커 컨테이너가 새로 띄워지는 것을 확인할 수 있습니다.
redis 컨테이너는 지정한 `redis:5.0.3-alpine` 이미지로 띄워진 것을 볼 수 있습니다.  
그렇다면 ryuk container는 뭐하는 컨테이너일까요?

<br>
 
### __Ryuk Container__

[Docker](https://hub.docker.com/r/testcontainers/ryuk)에서 이야기하는 Ryuk Container의 역할은 다음과 같습니다.

> This project helps you to remove containers/networks/volumes/images by given filter after specified delay.

Ryuk은 테스트에 사용되는 컨테이너를 관리합니다. 테스트 컨테이너를 모니터링하고 테스트가 종료되면 컨테이너/이미지를 종료/삭제하는 일을 합니다.

테스트 결과를 확인해보겠습니다.

![redis-test-image4](https://user-images.githubusercontent.com/28802545/190847909-aaa7de17-6022-45c6-a0dc-1c991fae344a.png)

정상적으로 테스트가 통과된것을 확인할 수 있습니다.  
이렇게 EmbeddedRedis, TestContainer를 이용한 테스트방법을 알아보았습니다.

감사합니다.

<hr>

## Reference

[https://www.baeldung.com/spring-boot-redis-testcontainers](https://www.baeldung.com/spring-boot-redis-testcontainers)
