# 지식 정리 (Knowledge Note)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **주제** | 분산 시스템에서의 채번(ID 생성) 전략과 DB 락 |
| **분류** | 인프라 / 데이터베이스 / 분산 시스템 |
| **키워드** | 채번, DB 락, Snowflake ID, Redis INCR, SELECT FOR UPDATE, Gap Lock, Oracle SEQUENCE, AUTO_INCREMENT, ULID, UPSERT, MERGE |
| **학습 계기** | 분산 DB 환경에서 SSOT 기반 시퀀스 채번 시 가용성 저하 문제 탐구 |
| **관련 업무 ID** | - |
| **최초 작성일** | 2026-04-27 |
| **최종 수정일** | 2026-04-27 |
| **숙련도** | 🌿 이해 |

---

## 1. 한 줄 요약 (TL;DR)

```
DB를 SSOT로 삼아 시퀀스를 채번하면 단일 row에 락 경합이 집중되어
직렬화 병목 → 커넥션 풀 고갈 → 서비스 장애로 번진다.
"전역 유일성"과 "가용성"은 트레이드오프 관계이므로
결번 허용 여부·원자성 요구·분산 규모에 따라 채번 전략을 선택해야 한다.
```

---

## 2. 배경 지식 (Prerequisites)

- **필수 선행 지식**:
  - SSOT(Single Source of Truth) 개념 — 단일 진실 공급원
  - 트랜잭션 ACID 속성 (특히 Isolation, Atomicity)
  - 비관적 락(Pessimistic Lock) vs 낙관적 락(Optimistic Lock)
  - DB 인덱스 구조 (B-Tree, 클러스터드 인덱스)

- **관련 개념과의 관계**:
  ```
  [채번 요청]
    ├── DB 기반 채번 ──────── 단일 row 락 경합 → 직렬화
    │     ├── MAX+1           (락 없음, 중복 위험)
    │     ├── 채번 테이블     (FOR UPDATE, 안전하지만 느림)
    │     │     └── 구분자 기반 채번 (계좌별 row 샤딩)
    │     └── SEQUENCE / AUTO_INCREMENT (DBMS 내장)
    │
    └── 분산 채번 ────────── 락 없이 분산 생성
          ├── Redis INCR      (원자 카운터, 순서 보장)
          ├── Snowflake ID    (타임스탬프+노드+시퀀스)
          ├── UUID v4         (완전 랜덤, 순서 미보장)
          ├── ULID            (타임스탬프 접두어 + 랜덤, 정렬 가능)
          └── ZooKeeper       (Zab 합의, 강한 일관성)
  ```

---

## 3. 개념 설명 (Concept)

### 3.1 정의 (What)

```
채번(採番)이란 시스템 내에서 중복 없이 유일한 순번/식별자를 발급하는 행위다.
분산 시스템에서는 여러 노드가 동시에 번호를 발급해야 하므로,
단일 DB에 의존하는 방식은 병목을 만들고 가용성을 저하시킨다.
```

### 3.2 존재 이유 (Why)

```
DB SSOT 방식의 채번은 다음 문제를 유발한다.

① 처리량 병목
   모든 채번 요청 → 단일 row의 Read-Modify-Write 패턴
   → 비관적 락으로 직렬화 → 초당 발급 ID 수 = DB 단일 row 처리 속도에 종속

② 커넥션 풀 고갈
   락 대기 트랜잭션이 DB 커넥션을 점유한 채 대기
   → 커넥션 풀 소진 → 전체 서비스 장애

③ 분산 DB에서 전역 유일성 보장 불가
   여러 노드 존재 시 단일 노드 락만으로는 전역 유일성 보장 불가
   → 별도 동기화 레이어 필요
```

### 3.3 동작 원리 (How it works)

**채번 테이블 방식 — 락 흐름**

```
Step 1. 트랜잭션 A: seq_table의 채번 row에 배타락(X Lock) 획득
Step 2. 트랜잭션 B, C, D: 대기 큐에서 블로킹
Step 3. 트랜잭션 A: seq + 1 UPDATE → COMMIT → 락 해제
Step 4. 트랜잭션 B가 락 획득 → 동일 과정 반복 (완전 직렬 처리)
```

**Oracle SEQUENCE 내부 — 락 계층**

```
NEXTVAL 호출
  ├── CACHE 옵션  → 메모리 캐시에서 발급 → SQ 락만 사용 (경량)
  ├── NOCACHE     → 매 호출마다 딕셔너리 물리 변경 → Row Cache 락 (무거움)
  └── CACHE + ORDER (RAC 환경)
                  → 노드 간 순서 보장 → SV 락 추가 (가장 무거움)
```

**MySQL InnoDB — AUTO-INC 락 계층**

```
INSERT 실행
  ├── innodb_autoinc_lock_mode = 0 → INSERT 끝까지 테이블 락 유지
  ├── innodb_autoinc_lock_mode = 1 → 단순 INSERT는 경량 뮤텍스 (기본값)
  └── innodb_autoinc_lock_mode = 2 → 락 없이 원자 카운터 (Binlog 복제 주의)
```

### 3.4 구성 요소 / 핵심 용어 (Key Terms)

| 용어 | 설명 | 비고 |
|------|------|------|
| SSOT | Single Source of Truth. 특정 데이터의 유일한 원본 | DB를 SSOT로 삼으면 채번 병목 발생 |
| 채번 | 중복 없는 순번/식별자 발급 | 결번 허용 여부가 설계 분기점 |
| 구분자 기반 채번 | 계좌번호 등 비즈니스 키를 채번 테이블 PK로 사용해 락 경합을 분산 | 계좌 수만큼 병렬화 |
| 비관적 락 | SELECT FOR UPDATE 등으로 미리 잠금 | 데드락 방지, 성능 낮음 |
| 낙관적 락 | 충돌 발생 시 재시도 | 경합 적을 때 유리 |
| Gap Lock | MySQL InnoDB에서 범위 조건 시 빈 공간까지 잠금 | Phantom Read 방지 목적 |
| Next-Key Lock | Record Lock + Gap Lock의 조합 | MySQL REPEATABLE READ 기본 |
| Snowflake ID | Twitter가 개발한 분산 ID 구조 | 41+5+5+12 bit |
| ULID | Crockford Base32 인코딩, 정렬 가능 UUID 대안 | 128 bit |
| Zab | ZooKeeper Atomic Broadcast. ZooKeeper 전용 합의 프로토콜 | Paxos에서 영감, 별개 프로토콜 |
| UPSERT | INSERT가 PK 중복 시 UPDATE로 자동 전환 | MySQL: ON DUPLICATE KEY UPDATE |
| MERGE | 존재 여부를 원자적으로 판단 후 INSERT/UPDATE 분기 | Oracle 전용 문법 |

---

## 4. 비교 및 구분 (Comparison)

### 4.1 채번 방식 종합 비교

| 전략 | 방식 | 유일성 | 성능 | 순서 보장 | 결번 | SPOF |
|------|------|--------|------|-----------|------|------|
| **DB Auto Increment** | 단일 row 락 | ✅ | ❌ 낮음 | ✅ | ⚠️ 가능 | ❌ DB |
| **채번 테이블** | SELECT FOR UPDATE | ✅ | ❌ 낮음 | ✅ | ⚠️ 조기COMMIT 시 | ❌ DB |
| **MAX+1** | 락 없이 본테이블 스캔 | ⚠️ 중복 위험 | ⚠️ 중간 | ✅ | ❌ 없음 | ❌ DB |
| **Redis INCR** | 원자적 카운터 | ✅ | ✅ 높음 | ✅ | ⚠️ 재시작 시 | ⚠️ Redis |
| **Snowflake ID** | 타임스탬프+노드+시퀀스 | ✅ | ✅ 매우 높음 | ⚠️ 대략적 | ⚠️ 있음 | ✅ 없음 |
| **UUID v4** | 완전 랜덤 128bit | ✅ 확률적 | ✅ 매우 높음 | ❌ | ⚠️ 있음 | ✅ 없음 |
| **ULID** | 타임스탬프 접두어+랜덤 | ✅ 확률적 | ✅ 매우 높음 | ✅ 시간순 정렬 | ⚠️ 있음 | ✅ 없음 |
| **ZooKeeper** | Zab 합의 기반 | ✅ 강력 | ⚠️ 중간 | ✅ | ❌ 없음 | ⚠️ ZK 클러스터 |

### 4.2 Oracle vs MySQL 락 동작 비교

| 항목 | Oracle | MySQL InnoDB |
|------|--------|-------------|
| 읽기 락(SELECT) | ❌ 없음 (UNDO로 해결) | ⚠️ 격리 수준에 따라 공유락 |
| 쓰기 락(UPDATE) | 해당 row 배타락 | 해당 row + Gap Lock |
| Lock Escalation (행→테이블) | ❌ 절대 없음 | ✅ 인덱스 없으면 사실상 발생 |
| FOR UPDATE 인덱스 없을 때 | 조건 매칭 row만 | 스캔한 전체 row |
| FOR UPDATE 인덱스 있을 때 | 조건 매칭 row만 | 조건 매칭 row + Gap Lock |
| Gap Lock | ❌ 없음 | ✅ REPEATABLE READ 기본 발생 |

### 4.3 Redis 채번 vs DB 구분자 기반 채번 비교

| 항목 | Redis INCR | DB 구분자 기반 채번 |
|------|-----------|-------------------|
| 성능 | ✅ 매우 빠름 | ❌ 상대적으로 느림 |
| 채번+저장 원자성 | ❌ 불가 (별개 시스템) | ✅ 단일 트랜잭션 가능 |
| 영속성 | ⚠️ AOF/RDB 설정 필요 | ✅ ACID 보장 |
| 인프라 복잡도 | ⚠️ Redis 별도 운영 | ✅ DB만으로 완결 |
| 채번 이력 추적 | ❌ 없음 | ✅ 컬럼 추가로 가능 |
| 결번 허용 여부 | ⚠️ 재시작 시 발생 가능 | ✅ 트랜잭션 롤백으로 통제 |

### 4.4 언제 무엇을 선택해야 하는가

```
결번 절대 불가 + 채번-저장 원자성 필요 + 단일 DB 환경
  → 채번 테이블 (FOR UPDATE) + 계좌별 구분자 + MERGE/UPSERT로 신규 계좌 처리

결번 허용 + 단일 DB 환경
  → Oracle SEQUENCE(CACHE + NOORDER) 또는 MySQL AUTO_INCREMENT(mode=1 또는 2)

분산 환경 + 순서 보장 필요 + 결번 허용 가능
  → Redis INCR (계좌별 key로 샤딩) 또는 Redis Cluster

분산 환경 + 초고성능 + 순서 대략적 허용
  → Snowflake ID (락 없이 노드 독립 생성)

UUID 대안 + 정렬 필요
  → ULID (Crockford Base32, 시간순 정렬 보장)

이미 Redis 채번 사용 중, 원자성/영속성 문제 발생
  → DB 구분자 기반 채번으로 전환 검토
```

---

## 5. 실전 예시 (Examples)

### 5.1 채번 테이블 — 기본 구현

```sql
-- 채번 테이블 생성 (Oracle / MySQL 공통)
CREATE TABLE seq_manager (
  table_name  VARCHAR(50) PRIMARY KEY,  -- 채번 대상 구분자
  current_seq NUMBER(10)                -- 현재 최종 발급 번호
);

-- 채번 처리 흐름
-- Step 1: 해당 row에 배타락 획득
SELECT current_seq FROM seq_manager
  WHERE table_name = 'ORDERS'
  FOR UPDATE;

-- Step 2: 채번 값 +1 갱신
UPDATE seq_manager
  SET current_seq = current_seq + 1
  WHERE table_name = 'ORDERS';

-- Step 3: 본 테이블 INSERT
INSERT INTO orders (order_id, ...) VALUES (:new_seq, ...);

-- Step 4: COMMIT → 락 해제
COMMIT;
```

**포인트:** table_name이 PK이므로 Oracle/MySQL 모두 자동 인덱스 생성. 서로 다른 table_name 채번은 병렬 처리 가능.

---

### 5.2 락 경합 최소화 — 조기 COMMIT 기법

```
[일반 방식]                     [조기 COMMIT 방식]
├─ SELECT FOR UPDATE            ├─ SELECT FOR UPDATE
├─ UPDATE seq_manager           ├─ UPDATE seq_manager
├─ (긴 비즈니스 로직)           ├─ COMMIT  ← 채번 트랜잭션 즉시 락 해제
├─ INSERT into orders           ├─ (긴 비즈니스 로직)
└─ COMMIT (락 해제)             ├─ INSERT into orders
   ↑ 락 오래 점유               └─ COMMIT
```

> **주의:** 조기 COMMIT 후 INSERT 실패 시 채번된 번호는 **결번**으로 남는다.
> 결번 허용 여부를 비즈니스와 반드시 합의해야 한다.

---

### 5.3 계좌별 구분자로 락 샤딩

```sql
-- 채번 테이블에 account_no 구분자 추가 (일자별 복합 PK)
CREATE TABLE seq_manager (
  account_no  VARCHAR(20),
  seq_date    VARCHAR(8),    -- 일자별 순번 필요 시
  current_seq NUMBER(10),
  PRIMARY KEY (account_no, seq_date)
);

-- ACC001 채번 → ACC001 row만 락
SELECT current_seq FROM seq_manager
  WHERE account_no = 'ACC001' AND seq_date = '20260427'
  FOR UPDATE;
```

**효과:** 계좌 수만큼 락 병렬화 → 동일 계좌 요청끼리만 직렬화

> **중요:** 채번 요청은 항상 "특정 계좌의 채번"이라는 맥락으로 들어온다.
> account_no는 요청 시점에 이미 알고 있는 값이므로, WHERE 절에 특정 계좌를 명시하는 것이 정상적인 사용 패턴이다.
> "전체에서 가장 큰 시퀀스를 가진 계좌를 찾는다"는 것은 구분자 기반 채번의 목적이 아니다.

> **MySQL 주의:** account_no에 인덱스가 없으면 풀스캔 → 전체 row 락 → 샤딩 효과 없음

---

### 5.4 신규 계좌 첫 채번 — UPSERT/MERGE 처리

당일 처음 채번하는 계좌는 seq_manager에 해당 row가 없다. SELECT FOR UPDATE 시 row가 없으면 락도 없고 반환도 없어, 동시 요청 시 PK 중복 충돌이 발생한다.

```
-- 문제 상황: ACC003 당일 첫 채번 동시 요청
트랜잭션 A: SELECT → 없음 → INSERT (ACC003, 20260427, 1) ┐
트랜잭션 B: SELECT → 없음 → INSERT (ACC003, 20260427, 1) ┘ → PK 중복 충돌!
```

**Oracle — MERGE 문으로 해결**

```sql
MERGE INTO seq_manager s
USING (SELECT 'ACC003' AS account_no, '20260427' AS seq_date FROM dual) src
ON (s.account_no = src.account_no AND s.seq_date = src.seq_date)
WHEN MATCHED THEN
  UPDATE SET current_seq = current_seq + 1    -- 기존 row: 증가
WHEN NOT MATCHED THEN
  INSERT (account_no, seq_date, current_seq)
  VALUES ('ACC003', '20260427', 1);            -- 신규 row: 1로 삽입
```

> MERGE는 내부적으로 row 존재 여부를 원자적으로 판단하므로 동시성 문제를 해결한다.

**MySQL — INSERT ... ON DUPLICATE KEY UPDATE 로 해결**

```sql
INSERT INTO seq_manager (account_no, seq_date, current_seq)
VALUES ('ACC003', '20260427', 1)
ON DUPLICATE KEY UPDATE
  current_seq = current_seq + 1;

-- 채번된 값 조회
SELECT current_seq FROM seq_manager
  WHERE account_no = 'ACC003' AND seq_date = '20260427';
```

> PK 중복 시 자동으로 UPDATE로 전환되어 동시 요청도 안전하게 처리된다.

---

### 5.5 Oracle SEQUENCE 생성

```sql
-- Oracle SEQUENCE 생성 (권장 옵션)
CREATE SEQUENCE order_seq
  START WITH 1
  INCREMENT BY 1
  MAXVALUE 99999999
  NOCYCLE
  CACHE 50      -- 50개 미리 메모리에 올려둠 (SQ 락만 사용, 빠름)
  NOORDER;      -- RAC 환경에서 순서 보장 불필요 시

-- 사용
INSERT INTO orders (order_id) VALUES (order_seq.NEXTVAL);
```

---

### 5.6 MySQL AUTO_INCREMENT 설정

```sql
-- 테이블 생성 시 설정
CREATE TABLE orders (
  order_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  ...
);

-- 락 모드 확인
SHOW VARIABLES LIKE 'innodb_autoinc_lock_mode';
-- my.cnf에서 변경: innodb_autoinc_lock_mode = 2
```

---

### 5.7 주의해야 할 패턴 (Anti-pattern)

```sql
-- ❌ Oracle에서 VARCHAR 타입 순번 컬럼에 MAX+1 사용
SELECT MAX(order_id) + 1 FROM orders;
-- 문자 사전순 정렬: '9' > '11' → MAX가 '9'가 되어 채번이 10에서 멈춤
```
> **문제점:** VARCHAR 컬럼에 MAX()는 숫자가 아닌 사전순 비교를 한다.

```sql
-- ✅ 올바른 사용법 (Oracle)
SELECT NVL(MAX(TO_NUMBER(order_id)), 0) + 1 FROM orders;
```

---

```sql
-- ❌ MySQL에서 인덱스 없는 컬럼에 SELECT FOR UPDATE
SELECT current_seq FROM seq_manager
  WHERE account_no = 'ACC001'  -- account_no에 인덱스 없음!
  FOR UPDATE;
-- 풀스캔 → 전체 row에 X Lock → WHERE 조건 무의미
```
> **문제점:** MySQL은 스캔한 모든 row를 잠근다. 계좌별 샤딩 효과가 사라진다.

```sql
-- ✅ account_no에 인덱스 생성
CREATE INDEX idx_seq_manager_account ON seq_manager(account_no);
```

---

## 6. 트러블슈팅 경험 (Troubleshooting)

### 문제 1. 채번 병목으로 커넥션 풀 고갈

- **증상**: 특정 시간대 서비스 전체 요청 실패. "Connection pool exhausted" 에러.
- **원인**: 채번 테이블 단일 row에 모든 요청이 직렬 대기 → 트랜잭션 처리 시간 증가 → 커넥션 점유 시간 증가 → 풀 고갈
- **해결**:
  ```sql
  -- 채번 트랜잭션을 조기 COMMIT하여 락 점유 시간 최소화
  SELECT current_seq FROM seq_manager WHERE table_name = 'ORDERS' FOR UPDATE;
  UPDATE seq_manager SET current_seq = current_seq + 1 WHERE table_name = 'ORDERS';
  COMMIT;  -- ← 이 시점에서 다른 트랜잭션이 즉시 채번 가능
  INSERT INTO orders (...) VALUES (...);
  COMMIT;
  ```
- **교훈**: 채번 트랜잭션 범위를 비즈니스 로직과 분리. 결번 허용 여부 사전 합의 필수.

### 문제 2. MySQL WHERE 조건이 있는데도 전체 락 발생

- **증상**: account_no 조건을 명시했음에도 다른 계좌 채번이 블로킹됨.
- **원인**: account_no 컬럼에 인덱스가 없어 풀스캔 발생 → 전체 row에 X Lock
- **해결**:
  ```sql
  CREATE INDEX idx_seq_account ON seq_manager(account_no);
  EXPLAIN SELECT ... FOR UPDATE;  -- 인덱스 사용 여부 확인
  ```
- **교훈**: MySQL에서 FOR UPDATE는 "조건에 맞는 row"가 아닌 "스캔한 row"를 잠근다. 인덱스 설계가 락 범위를 결정한다.

### 문제 3. 신규 계좌 첫 채번 시 PK 중복 에러

- **증상**: 당일 처음 요청하는 계좌에서 간헐적 PK 중복 오류 발생.
- **원인**: seq_manager에 해당 (account_no, seq_date) row가 없는 상태에서 동시 요청이 몰려 SELECT FOR UPDATE가 락을 잡지 못하고 두 트랜잭션 모두 INSERT 시도.
- **해결**:
  - Oracle: MERGE 문으로 원자적 Upsert 처리
  - MySQL: INSERT ... ON DUPLICATE KEY UPDATE 사용
- **교훈**: 채번 테이블의 row가 존재한다는 전제를 항상 확인해야 한다. 신규 항목 첫 채번은 일반 채번과 다른 코드 경로가 필요하다.

### 문제 4. Redis 채번 후 DB INSERT 실패로 결번 발생

- **증상**: Redis INCR은 성공했으나 이후 DB INSERT 실패 시 채번 번호 소멸, 결번 발생.
- **원인**: Redis와 DB는 서로 다른 시스템 → 두 작업을 원자적 트랜잭션으로 묶을 수 없음.
- **해결**: 결번이 비즈니스적으로 허용 불가한 경우, DB 구분자 기반 채번으로 전환하여 채번과 INSERT를 단일 트랜잭션으로 처리.
- **교훈**: Redis 채번은 "채번 성공 = 데이터 저장 성공"을 보장하지 못한다. 원자성이 중요한 금융/회계 도메인에서는 DB 기반 채번이 더 안전하다.

---

## 7. 실전 명령어 / 치트시트 (Cheat Sheet)

```sql
-- [Oracle] SEQUENCE 생성 (캐시 50, 순서 미보장)
CREATE SEQUENCE order_seq START WITH 1 INCREMENT BY 1 CACHE 50 NOORDER;

-- [Oracle] 다음 값 조회
SELECT order_seq.NEXTVAL FROM dual;

-- [Oracle] 신규 계좌 원자적 채번 (MERGE)
MERGE INTO seq_manager s
USING (SELECT 'ACC003' AS account_no, '20260427' AS seq_date FROM dual) src
ON (s.account_no = src.account_no AND s.seq_date = src.seq_date)
WHEN MATCHED THEN UPDATE SET current_seq = current_seq + 1
WHEN NOT MATCHED THEN INSERT (account_no, seq_date, current_seq) VALUES ('ACC003', '20260427', 1);

-- [Oracle] VARCHAR 컬럼 MAX+1 안전하게
SELECT NVL(MAX(TO_NUMBER(order_id)), 0) + 1 FROM orders;

-- [MySQL] AUTO_INCREMENT 락 모드 확인
SHOW VARIABLES LIKE 'innodb_autoinc_lock_mode';

-- [MySQL] 신규 계좌 원자적 채번 (UPSERT)
INSERT INTO seq_manager (account_no, seq_date, current_seq) VALUES ('ACC003', '20260427', 1)
ON DUPLICATE KEY UPDATE current_seq = current_seq + 1;

-- [MySQL] FOR UPDATE 락 범위 분석
EXPLAIN SELECT current_seq FROM seq_manager
  WHERE account_no = 'ACC001' FOR UPDATE;

-- [MySQL] 인덱스 생성 (계좌별 채번 병렬화 필수)
CREATE INDEX idx_seq_manager_account ON seq_manager(account_no);

-- [Redis] 계좌별 원자 카운터 채번
INCR seq:account:ACC001
-- TTL 설정 (일자별 리셋)
EXPIRE seq:account:ACC001:20260427 86400
```

---

## 8. 깊게 이해하기 (Deep Dive)

### 8.1 Snowflake ID 구조 (오류 수정)

```
[63 bit signed long]
 ├─ 41 bit : timestamp (ms 단위, Epoch 기준)
 ├─  5 bit : datacenter ID (0~31)
 ├─  5 bit : worker ID (0~31)
 └─ 12 bit : sequence (밀리초 내 순번, 0~4095)
```

> **⚠️ 오류 수정**: 원문 "초당 최대 4096개/노드"는 **단위 오류**다.
> 12-bit sequence field는 **동일한 밀리초 안에** 최대 2¹² = 4096개를 보장한다.
> 즉, **밀리초당 4096개** = 초당 약 **409만 6000개**/노드.
> "초당 4096개"로 이해하면 실제 처리량을 1000배 과소평가하게 된다.

**한계**: 노드 시계가 역행하면 ID 중복 가능성 존재. NTP 동기화와 시계 역행 방지 로직 필수.

---

### 8.2 ZooKeeper 합의 알고리즘 (오류 수정)

> **⚠️ 오류 수정**: 원문 "Paxos 기반 강한 일관성"은 부정확하다.
> ZooKeeper는 Paxos에서 영감을 받아 별도로 설계한 **Zab(ZooKeeper Atomic Broadcast)** 프로토콜을 사용한다.

**Zab vs Paxos 차이점:**

| 항목 | Paxos | Zab |
|------|-------|-----|
| 설계 목적 | 범용 분산 합의 | Primary-Backup 복제 특화 |
| 단계 | Prepare → Accept → Commit | Discovery → Sync → Broadcast |
| Leader 강조 | 없음 | Leader Epoch 명시적 관리 |
| 사용 시스템 | Chubby, etcd (Raft) | ZooKeeper |

ZooKeeper로 분산 락을 구현하면 강한 일관성은 보장되지만 Redis INCR보다 지연 시간이 높다.

---

### 8.3 구분자 기반 채번 — "전체 MAX 조회" 오해

구분자 기반 채번에서 자주 생기는 오해: "어떤 계좌가 가장 큰 시퀀스를 갖고 있는지 모르니까 전체 row를 스캔해야 하지 않나?"

**이것은 잘못된 전제다.** 채번 목적은 "전체 중 MAX를 찾는 것"이 아니라 "내 계좌의 현재 순번을 +1 하는 것"이다.

```
사용자 요청: "ACC001 계좌로 거래를 발생시켜주세요"
          ↓
애플리케이션: ACC001이 오늘 몇 번째 거래인지 채번
          ↓
SQL: WHERE account_no = 'ACC001'  ← 이미 알고 있는 값
```

채번 요청은 항상 특정 계좌 단위로 들어오기 때문에 account_no는 **이미 주어진 값**이다. 전체 row 스캔이 필요한 상황이 생긴다면, 그것은 구분자 기반 채번이 맞지 않는 요구사항이라는 신호다.

```
[ACC001 트랜잭션]                [ACC002 트랜잭션]  ← 동시 발생
  WHERE account_no = 'ACC001'     WHERE account_no = 'ACC002'
  FOR UPDATE                      FOR UPDATE
  → ACC001 row만 락               → ACC002 row만 락
  → 병렬 처리 ✅                   → 병렬 처리 ✅
```

---

### 8.4 Redis 채번의 숨겨진 문제점

Redis INCR는 빠르지만 다음 리스크가 존재한다.

```
① 데이터 휘발성
   AOF/RDB 영속성 설정 없거나 부족 → 재시작 시 카운터 초기화 → 채번 중복 발생

② 2개 시스템 원자성 문제
   Redis INCR 성공 후 DB INSERT 실패 시:
     → 채번은 됐지만 데이터는 없는 결번 발생
     → Redis 롤백 불가능 (별개 시스템)

   [Redis 채번]                    [DB 구분자 기반 채번]
     Redis INCR (성공)               SELECT FOR UPDATE
     DB INSERT  (실패 가능)  vs      UPDATE seq +1
     → 결번, 롤백 불가              INSERT orders
                                     COMMIT (원자적 완료, 실패 시 전체 롤백)

③ 별도 인프라 관리 비용
   Redis 클러스터 관리, 메모리 튜닝, 고가용성 구성 등 운영 부담
```

**DB 구분자 기반 채번으로 전환 시 실익:**
- 채번과 INSERT를 단일 트랜잭션으로 묶어 원자성 보장
- ACID 기반 영속성 → 재시작/장애 후에도 채번 상태 안전
- Redis 인프라 제거 → 운영 단순화
- 채번 테이블에 `last_updated`, `updated_by` 컬럼 추가 → 감사(Audit) 로그 겸용 가능

---

### 8.5 ULID — UUID의 정렬 가능 대안 (보강)

UUID v4는 완전 랜덤이라 B-Tree 인덱스에서 페이지 분열(Page Split)을 유발한다. ULID는 이 문제를 해결한다.

```
ULID 구조 (128 bit)
├─ 48 bit : millisecond timestamp (Unix epoch)
└─ 80 bit : random (암호학적 난수)

예시: 01ARZ3NDEKTSV4RRFFQ69G5FAV
     ├──────── 타임스탬프 ──────┤ ← 동일 밀리초 내 사전순 정렬 보장
```

**장점:** 시간순 정렬 가능, UUID와 동일한 128-bit 길이, 락 없이 노드 독립 생성
**단점:** 동일 밀리초 내 복수 ULID는 완전한 순서 보장 불가 (랜덤 부분)

---

### 8.6 Redis Redlock — 분산 락 패턴 (보강)

```
Redlock 동작:
1. N개(보통 5개)의 독립 Redis 노드에 동시 잠금 요청
2. 과반수(N/2+1) 이상에서 잠금 성공 시 락 획득
3. 잠금 유효 시간 = TTL - (잠금 획득 소요 시간)
4. 사용 후 모든 노드에서 잠금 해제
```

> **주의:** Redlock은 GC 일시정지, 시계 왜곡 시 잠금 보장이 무너질 수 있다는 비판(Martin Kleppmann)이 있다.
> 강한 일관성이 필요하면 ZooKeeper 또는 etcd 기반 락을 사용한다.

---

### 8.7 PostgreSQL SEQUENCE (보강)

```sql
-- PostgreSQL SEQUENCE 생성
CREATE SEQUENCE order_seq
  START WITH 1 INCREMENT BY 1 CACHE 50 NO CYCLE;

-- 사용
INSERT INTO orders (order_id) VALUES (nextval('order_seq'));
SELECT currval('order_seq');
```

Oracle과의 차이: 트랜잭션 롤백 시에도 채번된 값이 소진됨 (결번 가능). RAC 환경이 없으므로 ORDER 옵션 개념 없음.

---

### 8.8 채번 테이블 — 배치 할당 전략 (보강)

```
[배치 할당 흐름]
1. 서버 시작 시: FOR UPDATE → current_seq + N 으로 UPDATE → COMMIT
   → 메모리에 (current_seq ~ current_seq+N-1) 범위 보관
2. 채번 요청마다: 메모리 카운터 증가 (DB 접근 없음)
3. 메모리 소진 시: 다시 DB에서 N개 할당

장점: DB 접근 빈도 = 요청/N으로 감소, 단일 요청당 지연 최소화
단점: 서버 재시작 시 미사용 번호 결번, 여러 인스턴스 간 전역 순서 보장 불가
```

---

### 8.9 버전별 차이 / 변경 이력

| 시스템 | 버전/설정 | 변경 내용 | 영향 |
|--------|-----------|-----------|------|
| MySQL InnoDB | mode=0 | INSERT 전체 동안 테이블 락 | 가장 느림, 가장 안전 |
| MySQL InnoDB | mode=1 (기본) | 단순 INSERT는 경량 뮤텍스 | 빠름, 일반적으로 권장 |
| MySQL InnoDB | mode=2 | 락 없이 원자 카운터 | 가장 빠름, Statement Binlog 주의 |
| Oracle SEQUENCE | CACHE N | 메모리 선점, SQ 락만 사용 | DB 재시작/롤백 시 결번 가능 |
| Oracle SEQUENCE | NOCACHE | 매 호출 디스크 변경, Row Cache 락 | 고부하 시 CPU 급등 |
| Oracle SEQUENCE | CACHE + ORDER | RAC 간 순서 보장, SV 락 추가 | 성능 가장 낮음 |
| Snowflake | 원래 설계 (2010) | 41+5+5+12 bit, **밀리초당** 4096개 | 초당 약 409만 6000개/노드 |

---

## 9. 나만의 요약 (My Summary)

```
채번은 "전역 유일성"과 "가용성"의 트레이드오프 싸움이다.

DB 기반 채번의 병목 경로:
  단일 row 락 → 직렬화 → 커넥션 점유 → 풀 고갈 → 서비스 장애

Oracle은 인덱스와 무관하게 조건 매칭 row만 잠근다 (Lock Escalation 없음).
MySQL은 인덱스가 없으면 스캔한 모든 row를 잠근다 → 인덱스 설계가 락 범위를 결정.

구분자 기반 채번의 핵심:
  채번 요청은 항상 "특정 계좌"의 맥락으로 들어온다.
  account_no는 이미 알고 있는 값 → 전체 row 스캔 불필요.
  신규 계좌 첫 채번에는 MERGE(Oracle) / UPSERT(MySQL) 필수.

Redis 채번은 빠르지만 "채번 성공 = 데이터 저장 성공"을 보장하지 못한다.
원자성이 중요한 금융/회계 도메인에서는 DB 기반 채번이 더 안전하다.
```

**기억할 포인트 3가지:**
1. Snowflake ID의 12-bit sequence = **밀리초당** 4096개 (초당 약 409만 개), 초당 4096개가 아님
2. ZooKeeper는 Paxos가 아닌 **Zab(ZooKeeper Atomic Broadcast)** 프로토콜 사용
3. MySQL FOR UPDATE는 **스캔한 row** 전체를 잠금 → 인덱스 없으면 WHERE 조건 무의미

**다음에 헷갈릴 것 같은 부분:**
- Oracle SQ 락 / Row Cache 락 / SV 락 구분 (CACHE/NOCACHE/ORDER 옵션과의 매핑)
- 신규 계좌 첫 채번 시 FOR UPDATE로는 락을 잡을 수 없어서 MERGE/UPSERT 필요한 이유
- Redis 채번과 DB 채번의 원자성 차이 → 어느 도메인에서 무엇을 선택할지

---

## 10. 참고 자료 (References)

| 유형 | 제목 | 링크 | 메모 |
|------|------|------|------|
| 블로그 | 채번 테이블을 이용한 채번 동시성 제어 | https://swpju.tistory.com/entry/채번-테이블을-이용한-채번동시성제어 | 채번 테이블 패턴 상세 |
| 블로그 | MAX+1 채번 이슈 | https://redballs.tistory.com/entry/MAX-1-채번-이슈 | VARCHAR 정렬 버그 포함 |
| 블로그 | Sequence ORDER 옵션 성능 | https://redballs.tistory.com/entry/Sequence-order-옵션에-따른-성능 | Oracle SV/SQ/Row Cache 락 |
| 공식 문서 | MySQL InnoDB Locking Reads | https://dev.mysql.com/doc/refman/8.3/en/innodb-locking-reads.html | FOR UPDATE 락 범위 |
| 블로그 | 분산 DB 환경에서 분산 락 활용 | https://velog.io/@znftm97/동시성-문제-해결하기-V3-분산-DB-환경에서-분산-락Distributed-Lock-활용 | Redis Redlock 패턴 |
| 블로그 | DB Deadlock 원인: MAX+1 채번 문제점과 대안 | https://velog.io/@swjk78/DB-Deadlock-원인-MAX-1-채번-방식의-문제점과-대안 | MAX+1 데드락 시나리오 |
| 블로그 | 전문번호 채번 동시성 이슈 해결하기 | https://velog.io/@devmizz/전문번호-채번-동시성-이슈-해결하기 | 실무 채번 이슈 사례 |
| 블로그 | Redis 분산 락 | https://f-lab.kr/insight/distributed-lock-with-redis-20250215 | Redlock 패턴 |
| 블로그 | MySQL SELECT FOR UPDATE | https://miintto.github.io/docs/mysql-select-for-update | FOR UPDATE 동작 상세 |
| Ask Tom | Oracle SELECT FOR UPDATE 동작 | https://asktom.oracle.com/ords/f?p=100:11::::: | Oracle 공식 Q&A |
| 블로그 | Oracle 자동 증분 Sequence와 MAX+1 차이 | https://thatisgood.tistory.com/entry/Oracle-자동증분-Sequence와-maxseq1의-차이 | Oracle 관점 비교 |

---

*이 문서는 학습이 깊어질수록 계속 업데이트합니다.*
