> <br>
>
> ## **Overview**
>
> - ### [**개요**](#개요)
> - ### [**mapstruct**](#mapstruct)
>   - [**사용하기**](#사용하기)
>   - [**객체 변환하기**](#객체-변환하기)
>   - [**문제점**](#subject-2)
>     - [**NoArgsConstructer(access = AccessLevel.PROTECTED)**](#content-1)
>     - [**@Setter**](#content-2)
> - ### [**mapstruct builder pattern**](#subject-3)
>   - [**content 3 / 4**](#content-3--4)
>   - [**문제점**](#zz) <br><br>

<br />

#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin)

<br />

# **개요**

소스코드를 작성하다보면 Layer를 전환하며 객체를 전환하며 매핑하거나 여러 객체를 합치거나 하는 다양한 경우를 만나게 됩니다.  
흔히 겪는 예시로는 presentation layer 에서는 DTO , service layer , repository layer 에서는 Entity 를 사용하는 예시를 들 수 있겠죠.  
이를 매핑하기 위해서는 **model mapper , 정적 팩토리 , object mapping** 등의 방법을 다양한 이용해 모델을 매핑하고 있습니다.  
저는 제가 사용하는 **mapstruct** 에 대해 간략하게 소개하려고 합니다.

<br>

# **mapstruct**

[**mapstruct github page**](#https://github.com/mapstruct/mapstruct#what-is-mapstruct)에서는 mapstrut를 다음과 같이 소개하고 있습니다.

> <br>
> 간략하게 요약하면 다음과 같습니다.  <br>
> 타입에 안전한 Bean Mapping 클래스를 생성하기 위한 Java Annotation Processing
>
> 런타임에 작동하는 Mapping framework 와 비교했을때 MapStruct 는 다음과 같은 이점을 갖는다
>
> - reflection 대신 일반 method 를 사용하기 때문에 빠름
> - 컴파일시 오류를 확인할 수 있음
> - Implementation code 를 제공해 쉽게 디버깅이 가능 , 직접 확인 가능
>
>   <br>

<br />

## **사용하기**

<br>
<br>

# Subject 2

### content 1

...
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
