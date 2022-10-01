[**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/lock-example)

# **Redisson 라이브러리를 이용한 Distribute Lock 동시성 처리**

Redis를 통한 분산락을 이용해 동시성을 해결하는 방법에 대해 알아보고 예제코드를 함께 공유드리려 합니다.

<br>

## **분산 락(Distribute Lock) ?**

Lock: DB의 트랜잭션의 순차적 처리를 보장하기 위한 방법  
여러 서버에서 동기화된 처리를 하기 위해 Database, Redis와 같은 공통된 저장소를 이용한 방법  
**(공통된 저장소를 사용해 여러 서버에 대한 동기화된 처리가 가능함)**

<br>

## **Redisson 사용 이유**

Spring에서 제공하는 대표적인 redis 라이브러리로는 Lettuce가 있습니다.  
하지만 Lettuce는 다음과 같은 단점이 존재합니다.

```shell
1. Lock의 타임아웃이 지정되있지 않음
- 락 획득 후 모종의 이유로 어플리케이션이 종료된다거나 했을 경우엔 락은 해제되지 않음
그렇게 되면 타 프로세스들은 락이 해제되기만을 기다리는 무한정 대기상태로 빠지게됨, 이는 곧 시스템 장애로 이어짐

2. 스핀락으로 인해 Redis에 많은 부담을 가하게 됨
락을 획득하지 못한 경우 Redis에게 락을 획득하기 위해 계속 요청을 보내게 됨
```

Redisson은 다음과 같은 장점이 있습니다.

```shell
1. Lock에 타임아웃을 지정할 수 있음
- Redisson은 락 획득시도시 타임아웃을 명시하게 되어있습니다.
그래서 무한정 대기상태로 빠질 수 있는 위험이 없습니다.
2. pub/sub방식을 사용하므로 스핀락을 사용하지 않음
- 락이 해제되면 락을 subscribe하는 클라이언트들에게 락이 해제되었다는 신호를 보내게 됩니다.
그렇기에 락을 subscribe하는 클라이언트들은 더 이상 락을 획득해도 되냐고 redis로 요청을 보내지 않습니다.
```

<hr>

### **Redisson 분산락 사용방법**

Spring에서 Redisson을 사용하기 위해선 아래의 의존성이 필요합니다.  
그리고 Redisson에서 제공하는 인터페이스와 사용법을 간략하게 소개하겠습니다.

build.gradle

```java
dependencies {
	// redisson
	implementation 'org.redisson:redisson-spring-boot-starter:3.17.4'
}
```

Redisson에서는 Lock을 사용하기 위해 `RLock` 이라는 인터페이스를 제공합니다.

락을 획득하기 위해서는 `tryLock`이라는 메소드를 이용합니다.

```java
boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException;
```

- waitTime: 락을 획득하기 위한 대기 시간
- leaseTime: 락을 임대하는 시간
- unit: 시간 단위

<br>

```java
RLock rLock = redissonClient.getLock(key); // (1)

try {
		boolean available = rLock.tryLock(5, 3, TimeUnit.SECONDS); // (2)
		if (!available) {	// (3)
			return false;
		}

		// 락 획득 후 수행 로직...
} catch (InterruptedException e) {
		e.printStackTrace();
		throw new InterruptedException();
} finally {
		rLock.unlock(); // (4)
}
```

```Shell
(1) - key 이름에 해당하는 RLock 인스턴스를 가져온다
(2) - Lock 획득을 시도한다 (성공: true / 실패: false)
(3) - 획득 실패시 Lock을 subscribe 하며 해제되길 기다린다
(4) - finally에서 Lock을 해제한다
```

<hr>

## **Redisson 분산락 annotation 기반으로 사용하기**

<hr>

Stock 재고차감
시나리오1

1. 재고100개 생성
2. 동시성 thread 100개 생성
3. thread당 재고 1개씩 차감 실행
4. 재고 0개 확인

db connection pool 주의사항
트랜잭션 내부에서 새로운 트랜잭션을 여는 경우 cp 가 부족하면 cp dead lock을 유발하는 포인트가 될 수 있음
이 부분을 어떻게 풀어나가야하는지??

- facade pattern
- 트랜잭션 단위를 작업별로 짧게 가져

<hr>

#### Reference

https://hyperconnect.github.io/2019/11/15/redis-distributed-lock-1.html
