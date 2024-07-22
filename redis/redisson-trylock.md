![redisson-trylock-image1]()

# __[Redis] Redisson TryLock 동작과정 살펴보기__

어플리케이션을 개발하다보면 Redis 를 이용해 DistributedLock 을 구현하다보면 Redisson 라이브러리를 많이 사용하는데요.  
Redisson 은 pub/sub 을 이용하고, lua script 를 이용한다와 같은 특징이 있습니다.  

이번에는 좀 더 자세히 락을 획득하는 tryLock 메소드에 대해 어떻게 동작하는지  
그리고 위 특징들이 어떻게 동작하는지 한번 살펴보려 합니다.

DistributedLock 이 어떤건지 잘 모르시겠다면 제가 작성한 아래 글을 참고해보시는것을 추천드립니다.

- [**풀필먼트 입고 서비스팀에서 분산락을 사용하는 방법 - Spring Redisson**](https://helloworld.kurly.com/blog/distributed-redisson-lock/)
- [**[Spring] Redisson 라이브러리를 이용한 Distribute Lock 동시성 처리**](https://devoong2.tistory.com/entry/Spring-Redisson-%EB%9D%BC%EC%9D%B4%EB%B8%8C%EB%9F%AC%EB%A6%AC%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-Distribute-Lock-%EB%8F%99%EC%8B%9C%EC%84%B1-%EC%B2%98%EB%A6%AC-1)

<br />

## __Redisson tryLock 사용방법__

우선 __`tryLock`__ 메소드를 살펴보기 전에 어떻게 사용하는지를 간략히 살펴보겠습니다.

__RLock.java__
```java
public interface RLock extends Lock, RLockAsync {
    boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException;
}
```

__`tryLock`__ 메소드는 Redisson 에서 제공하는 __`RLock`__ 인터페이스에 선언되어있습니다.  
그리고 __`RLock`__ 는 Lock, RLockAsync 를 상속받고 있습니다.  
기본 메커니즘은 Java 의 Lock 인터페이스를 확장하는 구조입니다.

- __waitTime__: Lock 획득을 기다리는 시간 (정의된 waitTime 동안 Lock 획득을 시도)
- __leaseTime__: Lock 의 대여 시간 (락 획득 후 leaseTime 만큼 지나면 Lock 을 해제)
- __unit__: 시간 단위)

```java
public void testMethod() {
    RLock lock = redissonClient.getLock(LOCK_NAME); // (1)

    try {
      boolean available = lock.tryLock(10, 5, TimeUnit.SECONDS); // (2)

      if (available) { // (3)
        // business logic ...
      }
    } catch (InterruptedException e) {
      throw new RuntimeException(e);
    } finally {
      lock.unlock(); // (4)
    }
}
```

__`tryLock`__ 을 사용할때는 일반적으로 다음과 같이 처리하고 있습니다.

- (1): Lock 의 이름으로 __RLock__ 객체를 가져옵니다.
- (2): __tryLock__ 메소드를 호출해 락의 획득여부를 가져옵니다.
- (3): 락 획득에 성공하면 이후 로직을 진행합니다.
- (4): 마지막으로 락을 해제합니다.

<br />

## __Redisson TryLock 내부 동작__

__Redisson 3.19.1__ 버전이며  __RLock__ 의 구현체인 __RedissonLock__ 을 기준으로 살펴보겠습니다.

```java
@Override
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    long time = unit.toMillis(waitTime);
    long current = System.currentTimeMillis();
    long threadId = Thread.currentThread().getId();
    Long ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
    // lock acquired
    if (ttl == null) {
        return true;
    }
    
    time -= System.currentTimeMillis() - current;
    if (time <= 0) {
        acquireFailed(waitTime, unit, threadId);
        return false;
    }
    
    current = System.currentTimeMillis();
    CompletableFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);
    try {
        subscribeFuture.get(time, TimeUnit.MILLISECONDS);
    } catch (TimeoutException e) {
        if (!subscribeFuture.completeExceptionally(new RedisTimeoutException(
                "Unable to acquire subscription lock after " + time + "ms. " +
                        "Try to increase 'subscriptionsPerConnection' and/or 'subscriptionConnectionPoolSize' parameters."))) {
            subscribeFuture.whenComplete((res, ex) -> {
                if (ex == null) {
                    unsubscribe(res, threadId);
                }
            });
        }
        acquireFailed(waitTime, unit, threadId);
        return false;
    } catch (ExecutionException e) {
        acquireFailed(waitTime, unit, threadId);
        return false;
    }

    try {
        time -= System.currentTimeMillis() - current;
        if (time <= 0) {
            acquireFailed(waitTime, unit, threadId);
            return false;
        }
    
        while (true) {
            long currentTime = System.currentTimeMillis();
            ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
            // lock acquired
            if (ttl == null) {
                return true;
            }

            time -= System.currentTimeMillis() - currentTime;
            if (time <= 0) {
                acquireFailed(waitTime, unit, threadId);
                return false;
            }

            // waiting for message
            currentTime = System.currentTimeMillis();
            if (ttl >= 0 && ttl < time) {
                commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
            } else {
                commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
            }

            time -= System.currentTimeMillis() - currentTime;
            if (time <= 0) {
                acquireFailed(waitTime, unit, threadId);
                return false;
            }
        }
    } finally {
        unsubscribe(commandExecutor.getNow(subscribeFuture), threadId);
    }
//        return get(tryLockAsync(waitTime, leaseTime, unit));
}
```

위 코드는 __`RedissonLock`__ 의 __`tryLock`__ 메소드의 전체코드입니다.  
다음 코드를 하나씩 살펴보며 따라가보겠습니다.

### __1. 락의 획득을 시도한다.__

```java
long time = unit.toMillis(waitTime);
long current = System.currentTimeMillis();
long threadId = Thread.currentThread().getId();
Long ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
// lock acquired
if (ttl == null) {
    return true;
}

time -= System.currentTimeMillis() - current;
if (time <= 0) {
    acquireFailed(waitTime, unit, threadId);
    return false;
}
// ...
```

락 획득을 위해 __`tryLock`__ 메서드를 진입하면 바로 시간, 스레드Id 에 대해 변수로 만들고  
__`tryAcquire(waitTime, leaseTime, unit, threadId)`__ 메소드
에 진입합니다.

__`tryAcquire`__ 메서드를 따라 들어가보면 __`tryAcquireAsync`__ 메서드가 나옵니다.

__tryAcquireAsync Method__
```java
private <T> RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
    RFuture<Long> ttlRemainingFuture;
    if (leaseTime > 0) {
        ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
    } else {
        ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime,
                TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
    }
    CompletionStage<Long> f = ttlRemainingFuture.thenApply(ttlRemaining -> {
        // lock acquired
        if (ttlRemaining == null) {
            if (leaseTime > 0) {
                internalLockLeaseTime = unit.toMillis(leaseTime);
            } else {
                scheduleExpirationRenewal(threadId);
            }
        }
        return ttlRemaining;
    });
    return new CompletableFutureWrapper<>(f);
```

위 코드는 __`tryAcquireAsync`__ 메서드 입니다.

__`tryAcquireAsync`__ 내부에서 __`tryLockInnerAsync`__ 메서드를 호출해 __ttlRemainingFuture__ 를 받아옵니다.  
__ttlRemainingFuture__ 는 Lock 의 TTL 을 반환하는데요. __`tryLockInnerAsync`__ 내부를 살펴보겠습니다.

__tryLockInnerAsync__
```java
<T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
    return evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,
            "if ((redis.call('exists', KEYS[1]) == 0) " + // (1)
                "or (redis.call('hexists', KEYS[1], ARGV[2]) == 1)) then " + // (2)
                "redis.call('hincrby', KEYS[1], ARGV[2], 1); " + // (3)
                "redis.call('pexpire', KEYS[1], ARGV[1]); " + // (4)
                "return nil; " + // (5)
            "end; " +
            "return redis.call('pttl', KEYS[1]);", // (6)
            Collections.singletonList(getRawName()), unit.toMillis(leaseTime), getLockName(threadId));
}
```

__`tryLockInnerAsync`__ 에서 __Lua Script__ 를 통해 레디스에 커맨드를 호출하는것을 볼 수 있습니다.  
다음 스크립트를 설명드리겠습니다.

(1): __`redis.call('exists', KEYS[1]) == 0)`__ 
- 키가 존재하지 않는지 확인. 즉, 해당 잠금이 현재 없는지 확인합니다.

(2): __`redis.call('hexists', KEYS[1], ARGV[2]) == 1)`__ 
- 키가 존재하고, threadId 가 해당 키의 필드로 존재하는지 확인합니다. 이는 잠금의 재진입을 허용합니다.

(3): __`redis.call('hincrby', KEYS[1], ARGV[2], 1)`__ 
- KEYS[1] 해시의 ARGV[2] 필드의 값을 1 증가시킵니다. 이는 잠금을 설정하거나 재진입 시 횟수를 증가시킵니다.

(4): __`redis.call('pexpire', KEYS[1], ARGV[1])`__ 
- KEYS[1] 키의 TTL을 ARGV[1] 로 설정합니다. 즉, 키에 대해 TTL 을 설정합니다.
- 이때 TTL 은 전달받은 leaseTime 으로 설정됩니다.

(5): __`return nil`__
- null 을 리턴합니다.

(6): __`return redis.call('pttl', KEYS[1])`__
- KEYS[1] 의 TTL 을 리턴합니다.

요약하면 (1), (2) 둘 중 하나가 참이라면 락을 획득할 수 있다는 뜻입니다.  
그렇기에 해시에 키와 TTL 을 설정하고 null 을 리턴합니다.  
만약 이미 락이 선점되었다면 락을 획득할 수 없으니 해당 키의 TTL 을 리턴합니다.  

즉, __`tryLockInnerAsync`__ 의 반환된 TTL 이 null 이라면 락 획득에 성공한것이고 TTL 이 존재한다면 락 획득에 실패한것입니다.

이때 Redis 에 생성된 Lock 은 어플리케이션에서 설정한 Key 이름으로 설정됩니다.  
그리고 Hash 자료구조로 설정되고 TTL 도 설정된것을 확인할 수 있습니다.

![redisson-trylock-image4](https://private-user-images.githubusercontent.com/28802545/350759869-75f8f60b-bf51-44ea-89a4-1054fd0ae887.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjE1NTYwMDcsIm5iZiI6MTcyMTU1NTcwNywicGF0aCI6Ii8yODgwMjU0NS8zNTA3NTk4NjktNzVmOGY2MGItYmY1MS00NGVhLTg5YTQtMTA1NGZkMGFlODg3LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MjElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzIxVDA5NTUwN1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTZkZTM2YWI3ZjYwMmUwMGRlZjNmMDcxN2FkNjYwMzNmMTE2MjY4NmQxNWE1NjA3YzBhODNmNDBlNjVkNzUyYTMmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.ovtivHUZb8wytwOt4BjyCCadjYqL_YHCLSbb_MNYwf0)
![redisson-trylock-image5](https://private-user-images.githubusercontent.com/28802545/350759889-11d4686e-1380-43a7-a91c-0e8c3f0057bb.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjE1NTYwMDcsIm5iZiI6MTcyMTU1NTcwNywicGF0aCI6Ii8yODgwMjU0NS8zNTA3NTk4ODktMTFkNDY4NmUtMTM4MC00M2E3LWE5MWMtMGU4YzNmMDA1N2JiLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MjElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzIxVDA5NTUwN1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTMyZTA2YjE0OWM1MDczNmY2N2UyNTVjMmQ4YWU0Njk3YjlkNjU0YTg2ZDI1MDQ3MjQ2Zjg1YjZhMzliNTU4MjEmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.50_JkiaU8OxFWzO0CM9f3Yn2QgwA1keifZ9O1EzfOU8)

추가로 해당 Lock 은 Lock 을 점유한 스레드에 대한 LockName 과 count 정보도 함께 가지고 있는것을 확인할 수 있습니다.

![redisson-trylock-image6](https://private-user-images.githubusercontent.com/28802545/350759874-f4f8e1a7-8852-4504-b37e-b4d26cfaefe1.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjE1NTYwMDcsIm5iZiI6MTcyMTU1NTcwNywicGF0aCI6Ii8yODgwMjU0NS8zNTA3NTk4NzQtZjRmOGUxYTctODg1Mi00NTA0LWIzN2UtYjRkMjZjZmFlZmUxLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MjElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzIxVDA5NTUwN1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWQzYjkzN2MwN2FiY2YzNDU5ZWQxNWM3YzQ5NzIwMTIyYjllYTRlZjE3ZDBiNzU1MjRlYTZmMjgzNDRlMmIwNWMmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.aH1Qxjb59TKGUm4uGgMvHTgvwj1-U3nrPmQotY80Hac)

이때, 스레드에 대한 __LockName__ 은 Id + threadId 의 조합으로 설정됩니다.  
Id 는 Redisson ConnectionManager 의 Id 입니다.

![redisson-trylock-image7](https://private-user-images.githubusercontent.com/28802545/350760201-5c8173d5-4368-4397-ab26-5befe9002e90.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjE1NTYzNjQsIm5iZiI6MTcyMTU1NjA2NCwicGF0aCI6Ii8yODgwMjU0NS8zNTA3NjAyMDEtNWM4MTczZDUtNDM2OC00Mzk3LWFiMjYtNWJlZmU5MDAyZTkwLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MjElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzIxVDEwMDEwNFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWVjYzMxMjI1MmJkZGJiMjM5MDhjYjVmMmE3MzNiZDM2MDFhZDRkZjAzNGY2Y2JiOWI3ODIxYjhhNzY2NWEyYTEmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.IhL4Do1a-UqkGNumz6bg4XhK80_5dv1QHWa7JAaOo2E)

<hr />
<br />

```java
// ...
RFuture<Long> ttlRemainingFuture;
if (leaseTime > 0) {
    ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
} else {
    ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime,
            TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
}
// ...
```

그럼 다시 __`tryAcquireAsync`__ 로 돌아와서 __tryLockInnerAsync__ 메서드를 호출하는 부분을 보겠습니다.

앞서 TTL 은 leaseTime 으로 설정된다고 했었는데요.  
전달받은 __leaseTime__ 이 존재하면(0보다 크면) 락 획득에 대해 __leaseTiem__ 을 전달하고 그렇지 않다면 __internalLockLeaseTime__ 을 전달합니다.

__internalLockLeaseTime__ 은 Redisson Config 의 lockWatchdogTimeout 이며 기본값은 30초입니다.

즉, Lock 의 TTL 이 30초로 설정된다는 뜻입니다.

![redisson-trylock-image2](https://private-user-images.githubusercontent.com/28802545/350744638-9344a4ca-72e4-4844-9a4d-3dac1a6b25da.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjE1NDc2OTMsIm5iZiI6MTcyMTU0NzM5MywicGF0aCI6Ii8yODgwMjU0NS8zNTA3NDQ2MzgtOTM0NGE0Y2EtNzJlNC00ODQ0LTlhNGQtM2RhYzFhNmIyNWRhLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MjElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzIxVDA3MzYzM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTQzOTA3NGFlZjNlZDUyMzgzMWIwM2QxNzM2YWMxMDkzZjQ1NmYyYzQ4ZjgzYmIxYjg5NTY2YzY4ZWEyNzhmZDAmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.KmDLVCvr1fswCaCgQCWRMGU8x_vPxfJCRGcYRgeqmPQ)

![redisson-trylock-image3](https://private-user-images.githubusercontent.com/28802545/350744641-a8d1fe39-0cd2-484f-b8da-153646abdfc8.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjE1NDc2OTMsIm5iZiI6MTcyMTU0NzM5MywicGF0aCI6Ii8yODgwMjU0NS8zNTA3NDQ2NDEtYThkMWZlMzktMGNkMi00ODRmLWI4ZGEtMTUzNjQ2YWJkZmM4LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MjElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzIxVDA3MzYzM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWNlZTBmZWRkODVkNTA2ZDIwNDIyOWQ0ZDU2YTVmOGYxMzRmMWIyOTViNTllNmQwMjljOTM2ZDU4NTBjOTZiMTkmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.OcqzJafoYIoLvGrcFhGmx5_BN4L3tXDbvoLpt4DVgwQ)

<br />

```java
// ...
CompletionStage<Long> f = ttlRemainingFuture.thenApply(ttlRemaining -> {
    // lock acquired
    if (ttlRemaining == null) {
        if (leaseTime > 0) {
            internalLockLeaseTime = unit.toMillis(leaseTime);
        } else {
            scheduleExpirationRenewal(threadId);
        }
    }
    return ttlRemaining;
});
// ...
```

그리고 __`tryLockInnerAsync`__ 에서 TTL 을 받아온 후 로직입니다.

TTL 이 null 이고 설정한 __leaseTime__ 이 0보다 크다면 __internalLockLeaseTime__ 을 __leaseTime__ 으로 설정하는것을 볼 수 있습니다.

만약 설정한 __leaseTime__ 이 0보다 작다면 __`scheduleExpirationRenewal(threadId);`__ 을 호출합니다.

__`scheduleExpirationRenewal(threadId);`__ 의 역할은 간략히 말씀드리면 Lock 을 획득할때 __leaseTime__ 이 없어 기본 __internalLockLeaseTime__ 으로 설정된 이후 Lock 해제시까지 TTL 을 자동으로 갱신시켜주는 역할입니다.

<br />


```java
// ...
long time = unit.toMillis(waitTime);
long current = System.currentTimeMillis();
long threadId = Thread.currentThread().getId();
Long ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
// lock acquired
if (ttl == null) {
    return true;
}

time -= System.currentTimeMillis() - current;
if (time <= 0) {
    acquireFailed(waitTime, unit, threadId);
    return false;
}
// ...
```

그리고 다시 __`tryLock`__ 메서드입니다.
여기서 __`tryAcquire`__ 메서드로부터 TTL 을 가져와 null 인지 확인하고 null 이라면 Lock 획득에 성공했으니 true 를 리턴합니다.

이후 time 을 재계산하는데요, 처음 설정한 waitTime 에서 TTL 획득하기까지 걸린 시간을 차감합니다.  
그리고 time 이 0보다 작다는것은 waitTime 이 만료되었다는 뜻이니 false 를 리턴합니다.

<hr />

## __2. Lock에 대해 Redis Pub/Sub 채널 구독__

```java
// ...
current = System.currentTimeMillis();
CompletableFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);
try {
    subscribeFuture.get(time, TimeUnit.MILLISECONDS);
} catch (TimeoutException e) {
    if (!subscribeFuture.completeExceptionally(new RedisTimeoutException(
            "Unable to acquire subscription lock after " + time + "ms. " +
                    "Try to increase 'subscriptionsPerConnection' and/or 'subscriptionConnectionPoolSize' parameters."))) {
        subscribeFuture.whenComplete((res, ex) -> {
            if (ex == null) {
                unsubscribe(res, threadId);
            }
        });
    }
    acquireFailed(waitTime, unit, threadId);
    return false;
} catch (ExecutionException e) {
    acquireFailed(waitTime, unit, threadId);
    return false;
}
// ...
```

응답받은 TTL 이 존재하는 경우에는 __threadId__ 로 채널을 구독하게 됩니다.  
그리고 __`CompletableFuture`__ 로 응답을 가져오고 이때 __`TimeoutException, ExecutionException`__ 을 예외로 선언해 해당 예외가 발생하면 락 획득에 대해 실패로 처리하게 됩니다.

예외가 발생하는 경우는 아래와 같습니다.

- __`TimeoutException`__: 채널을 구독하여 대기하는 동안 __waitTime__ 이 지난 경우
- __`ExecutionException`__: __`CompletableFuture`__ 에서 예외가 발생한 경우

<br />

이제 락을 구독하는 __subscribe__ 메서드를 한번 살펴보겠습니다.  
아래는 __subscribe__ 메서드의 전체 코드입니다.

__subscribe method__
```java
public CompletableFuture<E> subscribe(String entryName, String channelName) {
    AsyncSemaphore semaphore = service.getSemaphore(new ChannelName(channelName));
    CompletableFuture<E> newPromise = new CompletableFuture<>();

    semaphore.acquire().thenAccept(c -> {
        if (newPromise.isDone()) {
            semaphore.release();
            return;
        }

        E entry = entries.get(entryName);
        if (entry != null) {
            entry.acquire();
            semaphore.release();
            entry.getPromise().whenComplete((r, e) -> {
                if (e != null) {
                    newPromise.completeExceptionally(e);
                    return;
                }
                newPromise.complete(r);
            });
            return;
        }

        E value = createEntry(newPromise);
        value.acquire();

        E oldValue = entries.putIfAbsent(entryName, value);
        if (oldValue != null) {
            oldValue.acquire();
            semaphore.release();
            oldValue.getPromise().whenComplete((r, e) -> {
                if (e != null) {
                    newPromise.completeExceptionally(e);
                    return;
                }
                newPromise.complete(r);
            });
            return;
        }

        RedisPubSubListener<Object> listener = createListener(channelName, value);
        CompletableFuture<PubSubConnectionEntry> s = service.subscribeNoTimeout(LongCodec.INSTANCE, channelName, semaphore, listener);
        newPromise.whenComplete((r, e) -> {
            if (e != null) {
                s.completeExceptionally(e);
            }
        });
        s.whenComplete((r, e) -> {
            if (e != null) {
                entries.remove(entryName);
                value.getPromise().completeExceptionally(e);
                return;
            }
            value.getPromise().complete(value);
        });

    });

    return newPromise;
}
```

우선 특정 채널에 대한 __세마포어__ 와 __newPromise__ 라는 __CompletableFuture__ 를 선언합니다.

이후 세마포어에선 acquire 시 실행할 콜백 함수를 선언합니다.  
그리고 아래에선 채널에 대한 Redis 채널에 대한 __Pub/Sub Listener__ 를 생성하고 __subscribeNoTimeout__ 를 통해 해당 채널을 구독합니다.

이후 __newPromise__ 와 __CompletableFuture<PubSubConnectionEntry>__ 에 대해 콜백함수를 선언합니다.  
마지막으로 __CompletableFuture__ 에 __RedissonLockEntry__ 을 담아 리턴합니다.

이때 레디스 채널명은 아래와 같이 __channelName + Lock Key__ 조합으로 생성되는것을 확인할 수 있습니다.

__RedissonLock.java__
![redisson-trylock-image8](https://private-user-images.githubusercontent.com/28802545/350962694-8bc3bdd0-5c4e-4b21-a5c2-740456a8ee10.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjE2NTEzMDMsIm5iZiI6MTcyMTY1MTAwMywicGF0aCI6Ii8yODgwMjU0NS8zNTA5NjI2OTQtOGJjM2JkZDAtNWM0ZS00YjIxLWE1YzItNzQwNDU2YThlZTEwLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MjIlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzIyVDEyMjMyM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTlkNzI3Njg0ZTQ0Njc2Njg0ZjkyOTZmODYwMmQwOTdlNGNhZmQ0ZWQzMTUyZTZhZjVhZWMzMTY4ZmM5YmU3ODUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.keAI2VR50AWtrjNEMFNavbgC8e8LAZ6r0L-ggY0cye8)

__Redis CLI__
![redisson-trylock-image9](https://private-user-images.githubusercontent.com/28802545/350963341-92d82756-38a7-45be-bdb0-3329e644e209.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjE2NTE0NDIsIm5iZiI6MTcyMTY1MTE0MiwicGF0aCI6Ii8yODgwMjU0NS8zNTA5NjMzNDEtOTJkODI3NTYtMzhhNy00NWJlLWJkYjAtMzMyOWU2NDRlMjA5LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MjIlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzIyVDEyMjU0MlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTQyMGVjMWMyZjNmMmUyZmU0NDg3ODVhMTRjNDIxYTM3OGUxYTEwZWUxZmZmYTUyN2U4OGYzNjNkNzI3N2UzM2QmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.gEWeBLXGPigSyBtK9tGJvlzKorL_xLl2Tlgay9Cx1cg)

<br />

> __`subscribeFuture.get(time, TimeUnit.MILLISECONDS);`__

이후 CompletableFuture 의 get 을 통해 값을 가져옵니다.  
이때 __RedissonLockEntry__ 객체에 세마포어를 가져오게 됩니다.

아래 디버깅 이미지를 보면 __latch__ 에는 세마포어 객체가 존재하는것을 확인할 수 있고 __counter__ 는 Lock을 관리하는 객체인 __RedissonLockEntry__ 에 acquire() 한 횟수를 나타냅니다.

![redisson-trylock-image10](https://private-user-images.githubusercontent.com/28802545/350961176-be64f7b3-d7f7-49ed-ab50-ae9ef26ed8be.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjE2NTMxNjAsIm5iZiI6MTcyMTY1Mjg2MCwicGF0aCI6Ii8yODgwMjU0NS8zNTA5NjExNzYtYmU2NGY3YjMtZDdmNy00OWVkLWFiNTAtYWU5ZWYyNmVkOGJlLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA3MjIlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNzIyVDEyNTQyMFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTBhMTFhZWMyMzhmMDhlODY4Y2IzMTI2NzkyYzA3MWE0ZTMyNzgzMjU1ZjMxYzk5ZjMwNTY0MWYzNWQ2OGU5ZjkmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.vtrHzIyMRGyTPAyPK9xk6c3VGG4xBMyM-E_RDYEp69s)

<br />

## 3. 락 획득 재시도

```java
// ...
try {
    time -= System.currentTimeMillis() - current;
    if (time <= 0) {
        acquireFailed(waitTime, unit, threadId);
        return false;
    }

    while (true) {
        long currentTime = System.currentTimeMillis();
        ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
        // lock acquired
        if (ttl == null) {
            return true;
        }

        time -= System.currentTimeMillis() - currentTime;
        if (time <= 0) {
            acquireFailed(waitTime, unit, threadId);
            return false;
        }

        // waiting for message
        currentTime = System.currentTimeMillis();
        if (ttl >= 0 && ttl < time) {
            commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
        } else {
            commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
        }

        time -= System.currentTimeMillis() - currentTime;
        if (time <= 0) {
            acquireFailed(waitTime, unit, threadId);
            return false;
        }
    }
} finally {
    unsubscribe(commandExecutor.getNow(subscribeFuture), threadId);
}
// ...
```

위 코드는 Redis 채널 구독 이후 로직입니다.  
여기에서는 락 획득을 위해 재시도하는 로직이 진행됩니다.

하나씩 따라가보겠습니다.

```java
// ...
time -= System.currentTimeMillis() - current;
if (time <= 0) {
    acquireFailed(waitTime, unit, threadId);
    return false;
}
// ...
```

우선 waitTime 인 time 변수에 시간을 차감하여 현재 남은시간을 다시 확인합니다.  
__`time <= 0`__ 이라는건 waitTime 이 모두 지났다는 뜻이므로 실패를 리턴합니다.

```java
// ...
while (true) {
    long currentTime = System.currentTimeMillis();
    ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
    // lock acquired
    if (ttl == null) {
        return true;
    }

    time -= System.currentTimeMillis() - currentTime;
    if (time <= 0) {
        acquireFailed(waitTime, unit, threadId);
        return false;
    }

    // ...
    // waiting for message
    currentTime = System.currentTimeMillis();
    if (ttl >= 0 && ttl < time) {
        commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
    } else {
        commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
    }

    time -= System.currentTimeMillis() - currentTime;
    if (time <= 0) {
        acquireFailed(waitTime, unit, threadId);
        return false;
    }
}
```

그 다음에는 while loop 에 진입합니다.  
__Redisson__ 은 스핀락 대신 Pub/Sub 구조를 사용한다고 말씀드렸는데요. 내부적으로는 스핀락을 사용하는것일까요?

일단 따라가보겠습니다.

while loop 진입 후 __tryAcquire__ 메서드를 호출해 락 획득을 시도하네요.  
처음과 동일하게 TTL 존재여부에 따라 성공/실패를 리턴합니다.

이후 time 변수를 재계산하여 waitTime 이 경과했는지도 확인하네요.

```java
// ...
while (true) {
    // ...
    
    // waiting for message
    currentTime = System.currentTimeMillis();
    if (ttl >= 0 && ttl < time) {
        commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
    } else {
        commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
    }

    time -= System.currentTimeMillis() - currentTime;
    if (time <= 0) {
        acquireFailed(waitTime, unit, threadId);
        return false;
    }
}
```


> __commandExecutor.getNow(subscribeFuture).getLatch().tryAcquire(long timeout, TimeUnit unit);__

해당 코드는 세마포어의 tryAcquire 메서드를 호출하는데요.  
즉, 해당 코드에서는 timeout 시간만큼 해당 세마포어의 허가를 얻기위해 대기합니다.  

이때, TTL 이 존재하고 time(남은 대기시간) 이 TTL 보다 크면 timeout parameter 로 TTL 을 전달합니다.  
TTL 만큼 세마포어 획득을 기다리겠다는 뜻입니다.

그렇지 않다면 남은 대기시간만큼 세마포어 획득을 기다립니다.  
왜냐하면 TTL 이 남은 대기시간보다 큰데 TTL 만큼 기다린다면 대기시간이 의미가 없어져서 그렇습니다.

그리고 대기중인 timeout 시간보다 Lock 이 먼저 해제된다면 바로 세마포어의 허가를 받아옵니다.

이후 세마포어의 허가를 얻고 난 이후 다시 time 을 재계산하여 대기시간이 경과했는지 확인합니다.  
그리고 while loop 의 처음으로 돌아가 다시 락 획득을 시도해 TTL 결과를 얻어오고 반복합니다.

이때 만약 기존 락이 해제되지 않았거나 다른 스레드가 먼저 락을 선점했다면 다시 이 과정을 반복하게 됩니다.

이 __while loop__ 때문에 Redisson 도 스핀락으로 동작한다고 볼 수도 있지만
이 사이에 __pub/sub__ 과 __세마포어__ 개념이 존재하여 세마포어 허가를 얻은 스레드에 대해서만 접근을 허용해 접근 횟수를 획기적으로 줄였기에 다르게도(?) 볼 수 있지 않을까 생각했습니다.

<br />

## __tryLock 의문...__

실제 pubsub 채널은 1개만생긴다 왜그럻지...


#### __reference__

- https://incheol-jung.gitbook.io/docs/q-and-a/spring/redisson-trylock
- https://redisson.org/glossary/java-semaphore.html