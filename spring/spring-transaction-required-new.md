#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/transaction-propagation)

<br />

# __트랜잭션 REQUIRES_NEW 옵션에서 예외와 롤백__

## Overview

이번에는 스프링 환경에서 __`@Transactional`__ 의 __`Propagation`__ 옵션인 __`REQUIRES_NEW`__ 와 해당 옵션을 사용할때의 예외/롤백에 대해 알아보겠습니다.

우선 트랜잭션 전파(`Transaction Propagation`)에 대해 먼저 알아보겠습니다.  
트랜잭션 전파는 한 트랜잭션이 실행중에 다른 트랜잭션을 실행할 경우 어떻게 동작할지를 __결정__ 하는것입니다.

트랜잭션 전파의 종류는 다음과 같습니다.

- __REQUIRED (Default)__
- __REQUIRES_NEW__
- __SUPPORTS__
- __NOT_SUPPORTED__
- __MANDATORY__
- __NEVER__
- __NESTED__

<br />

전파옵션의 기본값은 __`REQUIRED`__ 이며 별도의 트랜잭션이 실행되어도 부모 트랜잭션에 종속되어 실행됩니다.

저희가 알아볼 __`REQUIRES_NEW`__ 옵션은 부모 트랜잭션과는 별도의 트랜잭션을 생성하여 동작합니다.  

그렇다면 __`REQUIRES_NEW`__ 를 사용한 자식 트랜잭션에서 예외가 발생한다면 어떻게 될까요?  
부모와 자식 트랜잭션 모두 롤백이 될까요, 아니면 별도의 트랜잭션이니 자식 트랜잭션만 롤백이 될까요?

예제 코드를 통해 실제 자식 트랜잭션에서 예외가 발생했을때 어떻게 동작할지 한 번 알아보겠습니다.  

<br />

## 에제 코드

- __Spring Boot 3.2.1__
- __Java 17__
- __MySQL 8__

예제는 부모 트랜잭션에서 __`Team`__ 객체를 저장하고 자식 트랜잭션에서 __`TeamHistory`__ 에 저장/변경 이력이 쌓이게끔 했습니다.

<br />

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

  private Team(String name) {
    this.name = name;
  }

  public static Team from(String name) {
    return new Team(name);
  }
}
```

<br />

__TeamHistory.java__
```java
@Getter
@Entity(name = "team_histories")
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class TeamHistory {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column
  private String name;

  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "team_id", nullable = false)
  private Team team;

  private TeamHistory(Team team) {
    this.name = team.getName();
    this.team = team;
  }

  public static TeamHistory from(Team team) {
    return new TeamHistory(team);
  }
}
```

<br />

__TeamService.java__
```java
@Service
@RequiredArgsConstructor
public class TeamService {
  private final TeamRepository teamRepository;
  private final TeamHistoryService teamHistoryService;

  @Transactional
  public void save(String name) {
    Optional<Team> optionalTeam = teamRepository.findByName(name);
    if (optionalTeam.isPresent()) {
      throw new RuntimeException();
    }

    Team team = Team.from(name);
    teamRepository.save(team);
    teamHistoryService.saveHistory(team);
  }
}
```

<br />

__TeamHistoryService.java__
```java
@Service
@RequiredArgsConstructor
public class TeamHistoryService {
  private final TeamHistoryRepository teamHistoryRepository;

  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void saveHistory(Team team) {
    TeamHistory teamHistory = TeamHistory.from(team);
    teamHistoryRepository.save(teamHistory);
  }
}
```

<br />

__TeamController.java__
```java
@RestController
@RequiredArgsConstructor
@RequestMapping(value = "/teams")
public class TeamController {

  private final TeamService teamService;

  @PostMapping
  public ResponseEntity<Void> saveTeam(@RequestBody String name) {
    teamService.save(name);
    return new ResponseEntity<>(HttpStatus.CREATED);
  }
}
```

<br />

## 시나리오 검증

시나리오는 다음과 같이 정의하겠습니다.

<!-- ```
해피 케이스
- 부모 트랜잭션 실행
- 자식 트랜잭션 실행(REQUIRES_NEW)

예외 케이스 1
- 부모 트랜잭션 실행
- 자식 트랜잭션 실행(REQUIRES_NEW)
- 자식 트랜잭션 예외 발생

예외 케이스 2
- 부모 트랜잭션 실행
- 자식 트랜잭션 실행(REQUIRES_NEW)
- 부모 트랜잭션 예외 발생

예외 케이스 3
- 부모 트랜잭션 실행
- 자식 트랜잭션 실행(REQUIRES_NEW)
- 자식 트랜잭션 예외 발생 후 캐치
- 자식 트랜잭션 예외 캐치 후 현재 트랜잭션만 롤백
``` -->

```shell
curl --location 'http://localhost:8080/teams' \
--header 'Content-Type: application/json' \
--data 'test-team'
```

다음 curl 을 이용해서 각 예외상황을 한번 테스트 해보겠습니다.

<br />

다음과 같은 시나리오를 가정하고 예제 코드를 만들어 테스트 해보겠습니다.

```
- 부모 트랜잭션 실행
- 자식 트랜잭션 실행(REQUIRES_NEW)
- 자식 트랜잭션 예외 발생
```

예외 발생을 위해 코드를 다음과 같이 수정하겠습니다.

__TeamService.java__
```java
@Service
@RequiredArgsConstructor
public class TeamService {
  private final TeamRepository teamRepository;
  private final TeamHistoryService teamHistoryService;

  @Transactional
  public void save(String name) {
    Optional<Team> optionalTeam = teamRepository.findByName(name);
    if (optionalTeam.isPresent()) {
      throw new RuntimeException();
    }

    Team team = Team.from(name);
    teamRepository.save(team);
    teamHistoryService.saveHistory(team);
  }
}
```

<br />

__TeamHistoryService.java__
```java
@Service
@RequiredArgsConstructor
public class TeamHistoryService {
  private final TeamHistoryRepository teamHistoryRepository;

  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void saveHistory(Team team) {
    TeamHistory teamHistory = TeamHistory.from(team);
    teamHistoryRepository.save(teamHistory);

    throw new RuntimeException();
  }
}
```

다음 코드의 실행결과를 확인해보겠습니다.

![transaction-propagation-1](https://user-images.githubusercontent.com/28802545/294748262-c820f5b1-e858-4eb6-b8bc-649c1c8df5f9.png)

<br />

```
 curl --location 'http://localhost:8080/teams/v2' \
--header 'Content-Type: application/json' \
--data 'test-team'

{
    "timestamp": "2024-01-07T09:09:06.109+00:00",
    "status": 500,
    "error": "Internal Server Error",
    "path": "/teams/v2"
}
```

에러가 발생하였고 `Team, TeamHistory` 테이블 모두 아무런 데이터가 들어가지 않았습니다.

__`REQUIRES_NEW`__ 는 분명 별도의 트랜잭션으로 동작하는것으로 알고있는데요 부모 트랜잭션까지 롤백이 발생했습니다. 왜 부모 트랜잭션도 같이 롤백 되었을까요? 

__이유는 간단합니다. 바로 자식 트랜잭션에서 발생한 예외가 부모 트랜잭션에게 까지 전파되었기 때문입니다.__  
자바에서는 예외가 발생하면 중간에 별도의 예외처리를 하지 않는 이상 콜 스택을 따라 처음 호출한곳 까지 예외가 전파되는데요. 

`TeamService` 의 `teamHistoryService.saveHistory(team);` 로 호출한 부분까지 예외가 전파되어 결국 부모 트랜잭션 내에서도 예외가 발생하여 롤백이 진행된 것이었습니다.

![transaction-propagation-2](https://user-images.githubusercontent.com/28802545/295545946-73873dcc-a027-4072-99e7-3b401a0d96db.png)

<br />

로그를 확인해보면 자식 트랜잭션인 `saveHistory` 에서 RuntimeException 을 발생시키고 이후 해당 예외가  
부모 트랜잭션은 `save` 까지 전파되어 롤백이 진행되는것을 확인할 수 있습니다.

트랜잭션에서는 실제 예외가 발생하면 해당 트랜잭션에 롤백 마킹을 하게 되는데요  
`TransactionAspectSupport.java` 클래스의 `completeTransactionAfterThrowing` 메소드를 한번 살펴보겠습니다.

![transaction-propagation-3](https://user-images.githubusercontent.com/28802545/295547782-7edd7b4c-3f1e-45e1-838f-edb295fdc86e.png)

<br />

__rollbackOn method__
![transaction-propagation-4](https://user-images.githubusercontent.com/28802545/295547821-42c792ee-6bba-4e9b-99e4-cd1626d37692.png)

해당 예외가 트랜잭션에서 롤백하게 지정된 예외인지 확인하고 맞다면 롤백을 진행합니다.  
이렇게 자식 트랜잭션에서 발생한 예외가 부모 트랜잭션까지 전파되었기 때문에 부모 트랜잭션도 같이 롤백되게 되었습니다.


<br />
<hr />


그러면 이번에는 조금 바꿔서 예외를 던지지 않고 캐치하면 어떻게 될까요?

```
- 부모 트랜잭션 실행
- 자식 트랜잭션 실행(REQUIRES_NEW)
- 자식 트랜잭션 예외 발생 후 캐치
```

테스를 위해 코드를 다음과 같이 수정하겠습니다.

__TeamService.java__
```java
@Service
@RequiredArgsConstructor
public class TeamService {
  private final TeamRepository teamRepository;
  private final TeamHistoryService teamHistoryService;

  @Transactional
  public void save(String name) {
    Optional<Team> optionalTeam = teamRepository.findByName(name);
    if (optionalTeam.isPresent()) {
      throw new RuntimeException();
    }

    Team team = Team.from(name);
    teamRepository.save(team);
    teamHistoryService.saveHistory(team);
  }
}
```

<br />

__TeamHistoryService.java__
```java
@Service
@RequiredArgsConstructor
public class TeamHistoryService {
  private final TeamHistoryRepository teamHistoryRepository;

  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void saveHistory(Team team) {
    try {
      TeamHistory teamHistory = TeamHistory.from(team);
      teamHistoryRepository.save(teamHistory);

      throw new RuntimeException();
    } catch (Exception e) {
      log.error("error message {}", e.getMessage());
    }
  }
}
```

<br />

![transaction-propagation-5](https://user-images.githubusercontent.com/28802545/294747935-4e23b759-cfa1-4dd0-901c-526c2ac0bee3.png)

![transaction-propagation-6](https://user-images.githubusercontent.com/28802545/295557729-d99fb2a5-9e2a-4cc1-a684-f21d1e38470f.png)

<br />

예외처리를 하여 자식 트랜잭션, 부모 트랜잭션에서 예외를 던지지 않으니 롤백없이 커밋이 된 것을 확인할 수 있습니다.

<br />
<hr />

그렇다면 예외 발생에 대해 순서를 조금 바꿔보면 어떨까요?

```
- 부모 트랜잭션 실행
- 자식 트랜잭션 실행(REQUIRES_NEW)
- 부모 트랜잭션 예외 발생
```

예외 발생을 위해 코드를 다음과 같이 수정하겠습니다.

__TeamService.java__
```java
@Service
@RequiredArgsConstructor
public class TeamService {
  private final TeamRepository teamRepository;
  private final TeamHistoryService teamHistoryService;

  @Transactional
  public void save(String name) {
    Optional<Team> optionalTeam = teamRepository.findByName(name);
    if (optionalTeam.isPresent()) {
      throw new RuntimeException();
    }

    Team team = Team.from(name);
    teamRepository.save(team);
    teamHistoryService.saveHistory(team);

    throw new RuntimeException();
  }
}
```

<br />

__TeamHistoryService.java__
```java
@Service
@RequiredArgsConstructor
public class TeamHistoryService {
  private final TeamHistoryRepository teamHistoryRepository;

  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void saveHistory(Team team) {
    TeamHistory teamHistory = TeamHistory.from(team);
    teamHistoryRepository.save(teamHistory);
  }
}
```

다음 코드의 실행결과를 확인해보겠습니다.

```
curl --location 'http://localhost:8080/teams/v3' \
--header 'Content-Type: application/json' \
--data 'test-team'

{
    "timestamp":"2024-01-10T11:55:44.007+00:00",
    "status":500,
    "error":"Internal Server Error",
    "path":"/teams/v3"
}
```

![transaction-propagation-7](https://user-images.githubusercontent.com/28802545/294749092-99d0395a-00f0-489f-bf49-b9ea40af8756.png)

![transaction-propagation-8](https://user-images.githubusercontent.com/28802545/295561925-10445014-0a1f-46ee-91b2-ecfb2f1f89e3.png)

<br />

자식 트랜잭션에서 저장한 `TeamHistory` 는 저장되었지만 예외가 발생한 부모 트랜잭션에서 저장한 `Team` 은 롤백되었습니다.  

자식 트랜잭션에서는 예외가 발생하지 않아 롤백되지 않았지만 이후 부모 트랜잭션에서 예외가 발생했을때의 시점에는 자식 트랜잭션은 이미 커밋이 완료된 상태이기 때문입니다.

<br />
<hr />

### 후기

이렇게 트랜잭션의 __`REQUIRES_NEW`__ 옵션과 예외상황에 따라 어떻게 동작하는지, 그리고 롤백이 어떻게 되는지  
몇가지 케이스를 통해 알아보았습니다.

__`REQUIRES_NEW`__ 옵션을 사용하게 된다면 롤백을 어느 범위까지 발생시킬지, 예외 전파를 어떻게 해야할지에 대한 고민도 할 수 있었습니다.  
비즈니스에 적용할때도 성격에 맞게 위 부분을을 같이 고민하면서 사용하면 더 안전하고 효율적이게 사용할 수 있을것 같다는 생각이 들었습니다.

감사합니다.

#### reference
- https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Propagation.html
- https://techblog.woowahan.com/2606/