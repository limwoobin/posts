# 테스트 성능 및 정합성 개선하기(with ContextCaching, TestContainer)

![test-improvement-1](https://private-user-images.githubusercontent.com/28802545/384930115-9c86ad4b-75ea-41d4-997a-e31ff1122373.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MzEzMzAzNTgsIm5iZiI6MTczMTMzMDA1OCwicGF0aCI6Ii8yODgwMjU0NS8zODQ5MzAxMTUtOWM4NmFkNGItNzVlYS00MWQ0LTk5N2EtZTMxZmYxMTIyMzczLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDExMTElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQxMTExVDEzMDA1OFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTUyMjBhNmY5MDczOGFhNmQxZDJjY2YxZDQ0ZTFlNTI5YWM2MmI5ZDU2OGFmNzk4ZDljZmQ4NGIxZTViY2NlNDMmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.NGqAIksfvEHtBKEJoDbzwx6jK0iFozGQl7pK76o2qow)

# 개요 및 문제점

안녕하세요 이번에는 사내에서 담당하고있는 프로젝트의 테스트 성능과 정합성을 개선한 사례에 대해 공유드리려 합니다.  
프로젝트에는 현재 약 __1700+__ 정도의 테스트 케이스 가 존재하는데요.(현재 기준)  
실제 돈과 관련된 도메인을 담당하다 보니 보다 꼼꼼한 시나리오 테스트가 필요하고 사이드 이펙트를 방지하기 위해 개발하며 테스트를 자주 실행하게 됩니다.  
이때, 테스트 케이스가 증가함에 따라 테스트 코드에 여러 문제점들이 존재했습니다.

그래서 테스트 코드를 개선하기로 마음먹게 되는데요.  
테스트 코드에는 크게 아래와 같은 문제점이 있었습니다.

1. __일관성 없는 테스트 결과__
2. __테스트 성능 이슈__

문제에 대해 조금 더 살펴보겠습니다.

## __1. 일관성 없는 테스트 결과__

동일한 테스트 코드여도 프로덕션 코드 변경이 없음에도 실행할때마다 결과가 상이한 테스트 케이스가 존재했습니다.  
문제는 RDS 와 Redis 가 격리된 환경이 아니어서 발생했었습니다.

테스트 환경의 __RDS/Redis__ 의 경우 __dev__ 환경의 __RDS/Redis__ 를 그대로 사용하고 있었습니다.  
이는 테스트 환경 구축시에는 편리할 수 있지만 오직 테스트 환경에서만 사용되는 도구가 아니기에 격리된 환경이 아닙니다.

그래서 데이터에 대해 검증을 하더라도 다른 개발자가 개발을 하며 dev 환경의 데이터를 조작하게 되면 실패할수도 있었습니다.  
그렇기에 격리된 환경의 __RDS/Redis__ 구축이 필요했습니다.

## __2. 테스트 성능 이슈__

정산에는 총 1600+ 이상의 테스트 케이스가 존재한다고 말씀드렸었는데요.  
이때 테스트 코드에서의 __@MockBean, @SpyBean__ 사용으로 인한 __ApplicationContext__ 초기화가 굉장히 많았습니다.

전체 테스트 코드의 수행시간을 측정하면 %m 이상의 시간이 소요되었습니다.  
테스트를 통해 예외 케이스를 검증하며 개발을 경우에 오랜 시간이 소요되기에 생산성에 영향이 있었고  
심적으로도 테스트 수행에 부담이 느껴졌습니다.

<br />

# 개선 방향

위 두가지 문제점에 대해 개선이 필요하다고 생각했고 개선 방향에 대해 고민하였습니다.  

그래서 __"일관성 없는 테스트"__ 에 대해서는 __TestContainer__ 를 이용해  
__RDS/Redis__ 를 테스트를 위한 격리된 환경을 만들어 사용하기로 했습니다.

그리고 __"테스트 성능 이슈"__ 에 대해서는 초기화되는 __ApplicationContext__ 에 대해 __ContextCaching__ 을 이용해  
성능을 개선하기로 했습니다.

<br />

## TestContainer 

__"일관성 없는 테스트 결과"__ 문제에 대해서는 __TestContainer__ 를 이용해 해결했습니다.

### 도입 배경

테스트를 위한 격리된 __RDS/Redis__ 환경을 구축하는데에는 다른 선택지들도 존재했습니다.  
__RDS__ 의 경우 __H2 DB__ 를 활용할 수 있고, __Redis__ 는 __EmbeddedRedis__ 를 활용할 수 있습니다.

하지만 __H2__ 의 경우 테스트로 검증하기 어려운 부분이 존재한다는것이 문제였습니다.  
저희는 __RDS__ 로 __MySQL__ 을 사용하고 있었는데요, __H2__ 와 __MySQL__ 에는 여러 문법에서 차이점이 존재합니다.  
그렇기에 __QueryDSL, Mybatis__ 를 이용한 __NativeQuery__ 를 사용하는 저희 프로젝트에서는 
작성한 쿼리 문법이 __H2__ 와 호환이 되지 않는 경우에는 검증이 불가능했습니다.

그리고 __EmbeddedRedis__ 의 경우에는 컨텍스트 로딩 이슈와 지원이 중단되었다는 이슈가 존재했었습니다.

<br />
<hr />

### TestContainer 적용 과정

__Redis__ 를 __TestContainer__ 로 적용하는 과정과 __EmbeddedRedis__ 를 사용하지 않은 자세한 내용에 대해서는 해당 글을 참고부탁드립니다.  
(https://devoong2.tistory.com/entry/Springboot-Redis-%ED%85%8C%EC%8A%A4%ED%8A%B8-%ED%99%98%EA%B2%BD-%EA%B5%AC%EC%B6%95%ED%95%98%EA%B8%B0-Embedded-Redis-TestContainer)

<br />

__MySQL__ 에 대해 TestContainer 를 도입하는 과정과 겪었던 이슈에 대해 공유드리겠습니다.

__build.gradle__
```gradle
testImplementation group: 'org.testcontainers', name: 'testcontainers', version: '1.19.2'
testImplementation 'org.testcontainers:junit-jupiter:1.19.2'
testImplementation 'org.testcontainers:mysql:1.19.2'
testImplementation("mysql:mysql-connector-java:8.2.0")
```

우선 TestContainer 사용을 위해 gradle 에 의존성을 주입합니다.  
그리고 테스트 DB 로는 MySQL 을 사용하니 MySQL 설정도 진행합니다.

<br />

__application.yml__
```yml
spring:
  profiles:
    active: test
  datasource:
    driver-class-name: org.testcontainers.jdbc.ContainerDatabaseDriver
    url: jdbc:tc:mysql:8.2.0://test
    username: root
    password:
    initialization-mode: always
    schema: classpath:/sql/schema.sql
    data: classpath:/sql/data.sql

  jpa:
    database: mysql
    database-platform: org.hibernate.dialect.MySQL5InnoDBDialect
    hibernate:
      ddl-auto: none
    properties:
      hibernate:
        format_sql: true
        show_sql: true
```

그리고 yml 파일에서 TestContainer 설정을 진행합니다.  
- __driver-class-name: org.testcontainers.jdbc.ContainerDatabaseDriver__ 
	- TestContainer 에서 제공하는 드라이브버를 사용합니다.
- __url: jdbc:tc:mysql:8.2.0://__ 
	- __jdbc:tc__ 형식은 __Testcontainers__ 가 이 URL을 통해 자동으로 컨테이너를 생성, 실행, 종료하도록 만듭니다.
	- __8.2.0__ 은 사용할 MySQL 의 버전을 지정합니다.
- __schema: classpath:/sql/schema.sql__ : 스키마를 정의합니다.
- __data: classpath:/sql/data.sql__ : 사용할 초기 데이터를 정의합니다.

다음과 같이 설정하면 TestContainer 로 MySQL 을 사용할 준비는 완료되었습니다.  
그리고 테스트를 실행하면 MySQL 과 Redis 컨테이너가 잘 동작한것을 확인할 수 있습니다.

![test-improvement-2](https://private-user-images.githubusercontent.com/28802545/384599914-a5a99929-1ffb-4d61-9713-30279b7b8507.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MzExNTI0NjYsIm5iZiI6MTczMTE1MjE2NiwicGF0aCI6Ii8yODgwMjU0NS8zODQ1OTk5MTQtYTVhOTk5MjktMWZmYi00ZDYxLTk3MTMtMzAyNzliN2I4NTA3LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDExMDklMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQxMTA5VDExMzYwNlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWY3ZDNkNTEyMTI2N2Q3ODJjZmM3YmIzZTc5NzAwMTk5OWI4ZDE4YzY4NjUwZGI5MTk3OWRhZjExMTZmN2Q5N2QmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.M9FVzdyTomZp4GgC91AlSFX1FY9VRbPRxJvtUn8wzMo)

<br />

하지만 이때 두가지 문제점이 있었습니다.  

#### __DDL 중복 실행

__application.yml__ 에 설정한 __schema.sql__ 파일은 __ApplicationContext__ 가 로드되는 시점에 실행이 됩니다.  
만약 테스트 중 __ApplicationContext__ 가 2번 이상 초기화 되는 경우에는 DDL 문법이 2번 이상 실행되기에
아래와 같은 에러가 발생했습니다.

```java
Failed to execute SQL script statement #1 of class path resource [sql/schema.sql]: ... Table 'ORDERS' already exists
```

테이블 생성시 테이블이 이미 존재한다는 메시지였습니다. 그래서 DDL 문을 아래와 같이 변경하여 해결했습니다.

```sql
-- as-is
CREATE TABLE ORDERS {
}

-- to-be
CREATE TABLE IF NOT EXISTS ORDERS {
}
```

<br />

#### __SQL_MODE 가 다른 이슈__

실제 운영환경의 MySQL 과 테스트를 위해 컨테이너로 띄운 MySQL 은 버전은 동일하더라도 내부 설정은 다를 수 있습니다.  
설정에 따라 테스트가 실패하는 케이스가 존재할 수 있습니다.

저의 환경에서는 쿼리 수행시 아래와 같은 에러메시지가 존재했었는데요.(각 설정마다 발생하는 에러케이스는 다를 수 있습니다.)

> this is incompatible with sql_mode=only_full_group_by

운영환경과 TestContainer MySQL 의 SQL_MODE 를 동일한 설정으로 맞춰주어 해결하였습니다.  
설정은 data.sql 을 이용하였습니다.

__data.sql__
```sql
SET SESSION sql_mode = 'STRICT_TRANS_TABLES';
```

<br />

## __Application ContextCaching__

__"테스트 성능 이슈"__ 에 대해서는 Bean 을 의존이 필요한 테스트에 대해서 각 모듈별로 하나의 __ApplicationContext__ 를 사용할 수 있게끔 처리했습니다.

저희 프로젝트는 gradle 멀티모듈 환경으로 구성이 되어있는데요. 총 API, Batch, Core 3개의 모듈이 존재하고 있습니다.  
그래서 Bean 을 의존하는 테스트를 수행하는 경우 각 모듈별로 하나의 __ApplicationContext__ 을 이용해 테스트를 수행하도록 처리하였습니다.  
이전에 한번 글을 발행한적이 있었는데요, 여기서 주제에 맞게 다시 한번 공유드리겠습니다. (https://devoong2.tistory.com/entry/Spring-ContextCaching-%EC%9C%BC%EB%A1%9C-Test-%EC%84%B1%EB%8A%A5-%EA%B0%9C%EC%84%A0%ED%95%98%EA%B8%B0-MockBean-SpyBean)

저희 프로젝트의 테스트 코드에는 __@MockBean, @SpyBean__ 가 사용되고 있었습니다.  
__@MockBean, @SpyBean__ 옵션은 컨텍스트를 오염시키기에 __ApplicationContext__ 가 새로 초기화됩니다.  
그렇기에 __ApplicationContext__ 초기화가 많아짐에 따라 속도가 저하되는 이슈가 존재했습니다.

그래서 테스트시 Mocking 을 활용할 수 있고, 테스트 간 격리도 지켜지며 __ApplicationContext__ 는 초기화되지 않고 캐싱하여 사용을 할 수 있는 방안이 필요했습니다.  
그래서 테스트 코드 내에서 사용하던 __@MockBean, @SpyBean__ 을 __TestConfiguration__ 으로 설정하여 이슈를 해결하였습니다.

__MyService.java__
```java
@Service
public class MyService {

	public void doSomething() {
		// doSomething ...
	}
}
```

__MyServiceTestConfiguration.java__
```java
@TestConfiguration
public class __MyServiceTestConfiguration {

  @Autowired
  private MyService testMyService;

  @Bean
  @Primary
  public MyService testMyService() {
    return testMyService;
  }

  @Bean
  public MyService mockMyService() {
    return Mockito.mock(MyService.class);
  }

  @Bean
  public MyService spyMyService() {
    return Mockito.spy(testMyService);
  }
}
```

__SettlementTest.java__
```java
@SpringBootTest
@ContextConfiguration(classes = MyServiceTestConfig.class)
public @interface IntegrationTest {

	@Autowired
	private MyService myService;

	@Autowired
	@Qualifier("mockMyService")
	private MyService mockMyService;

	@Autowired
	@Qualifier("spyMyService")
	private MyService spyMyService;

	@Test
	void test() {
		// doSomething...
	}
 }
```

다음과 같이 선언하여 __MyService__ 를 선언 방법에 따라 실제 Bean 혹은 MockBean, SpyBean 을 __ApplicationContext__ 초기화 없이 사용할 수 있도록 처리했습니다.

테스트 환경에서는 MyService 타입의 Bean 이 여러개가 선언되있기에(Mock, Spy, 실제Bean) 해당 타입을 못찾는 이슈가 존재합니다.  
그래서 __@Qualifier("mockMyService")__ 로 명시적으로 Bean 을 가져오도록 하였고 Config 에서는 __@Primary__ 설정을 통해 기본 옵션을 지정하였습니다.

### __불필요한 Bean 의존 제거__

기존의 테스트 코드에는 불필요하게 Bean 을 의존하는 테스트 또한 상당히 존재했습니다.  
해당 테스트 케이스에 대해서는 Bean 의존을 제거하여 테스트 하도록 처리하였습니다.

예시 코드를 통해 공유드리겠습니다.

__MyService.java__
```java
@Service
public class MyService {

  private final MyComponent myComponent;

  public MyService(MyComponent myComponent) {
    this.myComponent = myComponent;
  }

  public String findComponentValue() {
    return myComponent.getValue();
  }

}
```

__MyComponent.java__
```java
@Component
public class MyComponent {
 
  public String getValue() {
    return "MyComponent";
  }
}
```

__MyService__ 의 __findComponentValue()__ 메소드는 __MyComponent__ 에서 선언한 __getValue()__ 를 통해  
__"MyComponent"__ 를 리턴한다고 가정해보겠습니다.

__ExampleTest.java__
```java
@SpringBootTest
class ExampleTest {

  @MockBean
  private MyComponent myComponent;

  @Autowired
  private MyService myService;

  @Test
  void test() {
    Mockito.when(myComponent.getValue()).thenReturn("MockValue");
    
    String result = myService.findComponentValue();
    Assertions.assertEquals(result, "MockValue");
  }
}
```

그리고 다음과 같이 __MyComponent__ 를 __MockBean__ 으로 선언해 __getValue()__ 메서드를 Mocking 한 테스트 코드입니다.  
이러한 테스트 코드는 Mocking 은 가능하지만 불필요하게 __ApplicationContext__ 를 띄우게 됩니다.

이러한 Bean 의존이 필요하지 않은 테스트들은 생성자 주입방식의 이점을 살려 테스트 코드를 개선할 수 있습니다.

__ExampleTestV2.java(개선버전)__
```java
public class ExampleTestV2 {

  private MyComponent myComponent;

  private MyService myService;

  @BeforeEach
  void setUp() {
    myComponent = Mockito.mock(MyComponent.class);
    myService = new MyService(myComponent);
  }

  @Test
  void test() {
    Mockito.when(myComponent.getValue()).thenReturn("MockValue");

    String result = myService.findComponentValue();
    Assertions.assertEquals(result, "MockValue");
  }
}
```

다음과 같이 처리를 하면 __ApplicationContext__ 를 의존하지 않고 가볍고 빠르게 동일한 테스트를 수행할 수 있습니다.


<br />

# __개선 결과__

### __개선 전__

![test-improvement-3](https://private-user-images.githubusercontent.com/28802545/384921071-e07cc5ee-9340-4080-b4d1-774e7533a25e.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MzEzMjg0NDYsIm5iZiI6MTczMTMyODE0NiwicGF0aCI6Ii8yODgwMjU0NS8zODQ5MjEwNzEtZTA3Y2M1ZWUtOTM0MC00MDgwLWI0ZDEtNzc0ZTc1MzNhMjVlLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDExMTElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQxMTExVDEyMjkwNlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWRjMzA3N2E0MjM1YjkxMWM0MDkwMzY0ZWExNTk3OTRhZmNhMGNhMWUxYjVjYmNhMjMzNWViNDBmMzZjYzNmYTgmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.dsY6iRZJmJ64cl-5yu6h9YhSeEBx5E6SsJ5IS2-2p88)

총 1,200 개 갸량의 테스트 케이스를 수행하였고 Intellij 에서 보여주는 시간은 2min 4s 가 나왔습니다.  
이 시간은 ApplicationContext 가 로드되는 시간을 제외한 실제 테스트 수행에 대한 시간을 보여주고 있는데요.  
그래서 타이머를 통해 측정했을때는 ApplicationContext 가 로드되는 시간까지 포함되어 총 7분 47초가 나왔습니다.  
(현재는 M2 Pro 노트북을 사용하고 있는데 이전 Macbook Pro 를 이용해 측정했을때는 10분 이상이 소요되었습니다.)


### __개선 후__

![test-improvement-4](https://private-user-images.githubusercontent.com/28802545/384921060-ec2865d3-f46c-4512-8b8c-236ac50e4c23.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MzEzMjg3ODMsIm5iZiI6MTczMTMyODQ4MywicGF0aCI6Ii8yODgwMjU0NS8zODQ5MjEwNjAtZWMyODY1ZDMtZjQ2Yy00NTEyLThiOGMtMjM2YWM1MGU0YzIzLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDExMTElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQxMTExVDEyMzQ0M1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWNhNzdjNjMyMzFkMWM3NGU5ZmUyZDU3Y2I4YmY5OTRmMmQ1ODVmOTE2MjFkNDE3NWM4ZTY0MzIwOTFmNzliMWUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.mnIMF6fu-sPVfHFntiq3BqWF_-9Gs7Ypi_VaTAzDY1Q)

개선 전 후의 시간 차이가 꽤 있어 그동안 500개 이상의 테스트 케이스가 추가되었습니다.  
그럼에도 불구하고 실제 테스트 수행시간은 2m 정도에서 17s 로 감소하였습니다.  
그리고 ApplicationContext 가 로드되는 시간을 포함하더라도 총 수행시간이 1/4 정도 감소한것을 확인할 수 있었습니다.

<br />

## __후기__

테스트의 정합성과 성능을 동시에 개선하는것을 목표로 잡았었는데요.  
TestContainer 를 도입하기전에는 사실 테스트 시간이 더 오래걸리지 않을까 고민했었습니다.

TestContainer 를 사용하면 편리하게 환경을 구성할 수 있지만 처음 컨테이너를 띄우고  
테스트 환경을 로드하는데 비용이 들어 기존 방법보다 더 오래걸리기 때문인데요.  
하지만 ContextCaching 을 적극적으로 활용했기에 전체적인 테스트 속도는 더 향상되었습니다.

개별로 하나의 테스트를 수행하는데에는 큰 체감은 아니지만 TestContainer 가 더 오래걸리긴 합니다.  
하지만 개선을 하고나니 전체적인 TestSuite 를 수행하는데 부담이 줄어들었습니다.  
테스트 실행시 항상 일관된 결과를 얻을 수 있다는 부분에서도 테스트 신뢰도가 올라간것 같습니다.

아직도 개선해야할 부분이 많은데 기회가 되면 다음 개선 내용도 공유드릴 수 있도록 하겠습니다.  
감사합니다.