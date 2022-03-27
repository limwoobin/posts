> <br>
>
> ## **Overview**
>
> - ### [**Java에서의 그룹핑 방법**](#subject-1)
>   - [단일 값으로 그룹핑](#content-0)
>   - [2개 이상의 값 그룹핑](#content-1)
> - ### [**여러개의 필드값으로 그룹핑**](#subject-2)
>   - [content 1](#content-1) <br><br>

<br />

#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/java-grouping-example)

<br />

# **Java에서의 그룹핑 방법**

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

### **단일 값으로 그룹핑**

### **2개 이상의 값 그룹핑**

...

<br>
<br>

# Subject 2

### content 1

...
<br>
<br>

### content 2

...
<br>
<br>

# **Subject 3**

### content 3 / 4

...
<br>
