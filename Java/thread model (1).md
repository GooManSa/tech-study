> **Java5 이전 → Runnable과 Thread**
> **Java5 → Callable, Future, Executor, ExecutorService, Executors**
> Java7 → Fork/Join 및 RecursiveTask
> **Java8 → CompletableFuture**
> Java9 → Flow
> 

# Java5 이전 - Thread

> Thread는 Java에서 스레드 생성을 위해 미리 구현해둔 클래스. (멀티스레드를 위해)
> 

Thread의 메서드로는,

- run : 현재 스레드에서 작업을 실행
- start : 새로운 스레드를 생성해 run() 실행
- sleep : 현재 스레드 멈추기 (자원 안놓아줌)
- interrupt : 다른 스레드 깨워 interruptedException 발생
- join : 다른 스레드 작업 끝날때까지 기다리게함

<img width="696" alt="image" src="https://github.com/user-attachments/assets/e3912728-979d-4480-a359-0a9e7ff56c8d" />


### 예시

> 내부에서 run() 메서드를 반드시 구현해야함
run() 메서드 : 실제로 스레드가 행동할 동작을 구현
> 

새로운 스레드에서 현재 스레드의 이름을 출력하는 코드

```java
void threadStart() {
    Thread thread = new MyThread();

    thread.start();
    System.out.println("Hello: " + Thread.currentThread().getName());
}

static class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread: " + Thread.currentThread().getName());
    }
}
```

### **✅ thread.start()**

**synchronized 메서드로, 새로운 스레드를 생성해 run()을 실행하는 메서드**

1. **쓰레드가 실행 가능한지 검사**
    1. 스레드의 상태 (New = 0, Runnable, Waiting, Timed Waiting, Terminated)
    2. New가 아니라면 IllegalThreadStateException 발생
2. **쓰레드를 쓰레드 그룹에 추가**
    1. ThreadGroup class : 서로 관련있는 쓰레드를 하나의 그룹으로 묶어 다루기 위한 장치
    2. 한번에 Interrupt() 등 처리를 공통적으로 할 수 있게끔 함
3. **스레드를 JVM이 실행시킴**
    1. start0() : native 메서드 → JVM에 의해 호출, 내부적으로 run() 호출
    2. 쓰레드 상태가 Runnable로 바뀜 → start는 여러번 호출이 불가
    3. 만약 run() 직접 호출하면 메인 쓰레드가 run() 메서드를 직접 호출

java.lang.Thread

```java
public class Thread implements Runnable {
		...
		//start()
		public synchronized void start() {
        if (threadStatus != 0)
            throw new IllegalThreadStateException();
            
        
        group.add(this);
        boolean started = false;
        
        try {
            start0();
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {}
        }
    }

    private native void start0();
    
    @Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }
}
```

### Runnable

> run() 추상 메서드만을 가지고 있는 함수형 인터페이스 (반환값이 없음)
> 

```java
@FunctionalInterface
public interface Runnable {

    public abstract void run();
    
}
```

람다식 표현

```java
Thread t = new Thread(() -> {
    System.out.println("Hello from thread!");
});
t.start();
```

# Java5 - Future & Callable

> Thread와 Runnable을 직접 사용하는 방식은 한계가 명확함
지나치게 저수준의 API에 의존, 값의 반환이 불가능, 매번 쓰레드 생성과 종료하는 오버헤드
= 쓰레드들 관리가 어려움
APP 개발자의 관심과는 거리가 멈
> 

*기존 Runnable 인터페이스는 start0()에서 native 메서드가 run을 실행하면서 반환값을 얻기가 힘들었다. 반환값을 얻으려면 공용 메모리 혹은 파이프 등을 사용*

### Callable 인터페이스 : 제네릭을 사용해 결과를 받을 수 있음

```java
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

미래에 완료된 Callable의 반환값을 구하기 위해 사용된다.

→ 비동기 작업을 갖고있어, 미래에 실행 결과를 얻도록 도와줌.

- 비동기 작업의 현재상태 확인
- 기다리기
- 결과 얻는 방법 등 제공

### Future 인터페이스 : 미래에 완료된 Callable의 반환값을 구하기 위해 사용된다

```java
public interface Future<V> {

    boolean cancel(boolean mayInterruptIfRunning);
    
    boolean isCancelled();
    
    boolean isDone();
    
    V get() throws InterruptedException, ExecutionException;

    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

- get : **블로킹 방식**으로 결과 가져오기 (timeout 설정 가능)
- isDone, isCancelled : 작업의 완료 여부, 취소 여부 boolean으로 빈환
- cancel : 작업 취소, 취소 여부를 boolean 반환

## Executors, Executor, ExecutorService

<img width="722" alt="image" src="https://github.com/user-attachments/assets/738acd20-7707-42cc-a2b4-9ad3d6fd1c15" />


### **Executor 인터페이스**

쓰레드 풀의 구현을 위한 인터페이스

- 등록된 작업 (Runnable)을 실행하기 위한 인터페이스
- 작업 등록과 작업 실행 중에서 작업 실행만을 책임진다

등록된 작업을 실행하는 책임만 가짐. (Runnable 받음)

```java
public interface Executor{
	void execute(Runnable command);
}
```

→ 개발자가 해당 작업 실행과 쓰레드 사용 및 스케줄링 등으로부터 벗어날 수 있도록 도와줌

### **ExcutorService 인터페이스**

작업(**Runnable, Callable**) 등록을 위한 인터페이스

- Executor 상속, 등록 및 실행을 위한 책임을 가짐
- 쓰레드 풀은 기본적으로 ExecutorService 인터페이스를 구현
- ThreadPoolExecutor, 내부의 blocking queue에 작업을 등록해둠

<img width="696" alt="image" src="https://github.com/user-attachments/assets/b78180f7-56d6-4519-bc64-8a40ab27d0a0" />



**퍼블릭 메서드**

- 라이프싸이클 관리를 위한 기능
    - shutdown, shutdownNow
    - isTerminated, isShutdown
- 비동기 작업을 위한 기능
    - 비동기 작업의 진행을 추적하기 위해 Future를 반환
    - submit : 실행 작업 추가, 작업 상태 결과 포함 Future 반환
    - invokeAll : 모든 결과 나올때까지 대기하는 블로킹 방식 요청. 각각 상태와 결과를 갖는 List<Future> 반환
    - invokeAny : 가장 빨리 실행된 결과가 나올때까지 대기하는 블로킹 방식 요청 → 가장 빨리 완료된 하나의 결과를 Future 반환

❓ Executor는 Runnable만 받는데, ExecutorService는 Runnable과 Callable을 받음

```java
//submit 예시
Future<?> future = executorService.submit(new Runnable() {
    public void run() {
        // 작업 수행
    }
});

// Runnable을 감싸 callable을 만든다
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    Callable<Object> callable = Executors.callable(task, null);
    return submit(callable);
}
```

→ Executor은 task 실행의 책임만 갖고, ExecutorService에서 반환값 및 예외처리 등 책임

invokeAll + get (Callable 작업 

```java
void invokeAll() throws InterruptedException, ExecutionException {
    ExecutorService executorService = Executors.newFixedThreadPool(10);
    Instant start = Instant.now();

		// Callable 작업 1
    Callable<String> hello = () -> {
        Thread.sleep(1000L);
        final String result = "Hello";
        System.out.println("result = " + result);
        return result;
    };

		// Callable 작업 2
    Callable<String> mang = () -> {
        Thread.sleep(4000L);
        final String result = "Java";
        System.out.println("result = " + result);
        return result;
    };

		// Callable 작업 3
    Callable<String> kyu = () -> {
        Thread.sleep(2000L);
        final String result = "kyu";
        System.out.println("result = " + result);
        return result;
    };

		// invokeAll, future 반환
    List<Future<String>> futures = executorService.invokeAll(Arrays.asList(hello, mang, kyu));
    for(Future<String> f : futures) {
    
		    // future get (blocking 방식)
        System.out.println(f.get());
    }

    System.out.println("time = " + Duration.between(start, Instant.now()).getSeconds());
    executorService.shutdown();
}
```

⚠️ ExecutorService를 만들어 작업 실행시, shutdown 호출 전까지 계속해서 다음 작업 대기.

작업 완료시 shutdown 명시적 호출 필수

대표적인 ExecutorService 구현체 - ThreadPoolTaskExecutor()

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean(name = "preProcessOrderExecutor")
    public Executor preProcessOrderExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(100);
        executor.setQueueCapacity(150);
        executor.setThreadNamePrefix("preProcessOrderExecutor Async-");
        // 자동 termination
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);
        // 설정 적용
        executor.initialize();
        return executor;
    }
```

> @Async 이벤트 핸들러는 Spring의 비동기 스레드 풀을 사용한다. 기본 설정된 스레드 풀은 SimpleAsyncTaskExecutor
> 

### ScheduledExecutorService 인터페이스

ExecutorService인데, 특정 시간마다 주기적으로 작업을 실행시키는 메서드가 추가됨

→ 스케줄링 + 비동기 병렬 스레드 작업시 사용

## Java8 - CompletableFuture 클래스

> 기존의 Future 기반으로 외부에서 완료시킬 수 있음. 
CompletionStage 인터페이스 : 작업들을 중첩시키거나 완료 후 콜백을 위한 인터페이스
Future에선 불가능했던 ‘몇 초 이내에 응답 안오면 기본값 반환” 과 같은 작업 가능
외부에서 작업을 완료시킬 수 있고, 콜백 등록 및 Future조합 가능
> 

*기존 Future의 한계 :*

- *외부에서 완료시킬 수 없고, get의 타임아웃 설정으로만 완료가 가능하다*
- *블로킹 코드(get)을 통해서만 이후의 결과를 처리할 수 있다*

### CompletableFuture 기능

1. 비동기 작업 실행
    - runAsync : 반환값 없음, 비동기로 작업 실행 콜 (runnable 추적)
    - supplyAsync : 반환값 있음, 비동기로 작업 실행 콜 (supplier 추적)
    
    <aside>
    
    
    **💡 왜 callable을 받지 않는가?**
    CompletableFuture은 비동기 함수형 체이닝 (thenXxx)를 위해 설계되어, 예외없는 함수형 인터페이스와 잘 맞는다.
    
    Callable<T>는 Exception을 Throw하기에 CompletableFuture의 흐름과 어울리지 않는다.
    
    </aside>
    
    - 기본적으로 Java7의 ForkJoinPool의 commonPool() 사용해 작업을 실행할 쓰레드를 쓰레드 풀로부터 얻어 실행시킴
        - 원하는 쓰레드 풀을 사용하려면, ExecutorService를 파라미터로 넘겨주면 됨
    
    ```java
    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
        // 백그라운드 작업 수행
        return "결과";
    });
    
    String result = future.get(); // 결과 가져오기 (blocking)
    ```
    

1. 작업 콜백
    - thenApply : 반환 값을 받아 다른 값을 반환 / 함수형 인터페이스 Function을 파라미터로
    - thenAccept : 반환 값 받아 처리, 값 반환하지 않음 / 함수형 인터페이스 Consumer를 파라미터로
    - thenRun : 반환값 받지 않고 다른 작업 실행 / 함수형 인터페이스 Runnable을 파라미터로
    
    ```java
    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello");
    
    future
        .thenApply(s -> s + " World")   // 값을 변환해서 넘김
        .thenAccept(System.out::println); // 최종 소비 (Consumer)
    ```
    

1. 작업 조합
    - thenCompose : 두 작업이 이어서 실행하도록 조합, 앞선 작업의 결과 받아 사용 가능 / 함수형 인터페이스 Function을 파라미터로
    - thenCombine : 두 작업을 독립적으로 실행, 둘다 완료되었을때 콜백 실행 / 함수형 인터페이스 Function을 파라미터로
    - allOf : 여러 작업 동시 실행, 모든 작업 결과에 콜백 실행
    - anyOf : 여러 작업들 중 가장 빨리 끝난 하나의 결과에 콜백 실행
    
    ```java
    CompletableFuture<Integer> a = CompletableFuture.supplyAsync(() -> 10);
    CompletableFuture<Integer> b = CompletableFuture.supplyAsync(() -> 20);
    
    CompletableFuture<Integer> sum = a.thenCombine(b, Integer::sum);
    System.out.println(sum.get()); // 30 (블로킹)
    ```
    

1. 예외 처리
    - excepytionally : 발생한 에러 받아 예외 처리 / 함수형 인터페이스 Function을 파라미터로
    - handle, handleAsync : 결과값,에러 반환받아 에러발생 or 아닌경우 모두 처리 가능 / 함수형 인터페이스 BiFunction을 파라미터로
    
    ```java
    CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
        if (true) throw new RuntimeException("문제 발생!");
        return "정상 결과";
    }).exceptionally(e -> {
        System.out.println("에러: " + e.getMessage());
        return "기본값";
    });
    ```
    

## Java9 - Flow API

- 비동기 스트림 처리를 위해 도입
- Reactive Streams 표준화
- Spring WebFlux나 RxJava 등의 Reactive 프로그래밍 프레임워크에서 사용
