![telnet-logo](https://private-user-images.githubusercontent.com/28802545/336922376-0c1236cd-56df-4fb9-845d-0f8fd8ea520a.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTc2MDQ1MDksIm5iZiI6MTcxNzYwNDIwOSwicGF0aCI6Ii8yODgwMjU0NS8zMzY5MjIzNzYtMGMxMjM2Y2QtNTZkZi00ZmI5LTg0NWQtMGY4ZmQ4ZWE1MjBhLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA2MDUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNjA1VDE2MTY0OVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTJhMzNmZWU3YmEwODUxMjYwNTY3MTg0Y2RlMmViOWY0MGUzZjUxZTc0ZDI2NjBiZGJiM2FjZmY2MWU3YTE4ZjYmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.A_49rCEPcfxb5b2bA6dhGhB9FrdAX6YkxDGFsvj54-g)

# Telnet 간단 사용법

안녕하세요 이번엔 __`Telnet`__ 에 대해 간단한 사용법을 알아보려합니다.

개발을 진행하다보면 내부망에서 외부 서비스와 연동을 한다던지 AWS 방화벽 및 보안그룹 설정 등을 통해 타 서비스와 연동이 필요한 경우가 간혹 존재하는데요.  

이때, 로컬PC 혹은 서버에서 연동할 서비스와 연결이 가능한 상태인지 (통신이 되는 상태인지) 확인하는 방법으로 __`Telnet`__ 을 이용하면 확인할 수 있습니다.

<br />

# __`Telnet`__ 이란?

__`Telnet`__ 은 원격지의 컴퓨터에 접속할 때에 지원되는 인터넷 표준 프로토콜입니다.

__`Telnet`__ 은 다음과 같은 특징을 가지고 있습니다.
- TCP/IP 기반의 프로토콜
- 23번 Port 를 기본포트로 사용
- 원격 터미널 접속 서비스

<br />

그럼 __`Telnet`__ 을 이용해 원격지 서비스와 통신이 가능한 상태인지 확인하는 방법을 알아보겠습니다.  
예시는 모두 MAC 기준으로 진행하겠습니다.

<br />

# __`Telnet`__ 사용법

## __`Telnet`__ 설치

우선 __`Telnet`__ 을 먼저 PC 에 설치하도록 하겠습니다.

> brew install telnet

다음 명령어를 이용해 우선 __`Telnet`__ 을 설치합니다.

![telnet-image-1](
https://private-user-images.githubusercontent.com/28802545/336908439-4608b567-d56d-4cce-9ecf-320e8a4f43de.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTc2MDIwNDQsIm5iZiI6MTcxNzYwMTc0NCwicGF0aCI6Ii8yODgwMjU0NS8zMzY5MDg0MzktNDYwOGI1NjctZDU2ZC00Y2NlLTllY2YtMzIwZThhNGY0M2RlLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA2MDUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNjA1VDE1MzU0NFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTg5YWVmNTFkM2RiMmI0MzRmYTRlMjE2YTNmMWRjY2U4NGJmMjMzNDljMTNhODVjODVkYzZhYzRjNTY1OTNmNjYmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.TA5j6jLdHAe6i55R4ciqHREwc3bp0NAW1mbu7pzNphk)

telnet 입력 시 다음과 같이 설치된것을 확인할 수 있습니다.

## __`Telnet`__ 접속하기

> __telnet [IP or Domain] [port]__

텔넷으로 서비스에 접근이 가능한지 확인할때는 위와 같은 커맨드로 실행합니다.  
그럼 접근에 가능한 경우를 먼저 보겠습니다.


### __접근에 성공하는 경우__

```shell
Trying [ip]...
Connected to [ip or domain]
Escape character is '^]'.
```

telnet 실행시 접근이 가능한 경우 나오는 메시지입니다.  
네이버에 한번 접근가능한지 확인해보겠습니다.


![telnet-image-2](
https://private-user-images.githubusercontent.com/28802545/336909415-4b945295-ffa5-44e4-b762-3e8bd2cdaa1c.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MTc2MDIyMTEsIm5iZiI6MTcxNzYwMTkxMSwicGF0aCI6Ii8yODgwMjU0NS8zMzY5MDk0MTUtNGI5NDUyOTUtZmZhNS00NGU0LWI3NjItM2U4YmQyY2RhYTFjLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA2MDUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwNjA1VDE1MzgzMVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWVjZGU4YTdmZmM2MDZjMTI5ODU4YzMwMmI3NDdiNDU5ZjBkNzJmYWYwZDYxMGRhYjZhNDgxY2I1ZDk5MWQzZWImWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.l-IeGn4sacKTDxfii3J45bKyICGXDifXO-2-78qKT4o)

네이버에 접근이 가능한지 확인해보기 위해 domain 을 `naver.com` port 는 HTTPS 의 기본포트인 `443` 을 넣어보았습니다.  

다음과 같이 나오면 접근이 가능하다고 볼 수 있습니다.

<br />

### __접근에 실패하는 경우__

접근에 실패하는 경우는 대표적으로 다음과 같은 경우가 존재합니다.

1. __프로세스는 살아있으나 방화벽이 닫혀있는 경우__

```shell
Trying [ip]...
telnet: connect to address [ip]: Connection timed out
Trying [ip]...
telnet: connect to address [ip]: Connection timed out
Trying [ip]...
telnet: connect to address [ip]: Connection timed out
telnet: Unable to connect to remote host
```

<br />

2. __방화벽은 열려있으나 프로세스가 올라가지 않은 경우__

```shell
Trying [ip]
telnet: connect to address [ip]: Connection refused
telnet: Unable to connect to remote host
```

<br />

3. __프로세스 자체가 잘못된 경우__

```shell
[ip or domain] nodename nor servname provided, or not known
```

<br />

#### __reference__

- https://blog.naver.com/yeopil-yoon/221286937410