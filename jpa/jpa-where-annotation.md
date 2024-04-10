![where-logo](https://private-user-images.githubusercontent.com/28802545/311180667-77940e83-11f0-4217-8212-57cf353669f0.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MDk4ODk0NzUsIm5iZiI6MTcwOTg4OTE3NSwicGF0aCI6Ii8yODgwMjU0NS8zMTExODA2NjctNzc5NDBlODMtMTFmMC00MjE3LTgyMTItNTdjZjM1MzY2OWYwLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAzMDglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMzA4VDA5MTI1NVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTJhOGNmNDE3OGFhZjgwZjEzNzM3NDI2YmQ5OTljZThkYmY1ZjZlZmExZDI0OGVjNmJlOTc4Yjk2Y2EzZjJiNWMmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.jthkT8kMyi9UWe9UyLjCMURAiyUfb5nmdceFJUpxpjM)

#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/jpa-where-annotation)

<br />

# JPA @Where 어노테이션 사용법

안녕하세요. 이번에는 JPA 의 __@Where__ 어노테이션의 사용 방법에 대해 알아보겠습니다.  
__@Where__ 을 이용하면 JPA, QueryDSL 을 사용할때 일괄적으로 조건을 추가할 수 있습니다.

예를 들면, 어떤 테이블에서 데이터 삭제시  __`soft delete`__ 로 처리하는 경우가 있다고 가정해보겠습니다.  

이때, 해당 테이블을 조회하는 경우 모든 쿼리 마다 삭제되지 않았다는 상태 조건을 추가해주어야 하는 번거로움이 있습니다.

이와 같은 경우에 굉장히 효율적으로 사용할 수 있습니다.

<br />

## @Where 예시, 사용법

여기 __Team__ 과 __Member__ 엔티티가 OneToMany 양방향 관계로 이루어져 있습니다.

![where-image1](https://private-user-images.githubusercontent.com/28802545/310961283-31bd3875-46c2-4405-884d-f8de044fb270.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MDk4MjkzMDIsIm5iZiI6MTcwOTgyOTAwMiwicGF0aCI6Ii8yODgwMjU0NS8zMTA5NjEyODMtMzFiZDM4NzUtNDZjMi00NDA1LTg4NGQtZjhkZTA0NGZiMjcwLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAzMDclMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMzA3VDE2MzAwMlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTY1YjA1YjMxYzE3OTYzZGVkMTY3YzFiMGMwMWNlN2Q0MGM3YjhlOTEyZGFkMWQwMDE5M2UwOTUyNjM3NDVmYWImWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.oa96mxRMoidNJoWvoWImhMA5QJAgsNwrIlTUYpGMGec)

__status__ 는 Enum 타입으로 __`ACTIVE, DISABLE`__ 로 __`soft delete`__ 상태를 구분해보겠습니다.  
삭제된 데이터의 status 는 __`DISABLE`__ 입니다.


__Status.java__
```java
@Getter
@AllArgsConstructor
public enum Status {
  ACTIVE,
  DISABLE;
}
```

<br />

__Team.java__
```java
@Getter
@Entity(name = "teams")
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Where(clause = "status = 'ACTIVE'")
public class Team {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column
  private String name;

  @OneToMany(mappedBy = "team", fetch = FetchType.LAZY)
  private List<Member> members = new ArrayList<>();

  @Enumerated(value = EnumType.STRING)
  private Status status;

  @Builder
  public Team(String name, List<Member> members, Status status) {
    this.name = name;
    this.members = members;
    this.status = status;
  }
}

```

<br />

__Member.java__
```java
@Getter
@Entity(name = "members")
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Where(clause = "status = 'ACTIVE'")
public class Member {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column
  private String name;

  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "team_id", nullable = false)
  private Team team;

  @Enumerated(value = EnumType.STRING)
  private Status status;

  @Builder
  public Member(String name, Team team, Status status) {
    this.name = name;
    this.team = team;
    this.status = status;
  }

  public void delete() {
    this.status = Status.DISABLE;
  }
}
```

<br />


__`@Where(clause = "status = 'ACTIVE'")`__ @Where 어노테이션은 다음과 같이 클래스에 선언해서 사용해보겠습니다.

그럼 이제 테스트를 통해서 케이스별로 실제 쿼리에 조건이 어떻게 들어가는지 확인해보겠습니다.

## @Where 테스트 코드

테스트 데이터는 다음과 같이 Team 하나에 Member 2개로 테스트를 진행해보겠습니다.

![where-image2](https://private-user-images.githubusercontent.com/28802545/311146511-09cd8f66-c5d2-49db-bb5a-c3751025ab68.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MDk4ODE5MTAsIm5iZiI6MTcwOTg4MTYxMCwicGF0aCI6Ii8yODgwMjU0NS8zMTExNDY1MTEtMDljZDhmNjYtYzVkMi00OWRiLWJiNWEtYzM3NTEwMjVhYjY4LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAzMDglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMzA4VDA3MDY1MFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTViM2VjZmI1MzU3ZTJhZGU2MzI4NmQyMjRlM2E1ZDNlM2U2OWJjNTIxZjgwMmM4NjQzZDc1YTRkNjMwNmRlNWEmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.Qunn_sOJO7o67tPZGT_Dmb-EzJxbG-0Fntv0kkSDll8)

<br />

### case1. 기본 엔티티 조회

다음과 같이 기본적으로 엔티티 조회시 쿼리가 어떻게 발생하는지 확인해보겠습니다.

```java
@Test
void test() {
  entityManager.flush();
  entityManager.clear();

  Team team = teamRepository.findById(1L);

  assertThat(team.isPresent()).isTrue();
}
```

![where-image3](https://private-user-images.githubusercontent.com/28802545/311147861-19be412b-097a-4a26-86df-224b4c3e26fd.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MDk4ODIyNzYsIm5iZiI6MTcwOTg4MTk3NiwicGF0aCI6Ii8yODgwMjU0NS8zMTExNDc4NjEtMTliZTQxMmItMDk3YS00YTI2LTg2ZGYtMjI0YjRjM2UyNmZkLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAzMDglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMzA4VDA3MTI1NlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWRiMDBlNjE1NzkzY2QyNmJlZTdlNWVlNDkzZGFlOGNkMGQxYTdlMGE5ZjBmYjgyODhjMmRlZjFhZTJmYWVjZDAmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.LKshiORZYOas24nwRjC9hdsEQH-QAmSwozSch_bF65c)

쿼리 메소드에는 아무 조건을 지정하지 않았지만 정상적으로 __`status = 'ACTIVE'`__ 조건이 들어간것을 확인할 수 있습니다.

### case2. Lazy Loading 조회

```java
@Test
void test() {
  entityManager.flush();
  entityManager.clear();

  Team team = teamRepository.findById(1L).get();
  List<Member> members = team.getMembers();

  assertThat(members.size()).isEqualTo(1);
}
```

![where-image4](https://private-user-images.githubusercontent.com/28802545/311148274-cf5a73ce-5001-47b7-bacd-e31889dffcce.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MDk4ODIzODgsIm5iZiI6MTcwOTg4MjA4OCwicGF0aCI6Ii8yODgwMjU0NS8zMTExNDgyNzQtY2Y1YTczY2UtNTAwMS00N2I3LWJhY2QtZTMxODg5ZGZmY2NlLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAzMDglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMzA4VDA3MTQ0OFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTE0MjJiMDUxMDMxYzc2ZTJlYjBkNDFmNDlkYmE1ZTQwM2YzNjdkZWIwMDljZmU3M2U3ZDhiZjJhZWZiMWNiNjkmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.lYA7LqsTBosT7_SkMxrCaDi6slu1KijBB41dY7DKeYM)

__LazyLoading__ 의 경우에도 team, member 에 모두 __`status = 'ACTIVE'`__ 조건이 정상적으로 들어갔네요.  

### case3. JPQL 조회

```java
@Test
void test() {
  List<Member> members = memberRepository.findAll();

  assertThat(members.size()).isEqualTo(1);
}
```

![where-image5](https://private-user-images.githubusercontent.com/28802545/311149051-57271ad1-7a74-47d3-8cd7-132fde63eee9.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MDk4ODI2MDcsIm5iZiI6MTcwOTg4MjMwNywicGF0aCI6Ii8yODgwMjU0NS8zMTExNDkwNTEtNTcyNzFhZDEtN2E3NC00N2QzLThjZDctMTMyZmRlNjNlZWU5LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAzMDglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMzA4VDA3MTgyN1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTAxMTU0OWE4YzRkNWI3MTBiNTc3Nzg0Yzg2ZDA1ZDU5MWI4MmQ0ZmQ2YjAzMjdkZGMzYTE3YTNkZDUwOThjM2MmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.jM2J0NGX9C5zaKfXWGJKjhgfAqPbugstp-q1cxK-rYQ)

JPQL 의 경우에도 조건이 정상적으로 포함된것을 확인할 수 있습니다.

### case4. QueryDSL 조회

```java
@Test
void test() {
  QTeam team = QTeam.team;

  Team result = jpaQueryFactory.selectFrom(team)
    .where(team.id.eq(1L))
    .fetchOne();

  assertThat(result.getId()).isEqualTo(1L);
}
```

![where-image6](https://private-user-images.githubusercontent.com/28802545/311153444-84360609-a438-4128-93b2-948245eafe05.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MDk4ODM2MDMsIm5iZiI6MTcwOTg4MzMwMywicGF0aCI6Ii8yODgwMjU0NS8zMTExNTM0NDQtODQzNjA2MDktYTQzOC00MTI4LTkzYjItOTQ4MjQ1ZWFmZTA1LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAzMDglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMzA4VDA3MzUwM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWM0MTdkZGI5ZGI3MzI1OGE4ODJiYjhkNmMwMDAyYzUxYTUzMjU4N2RmNjMwMWM2ZTY3NmNiNGU4YjllMWM2N2EmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.q-IUA8XzwScN7jDJs7Hy_Cj2dRnHZgsp-NTbirfcDXs)

QueryDSL 도 마찬가지로 잘 조회가 되네요.

### case5. QueryDSL Join 조회

```java
@Test
void test() {
  QTeam team = QTeam.team;
  QMember member = QMember.member;

  List<Member> result = jpaQueryFactory.selectFrom(member)
    .join(team).on(team.eq(member.team))
    .where(member.id.eq(1L))
    .fetch();

  assertThat(result.size()).isEqualTo(1L);
}
```

![where-image7](https://private-user-images.githubusercontent.com/28802545/311153842-9a8f9c5b-05a0-407c-8366-cd2bb2d3dcfb.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MDk4ODM2OTAsIm5iZiI6MTcwOTg4MzM5MCwicGF0aCI6Ii8yODgwMjU0NS8zMTExNTM4NDItOWE4ZjljNWItMDVhMC00MDdjLTgzNjYtY2QyYmIyZDNkY2ZiLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAzMDglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMzA4VDA3MzYzMFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWE4MDE3ZjVjNGM5NWM5NDA1YTljZTBkNWQ1NWE5M2Y3NzIwNGE0NWE5NTgxY2EyYzMyYWM2ZDljNjBiMDRmNzMmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.GyHUOKpcEF5RAmuxlvQZwiPSEyif6QHQHEtcq2_IIw4)

Team 과 Member 를 조인한 경우에도 두 엔티티 모두 조건에 status 가 잘 적용된 것을 보실 수 있습니다.

### casd6. QueryDSL DTO 조회

QueryDSL 에서 DTO 로 조회하는 경우도 살펴보겠습니다.

```java
// MemberDto.class
@Getter
public class MemberDto {
  private Long memberId;
  private String name;
  private Long teamId;

  public MemberDto(Long memberId, String name, Long teamId) {
    this.memberId = memberId;
    this.name = name;
    this.teamId = teamId;
  }
}

@Test
void test() {
  QTeam team = QTeam.team;
  QMember member = QMember.member;

  List<MemberDto> result = jpaQueryFactory.select(
    Projections.constructor(MemberDto.class,
      member.id,
      member.name,
      team.id
    ))
    .from(member)
    .join(team).on(team.eq(member.team))
    .where(member.id.eq(1L))
    .fetch();

  assertThat(result.size()).isEqualTo(1L);
}
```

![where-image8](https://private-user-images.githubusercontent.com/28802545/311155058-19e952be-cbc3-42be-822a-295a0e7f52f6.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MDk4ODM5MTEsIm5iZiI6MTcwOTg4MzYxMSwicGF0aCI6Ii8yODgwMjU0NS8zMTExNTUwNTgtMTllOTUyYmUtY2JjMy00MmJlLTgyMmEtMjk1YTBlN2Y1MmY2LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAzMDglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMzA4VDA3NDAxMVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWVkNGJiMzBjYzA4OWFmNzM0ZWUyNmNlMWE1OTFmYmM0YTdmMWIwMDZkZDUxMmY1ZGZmNjdkMTY3N2YwNTQzYmEmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.AhxRDmvvZuqchVoFp39fxW4RpJRvR4GVCD8Cmlohae8)

QueryDSL DTO 로 조회하는 경우에도 잘 동작하네요.

<br />

## @Where 주의사항

__`@Where`__ 어노테이션을 사용하며 동작하지 않는 경우도 한번 살펴보겠습니다.

### case1. native query 를 사용하는 경우

__MemberRepository.java__
```java
public interface MemberRepository extends JpaRepository<Member, Long> {

  @Query(value = "select * from members where team_id = :teamId", nativeQuery = true)
  List<Member> findByTeamId(Long teamId);
}
```

<br />

```java
@Test
void test() {
  List<Member> members = memberRepository.findByTeamId(1L);
  System.out.println(members.size());
}
```

![where-image9](https://private-user-images.githubusercontent.com/28802545/311158488-b82ff72f-8aaf-4a07-981d-ead952fc6bad.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MDk4ODQ3NDEsIm5iZiI6MTcwOTg4NDQ0MSwicGF0aCI6Ii8yODgwMjU0NS8zMTExNTg0ODgtYjgyZmY3MmYtOGFhZi00YTA3LTk4MWQtZWFkOTUyZmM2YmFkLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAzMDglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMzA4VDA3NTQwMVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTVlZGM2ZTZmYmU5ODQxMmViMGFlNTNlZjM5ZjRiYjk1NTFiY2ZhMGY0MTRkMTdiYmRmYjQ5ZmIzZjUzZGNmZDUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.niV_-tC1kroMPMLVm7LHRA-oBVYnbPcfkk9UCmNALUU)

nativeQuery 의 경우 Entity 의 속성을 사용하지 못하기 때문에 위에서 선언한  
__@Where__ 어노테이션도 동작하지 않습니다.

### case2. 식별자로 조회시 1차 캐시의 데이터를 조회하는 경우

1차 캐시는 JPA 특징 중 하나로 데이터 조회 시 1차 캐시에 데이터가 존재한다면 DB 를 거치지 않고 바로 메모리에서 엔티티를 가져옵니다.

```java
@Test
void test() {
  Optional<Member> optionalMember = memberRepository.findById(1L);
  if (optionalMember.isPresent()) {
    Member member = optionalMember.get();
    member.delete();
    memberRepository.save(member);
  }

  Optional<Member> persistMember = memberRepository.findById(1L);
  assertThat(persistMember.isEmpty()).isTrue();
}
```

테스트 코드의 시나리오는 다음과 같습니다.

1. ID가 1인 Member 를 soft delete
2. 식별자인 ID 1 로 다시 조회

이 경우에는 __`findById`__ 에서 Member 를 조회하더라도 Member 는 이미 삭제되어서 __`"status=DISABLE"`__ 이라 조회가 되지 않을것으로 생각할 수 있습니다.  

하지만 실제 동작은 그렇지 않습니다.

![where-image10](
https://private-user-images.githubusercontent.com/28802545/311168813-c2edc22d-7be4-4059-80a2-2e0cea082726.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MDk4ODY5NjIsIm5iZiI6MTcwOTg4NjY2MiwicGF0aCI6Ii8yODgwMjU0NS8zMTExNjg4MTMtYzJlZGMyMmQtN2JlNC00MDU5LTgwYTItMmUwY2VhMDgyNzI2LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAzMDglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMzA4VDA4MzEwMlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTg5MmVmZGM3ZTUwNzJiNjcyMTg3ZTM5ZjgyMGFlNDc5YmNjZmViMjQ1YmQzOWRkZTIxZjRiYWUxMGY2NDI3ZmUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.tIVN2L7Tsm99WcUarENztVTbjJMzbMsdF-bsrM6_bNg)

다음과 같이 데이터가 존재하기에 테스트 코드도 실패했고 데이터도 1차 캐시에서 가져왔기떄문에 SELECT QUERY 도 발생하지 않았습니다.

soft delete 이후 저장했을 시점에 변경된 UPDATE QUERY 는 아직 쓰기지연 저장소에 존재하기에 식별자로 조회하게 되면 @Where 조건 없이 삭제된 데이터를 그대로 가져오게 됩니다.

![where-image10](
https://private-user-images.githubusercontent.com/28802545/311170342-c9291579-48f0-4a88-a8f0-e7057584fac9.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MDk4ODcyNDcsIm5iZiI6MTcwOTg4Njk0NywicGF0aCI6Ii8yODgwMjU0NS8zMTExNzAzNDItYzkyOTE1NzktNDhmMC00YTg4LWE4ZjAtZTcwNTc1ODRmYWM5LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAzMDglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMzA4VDA4MzU0N1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTIxOTU5N2NiY2FkNjVkYWE2NGQ5NmM5NmY4N2IxMDZmMTJiMzU4NTM5OWJlYTVlZWI1NWFhNzRkN2FjYzdkYzkmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.y_xDotxgvu4Fv3ZkMzMj5OBUiwhKa2RS-sV4AStMi1c)

<br />

## @Where Deprecated

![where-image11](
https://private-user-images.githubusercontent.com/28802545/311171739-9131c41c-bf03-4666-99d6-49b8b9d8ea25.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MDk4ODc0OTUsIm5iZiI6MTcwOTg4NzE5NSwicGF0aCI6Ii8yODgwMjU0NS8zMTExNzE3MzktOTEzMWM0MWMtYmYwMy00NjY2LTk5ZDYtNDliOGI5ZDhlYTI1LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDAzMDglMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwMzA4VDA4Mzk1NVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTMwYTY5MmY3MmM1YjlmZWIxZjRjZTc2NzJlMTNlOWVhNGFmNDg5ZTY5MzdkNjZhNmMxZjE1ZDQwNjMxZjFiOWEmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.nGXjNqWXTqXaMmMWPsutkpVdoGZFxKlZcjC5E-cnggk)

__`@Where`__ 어노테이션은 hibernate 6.3 버전부터 __Deprecated__ 되었습니다.  

[해당 문서](https://docs.jboss.org/hibernate/stable/orm/javadocs/org/hibernate/annotations/Where.html)에서는 이후 버전부터는 __`SQLRestriction`__ 를 사용하라고 안내하고 있으니 해당 기능을 사용하게 된다면 참고하시면 좋을것 같습니다.

감사합니다.


#### reference

- https://cheese10yun.github.io/jpa-where/
- https://docs.jboss.org/hibernate/stable/orm/javadocs/org/hibernate/annotations/Where.html