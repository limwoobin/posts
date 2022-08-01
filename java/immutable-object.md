[**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/java-tech-example/src/main/java/immutable)

<br>

# **불변 객체(Immutable Object)란?**

객체 생성 이후 내부 상태가 변하지 않는, 변경할 수 없는 객체를 이야기합니다.  
<u>불변객체</u>는 내부 상태를 변경하는 메소드를 제공하지 않거나 방어적 복사를 통해 데이터를 제공합니다. 대표적으로 Java의 `String, Integer, Long, Double` 등등 있습니다.

<br>

# **불변 객체(Immutable Object) 의 장점**

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

GC는 새롭게 생성된 객체는 금방 죽는다는 <u>**[Weak Generational Hypothesis](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/generations.html)**</u> 가설에 맞춰 설계되었습니다.  
그래서 불변객체를 새로 생성한다 해서 GC입장에서는 생명주기가 짧은 객체를 처리하는것은 큰 부담이 되지 않습니다.  
그리고 불변객체를 이용하면 불변객체 내부의 객체에 대해서는 GC 스캔 대상에 제외됩니다.  
즉, 불변객체를 이용하면 GC의 스캔빈도와 범위가 줄게되어 GC의 성능에 도움이 된다고 할 수 있습니다.  
하지만 가변객체는 일반적으로 상태를 변경하는 형식으로 사용되고 있기에 불변객체보다 생명주기가 길다고 볼 수 있습니다. 그리고 지속적으로 GC의 스캔범위, 스캔빈도에 포함되어 불변객체보다는 GC성능상 이점이 없습니다.

[https://docs.oracle.com/javase/tutorial/essential/concurrency/immutable.html](https://docs.oracle.com/javase/tutorial/essential/concurrency/immutable.html)

<br>

# **불변객체 생성 규칙**

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

# **불변객체 작성시 유의사항**

불변객체 작성시 유의사항을 정리해보았습니다.  
아래 경우에 대한 예제코드와 함께 살펴보겠습니다.

## **reference type이 필드로 있는 경우**

```java
public class Amount {
    private int value;

    public Amount(int value) {
        this.value = value;
    }

    public void setValue(int value) {
        this.value = value;
    }

    public int getValue() {
        return value;
    }

    @Override
    public String toString() {
        return "amount=" + value;
    }
}
```

```java
public final class ImmutableReference {
    private final int num;
    private final Amount amount;

    public ImmutableReference(int num, Amount amount) {
        this.num = num;
        this.amount = amount;
    }

    public int getNum() {
        return num;
    }

    public Amount getAmount() {
        return amount;
    }
}
```

해당 불변객체는 int라는 primitive type, CustomTime이라는 reference type을 가지고 있습니다.  
이 불변객체는 어느 문제가 되고 있을지 같이 알아보겠습니다.

보기엔 불변객체로서의 모습을 모두 갖춘것처럼 보여집니다.
과연 어느 부분이 문제일지 코드를 이용해 확인해보겠습니다.

```java
@DisplayName("불변 객체 테스트")
class ImmutableReferenceTest {

    @Test
    void immutable_test() {
        Amount amount = new Amount(10);

        ImmutableReference immutableReference = new ImmutableReference(10, amount);
        System.out.println(immutableReference.getAmount()); // (1)

        Amount newAmount = immutableReference.getAmount();
        newAmount.setCount(50);

        System.out.println(immutableReference.getAmount()); // (2)
    }
}
```

![immutable-image-1](https://user-images.githubusercontent.com/28802545/188297925-197b523c-ffdc-4069-bd78-c4a3fd47ff08.png)

코드는 다음과 같습니다.
ImmutableReference로 부터 amount를 가져온 이후에 외부에서 amount의 값을 변경했습니다.  
하지만 변경된 값이 newAmount에만 적용된 것이 아닌 ImmutableReference에도 적용된것을 확인할 수 있습니다.

이 객체의 문제는 다음과 같습니다. 불변객체의 필드로 선언한 Amount 객체의 참조가 외부와 연결되어 있어 그렇습니다.  
그렇기 때문에 안전한 객체를 만드려면 불변객체 내부의 필드 또한 불변객체로 선언되어야 합니다. 혹은 방어적 복사를 이용해 참조관계를 끊어주어야 합니다.  
그래야 외부로부터의 변경에 안전하게 사용할 수 있습니다.

ImmutableReference 객체를 방어적 복사를 이용해 참조를 끊어보겠습니다.

```java
public final class ImmutableReference {
    private final int num;
    private final Amount amount;

    public ImmutableReference(int num, Amount amount) {
        this.num = num;
        this.amount = new Amount(amount.getValue()); // (1)
    }

    public int getNum() {
        return num;
    }

    public Amount getAmount() {
        return new Amount(amount.getValue()); // (2)
    }
}
```

(1): 전달받은 Amount 객체가 외부에서 변경될 수 있어 참조를 끊음
(2): 외부로 전달한 Amount 객체가 외부에서 변경될 수 있어 참조를 끊음

![immutable-image-2](https://user-images.githubusercontent.com/28802545/188298378-76e4a30f-3986-4a17-954d-5e8987a1709e.png)

다음과 같이 방어적 복사를 이용한해 참조관계를 끊고 실행하니 아까와는 달리 외부의 변경에도 불변객체는 값이 변하지 않는것을 확인할 수 있습니다.  
Amount를 방어적 복사가 아닌 불변객체를 이용하면 마찬가지로 객체의 불변성을 지킬 수 있습니다.

<br>

## **Collection이 필드로 있는 경우**

```java
public final class ImmutableCollection {
    private final int num;
    private final List<Integer> list;

    public ImmutableCollection(int num, List<Integer> list) {
        this.num = num;
        this.list = list;
    }

    public int getNum() {
        return num;
    }

    public List<Integer> getList() {
        return list;
    }
}
```

해방 불변객체는 List라는 CollectionType을 필드로 가지고 있습니다. 코드를 이용해 문제점을 살펴보겠습니다.

```java
@DisplayName("불변 객체 Collection 테스트")
class ImmutableCollectionTest {

    @Test
    void immutable_test() {
        List<Integer> list = new ArrayList<>();
        list.add(1);

        ImmutableCollection immutableCollection = new ImmutableCollection(10, list);
        System.out.println(immutableCollection.getList());

        List<Integer> newList = immutableCollection.getList();
        newList.add(2);

        System.out.println(immutableCollection.getList());
    }
}
```

다음 코드는 List를 ImmutableCollection로 전달합니다. 그리고 해당 List를 가져온 후 새로운 요소를 추가합니다.  
과연 ImmutableCollection는 상태가 어떻게 될지 확인해보겠습니다.

![immutable-image-3](https://user-images.githubusercontent.com/28802545/188298828-59b1750a-88c6-4a6a-b8e3-e2756f577205.png)

위 이미지와 같이 ImmutableCollection의 상태가 변경된 것을 확인할 수 있습니다.  
Collection 역시 참조가 연결되어 있기 때문입니다. 이를 방어하기 위해선 방어적 복사, UnmodifiableList와 같은 Unmodifiable Collection을 사용하는 방법이 있습니다.  
Unmodifiable Collection은 값 변경시 예외를 발생시킵니다. 그래서 직접 예외를 처리해주어야 하기 때문에 방어적 복사를 이용해보겠습니다.

<br>

```java
public final class ImmutableCollection {
    private final int num;
    private final List<Integer> list;


    public ImmutableCollection(int num, List<Integer> list) {
        this.num = num;
        this.list = new ArrayList<>(list);
    }

    public int getNum() {
        return num;
    }

    public List<Integer> getList() {
        return new ArrayList<>(list);
    }
}
```

![immutable-image-4](https://user-images.githubusercontent.com/28802545/188299879-81d9c671-7040-4590-9637-d6a89ce2b955.png)

다음과 같이 외부의 변경에도 Collection의 값은 변하지 않는 것을 확인할 수 있습니다.

하지만 위 예제에서 사용된 `List<Integer>` 의 Integer는 불변객체였습니다 가변객체가 들어온다면 어떻게 될까요?  
해당 Collection을 `List<Amount>` 로 변경해서 다시 테스트해보겠습니다.

<br>

## **Collection의 요소가 가변객체인 경우**

<br>

```java
public final class ImmutableCollection {
    private final List<Amount> amounts;

    public ImmutableCollection(List<Amount> amounts) {
        this.amounts = new ArrayList<>(amounts);
    }

    public List<Amount> getAmounts() {
        return new ArrayList<>(amounts);
    }

}

```

```java
@DisplayName("불변 객체 Collection 테스트")
class ImmutableCollectionTest {

    @Test
    void immutable_test() {
        Amount amount = new Amount(1);
        Amount amount2 = new Amount(2);
        List<Amount> amounts = List.of(amount, amount2);

        ImmutableCollection immutableCollection = new ImmutableCollection(amounts);
        System.out.println(immutableCollection.getAmounts());

        amount2.setValue(100);
        System.out.println(immutableCollection.getAmounts());
    }
}
```

List에 Amount객체를 넣고 ImmutableCollection에 전달해 객체를 생성합니다.  
그리고 amount2의 값을 100으로 변경합니다. 테스트를 통해 결과를 확인해보겠습니다.

![immutable-image-5](https://user-images.githubusercontent.com/28802545/188300613-e5ea7d84-ada6-424c-8f57-198a20e03297.png)

방어적 복사를 했음에도 바깥 Collection의 참조는 끊어졌지만 내부 Object에 대한 참조는 끊기지 않음을 확인할 수 있습니다.

```java
@HotSpotIntrinsicCandidate
public static native void arraycopy(Object src,  int  srcPos,
                                    Object dest, int destPos,
                                    int length);
```

방어적 복사시 호출되는 `arraycopy`는 얕은 복사를 이용해 객체를 카피하기 때문에 참조가 유지되게 됩니다.

[https://docs.oracle.com/javase/7/docs/api/java/lang/System.html](https://docs.oracle.com/javase/7/docs/api/java/lang/System.html)

결국 ImmutableCollection 객체는 불변성이 보장되지 않는 객체인 것입니다.  
결국 불변객체를 사용하기 위해서는 사용되는 필드 마찬가지로 불변객체를 사용하는것이 안전하다고 볼 수 있습니다.

감사합니다.

<br>

## reference

[https://howtodoinjava.com/java/basics/how-to-make-a-java-class-immutable/](https://howtodoinjava.com/java/basics/how-to-make-a-java-class-immutable/)  
[https://www.baeldung.com/java-immutable-object](https://www.baeldung.com/java-immutable-object)
