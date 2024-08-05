![tomcat-thread-image1](https://private-user-images.githubusercontent.com/28802545/354904925-90ac9575-3926-4a46-bb13-24dbc00197a4.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjI3Nzg1NzEsIm5iZiI6MTcyMjc3ODI3MSwicGF0aCI6Ii8yODgwMjU0NS8zNTQ5MDQ5MjUtOTBhYzk1NzUtMzkyNi00YTQ2LWJiMTMtMjRkYmMwMDE5N2E0LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA4MDQlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwODA0VDEzMzExMVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTY0MWE5YTQ3MWUyZWQ4MzM5NmE0MmQzYWNjZGFjYjZlZGIzZmY4ZWVhZjYxZTFkMTE3NDZkMTk2OWQ4OWM4YjQmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.xEnWHIc5_S1KcPHJrwSnaOFG-oT5zG4oS7z1hXND-kc)

#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/tomcat-thread-pool)

# __Apache Tomcat 설정에 대해 알아보자__

안녕하세요 이번에는 스프링의 WAS 인 __Apache Tomcat__ 의 설정 옵션들  
그리고 Thread 설정에 따라 어떻게 동작하는지 등에 대해 한번 알아보려합니다.

## __Tomcat 의 설정 옵션__

__application.yml__
```yml
server:
  tomcat:
    max-connections: 8192
    accept-count: 100
    threads:
      max: 200
      min-spare: 10
```

다음은 스프링 부트에서 톰캣을 설정에 대한 옵션입니다.  
각 옵션별로 하나씩 알아보겠습니다.

### __threads.max__
생성할수있는 최대 thread 의 갯수이며 실제 Active User 수를 뜻합니다.  
즉, 순간 처리가능한 트랜잭션의 수를 의미합니다.

### __threads.min-spare__
항상 활성화 되어있는(idle) 스레드 갯수입니다.

### __accept-count__
__maxConnections__ 이상의 요청이 들어왔을때 대기하는 대기열(Queue)의 길이입니다.  
큐가 꽉 찼을 때 수신된 모든 요청은 거부됩니다. 

해당 설정은 OS 에서 관리하는 설정이며 무시될 수도 있습니다.

### __max-connections__
서버가 수락하고 처리할 수 있는 최대 연결수를 의미합니다. 이 숫자에 도달하면 서버는 연결을 추가로 수락하지만 처리는 하지 않습니다.  

처리 중인 연결 개수가 __maxConnections__ 아래로 떨어지면 다시 새 연결을 수락하고 처리하기 시작합니다.  
(주의할 점은 이 수는 실제 연결된 Connection 수가 아닌 Socket File Descriptor 의 수 입니다.)

__maxConnections__ 에 연결이 가득 차더라도 운영체제는 __acceptCount__ 만큼 추가 연결을 허용합니다. 

Socket 은 Connection 이 끝난 후 바로 반환되지 않고 Connection 종료 후 FIN 신호, TIME_WAIT 시간을 거쳐서 Socket 의 Connection 을 종료합니다.

그리고 Http 의 KeepAlive 옵션을 사용하게되면 요청을 처리하지 않는 Connection 수도 유지되기에, 요청 처리 수보다 실제 연결되어있는 Connection 수가 높을 수 있습니다.

이러한 이유때문에 __maxConnections__ 은 넉넉하게 설정하는것이 좋습니다.

NIO/NIO2의 경우에는 10000개, APR/native의 경우에는 8192개가 기본값입니다.

![tomcat-thread-image9](https://private-user-images.githubusercontent.com/28802545/354901245-87c72bbf-497b-4b8c-a144-952e1f0ef9f3.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjI3NzgzNzcsIm5iZiI6MTcyMjc3ODA3NywicGF0aCI6Ii8yODgwMjU0NS8zNTQ5MDEyNDUtODdjNzJiYmYtNDk3Yi00YjhjLWExNDQtOTUyZTFmMGVmOWYzLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA4MDQlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwODA0VDEzMjc1N1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWU2OTkyZWUyYmNiYzg3OWE2YTQyYmY4MDYxNjczYjI4YTBkNDFmZjVhOWJmMTdiZjk1MGU1MDQ5YWJkZTllOWEmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.CagXvs4oPBiiTVQ20Is7_uNOgIQyAi3-DLddNg4yfeQ)

정리하자면 __Tomcat__ 의 스레드 풀 매커니즘은 다음과 같습니다.

1) __Tomcat__ 서버가 처음 실행될때 스레드 풀에 __min-spare__ 만큼의 유휴 스레드를 생성합니다.
2) 이후 요청이 유휴 스레드보다 많은 요청이 들어오게 되면 maxConnections 이 증가하고 요청을 처리할 스레드 수가 증가합니다. (최대 maxThreads 만큼 증가)
3) 요청이 많아 maxConnections 가 가득차고 이후 추가 요청이 들어온다면 __acceptCount__ 만큼 추가로 연결을 허용합니다.

Java 의 __ThreadPoolExecutor__ 의 경우 __queueSize__ 가 가득차면 이후 스레드가 증가하는것과는 다른 방식입니다.

<br />

## __Http Connector__

__Tomcat__ 서버는 __HttpConnector__ 를 통해 요청을 수신하고 처리합니다.  
그리고 __Tomcat__ 버전별로 사용하는 __HttpConnector__ 가 상이한데요 아래와 같습니다.

![tomcat-thread-image2](https://private-user-images.githubusercontent.com/28802545/354876315-8f80d4f0-0d04-4f6b-90ba-c7e46950fc39.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjI3NDk5OTMsIm5iZiI6MTcyMjc0OTY5MywicGF0aCI6Ii8yODgwMjU0NS8zNTQ4NzYzMTUtOGY4MGQ0ZjAtMGQwNC00ZjZiLTkwYmEtYzdlNDY5NTBmYzM5LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA4MDQlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwODA0VDA1MzQ1M1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWM2ZTVhN2E2YTllZWQ3Y2MxMjRhM2MxOWRjNjNlY2I5N2RjNDlhNTcxNmVmZjQ2Y2FiOWY3NGU3ZTg0NTQ1NmImWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.58TDn61C9xH6xWdnK4FF1Bv6wvMDNPmpiKmZj0GbtEQ)

### __BIO(Blocking I/O) Connector__

요청이 들어와 TCP 커넥션이 생성되면 하나의 Thread 가 할당됩니다.  
그리고 요청에 대해 응답하기까지 하나의 Thread 에서 처리되며 응답 이후 소켓 연결이 닫히고 나서야 Thread 는 Pool 에 반환됩니다.

즉, 커넥션과 Thread 가 1:1 로 매핑됩니다.
이는 Thread 가 Idle 상태로 있는 시간이 길어져서 낭비되는 시간이 많을 수 있습니다.

만약 요청이 들어왔을때 할당할 수 있는 Idle Thread 가 없다면 요청은 Block 됩니다.

### __NIO(Non Blocking I/O) Connector__

BIO 와 달리 소켓 연결을 담당하는 __Poller__ 라는 Thread 가 존재합니다.  
요청이 들어오면 __Poller__ 는 Socket 들을 캐시로 들고있다가 처리가 가능한 순간에만 Thread 를 할당합니다.

그리고 __NIO Connector__ 는 Thread 는 응답을 보내면 바로 Pool 로 반환됩니다.  
그렇기에 __BIO Connector__ 에 비해 Thread 가 Idle 상태로 있는 시간이 짧기에 효율적입니다.

<br />

## __Tomcat 설정옵션에 따른 테스트__

```
SpringBoot version: 3.3.2
Tomcat version: 10.1.26
```

Tomcat 은 __10.1.26__ 버전이기에 __NIOConnector__ 를 사용합니다.  
(BIO Connector 는 Tomcat 9 버전부터 Deprecated 되었습니다)

모든 테스트는 10개의 스레드를 동시에 요청하는것으로 진행하겠습니다.


__ThreadPoolController.java__
```java
@Slf4j
@RestController
@RequestMapping(value = "/")
public class ThreadPoolController {

  @GetMapping
  public ResponseEntity<String> api() {
    try {
      Thread.sleep(5000);
      log.info("http call");
    } catch (InterruptedException e) {
      throw new RuntimeException(e);
    }

    return new ResponseEntity<>("OK", HttpStatus.OK);
  }
}
```

요청이 시간의 흐름에 따라 어떻게 처리되는지 확인하기 위해 컨트롤러에서 5초간 sleep 을 하도록 하겠습니다.

__ThreadPoolTest.java__
```java
public class ThreadPoolTest {

  private static final HttpClient httpClient = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_1_1)
    .build();

  @Test
  void test() {
    final HttpRequest request = HttpRequest.newBuilder()
      .uri(URI.create("http://127.0.0.1:8080"))
      .build();

    Supplier<HttpResponse<String>> task = () -> {
      try {
        return httpClient.send(request, HttpResponse.BodyHandlers.ofString());
      } catch (Exception e) {
        throw new RuntimeException(e);
      }
    };

    List<CompletableFuture<HttpResponse<String>>> futures = new ArrayList<>();
    for (int i = 0; i < 10; i++) {
      CompletableFuture<HttpResponse<String>> future = CompletableFuture.supplyAsync(task);
      futures.add(future);
    }

    CompletableFuture<Void> combinedFuture = CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]));
    combinedFuture.join();
  }

}

```

### 1. __maxConnections 과 aceeptCount, maxThreads 가 모두 충분한 경우__

```yml
server:
  tomcat:
    max-connections: 10
    accept-count: 10
    threads:
      max: 10
```

![tomcat-thread-image3](https://private-user-images.githubusercontent.com/28802545/354899227-2010c402-d68b-413b-b851-1b42c21a4f67.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjI3NzIyMjcsIm5iZiI6MTcyMjc3MTkyNywicGF0aCI6Ii8yODgwMjU0NS8zNTQ4OTkyMjctMjAxMGM0MDItZDY4Yi00MTNiLWI4NTEtMWI0MmMyMWE0ZjY3LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA4MDQlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwODA0VDExNDUyN1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTZmZDBlZmUxZDEwYmE4NjBkYmQ3ZGNiZGE1MjE0YjcyM2QzZDlmYjUwYzM4ZjRmNTk1M2U1M2NhOTgyMjkyNGYmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.BkSvw_YAWlX53kE0_ePEXGUA8vYrnT-2XaMMr-2Jv3E)

10개의 요청 모두 한번에 처리되었습니다.

### 2. __acceptCount 는 부족하고 __maxConnections 와 maxThreads 가 충분한 경우__

```yml
server:
  tomcat:
    max-connections: 10
    accept-count: 2
    threads:
      max: 10
```

![tomcat-thread-image4](https://private-user-images.githubusercontent.com/28802545/354899797-ff0ea6ff-140f-4bf1-9bb2-62a8760e161f.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjI3NzI5NTIsIm5iZiI6MTcyMjc3MjY1MiwicGF0aCI6Ii8yODgwMjU0NS8zNTQ4OTk3OTctZmYwZWE2ZmYtMTQwZi00YmYxLTliYjItNjJhODc2MGUxNjFmLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA4MDQlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwODA0VDExNTczMlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTZlMWI0Yjc5NDYyYTk4ZTMzMWIyMDNiZDM4ZTU1YmNlZjhjMzk3MWNjYTdiNWI5NmFjZGUxYWYxNmI4NmVmYmUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.znvYRp518QWUf2Nom4ROLEj2SiPMHq8Gr9AjkQuJcww)

__maxConnections__ 의 수가 10개의 요청은 한번에 처리 가능한 수여서 모두 한번에 처리된것을 확인할 수 있습니다.

### 3. __maxConnections 는 부족하고 aceeptCount 와 maxThreads 가 충분한 경우__

```yml
server:
  tomcat:
    max-connections: 2
    accept-count: 10
    threads:
      max: 10
```

![tomcat-thread-image5](https://private-user-images.githubusercontent.com/28802545/354899628-349dc530-ed7f-4f38-8109-98b2906d69e9.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjI3NzI3NjUsIm5iZiI6MTcyMjc3MjQ2NSwicGF0aCI6Ii8yODgwMjU0NS8zNTQ4OTk2MjgtMzQ5ZGM1MzAtZWQ3Zi00ZjM4LTgxMDktOThiMjkwNmQ2OWU5LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA4MDQlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwODA0VDExNTQyNVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWVlZDEzYzUyYWI1ZjBjMWY1NjdlOTlmZWJkNTZjYTg5MjI0M2FiODE0MjdhNmMxYTViZjE3M2YxOWU2NWRhOWImWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.lpqQu7VSrhZVsHuv-O9BwMn1ohhWOSMDxOMks9aWMhk)

65초 간격으로 두개의 요청이 들어온것을 확인할 수 있습니다. 

10개의 요청이 오면 우선 __maxConnections__ 에 두개의 요청이 Queue 로 들어가고 나머지 요청은 __acceptCount__ 에서 대기하게 됩니다. 

그리고 두개가 처리 완료되었다면 __acceptCount__ 에서 __maxConnections__ 로 두개의 요청이 다시 이동해야 합니다.  
그렇다면 5초간격으로 수행되어야 할것같은데 그렇지 않았습니다.

왜 이렇게 동작한걸까요??

__maxConnections__ 은 요청의 Connection 이 끝나더라도 Socket 이 바로 종료되지 않는다고 위에서 이야기 했습니다.

Socket 은 Connection 연결해제 후 종료되기까지 FIN 신호 및 __TIME_WAIT__ 시간을 거쳐서 종료되는데요.  
이때 __TIME_WAIT__ 이 과정이 60초가 소요된 것입니다.

그리고 컨트롤러에서 걸어놓은 sleep 시간은 5초입니다. 그래서 __maxConnections__ 에 대해 Socket 이 종료되고 다음 요청에 대한 로그가 찍히기 까지 65초의 간격을 두고 요청을 처리하게 된겁니다.

#### __WireShark 로 패킷 확인해보기__

이 과정을 좀 더 자세히 보기 위해 __WireShark__ 를 이용해 패킷이 어떻게 요청되고 반환되는지 확인해보겠습니다.

__52546__ port 를 기준으로 따라보겠습니다.

1) __TCP 연결 요청__

![tomcat-thread-image10](https://private-user-images.githubusercontent.com/28802545/354984465-e1226ada-4709-476e-9fd2-a0fe63e0cac2.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjI4Mzc3MDMsIm5iZiI6MTcyMjgzNzQwMywicGF0aCI6Ii8yODgwMjU0NS8zNTQ5ODQ0NjUtZTEyMjZhZGEtNDcwOS00NzZlLTlmZDItYTBmZTYzZTBjYWMyLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA4MDUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwODA1VDA1NTY0M1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTc0NDEwYzBhMWYwMTNhYWQ1ODk3MTllZTQyMzQ0NGQzYTcwNTcyNjE5Mzk0ZjhjODYyODhjY2Q2NzE1YWZkYTUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.OvuNWTBkEJ8qKlQZ8jZd4FFGEBaejTDaDtlZau7eZOQ)

__52546__ Port 가 __8080__ Port 에게 TCP 요청을 하기 위해 __3-Way-Handshake__ 를 진행하는것을 확인할 수 있습니다.  
이때 시간은 4.43초 입니다.

2) __HTTP 요청__

![tomcat-thread-image11](https://private-user-images.githubusercontent.com/28802545/354988078-a2bd99e2-559f-4f3b-8c53-2f496e1d5126.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjI4Mzg2OTAsIm5iZiI6MTcyMjgzODM5MCwicGF0aCI6Ii8yODgwMjU0NS8zNTQ5ODgwNzgtYTJiZDk5ZTItNTU5Zi00ZjNiLThjNTMtMmY0OTZlMWQ1MTI2LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA4MDUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwODA1VDA2MTMxMFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTg1MzdjZDY2NmU5YjQ5YjQ5Nzk2NDg4MDk0NDVlYjY5OGMwMzg0ODMyZGU3YTI5OGZkN2VjMDEyMjIyMWE1ODUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.3EV-KvmsL-T-MqQXvZBwDFVymDRKQwm9IfdkeYq5bZ4)

TCP 연결에 성공했으니 이후 HTTP 요청을 진행합니다. 이후 서버는 요청을 잘 받았다는 신호로 ACK 를 응답합니다.  
이때 시간은 4.44초 입니다.

3) __HTTP 응답 반환 및 TCP 연결 해제 요청__

![tomcat-thread-image12](https://private-user-images.githubusercontent.com/28802545/354983833-57a6aafd-6344-432d-ab39-1a0122b655e9.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjI4MzgwMjUsIm5iZiI6MTcyMjgzNzcyNSwicGF0aCI6Ii8yODgwMjU0NS8zNTQ5ODM4MzMtNTdhNmFhZmQtNjM0NC00MzJkLWFiMzktMWEwMTIyYjY1NWU5LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA4MDUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwODA1VDA2MDIwNVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTRkMWE1MGRmNzAzZGY1OWUxNGY3ODdhYTRmMDIzYjA4MDQ3OWY3OGE2NmZhZjkyYTA3MzQ3ZmJiOTk0ZmQ0YjImWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.iTgET0eqYABrYiXbFL8K0SqVfkupK2p77G2Jxy9YADY)

컨트롤러에서 5초간 Sleep 후에 HTTP 응답을 반환합니다.  
이때 시간은 요청 후 약 5초가 지난 9.46초 입니다.

그리고 마지막 패킷로그를 보시면 응답을 받자마자 TCP 연결 해제를 위해 서버에게 ACK 신호를 전달합니다.

이 과정이 __4-Way-Handshake__ 의 시작입니다.

4) __TCP 연결 해제__

![tomcat-thread-image13](https://private-user-images.githubusercontent.com/28802545/354984182-a801917d-6775-44cc-97ae-2ab09df2d5c1.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjI4MzkxMjUsIm5iZiI6MTcyMjgzODgyNSwicGF0aCI6Ii8yODgwMjU0NS8zNTQ5ODQxODItYTgwMTkxN2QtNjc3NS00NGNjLTk3YWUtMmFiMDlkZjJkNWMxLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA4MDUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwODA1VDA2MjAyNVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWQxZWE2NzczOGE0MWNmNzI0NzcwMjI2ZjY3NTc0NGU1ZDU5OGEyZDViYjQzM2ZiZGM0YjI1ZGY1YmI5ZGQ4YjUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.hSJKxqWj2zXtXvx5wc27xqICUtwoppIAoR4WYdpQfec)

TCP 연결 해제를 위해 __4-Way-Handshake__ 를 진행합니다.  
연결해제 요청을 9.46초에 보냈지만 __FIN, ACK__ 응답이 60초 뒤인 69초에 온것을 볼 수 있습니다.

이때 서버는 자신의 통신이 끝날때까지 기다리는 __TIME_WAIT__ 상태를 거치게 되는데 이 시간이 60초가 걸려서 그렇습니다.

클라이언트는 __FIN,ACK__ 응답을 받고 확인했다는 ACK 를 서버에게 보냅니다.  
그리고 서버는 ACK 신호를 받고 소켓을 종료합니다.

<br />

### 4. __maxConnections, acceptCount 모두 부족하고 maxThreads 는 충분한 경우__

```yml
server:
  tomcat:
    max-connections: 5
    accept-count: 2
    threads:
      max: 10
```

![tomcat-thread-image6](https://private-user-images.githubusercontent.com/28802545/354903985-dc8e6209-6be8-4ec0-8c68-cd92dbd2c891.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjI3Nzc1MTIsIm5iZiI6MTcyMjc3NzIxMiwicGF0aCI6Ii8yODgwMjU0NS8zNTQ5MDM5ODUtZGM4ZTYyMDktNmJlOC00ZWMwLThjNjgtY2Q5MmRiZDJjODkxLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA4MDQlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwODA0VDEzMTMzMlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTY1MGJhYmFlZjQyZDg5MmE1YWNjYTAzY2QzNmExOTUzYTJmMTVlMWFiOTY5NDNiNGIyNzc4MDM0NjYwYTU4NmMmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.oK2xUzmlxKgJCfTgXC13uDqTw-lg9d0iIEQHaQp1H98)

처음 __maxConnections__ 만큼 5개의 요청을 처리하고 이후 __acceptCount__ 만큼의 2개의 요청을 처리한것을 확인할 수 있습니다.

총 요청은 10개이지만 수신할 수 있는 큐의 사이즈는 총 7개로 제한되었으니 7개의 요청만 처리된것입니다.

나머지 요청에 대해서는 __IOException__ 이 발생했습니다.

![tomcat-thread-image7](https://private-user-images.githubusercontent.com/28802545/354904030-e084aaf2-cb48-44a7-9d8f-8b16aae3034e.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjI3NzgwMDMsIm5iZiI6MTcyMjc3NzcwMywicGF0aCI6Ii8yODgwMjU0NS8zNTQ5MDQwMzAtZTA4NGFhZjItY2I0OC00NGE3LTlkOGYtOGIxNmFhZTMwMzRlLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA4MDQlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwODA0VDEzMjE0M1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTdjOTgzZmE1MmY0ZWM2MWM5MzE5NmEwZDNkZDUyMTk3ODM5N2RkZTU5NWRiN2EzYTAyOGNkNWQxYjg2MGU5ZTImWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.OWOYCPwqeXPpzkMpbAmS5X90SS5Krru2HALie1UBQ4A)

### 5. __maxConnections, acceptCount 는 충분하지만 maxThreads 가 부족한 경우__

```yml
server:
  tomcat:
    max-connections: 10
    accept-count: 10
    threads:
      max: 2
```

![tomcat-thread-image8](https://private-user-images.githubusercontent.com/28802545/354904570-d811bdec-e381-43f0-9316-d53e50a43442.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjI3NzgxOTcsIm5iZiI6MTcyMjc3Nzg5NywicGF0aCI6Ii8yODgwMjU0NS8zNTQ5MDQ1NzAtZDgxMWJkZWMtZTM4MS00M2YwLTkzMTYtZDUzZTUwYTQzNDQyLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA4MDQlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwODA0VDEzMjQ1N1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWMyOTVkYjliNGJlNzk1ZjU2OWZmOWQzMWIwMjczZGJlNDNmZGZlYjQ1MTk2YWMxNWVmODY4OWI0NTkxZDdhOGYmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.txe6wmSuGx7fiCrXLmMXoWTo6bPNCOju4UoyGDSnMbQ)

__maxThreads__ 만큼 5초 간격으로 10개의 요청을 모두 처리한것을 확인할 수 있습니다.

<br />
<hr />

## __Tomcat 스레드 갯수를 어떻게 설정해야할까?__

어플리케이션의 성격과 요구사항 그리고 처리량에 따라 다르겠지만 다음 부분을 고민해보면 좋을것 같습니다.

### CPU Bound, I/O Bound

어플리케이션의 성격이 주로 어떤 작업을 하는지에 대해 고민해볼 수 있습니다.

#### CPU Bound
- CPU 연산에 의존하는 작업 (CPU Burst)
- 머신 러닝 작업, 동영상 편집 작업 ..

#### I/O Bound
- I/O 작업을 요청하고 결과를 기다리는 작업 (I/O Burst)
- DB 접근, 네트워크 요청, 파일 입출력 ..

일반적으로 __CPU Bound__ 의 경우 대기시간이 거의 없기때문에 많인 스레드 수가 필요하지 않습니다.  
그렇기에 코어수 만큼의 스레드 수가 있어도 충분합니다.

__I/O Bound__ 의 경우 대기시간이 많기에 대기시간을 효율적으로 처리할만큼의 스레드 갯수가 있는게 작업을 더 효율적으로 처리할 수 있습니다.

![tomcat-thread-image3](https://private-user-images.githubusercontent.com/28802545/354882088-5c236d79-4eec-4a73-9522-d65107e2bf10.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjI3NTU3MzYsIm5iZiI6MTcyMjc1NTQzNiwicGF0aCI6Ii8yODgwMjU0NS8zNTQ4ODIwODgtNWMyMzZkNzktNGVlYy00YTczLTk1MjItZDY1MTA3ZTJiZjEwLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDA4MDQlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQwODA0VDA3MTAzNlomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWZkMGFlZTgyYWU5ZWNjN2ExN2VhNzQzZmM4YzlhYTVjN2Q1OTc2ZDIwYTNhNDc5ZDY2Nzg2MTFkNzBiYzMwYzEmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0._ekIwCZnicggP4inywOiw6RUmujK9mVEzZOlzHhLqtY)

Brian Goetz 의 __Java Concurrency in Practice__ 라는 책에서 다음과 같은 공식을 권장합니다.

여기서 대기시간은 스레드가 대기하는 시간과 I/O 작업시 대기하는 시간을 모두 포함합니다.  
서비스 시간은 대기시간을 제외한 실제 작업이 수행되는 시간을 뜻합니다.

### TPS 와 성능 테스트

스레드 개수에 정답은 없으므로 가장 중요한것은 어플리케이션의 성격과 현재 서비스의 상황에 맞게 적정 스레드 개수를 찾는것이 중요하지 않을까 싶습니다.

평소엔 잠잠하다가 특정 시간에만 TPS 가 급증하는 서비스도 있고, 전체적으로 적당량의 수치를 유지하는 서비스도 존재합니다.

서비스 성격에 알맞는 TPS 와 AutoScaling 을 고려한 성능 테스트를 진행하며 서비스에 맞는 적정 스레드 개수를 찾아내는것이 가장 좋은 방법이라 생각합니다.

감사합니다.

<br />
<hr />

#### __reference__

- https://tomcat.apache.org/tomcat-8.0-doc/config/http.html#Connector_Comparison
- https://hudi.blog/tomcat-tuning-exercise/
- https://velog.io/@jihoson94/BIO-NIO-Connector-in-Tomcat
- https://bcho.tistory.com/788
- https://code-lab1.tistory.com/269