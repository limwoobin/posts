> <br>
>
> ## **Overview**
>
> - ### [**Stream api 그룹핑 방법**](#stream-api-그룹핑-방법)
>   - [단일 값으로 그룹핑](#단일-값으로-그룹핑)
>   - [2개 이상의 값 그룹핑](#2개-이상의-값-그룹핑)
> - ### [**여러개의 필드값으로 그룹핑**](#여러개의-필드값으로-그룹핑)
>
>   - [Custom Class 를 이용한 그룹핑](#custom-class-를-이용한-그룹핑)
>
> - #### [**정리**](#정리) <br><br>

<br />

#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/java-grouping-example)

<br />

# **Stream api 그룹핑 방법**

Java 에서는 stream api 를 이용하여 손쉽게 grouping 을 진행할 수 있습니다.

예시를 통해 그룹핑 코드를 작성해보겠습니다.  
아래는 그룹핑에 사용될 예시 데이터입니다.

<br>

User.java

```java
public class User {
    private Long id;
    private String name;
    private Gender gender;	// MALE, FEMALE
    private Integer age;
    private City city;	// SEOUL, TOKYO, NEW_YORK, LA, PARIS
}
```

| id  | 이름  | 성별   | 나이 | 도시     |
| --- | ----- | ------ | ---- | -------- |
| 1   | user1 | MALE   | 22   | SEOUL    |
| 2   | user2 | FEMALE | 25   | SEOUL    |
| 3   | user3 | FEMALE | 18   | NEW_YORK |
| 4   | user4 | MALE   | 33   | PARIS    |
| 5   | user5 | FEMALE | 30   | TOKYO    |
| 6   | user6 | MALE   | 25   | PARIS    |
| 7   | user7 | MALE   | 28   | LA       |
| 8   | user8 | MALE   | 29   | LA       |
| 9   | user9 | FEMALE | 31   | TOKYO    |

<br>

위의 테스트 데이터로 그룹핑 예제를 작성해보겠습니다.

<br>

### **단일 값으로 그룹핑**

위의 값을 성별으로 그룹핑 해보겠습니다.  
그렇다면 MALE이 5명, FEMALE이 4명이 되어야 합니다. 테스트코드로 확인해보겠습니다.

```java

@DisplayName("유저목록을 성별로 그룹핑하면 MALE 5명, FEMALE 4명이 되어야한다")
@Test
void grouping_test_1() {
    // given
    List<User> 유저_목록 = UserData.users;

    // when
    Map<Gender, List<User>> result = 유저_목록.stream()
            .collect(groupingBy(User::getGender));

    // then
    List<User> 남자_회원_목록 = result.get(Gender.MALE);
    List<User> 여자_회원_목록 = result.get(Gender.FEMALE);

    assertEquals(남자_회원_목록.size(), 5);
    assertEquals(여자_회원_목록.size(), 4);
}

```

![grouping-image-1](https://user-images.githubusercontent.com/28802545/160381954-72b34aaa-2fe0-4db7-80fb-91760d54a810.PNG)

성별로 그룹핑한 결과 정상적으로 나눠진것을 확인할 수 있습니다.

### **2개 이상의 값 그룹핑**

그렇다면 이번엔 성별 + 도시별로 나누어 보겠습니다. 성별로 나눠진 후에 도시별로 추가로 나뉘어 지겠네요.  
MALE 의 경우 도시가 SEOUL, PARIS, LA 이 세곳에 살고있습니다. FEMALE 의 경우는 SEOUL, NEW_YORK, TOKYO 에 살고있네요.  
그렇다면 성별 + 도시 기준으로 그룹핑하면
MALE/SEOUL(1명), MALE/PARIS(2명), MALE/LA(2명), FEMALE/SEOUL(1명), FEMALE/NEW_YORK(1명), FEMALE/TOKYO(2명) 이렇게 그룹핑 될것입니다. 테스트 코드를 통해 확인해보겠습니다.

```java
@DisplayName("유저목록을 성별, 도시별로 그룹핑하라")
@Test
void grouping_test_2() {
    // given
    List<User> 유저_목록 = UserData.users;

    // when
    Map<Gender, Map<City, List<User>>> result = 유저_목록.stream()
            .collect(groupingBy(User::getGender, groupingBy(User::getCity)));

    // then
    Map<City, List<User>> 남자_그룹 = result.get(Gender.MALE);
    Map<City, List<User>> 여자_그룹 = result.get(Gender.FEMALE);

    List<User> 남자_SEOUL_그룹 = 남자_그룹.get(City.SEOUL);
    List<User> 남자_PARIS_그룹 = 남자_그룹.get(City.PARIS);
    List<User> 남자_LA_그룹 = 남자_그룹.get(City.LA);

    List<User> 여자_SEOUL_그룹 = 여자_그룹.get(City.SEOUL);
    List<User> 여자_NEW_YORK_그룹 = 여자_그룹.get(City.NEW_YORK);
    List<User> 여자_TOKYO_그룹 = 여자_그룹.get(City.TOKYO);

    assertEquals(남자_SEOUL_그룹.size(), 1);
    assertEquals(남자_PARIS_그룹.size(), 2);
    assertEquals(남자_LA_그룹.size(), 2);

    assertEquals(여자_SEOUL_그룹.size(), 1);
    assertEquals(여자_NEW_YORK_그룹.size(), 1);
    assertEquals(여자_TOKYO_그룹.size(), 2);
}
```

![grouping-image-2](https://user-images.githubusercontent.com/28802545/160383882-ee09de59-76d9-431d-9e35-238961913dd3.PNG)

예상했던것과 같이 정상적으로 그룹핑된것을 확인할 수 있습니다. 하지만 문제가 있습니다.  
지금까지는 1개 혹은 2개의 값으로 그룹핑을 진행해서 큰 무리없이 수행해왔습니다. 물론 2개의 값으로 그룹핑한 객체도 그다지 깔끔해 보이진 않습니다.  
하지만 3개 혹은 그 이상의 값으로 그룹핑을 해야한다면 어떨까요?? 그룹핑하는 개수만큼 __*<U>Map<T , Map<T2, Map<T3, Map<>>>> ...</U>*__ 이렇게 감싸서 원하는 객체를 만들어 사용할 수 있을까요? 아니면 이런 코드를 남들이 알아볼 수 있을까요?  
이 이상의 값을 기준으로 그룹핑하는것은 가독성으로나 유지보수성으로나 좋지 않다고 생각합니다. 그래서 이를 해결할 수 있는 방법을 소개하려합니다.

<br>

# 여러개의 필드값으로 그룹핑

### Custom Class 를 이용한 그룹핑

Custom Class 를 하나 선언하여 그룹핑할 값들을 선언합니다. 위의 예제와 동일하게 성별 + 도시의 기준으로 그룹핑을 해보겠습니다.

Tuple.java

```java

@Getter
@EqualsAndHashCode
public class Tuple {
    private Gender gender;
    private City city;
}

```

객체의 필드값을 이용하여 key를 생성할것이기 때문에 동등비교를 위해 @EqualsAndHashCode 어노테이션을 선언합니다.  
equals, hashcode 를 재정의하지 않으면 의도한대로 그룹핑되지 않습니다.

테스트 코드를 작성해서 검증해보겠습니다.

```java
@DisplayName("Tuple Class를 이용하여 그룹핑하라")
@Test
void grouping_test_3() {
    // given
    List<User> 유저_목록 = UserData.users;

    // when
    Map<Tuple, List<User>> result = 유저_목록.stream()
            .collect(groupingBy(user -> new Tuple(user.getGender(), user.getCity())));

    // then
    for (Map.Entry<Tuple, List<User>> elem : result.entrySet()) {
        System.out.print("[" + elem.getKey().getGender() + "," + elem.getKey().getCity() + "]: ");
            System.out.print(elem.getValue().toString());
            System.out.println();
    }
}
```

![grouping-image-3](https://user-images.githubusercontent.com/28802545/160390509-e5d51ce3-a76d-4c7d-ae6b-1f131a4f444f.PNG)

![grouping-image-4](https://user-images.githubusercontent.com/28802545/160391070-3d9ff7fe-89cc-471a-85a0-cc9cbf4eafec.PNG)

위 결과와 같이 6개의 그룹으로 그룹핑된것을 확인할 수 있습니다. 실제 값도 동일한지 검증해보겠습니다.

```java

@DisplayName("Custom Class 를 이용하여 그룹핑 후 검증하라")
@Test
void grouping_test_4() {
    // given
    List<User> 유저_목록 = UserData.users;

    // when
    Map<Tuple, List<User>> result = 유저_목록.stream()
            .collect(groupingBy(user -> new Tuple(user.getGender(), user.getCity())));

    // then
    Tuple 남자_SEOUL_tuple = new Tuple(Gender.MALE, City.SEOUL);
    Tuple 남자_PARIS_tuple = new Tuple(Gender.MALE, City.PARIS);
    Tuple 남자_LA_tuple = new Tuple(Gender.MALE, City.LA);

    Tuple 여자_SEOUL_tuple = new Tuple(Gender.FEMALE, City.SEOUL);
    Tuple 여자_NEW_YORK_tuple = new Tuple(Gender.FEMALE, City.NEW_YORK);
    Tuple 여자_TOKYO_tuple = new Tuple(Gender.FEMALE, City.TOKYO);

    assertEquals(result.get(남자_SEOUL_tuple).size(), 1);
    assertEquals(result.get(남자_PARIS_tuple).size(), 2);
    assertEquals(result.get(남자_LA_tuple).size(), 2);
    assertEquals(result.get(여자_SEOUL_tuple).size(), 1);
    assertEquals(result.get(여자_NEW_YORK_tuple).size(), 1);
    assertEquals(result.get(여자_TOKYO_tuple).size(), 2);
}

```

![grouping-image-5](https://user-images.githubusercontent.com/28802545/160391868-5dc15b16-7a77-4367-9786-922687b7095c.PNG)

정상적으로 통과된것을 확인할 수 있습니다.

<br>

# **정리**

이렇게 Custom Class 를 이용하여 객체를 그룹핑하는 방법을 알아보았습니다. 이와같은 방식을 이용하면 3개, 4개 그 이상의 값을 기준으로 그룹핑을 하더라도 무리없이 사용할 수 있습니다.

감사합니다.
