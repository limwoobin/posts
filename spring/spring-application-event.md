#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/spring-config-example)

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


의존성 분리를 하기 위해 __`@TransactionalEventListener`__ 를 알아보겠습니다.

<br />

## EventListener vs TransactionalEventListener 비교

__`EventListener`__ 는 위에서 같이 살펴본것처럼 이벤트를 수신하도록 하는 리스너입니다.  

__`TransactionalEventListener`__ 는 __`TransactionPhase`__ 에 따라 호출되는 이벤트 리스너입니다.  
즉, 트랜잭션 동작에 따라 실행되는 이벤트 리스너라 볼 수 있습니다.

그렇기에 __`TransactionalEventListener`__ 는 트랜잭션이 없다면 동작하지 않습니다.

__`TransactionPhase`__ 의 종류는 다음과 같습니다.

### TransactionalEventListener 의 phase 종류

- __BEFORE_COMMIT__: 트랜잭션이 커밋되기 전에 이벤트 처리
- __AFTER_COMMIT(Default)__: 트랜잭션 커밋된 직후 이벤트 처리
- __AFTER_ROLLBACK__: 트랜잭션 롤백 직후 이벤트 처리
- __AFTER_COMPLETION__: 트랜잭션이 완료된 뒤 이벤트 처리,  
트랜잭션 완료란 트랜잭션이 커밋되거나 롤백될때를 이야기함

<br />

__UserEventListener.java__
```java
@Component
public class UserEventListener {
  private final MailSenderService mailSenderService;

  public UserEventListener(MailSenderService mailSenderService) {
    this.mailSenderService = mailSenderService;
  }

  @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
  public void listen(MailSenderEvent event) {
    mailSenderService.send(event.getEmail());
  }
}
```

__TransactionalEventListener__ 의 __AFTER_COMMIT__ 을 이용하면 커밋이 완료된 이후 이벤트에서 예외가 발생하더라도 예외가 전파되지 않습니다.

트랜잭션을 처리하는 __`TransactionSynchronizationUtils`__ 클래스의 invokeAfterCompletion 메소드를 확인해보겠습니다.

__TransactionSynchronizationUtils.java__
```java
public abstract class TransactionSynchronizationUtils {
  // (중략 ...)

  public static void invokeAfterCompletion(@Nullable List<TransactionSynchronization> synchronizations,
			int completionStatus) {

		if (synchronizations != null) {
			for (TransactionSynchronization synchronization : synchronizations) {
				try {
					synchronization.afterCompletion(completionStatus);
				}
				catch (Throwable ex) {
					logger.error("TransactionSynchronization.afterCompletion threw exception", ex);
				}
			}
		}
	}
}
```

해당 클래스에서는 예외를 캐치한 후 디버깅으로 에러로그만 남기고 별도의 처리를 하지는 않습니다. 그렇기에 예외전파가 되지 않았습니다.

![spring-event-image1](https://private-user-images.githubusercontent.com/28802545/342329800-5ebca047-8289-4bc4-b192-4f778bdbfdfb.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTkyMzEwODUsIm5iZiI6MTcxOTIzMDc4NSwicGF0aCI6Ii8yODgwMjU0NS8zNDIzMjk4MDAtNWViY2EwNDctODI4OS00YmM0LWIxOTItNGY3NzhiZGJmZGZiLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA2MjQlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNjI0VDEyMDYyNVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTkzNjEwNzZiMjAyZTFiZjY0MDgzZmVlZjU2MDExZjU0ZjZjZWI4MWJjZTU5Y2Q2YTExOWI0OTQ0NGQ4NGVmZDkmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.wwlx7jXg1c4-yWle7Dp7VwpV3T0hi8MHzNh3B95jw6o)

javaDoc 을 확인해보면 __AFTER_COMMIT__ 은 __TransactionSynchronization__ 의 __afterCommit()__ 이 아닌 __afterCompletion()__ 에서 처리된다는것을 확인할 수 있습니다.

<br />

### __TransactionalEventListener__ 는 어떤 경우에 사용할까?

__TransactionalEventListener__ 는 앞에서 보았듯이 트랜잭션의 동작에 따라 이벤트를 처리한다는 특징이 있습니다.  
트랜잭션 처리 이후 이벤트가 실행되어야 하는 경우에 활용할 수 있는데요

다음 회원가입에 대한 간단한 예시를 보겠습니다.

```java
public void save(UserDto userDto) {
  userRepository.save(user); // (1)
  eventPublisher.publishEvent(new MailSenderEvent(user.getEmail()));  // (2)
  mailHistoryRepository.save(mail);  // (3)
}
```

1. 회원가입 한 유저 저장
2. 메일 이벤트 발행
3. 메일 이력 저장

코드는 다음과 같이 진행됩니다. 이벤트 리스너는 기본 __`@EventListener`__ 를 사용한다고 가정해보겠습니다.  
만약 이때 이벤트까지 발행한 후 (3) 번 메일 이력 저장에서 예외가 발생한다면 어떻게 될까요 ??

사용자에게 회원가입 완료 메일은 (3) 번 예외의 전파로 인해 유저 저장과 메일 이력 저장 모두 실패가 됩니다.  
사용자 입장에서는 회원가입 완료메일은 받았지만 실제 회원가입은 이뤄지지 않았으니 문제가 있다고 볼 수 있습니다.

이런 경우에 __TransactionalEventListener__ 을 사용하면 데이터 일관성을 지킬 수 있고 트랜잭션도 보다 짧기 때문에 롱 트랜잭션을 방지할 수 있다는 이점이 있습니다.  
그리고 롤백의 경우 특정 이벤트를 발행한다고 가정했을때에도 코드를 간결하게 처리할 수 있습니다.

### __TransactionalEventListener__ 주의사항

__`TransactionPhase.AFTER_COMMIT`__ 이후 이벤트에서 트랜잭션을 처리하지 못하는 이슈가 존재합니다.

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void listen(MailSenderEvent event) {
  xxxRepository.save(event); // (1) 이벤트 저장
}
```

다음과 같이 이벤트를 저장한다고 가정하겠습니다.  
이런 경우에는 트랜잭션이 커밋되지 않습니다. 왜 그런걸까요 ??

바로 이전에 이미 트랜잭션이 커밋되었기 떄문에 추가적인 트랜잭션 작업을 하려면 __`PROPAGATION_REQUIRES_NEW`__ 작업을 사용해야 한다고 합니다.

자세한 내용은 __TransactionSynchronization.java__ 의 `afterCommit / afterCompletion` 을 보면 확인할 수 있습니다.

> NOTE: The transaction will have been committed already, but the transactional resources might still be active and accessible. As a consequence, any data access code triggered at this point will still "participate" in the original transaction, allowing to perform some cleanup (with no commit following anymore!), unless it explicitly declares that it needs to run in a separate transaction. Hence: Use PROPAGATION_REQUIRES_NEW for any transactional operation that is called from here.

<br />

## 비동기 이벤트로 처리하기 위해서는 ?


### 비동기 스레드 풀 선언 방식(SimpleAsyncTaskExecutor vs ThreadExecutor)

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