> <br>
>
> ## **Overview**
>
> <br>

> - [**영속성 컨텍스트 특징**](#영속성-컨텍스트-특징)
>
>   - [1차 캐시](#1차-캐시)
>   - [변경 감지 (Dirty Checking)](#변경-감지-dirty-checking)
>   - [동일성 보장](#동일성-보장)
>   - [Lazy / Eager Loading](#lazy--eager-loading)
>   - [쓰기 지연](#쓰기-지연)
>
> - [**주의 사항**](#주의-사항)
>
> <br>

<br /><br /><br />

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
2. 그리고 트랜잭션이 끝나는 시점에 현재 Entity의 상태와 snapshot에 저장된 Entity 의 상태를 비교하여 update query 를 생성합니다.
3. 트랜잭션이 끝남과 동시에 update query 를 실행합니다.

> <br>
> 트랜잭션이 끝나는 시점이라 표현한 이유는 트랜잭션이 끝나는 시점에 flush 가 발생하기 떄문입니다. <br>
> 만약 트랜잭션 내에서 flush 를 실행하게 되는 JPQL 이 실행되거나 <br>
> 강제로 entityManager.flush() 를 호출하게 되어도 동일하게 update query 가 발생합니다. <br>
> 추가로 flush 가 발생했다고 바로 DB에 반영되는것은 아닙니다.
> flush 는 트랜잭션을 커밋하지 않기 때문에 추가로 커밋이 되어야 DB에 반영됩니다.
> <br>
> <br>

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

### **Lazy / Eager Loading**

<hr>

### **쓰기 지연**

<hr>
