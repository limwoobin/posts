# __Redis의 자료구조 및 CLI 사용법에 대해 알아보자__

Redis는 key/value 형태로 저장되는 database이며 다음과 같은 자료구조를 지원합니다.

- __Strings__
- __Lists__
- __Sets__\
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


### __Strings__


<br>

## __Redis 접속__

Redis는 다음과 같은 명령어를 통해 cli에 접속할 수 있습니다.

> redis-cli -h {host} -p {port}

ex)

```
redis-cli -h {host} -p {port}
```


## Redis 데이터 저장