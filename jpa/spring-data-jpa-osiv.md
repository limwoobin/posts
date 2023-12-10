#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/osiv-example)

<br />

## **OSIV(Open Session In View) 사용 시 주의사항**

이번엔 JPA/Hibernate 에서 사용되는 개념인 OSIV(Open Session In View) 에 대해 알아보겠습니다.  
OSIV 는 영속성 컨텍스트를 View 영역까지 열어둔다는 기능입니다. 즉, View 레이어에서도 지연로딩과 같은 영속성 컨텍스트의 특징을 사용할 수 있다는 이야기입니다.

`Spring Boot` 에서의 OSIV 는 기본적으로 활성상태입니다. 이를 사용하지 않기 위해서는 명시적으로 설정을 해주어야 합니다.  
> spring.jpa.open-in-view=false

OSIV 패턴은 현재 패턴인지 안티패턴인지에 대해 논쟁의 여지가 존재하는데요 아래 링크를 참고해보시면 좋습니다.  

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

### OSIV - Kafka Consumer 예제

간단하게 카프카 설정을 하고 메시지를 한번 받아보겠습니다. 자세한 설정은 [**github**](https://github.com/limwoobin/blog-code-example/tree/master/osiv-example) 을 참고해주세요.  

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
`Consumer` 는 View 영역은 아니지만 여기에서는 어떻게 동작할지 궁금했습니다. 바로 메시지를 소비해보겠습니다.  

![osiv-image-4](https://user-images.githubusercontent.com/28802545/287429249-fc6505ef-6253-47c1-a050-bb4081de3daa.png)

영속성 컨텍스트가 컨슈머까지 열려있지 않고 서비스 레이어에서 트랜잭션이 종료되면서 영속성 컨텍스트도 같이 종료되어 `LazyInitializationException` 이 발생했습니다.

그렇다면 `GraphQL` 에서는 OSIV가 잘 동작할까요? `GraphQL` 도 직접 확인해보겠습니다.

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

그렇다면 `RestAPI, GraphQL` 로 컨트롤러를 호출한것과 카프카 컨슈머로 메시지를 소비한것과 어떤 차이가 있어서 OSIV 동작의 유무를 판단하는 것일까요?  
OSIV 의 동작원리에 대해 조금 더 자세하 알아보아야 할 것 같습니다.

## OSIV - 동작 원리



<br />

## **reference**

https://www.baeldung.com/spring-open-session-in-view  
https://spring.io/guides/gs/graphql-server/  
https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/orm/hibernate5/support/OpenSessionInViewInterceptor.html
