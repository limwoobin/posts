#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/orm-bulk-insert)

![bulk-insert-image1]()

<br />

# __Spring Batch Writer 성능 개선하기 (feat. Bulk Insert)__

안녕하세요, 이번에는 Spring Batch 에서 Writer 의 성능을 개선한 사례에 대해 공유드리려 합니다.

제가 담당하는 정산 프로젝트에는 하루에 수만건에서 많으면 수십만건의 데이터를 처리하고 생성하고 있는데요.  
서비스가 커지고 처리하는 데이터가 많아지면서 배치의 처리속도가 점점 저하되는 이슈가 생겼습니다.  
그래서 Spring Batch 의 성능 개선이 필요했었는데요. 

그 중에서 배치의 Writer 에 대해 개선하기 위해 고민하고 적용했던 과정을 공유드리려 합니다.


## __문제점__

정산시스템은 Spring Batch + MySQL + JPA 의 환경으로 구성되어있습니다.  
그리고 Writer 는 JPA 에서 제공하는 기능인 `saveAll()` 메소드를 통해 데이터를 저장하고 있는데요.

이때, ID 전략이 auto-increment 전략인 경우 Hibernate 에서는 Batch Insert 를 지원하지 않다는 문제가 있습니다.  
auto-increment 전략의 경우 Insert Query 를 실행하기 전에는 ID 값을 알 수 없고 실행을 해야만 알 수 있는데요, 이 전략이 Hibernate 에서 채택한 __쓰기 지연(transactional write-behind)__ 전략과 충돌하기 때문입니다.

##### __https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#batch-session-batch-insert__

> @Id  
  @GeneratedValue(strategy = GenerationType.IDENTITY)  
  private Long id;

<br />

그래서 Writer 에서 `saveAll()` 을 사용하게 되면 데이터의 수 만큼 DB I/O 가 발생하고 있었습니다. 연관 관계의 엔티티까지 포함하면 더 많은 수의 I/O 가 발생하고 있었는데요.  
I/O 의 증가는 서버와 DB 에 부담을 주고 성능에 대해서도 저하를 일으키고 있었습니다.

그래서 Batch Insert 를 이용해 DB I/O 횟수를 줄여 성능을 개선하기로 하였습니다.

## __개선하기__

__BulkWriter__ 를 처리하는 것에 대해 중점적으로 고민한 부분은 아래 두가지였습니다.

1. __OneToMany 관계에서 연관된 테이블까지 Batch Insert 처리가 가능해야함.__
2. __어느정도(?) TypeSafe 해야한다.__

일반적으로 연관관계가 있는 경우 __Batch Insert__ 를 할때는 테이블의 PK 값을 모르기에  
저장하기 전까지는 연관 테이블에서 FK 값을 설정할 수 없습니다. 그래서 Insert Bulk 처리가 어려운데요.

위 두가지 부분을 준수하면서 Bulk Insert 처리를 진행한 과정에 대해 하나씩 살펴보겠습니다. 

## __연관 테이블이 존재하는 경우 BulkInsert 적용하기__

이 부분은 아래 블로그를 참고하여 __`SELECT LAST_INSERT_ID()`__ 를 이용하기로 하였습니다.  
https://helloworld.kurly.com/blog/bulk-performance-tuning/

예제 코드와 같이 살펴보겠습니다.

__Team.java__
```java
@Entity
@Table(name = "teams")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Team {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(name = "name")
  private String name;

  @Column(name = "status")
  @Enumerated(EnumType.STRING)
  private TeamStatus status;

  @Embedded
  private Location location;

  @OneToMany(mappedBy = "team", cascade = CascadeType.ALL)
  private List<Project> projects = new ArrayList<>();

  @Builder
  public Team(String name,
              TeamStatus status,
              Location location) {
    this.name = name;
    this.status = status;
    this.location = location;
  }
}
```

<br />

__Location.java__
```java
@Getter
@Embeddable
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Location {

  @Column(name = "country")
  private String country;

  @Column(name = "city")
  private String city;

  @Column(name = "addr")
  private String addr;

  @Column(name = "zip_code")
  private int zipCode;

  @Column(name = "etc")
  private String etc;

  @Builder
  public Location(String country, String city, String addr, int zipCode, String etc) {
    this.country = country;
    this.city = city;
    this.addr = addr;
    this.zipCode = zipCode;
    this.etc = etc;
  }
}
```

<br />


__Project.java__
```java
@Entity
@Table(name = "projects")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Project {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(name = "name")
  private String name;

  @Column(name = "project_code")
  private String projectCode;

  @Column(name = "level")
  private int level;

  @Column(name = "description")
  private String description;

  @Column(name = "status")
  @Enumerated(EnumType.STRING)
  private ProjectStatus status;

  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "team_id", nullable = false)
  private Team team;

  @Builder
  public Project(String name,
                 String projectCode,
                 int level,
                 String description,
                 ProjectStatus status,
                 Team team) {
    this.name = name;
    this.projectCode = projectCode;
    this.level = level;
    this.description = description;
    this.status = status;
    this.team = team;
  }

  public void addTeam(Team team) {
    this.team = team;
  }
}
```

<br />

다음과 같이 Team 과 Project 엔티티가 존재합니다. Team 과 Project 는 1:N 의 관계를 가지고 있습니다.

![bulk-insert-image2](https://private-user-images.githubusercontent.com/28802545/395841978-b840e207-150b-4288-930f-c052228edbb0.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MzQyNTE2MzEsIm5iZiI6MTczNDI1MTMzMSwicGF0aCI6Ii8yODgwMjU0NS8zOTU4NDE5NzgtYjg0MGUyMDctMTUwYi00Mjg4LTkzMGYtYzA1MjIyOGVkYmIwLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDEyMTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQxMjE1VDA4Mjg1MVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWUxYTI1YjhhN2RkZmU3NmIzZTMzOTI1OGUyNDk3OWJlMTJmODU0MzY2M2YwYjNiYTZmNjBkNTQ2YmRkNDY2NzUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.9d_V8XgMZN653rwA5kUfz1LFPtxcbJnD6VwE0GnHI98)

<br />

__TeamBatchRepository.java__
```java
@Repository
public class TeamBatchRepository {
  private final JdbcTemplate jdbcTemplate;

  private static final String LAST_INSERT_ID_SQL = "SELECT LAST_INSERT_ID()";

  public TeamBatchRepository(JdbcTemplate jdbcTemplate) {
    this.jdbcTemplate = jdbcTemplate;
  }

  @Transactional
  public void saveAll(List<Team> teams) {
    // 1. teams 테이블 bulk insert
    String teamSql = "INSERT INTO teams(name, status, country ...) VALUES (?, ? ...)";
    jdbcTemplate.batchUpdate(teamSql, ...);

    // 2. lastInsertId 조회
    Long lastInsertId = jdbcTemplate.queryForObject(LAST_INSERT_ID_SQL, Long.class);

    // 3. lastInsertId 를 이용해 teams 의 PK 설정 
    // lastInsertId 를 기반으로 Team 의 PK 를 알 수 있음
    for (int i = 0; i < team.size(); i++) {
      Team team = teams.get(i);
      team.setId(lastInsertId + i);
    }

    // 4. teams 의 연관관계인 projects 를 펼치고 bulk insert
    List<Project> projects = teams.stream()
      .map(Team::getProjectsWithFk)
      .flatMap(List::stream)
      .toList();

    String projectSql = "INSERT INTO projects(name, project_code, ...) VALUES (?, ? ...)";
    jdbcTemplate.batchUpdate(projectSql, ...);
  }
}
```

여기서 주의깊게 보셔야 할 부분은 `SELECT LAST_INSERT_ID()` 입니다.  

`LAST_INSERT_ID` 는 MySQL 세션내에서 가장 마지막에 성공한 Insert Query 의 AutoIncrement 값을 가져옵니다.  
하지만 Bulk Insert 의 경우에는 다른 서버에서 동일한 명령문을 쉽게 재현할 수 있도록 하기 위해 첫번째 삽입된 행의 AutoIncrement 값을 반환하고 있습니다.

> 
If you insert multiple rows using a single INSERT statement, LAST_INSERT_ID() returns the value generated for the first inserted row only. The reason for this is to make it possible to reproduce easily the same INSERT statement against some other server.

##### reference: [mysql_docs](https://dev.mysql.com/doc/refman/8.4/en/information-functions.html#function_last-insert-id:~:text=This%20behavior%20ensures%20that%20each%20client%20can%20retrieve%20its%20own%20ID%20without%20concern%20for%20the%20activity%20of%20other%20clients%2C%20and%20without%20the%20need%20for%20locks%20or%20transactions)

<br />

```sql
INSERT INTO TABLE(col1) VALUES('test_1');
INSERT INTO TABLE(col1) VALUES('test_2');
INSERT INTO TABLE(col1) VALUES('test_3');
SELECT LAST_INSERT_ID;
-- 결과: 3

INSERT INTO TABLE(col1) VALUES('test_1'), ('test_2'), ('test_3');
SELECT LAST_INSERT_ID;
-- 결과: 1
```

하지만 여기서 `SELECT LAST_INSERT_ID` 에 대해 같은 테이블에 동시에 Insert Query 를 실행할 경우 동시성 이슈로 인해  
AutoIncrement 값이 꼬이지 않을까 염려하시는 분들도 계실텐데요.

MySQL 공식문서에서는 다음과 같이 설명되어있습니다.

> If the only statements executing are “simple inserts” where the number of rows to be inserted is known ahead of time, there are no gaps in the numbers generated for a single statement, except for “mixed-mode inserts”. However, when “bulk inserts” are executed, there may be gaps in the auto-increment values assigned by any given statement.

##### __https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html__

실행되는 쿼리가 삽입할 행 수를 미리 알고 있는 __"Simple Insert"__ 인 경우 단일 문에 대해 생성되는 숫자에 간격이 없다고 합니다.  
이는 __innodb_autoinc_lock_mode__ 의 설정이 어떤 값이든 보장됩니다.  
+ (혼합 모드에서는 id 에 간격이 생길수 있습니다)

즉, 저희가 실행하는 쿼리는 아래와 같은 삽입될 행의 수를 미리 알 수 있는 __"Simple Insert"__ 이기에 해당 테이블에 여러개의 쿼리가 동시에 Insert 되더라도 각 실행문의 auto-increment 값은 순차적인게 보장된다는 뜻입니다.

그렇기에, __SELECT LAST_INSERT_ID__ 를 통해 id 값을 가져와 사용하는데에는 안전하다고 볼 수 있는데요.

```sql
INSERT INTO TEAMS(col1) values('col1'), ('col2'), ('col3') ...
```

<br />

동시성 테스트를 통해 정말 연속된 값이 보장되는지 확인해보겠습니다.

테스트 시나리오는 간단합니다. 3개의 스레드가 각자의 name 을 3만건씩 동시에 저장하게 진행합니다.  
그리고 해당 name 이 들어간 row 가 3만개씩 연속적인지 확인해보겠습니다.

__BulkTest.java__
```java
@SpringBootTest
public class BulkTest {

  @Autowired
  private TeamBatchRepository teamBatchRepository;

  @Test
  void test() throws InterruptedException {
    Callable<String> task1 = () -> {
      teamBatchRepository.saveAll(generateTeams("team-1"));
      return "test-1";
    };

    Callable<String> task2 = () -> {
      teamBatchRepository.saveAll(generateTeams("team-2"));
      return "test-2";
    };

    Callable<String> task3 = () -> {
      teamBatchRepository.saveAll(generateTeams("team-3"));
      return "test-3";
    };

    ExecutorService executorService = Executors.newFixedThreadPool(3);
    executorService.invokeAll(List.of(task1, task2, task3));
    executorService.shutdown();
  }

  private List<Team> generateTeams(String name) {
    List<Team> teams = new ArrayList<>();
    Location location = Location.builder()
      .build();

    for (int i = 0; i < 30000; i++) {
      Team team = Team.builder()
        .name(name)
        .location(location)
        .build();

      teams.add(team);
    }

    return teams;
  }
}
```

![bulk-insert-image3](https://private-user-images.githubusercontent.com/28802545/396793630-9def58b2-be75-43d6-b1a3-3d6403ac5be0.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MzQ1MDczNDIsIm5iZiI6MTczNDUwNzA0MiwicGF0aCI6Ii8yODgwMjU0NS8zOTY3OTM2MzAtOWRlZjU4YjItYmU3NS00M2Q2LWIxYTMtM2Q2NDAzYWM1YmUwLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDEyMTglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQxMjE4VDA3MzA0MlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTg2NGU3ZDJlYzg2ZWY3OWI0YjU3YWQxYTU2Y2E2YjczNWVlN2Q1MDU5NmVmMmRmOGJiYmE5MTI2YjhmN2QxNmEmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.gwZyNnXPpoepIBdyKQdKlc_lQQKWchsZgazRMuv6Fww)

![bulk-insert-image4](https://private-user-images.githubusercontent.com/28802545/396803718-32016a04-7d27-4c97-b081-7035a7ddb899.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MzQ1MDczNDIsIm5iZiI6MTczNDUwNzA0MiwicGF0aCI6Ii8yODgwMjU0NS8zOTY4MDM3MTgtMzIwMTZhMDQtN2QyNy00Yzk3LWIwODEtNzAzNWE3ZGRiODk5LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDEyMTglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQxMjE4VDA3MzA0MlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTM5NDNiM2FhNDJmYzJmOTA3MjZhN2Y0NWY2Y2ZlNTgxMjk1OWM5ZTlhOTg1NDk5MDY1M2UzNjMzMjI0OWFkYjgmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.HGAeB0HQRfXNJfhuwycxuSwKgpmoyv3No3l5qVIWukE)

3만건, 5만건 모두 동일하게 PK 가 연속되는것을 확인할 수 있었습니다.  
이외에도 스레드 수를 늘려보기도 하고 다양한 경우의 테스트를 진행했으나 모두 연속된 값은 보장되었습니다.


<br />

## __TypeSafe 적용 + NamedParameterJdbcTemplate__

OneToMany 의 관계에서 연관 테이블까지 __BulkInsert__ 처리는 문제없이 진행되었습니다.  
하지만 실제 프로덕션에 적용하는 과정에서 문제가 있었습니다.

바로 사용성에 대한 문제였는데요. 저희 테이블에는 한 테이블에 많은 경우 100개 이상의 컬럼이 존재하는 테이블이 있었습니다.  
해당 경우에는 jdbcTemplate 기반이다보니 컬럼이 많아짐에 따라 보기에도 너무 불편하고  
컬럼과 value 의 순서에 강제받게 됩니다. 그렇기에 순서가 뒤바뀌거나 오타 등 개발자가 실수할 수 있는 부분이 너무 많았습니다.

그래서 순서에 의존적이지 않고 객체 기반으로 처리할 수 있는 방안을 고민하게 되었습니다.

### __1. Bulk Insert 전용 객체 도입__

우선은 객체 기반으로 BulkInsert 쿼리를 만들기 위해 전용 DTO 객체를 생성했습니다.

__TeamBatchDto.java__
```java
@Getter
@Setter
public class TeamBatchDto {

  @Transient
  private Long id;
  private String name;
  private TeamStatus status;
  private String country;
  private String city;
  private String addr;
  private int zip_code;
  private String etc;

  @Transient
  private List<ProjectBatchDto> projects;

  @Builder
  public TeamBatchDto(String name,
                      TeamStatus status,
                      String country,
                      String city,
                      String addr,
                      int zip_code,
                      String etc,
                      List<ProjectBatchDto> projects) {
    this.name = name;
    this.status = status;
    this.country = country;
    this.city = city;
    this.addr = addr;
    this.zip_code = zip_code;
    this.etc = etc;
    this.projects = projects;
  }

  public static TeamBatchDto from(Team team) {
    Location location = team.getLocation();
    List<ProjectBatchDto> projects = team.getProjects().stream()
      .map(ProjectBatchDto::from)
      .toList();

    return TeamBatchDto.builder()
      .name(team.getName())
      .status(team.getStatus())
      .country(location.getCountry())
      .city(location.getCity())
      .addr(location.getAddr())
      .zip_code(location.getZipCode())
      .etc(location.getEtc())
      .projects(projects)
      .build();
  }

  public List<ProjectBatchDto> getProjectsWithFk() {
    this.projects.forEach(it -> it.setTeamId(this.id));
    return this.projects;
  }
}
```

__ProjectBatchDto.java__
```java
@Getter
@Setter
public class ProjectBatchDto {
  @Transient
  private Long id;
  private String name;
  private String projectCode;
  private int level;
  private String description;
  private ProjectStatus status;
  private Long team_id;

  @Builder
  public ProjectBatchDto(String name,
                         String projectCode,
                         int level,
                         String description,
                         ProjectStatus status,
                         Long teamId) {
    this.name = name;
    this.projectCode = projectCode;
    this.level = level;
    this.description = description;
    this.status = status;
    this.team_id = teamId;
  }

  public static ProjectBatchDto from(Project project) {
    return ProjectBatchDto.builder()
      .name(project.getName())
      .projectCode(project.getProjectCode())
      .level(project.getLevel())
      .description(project.getDescription())
      .status(project.getStatus())
      .build();
  }
}
```

여기서 객체의 이름은 __snake_case__ 로 설정했습니다.  
일반적으로 객체의 네이밍 컨벤션은 __camelCase__ 지만 저희는 이 객체를 MySQL 의 컬럼명과 매핑시켜야 했기에  
예외적으로 __BulkInsert__ 전용 객체에서는 __snake_case__ 를 사용했습니다.

그리고 저장을 위한 객체이기에 기존 Embedded 객체같은 경우 모두 펼쳐 담았고 저장하지 않을 객체에 대해서는 @Transient  어노테이션을 달아주었습니다.  
@Transient 이 아닌 의미에 맞는 다른 어노테이션을 달아도 무방합니다.

### __2. NamedParameterJdbcTemplate 사용__ 

__NamedParameterJdbcTemplate__ 는 __JdbcTemplate__ 과 다르게 이름을 기반으로 파라미터를 바인딩하게 됩니다.  
그래서 순서에 의존적이지 않습니다.

```sql
-- jdbcTemplate
INSERT INTO table_name (col1, col2) VALUES (?, ?);

-- namedParameterJdbcTemplate
INSERT INTO table_name (col1, col2) VALUES (:col1, :col2);
```

예제 코드를 통해 살펴보겠습니다.

__TeamBatchRepository.java__
```java
@Repository
public class TeamBatchRepository {
  private final JdbcTemplate jdbcTemplate;
  private final NamedParameterJdbcTemplate namedParameterJdbcTemplate;

  private static final String LAST_INSERT_ID_SQL = "SELECT LAST_INSERT_ID()";

  public TeamBatchRepository(JdbcTemplate jdbcTemplate,
                             NamedParameterJdbcTemplate namedParameterJdbcTemplate) {
    this.jdbcTemplate = jdbcTemplate;
    this.namedParameterJdbcTemplate = namedParameterJdbcTemplate;
  }

  @Transactional
  public void saveAll(List<Team> teams) {
    // (1) Team 을 BulkInsert 전용 객체인 TeamBatchDto 로 변환
    List<TeamBatchDto> teamBatchDtos = teams.stream()
      .map(TeamBatchDto::from)
      .toList();

    // (2) namedParameterJdbcTemplate 에서 쓸 수 있게끔 sql 로 변환
    String sql = generateSql("teams", TeamBatchDto.class);

    // (3) CustomBeanPropertySqlParameterSource 을 통해 파라미터값 관리
    SqlParameterSource[] batchParams = teamBatchDtos.stream()
      .map(CustomBeanPropertySqlParameterSource::new)
      .toArray(SqlParameterSource[]::new);

    // (4) teams bulkIsnert 처리
    namedParameterJdbcTemplate.batchUpdate(sql, batchParams);

    // (5) lastInsertId 조회
    Long lastInsertId = jdbcTemplate.queryForObject(LAST_INSERT_ID_SQL, Long.class);
    // (6) 연관 테이블인 projects 저장
    saveProjects(teamBatchDtos, lastInsertId);
  }

  private void saveProjects(List<TeamBatchDto> teams, Long lastInsertId) {
    // (7) 저장한 teams 의 PK 설정
    for (int i = 0; i < teams.size(); i++) {
      TeamBatchDto team = teams.get(i);
      team.setId(lastInsertId + i);
    }

    // (8) projects 를 BulkInsert 전용객체로 변환 (FK 와 함께)
    List<ProjectBatchDto> projects = teams.stream()
      .map(TeamBatchDto::getProjectsWithFk)
      .flatMap(List::stream)
      .toList();

    String sql = generateSql(TableConstants.PROJECT_TABLE, ProjectBatchDto.class);
    SqlParameterSource[] batchParams = projects.stream()
      .map(CustomBeanPropertySqlParameterSource::new)
      .toArray(SqlParameterSource[]::new);

    // (9) projects bulkInsert 실행
    namedParameterJdbcTemplate.batchUpdate(sql, batchParams);
  }

  private String generateSql(String tableName, Class<?> clazz) {
    // (10) 테이블 명과 BulkInsert 전용 클래스를 객체로 받아 쿼리 생성
    List<Field> fields = Arrays.stream(clazz.getDeclaredFields())
      .filter(it -> !it.isAnnotationPresent(Transient.class))
      .toList();

    String columnsStr = fields.stream()
      .map(Field::getName)
      .collect(Collectors.joining(", "));
    String parameterColumnsStr = fields.stream()
      .map(it -> ":" + it.getName())
      .collect(Collectors.joining(", "));

    StringBuilder sb = new StringBuilder();
    sb.append("INSERT INTO ")
      .append(tableName)
      .append("(").append(columnsStr).append(") ")
      .append("VALUES ")
      .append("(").append(parameterColumnsStr).append(") ");

    return sb.toString();
  }
}
```

<br />

__CustomBeanPropertySqlParameterSource.java__
```java
public class CustomBeanPropertySqlParameterSource extends BeanPropertySqlParameterSource {

  public CustomBeanPropertySqlParameterSource(Object object) {
    super(object);
  }

  @Override
  public Object getValue(String paramName) throws IllegalArgumentException {
    Object value = super.getValue(paramName);

    // Enum 에 대해 처리하기 위해 Custom SqlParameterSource 생성
    // 파라미터에 대한 추가 관리가 필요하다면 직접 오버라이딩하여 구현하면 됨.
    if (value instanceof Enum) {
      return ((Enum<?>) value).name();
    }

    return value;
  }
}
```

<br />

__SQL Generate__
```java
private String generateSql(String tableName, Class<?> clazz) {
    List<Field> fields = Arrays.stream(clazz.getDeclaredFields())
      .filter(it -> !it.isAnnotationPresent(Transient.class))
      .toList();

    String columnsStr = fields.stream()
      .map(Field::getName)
      .collect(Collectors.joining(", "));
    String parameterColumnsStr = fields.stream()
      .map(it -> ":" + it.getName())
      .collect(Collectors.joining(", "));

    StringBuilder sb = new StringBuilder();
    sb.append("INSERT INTO ")
      .append(tableName)
      .append("(").append(columnsStr).append(") ")
      .append("VALUES ")
      .append("(").append(parameterColumnsStr).append(") ");

    return sb.toString();
  }
```

Team 과 Project 에 대해 BulkInsert 전용 객체로 전환 후 NamedParameterJdbcTemplate 를 이용해 저장하고 있습니다.  
그리고 리플렉션을 이용해 BulkInsert 전용 클래스를 받아 SQL 을 만들게 됩니다.

아까 전용 Dto 에서 sane_case 를 이용해 변수명을 선언한 이유가 여기에 있는데요.  
필드의 변수명과 테이블 컬럼명을 맞춰주기 위함입니다.  
그리고 __clazz.getDeclaredFields()__ 을 통해 필드 목록을 가져와 저장할 필드에 대해 SQL 을 만들게 됩니다.  
그러면 다음과 같이 컬럼 순서에 의존하지 않고 객체 기반의 Insert Query 를 만들수 있었습니다.

```sql
INSERT INTO teams (name, status, country ...) VALUES (:name, :status, :country ...);
```

해당 코드를 적용하기전 가장 크게 고민했던것은 리플렉션으로 엔티티를 기반으로 쿼리를 만드는건 어떨지 고민했었습니다.  
하지만, @Embedded 와 여러 어노테이션에 대한 리플렉션 처리가 필요했었습니다.  
불가능하지는 않았지만 코드의 책임이 과하게 많아지는 느낌을 받았고 무엇보다 JPA 엔티티를 BulkInsert 하기 위해  
그런 방식으로 사용하는것은 엔티티의 영속성 성격과 맞지 않는다는 느낌을 받았습니다.

그래서 BulkInsert 전용 객체를 만드는게 더 효율적일거라 판단하여 이 방향으로 진행하게 되었습니다.


## __검증하기__

테스트 코드를 통해 id 가 정상적으로 매핑되는지 확인해보겠습니다.

__BatchInsertTest.java__
```java
@SpringBootTest
class BatchInsertTest {

  @Autowired
  private TeamBatchRepository teamBatchRepository;

  @Autowired
  private TeamRepository teamRepository;

  @Test
  void test() {
    // given
    Team team = TeamFixture.TEAM("team-1");
    team.addLocation(LocationFixture.LOCATION_GURO());
    team.addProject(ProjectFixture.PROJECT_1());

    Team team2 = TeamFixture.TEAM("team-2");
    team2.addLocation(LocationFixture.LOCATION_GANGNAM());
    team2.addProject(ProjectFixture.PROJECT_2());
    team2.addProject(ProjectFixture.PROJECT_3());

    Team team3 = TeamFixture.TEAM("team-3");
    team3.addLocation(LocationFixture.LOCATION_GURO());
    team3.addProject(ProjectFixture.PROJECT_4());
    team3.addProject(ProjectFixture.PROJECT_5());
    team3.addProject(ProjectFixture.PROJECT_6());

    // then
    teamBatchRepository.saveAll(List.of(team, team2, team3));

    List<Team> teams = teamRepository.findAllWithProjects();

    List<Project> projects = teams.stream()
      .map(Team::getProjects)
      .flatMap(List::stream)
      .collect(Collectors.toList());

    assertThat(teams).hasSize(3);
    assertThat(projects).hasSize(6);

    assertAll(
      () -> assertThat(teams.get(0).getId()).isEqualTo(projects.get(0).getTeam().getId()),
      () -> assertThat(teams.get(1).getId()).isEqualTo(projects.get(1).getTeam().getId()),
      () -> assertThat(teams.get(1).getId()).isEqualTo(projects.get(2).getTeam().getId()),
      () -> assertThat(teams.get(2).getId()).isEqualTo(projects.get(3).getTeam().getId()),
      () -> assertThat(teams.get(2).getId()).isEqualTo(projects.get(3).getTeam().getId()),
      () -> assertThat(teams.get(2).getId()).isEqualTo(projects.get(3).getTeam().getId())
    );
  }
}
```

3개의 Team 에 대해 각각 1,2,3개의 프로젝트가 존재하는 데이터를 넣어보겠습니다.  
그리고 프로젝트의 team_id FK 에는 위에 설정한대로 team 의 id 가 정상적으로 들어가있어야 합니다.

![bulk-insert-image5](https://private-user-images.githubusercontent.com/28802545/396838421-13193e9e-da87-4c81-8502-cf0b26e962f9.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MzQ1MTM0NTcsIm5iZiI6MTczNDUxMzE1NywicGF0aCI6Ii8yODgwMjU0NS8zOTY4Mzg0MjEtMTMxOTNlOWUtZGE4Ny00YzgxLTg1MDItY2YwYjI2ZTk2MmY5LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDEyMTglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQxMjE4VDA5MTIzN1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTBlNDY5Y2MyODY0OTZjZTZhOTBmZWU3YTBlMmEzYWY2YjQ4MWY4ZWE0NTkzMjc1N2M0YWIxMTQ0NjUyYzQ5YWUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.Hs9kHAzEefcLBlfK0q0G5A6zeu2Ku5ZTi7fDCCmwBik)

다음과 같이 Team 과 Project 가 정상적으로 매핑된것을 확인할 수 있습니다.  
그리고 실제 로그상으로도 Insert Query 가 BulkInsert 형태로 실행된것을 볼 수 있습니다.

![bulk-insert-image6](https://private-user-images.githubusercontent.com/28802545/396839374-073a1ec2-ea0b-494c-b4f7-133083958170.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MzQ1MTM1OTUsIm5iZiI6MTczNDUxMzI5NSwicGF0aCI6Ii8yODgwMjU0NS8zOTY4MzkzNzQtMDczYTFlYzItZWEwYi00OTRjLWI0ZjctMTMzMDgzOTU4MTcwLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDEyMTglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQxMjE4VDA5MTQ1NVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTQ0OGM4NWZhMjg2MDMwNmMyMTQwNmJlMzQ5NWQzNjgzYzUxODdlODE2OGQwYmNhOTEzNTY0MmJlY2YyYTUwMzImWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.PA79wuu0027Kh8dzf1_4-Uh6Yyw_0VSPRcQKfHlUUhE)

<br />

## 유의사항

### __rewriteBatchedStatements=true__

BulkInsert 를 적용했는데 실제 확인해보면 쿼리가 단건으로 나가는 경우가 있습니다.  
이때는 MySQL 의 jdbc 설정에 해당 속성을 지정해주어야 Bulk Insert 로 동작합니다.

기본값은 false 입니다.

```yml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/test?rewriteBatchedStatements=true
```

### __max_allowed_packet__

MySQL 에는 __max_allowed_packet__ 이라는 설정값이 있습니다. 이 값은 MySQL 서버로 한번에 보낼 수 있는 최대 패킷 사이즈입니다.  
기본값은 버전별로 다르지만 다음과 같습니다.
- 5.7: 4194304 (4MB)
- 8.0: 67108864 (64MB)

BulkInsert 의 경우 데이터가 많아지면 해당 패킷 사이즈를 넘게 될 수도 있습니다.  
그렇기에 BulkInsert 를 사용하는 경우 상황에 맞게 max_allowed_packet 사이즈에 대한 고려가 필요합니다.  
만약 이 사이즈를 초과하는 경우 PacketTooBigException 가 발생할 수 있습니다.

```sql
show global variables like 'max_allowed_packet'
```

지금까지 읽어주셔서 감사합니다!~

<br />

#### __reference__

- https://helloworld.kurly.com/blog/bulk-performance-tuning/
- https://dev.mysql.com/doc/refman/8.4/en/information-functions.html
- https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html