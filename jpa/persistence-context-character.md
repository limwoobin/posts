> <br>
>
> ## **Overview**
>
> - [**영속성 컨텍스트 특징**](#영속성-컨텍스트-특징)
>
>   - [1차 캐시](#1차-캐시)
>   - [변경 감지 (Dirty Checking)](#변경-감지-dirty-checking)
>   - [동일성 보장](#동일성-보장)
>   - [지연 로딩(Lazy Loading](#지연-로딩lazy-loading)
>   - [쓰기 지연](#쓰기-지연)
>
> - [**마무리**](#마무리)
>
> <br>

<br />

#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/jpa-example)

<br />

# **영속성 컨텍스트 특징**

### **1차 캐시**

<hr>

- 영속성 컨텍스트 내부에서 엔티티를 캐시로 저장하는 것
- 일반적으로 @Transactional 어노테이션과 라이프사이클이 동일함
- OSIV(Open Session In View) 가 true 라면 ServiceLayer 에서  
  @Transactional 이 종료되어도 PresentationLayer 까지도 1차 캐시는 유지됨
- Jpa 는 데이터 조회시 캐시를 우선적으로 조회하고 캐시에 데이터가 없으면 DB를 조회함

```java
@Transactional
public Team findTeam(Long id) {
    return teamRepository.findById(id)
        .orElseThrow(RuntimeException::new);
}
```

아래는 위 코드 호출시 발생한 log 입니다.  
예상대로 select query 가 찍힌것을 확인할 수 있습니다.

![posts-image-1](https://user-images.githubusercontent.com/28802545/137903178-75a9213b-b038-466d-b0e7-1c847da7e3fb.PNG)

<br /><br />

```java
@Transactional
public Team cashTest(TeamDto teamDto) {
	Team team = new Team(teamDto.getName());
	team = teamRepository.save(team);   // (1)

	Team team1 = teamRepository.findById(team.getId())
			.orElseThrow(RuntimeException::new);    // (2)

	return team;
}
```

아래 로그 이미지를 보시면 query는 insert 쿼리 한번 외에는 아무 query 도 찍히지 않았습니다.  
(1) 의 save 메소드가 호출되며 Team Entity 에 바로 1차캐시가 저장되었기 때문입니다.  
(2) 의 findById 를 호출해도 select 쿼리가 찍히지 않은것은 Team Entity 에 저장된 1차 캐시를 읽어왔기 때문입니다.

![postsimage-2](https://user-images.githubusercontent.com/28802545/137903177-a6cc1557-454e-4a80-a912-6098b8f11b5a.PNG)

<br /><br />

```java
@Transactional
public Team cashTest(Long id) {
    Team team = teamRepository.findById(id)
            .orElseThrow(RuntimeException::new);    // (1)

    Team team2 = teamRepository.findById(id)
            .orElseThrow(RuntimeException::new);    // (2)

    return team;
}
```

![posts-image-1](https://user-images.githubusercontent.com/28802545/137903178-75a9213b-b038-466d-b0e7-1c847da7e3fb.PNG)

위의 코드도 동일하게 select query 가 한번만 호출되었습니다.  
(1) 에서 findById 를 통해 Entity 를 가져와서 1차 캐시에 저장해놓았기 때문입니다.  
그래서 (2) 에서 동일한 동작을 하더라도 DB가 아닌 1차 캐시에서 데이터를 가져오게 됩니다.

<br />

### **주의**

<br />

```java
@Transactional
public Team cashTest3(Long id) {
	Team team1 = teamRepository.findById(id)
			.orElseThrow(RuntimeException::new);    // (1)

	Team team2 = teamRepository.findByName("test"); // (2)

	Team team3 = teamRepository.findByName("test"); // (2)

	return team1;
}
```

**위 코드를 호출하면 select query 가 몇번 호출될거라 예상하시나요 ?**  
정답은 3번입니다.  
(1) 에서 Team 이 호출되었으니 1차캐시에 저장되어 (2) , (3) 에선 1차 캐시를 조회하기 때문에  
한번만 호출될거라고 생각하셨을수 있습니다. <br>
지만 1차 캐시를 사용하는데에는 **식별자(id)** 로 조회하는 경우에만 해당된다는 특징이 있습니다.  
왜냐하면 영속성 컨텍스트는 내부적으로 식별자로 Entity를 관리하기 때문에 영속상태인 객체는 식별자가 반드시 있어야합니다. <br>
위의 (2) , (3) 처럼 query methods 방식을 사용하게되면 영속성 컨텍스트가 관리하는 식별자로 값을 찾는게 아닌 JPQL로 쿼리가 나가기 때문에 1차캐시에서 데이터를 불러오지 못합니다.

<br /><br />

```java
@Transactional
public Team cashTest3(Long id) {
	Team team1 = teamRepository.findById(id)
			.orElseThrow(RuntimeException::new);    // (1)

	Team team2 = teamRepository.findById(id)
			.orElseThrow(RuntimeException::new);    // (2)

	Team team3 = teamRepository.findById(id)
			.orElseThrow(RuntimeException::new);    // (3)

	return team1;
}
```

![posts-image-1](https://user-images.githubusercontent.com/28802545/137903178-75a9213b-b038-466d-b0e7-1c847da7e3fb.PNG)

위의 코드는 (1) , (2) , (3) 모두 식별자를 이용한 쿼리이기 때문에  
실제 select query 는 한번만 호출된걸 확인할 수 있습니다.

<br /><br />

### **변경 감지 (Dirty Checking)**

<hr>

- 트랜잭션 안에서의 엔티티의 변경을 감지하여 변경이 일어나면  
  변경 내용을 자동으로 DB에 반영

```java
@Transactional
public void getTeam(Long id) {
	Team team = teamRepository.findById(id)
			.orElseThrow(RuntimeException::new);

	team.setName("dirty-checking-test");
}
```

1. Transaction 이 시작합니다.
2. Team Entity 를 조회합니다.
3. Team Entity 의 Name 을 "dirty-checking-test" 로 수정합니다.
4. Transaction 이 종료됩니다.

로그를 확인해보겠습니다.

![dirty-checking-1](https://user-images.githubusercontent.com/28802545/138548230-84c70617-6286-497b-a663-70c5ae18eff4.PNG)

분명 update query 를 요청한 로직이 없는데 트랜잭션이 끝남과 동시에 update query 가 찍혀있는 것을 확인할 수 있습니다.

#### **동작 방식**

1. Jpa 는 영속상태의 엔티티가 최초 조회되었을때 최초 조회된 상태를 기준으로 따로 snapshot을 만들어 놓습니다.
2. 그리고 트랜잭션이 종료되기전 flush()가 일어나는 시점에 현재 Entity의 상태와 snapshot에 저장된 Entity 의 상태를 비교하여 update query 를 생성합니다.
3. 생성한 update query 를 쓰기지연 SQL 저장소에 보냅니다.
4. 트랜잭션이 commit() 되는 시점에 DB로 update query 를 전송합니다.

<br />

> 트랜잭션이 종료되기 전 이라 표현한 이유는 트랜잭션이 끝나는 시점에 flush 가 발생하기 떄문입니다. <br>
> 만약 트랜잭션 내에서 flush 를 실행하게 되는 JPQL 이 실행되거나 <br>
> 강제로 entityManager.flush() 를 호출하게 되어도 동일하게 update query 가 발생합니다. <br>
> 추가로 flush 가 발생했다고 바로 DB에 반영되는것은 아닙니다.
> flush 는 트랜잭션을 커밋하지 않기 때문에 이후에 커밋이 되어야 DB에 반영됩니다.

<br /><br />

### **동일성 보장**

<hr>

- 같은 Transaction 혹은 같은 EntityManager 에서 가져온 객체는  
  항상 동일한 객체를 리턴한다.

<br />

**아래와 같이 테스트코드를 작성해보았습니다.**

<br />

```java
@Test
@Transactional
public void 동일성_테스트() {
	Team team1 = teamRepository.findById(1L).get();
	Team team2 = teamRepository.findById(1L).get();

	boolean val = team1 == team2;
	System.out.println("### 동일성 테스트 결과 : " + val);
	assertThat(val).isEqualTo(true);
}
```

<br />

![team-domain-test-1](https://user-images.githubusercontent.com/28802545/138550238-f7a865c6-12a5-4add-bbd3-a3e4caade31f.PNG)

동일성을 보장하다는 테스트코드 결과를 확인할 수 있습니다.  
그렇다면 같은 트랜잭션이 아닌 케이스도 한번 확인해보겠습니다.

<br /><br />

```java
@Test
public void 동일성_테스트_v2() {
	Team team1 = teamRepository.findById(1L).get();
	Team team2 = teamRepository.findById(1L).get();

	boolean val = team1 == team2;
	System.out.println("### 동일성 테스트 결과 : " + val);
	assertThat(val).isEqualTo(true);
}
```

![team-domain-test-2](https://user-images.githubusercontent.com/28802545/138550239-8613b347-02eb-40df-96df-1b10fc8d9379.PNG)

<!-- <br /> -->

메소드에 @Transactional 어노테이션을 제거하여 테스트 해보았습니다.  
예상한대로 동일성 보장이 되지 않았고 테스트코드도 실패한걸 확인할 수 있습니다.

<br /><br />

### **지연 로딩(Lazy Loading)**

<hr>

- 연관관계에 있는 엔티티를 조회시 한번에 가져오지 않고
  필요시에 가져오는 것.
- 연관관계에 있는 객체는 프록시 상태로 초기화되지 않은 상태로 존재함.
- 필요시에 가져오기 때문에 불필요한 쿼리를 실행하지 않을 수 있음

<br />

> <br>
>
> ### **각 연관관계의 기본 로딩 방식**
>
> - **OneToMany : Lazy**
> - **ManyToMany : Lazy**
> - **ManyToOne : Eager**
> - **OneToOne : Eager**
>
> <br>

<br />

Team.java

```java
@NoArgsConstructor
@AllArgsConstructor
@Getter
@Setter
@Entity
public class Team {

    public Team(String name) {
        this.name = name;
    }

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(fetch = FetchType.LAZY , cascade = CascadeType.ALL)
    @JoinColumn(name = "team_id")
    private List<Member> members = new ArrayList<>();

    public void addMembers(Member member) {
        this.members.add(member);
    }
}
```

<br />

Member.java

```java
@NoArgsConstructor
@AllArgsConstructor
@Getter
@Setter
@Entity
public class Member {

	public Member(String name) {
		this.name = name;
	}

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;

	@Column(name = "team_id")
	private Long teamId;

	private String name;
}
```

예제에서 사용할 TeamEntity 입니다.  
Team(1) : Member(N) - 1:N 단방향으로 설계를 해봤습니다.

지연 로딩으로 객체를 가져오게 되면 객체는 사용되기 전까지는 프록시객체로  
초기화되지 않은 상태로 있게됩니다.  
이 점을 이용하여 테스트코드를 작성해보겠습니다.

```java
@SpringBootTest
class 지연_로딩_테스트_클래스 {

		@Autowired
		private TeamRepository teamRepository;

		@Autowired
		private MemberRepository memberRepository;

		@Autowired
		private EntityManager entityManager;

		@BeforeEach
		void init() {
			Team team = new Team("test");
			Member member = new Member("member1");
			Member member2 = new Member( "member2");
			Member member3 = new Member("member3");

			team.addMembers(member);
			team.addMembers(member2);
			team.addMembers(member3);
			teamRepository.save(team);

			entityManager.flush();
			entityManager.clear();
			entityManager.close();
		}

		@Test
		@Transactional
		public void 연관관계_객체가_프록시객체라면_성공() {
				Team team = teamRepository.findById(1L)
								.orElseThrow(RuntimeException::new); // (1)

				boolean isLoad = entityManager.getEntityManagerFactory().getPersistenceUnitUtil().isLoaded(team.getMembers()); // (2)
				assertThat(isLoad).isEqualTo(false); // (3)
		}

		@Test
		@Transactional
		public void 연관관계_객체가_초기화_되었다면_성공() {
				Team team = teamRepository.findById(1L)
								.orElseThrow(RuntimeException::new); // (1)

				System.out.println(team.getMembers().size()); // (2)

				boolean isLoad = entityManager.getEntityManagerFactory().getPersistenceUnitUtil().isLoaded(team.getMembers()); // (3)
				assertThat(isLoad).isEqualTo(true); // (4)
		}
}
```

**entityManager.getEntityManagerFactory().getPersistenceUnitUtil().isLoaded()**

- 해당 객체가 프록시 객체가 초기화되었는지 유무를 나타냄.
- 프록시 객체가 초기화되어 로드되었다면 true , 초기화되지 않았다면 false.

<br />

**첫번째 메소드** 우선 확인해보겠습니다.  
**(1)** - TeamEntity 를 조회합니다. 연관관계는 Lazy 입니다.  
**(2)** - team.getMembers() 는 List<Member> 객체를 리턴합니다.  
하지만 해당 객체는 한번도 사용되지 않았으니 초기화되지 않았으니 false 를 리턴할 것으로 예상됩니다.  
**(3)** - isLoad 는 예상대로 false 를 반환합니다.

![lazy-loading-1](https://user-images.githubusercontent.com/28802545/138583651-bd87aee0-d69d-4a73-824a-0616c3a68751.PNG)

<br />

**두번째 메소드** 를 한번 확인해보겠습니다.

**(1)** - TeamEntity 를 조회합니다. 연관관계는 Lazy 입니다.  
**(2)** - team.getMembers().size() 를 호출했습니다.  
즉, 프록시객체를 사용한 것입니다. 이렇게 되면 해당 객체는 초기화됩니다.  
**(3)** - team.getMembers() 객체는 위에서 사용되었으니 true를 리턴할 것으로 예상됩니다.  
**(4)** - isLoad 는 예상대로 true를 반환합니다.

![lazy-loading-2](https://user-images.githubusercontent.com/28802545/138583652-8c05fded-4934-44e8-8321-4f6fa0c2ffcd.PNG)

- 추가적으로 객체가 초기화되며 select query 발생한것을 이미지에서 학인할 수 있습니다.

<br /><br /><br />

### **쓰기 지연**

<hr>

- 한 트랜잭션 내에서 발생하는 insert query 혹은 영속상태인 객체에서 일어나는  
  update , delete query 는 쓰기지연 SQL 저장소에 저장되었다가 트랜잭션이 종료될때 한번에 날아감.

<br />
테스트코드로 udpate 먼저 확인해보겠습니다.

```java
@Nested
class 쓰기_지연_테스트_클래스 {

	@Test
	@Transactional
	@Rollback(value = false)
	public void 쓰기지연_update_test() {
			Long id = init().getId();

			Team team = teamRepository.findById(id).get();

			System.out.println("### UPDATE BEGIN");
			team.setName("CHANGE");
			System.out.println("### UPDATE END");
	}
}
```

![test-code-transactional](https://user-images.githubusercontent.com/28802545/138689374-d7cecfc8-e334-42f5-a470-a522c0079293.png)

> 정상적으로 query 가 찍히는것을 확인하기 위해 Rollback 옵션을 false 로 주었습니다.  
> Test 에서의 @Transactional 은 자동적으로 Rollback 이 되기 때문에 jpa 에서 query 가 찍히지 않는 경우가 있어 확인을 위해 넣었습니다.

<br />

변경감지 기능을 이용하여 update 해보겠습니다.  
과연 update query 가 어느 위치에 찍힐까요?

![update-test-query](https://user-images.githubusercontent.com/28802545/138690625-9b52f803-3f61-4ba9-afac-dcfd0f1234e3.PNG)

update query 가 가장 마지막에 찍힌것을 확인하실 수 있습니다.  
트랜잭션이 종료되며 쓰기지연 저장소에 있던 query 가 실행된것입니다.

<br />

마찬가지로 delete 도 확인해보겠습니다.

```java
@Test
@Transactional
@Rollback(value = false)
public void 쓰기지연_delete_test() {
		Long id = init().getId();

		Team team = teamRepository.findById(id).get();

		System.out.println("### DELETE BEGIN");
		teamRepository.delete(team);
		System.out.println("### DELETE END");
}
```

![delete-test-query](https://user-images.githubusercontent.com/28802545/138690629-7bed18c0-c23e-4ea5-86a2-fc61ac518b5b.PNG)

delete query 역시 update query와 동일합니다.

<br />

이번엔 마지막으로 insert query 에 대해 알아보겠습니다.

```java
@Test
@Transactional
@Rollback(value = false)
public void 쓰기지연_insert_test() {
		System.out.println("### INSERT BEGIN");
		Team team = new Team("test");
		Team team2 = new Team("test2");
		teamRepository.save(team);
		teamRepository.save(team2);
		System.out.println("### INSERT END");
}

Team.java
public class Team {
	public Team(String name) {
			this.name = name;
	}

	@Id
	@GeneratedValue(strategy = GenerationType.SEQUENCE)
	private Long id;
	...
```

테스트를 실행하기전에 GenerationType을 SEQUENCE 로 변경하였습니다. AUTO 도 상관없습니다.  
코드를 실행해보겠습니다.

<br />

![insert-test-1](https://user-images.githubusercontent.com/28802545/138691596-36632a67-0f4e-48ab-b4fd-5ae15981f2fa.PNG)

시퀀스를 저장과 동시에 가져오고 이후에 트랜잭션이 종료되는 시점에  
정상적으로 insert query 가 마지막에 찍힌것을 볼 수 있습니다.

<br />

이번엔 GenerationType을 IDENTITY으로 변경해서 다시 실행해보겠습니다.

```java
public class Team {
	public Team(String name) {
			this.name = name;
	}

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;
	...
```

![insert-test-2](https://user-images.githubusercontent.com/28802545/138691600-efc4e451-2f30-4b27-b931-4ebee9ecc2b8.PNG)

뭔가 이상합니다. GenerationType을 IDENTITY 으로 변경했을 뿐인데 예상과 다르게  
트랜잭션이 종료되기전인 저장하자마자 바로 query가 실행이 되었습니다.

이를 이해하기 위해서는 **GenerationType.IDENTITY** 의 특징에 대해 찾아볼 필요가 있습니다.

### **GenerationType.IDENTITY**

- 기본키 생성을 DB에 위임
- DB에서 자동으로 auto_increment 로 저장됨

> 즉, DB에 insert 가 되어야만 id 값을 알 수가 있습니다.  
> 기본키 매핑전략중 GenerationType.IDENTITY 가 유일합니다.  
> 그렇기 때문에 IDENTITY 전략은 insert 시 쓰기지연의 혜택을 받을 수 없습니다.

<br />
<br />
<br />

# 마무리

영속성 컨텍스트의 특징 5가지에 대해서 알아보았습니다.  
위에서 얘기한 기본키 매핑전략은 여기 내용에 담기엔 번잡해질것같아  
추후 뒤에서 자세하게 다뤄볼 수 있도록 하겠습니다.

<br />

추가적인 질문이나 지적해주실 부분이 있다면 댓글로 언제든지 부탁드립니다. 감사합니다.
