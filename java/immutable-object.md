[**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/java-grouping-example)

<br>

# **불변 객체(Immutable Object)란?**

객체 생성 이후 내부 상태가 변하지 않는, 변경할 수 없는 객체를 이야기합니다.  
<u>불변객체</u>는 내부 상태를 변경하는 메소드를 제공하지 않거나 방어적 복사를 통해 데이터를 제공합니다. 대표적으로 Java의 `String, Integer, Long, Double` 등등 있습니다.

<br>

## **불변 객체(Immutable Object) 의 장점**

1. Thread-Safe하여 병렬 프로그래밍에 유용하고 동기화를 고려하지 않아도 된다.
2. 내부 상태의 변경이 없기 때문에 Cache, Map, Set 등의 요소로 활용하기에 적합하다.
3. 외부에서 객체에 대해 변경할 수 없기 때문에 안정성이 있다.  
   (객체에 대한 신뢰도가 높아진다)
4. 불변객체

<br>

## **불변객체 생성 규칙**

1. setter method를 제공하지 않는다.
2. 모든 필드를 private, final로 선언한다.
3. 클래스를 final로 선언한다.  
   (하위 클래스에서의 overriding을 방지하기 위함)
4. 객체를 생성하기 위한 생성자 or 정적 팩토리를 추가한다.

## **불변객체 작성시 유의사항**

1. primitive type이 필드로 있는 경우
2. reference type이 필드로 있는 경우
3. collection이 필드로 있는 경우

<br>

## reference

[https://ko.wikipedia.org/wiki/%EB%B6%88%EB%B3%80%EA%B0%9D%EC%B2%B4](https://ko.wikipedia.org/wiki/%EB%B6%88%EB%B3%80%EA%B0%9D%EC%B2%B4)  
[https://howtodoinjava.com/java/basics/how-to-make-a-java-class-immutable/](https://howtodoinjava.com/java/basics/how-to-make-a-java-class-immutable/)
