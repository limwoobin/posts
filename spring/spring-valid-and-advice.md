> <br>
>
> ## **Overview**
>
> - ### [**@Valid 사용하기**](#valid-사용하기)
>
>   - [**사용 예제**](#사용-예제)
>   - [**Controller 에서 처리하기**](#content-1)
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

### **사용 예제**

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
    @Size(min = 2 , max = 10 , message = "이름은 2자 이상 , 5자 이하여야 합니다.")
    private String name;

    @NotBlank(message = "나이를 입력해주세요.")
    @Size(min = 20 , max = 100 , message = "나이는 20~100세 사이의 사용자만 가입이 가능합니다.")
    private int age;
}
```

- **@Email:** Email 형식인지 확인
- **@NotBlank:** null , 공백을 허용하지 않음
- **@Size:** 길이를 제한할때 사용(min: 최소 , max: 최대)
- **@max:** 지정한 값 이하인지
- **@min:** 지정한 값 이상인지

<br>

UserController.java

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/user")
public class UserController {
    private final UserService userService;

    @PostMapping(value = "")
    public ResponseEntity<UserDTO> create(@Valid UserRequest userRequest) {

        return new ResponseEntity<>(HttpStatus.CREATED);
    }
```

@Valid 만 체크하기 위해 별다른 로직 없이 바로 201 code 를 리턴하도록 만들었습니다.  
그렇다면 @Valid를 확인해보기 위해 테스트 코드를 작성해보겠습니다.

<br>

```java
@Test
@DisplayName("Valid 조건에 맞지 않는 파라미터를 넘기면 실패해야 한다")
void validTest() throws Exception {
    // given
    UserRequest userRequest = UserRequest.builder()
            .email("drogba02")
            .name("woobeen")
            .age(29)
            .build();

    // then
    mockMvc.perform(get("/user")
            .param("email" , userRequest.getEmail())
            .param("name" , userRequest.getName())
            .param("age" , Integer.toString(userRequest.getAge())))
            .andExpect(status().isCreated());
}
```

파라미터값을 보시면 이름과 나이는 validation에 알맞는 값이지만 email 값은 형식에 맞지 않습니다.  
그렇다면 이 테스트코드는 실패해야 정상입니다.  
실행해보겠습니다

![valid-test-code-1](https://user-images.githubusercontent.com/28802545/149118524-4fc6afbd-1080-4cc4-8a8e-4f95d3bdbe82.png)
![valid-test-code-2](https://user-images.githubusercontent.com/28802545/149118540-9898e83a-f6f2-44f8-8ce0-e3d344e71002.png)

예상대로 실패하였습니다.

그렇다면 이번엔 모든 파라미터가 정상적인 상황에서의 테스트를 진행해보겠습니다.

```java
@Test
@DisplayName("Valid 조건에 맞는 파라미터를 넘기면 성공해야 한다")
void validTest2() throws Exception {
    // given
    UserRequest userRequest = UserRequest.builder()
            .email("drogba02@naver.com")
            .name("woobeen")
            .age(29)
            .build();

    // then
    mockMvc.perform(get("/user")
                    .param("email" , userRequest.getEmail())
                    .param("name" , userRequest.getName())
                    .param("age" , Integer.toString(userRequest.getAge())))
            .andExpect(status().isCreated());
}
```

위의 파라미터를 확인해보면 email , name , age 모두 validation 형식에 맞는 값임을 확인할 수 있습니다.  
그렇다면 이 테스트코드는 성공해야 정상입니다.

![valid-test-code-2](https://user-images.githubusercontent.com/28802545/149120391-2727c0b2-0907-4300-8f97-bbff00494991.png)

정상적으로 테스트가 성공한것을 확인할 수 있습니다.

<br>

### **Controller에서 에러 메시지 처리하기**

위의 예시에서 @Valid 옵션에 따라 파라미터를 검증하는것을 확인했습니다.  
하지만 코드에 기재해놓은 message에 대해서는 전혀 찾아볼 수 없습니다.  
@Valid 에 대한 예외를 확인하려면 BindResult 라는 객체가 필요합니다.

```java
@GetMapping(value = "/v2")
    public ResponseEntity create(@Valid UserRequest userRequest , BindingResult bindingResult) {
        if (bindingResult.hasErrors()) {
            List<FieldError> list = bindingResult.getFieldErrors();
            for(FieldError error : list) {
                return new ResponseEntity<>(error.getDefaultMessage() , HttpStatus.BAD_REQUEST);
            }
        }

        return new ResponseEntity<>(HttpStatus.CREATED);
    }
```

validation 에 맞지 않는 값이 있으면 즉시 return 하도록 작성했습니다.

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
