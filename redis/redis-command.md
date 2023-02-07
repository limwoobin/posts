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

## __Strings__
- 일반적인 문자열 형태이다
- 카운터 구현(incr, decr), 비트 연산을 수행하는 기능을 지원한다.
- Redis 문자열의 최대 길이는 512MB이다

<br>

### __SET__ : 값을 저장한다.  
SYNTAX - `SET {key} {value}`

### __GET__ : 값을 조회한다.  
SYNTAX - `GET {key}`

```shell
> SET user devoong2
OK
> GET user
"devoong2"
```

<br>

### __DEL__ : 키를 삭제한다  
SYNTAX - `DEL {key}`

```shell
> DEL user
(integer) 1
> KEYS *
(empty array)
```

<br>

### __SETNX__ : 키가 존재하지 않는 경우에만 저장한다 (SET if Not eXists)  
SYNTAX - `SETNX {key} {value}`

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

### __SETEX__ : 지정한 시간이 지나면 삭제되는 키를 저장 (시간 단위는 s)  
SYNTAX - `SETEX {key} {value} {s}`

```shell
> SETEX user devoong2 30
OK
> TTL user (TTL -> 남은 시간을 알려주는 명령어)
(integer) 30
```

조회시마다 현재 남은 expire 시간을 알려준다

<br>

### __MSET__ : 여러개의 데이터를 한번에 저장  
SYNTAX - `MSET {key1} {value1} {key2} {value2} ...`

```shell
> MSET key1 Hello key2 Hello2
OK
> GET key1
"Hello"
> GET key2
"Hello2"
```

### __MGET__ : 여러개의 데이터를 한번에 조회  
SYNTAX - `MGET {key1} {key2}`

```shell
> MGET key1 key2
1) "hello"
2) "hello2
```

<br>

### __INCR__ : 숫자를 1씩 증가, redis에 값이 없을시에는 0부터 증가  
SYNTAX - `incr {key}`

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

### __INCRBY__ : 지정한 숫자(정수)만큼 증가, redis에 값이 없을경우 0을 기준으로 증가  
SYNTAX - `INCRBY {key1} {n}`

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

### __DECR__ : 숫자를 1씩 감소, redis에 값이 없을경우 0부터 감소  
SYNTAX - `DECR {key}`

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

### __DECRBY__ : 지정한 숫자(정수)만큼 감소, redis에 값이 없을경우 0을 기준으로 감소  
SYNTAX - `DECRBY {key} {n}`

```shell
> DECRBY key1 5
(integer) -5
> DECR key1 10
(integer) -15
> GET key1
"-15"
```

<br><br>

## __Sets__
- 중복되지 않는 문자열의 모음(Set 자료구조와 유사)
- 정렬되지 않음
- key와 value가 1:N 관계, Sets에서는 집합이라는 의미로 value를 member라고 부름
- 교집합, 합집합 및 차이와 같은 일반적인 집합 연산을 효율적으로 수행
- Redis 집합의 최대 크기는 2^32 - 1(4,294,967,295)

<br>

### __SADD__ : 집합에 데이터를 추가  
SYNTAX - `SADD {key} {member}`

```shell
> SADD key1 hello
(integer) 1
> SADD key1 hello2 hello3
(integer) 2
> SMEMBERS key1 (멤버 조회)
1) "hello3"
2) "hello2"
3) "hello"

------------------------------------
중복된 값을 저장하는 경우
> SADD key1 hello
(integer) 1
> SADD key1 hello
(integer) 0
> SMEMBERS key1
1) "hello"
```

__Subquery 사용 가능__
```shell
example
> SADD key1 (get key2)
> SADD key1 (lrange mylist2 0 -1)
...
```

<br>

### __SREM__ : 집합의 멤버를 삭제  
SYNTAX - `SREM {key} {member} [{member} ...]`

```shell
> SADD key1 hello hello2 hello3
(integer) 3
> SREM key1 hello
(integer) 1
> SMEMBERS key1
1) "hello3"
2) "hello2"

> SREM key1 hello2 hello3
(integer) 2
> SMEMBERS key1
(empty array)
```

<br>

### __SMEMBERS__ : 집합의 모든 데이터를 조회  
SYNTAX - `SMEMBERS {key}`

```shell
> SADD key1 hello hello2 hello3
(integer) 3
> SMEMBERS key1
1) "hello3"
2) "hello2"
3) "hello"
```

<br>

### __SCARD__ : 집합에 속한 member의 개수를 조회(SLEN을 사용해도 동일)  
SYNTAX - `SCARD {key}`

```shell
> SADD key1 hello hello2 hello3
(integer 3)
> SCARD key1
(integer 3)
> SREM key1 hello
(integer 1)
> SCARD key1
(integer 2)
```

<br>

### __SUNION__ : 집합들의 합집합을 구한다  
SYNTAX - `SUNION {key} [{key} ...]`

```shell
> SADD key1 hello hello2
(integer) 2
> SADD key2 hello3
(integer) 1

// key1 = {hello, hello2}
// key2 = {hello3}

> SUNION key1 key2
1) "hello3"
2) "hello"
3) "hello2"
```

### __SUNIONSTORE__ : 합집합을 구해 새로운 키에 저장
SINTAX - `SUNIONSTORE {destination_key} {source_key} [{source_key} ...]`

source_key의 합의 결과가 destination_key에 저장된다

<br>

### __SINTER__ : 집합들의 교집합을 구한다
SYNTAX - `SINTER {key} [{key} ...]`

```shell
교집합이 있는 경우
> SADD key1 hello hello2
(integer) 2
> SADD key2 hello
(integer) 1

// key1 = {hello, hello2}
// key2 = {hello}

> SINTER key1 key2
1) "hello"

-------------------------------------------

교집합이 없는 경우
> SADD key1 hello hello2
(integer) 2
> SADD key2 hello3
(integer) 1

// key1 = {hello, hello2}
// key2 = {hello3}

> SINTER key1 key2
(empty array)
```

### __SINTERSTORE__ : 교집합을 구해 새로운 키에 저장
SINTAX - `SINTERSTORE {destination_key} {source_key} [{source_key} ...]`

source_key의 교집합의 결과가 destination_key에 저장된다

<br>

### __SDIFF__ : 집합들의 차집합을 구한다
SYNTAX - `SDIFF {key} [{key} ...]`

```shell
> SADD key1 A B C
(integer) 3
> SADD key2 C D
(integer) 2

// key1 = {A, B, C}
// key2 = {C, D}

> SDIFF key1 key2
1) "A"
2) "B"
```

### __SDIFFSTORE__ : 차집합을 구해 새로운 키에 저장
SINTAX - `SDIFFSTORE {destination_key} {source_key} [{source_key} ...]`

source_key의 차집합의 결과가 destination_key에 저장된다

<br>

### __SISMEMBER__ : 집합에 member가 존재하는지 확인
SINTAX - `SISMEMBER {key} {member}`  
member가 있으면 1 없으면 0을 리턴

```shell
> SADD key1 A B C
(integer) 3

> SISMEMBER key1 A
(integer) 1
> SISMEMBER key1 D
(integer) 0
```

<br><br>

## __Redis 접속__

Redis는 다음과 같은 명령어를 통해 cli에 접속할 수 있습니다.

SYNTAX
> redis-cli -h {host} -p {port}

