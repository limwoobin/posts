#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/parallel-stream)

<br />

# __Java ParallelStream__

안녕하세요 이번에는 __Java Stream__ 의 병렬처리를 가능하게 하는 __ParallelStream__ 에 대해 알아보겠습니다.

__Java__ 의 __ParallelStream__ 은 별다른 설정없이 Stream 의 parallel(), Collection의 parallelStream() 메소드를 통해 손쉽게 병렬처리를 할 수 있습니다.  

```java
List<Integer> list = List.of(1, 2, 3, 4, 5, 6, 7, 8);

// parallel()
list.stream()
    .parallel()
    .forEach(System.out::println);

// parallelStream()
list.parallelStream()
    .forEach(System.out::println);
```

__ParallelStream__ 은 내부적으로 __ForkJoinFrameWork__ 방식으로 동작하며  
기본적으로는 ForkJoinPool.commonPool 을 사용한다는 특징을 가지고 있습니다.  
그리고 멀티코어를 이용해 여러 작업들을 동시에 처리하게 됩니다.

하지만 병럴 스트림을 사용한다고 해서 무조건적으로 순차 스트림보다 빠르지는 않습니다.  
오히려 더 느리게 동작할 수 있고 잘못사용하게되면 시스템 장애도 발생할 수 있습니다.

그럼 __ParallelStream__ 의 특징과 주의사항에 대해 한번 알아보겠습니다.

<br />

## __ForkJoinFrameWork 란?__

__ForkJoinFrameWork__ 는 큰 작업을 작은 작업들로 쪼개어 작업을 병렬로 처리하고 처리한 작업들을 다시 큰 작업으로 합치는 방식으로 동작합니다.  
(마치 분할정복 알고리즘과 같이 동작합니다.)

- __Fork:__ 작업들을 작은 작업들로 분할함.
- __Join:__ 분할된 작업들을 큰 작업들로 병합함.

__Fork__
![parallel-stream-5](https://private-user-images.githubusercontent.com/28802545/378394312-f37629e1-d402-43aa-af34-9c114e85450b.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk1MDkyNDQsIm5iZiI6MTcyOTUwODk0NCwicGF0aCI6Ii8yODgwMjU0NS8zNzgzOTQzMTItZjM3NjI5ZTEtZDQwMi00M2FhLWFmMzQtOWMxMTRlODU0NTBiLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDEwMjElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQxMDIxVDExMDkwNFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWM1NTk4NTE5YTNjZDY2NGJkOGYxZTQzOTMxYjhhMTJkMDVlMGExOTU3ZTNlYzc5YTVjMTRmZmRjODY2ODAzYzYmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.1wcSSci5cN7e-sqUS0PcZZABhUSEzlZPngw8rrL6Kus)

__Join__
![parallel-stream-6](https://private-user-images.githubusercontent.com/28802545/378394319-9b774eb2-126e-4f7e-9515-09c7896036f2.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk1MDkyNDQsIm5iZiI6MTcyOTUwODk0NCwicGF0aCI6Ii8yODgwMjU0NS8zNzgzOTQzMTktOWI3NzRlYjItMTI2ZS00ZjdlLTk1MTUtMDljNzg5NjAzNmYyLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDEwMjElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQxMDIxVDExMDkwNFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWE0OGFlNWNkOGU2YTRmNTUxMTdhNWNjMjFlMDIxYTUxMmRkNDQxMTlmNGE0OTBmYjVkYjViZWE1ZjgyZDVlY2EmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.4M7DJyJtZVSbTlY6Y8Ls5uigB1kowCqy6M-0z5H9zKY)

<br />

그리고 ForkJoinFrameWork 는 Work-Stealing 매커니즘으로 동작합니다.  
해당 방식은 다음과 같습니다.

![parallel-stream-7](https://private-user-images.githubusercontent.com/28802545/378399324-47276a80-49f8-4184-9872-f2f2692a7f6d.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk1MTAyMzMsIm5iZiI6MTcyOTUwOTkzMywicGF0aCI6Ii8yODgwMjU0NS8zNzgzOTkzMjQtNDcyNzZhODAtNDlmOC00MTg0LTk4NzItZjJmMjY5MmE3ZjZkLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDEwMjElMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQxMDIxVDExMjUzM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWY4MTA3YTg1M2RhN2NhMjg0MDUzMzQ0YTIxNDNkNmM4YWQ2ZWQ2MGQ2MDYwODZmN2VkNDc0N2ViNTFjODNlOTYmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.JGcgW9Yw9l5DPR407nXCot0Jbq1SEgTbmyOaNKf49Qo)

- 각 스레드는 각자의 WorkQueue 를 가지고 있습니다.
- WorkQueue 의 구조는 Dequeue 구조로 되어있어 head, tail 양쪽에서 모두 push/pop 이 가능합니다.



1. Inbound Queue 에 Task 가 존재하면 스레드는 자신의 Queue 에 가져와서 작업함
2. 다른 스레드가 자신의 Queue 에 Work-Stealing 하는 경우 Queue 의 tail 에서 pop 을 진행.
3. 스레드의 개수가 많아진다면 Work-Stealing 과정에서 경함이 발생할 수 있음

<br />

# __Parallel Stream 주의사항 및 예시__

## __commonPool 사용시 주의할 점__

__ParallelStream__ 은 기본적으로 ForkJoinPool 의 commonPool 을 사용합니다.  
이 commonPool 의 size 는 아래와 같이 설정됩니다. 

> Runtime.getRuntime().availableProcessors() - 1

__Runtime.getRuntime().availableProcessors()__ 는 CPU Core 의 사이즈이고 __-1__ 은 main 스레드를 제외하기 위함입니다.

__commonPool__ 은 JVM에서 공유됩니다. 그렇기에 __parallelStream__ 을 사용하는 경우 모두 동일한 commonPool 을 사용하게 됩니다.  
만약 어플리케이션에서 다양한 병렬작업이 존재해 commonPool 을 같이 사용하는 경우 혹은  
병렬 스트림에서 I/O 작업을 처리하는 경우에는 commonPool 의 리소스가 부족해 시스템 전체에 영향을 줄 수 있습니다.

### __Custom Thread Pool__

병렬 스트림을 처리할때 ThreadPool 을 직접 정의해서 처리할 수 있습니다.

```java
List<Integer> list = List.of(1, 2, 3, 4, 5, 6, 7, 8);

ForkJoinPool customThreadPool = new ForkJoinPool(4);
customThreadPool.submit(() -> list.stream()
      .parallel()
      .forEach(System.out::println)).get();
```

하지만 ThreadPool 을 직접 정의해서 사용하는 경우에는 OutOfMemoryError 에 주의해야 합니다.  
commonPool 은 기본적으로 JVM 에서 공유되며, 정적 ThreadPool 인스턴스입니다. 그래서 메모리 누수가 발생하지 않습니다.

![parallel-stream-1](https://private-user-images.githubusercontent.com/28802545/378052890-c95ed429-f912-42cb-8f75-035450f0f8b1.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3MjkzMTY0OTgsIm5iZiI6MTcyOTMxNjE5OCwicGF0aCI6Ii8yODgwMjU0NS8zNzgwNTI4OTAtYzk1ZWQ0MjktZjkxMi00MmNiLThmNzUtMDM1NDUwZjBmOGIxLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDEwMTklMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQxMDE5VDA1MzYzOFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTk1ZWNlYWEwN2QxMmU4OWMyZWMzNmQyMGViZTcxNjQwYzY0YTBhNjQwN2FjYjkzMjVkZGEwZTZiZjllNDg3YjUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.UytwmuU1ibilqXBipxiZ4W9SVqN6gPkHArPtFx_v8Nc)

하지만, Custom Thread Pool 의 경우에는 처리가 완료되어도 참조가 해제되지 않아 GC 에 수집되지 않을 수 있습니다.  
그래서 사용 후 꼭 아래와 같이 shutdown() 하여 ForkJoinPool 을 종료해주어야 합니다.

```java
List<Integer> list = List.of(1, 2, 3, 4, 5, 6, 7, 8);

ForkJoinPool customThreadPool = new ForkJoinPool(4);
try {
    customThreadPool.submit(() -> list.stream()
        .parallel()
        .forEach(System.out::println)).get();
} finally {
    customThreadPool.shutdown();
}
```

##### __https://www.baeldung.com/java-8-parallel-streams-custom-threadpool__

I/O 작업과 같은 처리에 commonPool 을 묶어두고 싶지 않다면 별도의 ThreadPool 을 정의해서 처리하는 방법이 효율적일 수 있습니다.  
이와 반대로, 간단한 작업이나 경합이 크게 일어나지 않을 환경이라면 commonPool 을 사용하는것이 자원을 더 효율적으로 사용할 수 있지 않을까 싶습니다.

## __독립된 처리__

__ParallelStream__ 사용시 각 작업이 독립되어야 좋은 성능을 낼 수 있습니다.  
stream() 연산 중 distinct(), sorted() 와 같은 작업들은 내부적으로 상태에 대한 변수를 공유하고 동기화하기에 
병렬처리에 대한 이점을 살릴 수 없습니다.

## __데이터 타입__

ParallelStream 은 ForkJoin 방식으로 동작하기에 작업을 나누는 단위가 균등해야하고, 작업을 나누기 위한 비용이 적어야 합니다.  
이러한 비용은 데이터 타입에 따라 차이나게 됩니다. 

이 비용에는 크게 두 가지를 들 수 있습니다.

1. __연속된 메모리 할당(참조 지역성)__

배열의 경우 메모리 상에서 연속된 공간에 저장되므로, 요소가 물리적으로 가까운 위치해 있습니다.  
특정 인덱스의 요소에 접근할 때 메모리에서 그 위치를 바로 계산할 수 있어 매우 빠릅니다.  
이는 분할 시 데이터에 빠르게 접근할 수 있는 장점이 됩니다.

2. __데이터 접근 방식__

ArrayList 의 경우 특정 인덱스에 있는 요소에 접근하는 시간 복잡도는 O(1)입니다.  
분할 시에도 특정 지점에서 데이터를 빠르게 나눌 수 있기 때문에 성능이 좋습니다.

하지만 LinkedList 의 경우 특정 인덱스에 접근하기 위해 순차적으로 노드를 탐색해야 하므로 시간 복잡도는 O(n)입니다.  
즉, 인덱스에 따라 요소를 찾는 데 더 많은 시간이 걸리기 때문에 분할 성능이 떨어집니다.


__ArrayList, IntStream.range, LongStream.range, TreeMap, HashSet__ 과 같은 데이터 타입은 분할하기에 용이합니다.

## __요소의 개수__

병렬처리는 동일한 작업이더라도 요소의 개수에 따라 성능이 좋을수도, 나쁠수도 있습니다.  
이는 작업을 분할하고(Fork) 다시 합치는(Join) 비용과 스레드간의 Context Switching 비용이 존재하기 때문인데요.

간단한 예시를 통해 소요시간을 확인해보겠습니다. 시간은 각 수행하는 PC 의 사양마다 다를 수 있습니다.

__ArrayParallelStreamExample.java__
```java
public class ArrayParallelStreamExample {
  static List<Long> arrayNumbers = new ArrayList<>();

  static {
    LongStream.rangeClosed(1, 1_000_000).forEach(it -> {
      arrayNumbers.add(it);
    });
  }

  public static void main(String[] args) {
    long startSequential = System.currentTimeMillis();
    long sumSequential = arrayNumbers.stream()
      .reduce(0L, Long::sum);

    long endSequential = System.currentTimeMillis();
    System.out.println("Sequential sum: " + sumSequential);
    System.out.println("Sequential processing time: " + (endSequential - startSequential) + "ms");

    long startParallel = System.currentTimeMillis();
    long sumParallel = arrayNumbers.stream()
      .parallel()
      .reduce(0L, Long::sum);

    long endParallel = System.currentTimeMillis();
    System.out.println("Parallel sum: " + sumParallel);
    System.out.println("Parallel processing time: " + (endParallel - startParallel) + "ms");
  }
}
```

ArrayList 기준으로 1 ~ 100만의 숫자를 더하는 코드를 만들었습니다.  
소요시간을 한번 확인해보겠습니다.

![parallel-stream-2](https://private-user-images.githubusercontent.com/28802545/378172426-19dc0c32-3ed0-4049-877d-4a1270daaea2.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0MTU5MDcsIm5iZiI6MTcyOTQxNTYwNywicGF0aCI6Ii8yODgwMjU0NS8zNzgxNzI0MjYtMTlkYzBjMzItM2VkMC00MDQ5LTg3N2QtNGExMjcwZGFhZWEyLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDEwMjAlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQxMDIwVDA5MTMyN1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTdhMDIyZTg0NWU4YmFmM2YzMGVmNWViOTRhNjcyNmZjM2IzMjU5ZGI1OGQ0OWEzNjEwN2E4YzM0Zjg0NjEwNmEmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.qtB3CIuyQm48Sl7jceqo6aMYLjCiJqXK4BtncfAII2M)

기본 스트림이 병렬 스트림보다 더 빠른것을 알 수 있습니다. 이는 100만개 요소를 직렬로 처리하는것보다 100만개 요소에 대한 분할/병합 비용 그리고 스레드간 Context Switching 비용이 더 크다고 볼 수 있습니다.

그렇다면 요소의 수를 100만개가 아니라 1000만개로 실행해보면 어떨까요 ??  
다음과 같이 변경해 실행해보겠습니다.

```java
LongStream.rangeClosed(1, 10_000_000).forEach(it -> {
    arrayNumbers.add(it);
});
```

![parallel-stream-3](https://private-user-images.githubusercontent.com/28802545/378172556-5c31e0a0-cdfa-43c3-bf67-2e7acf89fed8.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0MTYwMzgsIm5iZiI6MTcyOTQxNTczOCwicGF0aCI6Ii8yODgwMjU0NS8zNzgxNzI1NTYtNWMzMWUwYTAtY2RmYS00M2MzLWJmNjctMmU3YWNmODlmZWQ4LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDEwMjAlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQxMDIwVDA5MTUzOFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTI0MjU2OGNiNjRkNTMzNDQxMzA5YjIxMjZlMmY2ZjFhZmMwODczYzU0YTVkZDFlZWU1MjliNDk5YTMyOWViYjQmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.RAF9GynLUUoUYB5d115aR8axuZ6SUXw_tnmrKq25xUU)

요소의 수가 많아지니 병렬 스트림이 더 빠르게 동작한것을 확인할 수 있습니다.  
5000만개도 한번 확인해보겠습니다.

```java
LongStream.rangeClosed(1, 50_000_000).forEach(it -> {
    arrayNumbers.add(it);
});
```

![parallel-stream-4](https://private-user-images.githubusercontent.com/28802545/378173208-324b4c34-72cb-4699-8137-d10919f3fbe1.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3Mjk0MTY2MTQsIm5iZiI6MTcyOTQxNjMxNCwicGF0aCI6Ii8yODgwMjU0NS8zNzgxNzMyMDgtMzI0YjRjMzQtNzJjYi00Njk5LTgxMzctZDEwOTE5ZjNmYmUxLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFWQ09EWUxTQTUzUFFLNFpBJTJGMjAyNDEwMjAlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjQxMDIwVDA5MjUxNFomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWY4ZjJmNjI1ZTlmZjViYjdiNmM4YjMxMmQ1NjhhNjgyNmQ2ZTYwZTA5ZWNhOGExN2JlYzIwNzY4M2Y2ODJjMWImWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0In0.EUluKRS71BBnpucoWuBgYNQEVwpwuWrgm9fkAoc3Pis)

요소의 개수가 많아질수록 소요시간의 차이가 더 확연하게 나타났습니다.

이렇듯, 병렬 스트림은 고려할 변수가 많아서 직접 테스트를 거치며 사용여부를 결정하는것이 좋지 않을까 생각합니다.

<br />

#### __reference__

- https://dev-coco.tistory.com/183
- https://www.baeldung.com/java-when-to-use-parallel-stream
- https://www.baeldung.com/java-8-parallel-streams-custom-threadpool#bd-beware-of-the-memory-leak
- https://www.baeldung.com/java-8-parallel-streams-custom-threadpool