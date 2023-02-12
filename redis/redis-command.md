# __Redis의 기본 자료구조 및 사용법__

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

이중에서 자주 사용되는 `Strings, Lists, Sets, Hashes, SortedSets` 에 대해 정리했습니다.

<br>

## __Strings__
- 일반적인 문자열 형태이다
- 카운터 구현(incr, decr), 비트 연산을 수행하는 기능을 지원한다.
- Redis 문자열의 최대 길이는 512MB이다

<br>

> [SET](#SET)  [SETNX](#SETNX)  [SETEX](#SETEX)  [MSET](#MSET)  
> [GET](#GET)  [MGET](#MGET)  
> [DEL](#DEL)  
> [INCR](#INCR)  [INCRBY](#INCRBY)  [DECR](#DECR)  [DECRBY](#DECRBY)  

### __SET__
값을 저장한다.  
`SET {key} {value}`

### __GET__ 
값을 조회한다.  
`GET {key}`

```shell
> SET user devoong2
OK
> GET user
"devoong2"
```

<hr><br>

### __DEL__
키를 삭제한다  
`DEL {key}`

```shell
> DEL user
(integer) 1
> KEYS *
(empty array)
```

<hr><br>

### __SETNX__
키가 존재하지 않는 경우에만 저장한다 (SET if Not eXists)  
`SETNX {key} {value}`

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

<hr><br>

### __SETEX__
지정한 시간이 지나면 삭제되는 키를 저장 (시간 단위는 s)  
`SETEX {key} {value} {s}`

```shell
> SETEX user devoong2 30
OK
> TTL user (TTL -> 남은 시간을 알려주는 명령어)
(integer) 30
```

조회시마다 현재 남은 expire 시간을 알려준다

<hr><br>

### __MSET__
여러개의 데이터를 한번에 저장  
`MSET {key1} {value1} {key2} {value2} ...`

```shell
> MSET key1 Hello key2 Hello2
OK
> GET key1
"Hello"
> GET key2
"Hello2"
```

### __MGET__
여러개의 데이터를 한번에 조회  
`MGET {key1} {key2}`

```shell
> MGET key1 key2
1) "hello"
2) "hello2
```

<hr><br>

### __INCR__
숫자를 1씩 증가, redis에 값이 없을시에는 0부터 증가  
`incr {key}`

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

<hr><br>

### __INCRBY__
지정한 숫자(정수)만큼 증가, redis에 값이 없을경우 0을 기준으로 증가  
`INCRBY {key1} {n}`

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

<hr><br>

### __DECR__
숫자를 1씩 감소, redis에 값이 없을경우 0부터 감소  
`DECR {key}`

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

<hr><br>

### __DECRBY__
지정한 숫자(정수)만큼 감소, redis에 값이 없을경우 0을 기준으로 감소  
`DECRBY {key} {n}`

```shell
> DECRBY key1 5
(integer) -5
> DECR key1 10
(integer) -15
> GET key1
"-15"
```

<hr><br><br>

## __Sets__
- 중복되지 않는 문자열의 모음(Set 자료구조와 유사)
- 정렬되지 않음
- key와 value가 1:N 관계, Sets에서는 집합이라는 의미로 value를 member라고 부름
- 교집합, 합집합 및 차이와 같은 일반적인 집합 연산을 효율적으로 수행
- Redis 집합의 최대 크기는 2^32 - 1(4,294,967,295)

<br>

### __SADD__
집합에 데이터를 추가  
`SADD {key} {member}`

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

### __SREM__
집합의 멤버를 삭제  
`SREM {key} {member} [{member} ...]`

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

### __SMEMBERS__
집합의 모든 데이터를 조회  
`SMEMBERS {key}`

```shell
> SADD key1 hello hello2 hello3
(integer) 3
> SMEMBERS key1
1) "hello3"
2) "hello2"
3) "hello"
```

<br>

### __SCARD__
집합에 속한 member의 개수를 조회(SLEN을 사용해도 동일)  
`SCARD {key}`

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

### __SUNION__
집합들의 합집합을 구한다  
`SUNION {key} [{key} ...]`

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

### __SUNIONSTORE__
합집합을 구해 새로운 키에 저장  
`SUNIONSTORE {destination_key} {source_key} [{source_key} ...]`

source_key의 합의 결과가 destination_key에 저장된다

<br>

### __SINTER__
집합들의 교집합을 구한다  
`SINTER {key} [{key} ...]`

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

### __SINTERSTORE__
교집합을 구해 새로운 키에 저장  
`SINTERSTORE {destination_key} {source_key} [{source_key} ...]`

source_key의 교집합의 결과가 destination_key에 저장된다

<br>

### __SDIFF__
집합들의 차집합을 구한다  
`SDIFF {key} [{key} ...]`

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

### __SDIFFSTORE__
차집합을 구해 새로운 키에 저장  
`SDIFFSTORE {destination_key} {source_key} [{source_key} ...]`

source_key의 차집합의 결과가 destination_key에 저장된다

<br>

### __SISMEMBER__
집합에 member가 존재하는지 확인(member가 있으면 1 없으면 0을 리턴)  
`SISMEMBER {key} {member}`  

```shell
> SADD key1 A B C
(integer) 3

> SISMEMBER key1 A
(integer) 1
> SISMEMBER key1 D
(integer) 0
```

<br><br>

# __SortedSets__
- key하나에 여러개의 score와 value로 구성
- value는 score로 정렬되며 중복되지 않음
- score가 같은 경우 value로 정렬됨
- SortedSets에서는 집합이라는 의미로 value를 member라고 부름

<br>

### __ZADD__
데이터를 score와 함께 추가  
`ZADD {key} {score} {member} [{score} {member} ...]`  

```shell
> ZADD key 1 apple 2 banana 3 grape
(integer) 3

> ZRANGE key 0 -1
1) "apple"
2) "banana"
3) "grape"
```

<br>

### __ZREM__
member 삭제  
`ZREM {key} {member} [{member} ...]`  

```shell
> ZADD key 1 apple 2 banana 3 grape
(integer) 3

> ZREM key apple
"12"
> ZRANGE key 0 -1 withscores
1) "banana"
2) "2"
3) "grape"
4) "3"
```

<br>

### __ZRANGE__
member 목록을 조회 (score가 작은것부터 조회)  
`ZRANGE {key} {start} {stop}`  

- start, stop 은 index
- index가 작은것부터 0, 1, 2 ... 로 정해짐
- 전체조회는 start 0, stop -1 을 지정

```shell
> ZADD key 1 apple 2 banana 3 grape
(integer) 3

> ZRANGE key 0 -1
1) "apple"
2) "banana"
3) "grape"
-------------------------------------------------------
> ZRANGE key 0 -1 withscores  // withscores 옵션을 사용하면 score도 같이 보여줌
1) "apple"
2) "1"
3) "banana"
4) "2"
5) "grape"
6) "3"
```

### __ZREVRANGE__
member 목록을 score가 큰것부터 조회(index가 큰것부터 0, 1, 2 ... 로 정해짐)  
SINTAX - `ZRANGE {key} {start} {stop}`  

```shell
> ZADD key 1 apple 2 banana 3 grape
(integer) 3

> ZREVRANGE key 0 -1
1) "grape"
2) "3"
3) "banana"
4) "2"
5) "apple"
6) "1"
```

<br>

### __ZCARD__
key의 member 개수를 리턴  
`ZCARD {key}`  

```shell
> ZADD key 1 apple 2 banana 3 grape
(integer) 3

> ZCARD key
(integer) 3
```

<br>

### __ZINCRBY__
스코어를 increment만큼 증가 혹은 감소 (실수도 가능)  
`ZINCRBY {key} {increment} {member}`  

```shell
> ZADD key 1 apple 2 banana 3 grape
(integer) 3

> ZINCRBY key 10 banana
"12"
> ZRANGE key 0 -1 withscores
1) "apple"
2) "1"
3) "grape"
4) "3"
5) "banana"
6) "12"
```

<br>

### __ZSCORE__
member의 score를 조회  
`ZSCORE {key} {score}`  

- min, max에 지정한 숫자를 포함해 카운팅
- min에 -inf, max에 +inf 를 지정하면 전체를 조회

```shell
> ZADD key 1 apple 2 banana 3 grape
(integer) 3

> ZSCORE key grape
"3"
```

<br>

### __ZCOUNT__
score로 범위를 지정해서 카운트  
`ZCOUNT {key} {min} {max}`  

- min, max에 지정한 숫자를 포함해 카운팅
- min에 -inf, max에 +inf 를 지정하면 전체를 조회

```shell
> ZADD key 1 apple 2 banana 3 grape
(integer) 3

> ZCOUNT key -inf +inf  // 모든 member의 수
(integer) 3
> ZCOUNT key 2 +inf  // score가 2이상인 member의 수
(integer) 2
> ZCOUNT key -inf 1  // score가 1이하인 member의 수
(integer) 1
> ZCOUNT key 1 2  // score가 1이상 2이하인 member의 수
(integer) 2
```

<br>

### __ZPOPMAX__
집합의 큰 값부터 꺼내옴  
`ZPOPMAX {key} {count}`  

```shell
> ZADD key 1 apple 2 banana 3 grape
(integer) 3

> ZPOPMAX key 1
1) "grape"
2) "3"
> ZPOPMAX key 2
1) "banana"
2) "2"
3) "apple"
4) "1"
```

<br>

### __ZPOPMIN__
집합의 작은 값부터 꺼내옴  
`ZPOPMAX {key} {count}`  

```shell
> ZADD key 1 apple 2 banana 3 grape
(integer) 3

> ZPOPMAX key 1
1) "apple"
2) "1"
> ZPOPMAX key 2
1) "banana"
2) "2"
3) "grape"
4) "3"
```

<br><br>

# __Hashes__
- field-value 구조로 구성된 컬렉션 (java의 hashtable과 유사함)
- key하위에 subkey를 이용해 Hash Table을 제공
- 대부분의 해시 명령의 복잡도는 O(1), HKEYS, KVALS, HGETALL는 O(n)이며 여기서 n은 레코드의 개수 

redis hashes 는 다음과 같은 형태를 가지게 됨  
![redis-image](https://user-images.githubusercontent.com/28802545/217829213-af3cb736-031f-48bb-8876-329321c88860.png)

<br>

### __HSET__
key에 filed와 value를 저장  
`HSET {key} {field} {value} [{field} {value} ...]`

```shell
> HSET person name devoong2
(integer) 1
```

<br>

### __HDEL__
key의 filed로 value를 삭제  
`HDEL {key} {field} [{field} ...]`

```shell
> HSET person name devoong2
(integer) 1

> HDEL person name
(integer) 1
```

<br>

### __HGETALL__
key의 모든 field와 value를 조회  
`HGETALL {key}`

```shell
> HSET person name devoong2
(integer) 1
> HSET person gender male
(integer) 1

> HGETALL person
1) "name"
2) "devoong"
3) "gender"
4) "male"
```

<br>

### __HGET__
key와 filed로 value를 조회  
`HGET {key} {field}`

```shell
> HSET person name devoong2
(integer) 1

> HGET person name
"devoong2"
```

### __HMGET__
key와 filed로 여러개의 value를 조회  
`HGET {key} [{key} ...]`

```shell
> HSET person name devoong2
(integer) 1
> HSET person gender male
(integer) 1

> HMGET person name gender
1) "devoong"
2) "male"
```

<br>

### __HLEN__
key의 filed 개수를 조회  
`HLEN {key} {field}`

```shell
> HSET person name devoong2
(integer) 1
> HSET person gender male
(integer) 1

> HLEN person
(integer) 2
```

<br>

### __HKEYS__
key의 모든 filed를 조회  
`HKEYS {key}`

```shell
> HSET person name devoong2
(integer) 1
> HSET person gender male
(integer) 1

> HKEYS person
1) "name"
2) "gender"
```

### __HVALS__
key의 모든 value를 조회  
`HVALS {key}`

```shell
> HSET person name devoong2
(integer) 1
> HSET person gender male
(integer) 1

> HVALS person
1) "devoong"
2) "male"
```

<br>

### __HINCRBY__
field의 value를 increment만큼 증가or감소  
`HINCRBY {key} {field} {increment}`

```shell
> HINCRBY key count 1
"1"
> HINCRBY key count 5
"6"
```

<br>

### __HINCRBYFLOAT__
field의 value를 float만큼 증가or감소  
`HINCRBY {key} {field} {float}`

```shell
> HINCRBYFLOAT key count 1.1
"1.1"
> HINCRBYFLOAT key count 5.5
"6.6"
```

<br>

### __HEXISTS__
key의 field가 있는지 조회 (있으면 1, 없으면 0을 리턴)  
`HEXISTS {key} {field}`

```shell
> HEXISTS key name
(integer) 0

> HSET key name devoong2
(integer) 0
> HEXISTS key name
(integer) 1
```

<br><br>

# __Lists__
- Array 형태의 자료구조
- value는 입력한 순서대로 저장
- Stack, Queue 구현 가능
- key와 value가 1:N 관계
- Head, Tail에서의 작업은 O(1), 요소 중간에서의 작업은 일반적으로 O(n)의 복잡도를 가짐

<br>

### __LPUSH__
리스트의 왼쪽에 데이터를 저장  
`LPUSH {key} {element} [{element} ...]`

```shell
> LPUSH key el1 el2 el3
(integer) 3

> LRANGE key 0 -1
1) "el3"
2) "el2"
3) "el1"
```

### __RPUSH__
리스트의 오른쪽에 데이터를 저장  
`RPUSH {key} {element} [{element} ...]`

```shell
> RPUSH key el1 el2 el3
(integer) 3

> LRANGE key 0 -1
1) "el1"
2) "el2"
3) "el3"
```

<br>

### __LPOP__
리스트의 왼쪽에서 데이터를 꺼냄 (꺼내진 데이터는 리스트에서 삭제)  
`LPOP {key} {count}`

```shell
> LPUSH key el1 el2 el3
(integer) 3

> LPOP key
"el3"

> LRANGE key 0 -1
1) "el2"
2) "el1"
```

---------------------------------------------------------------------

```shell
> LPUSH key el1 el2 el3
(integer) 3

> LPOP key 2
1) "el3"
2) "el2"

> LRANGE key 0 -1
1) "el1"
```

<br>

### __RPOP__
리스트의 오른쪽에서 데이터를 꺼냄 (꺼내진 데이터는 리스트에서 삭제)  
`RPOP {key} {count}`

```shell
> LPUSH key el1 el2 el3
(integer) 3

> RPOP key
"el1"

> LRANGE key 0 -1
1) "el3"
2) "el2"
```

---------------------------------------------------------------------

```shell
> LPUSH key el1 el2 el3
(integer) 3

> RPOP key 2
1) "el1"
2) "el2"

> LRANGE key 0 1
1) "el3"
```

<br>

### __LRANGE__
인덱스의 범위로 리스트를 조회  
- 왼쪽부터 오른쪽으로 0, 1, 2... 의 index를 사용
- 전체를 조회하려면 start: 0, stop: -1 을 지정해야함

`RPUSH {key} {start} {stop}`

```shell
> LPUSH key el1 el2 el3
(integer) 3

> LRANGE key 0 -1
1) "el3"
2) "el2"
3) "el1"
```

__오른쪽 기준으로 조회시에는 음수를 사용__
- 오른쪽부터 왼쪽으로 -1, -2, -3 ... 의 index를 사용

```shell
> LPUSH key el1 el2 el3
(integer) 3


> LRANGE key -3 -2
1) "el3"
2) "el2"
```

<br>

### __LLEN__
리스트에서 value의 개수를 조회  
`LLEN {key}`

```shell
> LPUSH key el1 el2 el3
(integer) 3

> LLEN key
(integer) 3
```

<br>

### __LINDEX__
인덱스로 특정 위치의 데이터를 조회  
`LINDEX {key} {index}`

```shell
> LPUSH key el1 el2 el3
(integer) 3

> LINDEX key 0
"el3"
> LINDEX key 1
"el2"
```

<br>

### __LINDEX__
인덱스로 특정 위치의 데이터를 조회  
`LINDEX {key} {index}`

```shell
> LPUSH key el1 el2 el3
(integer) 3

> LINDEX key 0
"el3"
> LINDEX key 1
"el2"
```

<br>

### __LSET__
인덱스로 특정 위치의 데이터를 변경  
`LSET {key} {index} {value}`

```shell
> LPUSH key el1 el2 el3
(integer) 3

> LSET key 0 value1
OK
> LINDEX key 1
"value1"
```

<br>

### __LREM__
리스트의 데이터를 삭제
- count가 양수인 경우 : 지정한 value를 왼쪽에서부터 count만큼 삭제
- count가 음수인 경우 : 지정한 value를 오른쪽에서부터 count만큼 삭제
- count가 0인 경우 : 지정한 value를 모두 삭제

`LREM {key} {count} {value}`

```shell
> LPUSH key el1 el2 el2 el3
(integer) 4

> LREM key 1 el2
integer (1)
> LRANGE key 0 -1
1) "el3"
2) "el2"
3) "el1"
```

<br>

### __RPOPLPUSH__
오른쪽에서 데이터를 꺼내 왼쪽에 넣음  
`RPOPLPUSH {source_key} {destination_key}`

```shell
> LPUSH source el1 el2  el3
(integer) 3
> LPUSH destination value1 value2 value3
(integer) 3

> RPOPLPUSH source destination
"el1"
> LRANGE destination 0 -1
1) "el1"
2) "value3"
3) "value2"
4) "value1"
```
