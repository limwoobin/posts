> <br>
>
> ## **Overview**
>
> - ### [**Valid 사용하기**](#valid-사용하기)
>
>   - [**사용 예제**](#사용-예제)
>
> - ### [**Controller 에서 에러 메시지 처리하기**](#controller-에서-에러-메시지-처리하기-1)
>
>   - [**GetMapping**](#getmapping)
>   - [**PostMapping**](#postmapping)
>
> - ### [**advice 를 이용한 exception handling 적용**](#advice-를-이용한-exception-handling-적용-1)
>   - [**advice 를 이용한 handling 방법**](#advice-를-이용한-handling-방법) <br><br>

<br />

#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/valid-example)

<br />
<br />

# **Valid 사용하기**

Spring 에서는 유효성 체크를 위하여 @Valid annotation 을 지원합니다.  
Valid는 [**JSR-303(Bean Validation)**](https://beanvalidation.org/1.0/spec/) 표준 스펙으로서 제약조건이 있는 객체에게 Bean Validation 을 이용해 조건을 검증하는 어노테이션입니다.

<br>

## **사용 예제**

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

```javascript
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

![valid-test-code-3](https://user-images.githubusercontent.com/28802545/149120391-2727c0b2-0907-4300-8f97-bbff00494991.png)

정상적으로 테스트가 성공한것을 확인할 수 있습니다.

<br>

# **Controller 에서 에러 메시지 처리하기**

위의 예시에서 @Valid 옵션에 따라 파라미터를 검증하는것을 확인했습니다.  
하지만 코드에 기재해놓은 message에 대해서는 전혀 찾아볼 수 없습니다.  
@Valid 에 대한 예외를 확인하려면 BindingResult 라는 객체가 필요합니다.

<br>

## **GetMapping**

<hr>

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
테스트 코드를 통해 확인해보겠습니다.

```java
@DisplayName("BindResult Valid 테스트")
class BindResultValidTest {

    static final String EXPECTED_EMAIL_ERR_MESSAGE = "이메일 형식이 맞지 않습니다.";

    @Test
    @DisplayName("Valid 조건에 맞지 않는 파라미터를 Get으로 넘기면 status 400, 에러메시지를 응답받아야 한다")
    void validTest_get() throws Exception {
        // given
        UserRequest userRequest = UserRequest.builder()
                .email("drogba02")
                .name("woobeen")
                .age(29)
                .build();

        // then
        mockMvc.perform(get("/user/v2")
                .param("email" , userRequest.getEmail())
                .param("name" , userRequest.getName())
                .param("age" , Integer.toString(userRequest.getAge())))
                .andExpect(status().isBadRequest())
                .andExpect(content().string(EXPECTED_EMAIL_ERR_MESSAGE));
    }
}
```

![valid-test-code-4](https://user-images.githubusercontent.com/28802545/150322229-31e89f0f-a977-4fc3-9865-a3870bc7d7d3.png)

status 400 에 "**이메일 형식이 맞지 않습니다.**" 라는 문자열을 리턴해야 합니다.  
정상적으로 테스트가 통과된것을 확인할 수 있습니다.

<br>

## **PostMapping**

<hr>

이번엔 Post 방식으로도 한번 테스트해보겠습니다.

```java
@PostMapping(value = "/v2")
public ResponseEntity createForPost(@Valid @RequestBody UserRequest userRequest , BindingResult bindingResult) {
    if (bindingResult.hasErrors()) {
        List<FieldError> list = bindingResult.getFieldErrors();
        for(FieldError error : list) {
            return new ResponseEntity<>(error.getDefaultMessage() , HttpStatus.BAD_REQUEST);
        }
    }

    return new ResponseEntity<>(HttpStatus.CREATED);
}


@Test
@DisplayName("Valid 조건에 맞는 파라미터를 Post로 넘기면 성공해야 한다")
void validTest_post_ok() throws Exception {
    // given
    UserRequest userRequest = UserRequest.builder()
            .email("drogba02@naver.com")
            .name("woobeen")
            .age(29)
            .build();

    String jsonData = objectMapper.writeValueAsString(userRequest);

    // then
    mockMvc.perform(post("/user/v2")
            .content(jsonData)
            .contentType(MediaType.APPLICATION_JSON)
            .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isCreated())
            .andExpect(content().string(""));
}
```

![valid-test-code-5](https://user-images.githubusercontent.com/28802545/150325183-4d09d408-07c4-43fd-bf25-c675f625f0f6.png)

Post 방식도 동일합니다.

<br>

# **advice 를 이용한 exception handling 적용**

위에서의 BindResult 를 선언해서 에러를 처리하다보니 몇가지 문제점이 보였습니다.

- valid 를 사용하는 controller 마다 직접 에러처리 로직을 작성해야 한다.
- 코드의 중복이 발생한다.
- 일관성 있는 예외처리를 보장할 수 없다.

이와 같은 문제점들을 해결하기 위해 Spring의 ControllerAdvice 를 이용하여 공통 예외처리를 만들어보겠습니다.

<br>

## **advice 를 이용한 handling 방법**

@Valid 에서 발생한 예외를 캐치해서 응답하는 로직을 만들어보겠습니다.

우선 Advice class 를 만들어보겠습니다.

ExceptionAdvice.class

```java
@RestControllerAdvice
public class ExceptionAdvice extends ResponseEntityExceptionHandler {
     @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(MethodArgumentNotValidException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
        BindingResult result = ex.getBindingResult();
        StringBuilder errMessage = new StringBuilder();

        for (FieldError error : result.getFieldErrors()) {
            errMessage.append("[")
                    .append(error.getField())
                    .append("] ")
                    .append(":")
                    .append(error.getDefaultMessage());
        }

        return new ResponseEntity<>(errMessage , HttpStatus.BAD_REQUEST);
    }
}
```

@Valid 에 대한 예외는 MethodArgumentNotValidException 를 발생시킵니다.  
그래서 ResponseEntityExceptionHandler 의 handleMethodArgumentNotValid method 를 오버라이딩하여 재정의해서 예외를 캐치한 다음 응답하는 로직을 작성했습니다.

```java
@PostMapping(value = "")
public ResponseEntity createForPost(@Valid @RequestBody UserRequest userRequest) {

    return new ResponseEntity(HttpStatus.CREATED);
}
```

테스트할 Controller 는 다음과 같습니다.  
예외를 advice에서 처리할테니 더 이상 controller는 BindingResult 객체가 필요없겟죠?

PostMapping 에 대한 테스트 코드를 먼저 작성해보겠습니다.

```java
@Nested
@DisplayName("Valid Advice 테스트")
class ValidAdviceTest {

    static final String EMAIL_EXCEPTION_MESSAGE = "이메일 형식이 맞지 않습니다.";
        static final String AGE_EXCEPTION_MESSAGE = "나이는 20~100세 사이의 사용자만 가입이 가능합니다.";

    @Test
    @DisplayName("PostMapping시 Valid 예외가 Advice 에서 정상적으로 처리되어야 한다")
    void advice_post_test() throws Exception {
        // given
        UserRequest userRequest = UserRequest.builder()
                .email("drogba02")
                .name("woobeen")
                .age(18)
                .build();

        String jsonData = objectMapper.writeValueAsString(userRequest);

        // then
        mockMvc.perform(post("/user")
                .content(jsonData)
                .contentType(MediaType.APPLICATION_JSON)
                .accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isBadRequest())
                .andExpect(content().string(containsString(EMAIL_EXCEPTION_MESSAGE)))
                .andExpect(content().string(containsString(AGE_EXCEPTION_MESSAGE)))
                .andDo(print());
    }
}
```

일부러 email , age가 Valid에 걸리게끔 테스트 코드를 작성했습니다.  
예상대로라면 응답받은 메시지는 email , age 에 대한 메시지가 포함되어있어야 정상입니다. 실행해볼까요?

![valid-test-code-6](https://user-images.githubusercontent.com/28802545/150508036-6c6abea4-3a02-41f6-b626-17366e8b4966.png)

정상적으로 **ControllerAdvice** 가 동작하는것을 확인할 수 있습니다.

그렇다면 이제 **GetMapping**에 대한 테스트도 진행해보겠습니다.

```java
@Nested
@DisplayName("Valid Advice 테스트")
class ValidAdviceTest {

    static final String EMAIL_EXCEPTION_MESSAGE = "이메일 형식이 맞지 않습니다.";
        static final String AGE_EXCEPTION_MESSAGE = "나이는 20~100세 사이의 사용자만 가입이 가능합니다.";

    @Test
    @DisplayName("GetMapping시 Valid 예외가 Advice 에서 정상적으로 처리되어야 한다")
    void advice_post_get() throws Exception {
        // given
        UserRequest userRequest = UserRequest.builder()
                .email("drogba02")
                .name("woobeen")
                .age(18)
                .build();

        // then
        mockMvc.perform(get("/user")
                .param("email" , userRequest.getEmail())
                .param("name" , userRequest.getName())
                .param("age" , Integer.toString(userRequest.getAge())))
                .andExpect(status().isBadRequest())
                .andExpect(content().string(containsString(EMAIL_EXCEPTION_MESSAGE)))
                .andExpect(content().string(containsString(AGE_EXCEPTION_MESSAGE)))
                .andDo(print());
    }
}
```

예상대로라면 위와 같이 400 error 를 리턴하고 email , age 에 대한 메시지를 리턴해줘야 정상입니다.  
결과보겠습니다.

![valid-test-code-7](https://user-images.githubusercontent.com/28802545/150508451-19381113-aa2c-47fd-b1b3-05aee0ce956d.png)

하지만 실패했습니다. 에러 메시지를 보니 status()는 예상대로 받은것 같지만 에러메시지를 가져오지 못했네요.  
왜 Post 시에는 가져오고 Get 으로는 못가져왔을까요?

**<u>저희가 advice에서 선언한 **MethodArgumentNotValidException** 는 @RequestBody에 대한 exception 을 처리해주기 때문인데요.</u>**

**<u>위 GetMapping 테스트코드는 ModelAttribute 방식으로 객체에 바인딩되게 됩니다.</u>**  
**<u>ModelAttribute 방식으로 받은 파라미터에 대한 예외를 처리해주기 위해서는 **BindException** 을 advice에 선언해줘야 합니다.</u>**

advice 에 BindException 에 대한 처리를 추가하겠습니다.

```java
@Override
protected ResponseEntity<Object> handleBindException(BindException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {
    BindingResult result = ex.getBindingResult();
    StringBuilder errMessage = new StringBuilder();

    for (FieldError error : result.getFieldErrors()) {
        errMessage.append("[")
                .append(error.getField())
                .append("] ")
                .append(":")
                .append(error.getDefaultMessage());
    }

    log.info("errMsg ### {}" , errMessage);
    return new ResponseEntity<>(errMessage , HttpStatus.BAD_REQUEST);
}
```

ResponseEntityExceptionHandler 클래스의 handleBindException 메소드를 재정의하여 BindException 에 대한 처리를 작성했습니다.  
다시 테스트를 해보겠습니다.

![valid-test-code-8](https://user-images.githubusercontent.com/28802545/150510293-28d38f91-79a7-417d-9898-437ceacd429d.png)

이번엔 정상적으로 테스트가 통과되는것을 확인할수 있습니다.
<br>
