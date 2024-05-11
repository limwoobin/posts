![component-vs-configuration]()

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

- 하는일 
- 문서기반
- proxyBeanMethods cglib


<br />

## __그래서 둘의 차이는 ??__

- bean 메서드 생성차이
- 외부의 라이브러리 빈 등록 (용도 차이)

<br />

## 예제코드

#### __reference__

- http://dimafeng.com/2015/08/29/spring-configuration_vs_component/
- https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/stereotype/Component.html
- https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Configuration.html