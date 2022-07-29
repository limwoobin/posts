[**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/java-grouping-example)

<br>

# 불변 객체(Immutable Object)란?

객체 생성 이후 내부 상태가 변하지 않는, 변경할 수 없는 객체를 이야기합니다.  
<u>불변객체</u>는 내부 상태를 변경하는 메소드를 제공하지 않거나 방어적 복사를 통해 데이터를 제공합니다. 대표적으로 Java의 `String` 이 있습니다.

## 불변 객체(Immutable Object) 의 장점

1. Thread-Safe하여 병렬 프로그래밍에 유용하고 동기화를 고려하지 않아도 된다.
2. 실패 원자적인(Failure Atomic) 메소드를 만들 수 있다.
3. Cache, Map, Set 등의 요소로 활용하기에 적합하다
4. 부수 효과(Side Effect)를 피해 오류 가능성을 최소화할 수 있다.
5. 다른 사람이 작성한 함수를 예측 가능하며 안전하게 사용할 수 있다.

## 불변객체 작성시 유의사항

1. primitive type이 필드로 있는 경우
2. reference type이 필드로 있는 경우
3. collection이 필드로 있는 경우

<br>

## reference

[https://ko.wikipedia.org/wiki/%EB%B6%88%EB%B3%80%EA%B0%9D%EC%B2%B4](https://ko.wikipedia.org/wiki/%EB%B6%88%EB%B3%80%EA%B0%9D%EC%B2%B4)
