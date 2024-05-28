![test-context-caching-1]()

# Spring Test 시 ApplicationContext 캐싱하기

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

이 중 아래의 설정을 보겠습니다.

> `contextCustomizers` (from `ContextCustomizerFactory`) – this includes `@DynamicPropertySource` methods as well as various features from Spring Boot’s testing support such as `@MockBean` and `@SpyBean`.

`contextCustomizers` 는  `@MockBean` 과 `@SpyBean` 같은 SpringBoot 의 테스트를 지원하는 기능이 포함되어있다고 얘기합니다.  

즉, `@MockBean` 과 `@SpyBean` 를 사용할 시에 TestContext 가 변경되어 캐싱을 하지 못하는것입니다.

테스트를 진행하며 행위를 Mocking 하기 위해 `@MockBean` 과 `@SpyBean` 어노테이션을 많이 사용하실텐데요.  
이 기능을 사용하면서도 __`ApplicationContext`__ 를 캐싱하는 방법에 대해서 공유드리겠습니다.

<br />

## @MockBean, @SpyBean 으로부터 __`ApplicationContext`__ 캐싱하기


### 실제 Bean 과 MockBean / SpyBean 을 같이 쓰고싶을때는 ?

### 방법 1.

### 방법 2.

#### __reference__

- https://docs.spring.io/spring-framework/reference/testing/testcontext-framework/ctx-management/caching.html