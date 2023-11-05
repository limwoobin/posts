[**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/spring-test-context)

<br>

## __스프링에서 서로 다른 테스트 클래스에서 테스트 데이터를 공유하는 방법__

이번에는 Spring 환경에서 서로 다른 테스트 클래스에서 데이터를 공유하는 방법에 대해 소개해드리려 합니다.  
`Spring Integration Test` 를 작성하다 보면 테스트를 위한 데이터를 세팅하는 과정, 혹은 테스트를 위한 선행 작업 or 전처리 작업이 필요한 경우가 있습니다.  
이때, 일반적으로 많이 사용하는 방법으로는 `@BeforeEach`, `@BeforeAll` 과 같은 JUnit 라이프 사이클 어노테이션을 많이 이용하게 됩니다.

`@BeforeEach` 의 경우 테스트 마다 매번 실행되기에 테스트 간의 격리를 할 수 있어 보다 신뢰성 있는 테스트를 할 수 있다는 장점이 있지만 그만큼 테스트 수행시간이 오래걸리는 특징이 있습니다.  
`@BeforeAll` 의 경우 클래스에서 단 한번만 실행된다는 특징이 있지만 테스트를 위한 선행작업을 여러 테스트 클래스에서 해야 하는 경우 실행되는 테스트 클래스의 수 만큼 수행되게 됩니다.  
두 어노테이션 모두 반복되는 코드를 줄여주고 테스트간의 격리(ex. 데이터 초기화)를 할 수 있다는 점에서 큰 장점이 있다고 생각합니다.

어떤 테스트를 작성하는데 해당 테스트를 위한 선행 작업이 굉장히 오래걸리는 테스트가 있다고 가정해보겠습니다.  
그리고 해당 테스트는 굉장히 다양한 테스트 시나리오를 가지고 있습니다. 그렇기에 테스트 가독성을 위해 테스트를 시나리오별로 클래스로 분리하였습니다.  
이 테스트를 메소드 단위로, 혹은 클래스 단위로 테스트 선행 작업을 모두 수행하게 된다면 테스트 수행 시간은 굉장히 오래 걸리게 될텐데요.  
이때, 고민해 볼 수 있는 방법에 대해 공유해드리려 합니다.  

## __TestExecutionListener__



<br>

## reference

[https://www.baeldung.com/spring-testexecutionlistener](https://www.baeldung.com/spring-testexecutionlistener)  
[https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/context/support/DependencyInjectionTestExecutionListener.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/test/context/support/DependencyInjectionTestExecutionListener.html)  
