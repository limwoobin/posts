#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/lock-example)

이번엔 **낙관적 락(Optimistic Lock)** 을 이용해 동시성 처리를 하는 방법에 대해 알아보려 합니다.

<br>

그전에 **낙관적 락(Optimistic Lock)** 과 **비관적 락(Pessimistic Lock)** 의 간략한 차이점에 대해 먼저 설명드리겠습니다

### **낙관적 락(Optimistic Lock)**

충돌이 발생하지 않을 것이라 가정하고 Lock을 거는 방식

- 트랜잭션을 commit 하는 시점에 충돌을 알 수 있음
- DB Level 에서 동시성을 처리하는것이 아닌 Application Level 에서 처리

### **비관적 락(Pessimistic Lock)**

충돌이 발생할것이라 가정하고 우선 DB에 Lock을 거는 방식 (`select for update`)

- 데이터를 수정하는 즉시 충돌을 알 수 있음
- DB Level 동시성을 처리

<br />
