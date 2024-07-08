# __[Spring] @Async 비동기 처리 및 스레드 풀 설정__

![spring-async-image1](https://private-user-images.githubusercontent.com/28802545/346331401-fb31e27f-eacc-48fb-a126-589a7915ea76.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjAzNTI1MDksIm5iZiI6MTcyMDM1MjIwOSwicGF0aCI6Ii8yODgwMjU0NS8zNDYzMzE0MDEtZmIzMWUyN2YtZWFjYy00OGZiLWExMjYtNTg5YTc5MTVlYTc2LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MDclMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzA3VDExMzY0OVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWZlZTgzNTE3MDcwNTk4ZjJhNTY4NzQ3OTNhODliMWRiODAyNTkzMDEzZGQyMjcxNTE4YzhjNzlmMTEwMTNkY2EmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.ewNpMHZq0CppCaRsXH6DC4YFEVmNfbF427Qep5QwCeY)

#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/spring-async)

안녕하세요 이번에는 스프링에서 비동기처리를 할 수 있는 __`@Async`__ 에 대해 알아보겠습니다.

해당 어노태이션을 통해 실행된 비동기 함수는 별도의 스레드로 실행이 되게 됩니다.  
그렇기에 비동기 함수를 호출한 main 스레드에서는 해당 함수를 기다릴 필요가 없게됩니다.

그리고 비동기 함수를 실행하는 별도의 스레드는 스레드 풀을 통해 설정할 수 있습니다.

```java
@Async
public void asyncMethod() {
    // ...
}
```

<br />

## __@Async 의 리턴 타입__

__`@Async`__ 의 리턴 타입은 __Void, Future<T>, ListenableFuture<T>, CompletableFuture<T>__ 가 있습니다.  
(현재 ListenableFuture<T> 는 deprecated 되었습니다)

비동기로 처리하는 메소드의 결과값이 없다면 리턴타입은 __Void__ 를 사용하고  
결과값이 없다면 __Future<T>, ListenableFuture<T>, CompletableFuture<T>__ 를 사용합니다.

__ListenableFuture<T>__ 는 Deprecated 되었으니 이번 글에서는 제외하겠습니다.

리턴값이 있는 경우에는 __`AsyncResult<String>`__ 객체를 통해 반환하지만 Spring 6.0 부터는 Deprecated 되어 __`CompletableFuture<T>`__ 를 사용하라고 가이드하고 있습니다.

![spring-async-image2](https://private-user-images.githubusercontent.com/28802545/345425399-81b8ae43-1c49-4d9a-9448-83dd8f4715c4.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjAwMTA5OTIsIm5iZiI6MTcyMDAxMDY5MiwicGF0aCI6Ii8yODgwMjU0NS8zNDU0MjUzOTktODFiOGFlNDMtMWM0OS00ZDlhLTk0NDgtODNkZDhmNDcxNWM0LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MDMlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzAzVDEyNDQ1MlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTEwOTNkNTUzOTRmOGVjMDNmMGQyMjE4ZmIwMzE4NGQwZDdjYmVlZGM3ZmRlNDhhYjU4NzYyYmJlOWU1ZmM5ZGImWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.8ZFNhiD98NcFHtWw4B3E0Qyj-VEW07akXgkIcGRV6K8)

<br />

__Future 반환__
```java
@Async
public Future<String> asyncReturnFuture() {
    System.out.println("Execute Async Method Return Future: " + Thread.currentThread().getName());
    return new CompletableFuture<>();
}

// ...

public void getFuture() throws ExecutionException, InterruptedException {
    Future<String> future = asyncService.asyncReturnFuture();
    System.out.println(future.get());
}
```

__CompletableFuture 반환__
```java
@Async
public CompletableFuture<String> asyncReturnCompletableFuture() {
    String threadName = Thread.currentThread().getName();
    System.out.println("Execute Async Method Return CompletableFuture: " + threadName);
    return CompletableFuture.completedFuture(threadName);
}

// ...

public void getCompletableFuture() {
    CompletableFuture<String> completableFuture = asyncService.asyncReturnCompletableFuture();
    System.out.println(completableFuture.join());
}
```

두 객체 모두 __`CompletableFuture`__ 로 반환하여 사용할 수 있습니다.  
리턴 값이 필요한 경우 일반적으로 __`Future`__ 보다는 __`CompletableFuture`__ 사용을 권장드립니다.

__`Future`__ 의 경우에는 __`CompletableFuture`__ 에 비해 작업을 조합하거나 예외처리, 결과처리와 같은 부분들에 대해 사용의 불편함이 있기 때문입니다.  
(자세한 내용은 추후 CompletableFuture 를 다룰때 공유드리겠습니다.)

<br />

## __@Async 사용 방법 (예제 코드)__

비동기 함수를 실행하는 예제 코드를 작성해보겠습니다.

__AsyncService.java__
```java
@Slf4j
@Service
public class AsyncService {

  @Async
  public void asyncMethod() {
    try {
        Thread.sleep(2000);
        log.info("thread name: {}", Thread.currentThread().getName());
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
  }
```

비동기로 __`asyncMethod()`__ 를 선언했습니다. 해당 메소드에서는 sleep 을 주어 2초간 대기하도록 하였습니다.

__AsyncConfig.java__
```java
@EnableAsync
@Configuration
public class AsyncConfig {
}
```

__`@Async`__ 를 사용하기 위해서는 __`@EnableAsync`__ 를 선언해주어야 합니다.

__Controller.java__
```java
@Slf4j
@RestController
@RequestMapping(value = "/")
public class ExampleController {
  private final AsyncService asyncService;

  public ExampleController(AsyncService asyncService) {
    this.asyncService = asyncService;
  }

  @GetMapping
  public ResponseEntity<Void> temp() {
    log.info("controller start ...");
    asyncService.asyncMethod();
    log.info("controller end ...");
    return new ResponseEntity<>(HttpStatus.OK);
  }
}
```

비동기로 동작하는지 확인하기 위한 컨트롤러를 하나 생성했습니다.  
그리고 비동기로 잘 동작하는지 확인하기 위한 로그를 남겨놓았는데요.

비동기로 동작한다면 'controller start...' 로그가 찍히고 이후 2초간 대기하지 않고 바로 'controller end...' 로그가 찍혀야 합니다. 

테스트 해보겠습니다.

![spring-async-image3](https://private-user-images.githubusercontent.com/28802545/345436786-b2aa765f-a784-4934-b825-7abd1b7e154c.gif?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjAwMTI5ODYsIm5iZiI6MTcyMDAxMjY4NiwicGF0aCI6Ii8yODgwMjU0NS8zNDU0MzY3ODYtYjJhYTc2NWYtYTc4NC00OTM0LWI4MjUtN2FiZDFiN2UxNTRjLmdpZj9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MDMlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzAzVDEzMTgwNlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTllZWUyNTcxZWUwMzM0MGRhZTdjMTEwYzNjZDBlYWVmZTg0YjRhYWU0MDlhMTY0Njk0YmZjNWE1OGM5MTRjY2EmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.EW30BPIYf1p7UeVT6x-j9k4nNgKnS5uLpioAoU3Ge1k)

다음과 같이 비동기로 동작하는것을 확인할 수 있습니다. 

<br />

## __@Async 의 ThreadPool 관리 (SimpleAsyncTaskExecutor, ThreadPoolTaskExecutor)__

__`@Async`__ 는 비동기로 동작할때 별도의 스레드로 동작한다고 하였습니다.  
이때, 해당 비동기 작업은 __`TaskExecutor`__ 인터페이스를 통해 실행되게 되는데요.  
여기서 사용되는 구현체는 __`SimpleAsyncTaskExecutor, ThreadPoolTaskExecutor`__ 가 있습니다.

### __SimpleAsyncTaskExecutor__

__`SimpleAsyncTaskExecutor`__ 는 매 실행마다 새로운 스레드를 생성하여 작업을 실행하게 됩니다.  
스레드를 재사용하지 않기 때문에 매 작업시마다 스레드를 계속 만들어내기에 성능상 이슈가 존재해서 주의해야 합니다.

__AsyncConfig.java__
```java
@EnableAsync
@Configuration
public class AsyncConfig {

  @Bean
  public Executor getAsyncExecutor() {
    return new SimpleAsyncTaskExecutor();
  }
}
```

__ThreadTest.java__
```java
@SpringBootTest
public class ThreadTest {

  @Autowired
  private AsyncService asyncService;

  @Test
  void test() {
    for (int i = 0; i < 50; i++) {
      asyncService.asyncMethod();
    }
  }
}
```

![spring-async-image4](https://private-user-images.githubusercontent.com/28802545/346317697-d9b35b24-cff5-450e-9f89-4d9373b83785.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjAzMzY5MjIsIm5iZiI6MTcyMDMzNjYyMiwicGF0aCI6Ii8yODgwMjU0NS8zNDYzMTc2OTctZDliMzViMjQtY2ZmNS00NTBlLTlmODktNGQ5MzczYjgzNzg1LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MDclMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzA3VDA3MTcwMlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTI1Zjk3Mjg4OTBiOTk0MGQ3OGEwODAxZTQxYTMwM2FmMGNmMTg3YTE5MjYyZjc1OTcxOGI2NzdkMjNmYTM0MDMmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.Ntsxn2S8s0F82_ru8PlsIVyYF9ovf4ZavKzC3_V0qlA)

간단히 비동기 메소드를 50번 호출한 테스트 코드입니다.  
__VisualVM__ 을 통해 확인해보니 정말 작업 수 만큼 스레드가 새로 생성된것을 확인할 수 있습니다.

그렇기에 __`SimpleAsyncTaskExecutor`__ 를 지양하고 __`ThreadPoolTaskExecutor`__ 를 사용하는것을 권장하고 있습니다.

<br />

### __ThreadPoolTaskExecutor__

__`ThreadPoolTaskExecutor`__ 은 스레드 풀을 이용해 스레드를 관리하고, 스레드 재사용을 통해 성능을 최적화 할 수 있습니다.

__AsyncConfig.java__
```java
@EnableAsync
@Configuration
public class AsyncConfig implements AsyncConfigurer {

  @Override
  public Executor getAsyncExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(100);
    executor.setQueueCapacity(50);
    executor.setThreadNamePrefix("custom-");
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());
    executor.initialize();
    return executor;
  }
}
```

다음과 같이 스레드 풀의 설정을 정의해야합니다.  
그래야 __`@Async`__ 사용시 해당 설정을 통해 스레드 풀을 사용할 수 있습니다.

__`ThreadPoolTaskExecutor`__ 의 스레드 풀 설정에 대해 알아보겠습니다.

### __Thread Size__

#### __corePoolSize__
- thread pool 에서 기본적으로 유지되는 스레드의 갯수
- 기본값: 1 (SpringBoot 에서는 AutoConfiguration 으로 인해 8로 설정됨)

#### __maxPoolSize__
- thread pool 에서 사용할 수 있는 최대 thread 갯수 
- 기본값: Integer.MAX_VALUE

#### __queueCapacity__
- thread pool 작업 큐의 사이즈 
- 내부적으로 LinkedBlockingQueue 를 생성함
- 기본값: Integer.MAX_VALUE

<hr>

### __Thread Name__

#### __threadNamePrefix__
- 생성되는 스레드 이름에 사용될 접두사

<hr>

### __RejectedExecutionHandler__

#### __TaskRejectedException__
모든 스레드가 작업중이고 BlockingQueue 에서도 추가 작업을 받을 수 없을때 더 이상 작업을 처리할 수 없다고 발생하는 예외입니다.

전략에 따라 예외발생 대신 다른 선택을 할 수 있습니다.
- __AbortPolicy(Default):__ __RejectedExecutionException__ 예외를 발생시킵니다.
- __CallerRunsPolicy__: ThreadPoolTaskExecutor 에 작업을 요청한 스레드에서 직접 실행합니다.  
(호출한 곳에서 작업을 실행)
- __DiscardPolicy__: 해당 작업들을 skip 합니다.
- __DiscardOldestPolicy__: Queue 에 있는 오래된 작업들을 삭제하고 새로운 작업을 추가합니다.

<hr>

### __ThreadPoolTaskExecutor__ 동작 과정

1. ThreadPool 에 작업이 등록되면 ThreadPool 의 coreSize 만큼 스레드 수가 존재하는지 확인한다.
2. ThreadPool 의 coreSize 보다 스레드 수가 적다면 대기 중인 스레드가 존재하더라도 새로 스레드를 생성한다.  
coreSize 이상의 스레드가 존재한다면 대기중인 스레드에게 작업을 할당한다.
3. ThreadPool 의 모든 스레드가 작업중이라면 BlockingQueue 에 작업을 넣어 대기시킨다.  
(queueCapacity 를 설정한 LinkedBlockingQueue)
4. 작업중인 스레드가 작업을 완료하면 BlockingQueue 에 처리할 작업이 있는지 확인한다.  
처리할 작업이 있다면 큐에서 작업을 가져와 처리한다.
5. BlockingQueue 도 가득찬 상태에서 새로운 작업이 들어온다면 새로운 스레드를 생성해서 작업을 할당한다.  
이때 스레드 수는 최대 maxPoolSize 만큼 생성할 수 있다.
6. BlockingQueue 도 가득차고 스레드 수도 maxPoolSize 만큼 생성되어 모두 작업중이라면 TaskRejectedException 이 발생한다.  
(이때 예외발생 외에 다른 정책을 지정할 수 있음)

그리고 __ThreadPoolTaskExecutor__ 를 설정하고 테스트를 진행해보았습니다.

![spring-async-image5](
https://private-user-images.githubusercontent.com/28802545/346324742-a05d1dc9-f4e3-490a-96cb-c5585825d78b.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjAzNDUyNjgsIm5iZiI6MTcyMDM0NDk2OCwicGF0aCI6Ii8yODgwMjU0NS8zNDYzMjQ3NDItYTA1ZDFkYzktZjRlMy00OTBhLTk2Y2ItYzU1ODU4MjVkNzhiLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MDclMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzA3VDA5MzYwOFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWJkNzFiMjJiZjYyNmM0Y2UwZGUxNTAxNDU1YjM1MzY2MjI2NDNkMWU3OTE5YjRmZmEyZGFmNGY0MzVkOTgwODQmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.ZakXW0-kvF77JEnRQ2alqXtfoQ22haCP8MDu2lJPUUQ)

__SimpleAsyncTaskExecutor__ 와 다르게 coreSize 10개를 통해 작업을 처리한것과 __threadNamePrefix__ 도 설정대로 적용된것을 확인할 수 있습니다.

<br />
<hr>

#### __waitForTasksToCompleteOnShutdown__
- true 이면 어플리케이션 종료시 Queue 에 있는 모든 작업을 완료하고 종료함

#### __awaitTerminationSeconds__
- 어플리케이션 종료시 큐에 남아있는 작업이 너무 많아 모두 처리되기 힘든경우 __awaitTerminationSeconds__ 설정을 통해 최대 대기시간을 설정할 수 있음

#### __keepAliveSeconds__
- 스레드 수가 __maxPoolSize__ 만큼 생성되고 이후 작업이 완료되어 스레드가 idle 상태가 되었을때 종료되기까지의 대기시간

<hr>

## Async 의 예외처리

기본적으로 비동기 메서드는 별도의 스레드에서 실행되기에 비동기 메서드에서 발생한 예외는 호출자에게 전파되지 않습니다.

비동기에서 발생한 예외는 __`AsyncUncaughtExceptionHandler`__ 를 통해 처리할 수 있습니다.

__CustomAsyncExceptionHandler.java__
```java
@Slf4j
public class CustomAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {
  @Override
  public void handleUncaughtException(Throwable ex, Method method, Object... params) {
    log.error("error message: {}", ex.getMessage());
  }
}
```

__AsyncUncaughtExceptionHandler__ 인터페이스를 구현한 커스텀 에러 핸들러 객체를 생성합니다.  
예외가 발생하면 해당 핸들러로 예외가 전파됩니다.

그리고 __AsyncConfig.java__ 에서 해당 핸들러를 선언합니다.

__AsyncConfig.java__
```java
@EnableAsync
@Configuration
public class AsyncConfig implements AsyncConfigurer {

  @Override
  public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
    return new CustomAsyncExceptionHandler();
  }
}
```

이렇게 비동기에 대한 예외를 처리할 수 있습니다.

<br />

## __@Asnyc 주의사항 (proxy)__

__@Async__ 는 기본적으로 프록시 방식으로 동작합니다.  

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AsyncConfigurationSelector.class)
public @interface EnableAsync {
  // ...

  AdviceMode mode() default AdviceMode.PROXY;

  // ...
}
```

그렇기에 일반적인 프록시 모드 사용할때와 동일한 제약사항이 존재합니다.

- __public method 에서만 사용 가능하다__
- __self invocation 호출시 동작하지 않는다__

## 기본 TaskExecutor 가 이상하다.. 문서에 내용이 맞나?

__`@EnableAsync`__ 의 Javadoc 을 보면 스프링은 비동기 작업시 기본적으로 __`SimpleAsyncTaskExecutor`__ 설정을 사용한다고 합니다.

> By default, Spring will be searching for an associated thread pool definition: either a unique org.springframework.core.task.TaskExecutor bean in the context, or an java.util.concurrent.Executor bean named "taskExecutor" otherwise. If neither of the two is resolvable, a org.springframework.core.task.SimpleAsyncTaskExecutor will be used to process async method invocations. Besides, annotated methods having a void return type cannot transmit any exception back to the caller. By default, such uncaught exceptions are only logged.

__@EnableAsync__ 의 JavaDoc 의 내용입니다.  
기본적으로 TaskExecutor 는 __`org.springframework.core.task.SimpleAsyncTaskExecutor`__ 를 사용한다고 합니다.

하지만 제 SpringBoot 환경에서는 아무런 비동기 설정을 하지 않았음에도 기본값으로 __`ThreadPoolTaskExecutor`__ 설정을 사용하는것을 확인했습니다.

__ThreadTest.java__
```java
@SpringBootTest
public class ThreadTest {

  @Autowired
  private AsyncService asyncService;

  @Test
  void test() {
    for (int i = 0; i < 500; i++) {
      asyncService.asyncMethod();
    }
  }
}
```

비동기 작업 500개를 실행시키고 스레드 생성이 어떻게 되는지 확인해보았습니다.  
__SimpleAsyncTaskExecutor__ 를 사용한다면 매번 새로운 스레드를 생성하니 스레드도 500개가 생성될것으로 예상했습니다.

![spring-async-image6](https://private-user-images.githubusercontent.com/28802545/346327528-a7725f7d-2309-4acd-889c-1b9315efe661.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjAzNDgyODEsIm5iZiI6MTcyMDM0Nzk4MSwicGF0aCI6Ii8yODgwMjU0NS8zNDYzMjc1MjgtYTc3MjVmN2QtMjMwOS00YWNkLTg4OWMtMWI5MzE1ZWZlNjYxLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MDclMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzA3VDEwMjYyMVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTZlOTUwYzZlNDE1ZDY2NTBiMWNkZmNkMGQ4ZjBiZGM1YzFlYTRiNmZmZmE2YWQ0ZDBlMmY0MmYwY2ZiMzg5MzcmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.nj2kd85nifGsnX_V5bBpVJGSWQHn7L8gcOwl4x28RIk)

하지만 스레드는 8개만 생성된것을 확인할 수 있었습니다.  
8개는 __ThreadPoolTaskExecutor__ 의 기본 corePoolSize 입니다.

![spring-async-image7](https://private-user-images.githubusercontent.com/28802545/346327243-87763f89-eecc-4fab-bc93-1c8537fcd6fe.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjAzNDgyNjksIm5iZiI6MTcyMDM0Nzk2OSwicGF0aCI6Ii8yODgwMjU0NS8zNDYzMjcyNDMtODc3NjNmODktZWVjYy00ZmFiLWJjOTMtMWM4NTM3ZmNkNmZlLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MDclMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzA3VDEwMjYwOVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWQ5MTYwOGM2YzBmM2UwYTc5ZmZlM2QzNWUwMDQxMjdiYTcxZDY2ZmQ4MTRjYjlhMmE5NTdhOWRiOGZhZjRjZTMmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.Q9KtdILf2WdSvdzqfWSLvHxhmuspf1TXelpF6TRukfE)

그리고 실행시 __Executor__ 인터페이스가 어떤 구현체를 가지고 있는지도 확인해보았는데 __ThreadPoolTaskExecutor__ 를 가지고 있는것을 확인했습니다.

분명 __`@EnableAsync`__ 에서는 설정하지 않으면 기본적으로 __`SimpleAsyncTaskExecutor`__ 를 사용한다고 했는데 뭔가 이상했습니다.  

문서가 잘못되었을리는 없고... 뭔가 놓친게있나... 그래서 처음부터 다시 어떻게 설정이 되는지 살펴보았습니다.

우선 __`@SpringBootApplication`__ 의 내부 어노테이션을 보면 __`@EnableAutoConfiguration`__ 이 존재합니다.

해당 어노테이션은 어플리케이션 컨텍스트를 자동으로 설정하여 빈을 셋팅해주는데요.

이후 쭉 따라가보면 __`AutoConfigurationImportSelector`__ 에서 자동으로 설정할 Configuration 목록을 가져오는것을 확인할 수 있습니다.  
(자세한 추적과정은 다른글에서 소개할 수 있도록 하겠습니다.)

![spring-async-image8](https://private-user-images.githubusercontent.com/28802545/346329104-83315d7a-c7f5-41bb-a559-729de49794cb.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjAzNDk4ODIsIm5iZiI6MTcyMDM0OTU4MiwicGF0aCI6Ii8yODgwMjU0NS8zNDYzMjkxMDQtODMzMTVkN2EtYzdmNS00MWJiLWE1NTktNzI5ZGU0OTc5NGNiLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MDclMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzA3VDEwNTMwMlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTZiNzE4MGVkMDUwYTc5NWVlNDUzOTljNjUxN2YyNzY4ZWUzOThlMmE0MDJiOWY1ODc5Y2MxYmZiYWU4NDA2ZjImWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.krmvw0UARPpdGtFo13AEo7ErFOjaT4B4Q_LAGLgOsDI)

__`TaskExecutionAutoConfiguration`__ 클래스를 확인할 수 있습니다. 한번 살펴보겠습니다.

__TaskExecutionAutoConfiguration.java__
```java
@ConditionalOnClass(ThreadPoolTaskExecutor.class)
@AutoConfiguration
@EnableConfigurationProperties(TaskExecutionProperties.class)
@Import({ TaskExecutorConfigurations.ThreadPoolTaskExecutorBuilderConfiguration.class,
		TaskExecutorConfigurations.TaskExecutorBuilderConfiguration.class,
		TaskExecutorConfigurations.SimpleAsyncTaskExecutorBuilderConfiguration.class,
		TaskExecutorConfigurations.TaskExecutorConfiguration.class })
public class TaskExecutionAutoConfiguration {

	/**
	 * Bean name of the application {@link TaskExecutor}.
	 */
	public static final String APPLICATION_TASK_EXECUTOR_BEAN_NAME = "applicationTaskExecutor";

}
```

여기서 TaskExecutor 를 설정하는것을 알 수 있습니다.  
TaskExecutorBuilderConfiguration, SimpleAsyncTaskExecutorBuilderConfiguration 등을 주입받고 있네요.

__`TaskExecutorConfigurations`__ 를 한번 보겠습니다.

![spring-async-image9](
https://private-user-images.githubusercontent.com/28802545/346329665-90840ee7-fd4a-4fdd-9adf-464fcd00ac41.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjAzNTA0MTksIm5iZiI6MTcyMDM1MDExOSwicGF0aCI6Ii8yODgwMjU0NS8zNDYzMjk2NjUtOTA4NDBlZTctZmQ0YS00ZmRkLTlhZGYtNDY0ZmNkMDBhYzQxLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MDclMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzA3VDExMDE1OVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTEwNWEwMTc1MzY5MWRkMDJmMDE1MjgzZmVjOWQxZTk2MDVlYjQ2MWNjY2ZjNjZiNmRmMjMyNmMzYWU4NTU2ZWEmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.RxvIyn8B-tA5jee6_MpPo3fsVAtUrEPwW4sXiF0DF8M)

스레드의 타입에 따라 기본 __`TaskExecutor`__ 를 설정하는것을 볼 수 있습니다.
- 가상 스레드인 경우 __`SimpleAsyncTaskExecutor`__
- 플랫폼 스레드인 경우 __`ThreadPoolTaskExecutor`__

왜 기본 설정이 __`ThreadPoolTaskExecutor`__ 로 동작하는지에 대한 의문은 해소되었네요. 

결론은 Spring 환경에서는 기본으로 __`SimpleAsyncTaskExecutor`__ 를 사용하는것은 맞습니다.  
다만 SpringBoot 는 Spring 과 다르게 __AutoConfiguration__ 과정에서 해당 설정이 __`ThreadPoolTaskExecutor`__ 으로 설정되는것이었습니다.

추가로 공식문서에는 왜 저 내용이 없을까 의아해하며 열심히 찾아보던도중 역시 없을리가 없었습니다... 단순히 제가 뒤늦게 찾게된거였네요..

> If virtual threads are enabled (using Java 21+ and spring.threads.virtual.enabled set to true) this will be a SimpleAsyncTaskScheduler that uses virtual threads. This SimpleAsyncTaskScheduler will ignore any pooling related properties.
<br />
If virtual threads are not enabled, it will be a ThreadPoolTaskScheduler with sensible defaults. The ThreadPoolTaskScheduler uses one thread by default and its settings can be fine-tuned using the spring.task.scheduling namespace, as shown in the following example:

https://docs.spring.io/spring-boot/reference/features/task-execution-and-scheduling.html


<br />

#### __reference__

- https://spring.io/guides/gs/async-method
- https://www.baeldung.com/spring-async
- https://jeong-pro.tistory.com/253
- https://kapentaz.github.io/spring/Spring-ThreadPoolTaskExecutor-%EC%84%A4%EC%A0%95
