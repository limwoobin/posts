#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/transaction-propagation)

<br />

# __트랜잭션 REQUIRES_NEW 옵션에서 예외와 롤백__

## Overview

이번에는 스프링 환경에서 __`@Transactional`__ 의 __`Propagation`__ 옵션인 __`REQUIRES_NEW`__ 와 해당 옵션을 사용할때의 예외/롤백에 대해 알아보겠습니다.

우선 트랜잭션 전파(`Transaction Propagation`)에 대해 먼저 알아보겠습니다.  
트랜잭션 전파는 한 트랜잭션이 실행중에 다른 트랜잭션을 실행할 경우 어떻게 동작할지를 __결정__ 하는것입니다.

트랜잭션 전파의 종류는 다음과 같습니다.

- __REQUIRED (Default)__
- __REQUIRES_NEW__
- __SUPPORTS__
- __NOT_SUPPORTED__
- __MANDATORY__
- __NEVER__
- __NESTED__

<br />

전파옵션의 기본값은 __`REQUIRED`__ 이며 별도의 트랜잭션이 실행되어도 부모 트랜잭션에 종속되어 실행됩니다.

저희가 알아볼 __`REQUIRES_NEW`__ 옵션은 부모 트랜잭션과는 별도의 트랜잭션을 생성하여 동작합니다.  

그렇다면 __`REQUIRES_NEW`__ 를 사용한 자식 트랜잭션에서 예외가 발생한다면 어떻게 될까요?  
부모와 자식 트랜잭션 모두 롤백이 될까요, 아니면 별도의 트랜잭션이니 자식 트랜잭션만 롤백이 될까요?

예외와 롤백이 발생할 수 있는 시나리오를 정의하고 예제 코드를 통해 어떻게 동작할지 한 번 알아보겠습니다.  
시나리오는 다음과 같이 정의하겠습니다.

```
1. 해피 케이스
- 부모 트랜잭션 실행
- 자식 트랜잭션 실행(REQUIRES_NEW)

2. 예외 케이스 1
- 부모 트랜잭션 실행
- 자식 트랜잭션 실행(REQUIRES_NEW)
- 자식 트랜잭션 예외 발생

3. 예외 케이스 2
- 부모 트랜잭션 실행
- 자식 트랜잭션 실행(REQUIRES_NEW)
- 부모 트랜잭션 예외 발생

4. 예외 케이스 3
- 부모 트랜잭션 실행
- 자식 트랜잭션 실행(REQUIRES_NEW)
- 자식 트랜잭션 예외 발생 후 캐치
- 자식 트랜잭션 예외 캐치 후 현재 트랜잭션만 롤백
```

## 에제 코드



## 시나리오 검증

## 후기

#### reference
- https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Propagation.html
- https://oingdaddy.tistory.com/28