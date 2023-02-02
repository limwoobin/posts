# __Redis의 자료구조 및 CLI 사용법에 대해 알아보자__

Redis는 key/value 형태로 저장되는 database이며 다음과 같은 자료구조를 지원합니다.

- __Strings__
- __Lists__
- __Sets__
- __Hashes__
- __SortedSets__
- __Streams__
- __Geospatial indexes__
- __Bitmaps__
- __Bitfields__
- __HyperLogLog__

이중에서 자주 사용되는 `Strings, Lists, Sets, Hashes, SortedSets` 에 대해 한번 알아보겠습니다.

<br>

<!-- - __Strings__ : 텍스트, 직렬화된 객체, 이진 배열을 포함 및 바이트 시퀀스를 저장.
- __Lists__ : 문자열 값의 연결된 목록, 스택과 큐를 구현
- __Sets__ : 데이터의 모음, 정렬되지 않고 중복되지 않음
- __Hashes__ : field/value 형태의 컬렉션으로 구성됨,  
redis key 하나에 여러개의 field/value 가 저장됨
- __SortedSets__ : set에 score라는 필드가 추가된 데이터 유형
- __Streams__ : ㄹ
- __Geospatial indexes__
- __Bitmaps__
- __Bitfields__
- __HyperLogLog__ -->


## __Strings__
- 일반적인 문자열 형태이다
- 카운터 구현, 비트 연산을 수행하는 기능을 지원한다.
- Redis 문자열의 최대 길이는 512MB이다

<br>

## __Strings의 명렁어__

__SET__ : 값을 저장한다.  
ex) `SET {key} {value}`

__GET__ : 값을 조회한다.  
ex) `GET {key}`

```shell
> SET user devoong2
OK
> GET user
"devoong2"
```

<br>

__DEL__ : 키를 삭제한다  
ex) `DEL {key}`

```shell
> DEL user
(integer) 1
> KEYS *
(empty array)
```

<br>

__SETNX__ : 키가 존재하지 않는 경우에만 저장한다 (SET if Not eXists)  
ex) `SETNX {key} {value}`

```shell
> SETNX user devoong2
OK
> GET user
"devoong2"
> SETNX user woobeen
false
> GET user
"devoong2"
```

<br>

__SETEX__ : 지정한 시간이 지나면 삭제되는 키를 저장 (시간 단위는 s)  
ex) `SETEX {key} {value} {s}`

```shell
> SETEX user devoong2 30
OK
> TTL user (TTL -> 남은 시간을 알려주는 명령어)
(integer) 30
```

조회시마다 현재 남은 expire 시간을 알려준다

<br>

__MSET__ : 여러개의 데이터를 한번에 저장  
ex) `MSET {key1} {value1} {key2} {value2} ...`

```shell
> MSET key1 Hello key2 Hello2
OK
> GET key1
"Hello"
> GET key2
"Hello2"
```

__MGET__ : 여러개의 데이터를 한번에 조회  
ex) `MGET {key1} {key2}`

```shell
> MGET key1 key2
1) "hello"
2) "hello2
```

<br>

__INCR__ : 숫자를 1씩 증가, redis에 값이 없을시에는 0부터 증가  
ex) `incr {key}`

```shell
> INCR key1
(integer) 1
> INCR key1
(integer) 2
> GET key1
"2"

> SET key1 10
OK
> INCR key1
(integer) 11
> GET key1
"11"
```

### 주의사항
- 문자열의 경우에는 에러가 발생함
- redis 정수의 범위를 벗어나는 경우에도 에외 발생
-9,223,372,036,854,775,808(2^63) ~ 9,223,372,036,854,775,807(2^63-1) 

<br>

__INCRBY__ : 지정한 숫자(정수)만큼 증가, redis에 값이 없을경우 0을 기준으로 증가  
ex) `INCRBY {key1} {n}`

```shell
> INCRBY key1 5
(integer) 5
> INCRBY key1 10
(integer) 15
> GET key1
"15"

> INCRBY key1 -5
(integer) -5
> INCRBY key1 -10
(integer) -15
> GET key1
"-15"
```
#### 소수점이 있는 경우엔 에러가 발생

<br>

__DECR__ : 숫자를 1씩 감소, redis에 값이 없을경우 0부터 감소  
ex) `DECR {key}`

```shell
> DECR key1
(integer) -1
> DECR key1
(integer) -2
> GET key1
"-2"

> SET key1 10
OK
> DECR key1
(integer) 9
> DECR key1
(integer) 8
> GET key1
"8"
```

<br>

__DECRBY__ : 지정한 숫자(정수)만큼 감소, redis에 값이 없을경우 0을 기준으로 감소  
ex) `DECRBY {key} {n}`

```shell
> DECRBY key1 5
(integer) -5
> DECR key1 10
(integer) -15
> GET key1
"-15"
```

<br><br>

## __Redis 접속__

Redis는 다음과 같은 명령어를 통해 cli에 접속할 수 있습니다.

> redis-cli -h {host} -p {port}

ex)

```
redis-cli -h {host} -p {port}
```
