#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/parallel-stream)

<br />

# __Java ParallelStream__

안녕하세요 이번에는 __Java Stream__ 의 병렬 처리를 가능하게 하는 __ParallelStream__ 에 대해 알아보겠습니다.

__Java__ 의 __ParallelStream__ 은 별다른 설정없이 Stream 의 parallel(), parallelStream() 메소드를 통해 손쉽게 병렬처리를 할 수 있습니다.  

```java
List<Integer> list = List.of(1, 2, 3, 4, 5);

// parallel()
list.stream()
    .parallel()
    .forEach(System.out::println);

// parallelStream()
list.parallelStream()
    .forEach(System.out::println);
```

__ParallelStream__ 은 내부적으로 __ForkJoinFrameWork__ 방식으로 동작하며  
기본 스레드는 common ForkJoinPool 을 사용한다는 특징을 가지고 있습니다.  

이제 __ParallelStream__ 의 특징과 사용법 및 주의사항 등에 대해 하나씩 알아보겠습니다.

<br />

## __ForkJoinFrameWork 란?__

__ForkJoinFrameWork__ 는 큰 작업을 작은 작업들로 쪼개어 작업을 병렬로 처리하고 처리한 작업들을 다시 큰 작업으로 합치는 방식으로 동작합니다.  
(마치 분할정복 알고리즘과 같이 동작합니다.)

- Fork: 작업들을 작은 작업들로 분할함.
- Join: 분할된 작업들을 큰 작업들로 병합함.



ForkJoinPool 에 대해서는 다음 번에 더 상세히 작성해보겠습니다.

<br />

## __Parallel Stream 주의사항 및 예시__

__ParallelStream__ 은 기본적으로 ForkJoinPool 의 commonPool 을 사용합니다.  
이 commonPool 의 size 는 아래와 같이 설정됩니다. 

> Runtime.getRuntime().availableProcessors() - 1

__Runtime.getRuntime().availableProcessors()__ 는 CPU Core 의 사이즈이고 __-1__ 은 main 스레드를 제외하기 위함입니다.

commonPool 은 어플리케이션에서 공유되는 pool 입니다. 그렇기에 parallelStream 을 commonPool 로 사용하는 경우에는 이 부분에 꼭 유의해야 합니다.

예를들어, 외부 네트워크 통신과 같은 Blocking 작업을 한다고 가정했을때, 해당 작업을 하는동안은 commonPool 의 스레드는 모두 작업상태이고 이는 곧 다른 곳에서는 해당 commonPool 을 사용할 수 없다는 뜻입니다. 그렇기에 예기치 않게 다른 기능에 장애를 전파할 수 있습니다.

## Parallel Stream 을 올바르게 사용하는 방법

### 분할된 작업 ...

### 자료구조
- ArrayList
- LinkedList
- Set
- Map

## parallel() vs parallelStream() 간단 비교

## benchmark 를 이용한 성능테스트 작성 (JMH ?)

#### __reference__

- https://dev-coco.tistory.com/183
- https://www.baeldung.com/java-when-to-use-parallel-stream
- https://www.baeldung.com/java-8-parallel-streams-custom-threadpool#bd-beware-of-the-memory-leak