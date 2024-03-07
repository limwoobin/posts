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

__Team.java__
```java
@Getter
@Entity(name = "teams")
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@SQLRestriction(value = "status = 'ACTIVE'")
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
@SQLRestriction(value = "status = 'ACTIVE'")
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
}
```

<br />

__Status.java__
```java
@Getter
@AllArgsConstructor
public enum Status {
  ACTIVE,
  DISABLE;
}
```

그럼 이제 테스트를 통해서 케이스별로 실제 쿼리에 조건이 어떻게 들어가는지 확인해보겠습니다.

## @Where 조건 테스트



## @Where 주의사항

식별자 조회시 1차 캐시 조회

## @Where Deprecated


#### reference
https://cheese10yun.github.io/jpa-where/