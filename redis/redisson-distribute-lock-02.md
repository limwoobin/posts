[**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/lock-example)

<br>

# **Redisson 라이브러리를 이용한 Distribute Lock 동시성 처리 (2)**

이번엔 앞에서 만들어놓은 `@DistributeLock` 어노테이션을 이용해 동시성을 처리하는 예제코드, 테스트 코드를 작성해보겠습니다.

<br>

## **Case1 - 쿠폰 발급 서비스**

```shell
테스트 시나리오는 다음과 같습니다.
1. 쿠폰 100개가 준비되어있음
2. 사용자 100명이 동시에 쿠폰을 발급받기 위해 쿠폰 발급을 요청함
3. 정상적으로 남은 쿠폰 갯수가 0이 되어야 함
```

<br>

## **Reference**

https://www.inflearn.com/course/%EB%8F%99%EC%8B%9C%EC%84%B1%EC%9D%B4%EC%8A%88-%EC%9E%AC%EA%B3%A0%EC%8B%9C%EC%8A%A4%ED%85%9C
