> <br>
>
> ## **Overview**
>
> <br>
>
> - [**영속성 컨텍스트란 ?**](#영속성-컨텍스트란)
>
> - [**엔티티의 생명주기**](#엔티티의-생명주기)
>
>   - [비영속 (new/tansient)](#비영속-newtransient)
>   - [영속(managed)](#영속-managed)
>   - [준영속(detached)](#준영속-detached)
>   - [삭제(removed)](#삭제-removed)
>
> <br>

<br /><br /><br />

# **영속성 컨텍스트란 ?**

- 엔티티를 관리하고 저장하는 환경
- Jpa 어플리케이션과 Database 사이에서 객체를 관리하는 논리적 개념

<br /><br />

# **엔티티의 생명주기**

![persistence-context](https://user-images.githubusercontent.com/28802545/138093343-03a15707-38d4-416f-af84-39a8e5cbcf79.png)

<br /><br />
Team.java

```java
@NoArgsConstructor
@Entity
@Getter
public class Team {
    public Team(String name) {
        this.name = name;
    }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
}
```

<br />

### **비영속 (new/transient)**
<hr>

- 영속화 되기 전의 상태
- 순수한 Entity 객체
- EntityManager 로부터 관리받지 않는 상태

<br />

TeamService.java

```java
@RequiredArgsConstructor
@Service
public class TeamService {
    private final TeamRepository teamRepository;

    @Transactional
    public TeamDto save(TeamDto teamDto) {
		// 객체가 저장되기 전의 상태
        Team team = new Team(teamDto.getName());    // 비영속

        // 객체가 저장된 후의 상태
        team = teamRepository.save(team);   // 영속

        TeamDto teamResponse = TeamDto.builder()
                .id(team.getId())
                .name(team.getName())
                .build();

        return teamResponse;
    }
}
```

위 예제와 같이 객체가 영속성 컨텍스트에 저장되기 전의 상태가 비영속에 해당됩니다.

<br /><br />

### **영속 (managed)**
<hr>

- EntityManager로 부터 관리받는 상태, 즉 영속성 컨텍스트에 저장된 상태
- 영속성 컨텍스트의 특징을 사용할 수 있음
  - [1차 캐시](https://devoong.com/posts/2#1차-캐시)
  - [Eager or Lazy Loading](https://devoong.com/posts/2)
  - [변경 감지](https://devoong.com/posts/2)
  - [동일성 보장](https://devoong.com/posts/2)
  - [쓰기 지연](https://devoong.com/posts/2)
- EntityManager로 엔티티를 조회(find , jpql , queryDSL)하거나 저장하면 그 엔티티는 영속상태가 됨

```java
Team team = new Team(teamDto.getName());    // 비영속
team = teamRepository.save(team);   // 영속

// 혹은

Team team = teamRepository.findById(id)
    .orElseThrow(RuntimeException::new);  // 영속

Team team = teamRepository.findByName(name)
    .orElseThrow(RuntimeException::new);  // 영속
```


<br /><br />

### **준영속 (detached)**
<hr>

- 영속성 컨텍스트에 존재하다 분리된 상태
- 엔티티를 영속성 컨텍스트로 관리하지 않겠다는 의미

```java
public Team detachTest(Long id) {
    Team team = teamRepository.findById(id).get();

    entityManager.detach(team); // (1) 특정 엔티티만 분리
    entityManager.clear();  // (2) 영속성 컨텍스트 초기화(1차 캐시 or 쓰기지연 저장소에 있는 데이터들이 날라감)
    entityManager.close();  // (3) 영속성 컨텍스트 종료

    return team;
}
```

준영속 상태를 만드는 방법은 위의 3가지를 볼 수 있습니다.

혹은 @Transactional 이 ServiceLayer 에 선언되어있고, OSIV가 false 인 상태에서 해당 Entity가 Presentation Layer 에 있다면 준영속 상태가 될 수 있습니다.  
다만 Entity가 Presentation Layer 에 있는것은 지양하시는게 좋습니다.
<br /><br />

> <br>
>
> ### **비영속과의 차이?**
>
> <hr>
> 두 상태 모두 영속성 컨텍스트의 관리를 받지 않는다는 점은 동일합니다. <br>
> 다만 비영속은 한번도 저장되지 않았던 객체이기 때문에 id(식별자)가 없으며, <br> 준영속은 저장이 되어 영속성 컨텍스트의 관리를 받다가 분리된 객체이므로 id(식별자)가 존재합니다.<br>
> 따라서 의도적으로 준영속 entity 의 ID 를 삭제하지 않은 이상 둘의 차이는 Id 값의 유무로 파악해 볼 수 있습니다.
>
> <br>
>
> <br>

<br /><br />

### **삭제 (removed)**
<hr>

- 엔티티가 삭제된 상태

```java
public void deleteTest(Long id) {
		Team team = teamRepository.findById(id).get();
		teamRepository.delete(team);

        // 혹은

        Team team = teamRepository.findById(id).get();
        entityManager.remove(team);
}
```
