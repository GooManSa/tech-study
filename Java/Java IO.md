## Java IO (Blocking IO)

> Java 1.0부터 제공된 전통적인 입출력 방식. java.io 패키지를 중심으로 구성되고, 가장 큰 특징은 **Blocking IO**라는 것
> 

**대표 클래스**

- InputStream, OutputStream
- Reader, Writer
- FileInputStream, BufferedReader 등

```java
BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));
String line = reader.readLine(); // 사용자가 입력까지 스레드 블로킹
System.out.println("입력: " + line);
```

특징

- 단일 스레드로 여러 입출력 작업을 처리하기 어렵다
- 각 연결마다 스레드가 필요 → 동시 연결 수 증가 시 성능 저하

## **Java NIO (Non-blocking IO, Java 1.4)**

> Java 1.4부터 도입된 **New I/O**, 즉 NIO는 비동기 처리를 위한 기능을 제공. 스레드가 입출력 대기 중 블로킹되지 않고, 하나의 스레드가 여러 채널(Channel)을 **이벤트 기반으로 관리**할 수 있게 함
> 

**대표 클래스**

- Channel: 데이터 통신의 주체 (SocketChannel, FileChannel)
- Buffer: 읽고 쓰는 데이터를 담는 컨테이너 (ByteBuffer, CharBuffer)
- Selector: 하나의 스레드가 여러 채널을 관리할 수 있게 해줌

```java
Selector selector = Selector.open();
SocketChannel channel = SocketChannel.open(new InetSocketAddress("localhost", 8080));
channel.configureBlocking(false);
channel.register(selector, SelectionKey.OP_READ);

while (true) {
    selector.select(); // 이벤트가 발생할 때까지 대기
    Set<SelectionKey> keys = selector.selectedKeys();
    for (SelectionKey key : keys) {
        if (key.isReadable()) {
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            ((SocketChannel) key.channel()).read(buffer);
            buffer.flip();
            System.out.println("받은 데이터: " + new String(buffer.array()));
        }
    }
}
```

### **특징**

- 단일 스레드 기반 이벤트 루프 구현 가능
- **Non-blocking**, **Multiplexing**
- 고성능 서버에 적합 (Netty도 이 위에서 구현됨)

| **항목** | **Java IO (기존 Blocking 방식)** | **Java NIO (Non-blocking, 이벤트 기반)** |
| --- | --- | --- |
| 연결 수락 | ServerSocket.accept() | SelectionKey.OP_ACCEPT 이벤트 |
| 클라이언트 연결 | Socket.connect() | SelectionKey.OP_CONNECT 이벤트 |
| 데이터 수신 | InputStream.read() | SelectionKey.OP_READ 이벤트 |
| 데이터 송신 | OutputStream.write() | SelectionKey.OP_WRITE 이벤트 |

## **Java NIO.2 (Java 7)**

> Java 7에서는 파일 시스템을 중심으로 NIO 기능을 확장. 특히 Path, Files 등의 유틸리티가 추가되어 파일 작업이 유연해짐
> 

**주요 기능**

- java.nio.file 패키지
- Path, Files, WatchService API
- 파일 시스템 변경 감지, 비동기 채널 추가

```java
Path path = Paths.get("data.txt");
List<String> lines = Files.readAllLines(path); // 간단한 파일 읽기
Files.write(path, Arrays.asList("Hello", "World")); // 파일 쓰기

WatchService watchService = FileSystems.getDefault().newWatchService();
path.getParent().register(watchService, StandardWatchEventKinds.ENTRY_MODIFY);
```

**IO vs NIO**

| **항목** | **java.io** | **java.nio** |
| --- | --- | --- |
| 스레드당 연결 | 1:1 | 1:N (Selector를 통한 Multiplexing) |
| 처리 방식 | Blocking | Non-blocking, Event-driven |
| 예시 서버 | Tomcat (기본), Jetty | Netty, Undertow |

기존 IO는 클라이언트 요청 하나마다 스레드를 할당해 작업을 처리하는 반면, NIO는 Selector를 통해 단일 스레드가 **수천 개의 채널**을 관리하며, I/O 준비 상태만 처리하기 때문에 효율적

**왜 NIO를 도입하게 되었을까?**

**스레드 모델의 한계**

- 요청 수가 많아질수록 스레드 수가 늘어나고 → 문맥 전환(Context Switching) 비용 증가
- 많은 커넥션을 유지하려면 이벤트 기반 처리 필요

이를 해결하기 위해 NIO와 Netty 같은 프레임워크의 등장.
**Spring WebFlux**도 내부적으로 Netty 기반의 NIO로 작동, 고성능 비동기 API를 제공.

# Java 입출력 모델과 스레드 모델의 진화

**전통적인 Thread-Per-Request 방식**

- 기본 Java IO (java.io)는 요청마다 하나의 스레드를 사용한다.
- 이 방식은 **Blocking I/O** 기반이다. 입력이나 출력이 완료될 때까지 해당 스레드는 대기 상태로 머무른다.

```java
ServerSocket serverSocket = new ServerSocket(8080);
while (true) {
    Socket socket = serverSocket.accept(); // 블로킹
    new Thread(() -> handle(socket)).start(); // 요청마다 스레드 생성
}
```

**문제점**

- 커넥션 수가 증가하면 스레드 수가 증가 → 메모리/스케줄링 오버헤드 증가
- 특히 I/O가 느린 작업 (파일, 네트워크)에서 비효율적

## **NIO와 이벤트 기반 스레드 모델**

- Java NIO (java.nio)는 Non-blocking I/O를 도입하여, 하나의 스레드가 여러 채널을 다룰 수 있게 함
- Selector, Channel, Buffer 조합으로 이벤트 기반 처리

| **구성요소** | **역할** |
| --- | --- |
| **Channel** | I/O 대상 (소켓, 파일 등) |
| **Buffer** | 데이터 저장 버퍼 |
| **Selector** | 이벤트 감지 및 분기 처리 |

```java
Selector selector = Selector.open();
channel.register(selector, SelectionKey.OP_READ);

while (true) {
    selector.select(); // 이벤트 감지
    for (SelectionKey key : selector.selectedKeys()) {
        if (key.isReadable()) {
            // 읽기 처리
        }
    }
}
```

**스레드 모델 변화**

- **Thread-Per-Connection → Event-Driven Thread (1:N)**
- Netty, Undertow, Spring WebFlux 등이 이 모델을 채택

**스레드 풀 + Future 기반 비동기 처리**

Java 5 이후 ExecutorService, Future 등으로 작업을 큐에 넣고 스레드 풀에서 처리하는 방식이 등장

```java
ExecutorService pool = Executors.newFixedThreadPool(10);
Future<String> result = pool.submit(() -> longTask());
```

- 여전히 **블로킹 I/O**일 경우, 스레드는 작업 완료까지 점유됨
- I/O 자체가 블로킹이면 의미 없음

**CompletableFuture (Java 8)**

비동기 연산을 연결하고, 콜백처럼 작업을 이어갈 수 있는 API

```java
CompletableFuture.supplyAsync(() -> loadData())
                 .thenApply(data -> transform(data))
                 .thenAccept(result -> render(result));
```

- 비동기 + 논블로킹 연산 조합 가능 (병렬성의 이점 → )
- 하지만 여전히 I/O가 블로킹이면 한계

**Reactive Streams와 Netty 기반 WebFlux (Java 9~)**

I/O까지 완전히 비동기/논블로킹으로 처리할 수 있도록 설계된 **Reactive Streams API** 등장

→ Netty 위에서 동작하며, 이벤트 루프 기반

| **역할** | **예시** |
| --- | --- |
| **Publisher** | Mono, Flux |
| **Subscriber** | .subscribe() |
| **Processor** | .map(), .flatMap() 등 |
| **Backpressure** | request(n) 방식으로 수요 조절 |

```java
@GetMapping("/orders")
public Flux<Order> getOrders() {
    return orderService.findAllAsync(); // Non-blocking
}
```

- 클라이언트에게는 여전히 요청-응답 구조
- 내부적으로는 Netty의 이벤트 루프 기반으로 동작
- 하나의 스레드가 수천 개의 커넥션을 비동기 처리 가능

**요약**

| **진화 단계** | **모델** | **주요 특징** |
| --- | --- | --- |
| Thread-Per-Request | java.io, Tomcat | 스레드 하나가 요청 하나 담당 |
| Event-Driven (NIO) | java.nio, Netty | Selector 기반 이벤트 처리 |
| 비동기 Future 모델 | Executor, CompletableFuture | 작업 비동기화, 여전히 블로킹 가능성 |
| Reactive Streams | Flow, Mono, Flux | 완전한 논블로킹 + Backpressure 지원 |

### **결론: 왜 입출력 모델과 스레드 모델은 함께 진화했을까?**

- 입출력 작업은 대부분 “느린 작업”
- 효율적인 입출력은 곧 스레드 사용 효율과 직결
- 많은 연결을 효율적으로 처리하려면 → **비동기 + 논블로킹 + 이벤트 루프**

### **정리**

> 스레드 모델의 진화는 결국 입출력(I/O) 병목을 해결하고, 자원을 효율적으로 사용하기 위한 방향으로 발전해왔다.
>
