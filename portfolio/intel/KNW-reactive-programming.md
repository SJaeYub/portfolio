# 지식 정리 (Knowledge Note)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **주제** | 리액티브 프로그래밍 (Reactive Programming) |
| **분류** | 프레임워크 / 언어 / 비동기 프로그래밍 |
| **키워드** | Reactive Programming, Project Reactor, Mono, Flux, WebFlux, Backpressure, 비동기, 논블로킹, Kotlin Flow, Cold/Hot Publisher |
| **학습 계기** | 비동기·논블로킹 처리 방식과 리액티브 프로그래밍의 차이점 탐구 |
| **관련 업무 ID** | - |
| **최초 작성일** | 2026-04-27 |
| **최종 수정일** | 2026-04-27 |
| **숙련도** | 🌿 이해 |

---

## 1. 한 줄 요약 (TL;DR)

```
리액티브 프로그래밍은 단순히 논블로킹을 구현하는 게 아니라,
데이터를 스트림(흐름)으로 모델링하여 변화에 선언적으로 반응하는 패러다임이다.
논블로킹은 그 수단이고, Backpressure·스트림 합성·선언적 파이프라인이 리액티브의 본질이다.
```

---

## 2. 배경 지식 (Prerequisites)

- **필수 선행 지식**:
  - 스레드(Thread)와 스레드 풀 개념
  - 블로킹(Blocking) vs 논블로킹(Non-blocking)
  - 동기(Synchronous) vs 비동기(Asynchronous)
  - 옵저버 패턴(Observer Pattern)

- **관련 개념과의 관계**:
  ```
  [프로그래밍 패러다임]
    ├── 명령형(Imperative)
    │     ├── 동기·블로킹        ← 전통적인 방식
    │     └── 비동기·논블로킹    ← Coroutine, CompletableFuture
    │
    └── 리액티브(Reactive)      ← 데이터를 스트림으로 모델링
          ├── Project Reactor   (Mono / Flux)
          ├── RxJava            (Observable / Flowable)
          └── Kotlin Flow       (cold stream, coroutine 기반)

  [기반 스펙]
  Reactive Streams Specification
    └── Publisher / Subscriber / Subscription / Processor 인터페이스
          ├── Project Reactor 구현
          └── RxJava 구현
  ```

---

## 3. 개념 설명 (Concept)

### 3.1 정의 (What)

```
리액티브 프로그래밍이란 데이터나 이벤트의 변경이 발생했을 때
이에 자동으로 반응(react)하여 처리하는 프로그래밍 패러다임이다.

핵심 요소:
  ① 비동기·논블로킹: 스레드가 I/O를 기다리지 않음
  ② 데이터 스트림: 데이터를 이벤트의 연속적인 흐름으로 모델링
  ③ 선언적 파이프라인: filter, map, flatMap 등으로 흐름을 조합
  ④ Backpressure: 소비자가 처리 가능한 속도로 데이터 흐름을 제어
```

### 3.2 존재 이유 (Why)

```
전통적 블로킹 방식의 문제:
  요청 100개 → 스레드 100개 필요
  → 스레드 생성/유지 비용 증가 → 메모리·컨텍스트 스위칭 부담

리액티브 방식이 해결하는 것:
  스레드 1개로 수천 개 요청 처리 (I/O 대기 시간 = 다른 요청 처리)
  → API 게이트웨이, 채팅 서버, 실시간 스트리밍처럼
    I/O 집약적·고동시성 환경에서 효율적
```

### 3.3 동작 원리 (How it works)

**리액티브 처리 흐름 (Project Reactor)**

```
Step 1. Publisher(Flux/Mono) 생성 — 데이터 소스 정의
Step 2. 중간 연산자 체이닝 — filter, map, flatMap 등 파이프라인 구성
         (이 시점에는 아무것도 실행되지 않음 — Lazy 실행)
Step 3. subscribe() 호출 — 구독 시작, 이때 실행 시작
Step 4. 데이터가 도착할 때마다 파이프라인을 통과해 처리
Step 5. onComplete 또는 onError 신호로 스트림 종료
```

**Reactive Streams 4대 인터페이스**

```
Publisher<T>    → 데이터를 발행하는 쪽 (Flux, Mono)
Subscriber<T>   → 데이터를 소비하는 쪽 (subscribe 호출자)
Subscription    → Publisher-Subscriber 간 연결 (request(n)으로 데이터 요청량 조절)
Processor<T,R>  → Publisher + Subscriber 동시 역할 (중간 변환)
```

### 3.4 구성 요소 / 핵심 용어 (Key Terms)

| 용어 | 설명 | 비고 |
|------|------|------|
| Mono | 0 또는 1개의 데이터를 발행하는 Publisher | 단건 API 응답 |
| Flux | 0 ~ N개의 데이터를 발행하는 Publisher | 다건 조회, 스트리밍 |
| subscribe() | 구독을 시작하는 메서드. 이 호출 전까지 아무것도 실행 안 됨 | Lazy 실행의 핵심 |
| Backpressure | 소비자가 처리 가능한 속도로 생산자의 발행 속도를 제어하는 메커니즘 | 메모리 보호 |
| Cold Publisher | 구독할 때마다 새로운 데이터 흐름 시작 (Flux, Mono 기본값) | HTTP 요청처럼 구독마다 독립 |
| Hot Publisher | 구독 여부와 관계없이 데이터 발행. 늦은 구독자는 이전 데이터 놓침 | 주식 시세, 채팅 메시지 |
| Scheduler | 어느 스레드에서 연산을 실행할지 지정 (subscribeOn, publishOn) | 스레드 풀 전환 |
| WebFlux | Spring의 리액티브 웹 프레임워크. Netty 기반 논블로킹 서버 | Spring MVC 대체 |

---

## 4. 비교 및 구분 (Comparison)

### 4.1 동기 vs 비동기 vs 리액티브 비교

| 항목 | 동기(Blocking) | 비동기(Coroutine) | 리액티브(Reactor/WebFlux) |
|------|---------------|-----------------|--------------------------|
| 스레드 대기 | 응답까지 대기 | 코루틴 중단, 스레드 해방 | 구독 후 이벤트 반응 |
| 데이터 모델 | 단건 결과 반환 | 단건 결과 반환 | 스트림(흐름)으로 반환 |
| 코드 스타일 | 순차 명령형 | 순차처럼 읽히는 비동기 | 선언적 파이프라인 |
| Backpressure | ❌ 없음 | ❌ 없음 (List 전체 로드) | ✅ 내장 |
| 실시간 스트리밍 | ❌ | ❌ (Flow 별도 필요) | ✅ Flux로 자연스럽게 |
| 학습 난이도 | 낮음 | 중간 | 높음 |
| 적합한 상황 | 단순 CRUD | 대부분의 비동기 처리 | 스트리밍, 고동시성 I/O |

### 4.2 비동기(코루틴)와 리액티브의 핵심 차이

```
비동기(코루틴): "하나의 요청 → 하나의 응답을 기다리는 구조"
리액티브:      "하나의 구독 → 여러 데이터가 시간 차를 두고 흘러오는 스트림 구조"

[비동기 - 코루틴: 100명 조회]
요청 ──────────────────── 100명 전부 준비 완료 ──→ List<User> 반환 후 처리 시작
                              (전부 모일 때까지 대기)

[리액티브 - Flux: 100명 조회]
요청 → user1 도착 → 처리 → user2 도착 → 처리 → ... → user100
      (첫 번째 데이터부터 즉시 처리, 전체 완료를 기다리지 않음)
```

### 4.3 Kotlin에서의 라이브러리 선택

| 라이브러리 | 기반 | 스트림 타입 | 권장 상황 |
|-----------|------|-----------|---------|
| Kotlin Coroutines + Flow | 코루틴 | cold stream | 신규 Kotlin 프로젝트 공식 권장 |
| Project Reactor (Mono/Flux) | Reactive Streams | cold stream | Spring WebFlux 기반 |
| RxKotlin | RxJava 래핑 | hot/cold 모두 | 기존 RxJava 프로젝트 호환 필요 시 |

---

## 5. 실전 예시 (Examples)

### 5.1 동기 / 비동기 / 리액티브 코드 비교

**① 동기 (Blocking)**

```kotlin
// Spring MVC (RestTemplate)
@RestController
class UserController(val restTemplate: RestTemplate) {

    @GetMapping("/user/{id}")
    fun getUser(@PathVariable id: Long): User {
        // 🔴 스레드가 완전히 멈춤 — 응답 올 때까지 대기
        return restTemplate.getForObject(
            "https://api.example.com/users/$id",
            User::class.java
        )!!
    }
}
// 동시 요청 100개 → 스레드 100개 필요
```

**② 비동기 (Kotlin Coroutine)**

```kotlin
// Spring MVC + Coroutine (WebClient)
@RestController
class UserController(val webClient: WebClient) {

    @GetMapping("/user/{id}")
    suspend fun getUser(@PathVariable id: Long): User {
        // 🟡 코루틴 중단, 스레드는 해방됨
        return webClient.get()
            .uri("https://api.example.com/users/$id")
            .retrieve()
            .awaitBody<User>()
        // 응답 오면 코루틴 재개, 코드는 동기처럼 읽힘
    }
}
```

**③ 리액티브 (Project Reactor / WebFlux)**

```kotlin
// Spring WebFlux (Mono/Flux)
@RestController
class UserController(val webClient: WebClient) {

    // 단건 조회 → Mono
    @GetMapping("/user/{id}")
    fun getUser(@PathVariable id: Long): Mono<User> {
        return webClient.get()
            .uri("https://api.example.com/users/$id")
            .retrieve()
            .bodyToMono(User::class.java)
            .map { it.copy(name = it.name.uppercase()) }  // 도착 시 자동 변환
            .onErrorResume { Mono.just(User(id, "Unknown")) }  // 에러 자동 반응
    }

    // 다건 스트리밍 → Flux
    @GetMapping("/users")
    fun getAllUsers(): Flux<User> {
        return webClient.get()
            .uri("https://api.example.com/users")
            .retrieve()
            .bodyToFlux(User::class.java)
            .filter { it.age >= 20 }         // 오는 즉시 필터
            .map { it.copy(name = it.name.uppercase()) }
            .take(10)                          // 최대 10개
        // subscribe 없음 — WebFlux가 자동 구독
    }
}
```

---

### 5.2 Backpressure — 속도 조절

```kotlin
// DB는 초당 10,000건 생산, 처리 로직은 초당 100건만 가능
// 비동기(코루틴): List로 한꺼번에 받으면 메모리 폭발 가능
// 리액티브: Backpressure로 자동 조절

db.findAllUsersReactive()          // Flux<User>
    .onBackpressureBuffer(500)      // 최대 500개까지 버퍼, 초과 시 오류
    .subscribe { slowProcess(it) }
```

---

### 5.3 실시간 스트리밍 (비동기로 대체 불가)

```kotlin
// 주식 시세처럼 끝이 없는 데이터 스트림
// 비동기: "언제 끝나?" → suspend fun은 종료점이 있어야 함
// 리액티브: 흐름 자체를 구독

stockPricePublisher.toFlux()
    .filter { it.changePercent > 5.0 }
    .subscribe { sendAlert(it) }   // 올 때마다 자동 알림
```

---

### 5.4 Cold vs Hot Publisher

```kotlin
// Cold Publisher — 구독마다 새로운 스트림 (Flux 기본값)
val coldFlux = Flux.just("A", "B", "C")
coldFlux.subscribe { println("구독1: $it") }  // A, B, C
coldFlux.subscribe { println("구독2: $it") }  // A, B, C (독립 실행)

// Hot Publisher — 구독 시점 이후 데이터만 수신
val hotFlux = Flux.just("A", "B", "C").publish().autoConnect()
hotFlux.subscribe { println("구독1: $it") }  // A, B, C
// 잠시 후
hotFlux.subscribe { println("구독2: $it") }  // B, C (A는 이미 지남)
```

---

### 5.5 주의해야 할 패턴 (Anti-pattern)

```kotlin
// ❌ WebFlux에서 블로킹 코드 사용
@GetMapping("/bad")
fun bad(): Mono<String> {
    val result = blockingRepository.findAll()  // 블로킹 DB 호출
    return Mono.just(result.toString())
    // WebFlux의 이벤트 루프 스레드를 점유 → 전체 처리량 급감
}
```

```kotlin
// ✅ 리액티브 Repository 사용
@GetMapping("/good")
fun good(): Flux<User> {
    return reactiveRepository.findAll()  // R2DBC 등 논블로킹 드라이버 필요
}
```

> WebFlux를 사용하면 DB 드라이버도 논블로킹을 지원해야 효과가 있다.
> JPA는 블로킹 → R2DBC (관계형 DB 리액티브 드라이버) 로 교체 필요.

---

## 6. 트러블슈팅 경험 (Troubleshooting)

### 문제 1. subscribe() 없이 Mono/Flux가 실행되지 않음

- **증상**: map, filter 등 연산자를 붙였는데 아무것도 실행되지 않음.
- **원인**: Project Reactor는 Lazy 실행. subscribe()가 호출되어야 파이프라인 실행 시작.
- **해결**: subscribe() 또는 block()을 명시 호출. WebFlux 컨트롤러에서는 Mono/Flux를 반환하면 프레임워크가 자동 구독.
- **교훈**: Reactor는 "선언만 해두고, 구독할 때 실행한다." 파이프라인을 정의하는 것과 실행하는 것을 항상 구분해야 한다.

### 문제 2. WebFlux + JPA 사용 시 성능 개선이 없음

- **증상**: Spring WebFlux로 전환했는데 처리량이 기대만큼 늘지 않음.
- **원인**: JPA는 블로킹 I/O. WebFlux의 이벤트 루프 스레드를 JPA 호출이 점유해버려 리액티브의 장점이 사라짐.
- **해결**: R2DBC(리액티브 DB 드라이버)로 교체. 또는 Schedulers.boundedElastic()으로 블로킹 작업을 별도 스레드로 오프로드.
  ```kotlin
  Mono.fromCallable { blockingRepository.findById(id) }
      .subscribeOn(Schedulers.boundedElastic())  // 블로킹 전용 스레드 풀로 오프로드
  ```
- **교훈**: WebFlux를 쓰려면 I/O 레이어 전체가 논블로킹이어야 의미 있다.

### 문제 3. Backpressure 없이 메모리 폭발

- **증상**: 대량 데이터 스트리밍 시 OutOfMemoryError 발생.
- **원인**: 생산자는 초고속으로 발행하는데 소비자는 처리를 따라가지 못함. Backpressure 설정 없으면 무한 버퍼링.
- **해결**: `onBackpressureBuffer(N)`, `onBackpressureDrop()`, `onBackpressureLatest()` 중 비즈니스 요구에 맞는 전략 선택.
- **교훈**: 리액티브 스트림에서 생산자와 소비자의 속도 차이는 항상 Backpressure로 제어해야 한다.

---

## 7. 실전 명령어 / 치트시트 (Cheat Sheet)

```kotlin
// Mono 생성
Mono.just(value)
Mono.empty()
Mono.error(RuntimeException("에러"))
Mono.fromCallable { blockingCall() }

// Flux 생성
Flux.just("A", "B", "C")
Flux.fromIterable(list)
Flux.range(1, 10)  // 1~10

// 중간 연산자
.map { it.uppercase() }           // 1:1 변환 (동기)
.flatMap { fetchAsync(it) }        // 1:N 변환 (비동기, 순서 미보장)
.concatMap { fetchAsync(it) }      // 1:N 변환 (순서 보장, 느림)
.filter { it.length > 3 }         // 조건 필터
.take(10)                          // 최대 N개
.skip(5)                           // 앞 N개 스킵
.distinct()                        // 중복 제거
.timeout(Duration.ofSeconds(5))    // 타임아웃

// Backpressure
.onBackpressureBuffer(500)         // 버퍼 500개까지
.onBackpressureDrop()              // 초과분 버림
.onBackpressureLatest()            // 최신값만 유지

// 에러 처리
.onErrorReturn(defaultValue)       // 에러 시 기본값 반환
.onErrorResume { Mono.just(...) }  // 에러 시 대체 Publisher
.retry(3)                          // 최대 3회 재시도

// 스케줄러
.subscribeOn(Schedulers.boundedElastic())  // 구독 실행 스레드 지정
.publishOn(Schedulers.parallel())          // 이후 연산 스레드 전환

// 구독
.subscribe(
    { data -> println(data) },    // onNext
    { err -> println(err) },      // onError
    { println("완료") }           // onComplete
)
```

---

## 8. 깊게 이해하기 (Deep Dive)

### 8.1 리액티브 프로그래밍의 역사 (오류 수정)

> **⚠️ 오류 수정**: 원문 "2010년 에릭 마이어(Erik Meijer)가 마이크로소프트 .NET 생태계에서 처음 정의했습니다"는 부정확하다.

**정확한 역사:**

| 연도 | 사건 |
|------|------|
| 1997 | Conal Elliott & Paul Hudak — **FRP(Functional Reactive Programming)** 논문 발표. 리액티브 프로그래밍의 개념적 뿌리 |
| 2009~2010 | Erik Meijer (Microsoft) — **.NET용 Reactive Extensions(Rx)** 개발. Rx가 리액티브를 주류 생태계에 대중화 |
| 2012 | RxJava 공개 (Netflix) |
| 2013 | **Reactive Manifesto** 발표 — 분산 시스템 관점에서 리액티브 시스템 원칙 정립 (Responsive, Resilient, Elastic, Message-Driven) |
| 2015 | **Reactive Streams Specification** — JVM 리액티브 라이브러리 간 상호운용 표준 |
| 2017 | Spring 5 / Project Reactor 3 — Spring 생태계의 리액티브 공식 채택 |

에릭 마이어는 리액티브 프로그래밍을 "처음 정의"한 것이 아니라, Rx를 통해 이를 .NET 및 이후 JVM 생태계에 대중화시킨 인물로 이해하는 것이 정확하다.

---

### 8.2 Reactive Streams Specification (보강)

Project Reactor, RxJava는 모두 **Reactive Streams 표준**을 구현한다.

```
org.reactivestreams.*

Publisher<T>
  └── subscribe(Subscriber<T> s)

Subscriber<T>
  ├── onSubscribe(Subscription s)  ← 구독 시작, request(n) 호출 가능
  ├── onNext(T t)                  ← 데이터 도착
  ├── onError(Throwable t)         ← 에러 발생
  └── onComplete()                 ← 스트림 종료

Subscription
  ├── request(long n)              ← Backpressure: n개만 요청
  └── cancel()                     ← 구독 취소
```

표준화 덕분에 Reactor의 Flux를 RxJava의 Observable로, 또는 Kotlin Flow로 서로 변환이 가능하다.

---

### 8.3 subscribeOn vs publishOn 차이 (보강)

```kotlin
flux
    .map { heavyTransform(it) }    // 어떤 스레드?
    .subscribeOn(Schedulers.boundedElastic())
    // subscribeOn: "구독 자체를 어느 스레드에서 시작할지" 지정
    // → 전체 upstream(포함 위 map)이 해당 스레드에서 실행

flux
    .map { step1(it) }             // 기존 스레드
    .publishOn(Schedulers.parallel())
    .map { step2(it) }             // 이제부터 parallel 스레드
    // publishOn: "이 시점 이후의 downstream을 어느 스레드로 전환할지" 지정
```

---

### 8.4 Spring MVC vs Spring WebFlux 선택 기준 (보강)

```
Spring MVC 유지 권장:
  - 기존 팀의 동기 코드베이스
  - JPA 사용 (블로킹 드라이버)
  - 단순 CRUD 위주

Spring WebFlux 권장:
  - 고동시성 I/O (채팅, 실시간 피드)
  - SSE(Server-Sent Events), WebSocket 스트리밍
  - MSA 간 논블로킹 HTTP 체이닝 (API Gateway)
  - R2DBC + 완전 논블로킹 스택 구성 가능한 환경

WebFlux는 무조건 빠른 게 아니다.
CPU 집약적 작업(이미지 처리, 암호화 등)에서는 스레드 기반 MVC가 더 효율적일 수 있다.
```

---

### 8.5 Reactive Manifesto 4원칙 (보강)

```
분산 리액티브 시스템의 4원칙 (2013):

① Responsive (응답성)
   요청에 적시에 응답. 장애 감지·복구 포함

② Resilient (탄력성)
   장애 발생 시 격리, 복제, 위임으로 시스템 유지
   구성 요소 간 격리 → 장애가 전파되지 않음

③ Elastic (유연성)
   부하 변화에 따라 자원 동적 확장/축소

④ Message-Driven (메시지 기반)
   비동기 메시지 전달로 느슨한 결합
   → Backpressure, 위치 투명성 달성
```

---

## 9. 나만의 요약 (My Summary)

```
리액티브를 비동기·논블로킹과 동일하게 보는 오해가 가장 흔하다.

핵심 차이:
  비동기(코루틴): "기다리는 방식을 효율화한 것"
                  하나의 요청 → 하나의 응답
  리액티브:       "데이터 자체를 흐름으로 바라보는 패러다임"
                  하나의 구독 → 시간 차를 두고 흘러오는 N개의 데이터

리액티브가 필요한 결정적 순간:
  1. 실시간 스트리밍 (끝이 없는 데이터)
  2. Backpressure (생산 속도 > 소비 속도 제어)
  3. 복잡한 스트림 합성 (여러 Publisher를 zip, merge, concat)

WebFlux 쓸 때 함정:
  - JPA(블로킹) + WebFlux = 효과 없음 → R2DBC 필요
  - subscribe() 없으면 아무것도 실행 안 됨 (Lazy)
  - 학습 곡선 높음 → Kotlin Coroutines + Flow가 더 실용적인 경우 많음
```

**기억할 포인트 3가지:**
1. 에릭 마이어는 Rx를 만들어 리액티브를 **대중화**했지, 처음 정의하지 않았음 (FRP 뿌리 = 1997년)
2. Mono/Flux는 **Lazy**: subscribe() 호출 전까지 파이프라인이 실행되지 않음
3. WebFlux는 **전 레이어가 논블로킹**일 때 효과 있음 (JPA 사용 시 효과 없음)

**다음에 헷갈릴 것 같은 부분:**
- subscribeOn vs publishOn 실행 스레드 차이
- Cold Publisher vs Hot Publisher의 실제 동작 차이
- Backpressure 전략 선택 기준 (buffer vs drop vs latest)

---

## 10. 참고 자료 (References)

| 유형 | 제목 | 링크 | 메모 |
|------|------|------|------|
| 블로그 | 리액티브 프로그래밍 비동기·논블로킹 차이 | https://velog.io/@hodaessi/리액티브-프로그래밍2-비동기-논블로킹-리액티브-프로그래밍 | 논블로킹 vs 리액티브 구분 |
| 블로그 | Kotlin 리액티브 프로그래밍 | https://velog.io/@ekxk1234/kotlin-reactive-programing | Kotlin 중심 설명 |
| 블로그 | 동기·비동기·블로킹·논블로킹 이해 | https://f-lab.kr/insight/understanding-sync-async-blocking-nonblocking | 4가지 개념 정리 |
| 공식 문서 | Project Reactor 공식 | https://projectreactor.io/docs | Mono/Flux API |
| 질문 답변 | 코루틴이 Reactive Streams를 대신할 수 있나? | https://www.inflearn.com/community/questions/1093712 | Flow vs Reactor 비교 |
| 블로그 | 리액티브 프로그래밍 기초 | https://devsh.tistory.com/entry/리액티브-프로그래밍-기초-비동기-프로그래밍 | 기초 개념 |
| 공식 문서 | Reactive Manifesto | https://www.reactivemanifesto.org | 4원칙 원문 |

---

*이 문서는 학습이 깊어질수록 계속 업데이트합니다.*
