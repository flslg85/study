# 스프링으로 시작하는 리액티브 프로그래밍
1. [리액티브 프로그래밍](#1-리액티브-프로그래밍)
2. [리액티브 스트림즈(Reactive Streams)](#2-리액티브-스트림즈reactive-streams)
3. [Blocking I/O 와 Non-Blocking I/O](#3-blocking-io-와-non-blocking-io)
4. [리액티브 프로그래밍을 위한 사전지식](#4-리액티브-프로그래밍을-위한-사전지식)
5. 
6.
7.
8. [Backpressure](#8-backpressure)
9. [Sinks](#9-sinks)
10. [Scheduler](#10scheduler)
11. 
12.
13.
14. [Operators](#14-operators)
15. [Spring Webflux 개요](#15-spring-webflux-개요)
16. [애너테이션 기반 컨트롤러](#16-애너테이션-기반-컨트롤러)

---


# 1. 리액티브 프로그래밍
1.1 리액티브 시스템이란?
- Responsive(응답성)
- Resilient(회복성)
- Elastic(탄력성)
- Message Driven(메시지 기반)
- 참고
  - 리액티브 선언문 - https://www.reactivemanifesto.org/ko

1.4 리액티브 프로그래밍이란?
- reactive programming is a declarative programming paradigm concerned with data streams and the propagation of change.
- 참고
  - https://en.wikipedia.org/wiki/Reactive_programming

1.5 명령형 프로그래밍 vs 선언형 프로그래밍
- 명령형 프로그래밍
  - 어떤 작업을 처리하기 위해 실행할 동작을 코드로 명시
- 선언형 프로그래밍
  - 실행할 동작을 구체적으로 명시하지 않고 목표만 선언



1.6 리액티브 프로그래밍 코드 구성
- Publisher
  - 입력으로 들어오는 데이터를 제공하는 역할
  - 발행자, 생산자
- Subscriber
  - Publisher 가 제공한 데이터를 전달받아서 사용하는 주체
  - 구독자, 소비자
- Data Source
  - Publisher 의 입력으로 들어오는 데이터
- Operator
  - Publisher 와 Subscriber 사이에서 적절한 가공 처리를 담당

# 2. 리액티브 스트림즈(Reactive Streams)

## 2.1 리액티브 스트림즈란?
- 데이터 스트림을 Non-Blocking이면서 비동기적인 방식으로 처리하기 위해, 리액티브 라이브러리를 어떻게 구현할지 정의해 놓은 별도의 표준 사양
- 구현체
  - RxJava, Reactor, Akka, Streams, Java 9 Flow API

## 2.2 리액티브 스트림즈 구성요소

- Publisher
  - 데이터를 생성하고 발행하는 역할
- Subscriber
  - 구독한 Publisher로부터 발행된 데이터를 전달받아서 처리하는 역할
- Subscription
  - Publisher 에 요청할 데이터의 개수를 지정하고, 데이터의 구독을 취소하는 역할
- Processor
  - Publisher 와 Subscriber 의 기능을 모두 가지고 있음

## 2.3 리액티브 스트림즈 컴포넌트

### Publisher

~~~java
public interface Publisher<T> {
    public void subscribe(Subscriber<? super T> s);
}
~~~

> Publisher 는 데이터를 생성하고 통지하는 역할을 하고,
 Subscriber 는 Publisher 가 통지하는 데이터를 전달 받기 위해 구독을 한다고 이해하고 있는데,
 왜 Subscriber 가 아닌 Publisher 에 subscribe 메서드가 정의되어 있을까?

- kafka 같은 메시지 기반의 시스템에서의 pub/sub 모델과 의미가 다름
- kafka 는 중간에 Message Broker 가 있고, Publisher 와 Subscriber 는 Message Broker 를 바라보는 구조임
- 이러한 구조로 인해, Publisher 는 메세지를 보내기만하고 Subscriber 는 전달 받기만 하면 됨

- 반면에, 리액티브 스트림즈에서는 Publisher 와 Subscriber 개념상으로는 Subscriber 가 구독하는 것이 맞음
- 그러나 실제 코드상에서는 Publisher 의 subscribe 메서드를 통해 Subscriber 를 등록해야 pub -> sub 로 메세지 전달이 가능함

### Subscriber

~~~java
public interface Subscriber<T> {
    public void onSubscribe(Subscription s);
    public void onNext(T t);
    public void onError(Throwable t);
    public void onComplete();
}
~~~

- onSubscribe
  - 구독 시작 시점에 호출, Publisher 에게 요청할 데이터의 개수를 지정하거나, 구독을 해지하는 처리를 함
- onNext
  - Publisher 가 발행한 데이터가 있을 때 호출
- onError
  - Publisher 가 데이터를 발행하기 위한 처리 과정에서 에러가 발생했을때 호출
- onComplete
  - Publisher 가 데이터 발행을 완료했음을 알릴때 호출

### Subscription

~~~java
public interface Subscription {
    public void request(long n);
    public void cancel();
}
~~~

- request
  - Publisher 에게 데이터의 개수를 요청
- cancel
  - 구독을 해지


### Processor

~~~java
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {
}
~~~


### sample

~~~java
import org.reactivestreams.Publisher;
import org.reactivestreams.Subscriber;
import org.reactivestreams.Subscription;

import java.util.Arrays;
import java.util.Iterator;

public class MyApp {
    public static void main(String[] args) {
        MyPublisher publisher = new MyPublisher();

        publisher.subscribe(new MySubscriber());
    }

    public static class MyPublisher implements Publisher<Integer> {
        Iterable<Integer> its = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

        public MyPublisher() {
            System.out.println("Publisher 생성");
        }

        @Override
        public void subscribe(Subscriber<? super Integer> subscriber) {
            System.out.println("subscribe 구독 요청");
            subscriber.onSubscribe(new MySubscription(its, subscriber));
        }
    }

    public static class MySubscription implements Subscription {
        private Iterator<Integer> it;
        private final Subscriber subscriber;

        public MySubscription(Iterable<Integer> its, Subscriber subscriber) {
            System.out.println("  Subscription 생성");
            this.it = its.iterator();
            this.subscriber = subscriber;
        }

        @Override
        public void request(long n) {
            System.out.println("request " + n + "개 요청");
            while (n-- > 0) {
                if (it.hasNext()) {
                    subscriber.onNext(it.next());
                } else {
                    subscriber.onComplete();
                    break;
                }
            }
        }

        @Override
        public void cancel() {
            System.out.println("구독 취소");
        }

    }

    public static class MySubscriber implements Subscriber<Integer> {
        private Subscription subscription;
        private static final int REQUEST_SIZE = 2;
        private int size = 2;

        public MySubscriber() {
            System.out.println("Subscriber 생성");
        }


        @Override
        public void onSubscribe(Subscription subscription) {
            System.out.println("onSubscribe 구독 정보 전송");

            this.subscription = subscription;
            subscription.request(REQUEST_SIZE);
        }

        @Override
        public void onNext(Integer item) {
            System.out.println("  onNext 데이터: " + item);
            size--;
            if (size == 0) {
                size = REQUEST_SIZE;
                subscription.request(REQUEST_SIZE);
            }

        }

        @Override
        public void onError(Throwable throwable) {
            System.out.println("에러");
        }

        @Override
        public void onComplete() {
            System.out.println("onComplete 완료");
        }
    }

}
~~~

~~~
Publisher 생성
Subscriber 생성
subscribe 구독 요청
  Subscription 생성
onSubscribe 구독 정보 전송
request 2개 요청
  onNext 데이터: 1
  onNext 데이터: 2
request 2개 요청
  onNext 데이터: 3
  onNext 데이터: 4
request 2개 요청
  onNext 데이터: 5
  onNext 데이터: 6
request 2개 요청
  onNext 데이터: 7
  onNext 데이터: 8
request 2개 요청
  onNext 데이터: 9
  onNext 데이터: 10
request 2개 요청
onComplete 완료
~~~

## 2.4 리액티브 스트림즈 관련 용어
- signal
  - Publisher 와 Subscriber 간에 주고 받는 상호작용
  - ex> onSubscribe, onNext, onComplete, onError, request, cancel
- demand
  - Subscriber 가 Publisher 에게 요청하였지만, Publisher 가 아직 Subscriber 에게 전달하지 않은 데이터
- emit
  - 데이터를 내보내는 것을 의미
  - 다양한 용어(통지, 발행, 게시, 방출 등)가 있지만 책에서 emit 으로 사용함
- Downstream/ Upstream
  - 데이터 흐름 위로 아래로
- Sequence
  - 정확한 의미를 찾기 어렵지만, Operator 로 데이터의 연속적인 흐름의 정의한 것으로 생각하면 됨
- Operator
  - just, filter, map 같은 메서드들을 말함
- Source
  - 최초에 가장 먼저 생성된 무언가 정도로 기억
  - Original 이라는 용어도 Source 와 같다고 보면 됨


## 2.5 리액티브 스트림즈 구현 규칙
책에 표를 보면 됨... 타이핑 구찮..


## 2.6 리액티브 스트림즈 구현체
- `RxJava`, `Project Reactor`, `Akka Streams`, `Java Flow API`, `RxAndroid`, `RxJs`, `RxKotlin` 등


# 3. Blocking I/O 와 Non-Blocking I/O

## 3.1 Blocking I/O
- 하나의 스레드가 I/O 에 의해서 차단되어 대기하는 것을 **Blocking I/O** 라고 함
- Blocking I/O 방식의 문제점을 보완하기 위해 멀티스레딩 기법으로 추가 스레드를 할당
- cpu 대비 많은 수의 스레드를 할당하는 멀티스레딩 기법은 문제점이 존재함

### 단점
- context switching 으로 인한 스레드 전환 비용
  - 프로세스의 정보를 PCB(Process Control Block)에 저장, reload 하는 시간동안 CPU 가 다른 작업을 하지 못하고 대기
  - context switching 이 많을수록 cpu 전체 대기 시간이 길어지기 때문에 성능 저하
- 메모리 사용 오버헤드
  - 서블릿 컨테이이너 기반의 Java 웹 어플리케이션은 `요청당 하나의 스레드(thread per request)`를 할당
  - JVM 에서 스레드 정보를 stack 에 할당하여 메모리 증가
- thread pool 의 응답 지연
  - 대량의 요청이 발생하게 되어 thread pool 에 사용 가능한 유후 스레드가 없을 경우, 사용 가능한 스레드가 확보되기 전까지 응답 지연 발생


## 3.2 Non-Blocking I/O
- 작업 스레드의 종료 여부와 관계없이 요청한 스레드는 차단되지 않음
  - 스레드가 차단되지 않기 때문에 하나의 스레드로 많은 수의 요청을 처리할 수 있음
- Blocking I/O 보다 적은 수의 스레드를 사용하기 때문에 멀티스레딩 기법을 사용할때 문제점이 없음

### 단점
- 스레드 내부에 cpu 를 많이 사용하는 작업이 포함된 경우에는 성능에 악영향을 줌
- Blocking I/O 가 포함된 경우 Blocking I/O 로 스레드가 차단되면서 병목 구간이 발생하기 때문에 Non-Blocking의 이점을 발휘하기 힘듬


## 3.3 Spring Framework 에서의 Blocking I/O 와 Non-Blocking I/O
- 스마트폰이나 태블릿 등의 휴대기기, IOT 기술 등의 발전으로 Blocking I/O 방식의 애플리케이션이 감당하기 힘들 만큼의 클라이언트 요청 트래픽이 발생하는 상황이 많아짐.
- 이러한 문제점을 극복하기 위해 Spring WebFlux 가 나옴

- 책에서 간단한 코드로 설명하고 있지만, 별 영양가 없는 설명이므로 패스

## 3.4 Non-Blocking I/O 방식의 통신이 적합한 시스템
Spring WebFlux 도입 고려 사항
- 학습 난이도
- 개발 인력 확보 가능?

적합한 시스템
- 대량의 요청 트래픽이 발생하는 시스템
- 마이크로 서비스 기반 시스템
- 스트리밍 또는 실시간 시스템
# 4. 리액티브 프로그래밍을 위한 사전지식

## 4.1 함수형 인터페이스(Functional Interface)
- 추상 메서드가 오직 하나인 인터페이스를 함수형 인터페이스라고 한다.
- 디폴트 메서드를 포함할 수 있다.
- 메서드 레퍼런스, 람다를 함수형 인터페이스로 사용할 수 있다.

#### @FunctionalInterface 어노테이션
- 함수형 인터페이스임을 가리키는 어노테이션
- 이 어노테이션을 사용한 인터페이스가 함수형 인터페이스가 아니면 컴파일 에러가 발생한다.
- 예를 들어 추상 메서드가 한개 이상이라면 "Multiple nonoverriding abstract methods found in interface Foo"

## 4.2 람다 표현식(Lambda Expression)
- 람다는 메서드로 전달할 수 있는 익명 함수 생성을 단순화한 것

- 람다가 기술적으로 자바 8 이전의 자바로 할 수 없었던 일을 제공하는 것은 아니다. - 익명 클래스를 이용하여 가능했음

- 람다가 몇줄 이상으로 길어진다면, 익명 람다보다 코드가 수행하는 일을 잘 설명하는 이름을 가진 메서드를 정의하고 메서드 레퍼런스를 활용하는 것이 바람직히다. 코드의 명확성이 우선시 되어야 한다!

#### 특징
- 보통의 메서드와 달리 이름이 없으므로 "익명"이라 표현한다.
- 람다는 메서드처럼 특정 클래스에 종속되지 않으므로 "함수"라고 부른다. 하지만 메서드처럼 파라미터 리스트, 바디, 반환 형식, 가능한 예외 리스트를 포함한다.
- 람다 표현식을 메서드 "인수로 전달"하거나 "변수로 저장"할 수 있다.
- 익명 클래스처럼 많은 자질구레한 코드를 구현할 필요가 없어 "간결"하다.
- 어디에서 람다 표현식을 사용할 수 있는가?
  - 함수형 인터페이스라는 문맥에서 람다 표현식을 사용할 수 있다.

## 4.3 메서드 레퍼런스(Method Reference)
- 기존의 메서드 정의를 메서드를 값으로 사용하라는 의미 :: 를 이용해서 람다처럼 전달 할 수 있음
- 람다 표현식보다 메서드 레퍼런스를 사용하는 것이 더 가독성이 좋으며 자연스러울 수 있다.
- ex> Apple::getWeight

## 4.4 함수 디스크립터(Function Descriptor)
- 함수형 인터페이스의 추상 메서드 signature 는 람다 표현식의 signature 를 가리킨다.
- 람다 표현식의 signature 를 서술하는 메서드를 함수 디스크립터라고 부른다.

~~~java
// 이 인터페이스는 하나의 추상 메서드만 존재하므로 함수형 인터페이스이다.
@FunctionalInterface // 함수형 인터페이스임을 가리키는 
public interface Predicate<T> {
    boolean test(T t); << 함수 디스크립터
}
~~~

# 8. Backpressure

## 8.1 Backpressure 란?

Publisher 가 emit 하는 데이터를 적절하게 제어하여 데이터 처리에 과부하가 걸리지 않도록 제어 하는것
ex> Publisher 가 emit 하는 데이터를 처리하지 못하고 쌓이게 되면 오버플로가 발생하거나 최악의 경우 시스템이 다운되는 문제가 발생

## 8.2 Reactor 에서의 Backpressure 처리 방식

### 8.2.1 데이터 개수 제어

* Subscriber 가 request() 메서드를 통해서 적절한 데이터 개수를 요청하는 방식

~~~Java
import lombok.SneakyThrows;
import lombok.extern.slf4j.Slf4j;
import org.reactivestreams.Subscription;
import reactor.core.publisher.BaseSubscriber;
import reactor.core.publisher.Flux;

@Slf4j
public class Example8_1 {
    public static void main(String[] args) {
        Flux.range(1, 5)
                .doOnRequest(data -> log.info("# doOnRequest: {}", data))
                .subscribe(new BaseSubscriber<Integer>() {
                    @Override
                    protected void hookOnSubscribe(Subscription subscription) {
                        log.info("# hookOnSubscribe");
                        request(1);
                    }

                    @SneakyThrows
                    @Override
                    protected void hookOnNext(Integer value) {
                        Thread.sleep(200L);
                        log.info("# hookOnNext: {}", value);
                        request(1);
                    }
                });
    }
}
~~~

~~~
08:44:19.049 [main] INFO com.example.test.Example8_1 - # hookOnSubscribe
08:44:19.049 [main] INFO com.example.test.Example8_1 - # doOnRequest: 1        // hookOnSubscribe
08:44:19.256 [main] INFO com.example.test.Example8_1 - # hookOnNext: 1
08:44:19.256 [main] INFO com.example.test.Example8_1 - # doOnRequest: 1        // hookOnNext
08:44:19.461 [main] INFO com.example.test.Example8_1 - # hookOnNext: 2
08:44:19.463 [main] INFO com.example.test.Example8_1 - # doOnRequest: 1        // hookOnNext
08:44:19.668 [main] INFO com.example.test.Example8_1 - # hookOnNext: 3
08:44:19.669 [main] INFO com.example.test.Example8_1 - # doOnRequest: 1        // hookOnNext
08:44:19.874 [main] INFO com.example.test.Example8_1 - # hookOnNext: 4
08:44:19.875 [main] INFO com.example.test.Example8_1 - # doOnRequest: 1        // hookOnNext
08:44:20.080 [main] INFO com.example.test.Example8_1 - # hookOnNext: 5
08:44:20.081 [main] INFO com.example.test.Example8_1 - # doOnRequest: 1        // hookOnNext
~~~

* hookOnSubscribe() 에서 최초 데이터 요청 개수를 제어
* hookOnNext()
  * Publisher 가 emit 한 데이터를 처리한 후 Publisher 에게 데이터 개수를 지정하여 요청  
  * '# doOnRequest: 1' 로그에 데이터를 몇개씩 요청하는지 로그로 확인


### 8.2.2 Backpressure 전략 사용

#### IGNORE 전략
* 단순히 Backpressure 를 적용하지 않는 것으로 특별히 전략이라고 할것도 없음
* Downstream 에서의 Backpressure 요청이 무시되기 대문에 IllegalStateException 이 발생할수 있음

#### ERROR 전략
* Downstream 의 데이터 처리 속도가 느려서 Upstream 의 emit 속도를 따라가지 못할 경우, IllegalStateException 발생
* Publisher 는 Error Signal 을 Subscriber 에게 전송하고 삭제한 데이터는 폐기

~~~Java
public class Example8_2 {

    public static void main(String[] args) {
        try {
            Flux.interval(Duration.ofMillis(1L))
                    .onBackpressureError()
                    .doOnNext(data -> log.info("# doOnNext: {}", data))
                    .publishOn(Schedulers.parallel())
                    .subscribe(data -> {
                                try {
                                    Thread.sleep(5L);
                                } catch (InterruptedException e) {}

                                log.info("# onNext: {}", data);
                            },
                            error -> log.error("# onError", error));
            Thread.sleep(2000L);
        } catch (Exception e) {
            log.error(e.getMessage(), e);
        }
    }
}
~~~

~~~
09:02:01.380 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 0
09:02:01.385 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 1
09:02:01.385 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 2
09:02:01.385 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 3
09:02:01.385 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 4
09:02:01.385 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 5
09:02:01.386 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 6
09:02:01.387 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 7
09:02:01.388 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 8
09:02:01.389 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 9
09:02:01.390 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 10
09:02:01.390 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 0
09:02:01.391 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 11
09:02:01.392 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 12
09:02:01.393 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 13
09:02:01.394 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 14
09:02:01.395 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 15
09:02:01.396 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 16
09:02:01.397 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 1
09:02:01.397 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 17
09:02:01.398 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 18
09:02:01.399 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 19
09:02:01.400 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 20
09:02:01.401 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 21
09:02:01.402 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 22
09:02:01.402 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 2
09:02:01.403 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 23
09:02:01.404 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 24
09:02:01.405 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 25
09:02:01.406 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 26
09:02:01.407 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 27
09:02:01.408 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 3
09:02:01.408 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 28
09:02:01.409 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 29
09:02:01.410 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 30
09:02:01.411 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 31
09:02:01.412 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 32
09:02:01.413 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 33
09:02:01.414 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 34
09:02:01.414 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 4
09:02:01.415 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 35
09:02:01.416 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 36
09:02:01.417 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 37
09:02:01.418 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 38
09:02:01.419 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 39
09:02:01.420 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 40
09:02:01.420 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 5
09:02:01.421 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 41
09:02:01.422 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 42
09:02:01.423 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 43
09:02:01.424 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 44
09:02:01.425 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 45
09:02:01.425 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 6
09:02:01.426 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 46
09:02:01.427 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 47
09:02:01.428 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 48
09:02:01.429 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 49
09:02:01.430 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 50
09:02:01.431 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 51
09:02:01.432 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 7
09:02:01.432 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 52
09:02:01.433 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 53
09:02:01.434 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 54
09:02:01.435 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 55
09:02:01.436 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 56
09:02:01.437 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 57
09:02:01.438 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 58
09:02:01.438 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 8
09:02:01.439 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 59
09:02:01.440 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 60
09:02:01.441 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 61
09:02:01.442 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 62
09:02:01.443 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 63
09:02:01.444 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 64
09:02:01.444 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 9
09:02:01.445 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 65
09:02:01.446 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 66
09:02:01.447 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 67
09:02:01.448 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 68
09:02:01.449 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 69
09:02:01.450 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 70
09:02:01.451 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 10
09:02:01.451 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 71
09:02:01.452 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 72
09:02:01.453 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 73
09:02:01.454 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 74
09:02:01.455 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 75
09:02:01.456 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 76
09:02:01.457 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 77
09:02:01.457 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 11
09:02:01.458 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 78
09:02:01.459 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 79
09:02:01.460 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 80
09:02:01.461 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 81
09:02:01.462 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 82
09:02:01.463 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 83
09:02:01.463 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 12
09:02:01.464 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 84
09:02:01.465 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 85
09:02:01.466 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 86
09:02:01.467 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 87
09:02:01.468 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 88
09:02:01.469 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 89
09:02:01.469 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 13
09:02:01.470 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 90
09:02:01.471 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 91
09:02:01.472 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 92
09:02:01.473 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 93
09:02:01.474 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 94
09:02:01.475 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 95
09:02:01.475 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 14
09:02:01.476 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 96
09:02:01.477 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 97
09:02:01.478 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 98
09:02:01.479 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 99
09:02:01.480 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 100
09:02:01.481 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 15
09:02:01.481 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 101
09:02:01.482 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 102
09:02:01.483 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 103
09:02:01.484 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 104
09:02:01.485 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 105
09:02:01.486 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 106
09:02:01.487 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 107
09:02:01.488 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 108
09:02:01.488 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 16
09:02:01.489 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 109
09:02:01.490 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 110
09:02:01.491 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 111
09:02:01.492 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 112
09:02:01.493 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 113
09:02:01.494 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 17
09:02:01.495 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 114
09:02:01.495 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 115
09:02:01.496 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 116
09:02:01.497 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 117
09:02:01.498 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 118
09:02:01.499 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 119
09:02:01.500 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 18
09:02:01.500 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 120
09:02:01.501 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 121
09:02:01.502 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 122
09:02:01.503 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 123
09:02:01.504 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 124
09:02:01.505 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 125
09:02:01.505 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 19
09:02:01.506 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 126
09:02:01.507 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 127
09:02:01.508 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 128
09:02:01.509 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 129
09:02:01.510 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 130
09:02:01.511 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 131
09:02:01.511 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 20
09:02:01.512 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 132
09:02:01.513 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 133
09:02:01.514 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 134
09:02:01.515 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 135
09:02:01.516 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 136
09:02:01.517 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 137
09:02:01.517 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 21
09:02:01.518 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 138
09:02:01.519 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 139
09:02:01.520 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 140
09:02:01.521 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 141
09:02:01.522 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 142
09:02:01.523 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 143
09:02:01.524 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 22
09:02:01.524 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 144
09:02:01.525 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 145
09:02:01.526 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 146
09:02:01.527 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 147
09:02:01.528 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 148
09:02:01.529 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 149
09:02:01.530 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 150
09:02:01.530 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 23
09:02:01.531 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 151
09:02:01.532 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 152
09:02:01.533 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 153
09:02:01.534 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 154
09:02:01.535 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 155
09:02:01.536 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 156
09:02:01.536 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 24
09:02:01.537 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 157
09:02:01.538 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 158
09:02:01.539 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 159
09:02:01.540 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 160
09:02:01.541 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 161
09:02:01.542 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 162
09:02:01.543 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 25
09:02:01.543 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 163
09:02:01.544 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 164
09:02:01.545 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 165
09:02:01.546 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 166
09:02:01.547 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 167
09:02:01.548 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 168
09:02:01.549 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 26
09:02:01.549 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 169
09:02:01.550 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 170
09:02:01.551 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 171
09:02:01.552 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 172
09:02:01.553 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 173
09:02:01.554 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 174
09:02:01.555 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 175
09:02:01.555 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 27
09:02:01.556 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 176
09:02:01.557 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 177
09:02:01.558 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 178
09:02:01.559 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 179
09:02:01.560 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 180
09:02:01.561 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 181
09:02:01.561 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 28
09:02:01.562 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 182
09:02:01.563 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 183
09:02:01.564 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 184
09:02:01.565 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 185
09:02:01.566 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 186
09:02:01.566 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 29
09:02:01.567 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 187
09:02:01.568 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 188
09:02:01.569 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 189
09:02:01.570 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 190
09:02:01.571 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 191
09:02:01.572 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 192
09:02:01.572 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 30
09:02:01.573 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 193
09:02:01.574 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 194
09:02:01.575 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 195
09:02:01.576 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 196
09:02:01.577 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 197
09:02:01.577 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 31
09:02:01.578 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 198
09:02:01.579 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 199
09:02:01.580 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 200
09:02:01.581 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 201
09:02:01.582 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 202
09:02:01.583 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 203
09:02:01.584 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 32
09:02:01.584 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 204
09:02:01.585 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 205
09:02:01.586 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 206
09:02:01.587 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 207
09:02:01.588 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 208
09:02:01.589 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 209
09:02:01.590 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 210
09:02:01.590 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 33
09:02:01.591 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 211
09:02:01.592 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 212
09:02:01.593 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 213
09:02:01.594 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 214
09:02:01.595 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 215
09:02:01.596 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 216
09:02:01.596 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 34
09:02:01.597 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 217
09:02:01.598 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 218
09:02:01.599 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 219
09:02:01.600 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 220
09:02:01.601 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 221
09:02:01.602 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 222
09:02:01.603 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 35
09:02:01.603 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 223
09:02:01.604 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 224
09:02:01.605 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 225
09:02:01.606 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 226
09:02:01.607 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 227
09:02:01.608 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 228
09:02:01.609 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 229
09:02:01.609 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 36
09:02:01.610 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 230
09:02:01.611 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 231
09:02:01.612 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 232
09:02:01.613 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 233
09:02:01.614 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 234
09:02:01.615 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 235
09:02:01.615 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 37
09:02:01.616 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 236
09:02:01.617 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 237
09:02:01.618 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 238
09:02:01.619 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 239
09:02:01.620 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 240
09:02:01.621 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 241
09:02:01.621 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 38
09:02:01.622 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 242
09:02:01.623 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 243
09:02:01.624 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 244
09:02:01.625 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 245
09:02:01.626 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 246
09:02:01.627 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 247
09:02:01.627 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 39
09:02:01.628 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 248
09:02:01.629 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 249
09:02:01.630 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 250
09:02:01.631 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 251
09:02:01.632 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 252
09:02:01.633 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 253
09:02:01.633 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 40
09:02:01.634 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 254
09:02:01.635 [parallel-2] INFO com.example.test.Example8_2 - # doOnNext: 255
09:02:01.640 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 41
09:02:01.646 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 42
09:02:01.651 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 43
09:02:01.658 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 44
...
09:02:03.103 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 253
09:02:03.109 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 254
09:02:03.116 [parallel-1] INFO com.example.test.Example8_2 - # onNext: 255
09:02:03.121 [parallel-1] ERROR com.example.test.Example8_2 - # onError
reactor.core.Exceptions$OverflowException: The receiver is overrun by more signals than expected (bounded queue...)
	at reactor.core.Exceptions.failWithOverflow(Exceptions.java:220)
	at reactor.core.publisher.Flux.lambda$onBackpressureError$27(Flux.java:6739)
	at reactor.core.publisher.FluxOnBackpressureDrop$DropSubscriber.onNext(FluxOnBackpressureDrop.java:135)
	at reactor.core.publisher.FluxInterval$IntervalRunnable.run(FluxInterval.java:125)
	at reactor.core.scheduler.PeriodicWorkerTask.call(PeriodicWorkerTask.java:59)
	at reactor.core.scheduler.PeriodicWorkerTask.run(PeriodicWorkerTask.java:73)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.runAndReset(FutureTask.java:308)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$301(ScheduledThreadPoolExecutor.java:180)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:294)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
~~~

* doOnNext()
  * Publisher 가 emit 한 데이터를 확인하거나 추가적인 동작을 정의, 주로 디버깅 용도 
* publishOn() 별도의 스레드가 하나 더 실행됨
  * parallel-1, parallel-2 두개가 로그로 찍힘 
* publisher 가 0.0001 초에 한번씩 데이터 emit - doOnNext
* Subscriber 에서 0.005 초에 한번씩 데이터를 처리 - onNext
* OverflowException 이 발생하면서 Sequence 종료

#### DROP 전략

* Publisher 가 Downstream 으로 전달할 데이터가 버퍼에 가득찰 경우, 버퍼 밖에서 대기중인 데이터중에 **먼저 emit 된 데이터부터 drop**

~~~Java
public class Example8_3 {

    public static void main(String[] args) throws Exception{
        Flux.interval(Duration.ofMillis(1L))
                .onBackpressureDrop(dropped -> log.info("# onBackpressureDrop: {}", dropped))
                .doOnNext(data -> log.info("# doOnNext: {}", data))
                .publishOn(Schedulers.parallel())
                .subscribe(data -> {
                            try {
                                Thread.sleep(5L);
                            } catch (InterruptedException e) {}

                            log.info("# onNext: {}", data);
                        },
                        error -> log.error("# onError"));
        Thread.sleep(2000L);
    }
}
~~~

~~~
09:11:51.201 [parallel-2] INFO com.example.test.Example8_3 - # doOnNext: 253
09:11:51.201 [parallel-2] INFO com.example.test.Example8_3 - # doOnNext: 254
09:11:51.203 [parallel-2] INFO com.example.test.Example8_3 - # doOnNext: 255
09:11:51.204 [parallel-2] INFO com.example.test.Example8_3 - # onBackpressureDrop: 256
09:11:51.205 [parallel-2] INFO com.example.test.Example8_3 - # onBackpressureDrop: 257
09:11:51.206 [parallel-2] INFO com.example.test.Example8_3 - # onBackpressureDrop: 258
09:11:52.483 [parallel-1] INFO com.example.test.Example8_3 - # onNext: 255
09:11:52.484 [parallel-2] INFO com.example.test.Example8_3 - # onBackpressureDrop: 1536
09:11:52.484 [parallel-2] INFO com.example.test.Example8_3 - # onBackpressureDrop: 1537
09:11:52.486 [parallel-2] INFO com.example.test.Example8_3 - # onBackpressureDrop: 1538
09:11:52.487 [parallel-2] INFO com.example.test.Example8_3 - # onBackpressureDrop: 1539
09:11:52.488 [parallel-2] INFO com.example.test.Example8_3 - # onBackpressureDrop: 1540
09:11:52.488 [parallel-2] INFO com.example.test.Example8_3 - # onBackpressureDrop: 1541
09:11:52.489 [parallel-1] INFO com.example.test.Example8_3 - # onNext: 1157
09:11:52.490 [parallel-2] INFO com.example.test.Example8_3 - # onBackpressureDrop: 1542
~~~

* 256 ~ 1536 까지 drop 발생
* Subscriber 에서 1157 부터 처리함

#### LATEST 전략

* Publisher 가 Downstream 으로 전달할 데이터가 버퍼에 가득찰 경우, 버퍼 밖에서 대기중인 데이터중에 **가장 나중에 emit 된 데이터부터 버퍼에 채움**

~~~Java
public class Example8_4 {

    public static void main(String[] args) throws Exception{
        Flux.interval(Duration.ofMillis(1L))
                .onBackpressureLatest()
                .doOnNext(data -> log.info("# doOnNext: {}", data))
                .publishOn(Schedulers.parallel())
                .subscribe(data -> {
                            try {
                                Thread.sleep(5L);
                            } catch (InterruptedException e) {}

                            log.info("# onNext: {}", data);
                        },
                        error -> log.error("# onError"));
        Thread.sleep(2000L);
    }
}
~~~

~~~
09:15:55.312 [parallel-2] INFO com.example.test.Example8_4 - # doOnNext: 253
09:15:55.313 [parallel-2] INFO com.example.test.Example8_4 - # doOnNext: 254
09:15:55.313 [parallel-1] INFO com.example.test.Example8_4 - # onNext: 40
09:15:55.314 [parallel-2] INFO com.example.test.Example8_4 - # doOnNext: 255
09:15:55.319 [parallel-1] INFO com.example.test.Example8_4 - # onNext: 41
09:15:55.326 [parallel-1] INFO com.example.test.Example8_4 - # onNext: 42
09:15:55.333 [parallel-1] INFO com.example.test.Example8_4 - # onNext: 43
...
09:15:56.242 [parallel-1] INFO com.example.test.Example8_4 - # onNext: 189
09:15:56.249 [parallel-1] INFO com.example.test.Example8_4 - # onNext: 190
09:15:56.255 [parallel-1] INFO com.example.test.Example8_4 - # onNext: 191
09:15:56.260 [parallel-1] INFO com.example.test.Example8_4 - # doOnNext: 1196
09:15:56.261 [parallel-1] INFO com.example.test.Example8_4 - # doOnNext: 1201
09:15:56.262 [parallel-1] INFO com.example.test.Example8_4 - # doOnNext: 1202
09:15:56.262 [parallel-1] INFO com.example.test.Example8_4 - # doOnNext: 1203
...
09:15:56.655 [parallel-1] INFO com.example.test.Example8_4 - # onNext: 254
09:15:56.661 [parallel-1] INFO com.example.test.Example8_4 - # onNext: 255
09:15:56.668 [parallel-1] INFO com.example.test.Example8_4 - # onNext: 1196
09:15:56.674 [parallel-1] INFO com.example.test.Example8_4 - # onNext: 1201
09:15:56.680 [parallel-1] INFO com.example.test.Example8_4 - # onNext: 1202
~~~

* Publisher 에서 255 이후 1196 을 emit 함
* Publisher 에서 256 ~ 1195 데이터가 없음

#### Buffer 전략

세가지 방식 지원
* 버퍼의 데이터를 폐기하지 않고 버퍼링하는 전략
* 버퍼가 가득차면 버퍼 내의 데이터를 폐기하는 전략
* 버퍼가 가득 차면 에러를 발생시키는 전략

##### Buffer DROP_LATEST 전략

* 가장 나중에 버퍼 안에 채워진 데이터(버퍼 뒤를 비움)를 Drop 한후, 이렇게 확보된 공간에 emit 된 데이터를 채움

~~~Java
public class Example8_5 {

    public static void main(String[] args) throws Exception{
        Flux.interval(Duration.ofMillis(300L))
                .onBackpressureBuffer(
                        2,
                        dropped -> log.info("# overflow & dropped: {}", dropped),
                        BufferOverflowStrategy.DROP_LATEST
                )
                .doOnNext(data -> log.info("# doOnNext: {}", data))
                .publishOn(Schedulers.parallel(), false, 1)
                .subscribe(data -> {
                            try {
                                Thread.sleep(1000L);
                            } catch (InterruptedException e) {}

                            log.info("# onNext: {}", data);
                        },
                        error -> log.error("# onError"));
        Thread.sleep(6000L);
    }
}
~~~

~~~
09:27:26.471 [parallel-2] INFO com.example.test.Example8_5 - # doOnNext: 0            // publisher
09:27:27.373 [parallel-2] INFO com.example.test.Example8_5 - # overflow & dropped: 3
09:27:27.485 [parallel-1] INFO com.example.test.Example8_5 - # onNext: 0
09:27:27.486 [parallel-1] INFO com.example.test.Example8_5 - # doOnNext: 1            // publisher
09:27:27.968 [parallel-2] INFO com.example.test.Example8_5 - # overflow & dropped: 5
09:27:28.270 [parallel-2] INFO com.example.test.Example8_5 - # overflow & dropped: 6
09:27:28.487 [parallel-1] INFO com.example.test.Example8_5 - # onNext: 1
09:27:28.488 [parallel-1] INFO com.example.test.Example8_5 - # doOnNext: 2            // publisher
09:27:28.867 [parallel-2] INFO com.example.test.Example8_5 - # overflow & dropped: 8
09:27:29.170 [parallel-2] INFO com.example.test.Example8_5 - # overflow & dropped: 9
09:27:29.469 [parallel-2] INFO com.example.test.Example8_5 - # overflow & dropped: 10
09:27:29.494 [parallel-1] INFO com.example.test.Example8_5 - # onNext: 2
09:27:29.496 [parallel-1] INFO com.example.test.Example8_5 - # doOnNext: 4            // publisher
09:27:30.068 [parallel-2] INFO com.example.test.Example8_5 - # overflow & dropped: 12
09:27:30.367 [parallel-2] INFO com.example.test.Example8_5 - # overflow & dropped: 13
09:27:30.499 [parallel-1] INFO com.example.test.Example8_5 - # onNext: 4
09:27:30.500 [parallel-1] INFO com.example.test.Example8_5 - # doOnNext: 7            // publisher
09:27:30.970 [parallel-2] INFO com.example.test.Example8_5 - # overflow & dropped: 15
~~~

* publisher 에서 0,1,2,4,7 emit 됨
* 3,5,6 drop 됨

##### Buffer DROP_OLDEST 전략

* 버퍼 안에 채워진 데이터 중에서 가장 오래된 데이터를 Drop (버퍼 앞을 비움)하여 폐기한 후, 확보된 공간에 emit 된 데이터를 채움

~~~Java
public class Example8_6 {

    public static void main(String[] args) throws Exception{
        Flux.interval(Duration.ofMillis(300L))
                .onBackpressureBuffer(
                        2,
                        dropped -> log.info("# overflow & dropped: {}", dropped),
                        BufferOverflowStrategy.DROP_OLDEST
                )
                .doOnNext(data -> log.info("# doOnNext: {}", data))
                .publishOn(Schedulers.parallel(), false, 1)
                .subscribe(data -> {
                            try {
                                Thread.sleep(1000L);
                            } catch (InterruptedException e) {}

                            log.info("# onNext: {}", data);
                        },
                        error -> log.error("# onError"));
        Thread.sleep(6000L);
    }
}
~~~

~~~
09:32:03.051 [parallel-2] INFO com.example.test.Example8_6 - # doOnNext: 0            // publisher
09:32:03.957 [parallel-2] INFO com.example.test.Example8_6 - # overflow & dropped: 1
09:32:04.063 [parallel-1] INFO com.example.test.Example8_6 - # onNext: 0
09:32:04.064 [parallel-1] INFO com.example.test.Example8_6 - # doOnNext: 2            // publisher
09:32:04.551 [parallel-2] INFO com.example.test.Example8_6 - # overflow & dropped: 3
09:32:04.851 [parallel-2] INFO com.example.test.Example8_6 - # overflow & dropped: 4
09:32:05.065 [parallel-1] INFO com.example.test.Example8_6 - # onNext: 2
09:32:05.066 [parallel-1] INFO com.example.test.Example8_6 - # doOnNext: 5            // publisher
09:32:05.453 [parallel-2] INFO com.example.test.Example8_6 - # overflow & dropped: 6
09:32:05.752 [parallel-2] INFO com.example.test.Example8_6 - # overflow & dropped: 7
09:32:06.054 [parallel-2] INFO com.example.test.Example8_6 - # overflow & dropped: 8
09:32:06.071 [parallel-1] INFO com.example.test.Example8_6 - # onNext: 5
09:32:06.072 [parallel-1] INFO com.example.test.Example8_6 - # doOnNext: 9            // publisher
09:32:06.653 [parallel-2] INFO com.example.test.Example8_6 - # overflow & dropped: 10
09:32:06.954 [parallel-2] INFO com.example.test.Example8_6 - # overflow & dropped: 11
09:32:07.077 [parallel-1] INFO com.example.test.Example8_6 - # onNext: 9
09:32:07.078 [parallel-1] INFO com.example.test.Example8_6 - # doOnNext: 12            // publisher
09:32:07.554 [parallel-2] INFO com.example.test.Example8_6 - # overflow & dropped: 13
09:32:07.852 [parallel-2] INFO com.example.test.Example8_6 - # overflow & dropped: 14
~~~

* Publisher emit 0,2,5,9,12

# 9. Sinks

## 9.1 Sinks 란?

> Sinks are constructs through which Reactive Streams signals can be programmatically pushed, with Flux of Mono semantics.
- Sinks 는 리액티브 스트림즈의 Signal 을 프로그래밍 방식으로 푸시할수 있는 구조이며, Flux 또는 Mono 의 의미 체계를 가진다
- Sinks 를 사용하면 프로그래밍 코드를 통해 명시적으로 Signal 을 전송할수 있음
- Sinks 는 멀티스레드 방식으로 스레드 안정성(Thread Safety)을 보장해줌

- Reactor 에서 프로그래밍 방식으로 signal 을 전송하는 일방적인 방법은 generate(), create() operator 사용

Sinks vs Operator 에서 signal 전송
- generate(), create() 는 싱글 스레드 기반, Sinks 는 멀티스레드 방식으로 스레드 안정성을 보장해줌


~~~Java
public class Example9_1 {

    public static void main(String[] args) throws Exception{
        int tasks = 6;
        Flux.create((FluxSink<String> sink) ->
                IntStream.range(1, tasks).forEach(n -> sink.next(doTask(n))))
                .subscribeOn(Schedulers.boundedElastic())
                .doOnNext(n -> log.info("# create(): {} ", n))
                .publishOn(Schedulers.parallel())
                .map(result -> result + " success!")
                .doOnNext(n -> log.info("# map(): {} ", n))
                .publishOn(Schedulers.parallel())
                .subscribe(data -> log.info("# onNext: {}", data))
        ;

        Thread.sleep(500L);
    }

    public static String doTask(int taskNumber) {
        return "task " + taskNumber + " result";
    }
}
~~~

~~~
09:47:46.442 [boundedElastic-1] INFO com.example.test.Example9_1 - # create(): task 1 result 
09:47:46.443 [boundedElastic-1] INFO com.example.test.Example9_1 - # create(): task 2 result 
09:47:46.443 [boundedElastic-1] INFO com.example.test.Example9_1 - # create(): task 3 result 
09:47:46.443 [boundedElastic-1] INFO com.example.test.Example9_1 - # create(): task 4 result 
09:47:46.443 [boundedElastic-1] INFO com.example.test.Example9_1 - # create(): task 5 result 
09:47:46.443 [parallel-2] INFO com.example.test.Example9_1 - # map(): task 1 result success! 
09:47:46.443 [parallel-2] INFO com.example.test.Example9_1 - # map(): task 2 result success! 
09:47:46.443 [parallel-2] INFO com.example.test.Example9_1 - # map(): task 3 result success! 
09:47:46.443 [parallel-2] INFO com.example.test.Example9_1 - # map(): task 4 result success! 
09:47:46.443 [parallel-1] INFO com.example.test.Example9_1 - # onNext: task 1 result success!
09:47:46.443 [parallel-2] INFO com.example.test.Example9_1 - # map(): task 5 result success! 
09:47:46.443 [parallel-1] INFO com.example.test.Example9_1 - # onNext: task 2 result success!
09:47:46.443 [parallel-1] INFO com.example.test.Example9_1 - # onNext: task 3 result success!
09:47:46.443 [parallel-1] INFO com.example.test.Example9_1 - # onNext: task 4 result success!
09:47:46.443 [parallel-1] INFO com.example.test.Example9_1 - # onNext: task 5 result success!
~~~

* 총 3개의 스레드가 생성
  * doTask() -> boundedElastic-1
  * map() -> parallel-2
  * onNext() -> parallel-1
* Reactor Sequence 를 단계적으로 나누어 여러 개의 스레드로 처리할수 있음
* Sinks 를 사용하면 doTask() 작업을 싱글스레드가 아닌 여러개의 스레드에서 처리할 수 있음

~~~java
public class Example9_2 {

    public static void main(String[] args) throws Exception {
        int tasks = 6;
        Sinks.Many<String> unicastSink = Sinks.many().unicast().onBackpressureBuffer();
        Flux<String> fluxView = unicastSink.asFlux();

        IntStream.range(1, tasks).forEach(n -> {
            try {
                Thread thread = new Thread(() -> {
                    unicastSink.emitNext(doTask(n), Sinks.EmitFailureHandler.FAIL_FAST);
                    log.info("# create(): {} ", n);
                });
                thread.start();

                Thread.sleep(100L);
            } catch (InterruptedException e) {
                log.error(e.getMessage(), e);
            }
        });

        fluxView
                .publishOn(Schedulers.parallel())
                .map(result -> result + " success!")
                .doOnNext(n -> log.info("# map(): {} ", n))
                .publishOn(Schedulers.parallel())
                .subscribe(data -> log.info("# onNext: {}", data))
        ;

        Thread.sleep(500L);
    }

    public static String doTask(int taskNumber) {
        return "task " + taskNumber + " result";
    }
}
~~~

~~~
09:59:15.572 [Thread-0] INFO com.example.test.Example9_2 - # create(): 1
09:59:15.673 [Thread-1] INFO com.example.test.Example9_2 - # create(): 2
09:59:15.779 [Thread-2] INFO com.example.test.Example9_2 - # create(): 3
09:59:15.879 [Thread-3] INFO com.example.test.Example9_2 - # create(): 4
09:59:15.983 [Thread-4] INFO com.example.test.Example9_2 - # create(): 5
09:59:16.117 [parallel-2] INFO com.example.test.Example9_2 - # map(): task 1 result success!
09:59:16.117 [parallel-2] INFO com.example.test.Example9_2 - # map(): task 2 result success!
09:59:16.117 [parallel-2] INFO com.example.test.Example9_2 - # map(): task 3 result success!
09:59:16.117 [parallel-1] INFO com.example.test.Example9_2 - # onNext: task 1 result success!
09:59:16.117 [parallel-2] INFO com.example.test.Example9_2 - # map(): task 4 result success!
09:59:16.117 [parallel-1] INFO com.example.test.Example9_2 - # onNext: task 2 result success!
09:59:16.117 [parallel-2] INFO com.example.test.Example9_2 - # map(): task 5 result success!
09:59:16.117 [parallel-1] INFO com.example.test.Example9_2 - # onNext: task 3 result success!
09:59:16.117 [parallel-1] INFO com.example.test.Example9_2 - # onNext: task 4 result success!
09:59:16.117 [parallel-1] INFO com.example.test.Example9_2 - # onNext: task 5 result success!
~~~

* doTask 가 Thread-0 ~ Thread-4 까지 각각 다른 스레드에서 실행되는 것을 볼수 있음

## 9.2 Sinks 종류 및 특징

### Sinks.One
- 한 건의 데이터를 전송

~~~java
// Reactor 3.4.19 기준
public final class Sinks {
    public static <T> Sinks.One<T> one() {
		return SinksSpecs.DEFAULT_SINKS.one();
	}
	
	public interface EmitFailureHandler {
		EmitFailureHandler FAIL_FAST = (signalType, emission) -> false; // 람다표현식으로 EmitFailureHandler 인터페이스 구현체 
		static EmitFailureHandler busyLooping(Duration duration){
			return new OptimisticEmitFailureHandler(duration);
		}
		boolean onEmitFailure(SignalType signalType, EmitResult emitResult);
	}
}
~~~

* FAIL_FAST
  * 에러가 발생했을 때 재시도를 하지 않고 즉시 실패 처리
  * 빠른 실패 처리를 함으로써 스레드 간의 경합 등으로 발생하는 교착 상태 등을 미연에 방지할수 있고, 이는 결과적으로 thread safety 를 보장
  * 처음으로 emit 한 데이터만 정상적으로 emit 되고 나머지 데이터들은 drop 됨

~~~java
public class Example9_4 {

    public static void main(String[] args) throws Exception {
        Sinks.One<String> sinkOne = Sinks.one();
        Mono<String> mono = sinkOne.asMono();

        sinkOne.emitValue("Hello Reactor", Sinks.EmitFailureHandler.FAIL_FAST);
//        sinkOne.emitValue("Hi Reactor", Sinks.EmitFailureHandler.FAIL_FAST);

        mono.subscribe(data -> log.info("# Subscriber1: {}", data));
        mono.subscribe(data -> log.info("# Subscriber2: {}", data));
    }
}
~~~

* Hello Reactor 만 emit
~~~
10:07:55.220 [main] INFO com.example.test.Example9_4 - # Subscriber1: Hello Reactor
10:07:55.221 [main] INFO com.example.test.Example9_4 - # Subscriber2: Hello Reactor
~~~

* Hello, Hi Reactor 둘다 emit
  * Hi Reactor 가 drop 됨 
~~~
10:13:42.994 [main] DEBUG reactor.core.publisher.Operators - onNextDropped: Hi Reactor
10:13:42.996 [main] INFO com.example.test.Example9_4 - # Subscriber1: Hello Reactor
10:13:42.997 [main] INFO com.example.test.Example9_4 - # Subscriber2: Hello Reactor
~~~


### Sinks.Many
- 여러건의 데이터를 여러가지 방식으로 전송

~~~java
// Reactor 3.4.19 기준
public final class Sinks {
    public static ManySpec many() {
		return SinksSpecs.DEFAULT_SINKS.many();
	}
	
	public interface ManySpec {
		UnicastSpec unicast();
		MulticastSpec multicast();
		MulticastReplaySpec replay();
	}
}
~~~

#### UnicastSpec
  * 단 하나의 Subscriber 에게만 데이터를 emit

~~~java
public class Example9_5 {

    public static void main(String[] args) throws Exception {
        Sinks.Many<Integer> unicastSink = Sinks.many().unicast().onBackpressureBuffer();
        Flux<Integer> fluxView = unicastSink.asFlux();

        unicastSink.emitNext(1, Sinks.EmitFailureHandler.FAIL_FAST);
        unicastSink.emitNext(2, Sinks.EmitFailureHandler.FAIL_FAST);

        fluxView.subscribe(data -> log.info("# Subscriber1: {}", data));

        unicastSink.emitNext(3, Sinks.EmitFailureHandler.FAIL_FAST);

//        fluxView.subscribe(data -> log.info("# Subscriber2: {}", data));
    }
}
~~~

* 한번 subscribe 했을 때
~~~
10:19:17.684 [main] INFO com.example.test.Example9_5 - # Subscriber1: 1
10:19:17.685 [main] INFO com.example.test.Example9_5 - # Subscriber1: 2
10:19:17.685 [main] INFO com.example.test.Example9_5 - # Subscriber1: 3
~~~

* 두번 subscribe 했을 때
  * unicast 스펙이 단 하나의 subscriber 에게만 데이터를 emit 하는 것이기 때문에 두번 전달되면 에러 발생 
~~~
Caused by: java.lang.IllegalStateException: UnicastProcessor allows only a single Subscriber
~~~

#### MulticastSpec
* 하나 이상의 Subscriber 에게 데이터를 emit

~~~
public class Example9_6 {

    public static void main(String[] args) throws Exception {
        Sinks.Many<Integer> multicastSink = Sinks.many().multicast().onBackpressureBuffer();
        Flux<Integer> fluxView = multicastSink.asFlux();

        multicastSink.emitNext(1, Sinks.EmitFailureHandler.FAIL_FAST);
        multicastSink.emitNext(2, Sinks.EmitFailureHandler.FAIL_FAST);

        fluxView.subscribe(data -> log.info("# Subscriber1: {}", data));  // 첫번째 구독이 발생한 시점
        fluxView.subscribe(data -> log.info("# Subscriber2: {}", data));

        multicastSink.emitNext(3, Sinks.EmitFailureHandler.FAIL_FAST);
    }
}
~~~


~~~
10:23:02.058 [main] INFO com.example.test.Example9_6 - # Subscriber1: 1
10:23:02.059 [main] INFO com.example.test.Example9_6 - # Subscriber1: 2
10:23:02.059 [main] INFO com.example.test.Example9_6 - # Subscriber1: 3
10:23:02.059 [main] INFO com.example.test.Example9_6 - # Subscriber2: 3
~~~

* 처리 과정
  * 1,2 가 emit 됨
  * subscriber1 이 등록되어, 1, 2 처리
  * subscriber2 가 등록된 시점에 subscribe1 에 의해 1,2 가 처리되어 subscribe2 에서 1,2 처리되지 않음
  * 3 emit 되면 subscriber1, subscribe2 에서 처리
* 결론, subscriber1 은 데이터 1,2,3 받음, subscriber2 는 3만 받음
  * Sinks 가 Publisher 역할을 할 경우, 기본적으로 Hot Publisher 로 동작
  * onBackpressureBuffer() 메서드는 Warm up 의 특징을 가지는 Hot Sequence 로 동작
    * 이로 인해 첫번째 구독이 발생한 시점에 Downstream 쪽으로 데이터 전달됨


### MulticastReplaySpec
* emit 된 데이터를 다시 replay 해서 구독 전에 이미 emit 된 데이터라도 subscriber 가 전달 받을 수 있게함
* all()
  * 구독 전에 이미 emit 된 데이터가 있더라도 처음 emit 된 데이터부터 모든 데이터들이 subscriber 에게 전달됨
* limit()
  * emit 된 데이터중 입력한 개수만큼 가장 나중에 emit 된 데이터부터 subscriber 에게 전달
  * 즉, emit 된 데이터중에서 2개만 뒤로 돌려서(replay) 전달

~~~java
public class Example9_7 {

    public static void main(String[] args) throws Exception {
        Sinks.Many<Integer> multicastSink = Sinks.many().replay().limit(2);
        Flux<Integer> fluxView = multicastSink.asFlux();

        multicastSink.emitNext(1, Sinks.EmitFailureHandler.FAIL_FAST);
        multicastSink.emitNext(2, Sinks.EmitFailureHandler.FAIL_FAST);
        multicastSink.emitNext(3, Sinks.EmitFailureHandler.FAIL_FAST);

        fluxView.subscribe(data -> log.info("# Subscriber1: {}", data));

        multicastSink.emitNext(4, Sinks.EmitFailureHandler.FAIL_FAST);

        fluxView.subscribe(data -> log.info("# Subscriber2: {}", data));

    }
}
~~~

~~~
10:32:04.761 [main] INFO com.example.test.Example9_7 - # Subscriber1: 2
10:32:04.764 [main] INFO com.example.test.Example9_7 - # Subscriber1: 3
10:32:04.764 [main] INFO com.example.test.Example9_7 - # Subscriber1: 4
10:32:04.764 [main] INFO com.example.test.Example9_7 - # Subscriber2: 3
10:32:04.764 [main] INFO com.example.test.Example9_7 - # Subscriber2: 4
~~~

* 처리 과정
  * 1,2,3 emit 된 후 subscriber1 가 등록되면 replay limit 에 의해 2,3 만 처리됨
  * 4 가 emit 되서 Subscriber1 에서 4를 처리
  * subscriber2 가 등록되면 replay limit 에 의해 3,4 만 처리됨


# 10.Scheduler

* Scheduler
  * Reactor 에서 사용되는 Scheduler 는 Reactor Sequence 에서 사용되는 스레드를 관리해주는 관리자 역할 

## 10.1 Thread의 개념 이해

### 물리적인 스레드(Physical Thread)
- 하나의 core 는 두개의 thread 를 포함하고 있는데, 이 thread 가 물리적인 스레드임
- physical thread 는 병렬성(parallelism) 과 연관이 있으며
  - 병렬성은 physical thread 가 실제로 동시에 실행되기 대문에 여러 작업을 동시에 처리
![cpu](cpu.png){ border-effect="line" thumbnail="true" width="321"}

### 논리적인 스레드(Logical Thread)
- 스프트웨어 적으로 생성되는 스레드를 의미함
- logical thread 는 동시성(Concurrency) 와 관련이 있음
  - 동시성은 용어 때문에 동시에 실행된다고 생각할수 있지만, 동시에 실행되는 것처럼 보이는 것 
![thread](thread.png){ border-effect="line" thumbnail="true" width="321"}


## 10.2 Scheduler 란?

- Reactor 의 Scheduler 는 비동기 프로그래밍을 위해 사용되는 스레드를 관리해 주는 역할
- java 에서 멀티스레드를 제어하는게 쉽지 않음.
  - 스레드간의 경쟁조건(race condition) 등을 고려하다보면 코드의 복잡도가 높아지고 결과적으로 예상치 못한 오류가 발생할 가능성이 높음
- Scheduler 가 thread 의 제어를 대신해 주기 때문에 개발자가 thread 를 제어해야 하는 부담에서 벗어날수 있음


## 10.3 Scheduler 를 위한 전용 Operator

### subScribeOn()
- 구독이 발생한 직후 실행될 스레드를 지정
- Publisher 의 동작을 수행하기 위한 thread
- 체인상 어떤 위치에 있든 간에 구독 시점 직후, 즉 publisher 가 데이터를 emit 하기 전에 실행스레드를 변경

~~~java
public class Example10_1 {

    public static void main(String[] args) throws Exception {
        Flux.fromArray(new Integer[] {1,3,5,7})
                .subscribeOn(Schedulers.boundedElastic())
                .doOnNext(data -> log.info("# doOnNext: {}", data))
                .doOnSubscribe(subscription -> log.info("# doOnSubscribe"))
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}
~~~

~~~
11:41:45.656 [main] INFO com.example.test.Example10_1 - # doOnSubscribe
11:41:45.661 [boundedElastic-1] INFO com.example.test.Example10_1 - # doOnNext: 1
11:41:45.662 [boundedElastic-1] INFO com.example.test.Example10_1 - # onNext: 1
11:41:45.662 [boundedElastic-1] INFO com.example.test.Example10_1 - # doOnNext: 3
11:41:45.662 [boundedElastic-1] INFO com.example.test.Example10_1 - # onNext: 3
11:41:45.662 [boundedElastic-1] INFO com.example.test.Example10_1 - # doOnNext: 5
11:41:45.662 [boundedElastic-1] INFO com.example.test.Example10_1 - # onNext: 5
11:41:45.662 [boundedElastic-1] INFO com.example.test.Example10_1 - # doOnNext: 7
11:41:45.662 [boundedElastic-1] INFO com.example.test.Example10_1 - # onNext: 7
~~~

* doOnSubscribe - main thread 에서 실행
  * 최초 실행 스레드가 main 스레드이기 때문
* subscribeOn 이 추가됨으로써 구독이 발생한 직후부터 main thread 에서 실행되지 않음
  * doOnNext - boundedElastic-1 thread 에서 실행
* onNext - operator 체인상에서 특별히 다른 scheduler 를 지정하지 않았기 때문에, boundedElastic-1 이 사용됨 


### publishOn()
- Downstream 으로 signal 을 전송할때 실행되는 스레드를 제어하는 역할
- publishOn 을 기준으로 아래쪽인 Downstream 의 실행 스레드를 변경함
- 한개 이상 사용할수 있으며, 실행 스레드를 목적에 맞게 적절하게 분리할수 있음

~~~java
public class Example10_2 {

    public static void main(String[] args) throws Exception {
        Flux.fromArray(new Integer[] {1,3,5,7})
                .subscribeOn(Schedulers.boundedElastic())
                .doOnNext(data -> log.info("# doOnNext: {}", data))
                .doOnSubscribe(subscription -> log.info("# doOnSubscribe"))
                .publishOn(Schedulers.parallel())
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}
~~~

* onNext - publishOn 에 scheduler 가 parallel 로 지정되어 parallel-1 스레드에서 실행됨
~~~
13:14:21.643 [main] INFO com.example.test.Example10_2 - # doOnSubscribe
13:14:21.652 [boundedElastic-1] INFO com.example.test.Example10_2 - # doOnNext: 1
13:14:21.653 [boundedElastic-1] INFO com.example.test.Example10_2 - # doOnNext: 3
13:14:21.653 [boundedElastic-1] INFO com.example.test.Example10_2 - # doOnNext: 5
13:14:21.653 [boundedElastic-1] INFO com.example.test.Example10_2 - # doOnNext: 7
13:14:21.653 [parallel-1] INFO com.example.test.Example10_2 - # onNext: 1
13:14:21.653 [parallel-1] INFO com.example.test.Example10_2 - # onNext: 3
13:14:21.653 [parallel-1] INFO com.example.test.Example10_2 - # onNext: 5
13:14:21.653 [parallel-1] INFO com.example.test.Example10_2 - # onNext: 7
~~~

### parallel()

- subscribeOn(), publishOn() 은 동시성을 가지는 논리적인 스레드에 해당
- parallel() 은 병렬성을 가지는 physical thread 에 해당
  - 라운드 로빈(round robin) 방식으로 physical thread 만큼의 스레드를 병렬로 실행
  - ex> 4 core 8 thread cpu 라면 8 개의 스레드를 병렬로 실행


~~~java
public class Example10_3 {

    public static void main(String[] args) throws Exception {
        Flux.fromArray(new Integer[] {1,3,5,7,9,11,13,15,17,19})
                .parallel()
                .runOn(Schedulers.parallel())
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}
~~~

~~~
13:22:07.374 [parallel-2] INFO com.example.test.Example10_3 - # onNext: 3
13:22:07.375 [parallel-9] INFO com.example.test.Example10_3 - # onNext: 17
13:22:07.375 [parallel-5] INFO com.example.test.Example10_3 - # onNext: 9
13:22:07.375 [parallel-7] INFO com.example.test.Example10_3 - # onNext: 13
13:22:07.375 [parallel-6] INFO com.example.test.Example10_3 - # onNext: 11
13:22:07.375 [parallel-3] INFO com.example.test.Example10_3 - # onNext: 5
13:22:07.374 [parallel-4] INFO com.example.test.Example10_3 - # onNext: 7
13:22:07.375 [parallel-8] INFO com.example.test.Example10_3 - # onNext: 15
13:22:07.375 [parallel-1] INFO com.example.test.Example10_3 - # onNext: 1
13:22:07.375 [parallel-10] INFO com.example.test.Example10_3 - # onNext: 19
~~~


- physical thread 를 모두 사용할 필요가 없는 경우 thread 를 지정

~~~java
public class Example10_3 {

    public static void main(String[] args) throws Exception {
        Flux.fromArray(new Integer[] {1,3,5,7,9,11,13,15,17,19})
                .parallel(4)
                .runOn(Schedulers.parallel())
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}
~~~

~~~
13:23:54.648 [parallel-2] INFO com.example.test.Example10_3 - # onNext: 3
13:23:54.648 [parallel-1] INFO com.example.test.Example10_3 - # onNext: 1
13:23:54.648 [parallel-3] INFO com.example.test.Example10_3 - # onNext: 5
13:23:54.650 [parallel-2] INFO com.example.test.Example10_3 - # onNext: 11
13:23:54.648 [parallel-4] INFO com.example.test.Example10_3 - # onNext: 7
13:23:54.650 [parallel-3] INFO com.example.test.Example10_3 - # onNext: 13
13:23:54.650 [parallel-4] INFO com.example.test.Example10_3 - # onNext: 15
13:23:54.650 [parallel-1] INFO com.example.test.Example10_3 - # onNext: 9
13:23:54.650 [parallel-2] INFO com.example.test.Example10_3 - # onNext: 19
13:23:54.650 [parallel-1] INFO com.example.test.Example10_3 - # onNext: 17
~~~

## 10.4 publishOn() 과 subscribeOn() 의 동작 이해

* publishOn(), subscribeOn() 을 사용하지 않는 경우 main thread 사용
  * fromArray -> main thread
  * filter -> parallel-1
  * map -> parallel-1
  * subscribe -> parallel-1

~~~java
public class Example10_4 {

    public static void main(String[] args) throws Exception {
        Flux.fromArray(new Integer[] {1,3,5,7})
                .doOnNext(data -> log.info("# doOnNext fromArray: {}", data))
                .filter(data -> data > 3)
                .doOnNext(data -> log.info("# doOnNext filter: {}", data))
                .map(data -> data * 10)
                .doOnNext(data -> log.info("# doOnNext map: {}", data))
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}
~~~

~~~
13:28:02.580 [main] INFO com.example.test.Example10_4 - # doOnNext fromArray: 1
13:28:02.582 [main] INFO com.example.test.Example10_4 - # doOnNext fromArray: 3
13:28:02.582 [main] INFO com.example.test.Example10_4 - # doOnNext fromArray: 5
13:28:02.582 [main] INFO com.example.test.Example10_4 - # doOnNext filter: 5
13:28:02.582 [main] INFO com.example.test.Example10_4 - # doOnNext map: 50
13:28:02.582 [main] INFO com.example.test.Example10_4 - # onNext: 50
13:28:02.582 [main] INFO com.example.test.Example10_4 - # doOnNext fromArray: 7
13:28:02.582 [main] INFO com.example.test.Example10_4 - # doOnNext filter: 7
13:28:02.582 [main] INFO com.example.test.Example10_4 - # doOnNext map: 70
13:28:02.582 [main] INFO com.example.test.Example10_4 - # onNext: 70
~~~


* filter 앞 publishOn parallel 사용
  * fromArray -> main thread
  * filter -> parallel-1
  * map -> parallel-1
  * subscribe -> parallel-1

~~~java
public class Example10_4 {

    public static void main(String[] args) throws Exception {
        Flux.fromArray(new Integer[] {1,3,5,7})
                .doOnNext(data -> log.info("# doOnNext fromArray: {}", data))
                .publishOn(Schedulers.parallel())
                .filter(data -> data > 3)
                .doOnNext(data -> log.info("# doOnNext filter: {}", data))
                .map(data -> data * 10)
                .doOnNext(data -> log.info("# doOnNext map: {}", data))
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}
~~~

~~~
13:29:00.283 [main] INFO com.example.test.Example10_4 - # doOnNext fromArray: 1
13:29:00.287 [main] INFO com.example.test.Example10_4 - # doOnNext fromArray: 3
13:29:00.287 [main] INFO com.example.test.Example10_4 - # doOnNext fromArray: 5
13:29:00.287 [main] INFO com.example.test.Example10_4 - # doOnNext fromArray: 7
13:29:00.287 [parallel-1] INFO com.example.test.Example10_4 - # doOnNext filter: 5
13:29:00.288 [parallel-1] INFO com.example.test.Example10_4 - # doOnNext map: 50
13:29:00.289 [parallel-1] INFO com.example.test.Example10_4 - # onNext: 50
13:29:00.289 [parallel-1] INFO com.example.test.Example10_4 - # doOnNext filter: 7
13:29:00.289 [parallel-1] INFO com.example.test.Example10_4 - # doOnNext map: 70
13:29:00.289 [parallel-1] INFO com.example.test.Example10_4 - # onNext: 70
~~~

* filter 앞 publishOn parallel 사용
* map 앞 publishOn parallel 사용
  * fromArray -> main thread
  * filter -> parallel-2
  * map -> parallel-1
  * subscribe -> parallel-1

~~~java
public class Example10_4 {

    public static void main(String[] args) throws Exception {
        Flux.fromArray(new Integer[] {1,3,5,7})
                .doOnNext(data -> log.info("# doOnNext fromArray: {}", data))
                .publishOn(Schedulers.parallel())
                .filter(data -> data > 3)
                .doOnNext(data -> log.info("# doOnNext filter: {}", data))
                .publishOn(Schedulers.parallel())
                .map(data -> data * 10)
                .doOnNext(data -> log.info("# doOnNext map: {}", data))
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}
~~~

~~~
13:32:22.743 [main] INFO com.example.test.Example10_4 - # doOnNext fromArray: 1
13:32:22.747 [main] INFO com.example.test.Example10_4 - # doOnNext fromArray: 3
13:32:22.747 [main] INFO com.example.test.Example10_4 - # doOnNext fromArray: 5
13:32:22.747 [main] INFO com.example.test.Example10_4 - # doOnNext fromArray: 7
13:32:22.747 [parallel-2] INFO com.example.test.Example10_4 - # doOnNext filter: 5
13:32:22.748 [parallel-2] INFO com.example.test.Example10_4 - # doOnNext filter: 7
13:32:22.748 [parallel-1] INFO com.example.test.Example10_4 - # doOnNext map: 50
13:32:22.748 [parallel-1] INFO com.example.test.Example10_4 - # onNext: 50
13:32:22.749 [parallel-1] INFO com.example.test.Example10_4 - # doOnNext map: 70
13:32:22.749 [parallel-1] INFO com.example.test.Example10_4 - # onNext: 70
~~~

* filter 앞 subscribeOn boundedElastic 사용
* map 앞 publishOn parallel 사용
    * fromArray -> boundedElastic-1
    * filter -> boundedElastic-1
    * map -> parallel-1
    * subscribe -> parallel-1

~~~java
public class Example10_4 {

    public static void main(String[] args) throws Exception {
        Flux.fromArray(new Integer[] {1,3,5,7})
                .doOnNext(data -> log.info("# doOnNext fromArray: {}", data))
                .subscribeOn(Schedulers.boundedElastic())
                .filter(data -> data > 3)
                .doOnNext(data -> log.info("# doOnNext filter: {}", data))
                .publishOn(Schedulers.parallel())
                .map(data -> data * 10)
                .doOnNext(data -> log.info("# doOnNext map: {}", data))
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}
~~~

~~~
13:35:08.061 [boundedElastic-1] INFO com.example.test.Example10_4 - # doOnNext fromArray: 1
13:35:08.062 [boundedElastic-1] INFO com.example.test.Example10_4 - # doOnNext fromArray: 3
13:35:08.062 [boundedElastic-1] INFO com.example.test.Example10_4 - # doOnNext fromArray: 5
13:35:08.062 [boundedElastic-1] INFO com.example.test.Example10_4 - # doOnNext filter: 5
13:35:08.062 [boundedElastic-1] INFO com.example.test.Example10_4 - # doOnNext fromArray: 7
13:35:08.062 [boundedElastic-1] INFO com.example.test.Example10_4 - # doOnNext filter: 7
13:35:08.062 [parallel-1] INFO com.example.test.Example10_4 - # doOnNext map: 50
13:35:08.063 [parallel-1] INFO com.example.test.Example10_4 - # onNext: 50
13:35:08.063 [parallel-1] INFO com.example.test.Example10_4 - # doOnNext map: 70
13:35:08.063 [parallel-1] INFO com.example.test.Example10_4 - # onNext: 70
~~~

## 10.5 Scheduler 의 종류

### Scheduler.immediate()
- 별도의 스레드를 추가적으로 생성하지 않고, 현재 스레드에서 작업을 처리하고자 할때 사용
  * fromArray -> main
  * filter -> parallel-1
  * map -> parallel-1
  * subscribe -> parallel-1

~~~java
public class Example10_4 {

    public static void main(String[] args) throws Exception {
        Flux.fromArray(new Integer[] {1,3,5,7})
                .doOnNext(data -> log.info("# doOnNext fromArray: {}", data))
                .publishOn(Schedulers.parallel())                           // parallel
                .filter(data -> data > 3)
                .doOnNext(data -> log.info("# doOnNext filter: {}", data))
                .publishOn(Schedulers.immediate())                          // immediate
                .map(data -> data * 10)
                .doOnNext(data -> log.info("# doOnNext map: {}", data))
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}
~~~

~~~
13:40:31.328 [main] INFO com.example.test.Example10_4 - # doOnNext fromArray: 1
13:40:31.330 [main] INFO com.example.test.Example10_4 - # doOnNext fromArray: 3
13:40:31.330 [main] INFO com.example.test.Example10_4 - # doOnNext fromArray: 5
13:40:31.330 [main] INFO com.example.test.Example10_4 - # doOnNext fromArray: 7
13:40:31.330 [parallel-1] INFO com.example.test.Example10_4 - # doOnNext filter: 5
13:40:31.331 [parallel-1] INFO com.example.test.Example10_4 - # doOnNext map: 50
13:40:31.331 [parallel-1] INFO com.example.test.Example10_4 - # onNext: 50
13:40:31.331 [parallel-1] INFO com.example.test.Example10_4 - # doOnNext filter: 7
13:40:31.331 [parallel-1] INFO com.example.test.Example10_4 - # doOnNext map: 70
13:40:31.331 [parallel-1] INFO com.example.test.Example10_4 - # onNext: 70
~~~

### Scheduler.single()
- 스레드 하나만 생성해서 scheduler 가 제거되 전까지 재사용하는 방식

~~~java
public class Example10_5 {

    public static void main(String[] args) throws Exception {
        doTask("task1")
                .subscribe(data -> log.info("# onNext: {}", data));

        doTask("task2")
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }

    public static Flux<Integer> doTask(String taskName) {
        return Flux.fromArray(new Integer[] {1,3,5,7})
                .doOnNext(data -> log.info("# doOnNext fromArray: {}", data))
                .publishOn(Schedulers.single())
                .filter(data -> data > 3)
                .doOnNext(data -> log.info("# doOnNext filter: {}", data))
                .map(data -> data * 10)
                .doOnNext(data -> log.info("# doOnNext map: {}", data));
    }
}
~~~

* task1, task2 - single-1 을 사용
* doTask 를 두번 호출하더라도 첫번째 호출에서 이미 생성된 스레드를 재사용함

~~~
13:44:09.552 [main] INFO com.example.test.Example10_5 - # doOnNext fromArray: 1
13:44:09.555 [main] INFO com.example.test.Example10_5 - # doOnNext fromArray: 3
13:44:09.555 [main] INFO com.example.test.Example10_5 - # doOnNext fromArray: 5
13:44:09.555 [main] INFO com.example.test.Example10_5 - # doOnNext fromArray: 7
13:44:09.556 [main] INFO com.example.test.Example10_5 - # doOnNext fromArray: 1
13:44:09.556 [main] INFO com.example.test.Example10_5 - # doOnNext fromArray: 3
13:44:09.556 [main] INFO com.example.test.Example10_5 - # doOnNext fromArray: 5
13:44:09.555 [single-1] INFO com.example.test.Example10_5 - # doOnNext filter: 5
13:44:09.557 [main] INFO com.example.test.Example10_5 - # doOnNext fromArray: 7
13:44:09.557 [single-1] INFO com.example.test.Example10_5 - # doOnNext map: 50
13:44:09.557 [single-1] INFO com.example.test.Example10_5 - # onNext: 50
13:44:09.557 [single-1] INFO com.example.test.Example10_5 - # doOnNext filter: 7
13:44:09.557 [single-1] INFO com.example.test.Example10_5 - # doOnNext map: 70
13:44:09.557 [single-1] INFO com.example.test.Example10_5 - # onNext: 70
13:44:09.558 [single-1] INFO com.example.test.Example10_5 - # doOnNext filter: 5
13:44:09.558 [single-1] INFO com.example.test.Example10_5 - # doOnNext map: 50
13:44:09.558 [single-1] INFO com.example.test.Example10_5 - # onNext: 50
13:44:09.558 [single-1] INFO com.example.test.Example10_5 - # doOnNext filter: 7
13:44:09.558 [single-1] INFO com.example.test.Example10_5 - # doOnNext map: 70
13:44:09.558 [single-1] INFO com.example.test.Example10_5 - # onNext: 70
~~~


### Scheduler.newSingle()
- Scheduler.single() 은 하나의 스레드를 재사용 하는 반면, Scheduler.newSingle() 은 호출할 때마다 새로운 스레드 하나를 생성
- deamon 스레드
  - 보조 스레드라고도 불리는데, 주 스레드가 종료되면 자동으로 종료되는 특성이 있음
  - newSingle 두 번째 파라미터에 true 를 설정해서 main 스레드 종료시 자동으로 종료되도록 설정
  - true 옵션 주지 않으면 프로그램 종료되지 않음 

~~~java
public class Example10_5 {

    public static void main(String[] args) throws Exception {
        doTask("task1")
                .subscribe(data -> log.info("# onNext: {}", data));

        doTask("task2")
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }

    public static Flux<Integer> doTask(String taskName) {
        return Flux.fromArray(new Integer[] {1,3,5,7})
                .doOnNext(data -> log.info("# doOnNext fromArray: {}", data))
                .publishOn(Schedulers.newSingle("new-single", true))
                .filter(data -> data > 3)
                .doOnNext(data -> log.info("# doOnNext filter: {}", data))
                .map(data -> data * 10)
                .doOnNext(data -> log.info("# doOnNext map: {}", data));
    }
}
~~~

* task1 - new-single-1 을 사용
* task2 - new-single-2 을 사용
* 두번재 호출에서 새로운 thread new-single-2 를 생성
~~~
13:52:34.321 [main] INFO com.example.test.Example10_5 - # doOnNext fromArray: 1
13:52:34.324 [main] INFO com.example.test.Example10_5 - # doOnNext fromArray: 3
13:52:34.325 [main] INFO com.example.test.Example10_5 - # doOnNext fromArray: 5
13:52:34.325 [main] INFO com.example.test.Example10_5 - # doOnNext fromArray: 7
13:52:34.325 [main] INFO com.example.test.Example10_5 - # doOnNext fromArray: 1
13:52:34.325 [new-single-1] INFO com.example.test.Example10_5 - # doOnNext filter: 5
13:52:34.326 [main] INFO com.example.test.Example10_5 - # doOnNext fromArray: 3
13:52:34.326 [main] INFO com.example.test.Example10_5 - # doOnNext fromArray: 5
13:52:34.326 [main] INFO com.example.test.Example10_5 - # doOnNext fromArray: 7
13:52:34.326 [new-single-1] INFO com.example.test.Example10_5 - # doOnNext map: 50
13:52:34.326 [new-single-1] INFO com.example.test.Example10_5 - # onNext: 50
13:52:34.326 [new-single-2] INFO com.example.test.Example10_5 - # doOnNext filter: 5
13:52:34.326 [new-single-1] INFO com.example.test.Example10_5 - # doOnNext filter: 7
13:52:34.326 [new-single-1] INFO com.example.test.Example10_5 - # doOnNext map: 70
13:52:34.326 [new-single-1] INFO com.example.test.Example10_5 - # onNext: 70
13:52:34.326 [new-single-2] INFO com.example.test.Example10_5 - # doOnNext map: 50
13:52:34.326 [new-single-2] INFO com.example.test.Example10_5 - # onNext: 50
13:52:34.326 [new-single-2] INFO com.example.test.Example10_5 - # doOnNext filter: 7
13:52:34.326 [new-single-2] INFO com.example.test.Example10_5 - # doOnNext map: 70
13:52:34.326 [new-single-2] INFO com.example.test.Example10_5 - # onNext: 70
~~~


### Scheduler.boundedElastic()
- ExecutorService 기반의 thread pool 을 생성한후, 그안에서 정해진 수만큼의 스레드를 사용하고, 작업이 종료된 스레드는 반납하여 재사용하는 방식
- CPU core 수 * 10 만큼의 thread 를 생성
- pool 에 있는 모든 스레드가 작업을 처리하고 있다면, 이용 가능한 스레드가 생길때까지 최대 100,000 개의 작업이 queue 에서 대기  
- Blocking I/O 작업을 효과적으로 처리하기 위한 방식
- 실행 시간이 긴 Blocking I/O 작업이 포함된 경우, 다른 Non-Blocking 처리에 영향을 주지 않도록 전용 스레드를 할당해서 Blocking I/O 작업을 처리하기 때문에 처리 시간을 효율적으로 사용할수 있음

### Scheduler.parallel()
- Scheduler.boundedElastic() 가 Blocking I/O 에 최적화 되어 있다면, Scheduler.parallel() 은 Non-Blocking I/O 에 최적화 되어 있음
- CPU core 수만큼 스레드를 생성


### Scheduler.fromExecutorService()
- 기존에 이미 사용하고 있는 ExecutorService 가 있다면, ExecutorService 로 부터 생성
- ExecutorService 부터 생성할수 있으나 reactor 에서 ExecutorService 서비스부터 생성하는 것을 권장되지 않음

### Scheduler.newXXXX()
- 스레드이름, 생성 가능한 디폴트 스레드의 개수, 스레드의 유휴시간, 데몬 스레드로의 동작 여부 등을 직접 지정해서 커스텀 스레드 풀을 생성할 수 있음


# 14. Operators

## 14.2 Sequence 생성을 위한 Operator

### JustOrEmpty
* Just 는 emit 할 데이터가 null 일 경우, NullPointException 발생
* JustOrEmpty 는 emit 할 데이터가 null 일 경우, onCompleteSignal 을 전송
* `just(Optional.ofNullable(null))` -> onCompleteSignal
* `justOrEmpty(Optional.ofNullable(null))` -> onCompleteSignal

~~~Java
public static void main(String[] args) {
    Mono
            .justOrEmpty(null)
            .subscribe(data -> {},
                    error -> {},
                    () -> log.info("# onComplete"));
}
~~~

~~~
[main] INFO com.example.test.Example - # onComplete
~~~

### fromIterable
* Iterable 에 포함된 데이터를 emit 하는 Flux 생성

~~~Java
public static void main(String[] args) {
    Flux
            .fromIterable(SampleData.coins)
            .subscribe(coin ->
                    log.info("coin 명: {}, 현재가: {}", coin.getT1(), coin.getT2())
            );
}
~~~

~~~
08:18:32.636 [main] INFO com.example.test.Example - coin 명: BTC, 현재가: 52000000
08:18:32.640 [main] INFO com.example.test.Example - coin 명: ETH, 현재가: 1720000
08:18:32.640 [main] INFO com.example.test.Example - coin 명: XRP, 현재가: 533
08:18:32.641 [main] INFO com.example.test.Example - coin 명: ICX, 현재가: 2080
08:18:32.641 [main] INFO com.example.test.Example - coin 명: EOS, 현재가: 4020
08:18:32.641 [main] INFO com.example.test.Example - coin 명: BCH, 현재가: 558000
~~~

### fromStream
* stream 에 포함된 데이터를 emit 하는 Flux 를 생성
* stream 은 재사용할수 없으며, cancel, error, complete 시에 자동으로 닫힘

~~~Java
public static void main(String[] args) {
    Flux
        .fromStream(() -> SampleData.coinNames.stream())
        .filter(coin -> coin.equals("BTC") || coin.equals("ETH"))
        .subscribe(data -> log.info("{}", data));
}
~~~

~~~
08:20:03.850 [main] INFO com.example.test.Example - BTC
08:20:03.851 [main] INFO com.example.test.Example - ETH
~~~

### range
* n 부터 1씩 증가한 연속된 수를 emit 하는 Flux 를 생성

~~~Java
public static void main(String[] args) {
    Flux
        .range(5, 10)  // 5부터 시작해서 10개
        .subscribe(data -> log.info("{}", data));
}
~~~

~~~
08:21:55.673 [main] INFO com.example.test.Example - 5
08:21:55.674 [main] INFO com.example.test.Example - 6
08:21:55.674 [main] INFO com.example.test.Example - 7
08:21:55.674 [main] INFO com.example.test.Example - 8
08:21:55.674 [main] INFO com.example.test.Example - 9
08:21:55.674 [main] INFO com.example.test.Example - 10
08:21:55.674 [main] INFO com.example.test.Example - 11
08:21:55.674 [main] INFO com.example.test.Example - 12
08:21:55.674 [main] INFO com.example.test.Example - 13
08:21:55.674 [main] INFO com.example.test.Example - 14
~~~

### defer
* Operator 를 선언한 시점에 데이터를 emit 하지 않음
* 구독하는 시점에 데이터를 emit 하는 Flux, Mono 를 생성
* 데이터 emit 을 지연시키기 때문에 불필요한 프로세스를 줄일수 있음

~~~java
public static void main(String[] args) throws InterruptedException {
    log.info("# start: {}", LocalDateTime.now());
    Mono<LocalDateTime> justMono = Mono.just(LocalDateTime.now());
    Mono<LocalDateTime> deferMono = Mono.defer(() ->
                                                Mono.just(LocalDateTime.now()));

    Thread.sleep(2000);

    justMono.subscribe(data -> log.info("# onNext just1: {}", data));
    deferMono.subscribe(data -> log.info("# onNext defer1: {}", data));

    Thread.sleep(2000);

    justMono.subscribe(data -> log.info("# onNext just2: {}", data));
    deferMono.subscribe(data -> log.info("# onNext defer2: {}", data));
}
~~~

~~~
08:26:31.451 [main] INFO com.example.test.Example - # start: 2024-04-04T08:26:31.440
08:26:33.575 [main] INFO com.example.test.Example - # onNext just1: 2024-04-04T08:26:31.455
08:26:33.576 [main] INFO com.example.test.Example - # onNext defer1: 2024-04-04T08:26:33.576
08:26:35.583 [main] INFO com.example.test.Example - # onNext just2: 2024-04-04T08:26:31.455
08:26:35.583 [main] INFO com.example.test.Example - # onNext defer2: 2024-04-04T08:26:35.583
~~~

* deferMono 는 2초의 간격을 두고 emit 됨
  * `08:26:33.576` -> `08:26:35.583` 
* justMono 의 경우 지연 시간과 상관없이 동일한 시간을 출력함
  * `08:26:31.455`
* justMono 가 동일한 시간을 출력하는 이유?
  * Just 는 Hot Publisher 이기 때문에, Subscriber 의 구독 여부와 상관없이 데이터를 emit 함
  * 구독이 발생하면 emit 된 데이터를 replay 해서 Subscriber 에게 전달함
  * 따라서 justMono 의 경우 출력 결과가 같음


~~~java
public static void main(String[] args) throws InterruptedException {
    log.info("# start: {}", LocalDateTime.now());
    Mono
        .just("Hello")
        .delayElement(Duration.ofSeconds(3))
        .switchIfEmpty(sayDefault())
//      .switchIfEmpty(Mono.defer(() -> sayDefault()))
        .subscribe(data -> log.info("# onNext: {}", data));

    Thread.sleep(3500);
}

private static Mono<String> sayDefault() {
    log.info("# Say Hi");
    return Mono.just("Hi");
}
~~~

~~~
// `.switchIfEmpty(sayDefault())` -> sayDefault() 호출됨
08:35:22.647 [main] INFO com.example.test.Example - # start: 2024-04-04T08:35:22.635
08:35:22.736 [main] INFO com.example.test.Example - # Say Hi
08:35:25.762 [parallel-1] INFO com.example.test.Example - # onNext: Hello
~~~

~~~
// `.switchIfEmpty(Mono.defer(() -> sayDefault()))` -> sayDefault() 호출되지 않음
08:40:24.108 [main] INFO com.example.test.Example - # start: 2024-04-04T08:40:24.098
08:40:27.225 [parallel-1] INFO com.example.test.Example - # onNext: Hello
~~~

* `.switchIfEmpty(sayDefault())`
  * sayDefault 가 호출되지 않을것 처럼 보이지만 실제로 호출됨
* `.switchIfEmpty(Mono.defer(() -> sayDefault()))`
  * defer 를 사용하면 불필요한 호출을 막을수 있음


### using
* parameter 로 전달받은 resource 를 emit 하는 Flux 를 생성
* resourceSupplier
  * 읽어올 resource
* sourceSupplier
  * 읽어온 resource 를 emit 하는 Flux
* resourceCleanup
  * 종료 signal(onComplete 또는 onError)이 발생할 경우 resource 를 해제하는 등의 후처리를 할수 있게 함

~~~
public static <T, D> Flux<T> using(Callable<? extends D> resourceSupplier, 
                                   Function<? super D, ? extends Publisher<? extends T>> sourceSupplier, 
                                   Consumer<? super D> resourceCleanup) {
}
~~~

~~~java
public static void main(String[] args) throws IOException {
    Path path = new ClassPathResource("using_example.txt").getFile().toPath();

    Flux
            .using(() -> Files.lines(path), Flux::fromStream, Stream::close)
            .subscribe(log::info);
}
~~~

~~~
08:53:51.938 [main] INFO com.example.test.Example - test line1
08:53:51.938 [main] INFO com.example.test.Example - test line2
08:53:51.939 [main] INFO com.example.test.Example - test line3
~~~

### generate
* 프로그래밍 방식으로 Signal 이벤트를 발생시킴
* 동기적으로 데이터를 하나씩 순차적으로 emit 하고자 할때 사용
* SynchronousSink
  * 하나의 signal 만 동기적으로 발생시킬수 있으며, 최대 하나의 상태값만 emit 하는 인터페이스 

~~~java
public static <T, S> Flux<T> generate(Callable<S> stateSupplier, 
                                      BiFunction<S, SynchronousSink<T>, S> generator) {
}
~~~

~~~java
public static void main(String[] args) {
    Flux
            .generate(
                    () -> 0,  // 초깃값을 0으로 지정
                    (state, sink) -> {
                        sink.next(state);
                        if (state == 10)
                            sink.complete();  // onComplete signal 을 발생시켜 Sequence 가 종료되도록 함
                        return ++state;
                    })
            .subscribe(data -> log.info("# onNext: {}", data));
}
~~~

~~~
09:00:13.691 [main] INFO com.example.test.Example - # onNext: 0
09:00:13.692 [main] INFO com.example.test.Example - # onNext: 1
09:00:13.692 [main] INFO com.example.test.Example - # onNext: 2
09:00:13.692 [main] INFO com.example.test.Example - # onNext: 3
09:00:13.692 [main] INFO com.example.test.Example - # onNext: 4
09:00:13.692 [main] INFO com.example.test.Example - # onNext: 5
09:00:13.692 [main] INFO com.example.test.Example - # onNext: 6
09:00:13.692 [main] INFO com.example.test.Example - # onNext: 7
09:00:13.692 [main] INFO com.example.test.Example - # onNext: 8
09:00:13.692 [main] INFO com.example.test.Example - # onNext: 9
09:00:13.692 [main] INFO com.example.test.Example - # onNext: 10
~~~

* tuple 객체 사용하여 구구단 3단을 출력하는 예제

~~~java
public static void main(String[] args) {
    final int dan = 3;
    Flux
            .generate(
                    () -> Tuples.of(dan, 1),
                    (state, sink) -> {
                        sink.next(state.getT1() + " * " + state.getT2() + " = " + state.getT1() * state.getT2());
                        if (state.getT2() == 9) {
                            sink.complete();
                        }

                        return Tuples.of(state.getT1(), state.getT2() + 1);
                    }, 
                    state -> log.info("# 구구단 {}단 종료!", state.getT1()))
            .subscribe(data -> log.info("# onNext: {}", data));
}
~~~

~~~
09:07:46.571 [main] INFO com.example.test.Example - # onNext: 3 * 1 = 3
09:07:46.572 [main] INFO com.example.test.Example - # onNext: 3 * 2 = 6
09:07:46.572 [main] INFO com.example.test.Example - # onNext: 3 * 3 = 9
09:07:46.572 [main] INFO com.example.test.Example - # onNext: 3 * 4 = 12
09:07:46.572 [main] INFO com.example.test.Example - # onNext: 3 * 5 = 15
09:07:46.572 [main] INFO com.example.test.Example - # onNext: 3 * 6 = 18
09:07:46.572 [main] INFO com.example.test.Example - # onNext: 3 * 7 = 21
09:07:46.572 [main] INFO com.example.test.Example - # onNext: 3 * 8 = 24
09:07:46.572 [main] INFO com.example.test.Example - # onNext: 3 * 9 = 27
09:07:46.572 [main] INFO com.example.test.Example - # 구구단 3단 종료!
~~~

### create
* generate 처럼 프로그래밍 방식으로 Signal 이벤트를 발생시킴
* generate 와 차이점
  * generate 는 데이터를 동기적으로 한번에 한건씩 emit 함
  * create 는 데이터를 비동기적으로 한번에 여러건 emit 할수 있음

#### pull 방식으로 데이터 처리
~~~java
public class Example {
    static int SIZE = 0;
    static int COUNT = -1;
    final static List<Integer> DATA_SOURCE = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

    public static void main(String[] args) {
        log.info("# start");
        Flux.create((FluxSink<Integer> sink) -> {
            sink.onRequest(n -> {
                try {
                    Thread.sleep(1000L);
                    for (int i = 0; i < n; i++) {
                        if (COUNT >= 9) {
                            sink.complete();
                        } else {
                            COUNT++;
                            sink.next(DATA_SOURCE.get(COUNT));
                        }
                    }
                } catch (InterruptedException e) {}
            });

            sink.onDispose(() -> log.info("# clean up"));
        }).subscribe(new BaseSubscriber<Integer>() {
            @Override
            protected void hookOnSubscribe(Subscription subscription) {
                request(2);
            }

            @Override
            protected void hookOnNext(Integer value) {
                SIZE++;
                log.info("# onNext: {}", value);
                if (SIZE == 2) {
                    request(2);
                    SIZE = 0;
                }
            }

            @Override
            protected void hookOnComplete() {
                log.info("# onComplete");
            }
        });
    }
}
~~~

* 구독이 발생하면 hookOnSubscribe 에서 request(2) 를 호출하여 한번에 두개의 데이터를 요청
* sink.onRequest 메서드의 람다 표현식이 실행됨
* sink.next 로 지정한 개수 만큼 emit 함
* hookOnNext 에서 emit 된 데이터를 로그로 출력하고, request(2) 로 다시 2개의 데이터를 요청
* dataSource List 의 숫자를 모두 emit 하면 sink.complete 로 onComplete signal 을 발생 시킴
* hookOnComplete 에서 종료 로그를 출력
* sink.onDispose 에 람다 표현식이 실행되어 후처리 로그를 출력

~~~
09:12:53.811 [main] INFO com.example.test.Example - # start
09:12:55.037 [main] INFO com.example.test.Example - # onNext: 1
09:12:55.046 [main] INFO com.example.test.Example - # onNext: 2
09:12:56.049 [main] INFO com.example.test.Example - # onNext: 3
09:12:56.049 [main] INFO com.example.test.Example - # onNext: 4
09:12:57.054 [main] INFO com.example.test.Example - # onNext: 5
09:12:57.055 [main] INFO com.example.test.Example - # onNext: 6
09:12:58.057 [main] INFO com.example.test.Example - # onNext: 7
09:12:58.057 [main] INFO com.example.test.Example - # onNext: 8
09:12:59.064 [main] INFO com.example.test.Example - # onNext: 9
09:12:59.065 [main] INFO com.example.test.Example - # onNext: 10
09:13:00.072 [main] INFO com.example.test.Example - # onComplete
09:13:00.085 [main] INFO com.example.test.Example - # clean up
~~~

#### push 방식으로 데이터 처리
* 암호 화폐의 가격 변동이 있을때마다 변동되는 가격 데이터를 Subscriber 에게 비동기적으로 emit 하는 코드

~~~java
public class Example {
    public interface CryptoCurrencyPriceListener {
        void onPrice(List<Integer> priceList);
        void onComplete();
    }
    
    public static class CryptoCurrencyPriceEmitter {
        private CryptoCurrencyPriceListener listener;

        public void setListener(CryptoCurrencyPriceListener listener) {
            this.listener = listener;
        }

        public void flowInto() {
            listener.onPrice(Arrays.asList(50_000_000, 50_100_000, 50_700_000, 51_500_000, 52_000_000));
        }

        public void complete() {
            listener.onComplete();
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        CryptoCurrencyPriceEmitter priceEmitter = new CryptoCurrencyPriceEmitter();

        Flux.create((FluxSink<Integer> sink) ->
                        priceEmitter.setListener(new CryptoCurrencyPriceListener() {
                            @Override
                            public void onPrice(List<Integer> priceList) {
                                priceList.forEach(sink::next);
                            }

                            @Override
                            public void onComplete() {
                                sink.complete();
                            }
                        }))
                .publishOn(Schedulers.parallel())
                .subscribe(
                        data -> log.info("# onNext: {}", data),
                        error -> {},
                        () -> log.info("# onComplete"));

        Thread.sleep(3000L);
        log.info("# priceEmitter.flowInto");
        priceEmitter.flowInto();

        Thread.sleep(2000L);
        log.info("# priceEmitter.complete");
        priceEmitter.complete();
        Thread.sleep(2000L);
    }
}
~~~

* 가격 변동이 있을때마다 onPrice() 메서드가 호출됨
* 가격 변동을 발생시키기 위해 flowInto() 메서드를 호출

~~~
09:34:08.840 [main] INFO com.example.test.Example - # priceEmitter.flowInto
09:34:08.850 [parallel-1] INFO com.example.test.Example - # onNext: 50000000
09:34:08.855 [parallel-1] INFO com.example.test.Example - # onNext: 50100000
09:34:08.855 [parallel-1] INFO com.example.test.Example - # onNext: 50700000
09:34:08.855 [parallel-1] INFO com.example.test.Example - # onNext: 51500000
09:34:08.855 [parallel-1] INFO com.example.test.Example - # onNext: 52000000
09:34:10.855 [main] INFO com.example.test.Example - # priceEmitter.complete
09:34:10.859 [parallel-1] INFO com.example.test.Example - # onComplete
~~~

#### backpressure 방식으로 데이터 처리
* 한번에 여러건의 데이터를 비동기적으로 emit 할수 있기 때문에 Backpressure 전략이 필요

~~~java
public class Example {
    static int start = 1;
    static int end = 4;

    public static void main(String[] args) throws InterruptedException {
        Flux.create((FluxSink<Integer> emitter) -> {
                            emitter.onRequest(n -> {
                                log.info("# requested: " + n);
                                try {
                                    Thread.sleep(500L);
                                    for (int i = start; i <= end; i++) {
                                        emitter.next(i);
                                    }
                                    start += 4;
                                    end += 4;
                                } catch (InterruptedException e) {
                                }
                            });

                            emitter.onDispose(() -> {
                                log.info("# clean up");
                            });
                        },
                        FluxSink.OverflowStrategy.DROP)
                .subscribeOn(Schedulers.boundedElastic())
                .publishOn(Schedulers.parallel(), 2)
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(3000L);
    }

}
~~~

~~~
09:37:00.411 [boundedElastic-1] INFO com.example.test.Example - # requested: 2
09:37:00.915 [parallel-1] INFO com.example.test.Example - # onNext: 1
09:37:00.923 [parallel-1] INFO com.example.test.Example - # onNext: 2
09:37:00.932 [boundedElastic-1] INFO com.example.test.Example - # requested: 2
09:37:01.439 [parallel-1] INFO com.example.test.Example - # onNext: 5
09:37:01.439 [parallel-1] INFO com.example.test.Example - # onNext: 6
09:37:01.440 [boundedElastic-1] INFO com.example.test.Example - # requested: 2
09:37:01.945 [parallel-1] INFO com.example.test.Example - # onNext: 9
09:37:01.945 [parallel-1] INFO com.example.test.Example - # onNext: 10
09:37:01.946 [boundedElastic-1] INFO com.example.test.Example - # requested: 2
09:37:02.450 [parallel-1] INFO com.example.test.Example - # onNext: 13
09:37:02.451 [parallel-1] INFO com.example.test.Example - # onNext: 14
09:37:02.452 [boundedElastic-1] INFO com.example.test.Example - # requested: 2
09:37:02.957 [parallel-1] INFO com.example.test.Example - # onNext: 17
09:37:02.958 [parallel-1] INFO com.example.test.Example - # onNext: 18
09:37:02.959 [boundedElastic-1] INFO com.example.test.Example - # requested: 2
~~~

## 14.3 Sequence 필터링

### filter
* emit 된 데이터 중에서 조건에 일치하는 데이터만 Downstream 으로 emit

~~~java
public static void main(String[] args) {
    Flux
        .range(1, 20)
        .filter(num -> num % 2 != 0)
        .subscribe(data -> log.info("# onNext: {}", data));
}
~~~

~~~
09:39:51.035 [main] INFO com.example.test.Example - # onNext: 1
09:39:51.036 [main] INFO com.example.test.Example - # onNext: 3
09:39:51.036 [main] INFO com.example.test.Example - # onNext: 5
09:39:51.036 [main] INFO com.example.test.Example - # onNext: 7
09:39:51.036 [main] INFO com.example.test.Example - # onNext: 9
09:39:51.036 [main] INFO com.example.test.Example - # onNext: 11
09:39:51.036 [main] INFO com.example.test.Example - # onNext: 13
09:39:51.036 [main] INFO com.example.test.Example - # onNext: 15
09:39:51.036 [main] INFO com.example.test.Example - # onNext: 17
09:39:51.036 [main] INFO com.example.test.Example - # onNext: 19
~~~

### filterWhen
* Inner Sequence 를 통해 조건에 맞는 데이터인지를 비동기적으로 필터링을 수행
* 테스트 결과가 true 라면 Downstream 으로 emit 함 

~~~java
public static void main(String[] args) throws InterruptedException {
    Map<CovidVaccine, Tuple2<CovidVaccine, Integer>> vaccineMap = getCovidVaccines();
    Flux
            .fromIterable(SampleData.coronaVaccineNames)
            .filterWhen(vaccine -> Mono
                    .just(vaccineMap.get(vaccine).getT2() >= 3_000_000)
                    .publishOn(Schedulers.parallel()))
            .subscribe(data -> log.info("# onNext: {}", data));

    Thread.sleep(1000);
}
~~~


### skip
* 파라미터로 입력 받은 숫자만큼 건너뛴 후, Downstream 으로 emit
* 시간 단위로 건너뛰기도 가능

~~~java
public final Flux<T> skip(long skipped)
public final Flux<T> skip(Duration timespan)
~~~

* 숫자로 skip

~~~java
public static void main(String[] args) throws InterruptedException {
    Flux
            .interval(Duration.ofSeconds(1))
            .skip(2)
            .subscribe(data -> log.info("# onNext: {}", data));

    Thread.sleep(5500L);
}
~~~

* interval 로 1초에 한번씩 데이터를 emit 함
* `Thread.sleep(5500L)` main thread 가 5.5 초까지 실행되므로 0-4 까지 5개가 subscriber 로 전달됨  
* 0부터 증가하기 때문에 skip 으로 0,1 출력되지 않음

~~~
09:45:45.687 [parallel-1] INFO com.example.test.Example - # onNext: 2
09:45:46.691 [parallel-1] INFO com.example.test.Example - # onNext: 3
09:45:47.688 [parallel-1] INFO com.example.test.Example - # onNext: 4
~~~

* 시간단위로 skip

~~~java
public static void main(String[] args) throws InterruptedException {
    Flux
        .interval(Duration.ofMillis(300))
        .skip(Duration.ofSeconds(1))
        .subscribe(data -> log.info("# onNext: {}", data));

    Thread.sleep(2000L);
}
~~~

* interval 0.3 초씩 데이터를 emit 함
* 1초 skip 하여 0.9초까지 만들어진 0,1,2 가 skip 됨

~~~
09:51:17.108 [parallel-2] INFO com.example.test.Example - # onNext: 3
09:51:17.407 [parallel-2] INFO com.example.test.Example - # onNext: 4
09:51:17.706 [parallel-2] INFO com.example.test.Example - # onNext: 5
~~~


### take
* 파라미터로 입력 받는 숫자만큼 emit
  * skip 은 파라미터로 입력 받는 숫자만큼 건너뛰고 emit
* 시간 단위로 emit 도 가능

~~~java
public final Flux<T> take(long n)
public final Flux<T> take(Duration timespan)
~~~

~~~java
public static void main(String[] args) throws InterruptedException {
    Flux
        .interval(Duration.ofSeconds(1))
        .take(3)
        .subscribe(data -> log.info("# onNext: {}", data));

    Thread.sleep(4000L);
}
~~~

~~~
10:03:37.370 [parallel-1] INFO com.example.test.Example - # onNext: 0
10:03:38.367 [parallel-1] INFO com.example.test.Example - # onNext: 1
10:03:39.363 [parallel-1] INFO com.example.test.Example - # onNext: 2
~~~

### takeLast
* 파라미터로 입력 받은 개수만큼 가장 마지막에 emit 된 데이터를 Downstream 으로 emit

~~~java
public static void main(String[] args) {
    Flux
            .fromIterable(Arrays.asList(
                    Tuples.of(2010, 565L),
                    Tuples.of(2011, 36_094L),
                    Tuples.of(2012, 17_425L),
                    Tuples.of(2013, 1_405_209L),
                    Tuples.of(2014, 1_237_182L),
                    Tuples.of(2015, 557_603L),
                    Tuples.of(2016, 1_111_811L),
                    Tuples.of(2017, 22_483_583L),
                    Tuples.of(2018, 19_521_543L),
                    Tuples.of(2019, 15_761_568L),
                    Tuples.of(2020, 22_439_002L),
                    Tuples.of(2021, 63_364_000L)
            ))
            .takeLast(2)
            .subscribe(tuple -> log.info("# onNext: {}, {}", tuple.getT1(), tuple.getT2()));
}
~~~

* 가장 마지막 2 개인 2020, 2021 출력됨

~~~
10:06:24.447 [main] INFO com.example.test.Example - # onNext: 2020, 22439002
10:06:24.449 [main] INFO com.example.test.Example - # onNext: 2021, 63364000
~~~

### takeUntil
* 파라미터로 입력한 람다 표현식(Predicate)이 true 가 될때까지 Downstream 으로 emit
* Predicate 를 평가할때 사용한 데이터가 포함

~~~java
public static void main(String[] args) {
    Flux
            .fromIterable(Arrays.asList(
                    Tuples.of(2010, 565L),
                    Tuples.of(2011, 36_094L),
                    Tuples.of(2012, 17_425L),
                    Tuples.of(2013, 1_405_209L),
                    Tuples.of(2014, 1_237_182L),
                    Tuples.of(2015, 557_603L),
                    Tuples.of(2016, 1_111_811L),
                    Tuples.of(2017, 22_483_583L),
                    Tuples.of(2018, 19_521_543L),
                    Tuples.of(2019, 15_761_568L),
                    Tuples.of(2020, 22_439_002L),
                    Tuples.of(2021, 63_364_000L)
            ))
            .takeUntil(tuple -> tuple.getT2() > 20_000_000)
            .subscribe(tuple -> log.info("# onNext: {}, {}", tuple.getT1(), tuple.getT2()));
}
~~~

* 20_000_000 이상인 데이터가 처음 나오는 시점은 2017 
* Predicate 를 평가할때 사용한 데이터가 포함되기 때문에 2017 까지 출력됨

~~~
10:09:19.856 [main] INFO com.example.test.Example - # onNext: 2010, 565
10:09:19.858 [main] INFO com.example.test.Example - # onNext: 2011, 36094
10:09:19.858 [main] INFO com.example.test.Example - # onNext: 2012, 17425
10:09:19.858 [main] INFO com.example.test.Example - # onNext: 2013, 1405209
10:09:19.858 [main] INFO com.example.test.Example - # onNext: 2014, 1237182
10:09:19.858 [main] INFO com.example.test.Example - # onNext: 2015, 557603
10:09:19.858 [main] INFO com.example.test.Example - # onNext: 2016, 1111811
10:09:19.858 [main] INFO com.example.test.Example - # onNext: 2017, 22483583
~~~


### takeWhile
* 파라미터로 입력한 람다 표현식(Predicate)이 true 가 될때까지 Downstream 으로 emit
* Predicate 를 평가할때 사용한 데이터가 Downstream 으로 emit 되지 않음

~~~java
public static void main(String[] args) {
    Flux
            .fromIterable(Arrays.asList(
                    Tuples.of(2010, 565L),
                    Tuples.of(2011, 36_094L),
                    Tuples.of(2012, 17_425L),
                    Tuples.of(2013, 1_405_209L),
                    Tuples.of(2014, 1_237_182L),
                    Tuples.of(2015, 557_603L),
                    Tuples.of(2016, 1_111_811L),
                    Tuples.of(2017, 22_483_583L),
                    Tuples.of(2018, 19_521_543L),
                    Tuples.of(2019, 15_761_568L),
                    Tuples.of(2020, 22_439_002L),
                    Tuples.of(2021, 63_364_000L)
            ))
            .takeWhile(tuple -> tuple.getT2() < 20_000_000)
            .subscribe(tuple -> log.info("# onNext: {}, {}", tuple.getT1(), tuple.getT2()));
}
~~~

* 20_000_000 보다 큰 값이 나오는 처음이 2017
* 2016 이전까지는 데이터가 출력되고, Predicate 를 평가할때 사용한 데이터 2017 은 출력되지 않음

~~~
10:12:21.589 [main] INFO com.example.test.Example - # onNext: 2010, 565
10:12:21.590 [main] INFO com.example.test.Example - # onNext: 2011, 36094
10:12:21.591 [main] INFO com.example.test.Example - # onNext: 2012, 17425
10:12:21.591 [main] INFO com.example.test.Example - # onNext: 2013, 1405209
10:12:21.591 [main] INFO com.example.test.Example - # onNext: 2014, 1237182
10:12:21.591 [main] INFO com.example.test.Example - # onNext: 2015, 557603
10:12:21.591 [main] INFO com.example.test.Example - # onNext: 2016, 1111811
~~~


### next
* Upstream 에서 emit 되는 데이터중 첫번째 데이터만 Downstream 으로 emit 함
* 만약 Upstream 에서 emit 되는 데이터가 empty 라면 Downstream 으로 empty Mono 를 emit 함

~~~java
public static void main(String[] args) {
    Flux
            .fromIterable(Arrays.asList(
                    Tuples.of(2010, 565L),
                    Tuples.of(2011, 36_094L),
                    Tuples.of(2012, 17_425L),
                    Tuples.of(2013, 1_405_209L),
                    Tuples.of(2014, 1_237_182L),
                    Tuples.of(2015, 557_603L),
                    Tuples.of(2016, 1_111_811L),
                    Tuples.of(2017, 22_483_583L),
                    Tuples.of(2018, 19_521_543L),
                    Tuples.of(2019, 15_761_568L),
                    Tuples.of(2020, 22_439_002L),
                    Tuples.of(2021, 63_364_000L)
            ))
            .next()
            .subscribe(tuple -> log.info("# onNext: {}, {}",  tuple.getT1(), tuple.getT2()));
}
~~~

~~~
10:17:23.658 [main] INFO com.example.test.Example - # onNext: 2010, 565
~~~

## 14.4 Sequence 변환

### map
* Upstream 에서 emit 된 데이터를 mapper Function 을 사용하여 변환한 후, Downstream 으로 emit

#### map sample 1
~~~java
public static void main(String[] args) {
    Flux
        .just("1-Circle", "3-Circle", "5-Circle")
        .map(circle -> circle.replace("Circle", "Rectangle"))
        .subscribe(data -> log.info("# onNext: {}", data));
}
~~~

~~~
10:19:10.818 [main] INFO com.example.test.Example - # onNext: 1-Rectangle
10:19:10.819 [main] INFO com.example.test.Example - # onNext: 3-Rectangle
10:19:10.819 [main] INFO com.example.test.Example - # onNext: 5-Rectangle
~~~

#### map sample 2

~~~java
public class Example {
    public static void main(String[] args) {
        final double buyPrice = 50_000_000;
        Flux
                .fromIterable(SampleData.btcTopPricesPerYear)
                .filter(tuple -> tuple.getT1() == 2021)
                .doOnNext(data -> log.info("# doOnNext: {}", data))
                .map(tuple -> calculateProfitRate(buyPrice, tuple.getT2()))
                .subscribe(data -> log.info("# onNext: {}%", data));
    }

    private static double calculateProfitRate(final double buyPrice, Long topPrice) {
        return (topPrice - buyPrice) / buyPrice * 100;
    }
}
~~~

~~~
10:20:39.693 [main] INFO com.example.test.Example - # doOnNext: [2021,63364000]
10:20:39.695 [main] INFO com.example.test.Example - # onNext: 26.728%
~~~

### flatMap
* Upstream 에서 emit 된 한건의 데이터가 Inner Sequence 에서 여러건의 데이터로 변환됨
* 여러건의 데이터가 flatten 작업과 merge 작업을 거쳐 Downstream 으로 emit

#### flatMap sample 1
~~~java
public static void main(String[] args) {
    Flux
        .just("Good", "Bad")
        .flatMap(feeling -> Flux
                                .just("Morning", "Afternoon", "Evening")
                                .map(time -> feeling + " " + time))
        .subscribe(log::info);
}
~~~

~~~
10:23:35.456 [main] INFO com.example.test.Example - Good Morning
10:23:35.456 [main] INFO com.example.test.Example - Good Afternoon
10:23:35.456 [main] INFO com.example.test.Example - Good Evening
10:23:35.456 [main] INFO com.example.test.Example - Bad Morning
10:23:35.456 [main] INFO com.example.test.Example - Bad Afternoon
10:23:35.456 [main] INFO com.example.test.Example - Bad Evening
~~~

#### flatMap sample 2
* 비동기적으로 데이터를 emit

~~~java
public static void main(String[] args) throws InterruptedException {
    Flux
            .range(2, 8)
            .flatMap(dan -> Flux
                    .range(1, 9)
                    .publishOn(Schedulers.parallel())
                    .map(n -> dan + " * " + n + " = " + dan * n))
            .subscribe(log::info);

    Thread.sleep(100L);
}
~~~

* 3 단 로그를 보면 순서 보장이 되지 않음
~~~
10:26:47.951 [parallel-1] INFO com.example.test.Example - 2 * 1 = 2
...
10:26:47.953 [parallel-1] INFO com.example.test.Example - 2 * 8 = 16
10:26:47.953 [parallel-1] INFO com.example.test.Example - 2 * 9 = 18
10:26:47.956 [parallel-2] INFO com.example.test.Example - 3 * 1 = 3
10:26:47.956 [parallel-2] INFO com.example.test.Example - 4 * 1 = 4
...
10:26:47.956 [parallel-2] INFO com.example.test.Example - 4 * 9 = 36
10:26:47.956 [parallel-2] INFO com.example.test.Example - 5 * 1 = 5
...
10:26:47.956 [parallel-2] INFO com.example.test.Example - 5 * 9 = 45
10:26:47.956 [parallel-2] INFO com.example.test.Example - 6 * 1 = 6
...
10:26:47.956 [parallel-2] INFO com.example.test.Example - 6 * 9 = 54
10:26:47.956 [parallel-2] INFO com.example.test.Example - 7 * 1 = 7
...
10:26:47.956 [parallel-2] INFO com.example.test.Example - 7 * 9 = 63
10:26:47.956 [parallel-2] INFO com.example.test.Example - 8 * 1 = 8
...
10:26:47.957 [parallel-2] INFO com.example.test.Example - 8 * 9 = 72
10:26:47.957 [parallel-2] INFO com.example.test.Example - 9 * 1 = 9
...
10:26:47.957 [parallel-2] INFO com.example.test.Example - 9 * 9 = 81
10:26:47.957 [parallel-2] INFO com.example.test.Example - 3 * 2 = 6
10:26:47.957 [parallel-2] INFO com.example.test.Example - 3 * 3 = 9
10:26:47.957 [parallel-2] INFO com.example.test.Example - 3 * 4 = 12
10:26:47.957 [parallel-2] INFO com.example.test.Example - 3 * 5 = 15
10:26:47.957 [parallel-2] INFO com.example.test.Example - 3 * 6 = 18
10:26:47.957 [parallel-2] INFO com.example.test.Example - 3 * 7 = 21
10:26:47.957 [parallel-2] INFO com.example.test.Example - 3 * 8 = 24
10:26:47.957 [parallel-2] INFO com.example.test.Example - 3 * 9 = 27
~~~

### concat
* 파라미터로 입력되는 Publisher 의 Sequence 를 연결해서 데이터를 순차적으로 emit

#### concat sample 1 

~~~java
public static void main(String[] args) {
    Flux
            .concat(Flux.just(1, 2, 3), Flux.just(4, 5))
            .subscribe(data -> log.info("# onNext: {}", data));
}
~~~

~~~
10:29:23.290 [main] INFO com.example.test.Example - # onNext: 1
10:29:23.292 [main] INFO com.example.test.Example - # onNext: 2
10:29:23.292 [main] INFO com.example.test.Example - # onNext: 3
10:29:23.292 [main] INFO com.example.test.Example - # onNext: 4
10:29:23.292 [main] INFO com.example.test.Example - # onNext: 5
~~~

#### concat sample 2

~~~java
public static void main(String[] args) {
    Flux
            .concat(
                    Flux.fromIterable(Arrays.asList(
                            Tuples.of(CovidVaccine.AstraZeneca, 3_000_000),
                            Tuples.of(CovidVaccine.Janssen, 2_000_000)
                    )),
                    Flux.fromIterable(Arrays.asList(
                            Tuples.of(CovidVaccine.Pfizer, 1_000_000),
                            Tuples.of(CovidVaccine.Moderna, 4_000_000)
                    )),
                    Flux.fromIterable(Arrays.asList(
                            Tuples.of(CovidVaccine.Novavax, 2_500_000)
                    )))
            .subscribe(data -> log.info("# onNext: {}", data));
}
~~~

~~~
10:32:33.404 [main] INFO com.example.test.Example - # onNext: [AstraZeneca,3000000]
10:32:33.406 [main] INFO com.example.test.Example - # onNext: [Janssen,2000000]
10:32:33.406 [main] INFO com.example.test.Example - # onNext: [Pfizer,1000000]
10:32:33.406 [main] INFO com.example.test.Example - # onNext: [Moderna,4000000]
10:32:33.406 [main] INFO com.example.test.Example - # onNext: [Novavax,2500000]
~~~

### merge
* 파라미터로 입력되는 Publisher 의 Sequence 에서 emit 된 데이터를 interleave 방식으로 병합
  * interleave 는 emit 된 시간 순서대로 처리하는 방식
* concat 처럼 하나의 Publisher 의 Sequence 가 종료될때까지 나머지 Publisher 의 Sequence 가 대기하는 것이 아니라, 모든 Publisher 의 Sequence 가 즉시 subscribe 됨

#### merge sample 1

~~~java
public static void main(String[] args) throws InterruptedException {
    Flux
            .merge(
                    Flux.just(1, 2, 3, 4).delayElements(Duration.ofMillis(300L)),
                    Flux.just(5, 6, 7).delayElements(Duration.ofMillis(500L))
            )
            .subscribe(data -> log.info("# onNext: {}", data));

    Thread.sleep(2000L);
}
~~~

* parallel 6개로 실행되었고, onNext 를 보면 순서가 보장되지 않음을 알수 있음

~~~
10:38:06.134 [parallel-1] INFO com.example.test.Example - # onNext: 1
10:38:06.338 [parallel-2] INFO com.example.test.Example - # onNext: 5
10:38:06.443 [parallel-3] INFO com.example.test.Example - # onNext: 2
10:38:06.745 [parallel-5] INFO com.example.test.Example - # onNext: 3
10:38:06.844 [parallel-4] INFO com.example.test.Example - # onNext: 6
10:38:07.051 [parallel-6] INFO com.example.test.Example - # onNext: 4
10:38:07.349 [parallel-7] INFO com.example.test.Example - # onNext: 7
~~~

#### merge sample 2

~~~java
public class Example {
    public static void main(String[] args) throws InterruptedException {
        String[] usaStates = {
                "Ohio", "Michigan", "New Jersey", "Illinois", "New Hampshire",
                "Virginia", "Vermont", "North Carolina", "Ontario", "Georgia"
        };

        Flux
                .merge(getMeltDownRecoveryMessage(usaStates))
                .subscribe(log::info);

        Thread.sleep(2000L);
    }

    public static Map<String, Mono<String>> nppMap = Map.of(
            "Ontario", Mono.just("Ontario Done").delayElement(Duration.ofMillis(1500L)),
            "Vermont", Mono.just("Vermont Done").delayElement(Duration.ofMillis(400L)),
            "New Hampshire", Mono.just("New Hampshire Done").delayElement(Duration.ofMillis(700L)),
            "New Jersey", Mono.just("New Jersey Done").delayElement(Duration.ofMillis(500L)),
            "Ohio", Mono.just("Ohio Done").delayElement(Duration.ofMillis(1000L)),
            "Michigan", Mono.just("Michigan Done").delayElement(Duration.ofMillis(200L)),
            "Illinois", Mono.just("Illinois Done").delayElement(Duration.ofMillis(300L)),
            "Virginia", Mono.just("Virginia Done").delayElement(Duration.ofMillis(600L)),
            "North Carolina", Mono.just("North Carolina Done").delayElement(Duration.ofMillis(800L)),
            "Georgia", Mono.just("Georgia Done").delayElement(Duration.ofMillis(900L))
    );

    private static List<Mono<String>> getMeltDownRecoveryMessage(String[] usaStates) {
        List<Mono<String>> messages = new ArrayList<>();
        for (String state : usaStates) {
            messages.add(SampleData.nppMap.get(state));
        }

        return messages;
    }
}
~~~

~~~
10:44:50.997 [parallel-2] INFO com.example.test.Example - Michigan Done
10:44:51.100 [parallel-4] INFO com.example.test.Example - Illinois Done
10:44:51.200 [parallel-7] INFO com.example.test.Example - Vermont Done
10:44:51.300 [parallel-3] INFO com.example.test.Example - New Jersey Done
10:44:51.400 [parallel-6] INFO com.example.test.Example - Virginia Done
10:44:51.500 [parallel-5] INFO com.example.test.Example - New Hampshire Done
10:44:51.600 [parallel-8] INFO com.example.test.Example - North Carolina Done
10:44:51.698 [parallel-10] INFO com.example.test.Example - Georgia Done
10:44:51.800 [parallel-1] INFO com.example.test.Example - Ohio Done
10:44:52.300 [parallel-9] INFO com.example.test.Example - Ontario Done
~~~

### zip
* 파라미터로 입력되는 Publisher 에서 emit 된 데이터를 결합하는데,
* 각 Publisher  가 데이터를 하나씩 emit 할때까지 기다렸다가 결합

#### zip sample 1
* 0.3, 0.5 초마다 시간이 다르게 emit 하지만, 하나씩 emit 할때까지 기다렸다가 tuple2 객체로 묶어서 Subscriber 에게 전달

~~~java
Flux
    .zip(
        Flux.just(1, 2, 3).delayElements(Duration.ofMillis(300L)),
        Flux.just(4, 5, 6).delayElements(Duration.ofMillis(500L))
    )
    .subscribe(tuple2 -> log.info("# onNext: {}", tuple2));

Thread.sleep(2500L);
~~~

~~~
10:00:34.843 [parallel-2] INFO com.example.test.Example - # onNext: [1,4]
10:00:35.345 [parallel-4] INFO com.example.test.Example - # onNext: [2,5]
10:00:35.848 [parallel-6] INFO com.example.test.Example - # onNext: [3,6]
~~~


#### zip sample 2 - combinator 
* combinator 에서 변환 작업을 거친 후, 최종 변환된 데이터를 Subscriber 에게 전달

~~~java
Flux
    .zip(
        Flux.just(1, 2, 3).delayElements(Duration.ofMillis(300L)),
        Flux.just(4, 5, 6).delayElements(Duration.ofMillis(500L)),
        (n1, n2) -> n1 * n2
    )
    .subscribe(tuple2 -> log.info("# onNext: {}", tuple2));

Thread.sleep(2500L);
~~~

~~~
10:03:13.054 [parallel-2] INFO com.example.test.Example - # onNext: 4
10:03:13.561 [parallel-4] INFO com.example.test.Example - # onNext: 10
10:03:14.064 [parallel-6] INFO com.example.test.Example - # onNext: 18
~~~

### and
* Mono 의 Complete Signal 과 파라미터로 입력된 Publisher 의 Complete Signal 을 결합하여 새로운 Mono<Void> 를 반환
* Mono 와 파라미터로 입력된 Publisher 의 Sequence 가 모두 종료되었음을 Subscriber 에게 알릴수 있음
* 모든 Sequence 가 종료되길 기다렸다가 최종적으로 onComplete Signal 만 전송

#### and sample 1
* 1초에 Task1 emit, 0.6 초 Task2 emit, 1.2 초 Task3 emit
* Upstream 에서 emit 된 데이터가 Subscriber 에 전달되지 않음
  * onNext 로 찍히는 값이 없음
* onComplete 만 로그로 확인

~~~java
Mono
  .just("Task 1")
  .delayElement(Duration.ofSeconds(1))
  .doOnNext(data -> log.info("# Mono doOnNext: {}", data))
  .and(
      Flux
        .just("Task 2", "Task 3")
        .delayElements(Duration.ofMillis(600))
        .doOnNext(data -> log.info("# Flux doOnNext: {}", data))
  )
  .subscribe(
      data -> log.info("# onNext: {}", data),
      error -> log.error("# onError:", error),
      () -> log.info("# onComplete")
  );

Thread.sleep(5000);
~~~

~~~
10:09:15.422 [parallel-2] INFO com.example.test.Example - # Flux doOnNext: Task 2
10:09:15.821 [parallel-1] INFO com.example.test.Example - # Mono doOnNext: Task 1
10:09:16.027 [parallel-3] INFO com.example.test.Example - # Flux doOnNext: Task 3
10:09:16.028 [parallel-3] INFO com.example.test.Example - # onComplete
~~~

#### and sample 2
* restartApplicationServer, restartDBServer 두개 작업이 끝난 시점에 후처리 작업을 수행하는 예제

~~~java
public class Example {
    public static void main(String[] args) throws InterruptedException {
        restartApplicationServer()
                .and(restartDBServer())
                .subscribe(
                        data -> log.info("# onNext: {}", data),
                        error -> log.error("# onError:", error),
                        () -> log.info("# sent an email to Administrator: " +
                                "All Servers are restarted successfully")
                );

        Thread.sleep(6000L);
    }

    private static Mono<String> restartApplicationServer() {
        return Mono
                .just("Application Server was restarted successfully.")
                .delayElement(Duration.ofSeconds(2))
                .doOnNext(log::info);
    }

    private static Publisher<String> restartDBServer() {
        return Mono
                .just("DB Server was restarted successfully.")
                .delayElement(Duration.ofSeconds(4))
                .doOnNext(log::info);
    }
}
~~~

~~~
10:16:25.844 [parallel-1] INFO com.example.test.Example - Application Server was restarted successfully.
10:16:27.838 [parallel-2] INFO com.example.test.Example - DB Server was restarted successfully.
10:16:27.838 [parallel-2] INFO com.example.test.Example - # sent an email to Administrator: All Servers are restarted successfully
~~~

### collectList
* Flux 에서 emit 된 데이터를 모아서 List 로 변환 후, 변환된 List 를 emit 하는 Mono 를 반환
* Upstream Sequence 가 비어 있다면 비어있는 List 를 Downstream 으로 emit

~~~java
public class Example {
    public static void main(String[] args) {
        Flux
                .just("...", "---", "...")
                .map(code -> transformMorseCode(code))
                .collectList()
                .subscribe(list -> log.info(list.stream().collect(Collectors.joining())));
    }

    public static String transformMorseCode(String morseCode) {
        return SampleData.morseCodeMap.get(morseCode);
    }
}
~~~

~~~
10:18:46.289 [main] INFO com.example.test.Example - sos
~~~


### collectMap
* Flux 에서 emit 된 데이터를 기반으로 key 와 value 를 생성하여 Map 의 Element 로 추가한후, 최종적으로 Map 을 emit 하는 Mono 를 반환

~~~java
public class Example {
    public static void main(String[] args) {
        Flux
                .range(0, 26)
                .collectMap(key -> SampleData.morseCodes[key],
                        value -> transformToLetter(value))
                .subscribe(map -> log.info("# onNext: {}", map));
    }

    private static String transformToLetter(int value) {
        return Character.toString((char) ('a' + value));
    }
}
~~~

~~~
# onNext: {..=i, .---=j, --.-=q, ---=o, --.=g, .--=w, .-.=r, -.--=y, ....=h, -.-.=c, --=m, -.=n, .-..=l, ...-=v, -.-=k, -..=d, -=t, ..-=u, .=e, ...=s, -...=b, ..-.=f, .--.=p, -..-=x, --..=z, .-=a}
~~~

## 14.5 Sequence 의 내부 동작 확인을 위한 Operator
* Reactor 에서 Upstream Publisher 에서 emit 되는 데이터의 변경 없이 side effect 만을 수행하기 위한 Operator
* doOnXXX() 로 시작하는 Operator 임
* doOnXXX 로 시작하는 Operator 는 Consumer 또는 Runnable 타입의 함수형 인터페이스를 파라미터로 가지기 때문에 별도의 리턴 값이 없음
* Publisher 의 내부 동작을 엿볼 수 있으며, 로그를 출력하는 등의 디버깅 용도로 많이 사용

### doOnSubscriber
### doOnRequest
### doOnNext
### doOnComplete
### doOnError
### doOnCancel
### doOnTerminate
### doOnEach
### doOnDiscard
### doAfterTerminate
### doFirst
* 위치와 상관없이 제일 먼저 동작
### doFinally
* 위치와 상관없이 제일 마지막에 동작

~~~java
Flux.range(1, 5)
    .doFinally(signalType -> log.info("# doFinally 1: {}", signalType))
    .doFinally(signalType -> log.info("# doFinally 2: {}", signalType))
    .doOnNext(data -> log.info("# range > doOnNext(): {}", data))
    .doOnRequest(data -> log.info("# doOnRequest: {}", data))
    .doOnSubscribe(subscription -> log.info("# doOnSubscribe 1"))
    .doFirst(() -> log.info("# doFirst()"))
    .filter(num -> num % 2 == 1)
    .doOnNext(data -> log.info("# filter > doOnNext(): {}", data))
    .doOnComplete(() -> log.info("# doOnComplete()"))
    .subscribe(new BaseSubscriber<Integer>() {
        @Override
        protected void hookOnSubscribe(Subscription subscription) {
            request(1);
        }

        @Override
        protected void hookOnNext(Integer value) {
            log.info("# hookOnNext: {}", value);
            request(1);
        }
    });
~~~

~~~
10:28:48.129 [main] INFO com.example.test.Example - # doFirst()
10:28:48.131 [main] INFO com.example.test.Example - # doOnSubscribe 1
10:28:48.131 [main] INFO com.example.test.Example - # doOnRequest: 1
10:28:48.131 [main] INFO com.example.test.Example - # range > doOnNext(): 1
10:28:48.131 [main] INFO com.example.test.Example - # filter > doOnNext(): 1
10:28:48.131 [main] INFO com.example.test.Example - # hookOnNext: 1
10:28:48.131 [main] INFO com.example.test.Example - # doOnRequest: 1
10:28:48.131 [main] INFO com.example.test.Example - # range > doOnNext(): 2
10:28:48.132 [main] INFO com.example.test.Example - # range > doOnNext(): 3
10:28:48.132 [main] INFO com.example.test.Example - # filter > doOnNext(): 3
10:28:48.132 [main] INFO com.example.test.Example - # hookOnNext: 3
10:28:48.132 [main] INFO com.example.test.Example - # doOnRequest: 1
10:28:48.132 [main] INFO com.example.test.Example - # range > doOnNext(): 4
10:28:48.132 [main] INFO com.example.test.Example - # range > doOnNext(): 5
10:28:48.132 [main] INFO com.example.test.Example - # filter > doOnNext(): 5
10:28:48.132 [main] INFO com.example.test.Example - # hookOnNext: 5
10:28:48.132 [main] INFO com.example.test.Example - # doOnRequest: 1
10:28:48.132 [main] INFO com.example.test.Example - # doOnComplete()
10:28:48.132 [main] INFO com.example.test.Example - # doFinally 2: onComplete
10:28:48.132 [main] INFO com.example.test.Example - # doFinally 1: onComplete
~~~


## 14.6 에러 처리를 위한 Operator

### error
* 파라미터로 지정된 에러로 종료하는 Flux 를 생성
* throw 키워드를 사용해서 예외를 의도적으로 던지는것 같은 역할

#### error sample 1
* 명시적으로 error 이벤트를 발생시켜야 하는 경우

~~~java
Flux
  .range(1, 5)
  .flatMap(num -> {
      if ((num * 2) % 3 == 0) {
          return Flux.error(new IllegalArgumentException("Not allowed multiple of 3"));
      } else {
          return Mono.just(num * 2);
      }
  })
  .subscribe(data -> log.info("# onNext: {}", data),
          error -> log.error("# onError: ", error));
~~~

~~~
10:35:19.263 [main] INFO com.example.test.Example - # onNext: 2
10:35:19.263 [main] INFO com.example.test.Example - # onNext: 4
10:35:19.265 [main] ERROR com.example.test.Example - # onError: 
java.lang.IllegalArgumentException: Not allowed multiple of 3
	at com.example.test.Example.lambda$main$0(Example.java:28)
	at com.example.test.Example.main(Example.java:33)
~~~


#### error sample 2
* flatMap 처럼 Inner Sequence 가 존재하는 경우 체크 예외 발생 시 Flux 로 래핑해서 onError Signal 을 전송할 수 있다.

~~~java
public class Example {
    public static void main(String[] args) {
        Flux
                .just('a', 'b', 'c', '3', 'd')
                .flatMap(letter -> {
                    try {
                        return convert(letter);
                    } catch (DataFormatException e) {
                        return Flux.error(e);
                    }
                })
                .subscribe(data -> log.info("# onNext: {}", data),
                        error -> log.error("# onError: ", error));
    }

    private static Mono<String> convert(char ch) throws DataFormatException {
        if (!Character.isAlphabetic(ch)) {
            throw new DataFormatException("Not Alphabetic");
        }
        return Mono.just("Converted to " + Character.toUpperCase(ch));
    }
}
~~~

~~~
10:37:38.633 [main] INFO com.example.test.Example - # onNext: Converted to A
10:37:38.633 [main] INFO com.example.test.Example - # onNext: Converted to B
10:37:38.633 [main] INFO com.example.test.Example - # onNext: Converted to C
10:37:38.635 [main] ERROR com.example.test.Example - # onError: 
java.util.zip.DataFormatException: Not Alphabetic
	at com.example.test.Example.convert(Example.java:40)
	at com.example.test.Example.lambda$main$0(Example.java:29)
	at com.example.test.Example.main(Example.java:34)
~~~

### onErrorReturn
* 예외가 발생했을 때, error 이벤트를 발생시키지 않고, 대체값을 emit 함
* try ~ catch 문의 경우, catch해서 return default value 하는 것과 같다.

#### onErrorReturn sample 1

~~~java
public class Example {
    public static void main(String[] args) {
        getBooks()
                .map(book -> book.getPenName().toUpperCase())
                .onErrorReturn("No pen name")
                .subscribe(log::info);
    }

    public static Flux<Book> getBooks() {
        return Flux.fromIterable(Arrays.asList(
                new Book("Advance Java", "Tom", "Tom-boy", 25000, 100),
                new Book("Advance Python", "Grace", "Grace-girl", 22000, 150),
                new Book("Advance Reactor", "Smith", "David-boy", 35000, 200),
                new Book("Getting started Java", "Tom", "Tom-boy", 32000, 230),
                new Book("Advance Kotlin", "Kevin", "Kevin-boy", 32000, 250),
                new Book("Advance Javascript", "Mike", "Tom-boy", 32000, 320),
                new Book("Getting started Kotlin", "Kevin", "Kevin-boy", 32000, 150),
                new Book("Getting started Python", "Grace", "Grace-girl", 32000, 200),
                new Book("Getting started Reactor", "Smith", null, 32000, 250),
                new Book("Getting started Javascript", "Mike", "David-boy", 32000, 330)
        ));
    }
    
    @Getter
    @AllArgsConstructor
    public static class Book {
        private String bookName;
        private String authorName;
        private String penName;
        private int price;
        private int stockQuantity;
    }
}
~~~

~~~
10:38:44.384 [main] INFO com.example.test.Example - TOM-BOY
10:38:44.384 [main] INFO com.example.test.Example - GRACE-GIRL
10:38:44.384 [main] INFO com.example.test.Example - DAVID-BOY
10:38:44.384 [main] INFO com.example.test.Example - TOM-BOY
10:38:44.384 [main] INFO com.example.test.Example - KEVIN-BOY
10:38:44.384 [main] INFO com.example.test.Example - TOM-BOY
10:38:44.384 [main] INFO com.example.test.Example - KEVIN-BOY
10:38:44.384 [main] INFO com.example.test.Example - GRACE-GIRL
10:38:44.385 [main] INFO com.example.test.Example - No pen name
~~~

#### onErrorReturn sample 2
* 첫번째 파라미터로 특정 예외 타입을 지정해서 지정된 타입의 예외가 발생할 경우에만 대체값을 emit 하도록 함

~~~
public class Example {
    public static void main(String[] args) {
        getBooks()
                .map(book -> book.getPenName().toUpperCase())
                .onErrorReturn(NullPointerException.class, "no pen name")
                .onErrorReturn(IllegalFormatException.class, "Illegal pen name")
                .subscribe(log::info);
    }
}
~~~


### onErrorResume
* 에러 이벤트가 발생했을 때, 에러 이벤트를 Downstream 으로 전파하지 않고 대체 Publisher 를 리턴
* onErrorResume 에서 새로운 Publisher 로 리턴됨

~~~java
public class Example {
    public static void main(String[] args) {
        final String keyword = "DDD";
        getBooksFromCache(keyword)
                .onErrorResume(error -> getBooksFromDatabase(keyword))
                .subscribe(data -> log.info("# onNext: {}", data.getBookName()),
                        error -> log.error("# onError: ", error));
    }

    public static Flux<Book> getBooksFromCache(final String keyword) {
        return Flux
                .fromIterable(SampleData.books)
                .filter(book -> book.getBookName().contains(keyword))
                .switchIfEmpty(Flux.error(new NoSuchBookException("No such Book")));
    }

    public static Flux<Book> getBooksFromDatabase(final String keyword) {
        List<Book> books = new ArrayList<>(SampleData.books);
        books.add(new Book("DDD: Domain Driven Design",
                "Joy", "ddd-man", 35000, 200));
        return Flux
                .fromIterable(books)
                .filter(book -> book.getBookName().contains(keyword))
                .switchIfEmpty(Flux.error(new NoSuchBookException("No such Book")));
    }

    private static class NoSuchBookException extends RuntimeException {
        NoSuchBookException(String message) {
            super(message);
        }
    }
}
~~~

~~~
10:47:04.585 [main] INFO com.example.test.Example - # onNext: DDD: Domain Driven Design
~~~

### onErrorContinue
* 에러가 발생했을때, Sequence 가 종료되지 않고, 아직 emit 되지 않은 데이터를 다시 emit 해야하는 상황에 사용
* 에러 영역 내에 있는 데이터는 제거하고, Upstream 에서 후속 데이터를 emit 하는 방식으로 에러를 복구할수 있도록 해줌

~~~java
public static void main(String[] args) {
    Flux
            .just(1, 2, 4, 0, 6, 12)
            .map(num -> 12 / num)
            .onErrorContinue((error, num) -> log.error("error: {}, num: {}", error, num))
            .subscribe(data -> log.info("# onNext: {}", data),
                    error -> log.error("# onError: ", error));
}
~~~

~~~
10:51:53.993 [main] INFO com.example.test.Example - # onNext: 12
10:51:53.994 [main] INFO com.example.test.Example - # onNext: 6
10:51:53.994 [main] INFO com.example.test.Example - # onNext: 3
10:51:53.995 [main] ERROR com.example.test.Example - error: java.lang.ArithmeticException: / by zero, num: 0
10:51:53.995 [main] INFO com.example.test.Example - # onNext: 2
10:51:53.995 [main] INFO com.example.test.Example - # onNext: 1
~~~

### retry
* Publisher 가 데이터를 emit 하는 과정에서 에러가 발생하면, 파라미터로 입력한 횟수만큼 원본 Flux 의 Sequence 를 다시 구독
* 파라미터로 Long.MAX_VALUE 를 입력하면 재구독을 무한 반복함

#### retry sample 1

~~~java
public static void main(String[] args) throws InterruptedException {
    final int[] count = {1};
    Flux
        .range(1, 3)
        .delayElements(Duration.ofSeconds(1))
        .map(num -> {
            try {
                if (num == 3 && count[0] == 1) {
                    count[0]++;
                    Thread.sleep(1000);
                }
            } catch (InterruptedException e) {}

            return num;
        })
        .timeout(Duration.ofMillis(1500))
        .retry(1)
        .subscribe(data -> log.info("# onNext: {}", data),
                (error -> log.error("# onError: ", error)),
                () -> log.info("# onComplete"));

    Thread.sleep(7000);
}
~~~

~~~
10:53:48.898 [parallel-2] INFO com.example.test.Example - # onNext: 1
10:53:49.904 [parallel-4] INFO com.example.test.Example - # onNext: 2
10:53:51.911 [parallel-6] DEBUG reactor.core.publisher.Operators - onNextDropped: 3
10:53:52.418 [parallel-8] INFO com.example.test.Example - # onNext: 1
10:53:53.425 [parallel-10] INFO com.example.test.Example - # onNext: 2
10:53:54.430 [parallel-2] INFO com.example.test.Example - # onNext: 3
10:53:54.430 [parallel-2] INFO com.example.test.Example - # onComplete
~~~

#### retry sample 2
* .collect(Collectors.toSet()) 중복을 없애려고 toSet 을 사용
~~~java
public class Example {
    public static void main(String[] args) throws InterruptedException {
        getBooks()
                .collect(Collectors.toSet())
                .subscribe(bookSet -> bookSet.stream()
                        .forEach(book -> log.info("book name: {}, price: {}",
                                book.getBookName(), book.getPrice())));

        Thread.sleep(12000);
    }

    private static Flux<Book> getBooks() {
        final int[] count = {0};
        return Flux
                .fromIterable(SampleData.books)
                .delayElements(Duration.ofMillis(500))
                .map(book -> {
                    try {
                        count[0]++;
                        if (count[0] == 3) {
                            Thread.sleep(2000);
                        }
                    } catch (InterruptedException e) {
                    }

                    return book;
                })
                .timeout(Duration.ofSeconds(2))
                .retry(1)
                .doOnNext(book -> log.info("# getBooks > doOnNext: {}, price: {}",
                        book.getBookName(), book.getPrice()));
    }
}
~~~

~~~
10:56:32.253 [parallel-2] INFO com.example.test.Example - # getBooks > doOnNext: Advance Java, price: 25000
10:56:32.757 [parallel-4] INFO com.example.test.Example - # getBooks > doOnNext: Advance Python, price: 22000
10:56:35.259 [parallel-8] INFO com.example.test.Example - # getBooks > doOnNext: Advance Java, price: 25000
10:56:35.267 [parallel-6] DEBUG reactor.core.publisher.Operators - onNextDropped: com.example.test.Book@6464503f
10:56:35.765 [parallel-10] INFO com.example.test.Example - # getBooks > doOnNext: Advance Python, price: 22000
10:56:36.271 [parallel-2] INFO com.example.test.Example - # getBooks > doOnNext: Advance Reactor, price: 35000
10:56:36.776 [parallel-4] INFO com.example.test.Example - # getBooks > doOnNext: Getting started Java, price: 32000
10:56:37.282 [parallel-6] INFO com.example.test.Example - # getBooks > doOnNext: Advance Kotlin, price: 32000
10:56:37.793 [parallel-8] INFO com.example.test.Example - # getBooks > doOnNext: Advance Javascript, price: 32000
10:56:38.301 [parallel-10] INFO com.example.test.Example - # getBooks > doOnNext: Getting started Kotlin, price: 32000
10:56:38.804 [parallel-2] INFO com.example.test.Example - # getBooks > doOnNext: Getting started Python, price: 32000
10:56:39.309 [parallel-4] INFO com.example.test.Example - # getBooks > doOnNext: Getting started Reactor, price: 32000
10:56:39.811 [parallel-6] INFO com.example.test.Example - # getBooks > doOnNext: Getting started Javascript, price: 32000
10:56:39.814 [parallel-6] INFO com.example.test.Example - book name: Advance Kotlin, price: 32000
10:56:39.814 [parallel-6] INFO com.example.test.Example - book name: Getting started Javascript, price: 32000
10:56:39.814 [parallel-6] INFO com.example.test.Example - book name: Advance Javascript, price: 32000
10:56:39.814 [parallel-6] INFO com.example.test.Example - book name: Advance Python, price: 22000
10:56:39.814 [parallel-6] INFO com.example.test.Example - book name: Getting started Python, price: 32000
10:56:39.815 [parallel-6] INFO com.example.test.Example - book name: Getting started Kotlin, price: 32000
10:56:39.816 [parallel-6] INFO com.example.test.Example - book name: Getting started Java, price: 32000
10:56:39.816 [parallel-6] INFO com.example.test.Example - book name: Advance Java, price: 25000
10:56:39.816 [parallel-6] INFO com.example.test.Example - book name: Advance Reactor, price: 35000
10:56:39.816 [parallel-6] INFO com.example.test.Example - book name: Getting started Reactor, price: 32000
~~~

## 14.7 Sequence 의 동작 시간 측정을 위한 Operator

### elapsed
* `emit 된 데이터 사이의 경과 시간`을 측정해서 Tuple<Long, T> 형태로 Downstream 에 emit

#### elapsed sample 1
* emit 된 첫번째 데이터는 onSubscribe Signal 과 첫번째 데이터 사이의 시간을 기준으로 측정

~~~java
public static void main(String[] args) throws InterruptedException {
    Flux
            .range(1, 5)
            .delayElements(Duration.ofSeconds(1))
            .elapsed()
            .subscribe(data -> log.info("# onNext: {}, time: {}",
                    data.getT2(), data.getT1()));

    Thread.sleep(6000);
}
~~~

~~~
10:58:48.626 [parallel-1] INFO com.example.test.Example - # onNext: 1, time: 1014
10:58:49.627 [parallel-2] INFO com.example.test.Example - # onNext: 2, time: 1002
10:58:50.632 [parallel-3] INFO com.example.test.Example - # onNext: 3, time: 1005
10:58:51.638 [parallel-4] INFO com.example.test.Example - # onNext: 4, time: 1006
10:58:52.641 [parallel-5] INFO com.example.test.Example - # onNext: 5, time: 1003
~~~

#### elapsed sample 2

~~~java
public static void main(String[] args) {
    URI worldTimeUri = UriComponentsBuilder.newInstance().scheme("http")
            .host("worldtimeapi.org")
            .port(80)
            .path("/api/timezone/Asia/Seoul")
            .build()
            .encode()
            .toUri();

    RestTemplate restTemplate = new RestTemplate();
    HttpHeaders headers = new HttpHeaders();
    headers.setAccept(Collections.singletonList(MediaType.APPLICATION_JSON));


    Mono.defer(() -> Mono.just(
                            restTemplate
                                    .exchange(worldTimeUri,
                                            HttpMethod.GET,
                                            new HttpEntity<String>(headers),
                                            String.class)
                    )
            )
            .repeat(4)
            .elapsed()
            .map(response -> {
                DocumentContext jsonContext =
                        JsonPath.parse(response.getT2().getBody());
                String dateTime = jsonContext.read("$.datetime");
                return Tuples.of(dateTime, response.getT1());
            })
            .subscribe(
                    data -> log.info("now: {}, elapsed time: {}", data.getT1(), data.getT2()),
                    error -> log.error("# onError:", error),
                    () -> log.info("# onComplete")
            );
}
~~~

~~~
11:01:29.456 [main] DEBUG org.springframework.web.client.RestTemplate - HTTP GET http://worldtimeapi.org:80/api/timezone/Asia/Seoul
11:01:29.855 [main] INFO com.example.test.Example - now: 2024-04-11T11:01:29.767723+09:00, elapsed time: 381
11:01:29.855 [main] DEBUG org.springframework.web.client.RestTemplate - HTTP GET http://worldtimeapi.org:80/api/timezone/Asia/Seoul
11:01:29.973 [main] INFO com.example.test.Example - now: 2024-04-11T11:01:29.905697+09:00, elapsed time: 141
11:01:29.973 [main] DEBUG org.springframework.web.client.RestTemplate - HTTP GET http://worldtimeapi.org:80/api/timezone/Asia/Seoul
11:01:30.069 [main] INFO com.example.test.Example - now: 2024-04-11T11:01:30.026461+09:00, elapsed time: 97
11:01:30.069 [main] DEBUG org.springframework.web.client.RestTemplate - HTTP GET http://worldtimeapi.org:80/api/timezone/Asia/Seoul
11:01:30.165 [main] INFO com.example.test.Example - now: 2024-04-11T11:01:30.119298+09:00, elapsed time: 96
11:01:30.165 [main] DEBUG org.springframework.web.client.RestTemplate - HTTP GET http://worldtimeapi.org:80/api/timezone/Asia/Seoul
11:01:30.258 [main] INFO com.example.test.Example - now: 2024-04-11T11:01:30.216329+09:00, elapsed time: 93
11:01:30.261 [main] INFO com.example.test.Example - # onComplete
~~~

## 14.8 Flux Sequence 분할을 위한 Operator

### window(maxSize)
* Upstream에서 emit되는 첫 번째 데이터부터 maxSize의 숫자만큼의 데이터를 포함하는 새로운 Flux로 분할함
* 새롭게 생성되는 Flux를 윈도우(Window)라고 함

~~~java
public static void main(String[] args) {
    Flux.range(1, 11)
            .window(3)
            .flatMap(flux -> {
                log.info("======================");
                return flux;
            })
            .subscribe(new BaseSubscriber<Integer>() {
                @Override
                protected void hookOnSubscribe(Subscription subscription) {
                    subscription.request(2);
                }

                @Override
                protected void hookOnNext(Integer value) {
                    log.info("# onNext: {}", value);
                    request(2);
                }
            });
}
~~~

~~~
11:05:29.559 [main] INFO com.example.test.Example - ======================
11:05:29.560 [main] INFO com.example.test.Example - # onNext: 1
11:05:29.561 [main] INFO com.example.test.Example - # onNext: 2
11:05:29.561 [main] INFO com.example.test.Example - # onNext: 3
11:05:29.561 [main] INFO com.example.test.Example - ======================
11:05:29.561 [main] INFO com.example.test.Example - # onNext: 4
11:05:29.561 [main] INFO com.example.test.Example - # onNext: 5
11:05:29.561 [main] INFO com.example.test.Example - # onNext: 6
11:05:29.561 [main] INFO com.example.test.Example - ======================
11:05:29.561 [main] INFO com.example.test.Example - # onNext: 7
11:05:29.561 [main] INFO com.example.test.Example - # onNext: 8
11:05:29.561 [main] INFO com.example.test.Example - # onNext: 9
11:05:29.561 [main] INFO com.example.test.Example - ======================
11:05:29.561 [main] INFO com.example.test.Example - # onNext: 10
11:05:29.561 [main] INFO com.example.test.Example - # onNext: 11
~~~

### buffer(maxSize)
* Upstream에서 emit되는 첫 번째 데이터부터 maxSize 숫자만큼의 데이터를 List 버퍼로 한번에 emit

~~~java
public static void main(String[] args) {
    Flux.range(1, 95)
            .buffer(10)
            .subscribe(buffer -> log.info("# onNext: {}", buffer));
}
~~~

~~~
11:07:45.290 [main] INFO com.example.test.Example - # onNext: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
11:07:45.290 [main] INFO com.example.test.Example - # onNext: [11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
11:07:45.290 [main] INFO com.example.test.Example - # onNext: [21, 22, 23, 24, 25, 26, 27, 28, 29, 30]
11:07:45.290 [main] INFO com.example.test.Example - # onNext: [31, 32, 33, 34, 35, 36, 37, 38, 39, 40]
11:07:45.290 [main] INFO com.example.test.Example - # onNext: [41, 42, 43, 44, 45, 46, 47, 48, 49, 50]
11:07:45.290 [main] INFO com.example.test.Example - # onNext: [51, 52, 53, 54, 55, 56, 57, 58, 59, 60]
11:07:45.290 [main] INFO com.example.test.Example - # onNext: [61, 62, 63, 64, 65, 66, 67, 68, 69, 70]
11:07:45.290 [main] INFO com.example.test.Example - # onNext: [71, 72, 73, 74, 75, 76, 77, 78, 79, 80]
11:07:45.290 [main] INFO com.example.test.Example - # onNext: [81, 82, 83, 84, 85, 86, 87, 88, 89, 90]
11:07:45.290 [main] INFO com.example.test.Example - # onNext: [91, 92, 93, 94, 95]
~~~

### bufferTimeout(maxSize, maxTime)
* Upstream에서 emit되는 첫 번째 데이터부터 maxSize 숫자 만큼의 데이터 또는 maxTime 내에 emit된 데이터를 List 버퍼로 한번에 emit
* maxSize나 maxTime에서 먼저 조건에 부합할때까지 emit된 데이터를 List 버퍼로 emit 

~~~java
public static void main(String[] args) {
    Flux
        .range(1, 20)
        .map(num -> {
            try {
                if (num < 10) {
                    Thread.sleep(100L);
                } else {
                    Thread.sleep(300L);
                }
            } catch (InterruptedException e) {}
            return num;
        })
        .bufferTimeout(3, Duration.ofMillis(400L))
        .subscribe(buffer -> log.info("# onNext: {}", buffer));
}
~~~

~~~
11:09:00.156 [main] INFO com.example.test.Example - # onNext: [1, 2, 3]
11:09:00.469 [main] INFO com.example.test.Example - # onNext: [4, 5, 6]
11:09:00.777 [main] INFO com.example.test.Example - # onNext: [7, 8, 9]
11:09:01.483 [parallel-1] INFO com.example.test.Example - # onNext: [10, 11]
11:09:02.091 [parallel-1] INFO com.example.test.Example - # onNext: [12, 13]
11:09:02.700 [parallel-1] INFO com.example.test.Example - # onNext: [14, 15]
11:09:03.307 [parallel-1] INFO com.example.test.Example - # onNext: [16, 17]
11:09:03.919 [parallel-1] INFO com.example.test.Example - # onNext: [18, 19]
11:09:04.118 [main] INFO com.example.test.Example - # onNext: [20]
~~~

### groupBy(keyMapper)
* emit되는 데이터를 key를 기준으로 그룹화 한 GroupedFlux 를 리턴
* 그룹화 된 GroupedFlux로 그룹별 작업을 할 수 있음

### groupBy(keyMapper) sample 1

~~~java
public static void main(String[] args) {
    Flux.fromIterable(SampleData.books)
            .groupBy(book -> book.getAuthorName())
            .flatMap(groupedFlux ->
                    groupedFlux
                            .map(book -> book.getBookName() + "(" + book.getAuthorName() + ")")
                            .collectList()
            )
            .subscribe(bookByAuthor -> log.info("# book by author: {}", bookByAuthor));
}
~~~

~~~
# book by author: [Advance Kotlin(Kevin), Getting started Kotlin(Kevin)]
# book by author: [Advance Javascript(Mike), Getting started Javascript(Mike)]
# book by author: [Advance Java(Tom), Getting started Java(Tom)]
# book by author: [Advance Python(Grace), Getting started Python(Grace)]
# book by author: [Advance Reactor(Smith), Getting started Reactor(Smith)]
~~~

### groupBy(keyMapper) sample 2
* valueMapper를 추가로 전달해서 그룹화 되어 emit되는 데이터의 값을 미리 다른 값으로 변경할 수 있음

~~~java
public static void main(String[] args) {
    Flux.fromIterable(SampleData.books)
            .groupBy(book -> book.getAuthorName(),
                    book -> book.getBookName() + "(" + book.getAuthorName() + ")")
            .flatMap(groupedFlux -> groupedFlux.collectList())
            .subscribe(bookByAuthor -> log.info("# book by author: {}", bookByAuthor));
}
~~~

~~~
# book by author: [Advance Kotlin(Kevin), Getting started Kotlin(Kevin)]
# book by author: [Advance Javascript(Mike), Getting started Javascript(Mike)]
# book by author: [Advance Java(Tom), Getting started Java(Tom)]
# book by author: [Advance Python(Grace), Getting started Python(Grace)]
# book by author: [Advance Reactor(Smith), Getting started Reactor(Smith)]
~~~

### groupBy(keyMapper) sample 3
* 그룹화 된 GroupedFlux로 그룹별 작업을 할 수 있다.
* 저자 명으로 된 도서의 가격

~~~java
public static void main(String[] args) {
    Flux.fromIterable(SampleData.books)
      .groupBy(book -> book.getAuthorName())
      .flatMap(groupedFlux -> 
        Mono
          .just(groupedFlux.key())
          .zipWith(
              groupedFlux
                      .map(book -> (int)(book.getPrice() * book.getStockQuantity() * 0.1))
                      .reduce((y1, y2) -> y1 + y2),
              (authorName, sumRoyalty) -> authorName + "'s royalty: " + sumRoyalty)
      )
      .subscribe(log::info);
}
~~~

~~~
11:18:48.219 [main] INFO com.example.test.Example - Kevin's royalty: 1280000
11:18:48.219 [main] INFO com.example.test.Example - Mike's royalty: 2080000
11:18:48.219 [main] INFO com.example.test.Example - Tom's royalty: 986000
11:18:48.219 [main] INFO com.example.test.Example - Grace's royalty: 970000
11:18:48.219 [main] INFO com.example.test.Example - Smith's royalty: 1500000
~~~

## 14.9 다수의 Subscriber 에게 Flux 를 멀티캐스팅하기 위한 Operator
* Subscriber 가 구독을 하면 Upstream 에서 emit 된 데이터가 구독중인 모든 Subscriber 에게 멀티캐스팅 되는데,
* 이를 가능하게 해주는 Operator 들은 Cold Sequence 를 Hot Sequence 로 동작하게 하는 특징이 있음

### publish
* 구독을 하더라도 구독 시점에 즉시 데이터를 emit 하지 않고, connect 를 호출하는 시점에 데이터를 emit
* 다수의 Subscriber와 Flux를 공유함
* 즉, Cold Sequence를 Hot Sequence로 변환

#### publish sample 1
* subscriber1, subscriber2 모두 전달 받음
* subscriber3 은 4,5 만 받음

~~~java
public static void main(String[] args) throws InterruptedException {
    ConnectableFlux<Integer> flux =
            Flux
                    .range(1, 5)
                    .delayElements(Duration.ofMillis(300L))
                    .publish();

    Thread.sleep(500L);
    flux.subscribe(data -> log.info("# subscriber1: {}", data));

    Thread.sleep(200L);
    flux.subscribe(data -> log.info("# subscriber2: {}", data));

    flux.connect();

    Thread.sleep(1000L);
    flux.subscribe(data -> log.info("# subscriber3: {}", data));

    Thread.sleep(2000L);
}
~~~

~~~
11:24:59.625 [parallel-1] INFO com.example.test.Example - # subscriber1: 1
11:24:59.626 [parallel-1] INFO com.example.test.Example - # subscriber2: 1
11:24:59.931 [parallel-2] INFO com.example.test.Example - # subscriber1: 2
11:24:59.931 [parallel-2] INFO com.example.test.Example - # subscriber2: 2
11:25:00.237 [parallel-3] INFO com.example.test.Example - # subscriber1: 3
11:25:00.237 [parallel-3] INFO com.example.test.Example - # subscriber2: 3
11:25:00.538 [parallel-4] INFO com.example.test.Example - # subscriber1: 4
11:25:00.538 [parallel-4] INFO com.example.test.Example - # subscriber2: 4
11:25:00.538 [parallel-4] INFO com.example.test.Example - # subscriber3: 4
11:25:00.843 [parallel-5] INFO com.example.test.Example - # subscriber1: 5
11:25:00.843 [parallel-5] INFO com.example.test.Example - # subscriber2: 5
11:25:00.843 [parallel-5] INFO com.example.test.Example - # subscriber3: 5
~~~

#### publish sample 2

~~~java
public class Example {
    private static ConnectableFlux<String> publisher;
    private static int checkedAudience;
    static {
        publisher =
                Flux
                        .just("Concert part1", "Concert part2", "Concert part3")
                        .delayElements(Duration.ofMillis(300L))
                        .publish();
    }

    public static void main(String[] args) throws InterruptedException {
        checkAudience();
        Thread.sleep(500L);
        publisher.subscribe(data -> log.info("# audience 1 is watching {}", data));
        checkedAudience++;

        Thread.sleep(500L);
        publisher.subscribe(data -> log.info("# audience 2 is watching {}", data));
        checkedAudience++;

        checkAudience();

        Thread.sleep(500L);
        publisher.subscribe(data -> log.info("# audience 3 is watching {}", data));

        Thread.sleep(1000L);
    }

    public static void checkAudience() {
        if (checkedAudience >= 2) {
            publisher.connect();
        }
    }
}
~~~

~~~
11:29:27.300 [parallel-1] INFO com.example.test.Example - # audience 1 is watching Concert part1
11:29:27.301 [parallel-1] INFO com.example.test.Example - # audience 2 is watching Concert part1
11:29:27.613 [parallel-2] INFO com.example.test.Example - # audience 1 is watching Concert part2
11:29:27.613 [parallel-2] INFO com.example.test.Example - # audience 2 is watching Concert part2
11:29:27.613 [parallel-2] INFO com.example.test.Example - # audience 3 is watching Concert part2
11:29:27.918 [parallel-3] INFO com.example.test.Example - # audience 1 is watching Concert part3
11:29:27.918 [parallel-3] INFO com.example.test.Example - # audience 2 is watching Concert part3
11:29:27.918 [parallel-3] INFO com.example.test.Example - # audience 3 is watching Concert part3
~~~

### autoConnect
* publish() 는 구독이 발생하더라도 connect() 를 호출하기전까지는 데이터를 emit 하지 않기 때문에 코드상에서 connect() 를 직접 호출해야함
* 반면에 autoConnect 는 파라미터로 지정하는 숮자만큼의 구독이 발생하는 시점에 Upstream 소스에 자동으로 연결이 되기 때문에, 별도의 connect() 호출이 필요 없음

#### autoConnect sample 1

~~~java
public static void main(String[] args) throws InterruptedException {
    Flux<String> publisher =
            Flux
                    .just("Concert part1", "Concert part2", "Concert part3")
                    .delayElements(Duration.ofMillis(300L))
                    .publish()
                    .autoConnect(2);

    Thread.sleep(500L);
    publisher.subscribe(data -> log.info("# audience 1 is watching {}", data));

    Thread.sleep(500L);
    publisher.subscribe(data -> log.info("# audience 2 is watching {}", data));

    Thread.sleep(500L);
    publisher.subscribe(data -> log.info("# audience 3 is watching {}", data));

    Thread.sleep(1000L);
}
~~~

~~~
11:33:30.398 [parallel-1] INFO com.example.test.Example - # audience 1 is watching Concert part1
11:33:30.399 [parallel-1] INFO com.example.test.Example - # audience 2 is watching Concert part1
11:33:30.705 [parallel-2] INFO com.example.test.Example - # audience 1 is watching Concert part2
11:33:30.705 [parallel-2] INFO com.example.test.Example - # audience 2 is watching Concert part2
11:33:30.705 [parallel-2] INFO com.example.test.Example - # audience 3 is watching Concert part2
11:33:31.010 [parallel-3] INFO com.example.test.Example - # audience 1 is watching Concert part3
11:33:31.010 [parallel-3] INFO com.example.test.Example - # audience 2 is watching Concert part3
11:33:31.010 [parallel-3] INFO com.example.test.Example - # audience 3 is watching Concert 
~~~



### refCount
* 파라미터로 입력된 숫자만큼의 구독이 발생하는 시점에 Upstream 소스에 연결되며, 모든 구독이 취소되거나 Upstream 의 데이터 emit 이 종료되면 연결이 해체됨
* 무한 스트림 상황에서 모든 구독이 취소될 경우 연결을 해제하는데 사용할수 있음

#### refCount sample 1
* 1개의 구독이 발생하는 시점에 Upstream 소스에 연결되도록 함
  *  Disposable disposable = publisher.subscribe(data -> log.info("# subscriber 1: {}", data));
* 첫번쨰 구독이 발생한 이후 2.1초에 구독을 해제
  * disposable.dispose()
* 이 시점에 모든 구독이 취소된 상태이기 때문에 연결이 해제됨
* 두번째 구독이 발생할 경우 Upstream 소스에 다시 연결됨
  * publisher.subscribe(data -> log.info("# subscriber 2: {}", data)); 

~~~java
public static void main(String[] args) throws InterruptedException {
    Flux<Long> publisher =
            Flux
                    .interval(Duration.ofMillis(500))

//                    .publish().autoConnect(1);
                    .publish().refCount(1);
    Disposable disposable = publisher.subscribe(data -> log.info("# subscriber 1: {}", data));

    Thread.sleep(2100L);
    disposable.dispose();

    publisher.subscribe(data -> log.info("# subscriber 2: {}", data));

    Thread.sleep(2500L);
}
~~~

* .publish().autoConnect(1)
~~~
11:32:08.776 [parallel-1] INFO com.example.test.Example - # subscriber 1: 0
11:32:09.275 [parallel-1] INFO com.example.test.Example - # subscriber 1: 1
11:32:09.780 [parallel-1] INFO com.example.test.Example - # subscriber 1: 2
11:32:10.280 [parallel-1] INFO com.example.test.Example - # subscriber 1: 3
11:32:10.778 [parallel-1] INFO com.example.test.Example - # subscriber 2: 4
11:32:11.280 [parallel-1] INFO com.example.test.Example - # subscriber 2: 5
11:32:11.776 [parallel-1] INFO com.example.test.Example - # subscriber 2: 6
11:32:12.276 [parallel-1] INFO com.example.test.Example - # subscriber 2: 7
11:32:12.778 [parallel-1] INFO com.example.test.Example - # subscriber 2: 8
~~~

* publish().refCount(1)
~~~
11:35:29.240 [parallel-1] INFO com.example.test.Example - # subscriber 1: 0
11:35:29.743 [parallel-1] INFO com.example.test.Example - # subscriber 1: 1
11:35:30.240 [parallel-1] INFO com.example.test.Example - # subscriber 1: 2
11:35:30.743 [parallel-1] INFO com.example.test.Example - # subscriber 1: 3
11:35:31.347 [parallel-2] INFO com.example.test.Example - # subscriber 2: 0
11:35:31.845 [parallel-2] INFO com.example.test.Example - # subscriber 2: 1
11:35:32.350 [parallel-2] INFO com.example.test.Example - # subscriber 2: 2
11:35:32.849 [parallel-2] INFO com.example.test.Example - # subscriber 2: 3
11:35:33.349 [parallel-2] INFO com.example.test.Example - # subscriber 2: 4
~~~

# 15. Spring Webflux 개요

## 15.1 Spring Webflux의 탄생 배경

- Spring WebFlux 는 리액티브 웹 애플리케이션 구현을 위해 Spring 5.0 부터 지원하는 리액티브 웹 프레임워크
- 대량의 요청 트래픽을 Spring MVC(Blocking I/O) 방식이 처리하지 못하는 사황이 잦아짐에 따라 적은 수의 스레드로 대량의 요청을 안정적으로 처리할수 있는 Non-Blocking I/O 방식의 Webflux 가 탄생하게 됨

## 15.2 Spring Webflux 의 기술 스택

 . | Spring MVC | Spring Webflux
--- | --- | ---
서버 | 서블릿(Servlet) 기반의 프레임워크, apache tomcat 같은 servlet container 에서 Blocking I/O 방식으로 동작 | Non-Blocking I/O 방식으로 동작하는 Netty 등의 서버 엔진에서 동작
서버 API | 서블릿 기반의 프레임워크이기 때문에 서블릿 API 를 사용 | 기본 서버엔진이 Netty 이지만 Jetty 나 Undertow 같은 서버엔진에서 지원하는 이랙티브 스트림즈 어댑터를 통해 리액티브 스트림즈를 지원
보안 | 표준 서블릿 필터를 사용하는 Spring Security 가 서블릿 컨테이너와 통합됨 | WebFilter 를 이용해 Spring Security 를 사용
데이터 액세스 | Spring DATA JDBC, Spring Data JPA, Spring Data MongoDB | Spring Data R2DBC, Non-bolcking I/O 를 지원하는 NoSQL

## 15.3 Spring Webflux 요청 처리 흐름

## 15.4 Spring Webflux 의 핵심 컴포넌트

### HttpHandler
- 다른 유형의 Http 서버 API 로 request 와 response 를 처리하기 위해 추상화됨
- 단 하나의 메서드만 가짐

~~~java
public interface HttpHandler {
    Mono<Void> handle(ServerHttpRequest request, ServerHttpResponse response);
}
~~~

### WebFilter
- spring MVC 의 Servlet Filter 처럼 핸들러가 요청을 처리하기 전에 전처리 작업을 할수 있도록 해줌
- 주로 보안, 세션, 타임아웃 처리 등 애플리케이션에서 공통으로 필요한 전처리에 사용
- WebFilterChain 을 사용하여 원하는 만큼 WebFilter 를 추가할수 있음

~~~java
public interface WebFilter {
    Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain);
}
~~~

#### WebFilter Sample 1
- path 에 books 가 포함되어 있으면 로그를 출력하는 예제

~~~java
@Component
public class BookLogFilter implements WebFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();
        return chain.filter(exchange).doAfterTerminate(() -> {
            if (path.contains("books")) {
                System.out.println("path: " + path + ", status: " +
                        exchange.getResponse().getStatusCode());
            }
        });
    }
}
~~~

### HandlerFilterFunction
- 함수형 기반의 요청 핸들러에 적용할수 있는 Filter 임
- filter 메서드로 정의 되어 있고, 파라미터로 전달받은 HandlerFunction 에 연결됨

~~~java
@FunctionalInterface
public interface HandlerFilterFunction<T extends ServerResponse, R extends ServerResponse> {
    Mono<R> filter(ServerRequest request, HandlerFunction<T> next);
}
~~~

#### HandlerFilterFunction sample
- "/v1/router/books/{book-id}" 호출시 BookRouterFunctionFilter 가 적용되도록 함

~~~java
public class BookRouterFunctionFilter implements HandlerFilterFunction {
    @Override
    public Mono<ServerResponse> filter(ServerRequest request, HandlerFunction next) {
        String path = request.requestPath().value();

        return next.handle(request).doAfterTerminate(() -> {
            System.out.println("path: " + path + ", status: " +
                    request.exchange().getResponse().getStatusCode());
        });
    }
}

@Configuration
public class BookRouterFunction {
    @Bean
    public RouterFunction routerFunction() {
        return RouterFunctions
                .route(GET("/v1/router/books/{book-id}"),
                        (ServerRequest request) -> this.getBook(request))
                .filter(new BookRouterFunctionFilter());
    }

    public Mono<ServerResponse> getBook(ServerRequest request) {
        return ServerResponse
                .ok()
                .body(Mono.just(BookDto.Response.builder()
                        .bookId(Long.parseLong(request.pathVariable("book-id")))
                        .bookName("Advanced Reactor")
                        .author("Tom")
                        .isbn("222-22-2222-222-2").build()), BookDto.Response.class);
    }
}
~~~


### WebFilter 와 HandlerFilterFunction 차이점

WebFilter | HandlerFilterFunction
--- | ---
애플리케이션 내에 정의된 모든 핸들러에 공통으로 동작함. 애너테이션 기반과 함수형 기반의 핸들러에서 동작 | 함수형 기반의 핸들러에서만 동작

### DispatcherHandler
- WebHandler 의 구현체로서 Spring MVC 의 DispatcherServlet 처럼 중앙에서 먼저 요청을 전달 받은 후에 다른 컴포넌트에 요청 처리를 위임함
- Spring Bean 으로 등록되도록 설계되었으며, ApplicationContext 에서 HandlerMapping, HandlerAdapter, Handler ResultHandler 등의 요청 처리를 위한 위임 컴포넌트를 검색

### HandlerMapping
- Spring MVC 에서와 마찬가지로 request 와 handler object 에 대한 매핑을 정의하는 인터페이스
- HandlerMapping 인터페이스를 구현한 클래스로는 RequestMappingHandlerMapping, RouterFunctionMapping 등이 있음

~~~java
public interface HandlerMapping {
    Mono<Object> getHandler(ServerWebExchange exchange);
}
~~~

### HandlerAdapter
- HandlerMapping 을 통해 얻은 핸들러를 직접적으로 호출하는 역할
- 응답 결과로 Mono\<HandlerResult\> 를 리턴 받음

~~~java
public interface HandlerAdapter {
    boolean supports(Object handler);

    Mono<HandlerResult> handle(ServerWebExchange exchange, Object handler);
}
~~~

## 15.5 Spring Webflux 의 Non-Blocking 프로세스 구조
- Blocking I/O 방식의 Spring MVC 는 요청을 처리하는 스레드가 차단될수 있기 때문에 대용량의 thread pool 을 사용해서 하나의 요청을 하나의 스레드가 처리함(thread per request model)
- Non-Blokcing I/O 방식의 Spring Webflux 는 스레드가 차단되지 않기 때문에, 적은 수의 고정된 thread pool 을 사용해서 더 많은 요청을 처리
- Webflux 가 thread 차단 없이 더 많은 요청을 처리할수 있는 이유는 요청 처리 방식으로 이벤트 루프 방식을 사용하기 때문


## 15.6 Spring Webflux 의 스레드 모델
- cpu 코어 개수만큼의 thread 를 생성해서 대량의 요청을 처리함
- 서버측에서 복잡한 연산을 처리하는 등의 cpu 집약적인 작업을 하거나, 클라이언트의 요청으로부터 응답 처리 전 과정안에 Blocking 되는 지점이 존재한다면 오히려 성능이 저하될수 있음
- 이러한 성능 저하를 보완하고자 클라이언트의 요청을 처리하기 위해 서버 엔진에서 제공하는 스레드풀이 아닌 다른 스레드풀을 사용할수 있는 매커니즘(스케줄러)을 제공함

~~~java
@FunctionalInterface
public interface LoopResources extends Disposable {
        int DEFAULT_IO_WORKER_COUNT = Integer.parseInt(System.getProperty("reactor.netty.ioWorkerCount", "" + Math.max(Runtime.getRuntime().availableProcessors(), 4)));
    int DEFAULT_IO_SELECT_COUNT = Integer.parseInt(System.getProperty("reactor.netty.ioSelectCount", "-1"));
    boolean DEFAULT_NATIVE = Boolean.parseBoolean(System.getProperty("reactor.netty.native", "true"));
    long DEFAULT_SHUTDOWN_QUIET_PERIOD = Long.parseLong(System.getProperty("reactor.netty.ioShutdownQuietPeriod", "2"));
    long DEFAULT_SHUTDOWN_TIMEOUT = Long.parseLong(System.getProperty("reactor.netty.ioShutdownTimeout", "15"));

}
~~~

# 16. 애너테이션 기반 컨트롤러

## 16.1 Spring MVC 기반 Controller

~~~java
@RestController
@RequestMapping("/v1/mvc/books")
public class BookMvcController {
    private final BookMvcService bookMvcService;
    private final BookMvcMapper mapper;

    public BookMvcController(BookMvcService bookMvcService, BookMvcMapper mapper) {
        this.bookMvcService = bookMvcService;
        this.mapper = mapper;
    }

    @PostMapping
    public ResponseEntity postBook(@RequestBody BookDto.Post requestBody) {
        Book book = bookMvcService.createBook(mapper.bookPostToBook(requestBody));
        return ResponseEntity.ok(mapper.bookToBookResponse(book));
    }

    @PatchMapping("/{book-id}")
    public ResponseEntity patchBook(@PathVariable("book-id") long bookId,
                                    @RequestBody BookDto.Patch requestBody) {
        requestBody.setBookId(bookId);
        Book book =
                bookMvcService.updateBook(mapper.bookPatchToBook(requestBody));
        return ResponseEntity.ok(mapper.bookToBookResponse(book));
    }

    @GetMapping("/{book-id}")
    public ResponseEntity getBook(@PathVariable("book-id") long bookId) {
        Book book = bookMvcService.findBook(bookId);
        return ResponseEntity.ok(mapper.bookToBookResponse(book));
    }
}
~~~

## 16.2 Spring Webflux 기반 Controller

~~~java
@RestController
@RequestMapping("/v1/books")
public class BookController {
    private final BookService bookService;
    private final BookMapper mapper;

    public BookController(BookService bookService, BookMapper mapper) {
        this.bookService = bookService;
        this.mapper = mapper;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono postBook(@RequestBody BookDto.Post requestBody) {
        Mono<Book> book =
                bookService.createBook(mapper.bookPostToBook(requestBody));

        Mono<BookDto.Response> response = mapper.bookToBookResponse(book);
        return response;
    }

    @PatchMapping("/{book-id}")
    public Mono patchBook(@PathVariable("book-id") long bookId,
                                    @RequestBody BookDto.Patch requestBody) {
        requestBody.setBookId(bookId);
        Mono<Book> book =
                bookService.updateBook(mapper.bookPatchToBook(requestBody));

        return mapper.bookToBookResponse(book);
    }

    @GetMapping("/{book-id}")
    public Mono getBook(@PathVariable("book-id") long bookId) {
        Mono<Book> book = bookService.findBook(bookId);

        return mapper.bookToBookResponse(book);
    }
}

@Mapper(componentModel = "spring")
public interface BookMapper {
    Book bookPostToBook(BookDto.Post requestBody);
    Book bookPatchToBook(BookDto.Patch requestBody);
    BookDto.Response bookToResponse(Book book);
    default Mono<BookDto.Response> bookToBookResponse(Mono<Book> mono) {
        return mono.flatMap(book -> Mono.just(bookToResponse(book)));
    }
}
~~~

---

### 참고
* https://github.com/bjpublic/Spring-Reactive/tree/main
