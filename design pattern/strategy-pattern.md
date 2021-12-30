> <br>
>
> ## **Overview**
>
> - [**전략 패턴(Strategy Pattern)이란?**](#subject-1)
> - [**전략 패턴 예제 코드**](#subject-2)
>   - [content 1](#content-1)
>   - [content 2](#content-2)
> - [**전략 패턴을 사용하는 이유**](#subject-3)
>   - [content 3 / 4](#content-3--4)
> - [**전략 패턴 예시**](#subject-3)
>   - [content 3 / 4](#content-3--4)
> <br><br>

<br />

#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/oop-example/src/main/java/com/example/oopexample/ocp)

<br />
<br />

# **전략 패턴(Strategy Pattern)이란?**

> <br>
> 다음은 위키피디아에서 정의하는 전략 패턴(strategy pattern)입니다.
> <br>
> &nbsp;<br>
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

![strategy-pattern-image-1](https://user-images.githubusercontent.com/28802545/147659160-4e03d785-a022-44d3-a7d2-03ed50d2a590.png)
<br>

- Context: Strategy 를 사용하는 객체
- Strategy: 전략에 대한 인터페이스를 담당하는 객체
- ConcreteStrategy: 전략에 대해 캡슐화된 구현을 담당하는 객체

위 그림과 같이 전략(Strategy)을 인터페이스로 추상화하고  
그 전략을 각자의 캡슐화된 알고리즘으로 구현합니다.

<br>
<br>

# __전략 패턴 예제 코드__

실생활과 연관지어 예시를 들어보겠습니다.  


### content 1

...
<br>
<br>

### content 2

...
<br>
<br>

# **전략 패턴을 사용하는 이유**

- 추가되는 요구사항(strategy)에 대처가 유연하다(OCP 원칙을 준수할 수 있음)
- Strategy 의 구현체가 Context와 분리되기 때문에 알고리즘에만 집중할 수 있음
- 객체의 전략이 분리되어있기 때문에 전략이 필요한 곳에 재사용 가능

<br>

# **전략 패턴 예시**

### content 3 / 4

...
<br>
