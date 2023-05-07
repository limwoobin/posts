# __Spring Expresion Language(SpEL) 이란?__

#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/spel-example)

스프링 공식 문서에서는 SpEL을 다음과 같이 설명합니다.

> SpEL은 런타임 시 객체 그래프 쿼리 및 조작을 지원하는 강력한 표현언어이다.

##### 출처: https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/expressions.html__

<br>

| Type  | Operators  |
| --- | ----- |
| Arithmetic   | +, -, *, /, %, ^, div, mod |
| Relational   | <, >, ==, !=, <=, >=, lt, gt, eq, ne, le, ge |
| Logical   | 	and, or, not, &&, ||, ! |
| Conditional   | 	and, or, not, &&, ||, ! |
| Regex   | 	matches |

<br>

## __SpEL 사용법__
SpEL 표현식은 다음과 같은 규칙이 있습니다.
- `#` 기호로 시작하고 중괄호로 묶는다. ex) `#{expression}`
- 속성은 `$` 기호로 시작하고 중괄호로 감싼다. ex) `${property.name}`

스프링에서 `SpEL`이 자주 사용되는 어노테이션은 대략 다음과 같은 어노테이션들이 있습니다.
- @Value
- @PreAuthorize
- @PostAuthorize
- @PreFilter
- @PostFilter

표현식은 다음과 같이 사용할 수 있습니다.

```java
@Value("#{19 + 1}") // 20
private double add; 

@Value("#{'String1 ' + 'string2'}") // "String1 string2"
private String addString; 

@Value("#{1 == 1}") // true
private boolean equal;

@Value("#{250 > 200 && 200 < 4000}") // true
private boolean and; 

@Value("#{2 > 1 ? 'a' : 'b'}") // "a"
private String ternary;

@Value("#{'100' matches '\\d+' }") // true
private boolean validNumericStringResult;
...
```

<br>

## __SpEL 파싱__

### __ExpressionParser.class__

`ExpressionParser` 는 문자열 표현을 파싱하는 책임을 가지고 `Expression` 는 문자열 표현의 평가를 책임집니다.

`ExpressionParser, Expression` 클래스를 이용하면 다음과 같이 SpEL을 파싱할 수 있습니다.

```java
ExpressionParser expressionParser = new SpelExpressionParser();
Expression expression = expressionParser.parseExpression("'Any string'");
String result = (String) expression.getValue(); // "Any string"

Expression expression = expressionParser.parseExpression("'Any string'.length()");
Integer result = (Integer) expression.getValue(); // 10
```

<br>

### __EvaluationContext.class__

`EvaluationContext` 는 프로퍼티, 메서드, 필드를 확인하고 타입 변환을 수행하는 표현식을 평가할 때 사용됩니다.  

`EvaluationContext` 는 다음 두가지 구현체를 제공합니다.

- `SimpleEvaluationContext`
    - SpEL의 전체 범위가 필요하지 않으며 의미 있게 제한되어야 할 때 사용
    - 자바 타입 참조, 생성자 및 bean 참조를 지원하지 않음
- `StandardEvaluationContext`
    - 리플렉션을 기반으로 SpEL의 모든 기능 제공

```java
Car car = new Car();
car.setMake("Good manufacturer");
car.setModel("Model 3");

ExpressionParser expressionParser = new SpelExpressionParser();
Expression expression = expressionParser.parseExpression("model");

EvaluationContext context = new StandardEvaluationContext(car);
String result = (String) expression.getValue(context);  // "Model 3"
```

<br>

# __어노테이션에서 SpEL 활용하기__

이번엔 SpEL을 이용해 커스텀 어노테이션에 변수를 넘겨서 사용하는 방법에 대해 알아보겠습니다.

코드를 작성하다보면 공통 처리를 위해 `annotation + aop` 조합을 사용하는 경우가 있습니다. 이럴 때 활용할 수 있습니다.


__CustomAnnotation.java__

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface CustomAnnotation {
    String value() default "";
}
```

<br>

__SpELApplication.java__
```java
@Component
@Slf4j
public class SpELApplication {

    @CustomAnnotation(value = "#value")
    public void call(String value) {
    }
}
```
call 메소드를 호출하면 value변수를 어노테이션에 전달하고 AOP에서는 
해당 value변수를 실제 참조를 통해 읽어와야 합니다.

<br>

__CustomAnnotationAop.java__
```java
@Aspect
@Component
@Slf4j
public class CustomAnnotationAop {

    @Around("@annotation(com.example.spelexample.aop.CustomAnnotation)")
    public Object aopCall(final ProceedingJoinPoint joinPoint) throws Throwable {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        CustomAnnotation customAnnotation = method.getAnnotation(CustomAnnotation.class);

        String value = (String) CustomSpELParser.getDynamicValue(signature.getParameterNames(), joinPoint.getArgs(), customAnnotation.value());
        log.info("aop value ##### {}", value);

        return joinPoint.proceed();
    }
}
```
- AOP의 `MethodSignature` 를 이용해 메소드에 전달받은 파라미터 네임과 arguments를 `CustomSpELParser`로 전달합니다.  
그리고 `CustomAnnotation` 의 value도 같이 전달합니다.

<br>

__CustomSpELParser.java__
```java
import org.springframework.expression.EvaluationContext;
import org.springframework.expression.ExpressionParser;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;

public class CustomSpELParser {
    public static Object getDynamicValue(String[] parameterNames, Object[] args, String name) {
        ExpressionParser parser = new SpelExpressionParser();
        EvaluationContext context = new StandardEvaluationContext();

        for (int i = 0; i < parameterNames.length; i++) {
            context.setVariable(parameterNames[i], args[i]);
        }

        return parser.parseExpression(name).getValue(context);
    }
}
```

EvaluationContext에 전달받은 파라미터 네임과 arguments들을 모두 매핑합니다. 이후 가져올 값을 표현식으로 꺼내 리턴합니다.

![spel-image-1](https://user-images.githubusercontent.com/28802545/236626313-ab698679-1707-48da-a9c1-99d9e9e8ba2e.png)

`name`에 `#value` 라는 값이 들어있는 것을 확인할 수 있습니다.  
`CustomSpELParser` 클래스에서는 context 세팅을 생성자나 `setRootObject`가 아닌 `setVariable`를 이용하여 context에 값을 세팅했습니다. 

![spel-image-2](https://user-images.githubusercontent.com/28802545/236627323-a7c83827-b1ff-4047-9791-6a7bd601df03.png)

그리고 `name`을 `#value` 라는 값으로 전달하여 `value`로 파싱해 context에서 value에 해당하는 값을 꺼내옵니다.  

`InternalSpelExpressionParser.class` 내부를 살펴보면 `#value`의 맨 앞인 `#(hash)` 를 자르고 
`#value` 문자열에서 `value` 만 가져올수있게 position을 1, 6으로 설정하는 것을 확인 할 수 있습니다.

__InternalSpelExpressionParser.class__

![spel-image-3](https://user-images.githubusercontent.com/28802545/236629371-079f5250-42ec-4706-91cc-8ea893bc5f92.png)

<br>

__SpELTest.java__
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class SpELTest {
    @Autowired
    private SpELApplication spELApplication;

    @Test
    void spELCallTest() {
        spELApplication.spELCall("test annotation");
    }
}
```

그리고 해당 메소드를 호출하면 전달한 파라미터 값인 `test annotation` 가 로그에 찍힌것을 확인할 수 있습니다.

![spel-image-4](https://user-images.githubusercontent.com/28802545/236626690-3f9731df-f8fd-4e9b-885a-3498074037b9.png)

<hr>
<br>

#### reference
https://www.baeldung.com/spring-expression-language  
https://blog.outsider.ne.kr/835  
https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/expression