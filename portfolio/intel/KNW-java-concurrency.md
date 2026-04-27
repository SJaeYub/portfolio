# KNW-java-concurrency

> 학습한 내용을 나중에 언제 봐도 이해할 수 있도록 개념·원리·실제 사례를 함께 기록합니다.

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **주제** | Java 동시성(Concurrency) 제어 원리 — synchronized, volatile, Atomic, ConcurrentHashMap |
| **분류** | 언어 / 플랫폼 |
| **키워드** | Concurrency, Parallelism, Race Condition, synchronized, volatile, AtomicInteger, CAS, ABA, Memory Barrier, happens-before, ConcurrentHashMap, Deadlock, ReentrantLock |
| **학습 계기** | 자기계발 |
| **관련 업무 ID** | — |
| **최초 작성일** | 2026-04-27 |
| **최종 수정일** | 2026-04-27 |
| **숙련도** | 🌿 이해 |

---

## 1. 한 줄 요약 (TL;DR)

```
멀티스레드 환경에서는 공유 힙(Heap) 메모리에 대한 Race Condition과
CPU 캐시에 의한 가시성(Visibility) 문제가 발생한다.
Java는 이를 해결하기 위해 synchronized(원자성+가시성), volatile(가시성),
Atomic 변수(CAS 기반 원자성+가시성) 세 가지 메커니즘을 제공한다.
```

---

## 2. 배경 지식 (Prerequisites)

- **필수 선행 지식**:
  - 프로세스(Process)와 스레드(Thread)의 메모리 구조 차이
  - CPU 코어와 캐시 메모리(L1/L2/L3) 계층 구조
  - JVM 힙(Heap), 스택(Stack) 메모리 영역

- **관련 개념과의 위치**:
  ```
  [Java 동시성 도구]
    ├── 저수준 키워드
    │     ├── synchronized  → 원자성 + 가시성 (모니터 락)
    │     └── volatile      → 가시성 + 명령어 재정렬 방지
    ├── java.util.concurrent.atomic
    │     ├── AtomicInteger, AtomicLong, AtomicBoolean, AtomicReference
    │     └── AtomicStampedReference (ABA 문제 해결)
    ├── java.util.concurrent
    │     ├── ConcurrentHashMap (동시성 안전 Map)
    │     ├── ReentrantLock, ReadWriteLock (고수준 Lock)
    │     ├── ExecutorService, ThreadPoolExecutor
    │     └── CompletableFuture
    └── 동시성 문제
          ├── Race Condition (경쟁 조건)
          ├── 가시성 문제 (CPU 캐시)
          └── Deadlock (교착 상태)
  ```

---

## 3. 개념 설명 (Concept)

### 3.1 정의 (What)

```
동시성(Concurrency):
  하나의 CPU 코어에서 여러 작업이 빠르게 번갈아 실행되어 마치 동시에
  처리되는 것처럼 보이게 하는 기법. 컨텍스트 스위칭(Context Switching)을
  통해 구현되며, 애플리케이션의 응답성과 처리량을 향상시킨다.

병렬성(Parallelism):
  여러 CPU 코어에서 여러 작업이 물리적으로 동시에 실행되는 것.
  작업을 분할해 처리 속도를 높이는 것이 목적이다.
```

### 3.2 스레드 메모리 구조 (Why Race Condition이 생기는가)

```
[ 프로세스 ]
  ├── Code 영역  ← 모든 스레드 공유 (실행 코드)
  ├── Data 영역  ← 모든 스레드 공유 (static 변수)
  ├── Heap 영역  ← 모든 스레드 공유 ← ⚠️ Race Condition 발생 지점!
  ├── Stack (스레드1 전용) ← 지역 변수, 메서드 호출 스택
  ├── Stack (스레드2 전용)
  └── Stack (스레드3 전용)

힙에 있는 공유 변수를 여러 스레드가 동시에 읽고 쓸 때 데이터 충돌 발생.
```

### 3.3 주요 동시성 문제 (How it breaks)

```
1. 경쟁 조건 (Race Condition)
   두 스레드가 같은 자원에 동시 접근할 때 실행 순서에 따라 결과가 달라짐
   예: 좋아요 +2가 되어야 할 상황에서 +1만 반영됨

2. 가시성 문제 (Visibility Problem)
   각 스레드가 CPU 코어 전용 캐시에 캐싱된 값을 참조하여
   다른 스레드가 쓴 최신 값을 읽지 못하는 문제

3. 명령어 재정렬 (Instruction Reordering)
   JVM과 CPU는 성능 최적화를 위해 코드 실행 순서를 재배치할 수 있음
   단일 스레드에서는 결과가 동일하지만, 멀티스레드에서는 의도치 않은
   동작을 유발할 수 있음
```

### 3.4 동시성 vs 병렬성 비교

| 구분 | 동시성 (Concurrency) | 병렬성 (Parallelism) |
|------|--------------------|--------------------|
| **동작 방식** | 단일 코어에서 작업을 빠르게 전환 | 다중 코어에서 작업을 실제로 동시 실행 |
| **목적** | 작업 순서 조정으로 효율 향상 | 작업 분할로 처리 속도 향상 |
| **비유** | 한 요리사가 여러 요리를 번갈아 | 여러 요리사가 각자 요리를 동시에 |
| **코어 수** | 싱글 코어에서도 가능 | 멀티 코어 필수 |
| **메모리 공유** | 스레드끼리 공유 | 스레드끼리 공유 (동일) |
| **Race Condition** | 발생 가능 | 더 심하게 발생 (진짜 동시 실행이므로) |
| **개념 성격** | 논리적 개념 | 물리적 개념 |

### 3.5 구성 요소 / 핵심 용어 (Key Terms)

| 용어 | 설명 |
|------|------|
| **임계 영역 (Critical Section)** | 공유 자원에 접근하는 코드 구간 |
| **원자성 (Atomicity)** | 연산이 더 이상 쪼갤 수 없는 단일 단위로 처리됨 |
| **가시성 (Visibility)** | 한 스레드가 변경한 값이 다른 스레드에 즉시 보임 |
| **모니터 락 (Monitor Lock)** | 자바의 모든 객체가 보유한 내장 잠금 장치 |
| **happens-before** | 한 연산의 결과가 다른 연산에게 반드시 보임을 JMM이 보장하는 관계 |

---

## 4. 비교 및 구분 (Comparison)

### 4.1 synchronized vs volatile vs Atomic 비교

| 항목 | synchronized | volatile | Atomic (CAS) |
|------|-------------|----------|--------------|
| **가시성 보장** | ✅ | ✅ | ✅ |
| **원자성 보장** | ✅ | ❌ | ✅ (단순 연산) |
| **명령어 재정렬 방지** | ✅ | ✅ (Memory Barrier) | ✅ |
| **Lock 사용** | ✅ Blocking | ❌ Lock-free | ❌ Lock-free (재시도) |
| **적용 범위** | 메서드/블록 | 변수 단위 | 변수 단위 |
| **성능** | 경합 시 저하 | 빠름 (단순 읽기) | 대체로 빠름 |
| **적합한 상황** | 복잡한 임계 영역 | 읽기 위주·단순 플래그 | 단순 숫자·값 연산 |

### 4.2 ConcurrentHashMap vs HashMap vs synchronized Map

| 구분 | HashMap | Collections.synchronizedMap | ConcurrentHashMap |
|------|---------|-----------------------------|--------------------|
| **스레드 안전** | ❌ | ✅ | ✅ |
| **Lock 방식** | 없음 | 전체 Map에 통째로 Lock | 버킷 첫 노드 synchronized + CAS |
| **읽기 성능** | 빠름 | 느림 (Lock 경합) | 빠름 (Lock 없이 volatile로 처리) |
| **쓰기 성능** | (불안전) | 느림 | 빠름 (부분 Lock) |
| **null 허용** | key/value 모두 가능 | key/value 모두 가능 | ❌ key/value 모두 null 불허 |

---

## 5. 실전 예시 (Examples)

### 5.1 synchronized 4가지 사용 방법

```java
// 1. 인스턴스 메서드 동기화 — this 객체에 락
public synchronized void increment() {
    count++;  // 한 번에 하나의 스레드만 실행
}

// 2. static 메서드 동기화 — 클래스 단위 락 (인스턴스 달라도 공유)
public static synchronized void staticMethod() {
    count++;
}

// 3. synchronized 블록 (this) — 필요한 부분만 락, 성능 최적화
public void doWork() {
    prepare();  // 동기화 불필요한 코드, 동시 실행 가능
    synchronized (this) {
        count++;  // 꼭 필요한 부분만 락
    }
}

// 4. 별도 객체 락 — 독립적인 임계 영역 2개
private final Object lockA = new Object();
private final Object lockB = new Object();

public void methodA() { synchronized (lockA) { ... } }  // lockA만 잠김
public void methodB() { synchronized (lockB) { ... } }  // A와 동시 실행 가능!
```

**재진입성 (Reentrancy):** 이미 락을 보유한 스레드가 같은 락의 다른 synchronized 메서드를 호출해도 데드락 없이 진입 가능.

---

### 5.2 Atomic 변수 사용 예시

```java
AtomicInteger count = new AtomicInteger(0);

count.get();                   // 현재 값 반환: 0
count.incrementAndGet();       // 1 증가 후 반환: 1
count.getAndIncrement();       // 반환 후 1 증가 (반환값: 1, 이후 내부값: 2)
count.addAndGet(5);            // 5 더한 후 반환

// CAS 직접 사용: 현재 값이 0이면 1로 교체 → true, 아니면 false
count.compareAndSet(0, 1);
```

---

### 5.3 volatile + synchronized Double-Checked Locking (싱글톤)

```java
// volatile 없이는 명령어 재정렬로 인해 불완전한 객체가 참조될 수 있음
private static volatile Singleton instance;

public static Singleton getInstance() {
    if (instance == null) {                  // 1차 체크 (락 없이 빠르게)
        synchronized (Singleton.class) {
            if (instance == null) {          // 2차 체크 (락 안에서 안전하게)
                instance = new Singleton();
            }
        }
    }
    return instance;
}
```

> **volatile이 반드시 필요한 이유**: `new Singleton()`은 내부적으로 ① 메모리 할당 → ② 생성자 실행 → ③ instance에 참조 대입의 3단계인데, JVM이 ①③②로 재배치할 수 있다. volatile의 Memory Barrier가 이 재배치를 방지해 다른 스레드가 불완전한 객체를 참조하는 것을 막는다.

---

### 5.4 ConcurrentHashMap + AtomicLong 조합

```java
// ✅ 올바른 조합
private final ConcurrentHashMap<Long, User> store = new ConcurrentHashMap<>();
private final AtomicLong idGenerator = new AtomicLong(0);

public void addUser(User user) {
    long id = idGenerator.getAndIncrement();  // ID 생성: AtomicLong이 담당
    store.put(id, user);                      // 저장: ConcurrentHashMap이 담당
}
// AtomicLong → ID 중복 방지 / ConcurrentHashMap → Map 내부 구조 보호
```

---

### 5.5 주의해야 할 패턴 (Anti-pattern)

```java
// ❌ volatile은 가시성만 보장, 원자성은 없음
private volatile int count = 0;
count++;  // 읽기→더하기→쓰기 3단계 → Race Condition 여전히 발생!

// ✅ 단순 카운터는 AtomicInteger
private AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();  // 원자적으로 안전


// ❌ HashMap 멀티스레드 사용 — 데이터 유실 발생
Map<String, String> map = new HashMap<>();
// 두 스레드가 동시에 같은 버킷에 put() → 한쪽 데이터 유실

// ✅ 멀티스레드 환경에서는 ConcurrentHashMap
Map<String, String> map = new ConcurrentHashMap<>();
```

---

## 6. 트러블슈팅 경험 (Troubleshooting)

### 문제 1. Deadlock (교착 상태)

- **증상**: 서버가 특정 요청에서 무한 대기, CPU는 거의 0%, 스레드 덤프에서 BLOCKED 상태 스레드 발견
- **원인**: 두 스레드가 서로가 보유한 락을 기다리는 순환 대기
  ```
  스레드A: lockA 보유, lockB 대기
  스레드B: lockB 보유, lockA 대기
  → 서로 영원히 기다림
  ```
- **Deadlock 발생 4가지 조건 (모두 충족 시 발생)**:
  1. 상호 배제: 자원은 한 번에 하나의 스레드만 사용 가능
  2. 점유 대기: 자원을 가진 채 다른 자원을 기다림
  3. 비선점: 스레드가 보유한 자원을 강제로 빼앗을 수 없음
  4. 순환 대기: 스레드들이 서로의 자원을 순환적으로 기다림
- **해결/예방**:
  ```java
  // ✅ 예방: 항상 같은 순서로 락 획득 (순환 대기 조건 제거)
  synchronized (lockA) {      // 항상 A → B 순서
      synchronized (lockB) {
          // ...
      }
  }

  // ✅ ReentrantLock의 tryLock() 타임아웃으로 회피
  if (lockA.tryLock(1, TimeUnit.SECONDS)) {
      try {
          if (lockB.tryLock(1, TimeUnit.SECONDS)) {
              try { /* 작업 */ } finally { lockB.unlock(); }
          }
      } finally { lockA.unlock(); }
  }
  ```
- **진단**: `jstack <PID>` 또는 스레드 덤프로 BLOCKED 스레드와 waiting to lock 정보 확인

### 문제 2. HashMap 멀티스레드 사용으로 인한 데이터 유실

- **증상**: 동시 요청 처리 후 Map의 데이터가 누락됨
- **원인**: 두 스레드가 동시에 같은 버킷에 `put()` → 한쪽 데이터 덮어씌워짐
- **해결**: `ConcurrentHashMap` 으로 교체
- **교훈**: Java 7 이하에서는 Rehashing 시 사이클이 생겨 무한 루프도 발생했으나, Java 8 이후에는 해당 형태의 무한 루프는 발생하지 않는다. 그러나 **데이터 유실은 Java 8 이후에도 동일하게 발생**한다.

---

## 7. 실전 명령어 / 치트시트 (Cheat Sheet)

```java
// ──── synchronized ────
public synchronized void method() { ... }           // 인스턴스 락
public static synchronized void method() { ... }    // 클래스 락
synchronized (lockObject) { ... }                   // 블록 락

// ──── volatile ────
private volatile boolean flag = false;              // 단순 플래그 (쓰기 1개, 읽기 N개)
private static volatile Singleton instance;         // Double-Checked Locking용

// ──── Atomic ────
AtomicInteger n = new AtomicInteger(0);
n.incrementAndGet();                                // ++n (원자적)
n.getAndIncrement();                                // n++ (원자적)
n.compareAndSet(expected, newVal);                  // CAS 직접 사용
AtomicStampedReference<V> ref = new AtomicStampedReference<>(val, 0); // ABA 방지

// ──── ConcurrentHashMap ────
ConcurrentHashMap<K, V> map = new ConcurrentHashMap<>();
map.putIfAbsent(key, value);                        // 원자적 put
map.computeIfAbsent(key, k -> new ArrayList<>());  // 원자적 초기화

// ──── ReentrantLock ────
ReentrantLock lock = new ReentrantLock();
lock.lock(); try { ... } finally { lock.unlock(); } // 기본 사용
lock.tryLock(1, TimeUnit.SECONDS);                  // 타임아웃 락 시도
new ReentrantLock(true);                            // 공정 모드 (기다린 순서대로)

// ──── 스레드 덤프 (Deadlock 진단) ────
jstack <PID>
```

---

## 8. 깊게 이해하기 (Deep Dive)

### 8.1 CAS (Compare-And-Swap) 동작 원리

```
CAS는 세 가지 인자를 사용:
  M (Memory)         : 수정할 변수의 메모리 주소
  A (Expected Value) : 내가 방금 읽어온 예상값
  B (New Value)      : 저장하고 싶은 새 값

[ 스레드가 값을 읽는다 → 그 값을 "예상값 A"로 기억 → 변경 직전에 A와 메모리 M 비교 ]
  ├── 같으면 (아무도 안 건드림) → B로 교체 성공! ✅
  └── 다르면 (다른 스레드가 바꿈) → 포기하고 처음부터 재시도 🔄

내부 구현:
  public class AtomicInteger {
      private volatile int value;  // volatile → 항상 메인 메모리에서 읽음
      public final boolean compareAndSet(int expectedValue, int newValue) {
          return U.compareAndSetInt(this, VALUE, expectedValue, newValue);
          // ↑ Unsafe 클래스 → CPU의 하드웨어 명령(LOCK CMPXCHG)으로 처리
      }
  }
```

#### ABA 문제와 해결책

```
시나리오:
  스레드1: 값 0 읽음 (예상값 = 0)
  스레드2: 0 → 1 변경
  스레드2: 1 → 0 으로 되돌림 ← 값이 원래대로!
  스레드1: CAS(0, 새값) → 성공 ✅ (하지만 중간에 값이 바뀌었었음!)

문제: 값이 A → B → A로 변했는데 스레드1은 "안 바뀐 것"으로 인식

해결: AtomicStampedReference (버전 번호 = stamp 추가)
  AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(0, 0);
  int[] stampHolder = new int[1];
  Integer val = ref.get(stampHolder);         // 현재 값 + 현재 stamp 읽기
  ref.compareAndSet(val, newVal, stampHolder[0], stampHolder[0] + 1);
  // stamp가 다르면 CAS 실패 → ABA 문제 방지
```

### 8.2 volatile의 실제 메커니즘 — Memory Barrier

```
volatile은 단순히 "캐시를 우회"하는 것이 아니라
JVM이 해당 변수 접근 지점에 Memory Barrier(메모리 펜스)를 삽입한다.

Memory Barrier의 효과:
  1. 가시성 보장: 배리어 이전의 모든 쓰기가 메인 메모리에 즉시 반영
  2. 명령어 재정렬 방지: 배리어를 기준으로 앞/뒤 코드가 재배치되지 않음
  3. happens-before 관계 성립: volatile 변수 쓰기 → 읽기 사이에 순서 보장

Double-Checked Locking에서 volatile이 필요한 이유:
  new Singleton()의 실제 단계:
    ① 메모리 할당
    ② 생성자 실행 (객체 초기화)
    ③ instance 변수에 참조 대입

  JVM은 ① → ③ → ② 순으로 재배치 가능!
  → volatile 없으면: 다른 스레드가 ③ 이후 instance를 읽었을 때
    아직 ②가 완료되지 않은 불완전한 객체를 참조할 수 있음
  → volatile 있으면: Memory Barrier로 재배치 차단

volatile 성능 트레이드오프:
  일반 변수  → CPU 캐시에서 읽음 (빠름)
  volatile   → 메인 메모리에서 읽음 (느림)
  → 정말 가시성이 필요한 변수에만 선택적으로 사용
```

### 8.3 synchronized의 JVM 락 최적화 단계

```
JVM은 모든 synchronized가 커널 모드를 거치지 않도록 단계적으로 최적화한다.
경합 수준에 따라 다음 순서로 전환:

1. Biased Lock (편향 잠금)
   - 경합이 없는 경우: 최초 획득한 스레드 ID를 객체 헤더에 기록
   - 동일 스레드의 재진입 시 CAS 없이 헤더 확인만으로 처리 (가장 빠름)

2. Lightweight Lock (박형 잠금)
   - 두 번째 스레드가 진입 시도 시: 스택에 Lock Record를 생성하고 CAS로 처리
   - 커널 모드 전환 없음 (빠름)

3. Spin Lock (스핀 잠금)
   - 락을 얻지 못한 스레드가 루프를 돌며 잠깐 재시도
   - 짧은 대기 예상 시 컨텍스트 스위칭 비용 절약

4. Heavy Lock (팽창, 중량 잠금)
   - 경합이 심한 경우: OS의 Mutex를 사용, 커널 모드 전환 발생
   - BLOCKED 상태로 대기 (가장 느림)

→ 문서에서 "락 획득/해제 시 항상 커널 모드 전환"이라고 하는 것은 Heavy Lock의
   경우에만 해당하며, 경합이 없는 일반적인 경우에는 훨씬 가볍게 처리된다.
```

### 8.4 ConcurrentHashMap 내부 구조 (Java 8+)

```
Java 7 이하: Segment 배열 → 각 Segment별 ReentrantLock
Java 8+:    Segment 제거 → 버킷 배열 직접 사용 + CAS/synchronized 혼합

Java 8+ ConcurrentHashMap 쓰기(put) 동작:
  1. 빈 버킷에 삽입: CAS 연산으로 처리 (Lock 없음, 매우 빠름)
  2. 기존 버킷에 삽입: 해당 버킷의 첫 노드에만 synchronized
  3. 버킷 내 노드 수 >= 8: 연결 리스트 → TreeNode(Red-Black Tree)로 전환

읽기(get): volatile 변수로 항상 최신 값 직접 읽음 (Lock 없음)

결과: 읽기는 Lock-free, 쓰기는 버킷 단위 최소 Lock → 높은 동시성 처리량

Java 7과의 차이 요약:
  Java 7: Segment(16개 기본) 단위 ReentrantLock
  Java 8: Segment 삭제, 버킷 첫 노드 synchronized + CAS로 대체
          → 더 세밀한 락, 더 높은 동시성
```

### 8.5 HashMap의 버전별 멀티스레드 위험성

```
⚠️ Java 7 이하:
  - 데이터 유실: 두 스레드가 동시에 같은 버킷에 put() → 한쪽 데이터 덮어씌워짐
  - 무한 루프: Rehashing 시 연결 리스트 순서 역전 → 사이클(A→B→A) 생성
              → get() 시 사이클을 순회하다 CPU 100%, 서버 무응답

Java 8+:
  - 데이터 유실: 여전히 발생 (ConcurrentModificationException 포함)
  - 무한 루프: Java 7 형태는 발생하지 않음 (tail insertion으로 구현 변경)
              → 단, 비정상적인 내부 상태가 될 가능성은 여전히 존재

결론: Java 버전과 무관하게 멀티스레드 환경에서는 반드시 ConcurrentHashMap 사용
```

### 8.6 ReentrantLock vs synchronized

```
synchronized 한계:
  - 타임아웃 없음 (무한 대기 가능)
  - 락 획득 시도 여부 확인 불가
  - 공정성(Fairness) 제어 불가

ReentrantLock 장점:
  ReentrantLock lock = new ReentrantLock(true);  // 공정 모드: 기다린 순서대로

  lock.lock();           // 일반 락 (synchronized와 동일)
  lock.tryLock();        // 즉시 시도, 실패 시 false 반환 (Deadlock 회피)
  lock.tryLock(1, TimeUnit.SECONDS);  // 타임아웃 지정
  lock.lockInterruptibly();           // 인터럽트 가능한 락

  // 반드시 finally에서 unlock
  lock.lock();
  try { ... } finally { lock.unlock(); }

ReadWriteLock 활용 (읽기 많고 쓰기 드문 경우):
  ReadWriteLock rwLock = new ReentrantReadWriteLock();
  rwLock.readLock().lock();   // 읽기 락: 여러 스레드 동시 허용
  rwLock.writeLock().lock();  // 쓰기 락: 단독 접근만 허용
```

---

## 9. 나만의 요약 (My Summary)

```
Java 동시성 = "공유 힙에 여러 스레드가 접근하면서 생기는 문제들과 그 해결책"

문제 3가지:
  1. Race Condition   → 여러 스레드가 공유 변수를 동시에 수정
  2. 가시성 문제      → CPU 캐시로 인해 최신 값이 안 보임
  3. 명령어 재정렬    → JVM/CPU가 코드 실행 순서를 바꿔버림

해결책 선택 기준:
  단순 플래그 (1개 스레드 쓰기, N개 읽기)  → volatile
  단순 숫자 카운터                         → AtomicInteger/Long
  복잡한 임계 영역 (여러 변수, 여러 연산)   → synchronized
  타임아웃/공정성 필요                     → ReentrantLock
  공유 Map                                → ConcurrentHashMap

volatile의 핵심: 캐시 우회 + Memory Barrier로 재정렬 방지
CAS의 핵심: 예상값 비교 후 교체 (실패 시 재시도, Lock 없음)
ABA 해결: AtomicStampedReference로 버전 번호 추가
```

**기억할 포인트 3가지:**
1. **volatile은 가시성만**, synchronized/Atomic은 **원자성도** 보장. `i++`에 volatile만 쓰면 위험
2. **HashMap은 멀티스레드에서 절대 사용 금지** → ConcurrentHashMap (Java 8 이후 세그먼트 방식 아님, 버킷 CAS/synchronized 방식)
3. **Deadlock 예방**: 항상 같은 순서로 락 획득 / ReentrantLock tryLock() 타임아웃 활용

**다음에 헷갈릴 것 같은 부분:**
- volatile이 메모리 배리어라는 점, 단순한 "캐시 우회" 이상의 역할임
- ConcurrentHashMap의 Java 7 (Segment) vs Java 8+ (CAS+synchronized) 구조 차이
- ABA 문제: 값이 같아도 중간에 바뀌었을 수 있다는 것 (AtomicStampedReference로 해결)
- synchronized의 JVM 최적화 (Biased → Lightweight → Heavy) 순서

---

## 10. 참고 자료 (References)

| 유형 | 제목 | 링크 | 메모 |
|------|------|------|------|
| 블로그 | Java 동시성 이해 | https://f-lab.kr/insight/understanding-java-concurrency | |
| 블로그 | Java volatile과 CPU 캐시 | https://f-lab.kr/insight/understanding-volatile-and-cpu-cache-20250508 | Memory Barrier 관련 |
| 블로그 | Java synchronized 이해 | https://f-lab.kr/insight/understanding-java-synchronization | |
| 블로그 | Lock-Free 알고리즘 (CAS, Volatile, Atomic) | https://dalichoi.tistory.com/entry/Lock-Free-알고리즘-살펴보기CAS-Volatile-Java-Atomic-Variables | |
| 블로그 | HashMap 멀티스레드 문제 | https://aaronryu.github.io/2021/01/31/problems-when-using-parallel-stream-with-hash-map/ | |
| 블로그 | synchronized 상세 | https://mangkyu.tistory.com/458 | JVM 락 최적화 설명 포함 |
| 블로그 | ABA 문제 | https://f-lab.kr/insight/java-concurrency-cas-algorithm-20250209 | AtomicStampedReference |
| 블로그 | volatile과 synchronized 차이 | https://f-lab.kr/insight/volatile-and-synchronized-in-java-20250620 | |

---

*이 문서는 학습이 깊어질수록 계속 업데이트합니다.*
