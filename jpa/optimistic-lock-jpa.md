#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/lock-example)

<br />

이번엔 **낙관적 락(Optimistic Lock)** 을 이용해 동시성 처리를 하는 방법에 대해 알아보려 합니다.

그전에 **낙관적 락(Optimistic Lock)** 과 **비관적 락(Pessimistic Lock)** 의 간략한 차이점에 대해 먼저 설명드리겠습니다

<br />

### **낙관적 락(Optimistic Lock)**

충돌이 발생하지 않을 것이라 가정하고 Lock을 거는 방식

- 트랜잭션을 commit 하는 시점에 충돌을 알 수 있음
- DB Level 에서 동시성을 처리하는것이 아닌 Application Level 에서 처리

<br />

### **비관적 락(Pessimistic Lock)**

충돌이 발생할것이라 가정하고 우선 DB에 Lock을 거는 방식 (`select for update`)

- 데이터를 수정하는 즉시 충돌을 알 수 있음
- DB Level 동시성을 처리

<br />

## **낙관적 락(Optimistic Lock)**

JPA에서의 낙관적 락을 처리하는 방법은 **@Version Annotation** 을 이용해 처리할 수 있습니다.  
이 **@Version**은 버전 관리용 필드를 추가해 트랜잭션 내에서 처음 조회되었을때의 버전과 이후 수정 후 커밋될때의 버전을 비교합니다.

**@Version**을 적용할 수 있는 타입은 다음과 같습니다.

```
- Long
- Integer
- Short
- Timestamp
```

이 field 의 **값 혹은 시간**이 처음 조회될 때의 버전과 commit될때의 버전이 서로 다르다면 이는 충돌이 발생한 것으로 판단하고 예외를 발생시킵니다.

재고를 차감하는 예를 들어보겠습니다.  
치킨A라는 재고는 현재 단 한개가 남아 있습니다.

```shell
[transaction-1] : 치킨A의 재고를 확인 / 치킨A 재고: 1개, version: 1
[transaction-2] : 치킨A의 재고를 확인 / 치킨A 재고: 1개, version: 1

-- 이때 두 트랜잭션 중 transaction-1 가 먼저 완료되었다고 가정해보겠습니다.

[transaction-1] : 치킨A를 구매 / 치킨A 재고: 0개, version: 2 로 업데이트하고 커밋
[transaction-2] : 치킨A를 구매 / 치킨A 재고: 0개, version: 2 로 업데이트하고 커밋하려는데 version이 처음 조회했던 1이 아니라 [transaction-1]에서 2로 변경되어 현재 조회한 버전과 다르므로 업데이트 실패
```

그렇다면 이 과정을 코드예제로 한번 보여드리겠습니다.
