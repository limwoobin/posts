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

### __Strings의 명렁어__

__SET__ - 값을 저장한다.  
ex) `SET {key} {value}`

__GET__ - 값을 조회한다.  
ex) `GET {key}`

```shell
> SET user devoong2
OK
> GET user
"devoong2"
```

<br>

__SETNX__ - 키가 존재하지 않는 경우에만 저장한다 (SET if Not eXists)  
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

__SETEX__ - 지정한 시간이 지나면 삭제되는 키를 저장 (시간 단위는 s)
ex) `SETEX {key} {valul} {s}`

```shell
> SETEX user devoong2 30
OK
> TTL user (TTL -> 남은 시간을 알려주는 명령어)
(integer) 30
```

조회시마다 현재 남은 expire 시간을 알려준다

<br>

__DEL__ - 키를 삭제한다
ex) `DEL {key}`

```shell
> DEL user
(integer) 1
> KEYS *
(empty array)
```



## __Redis 접속__

Redis는 다음과 같은 명령어를 통해 cli에 접속할 수 있습니다.

> redis-cli -h {host} -p {port}

ex)

```
redis-cli -h {host} -p {port}
```


## Redis 데이터 저장