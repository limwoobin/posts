# __Rest API, Rest, Restful API 란__ ?

// rest api image 넣기

안녕하세요 이번엔 RestAPI, REST, Restful 에 대해 알아보겠습니다.  

REST는 __하이퍼미디어 기반 분산 시스템(WEB)__ 을 구축하기 위한 아키텍처 스타일입니다.

REST는 REpresentational State Transfer의 약자로 2000년도 __로이 필딩 (Roy Fielding)__ 이라는 사람의 [박사학위 논문](https://ics.uci.edu/~fielding/pubs/dissertation/top.htm) 에서 최초로 소개되었습니다.

REST의 등장 배경에는 분산 시스템에서의 효율적이고 확장 가능한 통신을 위한 표준을 제공하기 위해 만들어졌습니다.

<br />

## __REST 가 무엇인가요?__

__`REpresentational State Transfer`__ 의 약자로
자원을 이름(자원의 표현)으로 구분해 해당 자원의 상태(정보)를 주고 받는 모든 것을 의미합니다.

즉, REST는 HTTP URI를 이용해 자원을 URI로 식별하고, HTTP 메서드(GET, POST, PUT, DELETE 등)를 사용하여 자원에 대한 행위를 정의하고 상태를 전달합니다.

그리고 이런 REST 아키텍처 스타일을 따르는 API를 __REST API__ 라고 합니다.  
다음은 REST 의 구성 요소와 특징에 대해 알아보겠습니다.

<br />

### REST 의 구성 요소

REST API 는 다음의 구성으로 이루어져 있습니다.

1. __자원(Resource)__
- 모든 자원은 식별자가 존재하고, 이 자원은 Server 에 존합니다.
- 자원은 HTTP URI 를 통해 표현합니다.
- 자원은 Collection 과 Element 로 나누어 표현할 수 있습니다.
    - Collection URI: `http://test-example.com/items`
    - Element URI: `http://test-example.com/items/1`

<br />

2. __행위(Verb)__
- HTTP Method 를 사용합니다.
- HTTP Method 는 다음과 같은 CRUD Operation 을 제공합니다.

| HTTP Method  | Description  |
| --- | ----- |
| POST   | 생성(Create) |
| GET   | 조회(Read) |
| PUT   | 수정(Update) <br /> 자원을 모두 변경 |
| PUT   | 수정(Update) <br /> 자원의 일부를 변경 |
| DELETE   | 삭제(Delete) |

<br />

3. __표현(Representation of Resource)__
- Client 와 Server 의 데이터를 주고받는 형태, 표현 방법입니다.
- 하나의 자원은 JSON, XML, RSS, TEXT 등의 여러 형태로 표현될 수 있습니다.
- JSON, XML 을 통해 주고받는 것이 일반적입니다.

<br />

## __REST 의 특징__

### 1. __Server-Client 구조__

- REST 서버는 API 를 제공하고 Client 는 사용자 인증이나 컨텍스트를 관리합니다.
- 각각의 역할이 구분되기에 서로 간의 의존성이 줄어들게 됩니다.

### 2. __Stateless - 무상태__

- HTTP Protocol 은 무상태성의 성격을 갖기에 REST 역시 무상태성의 성격을 갖습니다.
- 쿠키나 세션 정보를 별도로 저장 / 관리하지 않습니다.
- 서버는 들어오는 요청만 단순히 처리하면 되어 구현이 단순해집니다.

### 3. __Cacheable - 캐싱 가능__

- HTTP 라는 웹 표준을 사용하기에 HTTP 의 캐싱 기능을 사용할 수 있습니다.
- HTTP 표준에서 사용하는 Last-Modified Tag or E-Tag를 이용해 캐싱을 구현할 수 있습니다.

### 4. __Layerd System - 계층화__

- Client 는 REST API Server 만 호출합니다.
- REST Server는 여러 계층으로 구성될 수 있습니다.
    - 여러 계층에는 보안, 로드밸런싱, 암호화 계층을 추가해 구조를 유연하게 가져갈 수 있습니다.
    - Proxy, Gateway 와 같은 네트워크 기반의 중간매체를 사용할 수 있습니다.

### 5. __Uniform Interface - 인터페이스 일관성__
- Resource(URI)에 대한 요청을 통일되고, 한정적인 인터페이스로 수행합니다.
- 특정 기술이나 플랫폼에 종속되지 않고 (ex. Android, IOS, Java ...) HTTP Protocol 을 따르는 모든 플랫폼에서 사용 가능합니다.

__Uniform Interface__ 에는 4가지 제약조건들이 존재합니다.

#### __Identification of Resources__
- 각 자원은 고유한 URI로 식별됩니다. 
- 클라이언트와 서버의 상호작용에서 자원은 고유하게 식별될 수 있어야 합니다.

#### __Manipulation of Resources through Representations__
- 자원의 표현(Representations) 을 통해 자원의 조작이 이루어져야 합니다.
- HTTP 메서드(GET, POST, PUT, DELETE 등) 를 이용하여 자원을 어떻게 조작할지 정의합니다.

#### __Self-Descriptive Messages__ 
- 메시지만 보고도 이를 쉽게 이해 할 수 있도록 자체 표현 구조로 되어있어야 합니다.

__요청__

[좋지 않은 예시]
```
GET / HTTP/1.1
```

위 요청은 루트를 호출하기는 하지만 목적지가 없어 self-descriptive 하지 않습니다.

<br />

[좋은 예시]
```
GET / HTTP/1.1
HOST : https://www.example.com 
```

위 예시는 목적지가 명시되어있으므로 해당 메시지가 어떤 메시지인지 알 수 있습니다. 그렇기에 self-descritive 하다고 할 수 있습니다.

<br />

__응답__

[좋지 않은 예시]
```
HTTP/1.1 200 OK
{
    id: "example", 
    name: "test name" 
}
```

해당 응답은 json 형식의 데이터를 담고 있지만 본문에 대해서는 어떤 데이터 형식인지 알 수 없습니다.

```
HTTP/1.1 200 OK
Content-Type: application/json
{
    id: "example", 
    name: "test name" 
}
```

Content-Type 을 추가해주어 이제 본문의 메시지가 어떤 타입인지,  
중괄호가 무엇인지 문자열은 무엇인지 알 수 있게되었습니다.

하지만 그렇다고 해도 이 정보만으로는 id, name 이 무엇을 의미하는지는 알 수 없습니다.

<br />

[좋은 예시]
```
HTTP/1.1 200 OK
Content-Type: application/json
Link: <https://devoong2.com/docs>; rel="profile"
{
    id: "example", 
    name: "test name" 
}
```

다음과 같이 Link 헤더에 API 의 명세를 확인할 수 있는 방법이 있습니다.  
해당 방법을 이용하면 응답을 받는 측에서도 id, name 이 무엇을 의미하는지 알 수 있습니다.

이런 응답은 self-descriptive 하다고 할 수 있습니다.

#### __Hypermedia as the Engine of Application State (HATEOAS)__
서버로부터 받은 리소스의 표현을 통해 다음에 취할 수 있는 액션에 대한 링크를 제공합니다.

<br />

### 6. __Code on Demand (Optional)__

- Server 로 부터 Script 를 받아서 Client 에서 실행합니다.
- 반드시 충족할 필요는 없습니다.

<br />

__Optional 에 대한 이유 ?__

__보안문제__
- 서버에서 실행 가능한 코드를 클라이언트로 전송하게 되면 해당 코드는 서버의 컨트롤을 벗어나기에 악용될 수 있는 위험이 존재합니다.

__호환성__ 
- REST 서비스는 WEB, IOS, Android 등 다양한 플랫폼들과 상호작용 합니다.  
- 만약 해당 코드가 특정 플랫폼에 종속된다면 API 의 활용도가 떨어지게 됩니다.

__성능 이슈__
- code on demand 의 경우 서버에서 내려주는 응답은 동적이기에 HTTP Protocol 의 장점인 캐싱을 적용하기 어렵습니다. 
- 캐싱 기능을 사용하지 못하면 성능에 악영향을 미칠 수도 있습니다.

그럼에도 불구하고 왜 REST 제약조건에 code on demand 가 포함되었는지 궁금해서  
stackoverflow 의 질문을 참고해 정리했습니다.

<br />

Code-on-demand 원칙은 요즘 거의 사용되고 있지 않습니다.  
REST 논문이 작성될 당시의 웹은 거의 정적 document 였고 Client 도 웹 브라우저 뿐이었습니다.  
당시의 클라이언트는 지금의 js 와는 다르게 로직을 구현하기 쉽지 않았다는 환경이었습니다.  
그래서 위와 같은 제약이 만들어 지지 않았을까 싶습니다.

##### 참고 https://stackoverflow.com/questions/32094952/code-demand-constraint-for-restful-apis

<br />
<hr />

## REST API 가 무엇인가요?


### REST API 설계 규칙

### REST API 작성 주의사항

<br />
<hr />

## RESTFul API 가 무엇인가요?

<br />
<hr>

#### __reference__

- https://www.youtube.com/watch?v=RP_f5dMoHFc
- https://learn.microsoft.com/ko-kr/azure/architecture/best-practices/api-design
- https://restfulapi.net/