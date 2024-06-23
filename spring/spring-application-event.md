# [Spring] ApplicationEvent 를 이용한 비동기 이벤트 처리하기

__`Spring Event`__ 는 스프링에서 제공하는 이벤트 기반 프로그래밍을 지원하기 위한 기능입니다.  
이벤트는 __이벤트를 발행하는 주체(publisher)__ 와 __이벤트를 처리하는 주체(listener)__ 로 나누어 집니다.

이벤트를 이용하면 코드에 대한 의존성을 분리할 수 있고 특정 작업 이후에 
이벤트를 통해 추가적인 작업을 의존성 없이 진행할 수 있다는 장점이 있습니다.

예를 들면, 회원가입 이후 가입 완료에 대한 메일을 발송하는 경우 메일서버에 장애가 발생해도 회원가입은 정상적으로 이루어져야 합니다.  
이때, 회원가입과 메일발송에 대한 의존성을 분리하는데 이벤트를 사용할 수 있습니다.

<br />

__`ApplicationEventPublisher`__  인터페이스를 이용해 이벤트를 발행할 수 있습니다.

__ApplicationEventPublisher.java__
```java
@FunctionalInterface
public interface ApplicationEventPublisher {
    default void publishEvent(ApplicationEvent event) {
		publishEvent((Object) event);
	}

	void publishEvent(Object event);
}
```

<br />

__`ApplicationListener`__ 혹은 __`@EventListener`__ 를 통해 이벤트를 수신받아 처리할 수 있습니다.  

```java
@Component
public class CustomSpringEventListener implements ApplicationListener<CustomSpringEvent> {
    @Override
    public void onApplicationEvent(CustomSpringEvent event) {
        System.out.println("Received spring custom event - " + event.getMessage());
    }
}

// or

@Component
public class CustomSpringEventListener {

    @EventListener
    public void onApplicationEvent(CustomSpringEvent event) {
        System.out.println("Received spring custom event - " + event.getMessage());
    }
}
```

<br />

## 이벤트 발행 및 처리 방법

간단한 회원가입 후 메일을 발송하는 이벤트를 발행하고 해당 이벤트를 처리하는 과정에 대한 예제 코드를 작성해보겠습니다.

### 이벤트 발행

먼저 이벤트 객체를 생성합니다.  
+ 스프링 4.2 이전버전에서는 `ApplicationEvent` 를 상속받아야했지만 이후부터는 바로 사용할 수 있습니다.

__MailSenderEvent.java__
```java
@Getter
public class MailSenderEvent {

  private String email;

  public MailSenderEvent(String email) {
    this.email = email;
  }
}
```

<br />

그리고 서비스에서 회원가입 후 이벤트를 발송합니다.  
다음 예제코드를 보면 이벤트 발행을 위해 __`ApplicationEventPublisher`__ 를 주입받은것을 볼 수 있습니다.

__UserService.java__
```java
@Service
public class UserService {
  private final UserRepository userRepository;
  private final ApplicationEventPublisher eventPublisher;

  public UserService(UserRepository userRepository,
                     ApplicationEventPublisher eventPublisher) {
    this.userRepository = userRepository;
    this.eventPublisher = eventPublisher;
  }

  @Transactional
  public void save(UserDto userDto) {
    User user = User.of(userDto.getName(), userDto.getEmail());
    userRepository.save(user);

    eventPublisher.publishEvent(new MailSenderEvent(user.getEmail()));
  }
}
```

### 이벤트 수신(처리)

메일 발송을 위한 인터페이스를 생성합니다.  
실제 메일을 발송하지 않고 행위 검증을 위해 인터페이스로 만들었습니다.

__MailSenderService.java__
```java
@Slf4j
@Component
public class MailSenderService {
  public void send(String email) {
    log.info("{} Mail Send", email);
  }
}
```

<br />

그리고 이벤트 처리를 위한 이벤트리스너를 생성합니다.  
+ 스프링 4.2 이전버전에서는 __`ApplicationListener`__ 를 구현해야 했습니다.  
이후 버전에서는 __`@EventListener`__ 를 통해 간단히 처리할 수 있습니다.

__UserEventListener.java__
```java
@Component
public class UserEventListener {
  private final MailSenderService mailSenderService;

  public UserEventListener(MailSenderService mailSenderService) {
    this.mailSenderService = mailSenderService;
  }

  @EventListener
  public void listen(MailSenderEvent event) {
    mailSenderService.send(event.getEmail());
  }
}
```

다음과 같이 이벤트를 발행하고 처리할 수 있습니다.  

하지만 위 코드에는 두 가지 문제가 있습니다.

1. 이벤트는 기본적으로 동기로 동작하기에 현재 상태로는 비동기 처리가 되지않습니다.  
    - 만약 메일 발송시 어떠한 이유로 시간이 오래걸린다면 회원가입에도 시간이 오래걸린다는 이야기가 됩니다.  
    - 이는 곧 사용자 경험 및 성능 이슈와도 연결될 수 있습니다.
2. 메일 발송시 예외가 발생하면 __`UserService`__ 까지 예외가 전파되어 회원가입도 롤백됩니다. 이러한 상황은 의존성이 분리되었다고 보기 어렵습니다.

<br />


우선 의존성 분리를 하기 위해 __`@TransactionalEventListener`__ 를 알아보겠습니다.

<br />

## EventListener vs TransactionalEventListener 비교

### TransactionalEventListener 의 phase 종류
- BEFORE_COMMIT
- AFTER_COMMIT
- AFTER_ROLLBACK
- AFTER_COMPLETION

AFTER_COMMIT 와 AFTER_COMPLETION 의 차이

## 비동기 이벤트로 처리하기 위해서는 ?

## 비동기 스레드 풀 선언 방식(SimpleAsyncTaskExecutor vs ThreadExecutor)

### AsyncConfig 옵션 설명
- coresize ...

### 주의사항
- private method는 사용 불가
- self-invocation(자가 호출) 불가, 즉 inner method는 사용 불가
- QueueCapacity 초과 요청에 대한 비동기 method 호출시 방어 코드 작성

## AsyncUncaughtExceptionHandler 예외처리

<hr />

#### __reference__

- https://www.baeldung.com/spring-events