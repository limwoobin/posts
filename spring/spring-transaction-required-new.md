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
시나리오는 다음과 같이 정의하겠습니다.

```
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
```

<br />

## 에제 코드

- __spring boot 3.2.1__
- __Java17__
- __mysql8__

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

```shell
curl --location 'http://localhost:8080/teams' \
--header 'Content-Type: application/json' \
--data 'test-team'
```

다음 curl 을 이용해서 각 시나리오별로 API 테스트를 실행해보겠습니다.

### 해피 케이스

```
- 부모 트랜잭션 실행
- 자식 트랜잭션 실행(REQUIRES_NEW)
```

위의 해피 케이스에 대한 예상 시나리오는 `Team`, `TeamHistory` 두 테이블 모두 정상적으로 데이터가 생성되어야 합니다.

다음과 같이 정상적으로 반영된것을 확인할 수 있습니다.

<br />

![transaction-propagation-1](https://user-images.githubusercontent.com/28802545/294747935-4e23b759-cfa1-4dd0-901c-526c2ac0bee3.png)

<br />
<hr />

### 예외 케이스 1

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
    throw new RuntimeException();
  }
}
```

다음 코드의 실행결과를 확인해보겠습니다.

![transaction-propagation-2](https://user-images.githubusercontent.com/28802545/294748262-c820f5b1-e858-4eb6-b8bc-649c1c8df5f9.png)

<br />

아무 데이터도 존재하지 않는걸 보니 부모, 자식 트랜잭션 모두 롤백 되었습니다.  
실제 응답도 실패로 다음과 같이 왔습니다.

```
{
    "timestamp": "2024-01-07T09:09:06.109+00:00",
    "status": 500,
    "error": "Internal Server Error",
    "path": "/teams/v2"
}
```

__`REQUIRES_NEW`__ 는 분명 별도의 트랜잭션으로 동작하는것으로 알고있는데요 부모 트랜잭션까지 롤백이 발생했습니다.  
우선 다음 예외 케이스도 모두 실행해 보겠습니다.

<br />
<hr />

### 예외 케이스 2

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

![transaction-propagation-3](https://user-images.githubusercontent.com/28802545/294749092-99d0395a-00f0-489f-bf49-b9ea40af8756.png)

<br />

자식 트랜잭션에서 저장한 `TeamHistory` 는 저장되었지만 예외가 발생한 부모 트랜잭션에서 저장한 `Team` 은 롤백되었습니다.  
예외의 시점에 따라 롤백되는 데이터도 달라지게 되는것을 확인할 수 있습니다.


<br />
<hr />

### 예외 케이스 3
```
- 부모 트랜잭션 실행
- 자식 트랜잭션 실행(REQUIRES_NEW)
- 자식 트랜잭션 예외 발생 후 캐치
- 자식 트랜잭션 예외 캐치 후 현재 트랜잭션만 롤백
```

코드를 다음과 같이 수정하겠습니다.

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
  public void saveHistory3(Team team) {
    try {
      throw new RuntimeException();
    } catch (Exception e) {
      log.error("error message {}", e.getMessage());
    }
  }
}
```


## 후기

#### reference
- https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Propagation.html
- https://oingdaddy.tistory.com/28