> <br>
>
> ## **Overview**
>
> - ### [**@Valid 사용하기**](#valid-사용하기)
>
>   - [**사용 방법**](#사용-방법)
>   - [**예제 코드**](#예제-코드)
>   - [**controller 에서 처리하기**](#content-1)
>
> - ### [**advice 를 이용한 Exception Handling 적용**](#subject-2)
>   - [**advice 를 이용한 handling 방법**](#content-2)
> - [**Subject 3**](#subject-3)
>   - [**content 3 / 4**](#content-3--4) <br><br>

<br />

#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/valid-example)

<br />
<br />

# **@Valid 사용하기**

Spring 에서는 유효성 체크를 위하여 @Valid annotation 을 지원합니다.  
Valid는 [**JSR-303(Bean Validation)**](https://beanvalidation.org/1.0/spec/) 표준 스펙으로서 제약조건이 있는 객체에게 Bean Validation 을 이용해 조건을 검증하는 어노테이션입니다.

### **사용 방법**

환경

- Spring boot 2.6.2
- java11

build.gradle

```java
// gradle
implementation('org.springframework.boot:spring-boot-starter-validation')
```

valid 를 사용하기 위해 위 의존성을 추가합니다.  
spring boot 2.3 이상부터는 spring-boot-starter-web 의존성 내부에 있던 validation 이 사라져서  
2.3 이상이라면 위처럼 선언해서 사용해야 합니다.

<br>

모든 어노테이션을 여기서 다루진 않겠습니다.  
우선 간략한 사용방법만 다루겠습니다.

UserRequest.java

```java
@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
public class UserRequest {
    @Email(message = "이메일 형식이 맞지 않습니다.")
    @NotBlank(message = "이메일을 입력해주세요.")
    private String email;

    @NotBlank(message = "이름을 입력해주세요.")
    @Size(min = 2 , max = 5 , message = "이름은 2자 이상 , 5자 이하여야 합니다.")
    private String name;

    @NotBlank(message = "나이를 입력해주세요.")
    @Size(min = 20 , max = 100 , message = "나이는 20~100세 사이의 사용자만 가입이 가능합니다.")
    private int age;
}
```

- @Email: Email 형식인지 확인
- @NotBlank: null , 공백을 허용하지 않음
- @Size: 길이를 제한할때 사용(min: 최소 , max: 최대)

UserController.java

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/user")
public class UserController {
    private final UserService userService;

    @PostMapping(value = "")
    public ResponseEntity<UserDTO> create(@Valid UserRequest userRequest) {
				// logic...
        return new ResponseEntity<>(HttpStatus.OK);
    }
```

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
