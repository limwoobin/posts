# __Rest API, Rest, Restful API 란__ ?

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
- HTTP 의 Content

<br />

## REST API 가 무엇인가요?

### REST API 의 개념

### REST API 의 특징

### REST API 의 장단점

## RESTFul API 가 무엇인가요?

<hr>

#### __reference__

- https://www.youtube.com/watch?v=RP_f5dMoHFc
- https://learn.microsoft.com/ko-kr/azure/architecture/best-practices/api-design
- https://restfulapi.net/