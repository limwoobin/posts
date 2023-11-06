[**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/spring-test-context)

<br>

## __스프링에서 여러 테스트 클래스에서 테스트 데이터를 공유하는 방법__

이번에는 Spring 환경에서 서로 다른 테스트 클래스에서 데이터를 공유하는 방법에 대해 소개해드리려 합니다.  
`Spring Integration Test` 를 작성하다 보면 테스트를 위한 데이터를 세팅하는 과정, 혹은 테스트를 위한 선행 작업 or 전처리 작업이 필요한 경우가 있습니다.  
이때, 일반적으로 많이 사용하는 방법으로는 `@BeforeEach`, `@BeforeAll` 과 같은 JUnit 라이프 사이클 어노테이션을 많이 이용하게 됩니다.

<br>

`@BeforeEach` 의 경우 테스트 마다 매번 실행되기에 테스트 간의 격리를 할 수 있어 보다 신뢰성 있는 테스트를 할 수 있다는 장점이 있지만 그만큼 테스트 수행시간이 오래걸리는 특징이 있습니다.  
`@BeforeAll` 의 경우 클래스에서 단 한번만 실행된다는 특징이 있지만 테스트를 위한 선행작업을 여러 테스트 클래스에서 해야 하는 경우 실행되는 테스트 클래스의 수 만큼 수행되게 됩니다.  
두 어노테이션 모두 반복되는 코드를 줄여주고 테스트간의 격리(ex. 데이터 초기화)를 할 수 있다는 점에서 큰 장점이 있다고 생각합니다.

<br>

어떤 테스트를 작성하는데 해당 테스트를 위한 선행 작업이 굉장히 오래걸리는 테스트가 있다고 가정해보겠습니다.  
그리고 해당 테스트는 굉장히 다양한 테스트 시나리오를 가지고 있습니다. 그렇기에 테스트 가독성을 위해 테스트를 시나리오별로 클래스로 분리하였습니다.  
이 테스트를 메소드 단위로, 혹은 클래스 단위로 테스트 선행 작업을 모두 수행하게 된다면 테스트 수행 시간은 굉장히 오래 걸리게 될텐데요.  

<br>

이때, 여러 테스트 클래스에서 데이터를 공유해서 사용하는 방법에 대해 공유해드리려 합니다.

<br>

## __TestExecutionListener__

`TestExecutionListener` 는 테스트의 라이프사이클 및 커스텀 로직을 사용할 수 있게 하는 도구입니다. 해당 리스너를 커스텀하게 사용하면 여러 테스트 클래스에서 같은 데이터를 공유할 수 있습니다.  
즉, 여러 테스트 클래스를 동시에 수행하더라도 테스트 선행 작업은 딱 한번만 수행되게 할 수 있습니다.

## __예제 코드__

테스트 코드에는 다음과 같은 시나리오가 있다고 가정해보겠습니다.  
통합 테스트 클래스 A와 B 가 존재합니다. 이 두 테스트는 테스트를 수행하기 위해서는 앞선 Coupon 데이터가 DB에 존재해야 한다는 전제조건이 있습니다.  

<br>

이때, 저장된 쿠폰 데이터는 A와 B 두 테스트 클래스에서 모두 필요한 데이터라고 해보겠습니다.

테스트 데이터를 공유하기 위해 커스텀 테스트 리스너를 만들겠습니다.

<br>

__CustomTestExecutionListener__
```java
public class CustomTestExecutionListener implements TestExecutionListener {
  private static boolean initialize = false;    // (1)

  @Autowired
  private CouponRepository couponRepository;

  @Override
  public void beforeTestClass(TestContext testContext) throws Exception {   // (2)
    testContext.getApplicationContext()
      .getAutowireCapableBeanFactory()
      .autowireBean(this);      // (3)

    if (!initialize) {  // (4)
      System.out.println("beforeTestClass Run ...");

      Coupon coupon = new Coupon("coupon");
      couponRepository.save(coupon);
      initialize = true;
    }
  }
}
```

<br>

__(1)__ 데이터가 초기화 되었는지 판단할 수 있는 initialize 라는 변수를 static 으로 만들어 둡니다.  
__(2)__ `beforeTestClass` 는 `TestExecutionListener` 에서 제공하는 라이프사이클 메소드입니다.  
`beforeTestClass` 외에도 `beforeTestMethod, beforeTestExecution ` 등의 여러 라이프사이클 메소드가 제공됩니다.  
저희는 초기화 유무를 판단하기 때문에 위의 메소드를 사용하여도 무관합니다.  
__(3)__ `CustomTestExecutionListener` 에서 의존성 주입을 받아야 하기 때문에 의존성 주입을 설정합니다.  
__(4)__ 데이터가 세팅 되었는지 initialize 변수를 보고 판단하여 세팅되지 않았다면 데이터 셋팅을 진행합니다.

<br>
<br>

__Coupon.java__
```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Table(name = "coupons")
public class Coupon {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column
  private String name;

  public Coupon(String name) {
    this.name = name;
  }
}
```

<br>

다음은 실제 테스트가 실행되는 클래스입니다.
__Integration A, B__
```java

class IntegrationTestA extends AbstractIntegrationTest {

  @Autowired
  private CouponRepository couponRepository;

  @DisplayName("TestExecution Listener Test A Run...")
  @Test
  void test() {
    List<Coupon> result = couponRepository.findAll();
    System.out.println("coupon size: " + result.size());
  }
}

class IntegrationTestB extends AbstractIntegrationTest {

  @Autowired
  private CouponRepository couponRepository;

  @DisplayName("TestExecution Listener Test B Run...")
  @Test
  void test() {
    List<Coupon> result = couponRepository.findAll();
    System.out.println("coupon size: " + result.size());
  }
}

```

<br>

테스트 데이터를 같이 사용하기 위해 공통 부모 클래스를 추가해줍니다.

__AbstractIntegrationTest.java__
```java
@TestExecutionListeners(value = {   // (1)
  CustomTestExecutionListener.class,
  DependencyInjectionTestExecutionListener.class
})
@SpringBootTest
public abstract class AbstractIntegrationTest {
}
```

__(1)__ 위에서 설정한 CustomTestExecutionListener 를 등록합니다. 의존성 주입을 위 `DependencyInjectionTestExecutionListener` 도 같이 등록합니다.

<br>

그럼 이제 `IntegrationTestA` , `IntegrationTestB` 테스트 클래스를 동시에 실행시켜 보겠습니다.  
기대하는 시나리오는 다음과 같습니다.

- `IntegrationTestA` 가 실행되면서 `CustomTestExecutionListener` 의 `beforeTestClass` 메소드가 수행된다.
- `CustomTestExecutionListener` 의 static 변수인 `initialize` 가 false 이므로 쿠폰 등록이 수행된다.
- `IntegrationTestB` 가 실행되면서 `CustomTestExecutionListener` 의 `beforeTestClass` 메소드가 수행된다.
- `CustomTestExecutionListener` 의 static 변수인 `initialize` 가 true 이므로 쿠폰 등록은 건너뛴다.
- `IntegrationTestA` 의 테스트 결과는 쿠폰등록이 한번만 되었기 때문에 coupon size 는 1이 되어야 한다.
- `IntegrationTestB` 의 테스트 결과 또한 쿠폰등록이 한번만 되었기 때문에 coupon size 는 1이 되어야 한다.

<br>

![spring-test-datainit](https://user-images.githubusercontent.com/28802545/280528663-1db15fea-97fe-4d62-a996-6ec85c3285ba.png)

테스트 결과는 다음과 같이 찍어놓은 메시지를 통해 확인할 수 있습니다.  
데이터를 세팅하는 `beforeTestClass Run...` 와 `insert dml` 쿼리가 한번만 찍힌것을 볼 수 있고  
`IntegrationTestA`, `IntegrationTestB` 테스트의 결과 또한 쿠폰 한개만 정상적으로 등록된 것을 볼 수 있습니다.  

### __고민하면 좋을 부분__

이렇게 테스트 수행시간을 단축시킬수 있다는 장점은 있지만 두 테스트 사이에 격리성이 있다고 보기에는 어렵습니다.  
만약 테스트 로직 중 쿠폰 데이터와 같이 세팅된 데이터를 수정하게 되면 다른 클래스에서는 테스트 시나리오가 깨질 위험도 존재하기 때문입니다.  
그렇기에 속도도 중요하지만 테스트간의 격리성 또한 중요하기에 정답은 없지만 테스트 작성시 기회비용이라던지 중요성 등 여러 부분에 대해 고민하면서 작성하는것이 가장 좋지 않을까 싶습니다.


<br>

더 나은 방법이나 다른 의견들이 있다면 코멘트로 많이 남겨주시면 감사하겠습니다.

<br>

## reference

[https://www.baeldung.com/spring-testexecutionlistener](https://www.baeldung.com/spring-testexecutionlistener)  
[https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/context/support/DependencyInjectionTestExecutionListener.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/context/support/DependencyInjectionTestExecutionListener.html)  
