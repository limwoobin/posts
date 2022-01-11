> <br>
>
> ## **Overview**
>
> - ### [**OCP (Open Closed Principle) 정의**](#ocp-open-closed-principle-정의)
>
>   - [OCP (개방 폐쇄 원칙)의 의미](#ocp-개방-폐쇄-원칙의-의미)
>
> - ### [**예제 코드**](#예제-코드)
>
>   - [OCP 를 준수하지 않은 코드](#ocp-를-준수하지-않은-코드)
>   - [OCP 를 준수한 코드](#ocp-를-준수한-코드)
>
> - ### [**OCP 를 준수해야 하는 이유**](#ocp-를-준수해야-하는-이유) <br><br>

<br />

#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/oop-example/src/main/java/com/example/oopexample/ocp)

<br />
<br />

# **OCP (Open Closed Principle) 정의**

- 개방 폐쇄 원칙
- 소프트웨어 개체는 확장에 열려있어야 하고 수정에는 닫혀있어야 한다는 원칙.

<br>

### **OCP (개방 폐쇄 원칙)의 의미**

<hr>

어떤 기능에 대해 추가 요구사항이 나타나도 그 기능을 사용하는 기존 코드는 수정되지 않아야 한다는 의미입니다.  
확장되는 기능이 기존 코드와 의존을 하게되면 추가되는 요구사항에 유연하지 못하고  
개발자가 유지보수하는데 어려움이 있습니다.

<br>

# **예제 코드**

**OCP (개방 폐쇄 원칙)** 을 준수하지 않은 예시와 준수한 예시 두가지를 들어 비교해보겠습니다

스카이스캐너 , 마이리얼트립에서 제공하는 항공편 예약 서비스를 만든다고 가정해보겠습니다.  
그렇다면 예약을 하기 위해서는 각 항공사와 예약에 관련된 api 통신을 주고 받아야 하고  
항공사들마다 요구하는 사항들도 각각 다릅니다.  
서비스를 운영하다보면 중간에 추가되거나 삭제되는 항공사도 있겠죠??

<br>

### **OCP 를 준수하지 않은 코드**

<hr>
<br>

ReservationService.java

```java
@RequiredArgsConstructor
@RestController
@RequestMapping("/reservation")
public class ReservationController {
	private final ReservationService reservationService;

	@PostMapping("/")
	public ResponseEntity<ReservationDTO> getReservation(ReservationRequest request) {
			ReservationDTO reservationDTO = reservationService.reservation(request);
			return new ResponseEntity<>(reservationDTO , HttpStatus.OK);
	}
}

```

ReservationService.java

```java
@Service
@RequiredArgsConstructor
public class ReservationService {

	public ReservationDTO reservation(ReservationRequest request) {
			AirlineType type = request.getAirlineType();

			if (AirlineType.KOREAN_AIR.equals(type)) {
				// 대한항공 예약 api 호출
			} else if (AirlineType.ASIANA_AIR.equals(type)) {
				// 아시아나 예약 api 호출
			} else if (AirlineType.AIR_SEOUL.equals(type)) {
				// 에어서울 예약 api 호출
			}
			...

	}
}
```

AirlineType.java

```java
public enum AirlineType {
	KOREAN_AIR,
	ASIANA_AIR,
	AIR_SEOUL;
}
```

위의 코드를 보시면 각 항공사에게 예약 api를 호출하는 부분을 reservation method 에서 처리하는것을 볼 수 있습니다.

그런데 새로운 요구사항을 받았습니다.

1. 진에어 항공사 추가
2. 예약 취소 기능 추가

구현해보겠습니다.

```java
@Service
@RequiredArgsConstructor
public class ReservationService {

	public ReservationDTO reservation(ReservationRequest request) {
			AirlineType type = request.getAirlineType();

			if (AirlineType.KOREAN_AIR.equals(type)) {
				// 대한항공 예약 api 호출
			} else if (AirlineType.ASIANA_AIR.equals(type)) {
				// 아시아나 예약 api 호출
			} else if (AirlineType.AIR_SEOUL.equals(type)) {
				// 에어서울 예약 api 호출
			} else if (AirlineType.JIN_AIR.equals(type)) {
				// 진에어 예약 api 호출
			}
			...
	}

	public ReservationDTO cancelReservation(ReservationRequest request) {
			AirlineType type = request.getAirlineType();

			if (AirlineType.KOREAN_AIR.equals(type)) {
				// 대한항공 예약 취소 api 호출
			} else if (AirlineType.ASIANA_AIR.equals(type)) {
				// 아시아나 예약 취소 api 호출
			} else if (AirlineType.AIR_SEOUL.equals(type)) {
				// 에어서울 예약 취소 api 호출
			} else if (AirlineType.JIN_AIR.equals(type)) {
				// 진에어 예약 취소 api 호출
			}
			...
	}
}
```

AirlineType.java

```java
public enum AirlineType {
	KOREAN_AIR,
	ASIANA_AIR,
	AIR_SEOUL,
	JIN_AIR;
}
```

여기에 만약 항공사마다 다른 방식의 데이터 형식을 요구한다고 가정해보겠습니다.  
그렇다면 각 항공사에 맞는 DTO 가 필요할수도 있습니다.  
어느 항공사는 강화된 인증을 필요로 하여 토큰을 요구한다거나 등등의 이유로 위 코드는 변경 및 확장될 여지가 너무나도 명확합니다.

**이 예제는 이와 같은 다이어그램을 갖게 됩니다.**

![ocp-image-1](https://user-images.githubusercontent.com/28802545/144230784-f5a63036-fcc7-4fba-ab4c-ae1f3d63fd19.PNG)

**이 코드의 문제점은 다음과 같습니다.**

- 기능 추가(예약 , 예약취소)에 따른 if - else 문이 계속 추가된다.
- 항공사가 추가될때마다 모든 if - else block 을 찾아 추가해야 한다.
- 기능이 확장되는데 기존 기능을 사용하는 코드도 같이 수정되기 때문에 side effect 위험이 있다.

즉, 이 코드는 확장이 발생했을때에 **해당 코드를 호출하는쪽** 에 변경이 발생하고 있습니다.  
이는 변경에 닫혀있지 않다는 뜻입니다.

> <br>
> 해당 코드를 호출하는쪽 ?
>
> - 확장이 발생했을때의 해당 코드: 추가된 항공사의 기능 구현(api 호출 코드)
> - 해당 코드를 호출하는 쪽: 추가된 항공사의 기능 구현 코드를 호출하는 Service 의 method (예약 , 예약취소)
>
> <br>

<br>

#### **이러한 문제들을 추상화, 다형성, factory method 를 이용해 리팩토링 해보겠습니다.**

<br>

### **OCP 를 준수한 코드**

<hr>

Service 에서 구현에 직접 의존하던 로직을 인터페이스에 의존하도록 인터페이스를 추가합니다.

```java
public interface AirlineReservation {
    ReservationDTO reserve(ReservationRequest request);
    ReservationDTO cancel(ReservationRequest request);
}
```

<br>

그리고 각 항공사별로 구현클래스를 만들어 Service 로직에서 가지고 있던 책임을 분리했습니다.

```java
@Component
public class KoreanAirReservation implements AirlineReservation {

    @Override
    public ReservationDTO reserve(ReservationRequest request) {
        // 대한항공 예약 로직...
				// 추가 인증 로직...
    }

    @Override
    public ReservationDTO cancel(ReservationRequest request) {
        // 대한항공 예약 취소 로직...
    }
}

@Component
public class AsianaReservation implements AirlineReservation {

    @Override
    public AsianaResponseDTO reserve(ReservationRequest request) {
        // 아시아나 예약 로직...
    }

    @Override
    public AsianaResponseDTO cancel(ReservationRequest request) {
        // 아시아나 예약 취소 로직...
    }

		// 아시아나 전용 response 에서 우리 서비스의 DTO 로 변환
		private ReservationDTO convert(AsianaResponseDTO responseDTO) {
			// convert...
		}
}

@Component
public class AirSeoulReservation implements AirlineReservation {

    @Override
    public ReservationDTO reserve(ReservationRequest request) {
        // 에어서울 예약 로직...
    }

    @Override
    public ReservationDTO cancel(ReservationRequest request) {
        // 에어서울 예약 취소 로직...
    }
}
```

각 항공사 구현체는 AirlineReservation 인터페이스를 상속받았습니다.  
그리고 구현 클래스에서는 각 항공사별 추가 요구사항을 구현했습니다.

1. 대한항공은 추가 인증로직이 필요
2. 아시아나항공은 api통신에 필요한 전용 format에 맞춰 통신

이러한 추가 요구사항들을 각 항공사별 클래스에서 담당하게 했습니다.

<br>

AirLineReservationFactory.java

```java
@Component
@RequiredArgsConstructor
public class AirLineReservationFactory {
    private final KoreanAirReservation koreanAirReservation;
    private final AsianaReservation asianaReservation;
    private final AirSeoulReservation airSeoulReservation;

    public AirlineReservation createReservationService(AirlineType airlineType) {
        switch (airlineType) {
            case KOREAN_AIR: return koreanAirReservation;
            case ASIANA_AIR: return asianaReservation;
            case AIR_SEOUL: return airSeoulReservation;
        }

        throw new IllegalArgumentException();
    }
}
```

<br>

ReservationService.java

```java
@Service
@RequiredArgsConstructor
public class ReservationService {
    private final AirLineReservationFactory airLineReservationFactory;

    public ReservationDTO reservation(ReservationRequest request) {
        AirlineReservation airlineReservation = airLineReservationFactory.createReservationService(request.getAirlineType());
        return airlineReservation.reserve(request);
    }

    public ReservationDTO cancelReservation(ReservationRequest request) {
        AirlineReservation airlineReservation = airLineReservationFactory.createReservationService(request.getAirlineType());
        return airlineReservation.cancel(request);
    }
}
```

**기존에 if ~ else 블록으로 처리하던 코드를 factory 클래스를 이용해 객체 생성을 하게끔 변경했습니다.**  
이제 항공사가 추가되어도 항공사 api 를 호출하는 Service 의 로직은 수정이 필요 없어졌습니다.  
항공사 관련 로직들은 각 구현체에서 처리하고 그 구현체는 factory class 에서 가져오기 떄문입니다. 추가된다하면 factory class , 구현체만 추가하면 되기 때문이죠.

그렇다면 항공사를 하나 추가해보겠습니다.

```java
@Component
public class TwayReservation implements AirlineReservation {
    @Override
    public ReservationDTO reserve(ReservationRequest request) {
        // 티웨이 예약 로직...
    }

    @Override
    public ReservationDTO cancel(ReservationRequest request) {
        // 티웨이 예약 취소 로직...
    }
}
```

<br>

```java
@Component
@RequiredArgsConstructor
public class AirLineReservationFactory {
    private final KoreanAirReservation koreanAirReservation;
    private final AsianaReservation asianaReservation;
    private final AirSeoulReservation airSeoulReservation;
    private final TwayReservation twayReservation;

    public AirlineReservation createReservationService(AirlineType airlineType) {
        switch (airlineType) {
            case KOREAN_AIR: return koreanAirReservation;
            case ASIANA_AIR: return asianaReservation;
            case AIR_SEOUL: return airSeoulReservation;
            case T_WAY: return twayReservation;
        }

        throw new IllegalArgumentException();
    }
}
```

티웨이 항공이 추가되며 TwayReservation 가 추가되었고 factory class 도 그에 맞춰 수정되었습니다.

<br>

```java
@Service
@RequiredArgsConstructor
public class ReservationService {
    private final AirLineReservationFactory airLineReservationFactory;

    public ReservationDTO reservation(ReservationRequest request) {
        AirlineReservation airlineReservation = airLineReservationFactory.createReservationService(request.getAirlineType());
        return airlineReservation.reserve(request);
    }

    public ReservationDTO cancelReservation(ReservationRequest request) {
        AirlineReservation airlineReservation = airLineReservationFactory.createReservationService(request.getAirlineType());
        return airlineReservation.cancel(request);
    }
}
```

**이 예제는 이와 같은 다이어그램을 갖게 됩니다.**

![ocp-image-1](https://user-images.githubusercontent.com/28802545/144230789-f0cbf236-83c2-42f7-818d-80eaf01a9027.PNG)

기존 다이어그램과 비교해보면 AirlineReservation 이라는 추상화된 인터페이스를 가지고 통신하므로 항공사가 추가되거나 변경되어도 호출하는쪽은 전혀 영향을 받지 않습니다.

실제로도 확장된 코드를 호출하는 ReservationService.java 는 코드가 전혀 수정되지 않았습니다.  
기존 구조였다면 Service에 수많은 if ~ else 블록들을 수정하고 각 항공사에 맞는 기능들을 구현했을텐데요.  
**추가되는 항공사 , 기능에도 훨씬 유연하게 대응할 수 있을것으로 보입니다.**

<br>

# **OCP 를 준수해야 하는 이유**

OCP 원칙은 객체지향 프로그래밍의 핵심 원칙이라고 볼 수 있습니다.  
위 예제를 보셨듯이 OCP원칙을 준수하면 다음과 같은 장점을 얻을 수 있습니다.

- **기능이 추가되거나 기존 로직을 건들지 않으니 확장에 유연하다**
- **기존 로직에 비해 유지보수하기 좋다**
- **클래스의 재사용성 증가**

<br>
<br>

감사합니다.
