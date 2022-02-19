> <br>
>
> ## **Overview**
>
> - ### [**개요**](#개요)
> - ### [**mapstruct**](#subject-1)
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

```java

public class Template {
	private int a;
	private int b;

	public Template(int a , int b) {
		this.a = a;
		this.b = b;
	}
}

```

<br />
<br />

`test word block`

<br />
<br />

# Subject 1

### content 0

...

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
