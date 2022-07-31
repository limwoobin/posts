[**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/java-grouping-example)

<br>

# **불변 객체(Immutable Object)란?**

객체 생성 이후 내부 상태가 변하지 않는, 변경할 수 없는 객체를 이야기합니다.  
<u>불변객체</u>는 내부 상태를 변경하는 메소드를 제공하지 않거나 방어적 복사를 통해 데이터를 제공합니다. 대표적으로 Java의 `String, Integer, Long, Double` 등등 있습니다.

<br>

## **불변 객체(Immutable Object) 의 장점**

#### **Thread-Safe하여 병렬 프로그래밍에 유용하고 동기화를 고려하지 않아도 된다.**

멀티 스레드 환경에서 발생하는 주된 문제는 공유자원에 대해 서로 변경하다보니 값이 덮어씌워지는 문제가 있습니다.  
하지만 불변객체는 항상 동일한 값을 보장하므로 동기화를 신경쓸 필요가 없다는 장점이 있습니다.

#### **내부 상태의 변경이 없기 때문에 Cache, Map, Set 등의 요소로 활용하기에 적합하다.**

Cache, Map, Set 의 요소가 변경되었다면 이를 갱신하는 작업이 필요하지만, 불변객체의 경우 데이터를 저장 후 이후 부가작업을 고려하지 않아도 되어 용이합니다.

#### **외부에서 객체에 대해 변경할 수 없기 때문에 안정성이 있다.**

가변객체에 객체의 값을 set할 수 있는 메소드가 열려있다면 이 객체의 값이 무엇인지 파악하기 어렵고 어떤 값이 들어가 있을지 파악하기 쉽지 않습니다.  
하지만 불변객체는 값의 수정이 불가능해 객체의 상태가 변경되지 않아 객체의 신뢰성이 높습니다.

#### **가비지 컬렉션의 성능을 높일 수 있다.**

불변객체는 기본적으로 값의 변경이 불가합니다. 상태의 변경이 필요한 경우 새로운 객체를 생성해서 사용해야 합니다.  
새로운 객체가 많이 생성될 경우 성능상 좋지 않을거라 생각하는 의견이 많이 있습니다. 저 역시도 이와 비슷한 의문을 가지고 있습니다.

하지만 Oracle에서의 의견은 아래와 같습니다.

> Programmers are often reluctant to employ immutable objects, because they worry about the cost of creating a new object as opposed to updating an object in place. The impact of object creation is often overestimated, and can be offset by some of the efficiencies associated with immutable objects. These include decreased overhead due to garbage collection, and the elimination of code needed to protect mutable objects from corruption.

<u>**객체 생성에 대한 비용은 과대평가되고있으며, 이는 불변객체를 이용한 효율로 충분히 상쇄할 수 있다**</u> 라고 얘기합니다.

[https://docs.oracle.com/javase/tutorial/essential/concurrency/immutable.html](https://docs.oracle.com/javase/tutorial/essential/concurrency/immutable.html)

불변객체를 이용하면 불변객체 내부의 객체에 대해서는 GC 스캔 대상에 제외됩니다.  
즉, 불변객체를 이용하면 GC의 스캔빈도와 범위가 줄게되어 GC의 성능에 도움이 된다고 할 수 있습니다.

<br>

## **불변객체 생성 규칙**

불변객체를 생성할때는 다음과 같은 규칙이 있습니다.

1. setter method를 제공하지 않는다.  
   (read-only method만 제공, 다만 타입에 따라 <u>방어적 복사</u>를 통해 참조를 끊어 제공하기도 한다)
2. 모든 필드를 private, final로 선언한다.
3. 클래스를 final로 선언한다.  
   (하위 클래스에서의 overriding을 방지하기 위함)
4. 객체를 생성하기 위한 생성자 or 정적 팩토리를 추가한다.
5. 불변객체 사용시 방어적 복사를 고려하지 않아도 된다.

```java
public final class ImmutableObj {
    private final int num;

    public ImmutableObj(int num) {
        this.num = num;
    }

    public int getNum() {
        return num;
    }
}
```

불변객체를 작성하면 대략 다음과 같은 모양을 갖습니다.

<br>

## **불변객체 작성시 유의사항**

불변객체 작성시 유의사항을 정리해보았습니다.  
아래 경우에 대한 예제코드와 함께 살펴보겠습니다.

1. primitive type이 필드로 있는 경우
2. reference type이 필드로 있는 경우
3. collection이 필드로 있는 경우

<br>

## reference

[https://ko.wikipedia.org/wiki/%EB%B6%88%EB%B3%80%EA%B0%9D%EC%B2%B4](https://ko.wikipedia.org/wiki/%EB%B6%88%EB%B3%80%EA%B0%9D%EC%B2%B4)  
[https://howtodoinjava.com/java/basics/how-to-make-a-java-class-immutable/](https://howtodoinjava.com/java/basics/how-to-make-a-java-class-immutable/)  
[https://www.baeldung.com/java-immutable-object](https://www.baeldung.com/java-immutable-object)
