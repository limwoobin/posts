#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/osiv-example)

<br />

## **OSIV(Open Session In View) 동작원리 및 주의사항**

이번엔 JPA/Hibernate 에서 사용되는 개념인 OSIV(Open Session In View) 에 대해 알아보겠습니다.  
OSIV 는 영속성 컨텍스트를 View 영역까지 열어둔다는 기능입니다. 즉, View 레이어에서도 지연로딩과 같은 영속성 컨텍스트의 특징을 사용할 수 있다는 이야기입니다.

`Spring Boot` 에서의 OSIV 는 기본적으로 활성상태입니다. 그리고 default 로 어플리케이션을 실행하게 되면 다음과 같은 경고메시지를 만나볼 수 있습니다.

![osiv-image-6](https://user-images.githubusercontent.com/28802545/289344266-ae527a8a-111e-44c6-8eff-a59c22d402ef.png)

```
spring.jpa.open-in-view is enabled by default. Therefore, database queries may be performed during view rendering. Explicitly configure spring.jpa.open-in-view to disable this warning

---

spring.jpa.open-in-view는 기본적으로 활성화되어 있습니다. 따라서 보기 렌더링 중에 데이터베이스 쿼리가 수행될 수 있습니다. 이 경고를 비활성화하려면 spring.jpa.open-in-view를 명시적으로 구성하세요.
```

OSIV 를 의도하지 않고 default 로 사용하는 경우에는 의도치 않은 쿼리가 발생할 수 있다고 경고를 해주고 있습니다.

OSIV 를 사용하지 않기 위해서는 아래와 같이 명시적으로 설정을 해주어야 합니다.  

> spring.jpa.open-in-view=false

추가로 OSIV 패턴은 현재 패턴인지 안티패턴인지에 대해 논쟁의 여지가 존재하는데요 아래 링크를 참고해보시면 좋습니다.  

#### __https://github.com/spring-projects/spring-boot/issues/7107__

<br />

### OISV - RestAPI Controller 예제

간단한 예제 코드를 작성하여 OSIV 가 정말로 잘 동작하는지 한번 확인해보겠습니다. 

__Team.java__
```java
@Getter
@Entity(name = "teams")
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Team {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column
  private String name;

  @OneToMany(mappedBy = "team", fetch = FetchType.LAZY)
  private List<Member> members = new ArrayList<>();
}
```

<br />

__Member.java__
```java
@Getter
@Entity(name = "members")
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Member {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column
  private String name;

  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "team_id", nullable = false)
  private Team team;
}

```

Team 과 Member 를 1:N 관계로 설정했습니다.

__TeamService.java__
```java
@Service
@RequiredArgsConstructor
public class TeamService {
  private final TeamRepository teamRepository;

  @Transactional(readOnly = true)
  public Team findTeam(Long id) {
    return teamRepository.findById(id)
      .orElseThrow(() -> new RuntimeException("Not Found Team"));
  }
}
```

<br />

__TeamController.java__
```java
@RestController
@RequiredArgsConstructor
@RequestMapping(value = "/teams")
@Sl4j
public class TeamController {
  private final TeamService teamService;

  @GetMapping(value = "/{id}")
  public ResponseEntity<TeamResponse> findTeam(@PathVariable Long id) {
    Team team = teamService.findTeam(id);
    log.info("member size = {}", team.getMembers().size());
    TeamResponse response = new TeamResponse(team);

    return new ResponseEntity<>(response, HttpStatus.OK);
  }
}
```

컨트롤러와 서비스 코드를 작성했습니다. osiv 설정은 따로 건드리지 않았으니 default 설정에 따라 활성화가 되어있습니다.

데이터는 다음과 같이 하나의 팀에 3명의 멤버가 존재합니다.

![osiv-image-1](https://user-images.githubusercontent.com/28802545/287427813-f22559ce-60f1-49b5-80c4-09990aa51da6.png)

![osiv-image-2](https://user-images.githubusercontent.com/28802545/287427816-9a3c7369-5c61-4c6d-bf64-66ccb7e942ef.png)

<br />

`@Transactional` 은 서비스 레이어에 선언되어있고 그 바깥인 컨트롤러에서 `System.out.println(team.getMembers());` 를 호출하여 Team 이 가지고 있는 Members 객체를 초기화 하겠습니다.  

영속성컨텍스트가 정말 뷰 레이어까지 열려있다면 지연로딩도 문제가 없어야 합니다.

![osiv-image-3](https://user-images.githubusercontent.com/28802545/287428360-5edb2551-9086-4412-a275-f3e2c6d5dc82.png)

정상적으로 트랜잭션 바깥에서도 프록시 객체인 Members 를 지연로딩 하는데에 성공하였습니다.

컨트롤러에서도 문제없이 OSIV 를 잘 사용했으니 이번엔 카프카 컨슈머에서 한번 사용해보겠습니다.

<br />

### OSIV - Kafka Consumer 예제

간단하게 카프카의 컨슈머를 설정하고 메시지를 받아보겠습니다. 자세한 내용은 [**github**](https://github.com/limwoobin/blog-code-example/tree/master/osiv-example) 을 참고해주세요.  

__KafkaConsumer.java__
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class KafkaConsumer {

  private final TeamService teamService;

  @KafkaListener(
    topics = "${spring.kafka.topics.osiv-test}",
    groupId = "${spring.kafka.consumer.group-id}",
    containerFactory = "containerFactory"
  )
  public void consume(@Payload Long id, Acknowledgment ack) {
    Team team = teamService.findTeam(id);

    log.info("member size = {}", team.getMembers().size());
    ack.acknowledge();
  }
}
```

OSIV 는 View 영역까지 영속성 컨텍스트를 열어주는 기능이라 했습니다.  
`Consumer` 는 View 영역은 아니지만 여기에서는 어떻게 동작할지 궁금했습니다. 바로 메시지를 컨슈밍 해보겠습니다.  

![osiv-image-4](https://user-images.githubusercontent.com/28802545/287429249-fc6505ef-6253-47c1-a050-bb4081de3daa.png)

영속성 컨텍스트가 컨슈머까지 열려있지 않고 서비스 레이어에서 트랜잭션이 종료되면서 영속성 컨텍스트도 같이 종료되어 `LazyInitializationException` 이 발생했습니다.

그렇다면 `GraphQL` 에서는 OSIV가 잘 동작할까요? `GraphQL` 도 직접 확인해보겠습니다.

<br />

### OISV - GraphQL 예제

`GraphQL` 에서는 OSIV 가 잘 동작하는지 확인해보기 위해 간단하게 `GraphQL` 예제코드를 작성해보겠습니다.

__build.gradle__
```gradle
implementation 'org.springframework.boot:spring-boot-starter-graphql'
```

<br />

__resources/graphql/teams.graphqls__
```graphqls
type Query {
    teamById(id: ID): TeamResponse
}

type TeamResponse {
    id: ID,
    name: String,
    members: [MemberResponse]
}

type MemberResponse {
    id: ID,
    name: String
}
```

<br />

__GqlTeamController.java__
```java
@RestController
@RequiredArgsConstructor
@Slf4j
public class GqlTeamController {
  private final TeamService teamService;

  @QueryMapping
  public TeamResponse teamById(@Argument Long id) {
    Team team = teamService.findTeam(id);
    log.info("member size = {}", team.getMembers().size());

    return new TeamResponse(team);
  }
}
```

<br />

작성한 `GraphQL` 컨트롤러도 마찬가지로 View 영역이기에 영속성 컨텍스트가 유지될것으로 예상됩니다. 이제 아래 쿼리로 `GraphQL` 을 호출해보겠습니다. 

__query__
```
query teams {
    teamById(id: 1) {
        id,
        name,
        members {
            id,
            name
        }
    }
}
```

<br />

아래 결과와 같이 Member 프록시 객체에 대해 정상적으로 지연로딩에 성공한 것을 알 수  있습니다.

![osiv-image-5](https://user-images.githubusercontent.com/28802545/287432179-9e25d3b1-4bea-45f5-a1a0-42bf4cb48091.png)

그렇다면 `RestAPI, GraphQL` 로 API를 호출한것과 `Kafka Consumer` 로 메시지를 소비한것.  
둘 사이에 어떤 차이가 있어서 OSIV 동작의 유무를 판단하는 것일까요?  
OSIV 의 동작원리에 대해 조금 더 자세하 알아보아야 할 것 같습니다.

<br />

## OSIV - 동작 원리

OSIV 의 동작원리를 알아보기 전에 설정을 다시 한번 확인해보겠습니다.  
앞서 말했듯이 OSIV 는 `application.yml` 에서 설정할 수 있는데요. true 와 default 는 활성화, false 는 비활성화입니다.

> spring.jpa.open-in-view=false

그렇다면 우선 해당 설정을 키워드로 하여금 설정을 사용하는 곳을 먼저 찾아보아야 할 것 같습니다.  

해당 설정을 사용하는 곳을 따라가니 __`JpaBaseConfiguration.java`__ 파일이 있습니다. 해당 클래스의 내부를 살펴보니 OSIV 에 관련된 Bean 을 생성하는 객체가 존재합니다.

__JpaWebConfiguration.java__
```java
@Configuration(proxyBeanMethods = false)
	@ConditionalOnWebApplication(type = Type.SERVLET)
	@ConditionalOnClass(WebMvcConfigurer.class)
	@ConditionalOnMissingBean({ OpenEntityManagerInViewInterceptor.class, OpenEntityManagerInViewFilter.class })
	@ConditionalOnMissingFilterBean(OpenEntityManagerInViewFilter.class)
	@ConditionalOnProperty(prefix = "spring.jpa", name = "open-in-view", havingValue = "true", matchIfMissing = true)
	protected static class JpaWebConfiguration {

		private static final Log logger = LogFactory.getLog(JpaWebConfiguration.class);

		private final JpaProperties jpaProperties;

		protected JpaWebConfiguration(JpaProperties jpaProperties) {
			this.jpaProperties = jpaProperties;
		}

		@Bean
		public OpenEntityManagerInViewInterceptor openEntityManagerInViewInterceptor() {
			if (this.jpaProperties.getOpenInView() == null) {
				logger.warn("spring.jpa.open-in-view is enabled by default. "
						+ "Therefore, database queries may be performed during view "
						+ "rendering. Explicitly configure spring.jpa.open-in-view to disable this warning");
			}
			return new OpenEntityManagerInViewInterceptor();
		}

		@Bean
		public WebMvcConfigurer openEntityManagerInViewInterceptorConfigurer(
				OpenEntityManagerInViewInterceptor interceptor) {
			return new WebMvcConfigurer() {

				@Override
				public void addInterceptors(InterceptorRegistry registry) {
					registry.addWebRequestInterceptor(interceptor);
				}

			};
		}

	}
```

<br />

`JpaWebConfiguration.java` 파일을 천천히 살펴보겠습니다.  
처음 Bean 을 로드하는 부분을 보면 `@ConditionalOnProperty(prefix = "spring.jpa", name = "open-in-view", havingValue = "true", matchIfMissing = true)` 이 부분이 보입니다.  

`@ConditionalOnProperty` 어노테이션은 조건에 따라 Bean 을 생성하는 어노테이션입니다.

1. yml 설정의 `spring.jpa` prefix 의 `open-in-view` 옵션의 값을 가져온다.
2. `open-in-view` 의 값이 `havingValue` 옵션과 같은 `true` 라면 Bean 을 생성한다.
3. `matchIfMissing` 가 `true` 이므로 매칭되는 설정이 없어도 Bean 을 생성한다.

해당 조건을 통해 `spring.jpa.open-in-view` 이 true 혹은 설정을 하지 않아도 default 로 OSIV 가 활성화 되는것이었습니다.

추가로 OSIV 설정을 명시적으로 하지 않은 경우 위에서 보았던 `WARN LOG` 를 찍어주는 부분도 해당 Bean 을 로드하는 시점에 담당하는것을 확인할 수 있습니다.

![osiv-image-6](https://user-images.githubusercontent.com/28802545/289344266-ae527a8a-111e-44c6-8eff-a59c22d402ef.png)

<br />

그리고 __`OpenEntityManagerInViewInterceptor.java`__ 해당 클래스를 Bean 으로 등록하고 이후 해당 Bean을 `InterceptorRegistry` 의 인터셉터로 등록하는 것을 볼 수 있습니다.  

이후 Bean 이 등록되면 __`OpenEntityManagerInViewInterceptor.java`__ 는  
`preHandle(), postHandle(), afterCompletion()` 메소드에서 `WebRequest.java` 을 인자로 받아 처리합니다.  

그래서 OSIV 가 `RestAPI, GraphQL` 요청시에는 인터셉터가 존재하였기에 정상동작했지만, `Kafka Consumer` 에서는 해당 인터셉터가 실행되지 않기때문에 OSIV 가 동작하지 않았던 것입니다.

<br />

그러면 __`OpenEntityManagerInViewInterceptor.java`__ 내부를 한번 살펴보겠습니다.




<br />



<br />

## **reference**

- https://www.baeldung.com/spring-open-session-in-view  
- https://spring.io/guides/gs/graphql-server/  
- https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/orm/hibernate5/support/OpenSessionInViewInterceptor.html
- https://junhyunny.github.io/spring-mvc/jpa/open-session-in-view/