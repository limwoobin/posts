![test-context-caching-1](https://private-user-images.githubusercontent.com/28802545/335857020-9e91c0b3-5058-4bb5-816c-10cadfe1ee73.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTczMjQxNTIsIm5iZiI6MTcxNzMyMzg1MiwicGF0aCI6Ii8yODgwMjU0NS8zMzU4NTcwMjAtOWU5MWMwYjMtNTA1OC00YmI1LTgxNmMtMTBjYWRmZTFlZTczLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA2MDIlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNjAyVDEwMjQxMlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTE4ZTRhMzg0NDI4ZjMzMWE2NGEwMmE4N2FjZTIyYzNlOGEyYTg4Yjc1NmZmOWU4MGNlNmI3YzY3MzAzOTRkMDcmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.xxkno5eJjPffppTJC2WwH6uCFbF7RE5B5hEohsQ9ulY)

# __Spring Test 에서 ApplicationContext 캐싱하기__

안녕하세요 Spring 환경에서 테스트시에 테스트 코드가 늘어남에 따라 테스트 수행시간이 오래걸리는 경우를 한번쯤 겪어보셨을텐데요.

일반적으로는 테스트시 __`@SpringBootTest`__ 를 이용하게 되면 스프링의 __`ApplicationContext`__ 에 있는 모든 Bean 을 띄우게 됩니다. 이 과정에서의 소요시간이 상당한데요.

만약 테스트마다 __`ApplicationContext`__ 를 새로 띄우게 되면 그만큼 시간이 더 오래 소요될텐데요. 스프링에서는 이런 문제가 없기 위해 __`Context Caching`__ 이라는 개념이 존재합니다.  
테스트시 __`ApplicationContext`__ 를 캐싱해서 재사용하는 개념입니다.

이 __`Context Caching`__ 을 이용해 테스트 성능개선을 할 수 있습니다.  
이외에도 성능을 개선하기 위해서는 테스트에 꼭 필요한 Bean 들만 띄운다거나 외부 I/O 가 존재하는 경우 해당 부분을 개선하거나 서버를 업그레이드 하는 여러 방법이 존재하는데요.

이 글에서는 __`ApplicationContext`__ 를 캐싱해서 테스트 코드 성능을 개선하는 방법에 대해 알아보겠습니다.

<br />

## __`ApplicationContext`__ 가 캐싱되지 않는 이유

스프링 공식문서에서는 다음과 같이 이야기하고있습니다.

> Once the TestContext framework loads an ApplicationContext (or WebApplicationContext) for a test, that context is cached and reused for all subsequent tests that declare the same unique context configuration within the same test suite. To understand how caching works, it is important to understand what is meant by “unique” and “test suite.”
> 
> An ApplicationContext can be uniquely identified by the combination of configuration parameters that is used to load it. Consequently, the unique combination of configuration parameters is used to generate a key under which the context is cached.

TestContext 프레임워크에서 테스트에 대한 __`ApplicationContext`__ 를 로드하면 해당 컨텍스트는 캐시되어 동일한 환경내에서 재사용된다고 합니다.  

그리고 __`ApplicationContext`__ 를 로드하는데 `configuration parameters` 이 사용됩니다.  
그리고 `configuration parameters` 의 조합으로  __`ApplicationContext`__ 를 식별할 수 있는 __CacheKey__ 를 생성합니다.  
이 __Key__ 를 이용하여 __`ApplicationContext`__ 가 동일한 환경인지 판단하고 __Key__ 가 같으면 __`ApplicationContext`__ 를 재사용하고 아니라면 새로운 컨텍스트를 띄우게 됩니다.

__`TestContext`__ 의 구성 매개변수는 다음과 같습니다.

- `locations` (from `@ContextConfiguration`)
- `classes` (from `@ContextConfiguration`)
- `contextInitializerClasses` (from `@ContextConfiguration`)
- `contextCustomizers` (from `ContextCustomizerFactory`) – this includes `@DynamicPropertySource` methods as well as various features from Spring Boot’s testing support such as `@MockBean` and `@SpyBean`.
- `contextLoader` (from `@ContextConfiguration`)
- `parent` (from `@ContextHierarchy`)
- `activeProfiles` (from `@ActiveProfiles`)
- `propertySourceDescriptors` (from `@TestPropertySource`)
- `propertySourceProperties` (from `@TestPropertySource`)
- `resourceBasePath` (from `@WebAppConfiguration`)

해당 매개변수들의 설정이 변경되면 __`ApplicationContext`__ 의 __CacheKey__ 가 변경되어 TestContext 가 캐싱되지 않는것인데요.

이 중 네번째 항목을 한번 보겠습니다.

> `contextCustomizers` (from `ContextCustomizerFactory`) – this includes `@DynamicPropertySource` methods as well as various features from Spring Boot’s testing support such as `@MockBean` and `@SpyBean`.

`contextCustomizers` 는  `@MockBean` 과 `@SpyBean` 같은 SpringBoot 의 테스트를 지원하는 기능이 포함되어있다고 얘기합니다.  

즉, `@MockBean` 과 `@SpyBean` 를 사용할 시에 TestContext 가 변경되어 캐싱을 하지 못하는것입니다.

테스트에서 가짜 객체를 사용하기 위해 `@MockBean` 과 `@SpyBean` 어노테이션을 많이 사용하실텐데요.  
이 기능을 사용하면서도 __`ApplicationContext`__ 를 캐싱하는 방법에 대해서 공유드리겠습니다.

<br />

## __`ApplicationContext`__ 가 캐싱되지 않는 예제 코드

우선 `@MockBean`, `@SpyBean` 을 사용시 정말 Context 가 캐싱되지 않는지 먼저 확인해보겠습니다.

<br />

__MyService.java__
```java
@Component
public class MyService {
  public String get() {
    return "MyService";
  }
}
```

Mocking 할 클래스를 작성하겠습니다. @Component 를 이용해 Bean 으로 등록해줍니다.

<br />

__ApplicationContextInitListener.java__
```java
@Component
public class ApplicationContextInitListener implements ApplicationListener<ContextRefreshedEvent> {
  private static final AtomicInteger contextInitCount = new AtomicInteger(0);

  @Override
  public void onApplicationEvent(ContextRefreshedEvent event) {
    contextInitCount.incrementAndGet();
  }

  public Integer getContextInitCount() {
    return contextInitCount.get();
  }
}
```

`ApplicationContext` 가 로딩될때마다 실행되는 리스너 객체를 생성합니다.  
이 리스너에서는 `ApplicationContext` 가 로드될때마다 AtomicValue 를 하나씩 증가하여 총 몇번의 `ApplicationContext` 가 초기화되었는지 카운팅 하겠습니다.

<br />

__IntegrationNoCacheTest.java__
```java
@SpringBootTest
public class IntegrationNoCacheTest {

  @Autowired
  private MyService myService;

  @Autowired
  private ApplicationContext applicationContext;

  @Autowired
  private ApplicationContextInitListener initListener;

  @Test
  void test() {
    System.out.println("myService: " + myService);

    System.out.println("applicationContext.hashCode(): " + applicationContext.hashCode());
    System.out.println("context init Count: " + initListener.getContextInitCount());
  }
}
```

<br />

__IntegrationNoCacheTestV2.java__
```java
@SpringBootTest
public class IntegrationNoCacheTestV2 {

  @MockBean
  private MyService mockMyService;

  @Autowired
  private ApplicationContext applicationContext;

  @Autowired
  private ApplicationContextInitListener initListener;

  @Test
  void test() {
    System.out.println("mockMyService: " + mockMyService);

    System.out.println("applicationContext.hashCode(): " + applicationContext.hashCode());
    System.out.println("context init Count: " + initListener.getContextInitCount());
  }
}
```

<br />

__IntegrationNoCacheTestV3.java__
```java
@SpringBootTest
public class IntegrationNoCacheTestV3 {

  @SpyBean
  private MyService spyMyService;

  @Autowired
  private ApplicationContext applicationContext;

  @Autowired
  private ApplicationContextInitListener initListener;

  @Test
  void test() {
    System.out.println("mockMyService: " + spyMyService);

    System.out.println("applicationContext.hashCode(): " + applicationContext.hashCode());
    System.out.println("context init Count: " + initListener.getContextInitCount());
  }
}
```

<br />

- IntegrationNoCacheTest: `@Autowrited` 로 MyService 주입
- IntegrationNoCacheTestV2: `@MockBean` 로 MyService 주입
- IntegrationNoCacheTestV3: `@SpyBean` 로 MyService 주입

모든 테스트 코드가 `MyService` 객체를 주입받는 방법이 상이합니다.  
이 테스트를 모두 한번에 실행한 경우 예상되는 시나리오는 각 테스트마다 `ApplicationContext` 가 초기화되어 총 3번이 초기화될것으로 예상됩니다.

![test-context-caching-2](
https://private-user-images.githubusercontent.com/28802545/335853260-fcc66083-c6d3-40f0-a2d3-52e92d4bcc56.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTczMjA4NjcsIm5iZiI6MTcxNzMyMDU2NywicGF0aCI6Ii8yODgwMjU0NS8zMzU4NTMyNjAtZmNjNjYwODMtYzZkMy00MGYwLWEyZDMtNTJlOTJkNGJjYzU2LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA2MDIlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNjAyVDA5MjkyN1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTdiNzE3OTBiZjE5ZTc3Mjc3ZDFlMzJiNmNhMGNiNzk1ZjZkYWQwZmM4NzdkYjNiZmFhYWUwZGJhOGUyZDY1MDAmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.ppeKIxAhgVa7KLp9RRx3bZstE58IXC2JFksui7vyk6o)

다음과 같이 `ApplicationContext` 초기화가 3번 된것을 확인할 수 있습니다.

<br />

## `@MockBean`, `@SpyBean` 사용시 __`ApplicationContext`__ 캐싱하기

`@MockBean`, `@SpyBean` 을 사용하면서 ApplicationContext 를 캐싱하기 위해서는  
가짜객체에 대해 `TestConfiguration` 으로 설정하여 테스트 실행시 같이 Bean 을 로드시키는 방법이 있습니다.

아래 예제코드를 한번 살펴보겠습니다.

__MyServiceTestConfig.java__
```java
@TestConfiguration
public class MyServiceTestConfig {

  @Autowired
  private MyService myService;

  @Bean
  public MyService mockMyService() {
    return Mockito.mock(MyService.class);
  }

  @Bean
  public MyService spyMyService() {
    return Mockito.spy(myService);
  }
}
```

`MyService.class` 객체에 대해 테스트용 configuration 을 사용하도록 다음과 같이 설정했습니다.  
추가로 Bean 의 이름은 실제 환경과 충돌을 방지하기 위해 mock 전용, spy 전용이라는 의미로 `mockMyService`, `spyMyService` 로 명시해주었습니다.

<br />

__IntegrationCacheTest.java__
```java
@SpringBootTest
@ContextConfiguration(classes = MyServiceTestConfig.class)
public class IntegrationCacheTest {

  @Autowired
  private MyService mockMyService;

  @Test
  void test() {
    // given
    String expected = "test";

    // when
    when(mockMyService.get()).thenReturn(expected);

    // then
    boolean result = MockUtil.isMock(mockMyService);
    String value = mockMyService.get();

    assertThat(result).isTrue();
    assertThat(value).isEqualTo(expected);
  }
}
```

`@ContextConfiguration(classes = MyServiceTestConfig.class)` 를 이용해 테스트 Config 를 설정한 클래스를 가져옵니다.  
그리고 테스트 코드에서 `MyService` 의 mocking 이 잘 동작하는지, Mock 객체가 맞는지를 검증합니다.

<br />

![test-context-caching-3](
https://private-user-images.githubusercontent.com/28802545/335853720-b04f130d-d0d8-475a-9233-0d51d409ea2b.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTczMjEzNjAsIm5iZiI6MTcxNzMyMTA2MCwicGF0aCI6Ii8yODgwMjU0NS8zMzU4NTM3MjAtYjA0ZjEzMGQtZDBkOC00NzVhLTkyMzMtMGQ1MWQ0MDllYTJiLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA2MDIlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNjAyVDA5Mzc0MFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTM4NmJkZDJmMGIzNzExNWRjYTU2Yzk4YzQ1M2NkYzJmMzYzMzBhZTdmZDhiZmQ3YjgwZTU4MmM3MmI5YTVjY2EmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.-mTEyQPqLSuEY5M5WTgMjsiNjFGZuX09cvBZQ5AXZmQ)

다음과 같이 테스트가 통과한것을 확인할 수 있습니다.

그렇다면 하나는 `@Autowired` 로 나머지는 `@MockBean`, `@SpyBean` 으로 주입받으면 어떨지 한번 확인해보겠습니다.

__IntegrationCacheTest.java__
```java
@SpringBootTest
@ContextConfiguration(classes = MyServiceTestConfig.class)
public class IntegrationCacheTest {

  @Autowired
  private MyService myService;

  @Autowired
  private ApplicationContext applicationContext;

  @Autowired
  private ApplicationContextInitListener initListener;

  @Test
  void test() {
    System.out.println("context init Count: " + initListener.getContextInitCount());

    assertThat(MockUtil.isMock(myService)).isFalse();
    assertThat(MockUtil.isSpy(myService)).isFalse();
  }
}
```

<br />

__IntegrationCacheTestV2.java__
```java
@SpringBootTest
@ContextConfiguration(classes = MyServiceTestConfig.class)
public class IntegrationCacheTestV3 {

  @Autowired
  private MyService mockMyService;

  @Autowired
  private ApplicationContext applicationContext;

  @Autowired
  private ApplicationContextInitListener initListener;

  @Test
  void test() {
    System.out.println("context init Count: " + initListener.getContextInitCount());

    assertThat(MockUtil.isMock(mockMyService)).isTrue();
  }
}
```

<br />

__IntegrationCacheTestV3.java__
```java
@SpringBootTest
@ContextConfiguration(classes = MyServiceTestConfig.class)
public class IntegrationCacheTestV3 {

  @Autowired
  private MyService spyMyService;

  @Autowired
  private ApplicationContext applicationContext;

  @Autowired
  private ApplicationContextInitListener initListener;

  @Test
  void test() {
    System.out.println("context init Count: " + initListener.getContextInitCount());

    assertThat(MockUtil.isSpy(spyMyService)).isTrue();
  }
}
```

- IntegrationCacheTest: `MyService` 를 그대로 Bean 주입 받음
- IntegrationCacheTestV2: `MyService` 를 mock 으로 Bean 주입받음
- IntegrationCacheTestV3: `MyService` 를 spy 로 Bean 주입받음

각 테스트들은 모두 서로다른 `MyService` 객체를 주입받고 있습니다.  
앞선 `@MockBean`, `@SpyBean` 을 직접 이용했을때는 `ApplicationContext` 가 캐싱되지 않았지만 이번엔 `MyServiceTestConfig` 를 이용해 모두 테스트 환경에서 Bean 을 등록시켰습니다.

테스트를 실행해보겠습니다.

![test-context-caching-4](
https://private-user-images.githubusercontent.com/28802545/335855955-de4c4532-6f54-4ef6-bd04-c1c4f1676a32.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTczMjMyMjIsIm5iZiI6MTcxNzMyMjkyMiwicGF0aCI6Ii8yODgwMjU0NS8zMzU4NTU5NTUtZGU0YzQ1MzItNmY1NC00ZWY2LWJkMDQtYzFjNGYxNjc2YTMyLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA2MDIlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNjAyVDEwMDg0MlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTBiMjIzYjVkYTYwM2U2ODhjMDEyYmVlYWVkNDU3ZmQ3Nzg1ZWM4YzllMWE5Yzk1OTQ0MDc5ZTlhYzYxMmM0YzUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.5V-3rzVR_wyU7xNFA3iyLE-wS4sejzrf2aUANwIMH3s)

`ApplicationContext` 초기화도 한번만 되었고 각 테스트에서 주입받은  `MyService` 객체도 `Mock`, `Spy` 로 알맞게 주입받아 테스트에 통과한것을 확인할 수 있습니다.

### __실제 Bean 과 MockBean / SpyBean 을 같이 쓰고 싶은 경우에는?__

가짜객체에 대해서는  `MyServiceTestConfig.class` 에서 별도의 Bean 이름으로 명시해주고 있습니다.

__MyServiceTestConfig.java__
```java
@TestConfiguration
public class MyServiceTestConfig {

  @Autowired
  private MyService myService;

  @Bean
  public MyService mockMyService() {
    return Mockito.mock(MyService.class);
  }

  @Bean
  public MyService spyMyService() {
    return Mockito.spy(myService);
  }
}
```

이렇게 설정한 경우 사용시 아래와 같이 설정하면 실제 해당 객체를 주입받을 수 있습니다.

```java
@Autowired
private MyService myService;
-> myService 라는 네이밍으로 Bean 을 가져오기에 실제 MyService Bean 객체를 주입

@Autowired
private MyService mockMyService;
-> mockMyService 라는 네이밍으로 Bean 을 가져오기에 MyServiceTestConfig.class 에서 mockMyService Bean 을 찾아 주입

@Autowired
private MyService spyService;
-> spyMyService 라는 네이밍으로 Bean 을 가져오기에 MyServiceTestConfig.class 에서 spyMyService Bean 을 찾아 주입
```

<br />

__IntegrationCacheTest.java__
```java
@SpringBootTest
@ContextConfiguration(classes = MyServiceTestConfig.class)
public class IntegrationCacheTest {

    @Autowired
    private MyService myService;

    @Autowired
    private MyService mockMyService;

    @Autowired
    private MyService spyMyService;

    @Test
    void test() {
        System.out.println("myService: " + myService);
        System.out.println("mockMyService: " + mockMyService);
        System.out.println("spyMyService: " + spyMyService);

        assertThat(MockUtil.isMock(myService)).isFalse();
        assertThat(MockUtil.isSpy(myService)).isFalse();
        assertThat(MockUtil.isMock(mockMyService)).isTrue();
        assertThat(MockUtil.isSpy(spyMyService)).isTrue();
    }
}
```

`MyService` 를 각 환경에 맞게 주입받아 로그를 찍어서 mock, spy 를 잘 주입받았는지 확인해보겠습니다.

![test-context-caching-5](
https://private-user-images.githubusercontent.com/28802545/335856464-2bd91a83-6f48-484d-8eb1-c1dc44159399.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTczMjM3OTAsIm5iZiI6MTcxNzMyMzQ5MCwicGF0aCI6Ii8yODgwMjU0NS8zMzU4NTY0NjQtMmJkOTFhODMtNmY0OC00ODRkLThlYjEtYzFkYzQ0MTU5Mzk5LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA2MDIlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNjAyVDEwMTgxMFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTdkODA4N2ZiMDAxNzI3YjEwZjVkZTM2YzRlZWVmM2Y5NTllYmFmZGJkNjg5NmEyNTkyNjdkMThmYmYyYTRhNzYmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.Tiz3jFcrUaELeH7-22Sa2pxkrTSBX2Hiy3jc8S0ellQ)

다음과 같이 정상적으로 Bean 을 주입받고 테스트코드도 통과한것을 확인할 수 있습니다.  
감사합니다.

#### __reference__

- https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/ctx-management/caching.html