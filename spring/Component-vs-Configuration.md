![component-vs-configuration](https://private-user-images.githubusercontent.com/28802545/332433095-95f9d04f-e44d-46c9-95e3-94fb5262bc73.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTYyOTY1NDUsIm5iZiI6MTcxNjI5NjI0NSwicGF0aCI6Ii8yODgwMjU0NS8zMzI0MzMwOTUtOTVmOWQwNGYtZTQ0ZC00NmM5LTk1ZTMtOTRmYjUyNjJiYzczLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA1MjElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNTIxVDEyNTcyNVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTcxMDQ5NjQxODY1NzMzNmJjOTEyYWQ2OTBkZjM3YzMyZGQwZjlhMWQzZmE1NzU0MzJhM2Y5Mzg1ZTA5NTRmM2EmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.h99JBINcwGy0N5sPW91_BXDtFPfAQpuDAC3AZ3kji58)

#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/spring-config-example)

<br />

# __@Component VS @Configuration__

안녕하세요. 스프링을 사용하면서 위의 두 어노테이션을 자주 사용하고 계실텐데요.  
저도 마찬가지로 두 어노테이션을 자주 사용하지만 Bean 으로 선언하거나 생성할 수 있다는 것만 알고있지  
둘 사이에 어떤 차이가 있는지 그리고 어떤 상황에 사용해야 할지에 대해서는 명확하게 알고있지 않습니다.

그래서 둘 사이의 용도 그리고 동작방식에 어떤 차이가 있는지 좀 더 알아보기 위해 글을 작성하게 되었습니다.

<br />

## __@Component__

먼저 __`@Component`__ 에 대해 간단하게 어떤 어노테이션인지 알아보겠습니다.  
스프링 공식문서에서는 __`@Component`__ 를 다음과 같이 이야기 합니다.

```
Indicates that the annotated class is a component.
Such classes are considered as candidates for auto-detection when using annotation-based configuration and classpath scanning.

...(중략)
```

ApplicationContext 로딩시 __`@Component`__ 가 달린 클래스에 대해 Bean 으로 사용될 수 있도록 탐지한다는 의미로 볼 수 있습니다.  
즉, __`@Component`__ 가 달린 클래스에 대해서는 스프링에서 Bean 으로 사용하겠다는 의미입니다.

간단하게 __`@Component`__ 를 빈으로 등록하는 과정을 살펴보겠습니다.

![component-vs-configuration-1](https://private-user-images.githubusercontent.com/28802545/329785649-f9aadc0a-5a53-48c4-82a5-64b1d4d3ce43.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTU0MzYyNzEsIm5iZiI6MTcxNTQzNTk3MSwicGF0aCI6Ii8yODgwMjU0NS8zMjk3ODU2NDktZjlhYWRjMGEtNWE1My00OGM0LTgyYTUtNjRiMWQ0ZDNjZTQzLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA1MTElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNTExVDEzNTkzMVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWE4OGE0NjRiZDhiYzY1YjIyYzI4ZDg1ODkxZTI0NWQxNzg5MjUxOTljYzE5NmMyNWU0M2M1NzJjNGFhNjQ0OTkmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.Z6ra2CUfm5Wx2OkwVSEnuqaSIopnVmff8xHPO8Id2qM)

__`@Component`__ 가 정의된 클래스를 찾을때는 __`@ComponentScan`__ 어노테이션을 이용하여 찾게됩니다.  
__`@ComponentScan`__ 의 basePackageClasses or basePackages 를 지정하여 찾게됩니다. 지정된 패키지가 없다면 __`@ComponentScan`__ 가 정의된 패키지에서 부터 스캔이 이루어집니다.

### __`ComponentScanAnnotationParser`__

![component-vs-configuration-2](https://private-user-images.githubusercontent.com/28802545/329785960-5c92dd8a-2535-445e-a19d-227cb02259ff.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTU0MzYyNzEsIm5iZiI6MTcxNTQzNTk3MSwicGF0aCI6Ii8yODgwMjU0NS8zMjk3ODU5NjAtNWM5MmRkOGEtMjUzNS00NDVlLWExOWQtMjI3Y2IwMjI1OWZmLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA1MTElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNTExVDEzNTkzMVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTY0NjFlMjZmNGYxYmMwOGE5NDlhMWY2OTNkZDg5YjY3YmJiOTgxZWExMmMxM2ZlOTc4NjUxOTQ1ZTM5ODBmOGUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.YOFTAjzPGyxYGkBLD7IZZvnqUFIEowJzVYbp62oD7dc)

그리고 __`ComponentScanAnnotationParser`__ 클래스의 `parse` 메소드에서 __`@ComponentScan`__ 의 설정을 파싱하게 됩니다.  
이때 __`@Component`__ 를 스캔하기 위한 basePackages, excludeFilters, includeFilters 등등 여러 정보들을 파싱합니다.

### __`ClassPathBeanDefinitionScanner`__

![component-vs-configuration-3](https://private-user-images.githubusercontent.com/28802545/329787137-ac9bedab-6b09-43d5-b200-be6812bd476c.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTU0MzY2MDQsIm5iZiI6MTcxNTQzNjMwNCwicGF0aCI6Ii8yODgwMjU0NS8zMjk3ODcxMzctYWM5YmVkYWItNmIwOS00M2Q1LWIyMDAtYmU2ODEyYmQ0NzZjLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA1MTElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNTExVDE0MDUwNFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTJlNzgzYjBiZDNkMTVlMTM4OWEyMzgxMThmZmYwM2ZjNWVhOGZiMDUxZTQzMTM4Y2E0YjZmMDlhMGIzNTVjYWMmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.NifIG1EJYCGI3aYTy99RJjZgOd_1fpc4TLk5Z8VQENg)

>  A bean definition scanner that detects bean candidates on the classpath, registering corresponding bean definitions with a given registry (BeanFactory or ApplicationContext).
Candidate classes are detected through configurable type filters. The default filters include classes that are annotated with Spring's @Component, @Repository, @Service, or @Controller stereotype.

BeanFactory 클래스패스에서 Bean 후보를 탐지하고 해당 클래스를 BeanFactory 또는 ApplicationContext 에 등록하는 Scanner

Bean 등록 후보 클래스는 구성 가능한 필터를 통해 감지되는데 Spring의 @Component, @Repository, @Service, @Controller 어노테이션이 포함된다고 이야기 하고 있습니다.

즉,  __`ClassPathBeanDefinitionScanner`__ 클래스에서 __`@Component`__ 가 정의된 클래스를 스캔하고 Bean 으로 등록하는데요.

![component-vs-configuration-4](https://private-user-images.githubusercontent.com/28802545/329787903-44d7fd28-413c-4c58-b599-b7c3d456cd2a.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTU0MzcyMjMsIm5iZiI6MTcxNTQzNjkyMywicGF0aCI6Ii8yODgwMjU0NS8zMjk3ODc5MDMtNDRkN2ZkMjgtNDEzYy00YzU4LWI1OTktYjdjM2Q0NTZjZDJhLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA1MTElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNTExVDE0MTUyM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTBiYTE4ZTA5NTAyMjM2NWM3MzE5NDZkYzdhNjI4OTg3Yjg2ZmE5YmQwMzU3MTM0MWFkODhhNzJkMjMzMDcxZTImWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.vZ4aKeozQ9KyYoe-Eh-SMQk1aZBwlfEyApfyk4eZMI4)

- __`Set<BeanDefinition> candidates = findCandidateComponents(basePackage);`__ :Bean 등록 후보 클래스 조회
- __`String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);`__: Bean 이름 생성
- __`registerBeanDefinition(definitionHolder, this.registry);`__: Bean 등록


다음과 같이 Bean 등록이 되는것을 볼 수 있습니다.

<br />

## __@Configuration__

다음은 __@Configuration__ 에 대한 스프링 공식문서에서 설명하는 내용입니다.

> Indicates that a class declares one or more @Bean methods and may be processed by the Spring container to generate bean definitions and service requests for those beans at runtime, for example:

```java
@Configuration
 public class AppConfig {

     @Bean
     public MyBean myBean() {
         // instantiate, configure and return bean ...
     }
 }
```

__@Configuration__ 이 선언된 클래스는 하나 이상의 @Bean 메소드를 선언하고 해당 @Bean 메소드는 스프링 컨테이너를 통해 처리된다고 이야기하고 있습니다.

<br />

그리고 __`@Configuration`__ 내부에 __`@Component`__ 를 가지고 있기에 __`@Configuration`__ 이 선언된 클래스도 Bean으로 등록됩니다. 

__@Configuration__
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {

	@AliasFor(annotation = Component.class)
	String value() default "";

	boolean proxyBeanMethods() default true;

	boolean enforceUniqueMethods() default true;
}
```

<br />

또한, @Bean 메소드는 __@Configuration__ 내부에서 선언해야 __Singleton Scope__ 를 가질 수 있습니다.  
바로 __@Configuration__ 의 __proxyBeanMethods__ 설정때문입니다.

<br />

### __proxyBeanMethods__

> Specify whether @Bean methods should get proxied in order to enforce bean lifecycle behavior, __e.g. to return shared singleton bean instances even in case of direct @Bean method calls in user code.__ This feature requires method interception, implemented through a runtime-generated CGLIB subclass which comes with limitations such as the configuration class and its methods not being allowed to declare final.

Bean 메서드를 직접 호출하는 경우에도 싱글턴 Bean Instance 를 적용하기 위한 @Bean 메서드의 프록시 여부를 지정한다고 이야기합니다.  

즉, __proxyBeanMethods__ 이 true 이면 CGLib 프록시 패턴을 적용해 Bean 이 싱글톤으로 생성됩니다. false 라면 매번 호출시마다 새로운 객체가 반환됩니다.

<br />

예제코드

__ExamConfiguration.java__
```java
@Configuration
public class ExamConfiguration {

}
```

__ExamComponent.java__
```java
@Component
public class ExamComponent {

}
```

<br />

![component-vs-configuration-5](https://private-user-images.githubusercontent.com/28802545/332416256-88fcd9c5-b9f7-4995-937a-b39c63a7eb71.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTYyOTMyMDAsIm5iZiI6MTcxNjI5MjkwMCwicGF0aCI6Ii8yODgwMjU0NS8zMzI0MTYyNTYtODhmY2Q5YzUtYjlmNy00OTk1LTkzN2EtYjM5YzYzYTdlYjcxLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA1MjElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNTIxVDEyMDE0MFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWZmZTFjYTgzMWNhZDczY2ZjMTczODM3ZDZkMmUzMWYwNTVlMWNlMGM1NGMzYmRkNDcyYzkyZjVjODQ2OTg1ZmUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.KdnOEWRZpp0Ph0frvzVsGl0rbOJFsFZ7Qwl38z0NA7k)


다음과 같이 __`@Configuration`__ 이 선언된 클래스는 CHLIB Proxy 로 등록된것을 확인할 수 있습니다.

## __그래서 @Component, @Configuration 둘의 차이는 ??__

### 시용용도

__`@Component`__ 의 경우 일반적으로 직접 컨트롤 가능한 Class에 대해 선언합니다.  
자신이 작성한 Class 에 대해 __`@ComponentScan`__ 을 이용한 Bean 등록도 편리하고  
또한 __`@Component`__ 어노테이션은 클래스 레벨에 선언할 수 있습니다.

__@Component__
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Indexed
public @interface Component {
}
```

__`@Configuration`__ 의 경우에는 Bean 으로 등록하기 위해서는 __`@Bean`__ 이 선언된 메서드 내부에서 생성한 객체를 Bean 으로 등록할 수 있는데요.

__@Bean__
```java
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Bean {
}
```

다음과 같이 클래스 레벨에는 선언할 수 없습니다.  
외부 라이브러리를 Bean 으로 등록해서 사용하려는 경우 해당 코드를 직접 수정할 수 없기에 __`@Component`__ 사용할 수 없습니다.

대신 __`@Configuration`__ 의 __`@Bean method`__ 을 이용해서 Bean 으로 등록해서 사용하게 됩니다.

```java
@Configuration
public class ExampleConfiguration {

 @Bean
 public ObjectMapper objectMapper() {
   return new ObjectMapper();
 }
}
```

이렇듯, __`@Configuration`__ 은 외부 라이브러리같은 개발자가 제어할 수 없는 코드에 대해 Bean 으로 등록하거나 특정 Bean 들을 한곳에서 관리하고 싶을때 사용할 수 있습니다.

<br />

### Bean LifeCycle 예제코드

위에서 이야기했든 __`@Configuration`__ 에서 __`Bean Method`__ 를 선언하면 proxyBeanMethods 옵션에 따라 싱글턴으로 Bean 이 등록될 수 있다고 했습니다.

그렇다면 __`@Component`__ 에서는 __`Bean Method`__ 를 선언했을때 __`@Configuration`__ 와 어떻게 다를지 코드로 한번 살펴보겠습니다.

<br />

__Exam.java__
```java
public class Exam {
}
```

<br />

__ExamComponent.java__
```java
@Component
public class ExamComponent {

  @Bean(name = "exam2")
  public Exam exam() {
    return new Exam2();
  }
}
```

<br />

__ExamConfiguration.java__
```java
@Configuration
public class ExamConfiguration {

  @Bean
  public Exam exam() {
    return new Exam();
  }

}
```

<br />

__ExamTest.java__
```java
@SpringBootTest
public class ExamTest {

  @Autowired
  private ExamConfiguration examConfiguration;

  @Test
  void configuration_test() {
    Exam exam = examConfiguration.exam();
    Exam exam2 = examConfiguration.exam();
    Exam exam3 = examConfiguration.exam();
    Exam exam4 = examConfiguration.exam();
    Exam exam5 = examConfiguration.exam();

    System.out.println("exam: " + exam);
    System.out.println("exam2: " + exam2);
    System.out.println("exam3: " + exam3);
    System.out.println("exam4: " + exam4);
    System.out.println("exam5: " + exam5);
  }
}    
```

![component-vs-configuration-6](https://private-user-images.githubusercontent.com/28802545/332429755-c7dccd9e-91e0-47ec-a6ba-ecebb2fbc67a.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTYyOTU5MzksIm5iZiI6MTcxNjI5NTYzOSwicGF0aCI6Ii8yODgwMjU0NS8zMzI0Mjk3NTUtYzdkY2NkOWUtOTFlMC00N2VjLWE2YmEtZWNlYmIyZmJjNjdhLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA1MjElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNTIxVDEyNDcxOVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTU5NjMzNzkzMzIyM2ViNzIzMzk3OTkyYjdmNzJiNDM3ODdjYmFhNDkyMGZiYzczZjYzNzBiNWZjN2JjOWZhOTEmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.Fpzmc-jGC18U3HJyqZNAkB_WE4QU9K603HbdaHe_I_A)


__`@Configuration`__ 이 싱글턴으로 Bean 을 생성한다면 해당 테스트 코드에서 Exam 객체의 모든 값은 동일해야 합니다.  
모든 객체의 값이 동일한 것을 보아 싱글턴으로 Bean 이 등록된것을 알 수 있습니다.

<br />

__ExamTest.java__
```java
@SpringBootTest
public class ExamTest2 {

  @Autowired
  private ExamComponent examComponent;

  @Test
  void configuration_test() {
    Exam exam = examComponent.exam();
    Exam exam2 = examComponent.exam();
    Exam exam3 = examComponent.exam();
    Exam exam4 = examComponent.exam();
    Exam exam5 = examComponent.exam();

    System.out.println("exam: " + exam);
    System.out.println("exam2: " + exam2);
    System.out.println("exam3: " + exam3);
    System.out.println("exam4: " + exam4);
    System.out.println("exam5: " + exam5);
  }
}    
```

![component-vs-configuration-7](https://private-user-images.githubusercontent.com/28802545/332430481-eecddc13-298c-483d-9429-cf85a96027fb.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTYyOTYwNzgsIm5iZiI6MTcxNjI5NTc3OCwicGF0aCI6Ii8yODgwMjU0NS8zMzI0MzA0ODEtZWVjZGRjMTMtMjk4Yy00ODNkLTk0MjktY2Y4NWE5NjAyN2ZiLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA1MjElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNTIxVDEyNDkzOFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWQyY2VhNjJkYjIyNmMzZGEzZDc3MzU5YmQ4NGNlM2FlNmU4OTBmZWM2ZDU0NGFkOGJmNjIwZjJiN2MxOGQ5YjEmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.0ocmFvpnBwMLdrDfwPIaQYy6dWYzwZpjxUuvegv2KIM)

__`@Component`__ 는 생성된 모든 Bean 객체의 값이 상이한걸 확인할 수 있습니다. 이렇듯 __`@Component`__ 내에서 __`Bean Method`__ 를 선언하게 되면 싱글턴으로 생성되는것이 아니라 호출시마다 새로운 Bean 이 생성되는것을 확인할 수 있습니다.

감사합니다.

<br />

#### __reference__

- http://dimafeng.com/2015/08/29/spring-configuration_vs_component/
- https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/stereotype/Component.html
- https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Configuration.html
- https://mangkyu.tistory.com/75