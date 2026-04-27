# KNW-spring-transaction

> 학습한 내용을 나중에 언제 봐도 이해할 수 있도록 개념·원리·실제 사례를 함께 기록합니다.

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **주제** | Spring @Transactional 동작 원리, AOP 프록시, 전파 속성, 격리 수준 |
| **분류** | 프레임워크 |
| **키워드** | Spring, @Transactional, AOP, Proxy, CGLIB, JDK Dynamic Proxy, Propagation, Isolation, rollback, PlatformTransactionManager |
| **학습 계기** | 자기계발 |
| **관련 업무 ID** | — |
| **최초 작성일** | 2026-04-27 |
| **최종 수정일** | 2026-04-27 |
| **숙련도** | 🌿 이해 |

---

## 1. 한 줄 요약 (TL;DR)

```
@Transactional은 AOP 프록시를 통해 "커넥션 획득 → setAutoCommit(false) →
비즈니스 로직 → commit/rollback"을 자동화하는 메커니즘이다.
프록시는 외부 호출만 가로챌 수 있으므로, 같은 클래스 내부에서 @Transactional
메서드를 호출하면 트랜잭션이 무시된다.
```

---

## 2. 배경 지식 (Prerequisites)

- **필수 선행 지식**:
  - JDBC Connection, setAutoCommit(), commit(), rollback() 기본 동작
  - 스프링 빈(Bean) 생명주기 및 DI(의존성 주입) 개념
  - 디자인 패턴: Proxy 패턴

- **관련 개념과의 위치**:
  ```
  [Spring Framework]
    ├── IoC Container (Bean 관리)
    │     └── BeanPostProcessor → 프록시 생성·등록
    ├── AOP (Aspect Oriented Programming)
    │     ├── Aspect, Advice, PointCut, JoinPoint, Proxy
    │     └── @Transactional ← 이 문서의 핵심
    └── PlatformTransactionManager
          ├── DataSourceTransactionManager (JDBC)
          ├── JpaTransactionManager (JPA)
          └── HibernateTransactionManager
  ```

---

## 3. 개념 설명 (Concept)

### 3.1 정의 (What)

```
트랜잭션(Transaction)이란 데이터베이스의 상태를 변화시키기 위해 수행하는
논리적 작업의 최소 단위로, 여러 작업이 모두 성공하거나 모두 실패(롤백)하도록
보장하는 메커니즘이다.

스프링 @Transactional은 AOP 프록시를 활용해 트랜잭션 시작/커밋/롤백/
리소스 정리를 개발자 코드에서 완전히 분리하는 선언적 트랜잭션 관리 방식이다.
```

### 3.2 존재 이유 (Why)

```
@Transactional 없이 트랜잭션을 관리하면 모든 서비스 메서드마다
아래 구조를 반복 작성해야 한다:

  Connection conn = dataSource.getConnection();
  conn.setAutoCommit(false);          // 트랜잭션 시작 (직접!)
  try {
      // 비즈니스 로직
      accountRepo.withdraw(conn, fromId, amount);
      accountRepo.deposit(conn, toId, amount);
      conn.commit();                  // 성공 시 커밋 (직접!)
  } catch (Exception e) {
      conn.rollback();                // 실패 시 롤백 (직접!)
  } finally {
      conn.setAutoCommit(true);
      conn.close();                   // 리소스 정리 (직접!)
  }

@Transactional 적용 후:

  @Transactional
  public void transfer(Long fromId, Long toId, int amount) {
      accountRepo.withdraw(fromId, amount);   // 비즈니스 로직만!
      accountRepo.deposit(toId, amount);
  }
```

### 3.3 동작 원리 (How it works)

#### @Transactional 호출 시 내부 동작 순서

```
Step 1. 클라이언트가 @Transactional 메서드 호출
Step 2. 실제 객체 대신 스프링 프록시 객체가 요청을 가로챔
Step 3. TransactionManager → DataSource에서 DB 커넥션 획득
Step 4. setAutoCommit(false) → 트랜잭션 시작
Step 5. 실제 비즈니스 로직(메서드 내용) 실행
Step 6. 정상 종료 → commit() / 예외 발생 → rollback()
Step 7. DB 커넥션 반환 (Connection Pool로 반환)
```

#### 프록시 자동 생성 흐름

```
앱 시작 시:
  BeanPostProcessor가 @Transactional 붙은 클래스 감지
  → 프록시 객체를 자동 생성해 스프링 빈으로 등록

@Autowired OrderService orderService;
// 개발자는 OrderService라고 생각하지만
// 실제로는 OrderService$$CGLIB (프록시 객체)가 주입됨

OrderService$$CGLIB (프록시):
  public void placeOrder() {
      try {
          트랜잭션 시작();         // AOP가 추가한 부가 기능
          실제_placeOrder();      // 핵심 로직 호출
          커밋();
      } catch (Exception e) {
          롤백();
      }
  }
```

### 3.4 구성 요소 / 핵심 용어 (Key Terms)

#### AOP 핵심 용어

| 용어 | 의미 | 트랜잭션 예시 |
|------|------|--------------|
| **Aspect** | 부가 기능 모듈 전체 | 트랜잭션 처리 모듈 |
| **Advice** | 실제 부가 기능 구현체 | 커밋/롤백 로직 |
| **PointCut** | 어느 메서드에 적용할지 기준 | @Transactional 붙은 메서드 |
| **JoinPoint** | Advice가 실제 적용되는 지점 | 메서드 호출 시점 |
| **Proxy** | 원본 객체를 감싸는 대리 객체 | OrderService$$CGLIB |

#### @Transactional 주요 속성

| 속성 | 설명 | 기본값 |
|------|------|--------|
| `propagation` | 트랜잭션 전파 방식 | REQUIRED |
| `isolation` | 동시 트랜잭션 간 격리 수준 | DEFAULT (DB 기본값 사용) |
| `readOnly` | 읽기 전용 여부 | false |
| `rollbackFor` | 롤백을 유발할 예외 클래스 지정 | 미지정 (RuntimeException + Error만 롤백) |
| `noRollbackFor` | 롤백 제외할 예외 클래스 지정 | 미지정 |
| `timeout` | 트랜잭션 제한 시간 (초) | -1 (무제한) |

---

## 4. 비교 및 구분 (Comparison)

### 4.1 선언적 vs 프로그래밍적 트랜잭션

| 구분 | 선언적 트랜잭션 (권장) | 프로그래밍적 트랜잭션 |
|------|----------------------|---------------------|
| **방법** | `@Transactional` 어노테이션 | `TransactionTemplate` 또는 `PlatformTransactionManager` 직접 호출 |
| **장점** | 코드 간결, 비즈니스 로직 집중 | 세밀한 제어 가능 |
| **단점** | 프록시 기반 제약 존재 | 반복 코드 발생 |
| **사용 시점** | 일반적인 서비스 계층 | 트랜잭션 범위를 동적으로 결정해야 할 때 |

### 4.2 JDK 동적 프록시 vs CGLIB 프록시

> ⚠️ **Spring Boot 기본값 주의**: Spring Boot 2.0부터 `spring.aop.proxy-target-class=true`가 기본값으로 변경되어, **인터페이스 유무와 관계없이 CGLIB이 기본으로 사용된다.** JDK 동적 프록시는 `spring.aop.proxy-target-class=false`로 명시해야만 사용된다.

| 방식 | 조건 | 동작 | 제약사항 |
|------|------|------|---------|
| **JDK 동적 프록시** | 인터페이스 기반, `proxy-target-class=false` 설정 시 | 인터페이스를 구현한 프록시 생성 | 인터페이스 메서드만 프록시 가능 |
| **CGLIB 프록시** | Spring Boot 기본값 (`proxy-target-class=true`) | 바이트코드 조작으로 클래스를 **상속**한 프록시 생성 | `final class`, `final method`에는 적용 불가 |

### 4.3 트랜잭션 전파 속성 7가지 (Propagation)

> **핵심 개념**: 스프링은 **물리 트랜잭션**(실제 DB 커넥션)과 **논리 트랜잭션**(스프링이 관리하는 단위)을 구분한다. REQUIRED는 2개의 논리 트랜잭션이 1개의 물리 트랜잭션을 공유하고, REQUIRES_NEW는 각각 별도의 물리 트랜잭션을 사용한다.

| 전파 속성 | 기존 트랜잭션 있을 때 | 기존 트랜잭션 없을 때 | 설명 |
|----------|--------------------|--------------------|------|
| **REQUIRED** *(기본값)* | 기존에 합류 | 새로 생성 | 가장 일반적, 하나의 트랜잭션으로 묶임 |
| **REQUIRES_NEW** | 기존 일시 중단 → 새로 생성 | 새로 생성 | 기존 트랜잭션과 완전히 분리 |
| **NESTED** | Savepoint 생성 후 중첩 실행 | 새로 생성 | 내부만 부분 롤백 가능 |
| **SUPPORTS** | 기존에 합류 | 트랜잭션 없이 실행 | 읽기 전용 조회에 유용 |
| **MANDATORY** | 기존에 합류 | **예외 발생** | 반드시 트랜잭션 안에서 호출되어야 함 |
| **NOT_SUPPORTED** | 기존 일시 중단 | 트랜잭션 없이 실행 | 트랜잭션 없이 실행 강제 |
| **NEVER** | **예외 발생** | 트랜잭션 없이 실행 | 트랜잭션이 있으면 안 됨 |

### 4.4 격리 수준 (Isolation Level)

> `isolation = DEFAULT`는 DB의 기본 격리 수준을 따른다 (MySQL InnoDB: REPEATABLE_READ, PostgreSQL: READ_COMMITTED).

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read | 설명 |
|----------|-----------|-------------------|--------------|------|
| **READ_UNCOMMITTED** | 발생 | 발생 | 발생 | 커밋 안 된 데이터도 읽음 (가장 낮음) |
| **READ_COMMITTED** | 방지 | 발생 | 발생 | 커밋된 데이터만 읽음 (PostgreSQL 기본) |
| **REPEATABLE_READ** | 방지 | 방지 | 발생 | 같은 조회 결과 보장 (MySQL 기본) |
| **SERIALIZABLE** | 방지 | 방지 | 방지 | 완전 직렬화 (성능 최저, 안전 최고) |

```
동시성 문제 설명:
  Dirty Read          → 다른 트랜잭션이 롤백하기 전 데이터를 읽는 문제
  Non-Repeatable Read → 같은 행을 두 번 읽었는데 다른 결과가 나오는 문제 (UPDATE)
  Phantom Read        → 같은 범위를 두 번 조회했는데 행 수가 달라지는 문제 (INSERT/DELETE)
```

### 4.5 언제 무엇을 선택해야 하는가

```
REQUIRED   → 기본값, 같은 작업의 연장선상에 있는 모든 메서드
REQUIRES_NEW → 부모 트랜잭션 실패와 무관하게 독립적으로 커밋해야 할 때
               (예: 감사 로그, 알림 발송)
NESTED     → 배치 처리 중 일부 항목 실패 시 해당 항목만 부분 롤백하고 싶을 때
SUPPORTS   → 트랜잭션이 있으면 같이 쓰고, 없어도 괜찮은 단순 조회
MANDATORY  → 반드시 기존 트랜잭션 안에서만 실행되어야 하는 내부 메서드
NOT_SUPPORTED → 트랜잭션 없이 실행해야 하는 배치성 처리
NEVER      → 트랜잭션이 열려 있으면 버그인 케이스 (방어 코드)
```

---

## 5. 실전 예시 (Examples)

### 5.1 기본 사용 예시

```java
@Transactional
public void transfer(Long fromId, Long toId, int amount) {
    accountRepository.withdraw(fromId, amount);
    accountRepository.deposit(toId, amount);
    // 예외 발생 시 두 작업 모두 rollback 보장
}
```

**포인트**: 어노테이션 하나로 커넥션 획득, autoCommit 설정, commit/rollback, 커넥션 반환을 자동 처리한다.

---

### 5.2 전파 속성 실무 적용 예시

**REQUIRED (기본값) — 하나의 트랜잭션으로 묶임**

```java
// 외부 메서드
@Transactional  // 트랜잭션 A 시작
public void placeOrder(Order order) {
    orderRepository.save(order);     // 트랜잭션 A 사용
    paymentService.pay(order);       // 트랜잭션 A에 합류
}

// 내부 메서드 (다른 Bean)
@Transactional(propagation = Propagation.REQUIRED)
public void pay(Order order) {
    // 새 트랜잭션 생성 없이 외부 트랜잭션 A에 합류
    paymentRepository.save(...);
}
// pay()에서 예외 발생 시 orderRepository.save()도 함께 롤백
```

**REQUIRES_NEW — 독립 트랜잭션**

```java
// 외부 메서드
@Transactional  // 트랜잭션 A 시작
public void placeOrder(Order order) {
    orderRepository.save(order);      // 트랜잭션 A 사용
    logService.saveLog("주문");        // 트랜잭션 A 일시 중단 → 새 트랜잭션 B 시작
}

// 내부 메서드
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveLog(String msg) {
    // 독립된 트랜잭션 B에서 실행
    // saveLog()가 실패해도 트랜잭션 A(주문 저장)는 영향 없음
    logRepository.save(msg);
}
```

---

### 5.3 주의해야 할 패턴 (Anti-pattern)

**같은 클래스 내부 호출 (Self-Invocation)**

```java
// ❌ 잘못된 사용: 전파 속성이 무시됨
@Service
public class OrderService {

    @Transactional
    public void placeOrder() {
        this.saveOrder();  // ← 프록시가 아닌 this(실제 객체)로 직접 호출!
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void saveOrder() {
        // @Transactional이 있어도 완전히 무시됨!
        // 이유: this 호출은 스프링 프록시를 거치지 않음
    }
}
```
> **왜 무시되는가**: 자바의 `this.메서드()` 호출은 스프링 빈(프록시)을 거치지 않고 메모리 상의 실제 객체를 직접 참조하기 때문에 프록시가 가로챌 수 없다.

```java
// ✅ 올바른 사용: 별도 Bean으로 분리
@Service
public class OrderService {
    private final OrderSaveService orderSaveService;  // 별도 Bean 주입

    @Transactional
    public void placeOrder() {
        orderSaveService.saveOrder();  // 프록시를 통해 호출 → 전파 속성 정상 적용
    }
}

@Service
public class OrderSaveService {
    @Transactional(propagation = REQUIRES_NEW)
    public void saveOrder() {
        // 이제 독립 트랜잭션으로 정상 동작
    }
}
```

**CheckedException은 기본적으로 롤백되지 않음**

```java
// ❌ 잘못된 기대: IOException은 CheckedException이므로 롤백 안 됨
@Transactional
public void process() throws IOException {
    repository.save(...);
    throw new IOException("파일 오류");  // 롤백 안 됨!
}

// ✅ 올바른 사용: rollbackFor 명시
@Transactional(rollbackFor = Exception.class)
public void process() throws IOException {
    repository.save(...);
    throw new IOException("파일 오류");  // 롤백됨
}
```

**단순 조회에 무분별한 @Transactional 사용**

```java
// ⚠️ 주의: readOnly = true도 트랜잭션을 실제로 열고 닫음
// 상위 트랜잭션이 없으면 아래 쿼리가 추가로 실행됨:
//   SET autocommit = 0
//   SET session transaction ...
//   SELECT * FROM ...
//   COMMIT
//   SET autocommit = 1

// ❌ 불필요한 @Transactional 남발
@Transactional(readOnly = true)
public Member findOne(Long id) {
    return memberRepository.findById(id).orElseThrow();
    // 단순 조회에 트랜잭션 오버헤드 발생
}

// ✅ JPA 사용 시: readOnly = true는 JPA 최적화 효과가 있어 Service 레이어 조회 메서드에 권장
// (스냅샷 미생성, flush 모드 MANUAL로 변경 → 변경 감지 비용 제거)
@Transactional(readOnly = true)
public List<Order> findAllOrders() {
    return orderRepository.findAll();  // 대량 조회 시 readOnly 효과 유의미
}
```

---

## 6. 트러블슈팅 경험 (Troubleshooting)

### 문제 1. @Transactional이 적용되지 않는 현상

- **증상**: @Transactional 메서드에서 예외가 발생했는데 DB가 롤백되지 않음
- **원인**: 같은 클래스 내부 호출(Self-Invocation) 또는 CheckedException에 rollbackFor 미지정
- **해결**:
  ```java
  // Self-Invocation → 별도 Bean 분리
  // CheckedException → rollbackFor = Exception.class 추가
  @Transactional(rollbackFor = Exception.class)
  public void process() throws IOException { ... }
  ```
- **교훈**: @Transactional 미동작 의심 시 먼저 "외부 호출인지 내부 호출인지"와 "예외 타입"을 확인한다.

### 문제 2. CGLIB 프록시 적용 불가

- **증상**: `@Transactional`이 붙은 메서드에서 트랜잭션 미적용, `Cannot subclass final class` 오류
- **원인**: `final class` 또는 `final method` 사용 (CGLIB은 상속 기반이므로 final 불가)
- **해결**:
  ```java
  // ❌ final class에는 CGLIB 프록시 생성 불가
  @Service
  public final class OrderService { ... }

  // ✅ final 제거
  @Service
  public class OrderService { ... }
  ```
- **교훈**: Spring Boot 환경에서 서비스 클래스와 트랜잭션이 필요한 메서드에 `final`을 붙이지 않는다.

---

## 7. 실전 명령어 / 치트시트 (Cheat Sheet)

```java
// 기본 사용
@Transactional
public void method() { ... }

// 읽기 전용 (JPA 최적화 + 의도 명시)
@Transactional(readOnly = true)
public List<Foo> findAll() { ... }

// CheckedException도 롤백
@Transactional(rollbackFor = Exception.class)
public void method() throws Exception { ... }

// 특정 예외는 롤백 제외
@Transactional(noRollbackFor = BusinessException.class)
public void method() { ... }

// 독립 트랜잭션 (감사 로그, 알림 등)
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAuditLog(...) { ... }

// 타임아웃 설정 (5초)
@Transactional(timeout = 5)
public void longTask() { ... }

// 격리 수준 명시
@Transactional(isolation = Isolation.READ_COMMITTED)
public void sensitiveRead() { ... }
```

---

## 8. 깊게 이해하기 (Deep Dive)

### 8.1 외부 호출 vs 내부 호출 프록시 동작 비교

```
외부 호출 (정상 동작):
  컨트롤러
    ↓
  [OrderService 프록시] ← 스프링이 주입한 프록시 객체
    │ ① 요청 가로챔
    │ ② 트랜잭션 시작
    ↓
  [실제 OrderService] ← placeOrder() 실행
    ↓
  [프록시] ← 커밋 / 롤백

내부 호출 (Self-Invocation, 트랜잭션 무시):
  컨트롤러
    ↓
  [OrderService 프록시]
    ↓
  [실제 OrderService] ← placeOrder() 실행 중
    └──▶ this.saveOrder()  ← 프록시 없이 실제 객체가 자기 자신을 직접 호출
                              프록시가 개입할 틈이 없음!
```

| 상황 | 프록시 경유 | 트랜잭션 적용 |
|------|-----------|-------------|
| 외부(컨트롤러 등)에서 @Transactional 메서드 호출 | ✅ 경유함 | ✅ 정상 적용 |
| 같은 클래스 내부에서 @Transactional 메서드 호출 | ❌ 우회됨 | ❌ 무시됨 |

### 8.2 readOnly = true의 JPA/Hibernate 최적화 효과

```
@Transactional(readOnly = true) 를 설정하면:

1. JPA 1차 캐시 스냅샷(Snapshot) 미생성
   → 변경 감지(Dirty Checking)를 위한 복사본을 만들지 않으므로 메모리 절감

2. flush 모드가 MANUAL로 변경
   → 트랜잭션 종료 시 변경 사항을 DB에 쓰는 flush 동작이 비활성화
   → 실수로 엔티티를 수정해도 DB에 반영되지 않음 (안전)

3. DB 수준 최적화 (일부 DB, 드라이버)
   → 읽기 전용 힌트를 DB에 전달해 읽기 복제본(Slave) 라우팅 가능

⚠️ 주의: readOnly여도 상위 트랜잭션이 없으면 새 트랜잭션이 열린다.
단순 조회 1건에도 SET autocommit=0 / COMMIT / SET autocommit=1 등의 쿼리가 추가 실행된다.
```

### 8.3 Spring Boot의 CGLIB 기본값 변경 이력

```
Spring (Boot 이전 / Boot 1.x):
  - 기본값: proxy-target-class=false
  - 인터페이스 있음 → JDK 동적 프록시
  - 인터페이스 없음 → CGLIB

Spring Boot 2.0+:
  - 기본값: proxy-target-class=true
  - 인터페이스 유무와 무관하게 → CGLIB 기본 사용
  - JDK 동적 프록시 원하면: spring.aop.proxy-target-class=false 명시 필요

Spring Boot 3.x (Spring Framework 6.x):
  - 동일하게 CGLIB 기본값 유지
  - CGLIB 제약: final class, final method 사용 불가 (상속 기반이므로)
```

### 8.4 istiod 구성요소 의존 관계 — @Transactional 처리 흐름 전체

```
앱 시작 시 준비 단계:
  @Transactional 클래스 감지
    → BeanPostProcessor
    → CGLIB 프록시 생성 (클래스 바이트코드 조작, 상속)
    → 프록시를 스프링 빈으로 등록

런타임 처리 단계:
  클라이언트 → 프록시.placeOrder()
    → TransactionInterceptor (AOP Advice)
    → TransactionManager.getTransaction()
    → DataSource.getConnection()
    → connection.setAutoCommit(false)
    → 실제 OrderService.placeOrder()
    → (성공) connection.commit()
    → (실패) connection.rollback()
    → connection 반환 (Pool)
```

---

## 9. 나만의 요약 (My Summary)

```
@Transactional = "비서(프록시)" + "금고(트랜잭션)"

비서(프록시): 외부에서 전화가 오면(외부 호출) 가로채서 트랜잭션을 열고 닫는다.
              같은 사무실 내 직통 전화(this 내부 호출)는 가로채지 못한다.

금고(트랜잭션): setAutoCommit(false)로 문을 잠그고, 성공하면 커밋(열쇠), 실패하면 롤백(없던 일).

Spring Boot에서 프록시는 무조건 CGLIB (인터페이스 있어도), 단 final에는 못 씀.
```

**기억할 포인트 3가지:**
1. 같은 클래스 내부 호출은 프록시를 우회 → **@Transactional 무시** → 별도 Bean으로 분리
2. **CheckedException은 기본 롤백 안 됨** → rollbackFor = Exception.class 명시 필요
3. **Spring Boot는 CGLIB이 기본** (인터페이스 있어도) → final class/method에는 프록시 적용 불가

**다음에 헷갈릴 것 같은 부분:**
- REQUIRED에서 내부 롤백 → 외부도 함께 롤백 (같은 물리 트랜잭션이므로)
- REQUIRES_NEW는 "일시 중단"이지 "취소"가 아님 — 내부가 커밋되고 외부가 나중에 롤백되면 내부는 이미 커밋된 상태
- readOnly = true도 새 트랜잭션을 열고 닫는다 (상위 없을 때)

---

## 10. 참고 자료 (References)

| 유형 | 제목 | 링크 | 메모 |
|------|------|------|------|
| 공식 문서 | Spring Transaction Management | https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction | |
| 공식 문서 | Spring AOP | https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop | |
| 블로그 | 카카오페이 @Transactional readOnly 분석 | https://tech.kakaopay.com/post/jpa-transactional-bri/ | readOnly의 실제 쿼리 발생 분석 |
| 블로그 | 트랜잭션 전파 상세 | https://mangkyu.tistory.com/269 | 전파 속성 상세 설명 |
| 블로그 | AOP와 트랜잭션 관계 | https://f-lab.kr/insight/spring-aop-transaction-management-20250207 | |
| 블로그 | Self-Invocation 문제 | https://curiousjinan.tistory.com/entry/spring-transactional-not-working-in-transactional | |
| 블로그 | AOP와 Proxy 이해 | https://f-lab.kr/insight/understanding-spring-aop-and-proxy | |

---

*이 문서는 학습이 깊어질수록 계속 업데이트합니다.*
