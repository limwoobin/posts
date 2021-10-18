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
>   - [비영속 (new/tansient)](#비영속-new/transient)
>   - [영속(managed)](#영속-managed)
>   - [준영속(detached)](#준영속-detached)
>   - [삭제(removed)](#삭제-removed)
>
> - [**영속성 컨텍스트 특징**](#영속성-컨텍스트-특징)
>
>   - [1차 캐시](#1차-캐시)
>   - [변경 감지 (Dirty Checking)](#변경감지-Dirty-Checking)
>   - [동일성 보장](#동일성-보장)
>   - [Lazy / Eager Loading](#Lazy-/-Eager-Loading)
>
> - [**주의 사항**](#주의-사항)
>
> <br>

<br /><br /><br />

# **영속성 컨텍스트란 ?**

- 엔티티를 관리하고 저장하는 환경
- Jpa 어플리케이션과 Database 사이에서 객체를 관리하는 논리적 개념

<br /><br />

# **엔티티의 생명주기**

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

#### **비영속 (new/transient)**

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

<hr>

#### **영속 (managed)**

- EntityManager로 부터 관리받는 상태, 즉 영속성 컨텍스트에 저장된 상태
- 영속성 컨텍스트의 특징을 사용할 수 있음
  - 1차 캐시
  - Eager or Lazy Loading
  - 변경 감지
  - 동일성 보장

<hr>

#### **준영속 (detached)**

- 영속성 컨텍스트에 존재하다 분리된 상태
- 영속성 컨텍스트로 관리하지 않겠다는 의미

```java
public Team detachTest(Long id) {
		Team team = teamRepository.findById(id).get();
		entityManager.detach(team);	// 준영속

		return team;
}
```

@Transactional 이 Service Layer 에 선언되어있고, osiv 가 false 인 상태에서 해당 Entity가 Presentation Layer 에 있다면 준영속 상태가 될 수 있습니다.  
하지만 실무에선 일반적으로 Service Layer , Presentation Layer 각각 다른 Model 을 사용하게 되고 @Transactional 은 Service Layer 에 선언하는 경우가 일반적이어서 준영속 상태를 만날 일이 흔치는 않습니다.
<br /><br />

> <br>
>
> **비영속과의 차이?**
>
> <hr>
> 두 상태 모두 영속성 컨텍스트의 관리를 받지 않는다는 점은 동일합니다.  
> 다만 비영속은 한번도 저장되지 않았던 객체이고, 준영속은 저장이 되어 영속성 컨텍스트의 관리를 받다가 분리된 객체입니다.  
> 따라서 의도적으로 준영속 entity 의 ID 를 삭제하지 않은 이상 둘의 차이는 ID 값의 유무로 파악할 수 있습니다.
>
> <br>

<hr>

#### **삭제 (removed)**

- 엔티티가 삭제된 상태

```java
public void deleteTest(Long id) {
		Team team = teamRepository.findById(id).get();
		teamRepository.delete(team);
}
```

<hr>

<br>
<br>

# **영속성 컨텍스트 특징**

- #### **1차 캐시**
- #### **변경감지 (Dirty Checking)**
- #### **동일성 보장**
- #### **Lazy / Eager Loading**

# **주의 사항**
