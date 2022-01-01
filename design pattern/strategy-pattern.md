> <br>
>
> ## **Overview**
>
> - [**전략 패턴(Strategy Pattern)이란?**](#subject-1)
> - [**전략 패턴 예제 코드**](#subject-2)
>   - [content 1](#content-1)
>   - [content 2](#content-2)
> - [**전략 패턴을 사용하는 이유**](#subject-3)
>   - [content 3 / 4](#content-3--4) <br><br>

<br />

#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/oop-example/src/main/java/com/example/oopexample/ocp)

<br />
<br />

# **전략 패턴(Strategy Pattern)이란?**

> <br>
> 다음은 위키피디아에서 정의하는 전략 패턴(strategy pattern)입니다.
> <br>
> <br>
>
> **전략 패턴(strategy pattern) 또는 정책 패턴(policy pattern)은 실행 중에 알고리즘을 선택할 수 있게 하는 행위 소프트웨어 디자인 패턴이다.**
>
> - 특정한 계열의 알고리즘들을 정의하고
> - 각 알고리즘을 캡슐화하며
> - 이 알고리즘들을 해당 계열 안에서 상호 교체가 가능하게 만든다.
>   <br><br>

객체의 행위 혹은 알고리즘을 전략(strategy)이라 합니다.  
즉 객체의 행위를 쉽게 변경하기 위해 등장한 패턴이라 할 수 있습니다.

<br>

![strategy-pattern-image-1](https://user-images.githubusercontent.com/28802545/147727923-03894d2d-ba83-478c-8c75-fd95383ac972.png)
<br>

- **Context**: Strategy 를 사용하는 객체
- **Strategy**: 전략에 대한 인터페이스를 담당하는 객체
- **ConcreteStrategy**: 전략에 대해 캡슐화된 구현을 담당하는 객체

위 그림과 같이 전략(Strategy)을 인터페이스로 추상화하고  
그 전략을 각자의 캡슐화된 알고리즘으로 구현합니다.  
어느 부분이 변화되는지 찾고 변화되는 부분을 캡슐화하는것이 중요합니다.

<br>
<br>

# **전략 패턴 예제 코드**

실생활과 연관지어 예시를 들어보겠습니다.  
집앞에 대형마트가 있어 평소 물건을 사러 자주 방문합니다.  
그 대형마트에는 고객의 등급별로 결제시 포인트 적립 금액이 다릅니다.

마트에는 아래와 같은 고객 등급이 존재합니다.

- **VIP : 결제금액의 10% 적립**
- **GOLD : 결제금액의 5% 적립**
- **NORMAL : 혜택 없음**

<br>

고객의 등급에 따라 포인트를 적립하는 행위를 코드로 작성해보겠습니다

Consumer.java

```java
public interface Consumer {
    int payment(int price);
}
```

<br>

GoldConsumer.java

```java
public class GoldConsumer implements Consumer {
    private static final double POINT_RATE = 0.05;

    @Override
    public int payment(int price) {
        return (int) (price * POINT_RATE);
    }
}
```

<br>

VipConsumer.java

```java
public class VipConsumer implements Consumer {
    private static final double POINT_RATE = 0.1;

    @Override
    public int payment(int price) {
        return (int) (price * POINT_RATE);
    }

}

```

<br>

위 코드를 보시면 Consumer Interface 를 구현한 클래스들이 각 Gold , Vip 등급에 따라  
POINT_RATE(적립 비율)을 다르게 갖고 있는것을 보실 수 있습니다.

<br>

PaymentService.java

```java
public class PaymentService {
    private final Consumer consumer;

    public PaymentService(Consumer consumer) {
        this.consumer = consumer;
    }

    public int payForAmount(int price) {
        return consumer.payment(price);
    }
}
```

PaymentService 는 Consumer 를 생성자로 주입받아 payment를 호출합니다.

<br>

PayController.java (main)

```java
public class PayController {
    public static void main(String[] args) {
        Consumer goldConsumer = new GoldConsumer();

        PaymentService paymentService = new PaymentService(goldConsumer);
        int goldPoint = paymentService.payForAmount(30000);
        System.out.println("goldPoint: " + goldPoint);

        Consumer vipConsumer = new VipConsumer();
        paymentService = new PaymentService(vipConsumer);
        int vipPoint = paymentService.payForAmount(30000);
        System.out.println("vipPoint: " + vipPoint);
    }
}
```

```
goldPoint: 1500
vipPoint: 3000

Process finished with exit code 0
```

언뜻보면 위 코드도 이미 전략 패턴을 이용한 코드로 느껴지실 수 있습니다.  
반은 맞고 반은 틀리다 생각합니다.  
PaymentService 에서는 Consumer 를 주입받아 각 행위마다 GoldConsumer , VipConsumer 라는 알고리즘을 변경하며 사용하기 때문이죠.  
VipConsumer , GoldConsumer 각자의 책임에 대한 적립정책을 가지고 있습니다.  
<br>
하지만 이 코드에도 역시 문제점이 존재합니다.  
앞서 전략 패턴은 변경되는 행위에 대해 캡슐화를 해야 한다고 했습니다.

만약 VipConsumer 는 적립비율외에 구매금액이 30000원 이상이라면 추가 적립금 2000원이 주어진다고 가정해보겠습니다.

```java
public class VipConsumer implements Consumer {
   private static final double POINT_RATE = 0.1;

    @Override
    public int payment(int price) {
			int point = 0;
			if (price > 30000) {
				point += 2000;
			}

			point += (int) (price * POINT_RATE);
			return point;
    }
}
```

새로운 기능으로 변경하기 위해 기존 코드가 수정된 모습입니다.  
이는 흔히 말하는 [**ocp(open-closed-principle)**](https://www.devoong.com/posts/3)이 위배된 모습입니다.

- 다른 등급이지만 같은 정책일 경우 코드가 중복될 수 있음
- 중복이라는것은 즉 코드변경시 변경이 여러군데에서 일어날 수 있음
  <br>

포인트 적립 정책은 언제든지 변경될 수 있는 여지가 있습니다.  
이를 캡슐화하여 전략 패턴을 적용해보겠습니다.

<br>

Vip고객의 포인트 적립 정책에 대해 전략패턴을 적용해보겠습니다.

<br>

VipPointStrategy.java

```java
public interface VipPointStrategy {
    double POINT_RATE = 0.1;

    int earn(int price);
}
```

```java
public class VipOverPriceStrategy implements VipPointStrategy {
    @Override
    public int earn(int price) {
        int point = 0;
        if (price >= 30000) {
            point += 2000;
        }

        point += price * POINT_RATE;
        return point;
    }
}
```

<br>

VipConsumer.java

```java
public class VipConsumer implements Consumer {
    private VipPointStrategy vipPointStrategy;

    public VipConsumer(VipPointStrategy vipPointStrategy) {
        this.vipPointStrategy = vipPointStrategy;
    }

    @Override
    public int payment(int price) {
        return vipPointStrategy.earn(price);
    }
}
```

PayController.java(main)

```java
public class PayController {
    public static void main(String[] args) {
        Consumer goldConsumer = new GoldConsumer();

        PaymentService paymentService = new PaymentService(goldConsumer);
        int goldPoint = paymentService.payForAmount(30000);
        System.out.println("goldPoint: " + goldPoint);

        Consumer vipConsumer = new VipConsumer(new VipOverPriceStrategy());
        paymentService = new PaymentService(vipConsumer);
        int vipPoint = paymentService.payForAmount(30000);
        System.out.println("vipPoint: " + vipPoint);
    }
}
```

```
goldPoint: 1500
vipPoint: 3000

Process finished with exit code 0
```

VipConsumer 는 이제 VipPointStrategy 를 주입받아 적립금을 계산합니다.  
만약 Vip고객의 포인트정책이 추가된다면 새로운 전략을 추가하면 되고  
GoldConsumer 에게 새로운 포인트 정책이 추가되거나 변경된다면 새로운 전략을 추가하여 주입해주면 됩니다.

<br>

# **전략 패턴을 사용하는 이유**

- 추가되는 요구사항(strategy)에 대처가 유연하다(OCP 원칙을 준수할 수 있음)
- Strategy 의 구현체가 Context와 분리되기 때문에 알고리즘에만 집중할 수 있음
- 객체의 전략이 분리되어있기 때문에 전략이 필요한 곳에 재사용 가능

<br>
