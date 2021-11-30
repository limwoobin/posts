> <br>
>
> ## **Overview**
>
> - [**OCP (Open Closed Principle) 정의**](#ocp-open-closed-principle-란)
>   - [content 0](#content-0)
> - [**요구사항**](#요구사항)
>
>   - [content 1](#content-1)
>   - [content 2](#content-2)
>
>     <br>

<br />
<br />
<br />

# **OCP(Open Closed Principle) 정의**

- 개방 폐쇄 원칙
- 소프트웨어 개체는 확장에 열려있어야 하고 수정에는 닫혀있어야 한다는 원칙.

<br>
<br>

### **OCP(개방 폐쇄 원칙)이 무슨 의미일까 ??**

<hr>

어느 기능에 대한 추가 요구사항이 나타나도 그 기능을 사용하는 기존 코드는 수정되지 않아야 한다는 의미입니다.  
확장되는 기능이 기존 코드와 의존을 하게되면 추가되는 요구사항에 유연하지 못하고  
개발자가 유지보수하는데 어려움이 있습니다.

<br>

**OCP(개방 폐쇄 원칙)** 을 준수하지 않은 예시와 준수한 예시 두가지를 들어 비교해보겠습니다

<br>

스카이스캐너 , 마이리얼트립과 같은 항공편 예약 서비스를 만든다고 가정해보겠습니다.  
그렇다면 예약을 하기 위해서는 각 항공사과 예약에 관련된 api 통신을 주고 받아야 하고  
각 항공사들마다 사용하는 데이터 포맷(ex - DTO)도 맞춰야 하고 신경써야할 부분이 굉장히 많을것 같습니다.  
서비스를 운영하다보면 중간에 추가되거나 삭제되는 항공사도 있겠죠??

> OCP 를 준수하지 않은 코드

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

그리고 새로운 요구사항을 받았습니다.

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

<br>

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
<br>

#### **이러한 문제들을 추상화 , factory method 를 이용해 유연하게 리팩토링 해보겠습니다.**

controller와 enum 은 동일하게 사용하겠습니다

<br>
<br>

### **OCP 를 준수해야 하는 이유**

<hr>

OCP 원칙은 객체지향 프로그래밍의 핵심 원칙이라고 볼 수 있습니다.
이 원칙을 지키면 객체의 유연성 , 재사용성 , 유지보수성을 얻을 수 있습니다.

<br>
<br>

# **요구사항**

OCP 를 팩토리메서드를 이용해서 준수하는 방법을 예시로 작성

<br>
<br>

### content 2

...
<br>
<br>

# **Subject 3**

### content 3 / 4

...
<br>

```

```
