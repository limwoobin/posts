# __[Spring] @Async 비동기 처리 및 스레드 풀 설정__

![spring-async-image1]()

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

### RejectedExecutionException, RejectedExecutionHandler

### taskExecutor.setWaitForTasksToCompleteOnShutdown(WAIT_TASK_COMPLETE);


### Future, ListenableFuture, CompletableFuture

### return new AsyncResult();

## @Asnyc 주의사항 (proxy)

#### __reference__

## 예외처리

- https://spring.io/guides/gs/async-method
- https://www.baeldung.com/spring-async

TaskExecutorConfigurations 174 line prefix