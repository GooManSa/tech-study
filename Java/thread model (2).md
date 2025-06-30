# 1. Reactive

> by Wiki
> 
> 
> 리액티브 프로그래밍은 데이터 스트림과 변경 사항 전파를 중심으로하는 비동기 프로그래밍 패러다임이다. 이것은 프로그래밍 언어로 정적 또는 동적인 데이터 흐름을 쉽게 표현할 수 있어야하며, 데이터 흐름을 통해 하부 실행 모델이 자동으로 변화를 전파할 수 있는 것을 의미한다.
> 
> 즉, 데이터 스트림을 선언적으로 조합, 이벤트 기반으로 처리 흐름을 제어
> 

- 기존 동기 방식의 한계
- Thread 기반 → 수천 개 요청에서 자원 낭비
- `Future` → 콜백 지옥, 조합 불편
- `CompletableFuture` → 개선되었지만 여전히 명시적 스레드 제어 필요
- Reactive의 필요성
    - **I/O 병목 해소**
    - **자원 효율적인 처리**
    - **높은 처리량을 가진 시스템 요구에 대응**

## 2. Reactive Streams

- **비동기 스트림 처리 표준 인터페이스**
- 주요 목표: **표준화된 Backpressure 지원**

### 핵심 개념

- **Publisher**: 데이터를 발행함
- **Subscriber**: 데이터를 구독하며 처리
- **Backpressure**: 구독자가 처리 가능한 만큼만 데이터를 요청

```java
publisher.subscribe(new Subscriber<T>() {
  public void onSubscribe(Subscription s) {
      s.request(1); // 한 번에 하나씩 요청
  }
  public void onNext(T item) { ... }
  public void onError(Throwable t) { ... }
  public void onComplete() { ... }
});
```

**메서드**

`onSubscribe(Subscription subscription)`

- Publisher가 Subscriber와 연결시, 초기 handshake로 메서드 호출됨
- Subscriber에게 데이터 얼마나 받을지 요청하게 함
- 백프레셔의 시작점

`onNext(T data)`

- Publisher가 Subscriber에게 데이터를 emit(방출)할 때마다 호출
- subscription.request(n)을 통해 요청된 개수만큼 호출
- 실제 데이터 응답 처리하는 핵심 메서드
- Flux의 경우 여러번 호출될 수 있다. Mono의 경우는 한번 or 0

`onComplete()`

- 모든 데이터가 성공정으로 emit되었을 때
- Publisher가 데이터 스트림의 종료를 알림 (에러 없이 끝)

`onError()`

- 데이터 전송 중 예외가 발생했을 때 호출
- Subscriber에게 에러 발생 시그널을 전달
- 이 이후로 onNext나 onComplete는 호출되지 않음

전체 흐름

```
1. onSubscribe(subscription)
   → Subscriber가 데이터를 받을 준비가 되었음을 Publisher에 알림
   → subscription.request(n)으로 몇 개의 데이터를 받을지 요청

2. onNext(data)
   → Publisher가 데이터를 emit함
   → 받은 후 다음 데이터를 계속 요청할 수도 있음

3. onComplete()
   → 모든 데이터를 정상적으로 받은 경우

   OR

4. onError(Throwable)
   → 처리 도중 문제가 발생한 경우
```

## Reactor - Mono & Flux (Java 8+)

> org.reactiveStreams.Publisher의 구현체로, 스프링 WebFlux의 기본 리턴 타입
> 

### **Mono**

단일 데이터 또는 0개를 처리

단일 객체, API 응답, 단건 DB 조회 등

**메서드**

- Mono.just(T.data)
    - 단일 데이터 emit(방출)
- Mono.empty()
    - 아무것도 emit하지 않음
- Mono.error()
    - 에러 발생시키는 Mono

**시그널**

- onNext(T)
    - 데이터 1건 emit될 때마다 호출됨
- onError(Throwable)
    - 에러 발생 시 호출되고 스트림 종료
- onComplete()
    - 더 이상 emit할 데이터가 없을 때 호출됨

### **Flux**

다중 데이터 스트림 (0개 이상의 데이터)

리스트 조회, 지속적인 스트리밍 응답, 이벤트 스트림 등

**메서드**

| Flux.just(T...) | 여러 개의 데이터를 emit | Flux.just(1,2,3) |
| --- | --- | --- |
| Flux.fromIterable(Iterable<T>) | 컬렉션을 Flux로 변환 | Flux.fromIterable(List.of("a","b")) |
| Flux.range(start, count) | 숫자 범위 emit | Flux.range(1, 5) → 1~5 emit |
| Flux.interval(Duration) | 일정 간격으로 Long 타입 emit (무한 스트림) | Flux.interval(Duration.ofSeconds(1)) |

**시그널**

| onNext(T) | 데이터 1건 emit될 때마다 호출됨 |
| --- | --- |
| onError(Throwable) | 에러 발생 시 호출되고 스트림 종료 |
| onComplete() | 더 이상 emit할 데이터가 없을 때 호출됨 |

## Flow API (Java 9+)

> 기존 Reactive Streams는 자바 표준이 아니었고, Reactor 또한 reactiveStreams 구현체이기에 Reactive Streams 표준을 Java 표준으로 내재화하기 위한 (JDK 공식 지원) API
> 
- Java 9의 `java.util.concurrent.Flow` 패키지로 공식 Reactive Streams 도입
- 주요 인터페이스
    - `Publisher<T>`: 데이터 생산자
    - `Subscriber<T>`: 데이터 소비자
    - `Subscription`: 요청 및 취소 관리
    - `Processor<T,R>`: Publisher + Subscriber 역할 병합
- 구조 요약
    - Publisher → Subscription → Subscriber
- 특징
    - `request(n)`을 통해 **Backpressure** 제어
    - Non-blocking 기반 처리

**Flow API 주요 인터페이스**

```java
interface Publisher<T> {
    void subscribe(Subscriber<? super T> s);
}

interface Subscriber<T> {
    void onSubscribe(Subscription s);
    void onNext(T t);
    void onError(Throwable t);
    void onComplete();
}
```

### 기존 Servlet/Tomcat 서버 모델과 뭐가 다를까

**기존 서버 모델**

- 각 요청이 생길때마다, 스레드 1개가 할당됨 (ex : Tomcat의 쓰레드 풀)
- 그 스레드가 DB, API를 기다리는 동안 해당 스레드는 블로킹되어 놀고 있음
- 동접자 증가 → 스레드 수 급증 → blocking된 스레드가 많아져서 메모리/CPU 한계 초과

**Reactive 모델**

- 고정된 스레드 풀로 작동
- 이벤트 루프 구조
- 요청이 들어오면, 작은 작업 단위(Task)로 쪼개져서 처리
- 외부 작업(DB, API) 요청시 결과가 준비될 때까지 다른 작업을 먼저 수행

⚠️ 그럼 node.js같은 싱글 스레드 이벤트 루프 구조와 같은 것일까?

- Node.js
    - 단일 스레드 이벤트 루프
    - 모든 작업을 논블로킹 콜백으로 처리
    - 이벤트 큐 작업은 순차적 처리, 작업이 끝나야 다음으로 넘어감
    - 스레드 풀을 통해 blocking I/O 위임
    
    ![image.png](attachment:9e99a12b-b7f5-4db1-932a-885422449e91:image.png)

- WebFlux
  <img width="674" alt="image" src="https://github.com/user-attachments/assets/9ca6d2de-e677-4d3b-8402-97db162f95d8" />

  
    
요약 : 
<img width="978" alt="image" src="https://github.com/user-attachments/assets/92086281-3d4c-491b-aad9-4024dd05c45e" />


 
- Servlet 기반
    - 기본적으로 멀티스레드 기반
    - Tomcat 기반에서 Servlet 3.1+ 비동기 기능을 이용
        - Reactor의 내부 스케줄러와 논블로킹 흐름으로 유사하게 구현

- Netty 기반
    - 이벤트 루프 기반은 동일 (적은 스레드 풀로)
    - node와 다르게 N개의 이벤트 루프 스레드로 병렬 처리가 가능


## **Spring WebFlux 소개**

### **WebFlux란?**

- Spring 5에서 등장한 **Non-blocking 웹 프레임워크**
- Servlet 기반인 Spring MVC와 다르게 **Netty 기반**
- 내부적으로 **Reactor**를 사용해 Mono / Flux로 처리

| **항목** | **Spring MVC** | **Spring WebFlux** |
| --- | --- | --- |
| 동작 방식 | Blocking | Non-blocking |
| 기본 서버 | Tomcat | Netty |
| Return 타입 | String, Model | Mono<T>, Flux<T> |
| 확장성 | 낮음 (Thread-per-request) | 높음 (Event loop 기반) |

```java
@RestController
public class UserController {

    @GetMapping("/user")
    public Mono<User> getUser() {
        return userService.findUserAsync(); // Non-blocking
    }
}
```

### 6. 흐름

예시 흐름

1. 클라이언트: /user 요청
2. 서버에서 Mono<User> 반환 → Publisher 생성
3. Spring WebFlux가 Subscriber 등록 → onSubscribe()
4. 데이터 준비 완료 시 → onNext(user)
5. 모든 데이터 완료되면 → onComplete()
6. 응답 완료: HTTP 응답 전송

## WebFlux 실전 적용

- 기존 MVC 방식

```java
@GetMapping("/orders")
public List<Order> getOrders() {
    return orderService.findAll(); // 서버 내부 Blocking
}
```

- WebFlux 방식

```java
@GetMapping("/orders")
public Flux<Order> getOrders() {
    return orderService.findAllAsync(); // 서버 내부 Non-blocking
}
```

- 비동기 서비스 예시

```java
public Mono<Order> findById(Long id) {
    return reactiveOrderRepository.findById(id);
}
```

- WebClient로 외부 API 호출

```java
WebClient client = WebClient.create("http://example.com");

Mono<String> result = client.get()
    .uri("/data")
    .retrieve()
    .bodyToMono(String.class);
```

- RestTemplate → Blocking
- WebClient → Non-blocking

## **WebFlux 사용 시 주의점**

### **1. 블로킹 코드 사용 금지**

- JDBC, JPA 등은 Blocking I/O 기반
- 해결 방법: R2DBC 등 Non-blocking 대안 사용

```java
// 잘못된 예 (Blocking DB)
public Mono<User> getUser() {
    return Mono.fromCallable(() -> userRepository.findById(1L)); // 위험
}
```

### **2. Context 전파 어려움**

- ThreadLocal 기반 로깅, 트랜잭션 등 사용 어려움
- 해결: Context API 사용 필요

### **3. 에러 처리 복잡성**

- 기존 try-catch → 사용 불가
- Reactor 전용 .onErrorResume(), .doOnError() 사용

```java
someMono()
  .onErrorResume(e -> Mono.just(defaultValue));
```

## **정리 및 마무리**

### **요약**

| **구분** | **특징** |
| --- | --- |
| Java 9 Flow | Reactive Streams의 공식 Java 표준 |
| Reactor | Spring에서 채택한 Reactive 라이브러리 |
| WebFlux | Non-blocking 기반의 Spring 웹 프레임워크 |
| WebClient | WebFlux에 적합한 HTTP 클라이언트 |

### **어떤 경우 WebFlux를 쓰는 게 좋을까?**

- 수천 개의 동시 요청을 처리하는 **고성능 시스템**
- **I/O 중심** (DB, 외부 API 등) 작업이 많은 시스템
- 서버 자원을 **효율적으로 사용**하고 싶은 경우

### **어떤 경우는 WebFlux가 적합하지 않다?**

- 간단한 CRUD 시스템 (불필요한 복잡성)
- JDBC, JPA만 사용하는 Blocking 환경
