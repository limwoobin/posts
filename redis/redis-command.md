# __Redis의 자료구조 및 CLI 사용법에 대해 알아보자__

Redis는 key/value 형태로 저장되는 database이며 다음과 같은 자료구조를 지원합니다.

- Strings
- Hashes
- Lists
- Sets
- SortedSets

<br>

## __Redis 접속__

Redis는 다음과 같은 명령어를 통해 cli에 접속할 수 있습니다.

> redis-cli -h {host} -p {port}

ex)

```
redis-cli -h {host} -p {port}
```


## Redis 데이터 저장