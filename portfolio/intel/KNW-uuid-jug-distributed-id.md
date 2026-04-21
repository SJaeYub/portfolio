# 지식 정리 (Knowledge Note)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **주제** | UUID / JUG 라이브러리 / 분산 ID 전략 |
| **분류** | 언어 / 프레임워크 / 아키텍처 |
| **키워드** | UUID, UUIDGen, JUG, java-uuid-generator, B+Tree, 분산 ID, Snowflake, ULID, Trace ID, Span ID |
| **학습 계기** | 업무 중 필요 — UUIDGen.getpid() 메서드의 역할 파악 및 UUID 버전별 성능 차이 분석 |
| **관련 업무 ID** | - |
| **최초 작성일** | 2026-04-20 |
| **최종 수정일** | 2026-04-20 |
| **숙련도** | 🌿 이해 |

---

## 1. 한 줄 요약 (TL;DR)

```
UUID v4(랜덤)는 DB B+Tree 인덱스에서 지속적인 페이지 분할을 유발한다.
JUG 라이브러리의 UUIDGen.getpid()는 UUID v1 생성 시 동일 호스트 내 JVM 간 고유성을 보장하기 위해
PID를 노드 ID 구성에 활용하는 내부 유틸리티 메서드이며,
분산 환경에서 DB PK용 ID는 시간 기반 정렬 ID(UUID v7, Snowflake, ULID)를 사용하는 것이 Best Practice다.
```

---

## 2. 배경 지식 (Prerequisites)

- **필수 선행 지식**:
  - Java Process / JVM 기본 개념
  - UUID 표준 개념 (RFC 4122)
  - DB 인덱스의 기본 개념 (조회 vs 삽입 성능 트레이드오프)

- **관련 개념과의 관계**:
  ```
  [JUG 라이브러리: com.fasterxml.uuid]
    └── [UUIDGen (유틸리티 클래스)]
          ├── [getpid()]              ← 현재 JVM PID 조회 (내부 유틸리티)
          └── [Generators 팩토리]
                ├── timeBasedGenerator()    → UUID v1 (타임스탬프 + 노드ID)
                ├── randomBasedGenerator()  → UUID v4 (완전 랜덤)
                └── nameBasedGenerator()    → UUID v3/v5 (이름 기반)

  [UUID 버전별 정렬 가능성]
    v4 (랜덤)   → ❌ 정렬 불가 → B+Tree 페이지 분할 폭발
    v1 (시간)   → ✅ 정렬 가능 → JUG로 생성
    v7 (시간)   → ✅ 정렬 가능 → 최신 표준 권장
    Snowflake   → ✅ 정렬 가능 → 고성능 분산 환경
    ULID        → ✅ 정렬 가능 → URL-safe, 사전순
  ```

---

## 3. 개념 설명 (Concept)

### 3.1 정의 (What)

```
UUID(Universally Unique Identifier)는 128비트 크기의 범용 고유 식별자로,
32자리 16진수를 8-4-4-4-12 패턴(하이픈 포함 36자)으로 표현한다.
이론적으로 2^128(약 340조 개) 이상의 고유값을 생성할 수 있어 충돌 가능성이 사실상 없다.

예시: 550e8400-e29b-41d4-a716-446655440000

JUG(Java UUID Generator)는 FasterXML이 관리하는 오픈소스 라이브러리로,
표준 java.util.UUID(v4만 지원)보다 다양한 버전(v1, v3, v4, v5)을 생성할 수 있다.

UUIDGen.getpid()는 JUG 내부 유틸리티 메서드로,
현재 JVM 프로세스의 PID를 조회하여 UUID v1 생성 시 노드 ID 구성에 활용한다.
```

### 3.2 존재 이유 (Why)

```
[표준 Java의 한계]
java.util.UUID.randomUUID()는 v4(완전 랜덤)만 제공한다.
랜덤 UUID는 DB PK로 사용 시 B+Tree 인덱스에서 지속적인 페이지 분할을 유발해
대규모 삽입 성능이 크게 저하된다.

[JUG가 필요한 이유]
v1(시간 기반) UUID는 타임스탬프를 포함해 시간 순 정렬이 가능하다.
JPA 엔티티의 PK를 UUID로 사용할 때 DB 인덱스 성능을 유지하려면 v1이 필요하고,
이를 Java에서 쉽게 생성하려면 JUG 같은 라이브러리가 필요하다.

[UUIDGen.getpid()가 필요한 이유]
UUID v1은 "타임스탬프 + 클록 시퀀스 + 노드 ID"로 구성된다.
노드 ID는 원래 MAC 주소를 사용하지만, 동일 호스트에서 여러 JVM이 실행될 경우
MAC 주소만으로는 구분이 안 된다.
이때 PID를 노드 ID에 혼합해 같은 머신의 JVM 간 UUID 충돌을 방지한다.
```

### 3.3 동작 원리 (How it works)

#### UUID v1 생성 흐름 (JUG)

```
① Generators.timeBasedGenerator().generate() 호출
        ↓
② 현재 타임스탬프(100ns 단위, 60비트) 획득
        ↓
③ 클록 시퀀스(14비트) — 같은 타임스탬프 내 순서 보장
        ↓
④ 노드 ID(48비트) 구성
   ├── MAC 주소 기반 (기본)
   └── PID 혼합 — UUIDGen.getpid()로 현재 JVM PID 조회
        ↓
⑤ 128비트 UUID 조합 후 반환
```

#### Java 버전별 getpid() 동작 방식

```
Java 8 이하:
  Reflection으로 UNIXProcess.pid 또는 ProcessImpl.handle 필드에 직접 접근
  OS(Unix/Windows)에 따라 동작 방식이 다름
  접근 실패 시 -1 반환 (폴백 로직 포함)

Java 9 이상:
  ProcessHandle.current().pid() 또는 Process.getPid() 공식 API 사용
  플랫폼 독립적, Reflection 불필요
```

#### B+Tree 인덱스와 UUID 삽입 패턴

```
[B+Tree 리프 노드 구조]
  데이터가 항상 값(UUID) 기준으로 정렬된 순서로 저장됨
  삽입 시 정렬된 위치를 찾아 끼워 넣음
  노드가 꽉 차면 페이지 분할(Page Split) 발생 → 트리 재구성

UUID v4 삽입 (완전 랜덤):
  현재: [1a2b...][3c4d...][8a9b...][f1e2...]
  삽입: c3d4... → 8a9b와 f1e2 사이에 끼어야 함 → 페이지 분할
  매 삽입마다 인덱스 임의 위치에서 페이지 분할 반복 → 단편화 폭발

UUID v7 삽입 (타임스탬프 앞에 위치):
  현재: [018fda2a-3b1c][018fda2a-3b20][018fda2a-3b24]
  삽입: 018fda2a-3b28... → 항상 맨 오른쪽 끝에 추가
  페이지 분할이 트리 끝에서만 발생 → Auto Increment와 동일한 성능
```

### 3.4 구성 요소 / 핵심 용어 (Key Terms)

| 용어 | 설명 | 비고 |
|------|------|------|
| **UUID v1** | 타임스탬프 + 클록 시퀀스 + 노드 ID(MAC+PID) 조합 | 시간 순 정렬 가능, JUG로 생성 |
| **UUID v4** | 128비트 완전 랜덤 | java.util.UUID.randomUUID() 기본값 |
| **UUID v7** | 앞 48비트 = Unix 타임스탬프(ms), 나머지 랜덤 | 최신 표준, 정렬 가능 |
| **JUG** | Java UUID Generator, FasterXML 관리 오픈소스 | v1/v3/v4/v5 지원 |
| **UUIDGen.getpid()** | 현재 JVM PID 조회 내부 유틸리티 | 직접 호출 비권장, 팩토리가 내부 사용 |
| **B+Tree** | DB 인덱스의 기본 자료구조, 정렬된 리프 노드 연결 | 순차 삽입에 최적화 |
| **페이지 분할** | B+Tree 노드가 꽉 찰 때 둘로 쪼개지는 현상 | 랜덤 삽입 시 빈번 발생 |
| **Snowflake ID** | Twitter 고안, 64비트(타임스탬프+WorkerID+시퀀스) | 초당 인스턴스당 4,096개 생성 |
| **ULID** | 타임스탬프 + 랜덤, URL-safe, 사전순 정렬 가능 | UUID 호환 대안 |
| **Trace ID** | 분산 시스템에서 요청 전체를 관통하는 불변 식별자 | 최초 생성 후 절대 변경 안 함 |
| **Span ID** | 각 시스템이 요청 수신 시 생성하는 구간 식별자 | 부모 Span ID와 함께 기록 |

---

## 4. 비교 및 구분 (Comparison)

### 4.1 UUID 버전별 비교

| 구분 | v1 (시간+MAC) | v4 (랜덤) | v7 (시간 기반) | Snowflake |
|------|--------------|-----------|---------------|-----------|
| **정렬 가능** | ✅ | ❌ | ✅ | ✅ |
| **DB 인덱스 효율** | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **저장 크기** | 16바이트 | 16바이트 | 16바이트 | 8바이트 |
| **분산 친화성** | ✅ | ✅ | ✅ | ✅ |
| **표준** | RFC 4122 | RFC 4122 | RFC 9562 | 비표준 |
| **Java 생성** | JUG 라이브러리 | java.util.UUID | JUG / 직접 구현 | 직접 구현 |

> UUID v4는 DB PK로 사용 시 삽입 성능 −34.8%, 인덱스 크기 +22%, 버퍼 접근 횟수 8,562,960회
> → v7 전환 시 bigint 수준으로 개선 (Cybertec PostgreSQL 실측 기준)

### 4.2 분산 ID 전략 비교

| 전략 | 원리 | 장점 | 단점 |
|------|------|------|------|
| **다중 마스터 복제** | DB auto_increment를 서버 수만큼 증가 | 간단한 구현 | 서버 추가 시 충돌 위험 |
| **UUID v4** | 완전 랜덤 128비트 | 조율 불필요, 구현 쉬움 | 정렬 불가, 인덱스 성능 저하 |
| **Ticket 서버** | 중앙 서버가 ID 발급 | 순차 숫자 ID 보장 | 중앙 서버 SPoF |
| **UUID v7** | 앞 48비트 타임스탬프 | 정렬 가능, 표준, 조율 불필요 | 표준 대비 신생 |
| **Snowflake/TSID** | 타임스탬프+노드ID+시퀀스 | 64비트, 고성능, 정렬 | 설계 복잡도 높음 |

### 4.3 상황별 선택 기준

```
소규모 단일 DB              → Auto Increment
MSA / 중규모 분산 DB PK     → UUID v7 또는 ULID
초대규모 고성능 분산        → Snowflake ID 또는 TSID
외부 API 연동 노출용 ID     → UUID v7 (표준 호환)
내부 인덱스 + 외부 노출 분리 → 숫자 PK(내부) + UUID(외부) 이중 키 전략
```

---

## 5. 실전 예시 (Examples)

### 5.1 JUG 라이브러리 기본 사용

```xml
<!-- Maven 의존성 -->
<dependency>
  <groupId>com.fasterxml.uuid</groupId>
  <artifactId>java-uuid-generator</artifactId>
  <version>5.1.0</version>
</dependency>
```

```java
import com.fasterxml.uuid.Generators;
import java.util.UUID;

// ✅ 권장: 팩토리 메서드 사용 (v1 시간 기반)
UUID v1 = Generators.timeBasedGenerator().generate();
// 예: 6ba7b810-9dad-11d1-80b4-00c04fd430c8

// v4 랜덤 (표준 Java와 동일)
UUID v4 = Generators.randomBasedGenerator().generate();

// ❌ 비권장: 내부 유틸리티 직접 호출
// int pid = UUIDGen.getpid(); // 내부 구현 세부사항에 의존
```

**포인트:**
- `getpid()`는 팩토리 메서드가 내부적으로 호출하는 유틸리티 — 직접 호출 불필요
- DB PK 용도라면 `timeBasedGenerator()` (v1) 또는 v7 사용

### 5.2 UUID v4 vs v7 실제 값 비교

```
[UUID v4: 완전 랜덤]
550e8400-e29b-41d4-a716-446655440000  ← 연속 생성해도 앞자리가 제각각
f47ac10b-58cc-4372-a567-0e02b2c3d479
9b2a3c7d-1e4f-4b8a-9c0d-2e5f6a7b8c9d
→ B+Tree: 삽입할 때마다 임의 위치 → 페이지 분할 폭발

[UUID v7: 타임스탬프 기반]
018fda2a-3b1c-7e2d-8a4b-1c2d3e4f5a6b  ← 2024-05-01 09:00:00.000
018fda2a-3b20-7a1c-9b3c-2d3e4f5a6b7c  ← 1ms 후
018fda2a-3b24-7f3e-ac4d-3e4f5a6b7c8d  ← 2ms 후
→ 앞 12자리(018fda2a-3b1c)가 타임스탬프(48비트)
→ B+Tree: 항상 맨 오른쪽 끝에 추가 → 페이지 분할 최소
```

### 5.3 분산 추적: 커스텀 GUID + Trace/Span 패턴

```
[GUID 설계 — 한 번 생성 후 절대 불변]
timestamp(14) + WASinstanceID(8) + HEX랜덤(6) + 고정값(2)
→ 예: 20260420112530WASA00013F9B2C00

[요청 전파 시 HTTP 헤더로 분리 관리]
X-Trace-ID:      20260420112530WASA00013F9B2C00  ← 모든 시스템 동일
X-Span-ID:       03                              ← 현재 시스템 홉 번호
X-Parent-Span:   02                              ← 직전 시스템 홉 번호
X-System-Name:   이체WAS

[로그 조회: Trace ID 하나로 전체 흐름 추적]
채널WAS | Span:01 | Parent:없음 | 09:00:00.000 | 요청 수신
입금WAS | Span:02 | Parent:01  | 09:00:00.012 | 계좌 조회
이체WAS | Span:03 | Parent:02  | 09:00:00.025 | 이체 처리
```

### 5.4 주의해야 할 패턴 (Anti-pattern)

```java
// ❌ DB PK에 UUID v4(랜덤) 사용
@Id
@GeneratedValue
@Column(columnDefinition = "BINARY(16)")
private UUID id = UUID.randomUUID();  // 랜덤 → 인덱스 성능 저하
```
> 문제점: 랜덤 값이 B+Tree 임의 위치에 삽입 → 페이지 분할 반복 → 쓰기 성능 급감

```java
// ✅ 시간 기반 UUID 사용 (JUG v1 또는 v7)
@Id
@Column(columnDefinition = "BINARY(16)")
private UUID id = Generators.timeBasedGenerator().generate();  // 순차 삽입
```
> 이유: 타임스탬프가 앞에 위치해 항상 인덱스 맨 끝에 삽입 → Auto Increment 수준 성능

---

```
// ❌ GUID 값 자체를 홉마다 변경 (불변성 파괴)
String guid = baseGuid + hopCount;  // 홉마다 새로운 값 → 참조 무결성 위반
```
> 문제점: 같은 트랜잭션인데 시스템마다 다른 GUID → DB에서 연결 불가, FK 위반

```
// ✅ GUID는 고정, 홉 정보는 헤더로 분리
String traceId = baseGuid;          // 불변
header("X-Trace-ID", traceId);      // 전체 전파
header("X-Span-ID",  newSpanId);    // 현재 구간만
```
> 이유: GUID 불변성을 유지하면서 분산 추적도 가능한 업계 표준 패턴

---

## 6. 트러블슈팅 경험 (Troubleshooting)

### 문제 1. UUIDGen.getpid() 메서드의 역할이 불명확할 때

- **증상**: JUG 라이브러리 코드에서 `UUIDGen.getpid()`가 호출되는데 왜 필요한지 이해 안 됨
- **원인**: UUID v1의 노드 ID 구성 원리를 모르면 PID가 왜 등장하는지 연결이 안 됨
- **해결**:
  ```
  UUID v1 = 타임스탬프(60비트) + 클록 시퀀스(14비트) + 노드 ID(48비트)
  노드 ID는 원래 MAC 주소 → 동일 호스트의 여러 JVM은 MAC이 같음
  → PID를 혼합해 JVM 간 구분 → getpid() 필요
  ```
- **교훈**: 직접 호출할 일은 없다. `Generators.timeBasedGenerator().generate()` 팩토리만 사용하면 내부에서 자동 처리된다.

### 문제 2. Java 8 환경에서 getpid() 접근 실패

- **증상**: Java 8에서 JUG 사용 시 PID 조회가 실패해 `-1` 반환
- **원인**: Java 8은 Reflection으로 `UNIXProcess.pid` 필드에 접근하는데, Security Manager나 OS 환경에 따라 접근이 차단될 수 있음
- **해결**:
  ```xml
  <!-- Java 9 이상으로 마이그레이션하거나, JVM 옵션 추가 -->
  <!-- Java 9+ 에서는 ProcessHandle.current().pid() 공식 API 사용 -->
  ```
  ```java
  // Java 9+: ProcessHandle 공식 API
  long pid = ProcessHandle.current().pid();
  ```
- **교훈**: JUG 팩토리 메서드만 사용하면 내부에서 버전별 폴백 로직이 처리되므로 직접 getpid()를 신경 쓸 필요가 없다.

---

## 7. 실전 명령어 / 치트시트 (Cheat Sheet)

```java
// ─────────────────────────────────────────────
// [JUG 라이브러리 UUID 생성]
// ─────────────────────────────────────────────

// v1: 시간 기반 (DB PK 권장)
UUID v1 = Generators.timeBasedGenerator().generate();

// v4: 랜덤 (외부 노출용 ID)
UUID v4 = Generators.randomBasedGenerator().generate();

// v3: 이름 기반 (MD5)
UUID v3 = Generators.nameBasedGenerator(Generators.NAMESPACE_URL)
                     .generate("https://example.com");

// v5: 이름 기반 (SHA-1, v3보다 보안성 높음)
UUID v5 = Generators.nameBasedGenerator()
                     .generate("unique-name");

// ─────────────────────────────────────────────
// [표준 Java]
// ─────────────────────────────────────────────

// v4 랜덤 (외부 노출용으로만 사용)
UUID uuid = UUID.randomUUID();

// String → UUID 변환
UUID parsed = UUID.fromString("550e8400-e29b-41d4-a716-446655440000");

// ─────────────────────────────────────────────
// [JPA 엔티티 PK 설정 예시]
// ─────────────────────────────────────────────

// ❌ 비권장: 랜덤 UUID → DB 인덱스 성능 저하
@Id
@GeneratedValue
private UUID id;

// ✅ 권장: 시간 기반 UUID → B+Tree 순차 삽입
@Id
@Column(columnDefinition = "BINARY(16)")
private UUID id = Generators.timeBasedGenerator().generate();
```

---

## 8. 깊게 이해하기 (Deep Dive)

### 8.1 B+Tree 페이지 분할 상세

```
[B+Tree 리프 노드 - UUID v4 삽입 시나리오]

현재 상태 (값 기준 정렬):
리프1: [1a2b... | 3c4d... | 5e6f... | 8a9b... | f1e2...] ← 꽉 참

새 UUID v4: c3d4e5f6...
→ 8a9b와 f1e2 사이에 들어가야 함
→ 리프1이 꽉 찼으므로 페이지 분할:
  리프1a: [1a2b... | 3c4d... | 5e6f...]
  리프1b: [8a9b... | c3d4... | f1e2...]  ← 새 노드
  중간값(8a9b)이 부모 노드로 올라감

이 상황이 모든 삽입마다 반복 → 인덱스 단편화, 트리 높이 증가

[UUID v7 삽입: 항상 끝에 추가]
현재:  [018fda2a-3b1c | 018fda2a-3b20 | 018fda2a-3b24]
새값:  018fda2a-3b28 (항상 기존 최대값보다 큼)
→ 맨 오른쪽 리프의 맨 끝에만 추가
→ 분할이 필요하면 오른쪽 끝 노드에서만 1회 발생
→ 나머지 노드는 전혀 건드리지 않음

[핵심 메시지]
성능 차이의 원인은 "해시 재생성"이 아니라
"랜덤 삽입으로 인한 페이지 분할 폭발"이다.
```

### 8.2 성능 특성 및 주의사항

```
[UUID v4 vs v7 실측 비교 - PostgreSQL, Cybertec]
항목                     UUID v4      UUID v7
─────────────────────────────────────────────
삽입 성능                기준         +34.8% 빠름
인덱스 크기              기준         22% 더 작음
인덱스 페이지 충전률     79.06%       90.09%
버퍼 접근 횟수           8,562,960회  bigint 수준
디스크 사용량(1000만 행) 기준         175MB 절감

[저장 공간 비교]
int(32비트):        4바이트
bigint(64비트):     8바이트
Snowflake:          8바이트    ← bigint와 동일
UUID(바이너리):    16바이트
UUID(문자열):      36바이트    ← 하이픈 포함

[문자열 UUID의 추가 비용]
정수형 비교 vs 문자열 비교: 2.5배~28배 성능 차이
→ 가능하면 BINARY(16)으로 저장하고 애플리케이션에서 변환
```

### 8.3 Snowflake ID 구조

```
[64비트 분할]
[ 1비트: 부호 ] [ 41비트: 타임스탬프(ms) ] [ 10비트: Worker ID ] [ 12비트: 시퀀스 ]

Worker ID 10비트 → 최대 1,024개 분산 인스턴스 지원
시퀀스 12비트   → 인스턴스당 초당 최대 4,096개 ID 생성 보장
41비트 타임스탬프 → 2^41ms ≈ 69년간 사용 가능

[장점]
- 64비트 Long 타입 → UUID(128비트) 대비 저장 공간 절반
- 순차 증가 보장 → B+Tree 최적 성능
- 중앙 서버 불필요 → SPoF 없음
```

### 8.4 커스텀 GUID 설계 시 체크리스트

```
GUID 설계 시 반드시 보장해야 할 불변성 원칙:
  ✅ 한 번 생성된 GUID는 절대 변경되지 않는다
  ✅ 동일 GUID = 동일 요청/엔티티 (참조 무결성 보장)
  ✅ 시스템 간 전파 시에도 동일한 값이 그대로 전달된다

GUID에 포함하면 좋은 요소:
  ✅ 타임스탬프 (앞에 배치) → DB 인덱스 성능
  ✅ 인스턴스 ID → 장애 발생 시 원인 시스템 즉시 특정
  ✅ 랜덤 요소 → 동시 생성 시 충돌 방지

GUID에 포함하면 안 되는 요소:
  ❌ 변동 값 (홉 수, 상태, 카운터 등) → 불변성 파괴
  ❌ 민감 정보 (사용자 ID, 계좌번호 등) → 노출 위험

분산 추적이 필요하다면:
  → GUID와 별도로 Trace ID + Span ID 이중 구조 사용
  → HTTP 헤더(X-Trace-ID, X-Span-ID, X-Parent-Span)로 전파
  → 로그에 Trace ID로 인덱싱 → 전체 흐름 단일 조회 가능
```

---

## 9. 나만의 요약 (My Summary)

```
UUID v4(랜덤)를 DB PK로 쓰면 안 되는 이유는 "랜덤"이라는 성질 때문이다.
B+Tree는 값 크기 순으로 정렬해서 저장하는데, 랜덤 UUID는 삽입마다
인덱스의 임의 위치에 끼어들어 페이지 분할을 반복시킨다.
반면 UUID v7은 앞 48비트가 타임스탬프라 새로 생성되는 UUID는 항상
이전 UUID보다 크고, B+Tree 맨 끝에만 추가된다. Auto Increment와 동일한 패턴.

JUG 라이브러리의 getpid()는 UUID v1 생성 시 노드 ID에 PID를 혼합하는
내부 메서드다. 같은 머신에서 여러 JVM이 돌 때 MAC 주소만으로는 구분이
안 되기 때문에 PID를 섞어 고유성을 높인다.
직접 호출할 일은 없고, 팩토리 메서드(timeBasedGenerator)가 알아서 처리한다.

분산 시스템에서 ID를 설계할 때 핵심 원칙 하나:
"GUID는 불변이다." 홉 추적 등 변동 정보는 절대 GUID 값에 담지 말고
HTTP 헤더(X-Trace-ID, X-Span-ID)로 분리해서 전달한다.
```

**기억할 포인트 5가지:**
1. UUID v4는 DB PK 금지 — 랜덤 삽입 → 페이지 분할 폭발 → 쓰기 성능 급감
2. DB PK에는 UUID v7 또는 JUG v1 — 타임스탬프 앞에 배치 → B+Tree 순차 삽입
3. UUIDGen.getpid()는 내부 유틸리티 — 팩토리 메서드만 쓰면 신경 안 써도 됨
4. GUID는 절대 불변 — 홉 수, 상태 등 변동 값을 GUID 값 자체에 담으면 참조 무결성 파괴
5. 분산 추적은 Trace ID(불변) + Span ID(구간별 새 생성) 이중 구조가 업계 표준

**다음에 헷갈릴 것 같은 부분:**
- B+Tree 페이지 분할이 "해시 재생성" 때문이 아니라 "정렬된 위치 찾기 + 노드 꽉 참" 때문임
- UUID v1과 v7은 둘 다 시간 기반이지만, v7이 더 최신 표준(RFC 9562)이고 타임스탬프 위치가 더 앞에 배치됨
- JUG의 getpid()가 Java 8에서 Reflection, Java 9+에서 ProcessHandle로 다르게 동작함
- Snowflake는 64비트(Long)라 UUID(128비트)보다 저장 공간이 절반 — 초대규모 환경에서 유리

---

## 10. 참고 자료 (References)

| 유형 | 제목 | 링크 | 메모 |
|------|------|------|------|
| GitHub | java-uuid-generator (JUG) | https://github.com/cowtowncoder/java-uuid-generator | FasterXML 공식 레포 |
| 공식 문서 | RFC 9562 (UUID v7 표준) | https://www.rfc-editor.org/rfc/rfc9562 | v7 포함 최신 UUID 표준 |
| 공식 문서 | RFC 4122 (기존 UUID 표준) | https://www.rfc-editor.org/rfc/rfc4122 | v1~v5 정의 |
| 블로그 | UUID v4 vs v7 성능 비교 | https://www.cybertec-postgresql.com/en/uuid-serial-or-identity-columns-for-postgresql-primary-key/ | Cybertec PostgreSQL 실측 |
| 공식 문서 | Twitter Snowflake | https://github.com/twitter-archive/snowflake | 원조 Snowflake 설계 문서 |
| 공식 문서 | ProcessHandle (Java 9+) | https://docs.oracle.com/en/java/docs/api/java.base/java/lang/ProcessHandle.html | PID 조회 공식 API |

---

*이 문서는 학습이 깊어질수록 계속 업데이트합니다.*
