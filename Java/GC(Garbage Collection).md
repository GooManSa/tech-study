
> 📄 **목차**
> 1. GC란
> 2. GC 알고리즘
> 3. JVM의 GC
> 4. GC가 일어나는 과정
> 5. GC의 종류
> 6. 면접 대비 질문

 <br>



## ❓ GC란
GC(Garbage Collection)는 `JVM의 Heap영역`에서 더 이상 사용되지 않는 객체를 자동으로 삭제해주는 메모리 관리 프로세스입니다.
- Heap 영역: new 키워드를 통해 생성된 Object 타입의 데이터가 저장되는 공간 <br>
  ex) String, List, Map, 사용자 정의 클래스 등

<br>

### GC가 왜 필요할까?
GC는 프로그램이 동적으로 할당된 메모리 중 더 이상 필요 없는 영역을 알아서 해제해줍니다.


✅ **장점**
- 메모리 누수 방지
- 해제된 메모리에 접근하는 오류 방지
- 이중 해제 방지

⚠️ **단점**
- GC 수행 자체가 오버헤드 (성능 저하 가능)
- 해제 타이밍을 개발자가 제어하기 어려움
- 실시간성이 중요한 시스템에서는 적합하지 않을 수 있음


<hr>


## 🔍 GC 알고리즘
### 1️⃣ Reference Counting
- 각 객체가 몇 개의 참조를 받고 있는지 카운트(reference count)합니다.
- reference count가 0이 되면 수거 대상

<br>

🚨 **문제점** <br>
순환 참조 발생 시 참조 수가 0이 되지 않아 메모리 누수 발생


<br>

### 2️⃣ Mark&sweep
- GC Root에서 시작하여 객체를 탐색 → 도달 가능한 객체를 마킹(Mark)
- 도달하지 못한 객체는 수거(Sweep)

  ⇒ 순환 참조 문제 해결 가능👍

<br>

**Mark&sweep 특징**
1. 의도적으로 GC를 실행시켜야한다.
2. 어플리케이션과 GC실행이 병행된다. <br>
   ⇒  즉 어느 순간에는 실행 중인 어플리케이션이 GC에게 컴퓨터 리소스들을 내줘야 한다는 말이죠. 따라서 어플리케이션의 사용성을 유지하면서 효율적이게 GC를 실행하는 것이 꽤나 어려운 최적화 작업이라고 합니다.

<br><br>

## JVM의 GC
### JVM의 구조(Java8기준)

<img width="540" alt="스크린샷 2025-06-29 오후 10 13 10" src="https://github.com/user-attachments/assets/f06dc676-bf18-429c-a6f4-3914d5cf7415" />

- **Class Loader**
  : 바이트코드를 읽고, 클래스 정보를 메모리의 Heap/Method Area에 저장
  - **JVM Memory**
  : 실행 중인 프로그램의 정보가 올라가 있는 메모리
  - **Execution Engine(실행 엔진)**
  : 바이트 코드를 네이티브 코드로 변화시켜주고, GC를 실행

<br><br>

### JVM 메모리 구조
JVM은 OS로부터 메모리를 할당 받은 후, 해당 메모리를 용도에 따라 여러 영역으로 나눠서 관리합니다.

<img width="579" alt="스크린샷 2025-06-29 오후 10 13 25" src="https://github.com/user-attachments/assets/297fe332-8d30-458d-a17c-43d92472f5e0" />

- 모든 스레드가 공유하는 영역
    - Method Area + Heap
- 각 스레드마다 공유하게 생성되며, 스레드 종료시 소멸하는 영역
    - JVM Language Stacks + PC Registers + Native Method Stacks

<br><br>


## ⚙️ GC의 수거 대상
**GC Root**를 기준으로 참조되는 객체(Reachable Objects)와 참조관계가 끊긴 객체(Unreachable Objects)를 구분한 후, `Unreachable 객체만 수거`합니다. <br>

<img width="582" alt="스크린샷 2025-06-29 오후 10 13 36" src="https://github.com/user-attachments/assets/13d377c7-69a7-46e7-b69a-94f93ab53f8c" />


### ✅ GC루트가 될 수 있는 조건
|  구분       | 설명               |
|:----------|:-------------------|
| Stack 영역  | 지역 변수, 매개변수 등      |
| Method 영역 | static 변수          |
| JNI 참조    | 네이티브 코드에서 참조 중인 객체 |

<br>

> **💬 JNI(Java Native Interface)이란?** <br>
> JNI는 자바 가상머신(JVM)위에서 실행되고 있는 자바코드가 네이티브 응용 프로그램(하드웨어와 운영 체제 플랫폼에 종속된 프로그램들) 그리고 C, C++ 그리고 어셈블리 같은 다른 언어들로 작성된 라이브러리들을 호출하거나 반대로 호출되는 것을 가능하게 하는 프로그래밍 프레임워크


<br><br>

## 🔄 GC의 동작 순서

Java의 GC는 주로 `Mark and Sweep 알고리즘`을 사용합니다. 필요에 따라 Compact 단계도 추가되어 메모리 단편화를 방지합니다.
<br>

**1️⃣ Mark 단계**
GC Root로 부터 모든 reachable한 객체와 unreachable한 객체를 식별한후, unreachable한 객체를 마킹

**2️⃣ Sweep 단계**
마킹되지 않은 객체(unreachable한 객체)를 Heap에서 삭제

**3️⃣ Compaction 단계 (선택적)**
살아남은 객체들을 한쪽으로 모아 빈 공간을 정리
(= 메모리 단편화를 막아주는 작업)

<br><br>

## ⏱️ GC는 언제 일어날까?
### Heap의 구조
<img width="592" alt="스크린샷 2025-06-29 오후 10 13 48" src="https://github.com/user-attachments/assets/0eec6e46-eb45-4b7e-981c-6fe6e6e3d8e0" />

- **Young Generation**: 새로 생성된 객체가 저장되는 공간
    - Eden: 새 객체 저장
    - Survivor(0/1): Eden에서 살아남은 객체 임시 보관
- **Old Generation**: Young Generation에서 오래 살아남은 객체가 저장되는 공간
- **Metaspace**: GC시에 필요한 클래스와 메소드의 메타데이터 저장 공간

<br><br>

## 🔁 GC의 과정
### Step 1: 객체 생성
new 키워드로 객체가 생성되면 **Eden영역에 할당**됩니다.

<img width="545" alt="스크린샷 2025-06-29 오후 10 14 02" src="https://github.com/user-attachments/assets/e64f4902-e294-474c-854d-fe88edd0b076" />

<br><br>

### Step 2: Eden 영역이 가득차서 더이상 할당될 공간이 없다면, **Minor GC** 발생!

<img width="529" alt="스크린샷 2025-06-29 오후 10 14 16" src="https://github.com/user-attachments/assets/45aa784d-8b04-409b-884f-700734db55d8" />

<br><br>

**Mark&Sweep 진행**
1. Mark: GC Root 기준으로 unreaahable한 객체 마킹 <br><br>
   <img width="582" alt="스크린샷 2025-06-29 오후 10 14 30" src="https://github.com/user-attachments/assets/80e57a1a-4157-4104-b834-889102dda7d1" />

<br><br>

2. Survivor 이동: GC에서 살아남은 객체들은 Survivor 영역으로 이동 <br>
   <img width="538" alt="스크린샷 2025-06-29 오후 10 14 41" src="https://github.com/user-attachments/assets/49731c1b-36c8-4987-8bca-123f97a4bcc3" />

   ⚠️ Survivor 영역 규칙 <br>
   - Survivor 0에 객체가 들어있으면 Survivor 1은 무조건 비워줘야하며, 반대로 Survivor 1에 객체가 들어있으면 Survivor 0은 무조건 비워줘야한다.

<br><br>

3. Sweep: unreachable한 객체 삭제 <br><br>
   <img width="546" alt="스크린샷 2025-06-29 오후 10 14 56" src="https://github.com/user-attachments/assets/231ba853-f7cf-41f9-9ff7-ad5f31721af5" />

<br><br>

4. Aging: Survivor 영역에 있는 객체들의 age가 1증가 <br><br>
   <img width="528" alt="스크린샷 2025-06-29 오후 10 17 50" src="https://github.com/user-attachments/assets/370525de-f8a0-4f80-9adc-04819a9b6e6a" />

<br><br>

### Step 3: Old 영역이 꽉 차면 **Major GC** 발생

1. 다시 Eden 영역에 새로운 객체들이 할당되며, 할당될 자리가 없다면 `Minor GC`가 발생 <br><br>
   <img width="541" alt="스크린샷 2025-06-29 오후 10 18 03" src="https://github.com/user-attachments/assets/2def0bf4-ec39-48ad-9805-0b6d1010b078" />

<br>

2. Mark&Sweep 진행 <br> 
   ⚠️ Minor GC가 한 번 발생할 때마다, Survivor영역을 왔다갔다한다. <br><br>
   <img width="563" alt="스크린샷 2025-06-29 오후 10 18 18" src="https://github.com/user-attachments/assets/929fcb42-1372-4b4e-857d-68b1800b3131" />

<br>

3. 이와 같은 과정에 계속 반복

4. 객체의 age가 age threshold에 도달하게 되면, Old Generation으로 이동 <br><br>
   <img width="543" alt="스크린샷 2025-06-29 오후 10 18 36" src="https://github.com/user-attachments/assets/b5a514b4-f0ab-4192-9942-d75e1c136578" />

<br><br>

5. Old Gereration이 꽉차면, `Major GC`가 발생 <br><br>
   <img width="534" alt="스크린샷 2025-06-29 오후 10 18 47" src="https://github.com/user-attachments/assets/24a62d11-9a98-41a1-bc06-87db0e4e66ba" />

<br>
<br>

### 왜 Heap 영역을 Young Generation과 Old Generation으로 나눈 이유는?
대부분의 객체는 수명이 짧다! <br>
→ Young 영역에서 GC를 자주, 빠르게 수행 <br>
→ Old 영역은 GC 빈도는 낮지만 성능에 영향 큼

<br>

<img width="595" alt="스크린샷 2025-06-29 오후 10 18 57" src="https://github.com/user-attachments/assets/fdecf5f6-d0e9-4e70-a672-8e419b400521" />  <br>
x축: 객체 수명(바이트 단위) <br> 
y축: 해당 수명을 가진 객체의 총 바이트 수

<br> 
<br>
<br>

### Stop the world
GC를 실행하기 위해 JVM이 애플리케이션 실행을 멈추는 것을 말합니다. GC 알고리즘마다 STW 시간이 다르며, 짧을수록 성능에 유리합니다.

 <br>

## 🧭 GC의 종류
### Serial GC
- GC를 처리하는 쓰레드가 1개(싱글 쓰레드)
- 다른 GC에 비해 stop-the-world 시간이 길다
- 작은 Heap, 단일 코어 환경에 적합


<br>

### Parallel GC
- Java 8의 default GC 방식
- 여러 개의 스레드로 GC 실행
- 멀티코어 환경에서 어플리케이션 처리 속도를 향상시키기 위해 사용

<br>


### CMS GC(Concurrent Mark Sweep)
- Stop the World 최소화 목적
- compaction 미지원 → 단편화 발생
- G1 GC로 대체됨
<img width="589" alt="스크린샷 2025-06-29 오후 10 19 09" src="https://github.com/user-attachments/assets/ed4a16c7-d139-4322-86f4-f61518fc38d2" />

<br>

### G1 GC(Garbage First)
- Region 단위로 Heap 분할
- GC 효율 향상 + 짧은 STW
- Java9부터 default GC방식


<br><br>


> 🔗 참고 <br>
[JVM 이미지 - wikipedia JVM ](https://ko.wikipedia.org/wiki/%EC%9E%90%EB%B0%94_%EA%B0%80%EC%83%81_%EB%A8%B8%EC%8B%A0) <br>
[객체 수명 분포도 - oracle ](https://docs.oracle.com/en/java/javase/16/gctuning/garbage-collector-implementation.html#GUID-71D796B3-CBAB-4D80-B5C3-2620E45F6E5D) <br>
[우아한테크 - 조엘의 GC](https://youtu.be/FMUpVA0Vvjw?si=Bp-Z4tctoyK126y5)
  
  
  
  
