# 지식 정리 (Knowledge Note)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **주제** | 애플리케이션 멀티모듈(Multi-Module) 구조 |
| **분류** | 플랫폼 / 프레임워크 / 빌드 시스템 |
| **키워드** | 멀티모듈, Gradle, Maven, module-core, spring.config.import, ArchUnit, Fat JAR, 의존성 방향 |
| **학습 계기** | 멀티모듈 프로젝트 구조와 싱글모듈 대비 강점 탐구 |
| **관련 업무 ID** | - |
| **최초 작성일** | 2026-04-27 |
| **최종 수정일** | 2026-04-27 |
| **숙련도** | 🌿 이해 |

---

## 1. 한 줄 요약 (TL;DR)

```
멀티모듈은 하나의 빌드 시스템(Gradle/Maven) 아래에서
코드를 기능 단위로 분리해 독립적으로 빌드·배포하는 구조다.
코드 중복 제거, 의존성 방향 강제, 빌드 시간 단축이 핵심 강점이며
실행 모듈(api, admin, batch)은 각각 Fat JAR로 독립 배포된다.
```

---

## 2. 배경 지식 (Prerequisites)

- **필수 선행 지식**:
  - Gradle / Maven 빌드 시스템 기본 개념
  - Spring Boot 애플리케이션 구조 (빈, 컴포넌트 스캔, yml 로딩)
  - JAR 패키징 방식 (Fat JAR vs Thin JAR)

- **관련 개념과의 관계**:
  ```
  [루트 프로젝트 (Gradle)]
    ├── module-core      ← 공통 도메인 / 유틸 (실행 불가)
    ├── module-api       ← REST API 서버 (Fat JAR 배포)
    │     └── depends on module-core
    ├── module-admin     ← 관리자 서버 (Fat JAR 배포)
    │     └── depends on module-core
    └── module-batch     ← 배치 처리 (Fat JAR 배포)
          └── depends on module-core

  module-api.jar 내부:
    ├── module-api 클래스
    └── module-core 클래스 (통째로 포함됨)
  ```

---

## 3. 개념 설명 (Concept)

### 3.1 정의 (What)

```
멀티모듈이란 하나의 큰 애플리케이션을 여러 개의 독립적인 모듈로 나누어
개발·관리하는 프로젝트 구조다.

모듈은 패키지보다 한 단계 위의 코드 집합체로,
독립적으로 빌드 가능한 코드 단위를 의미한다.
예: 도메인(Domain), 공통 유틸(Util), API 서버, 관리자 서버, 배치 처리
```

### 3.2 존재 이유 (Why)

```
싱글모듈의 문제점:

① 배포 비효율
   배치 코드 한 줄 수정 → API, 관리자, 배치 전체 재빌드·배포

② 의존성 오염
   admin 코드가 실수로 batch 코드를 참조해도 아무 제약 없음
   → 아키텍처 규칙을 코드 레벨에서 강제할 방법 없음

③ 코드 중복
   다른 프로젝트에서 User.java를 쓰고 싶어도 파일을 직접 복사해야 함
   → 중복 수정, 동기화 누락 버그 발생
```

### 3.3 동작 원리 (How it works)

**빌드 흐름**

```
Step 1. Gradle이 settings.gradle의 include 선언으로 모든 모듈 인식
Step 2. 의존 관계 분석 → module-core 먼저 컴파일
Step 3. module-api가 module-core를 참조해서 컴파일
Step 4. module-api의 bootJar 태스크로 Fat JAR 생성
        (module-core 클래스가 module-api.jar 안에 포함됨)
Step 5. 실행 모듈(api, admin, batch)별로 독립 배포
```

**실제 의존 선언 방식**

```groovy
// module-api/build.gradle
dependencies {
    implementation(project(":module-core"))  // 로컬 모듈 직접 참조
    // 외부 maven 저장소에서 다운로드가 아닌, 동일 루트 프로젝트 내 참조
}
```

### 3.4 구성 요소 / 핵심 용어 (Key Terms)

| 용어 | 설명 | 비고 |
|------|------|------|
| 루트 프로젝트 | 모든 모듈을 관리하는 최상위 Gradle 프로젝트 | settings.gradle에 모듈 등록 |
| module-core | 공통 도메인·유틸 모음. 실행 모듈이 아님 | Fat JAR 생성 안 함 |
| Fat JAR | 실행에 필요한 모든 클래스(의존 모듈 포함)를 하나의 jar로 패키징 | 배포 단위 |
| ArchUnit | 아키텍처 의존성 규칙을 JUnit 테스트로 검증하는 라이브러리 | 의존 방향 강제 |
| 순환 의존성 | A→B→A처럼 모듈이 서로 참조하는 상태 | Gradle이 빌드 시 감지 |

---

## 4. 비교 및 구분 (Comparison)

### 4.1 싱글모듈 vs 멀티모듈

| 항목 | 싱글모듈 | 멀티모듈 |
|------|---------|---------|
| 코드 공유 | 복붙 필요 | module-core 공유 |
| 재빌드 범위 | 전체 항상 재빌드 | 변경 모듈만 재빌드 |
| 의존성 제어 | 없음 (컴파일 통과) | build.gradle 선언 강제 |
| 장애 격리 | 전체 하나 | 모듈별 독립 배포 |
| 테스트 범위 | 전체 함께 | 모듈 단위 격리 테스트 |
| 초기 설계 비용 | 낮음 | 높음 |

### 4.2 application.yml 관리 방법 비교

| 방법 | 방식 | 장점 | 단점 |
|------|------|------|------|
| `spring.config.import` ✅ | 실행 모듈 yml에서 명시적 import | 어떤 파일을 포함하는지 명확 | 파일마다 일일이 명시 |
| `spring.profiles.include` | 프로필 네이밍 컨벤션 활용 | 프로필 단위로 분리 가능 | application-{name}.yml 형식 필수, 프로필 활성화 부작용 가능 |
| `System.setProperty` | main()에서 직접 파일명 지정 | 유연함 | 코드에 설정 혼재 |

> **주의 — `spring.profiles.include` 부작용**: 이 방법은 단순히 yml 파일을 import하는 게 아니라 해당 profile을 **활성화**한다. `@Profile("rds")`로 조건부 등록된 빈이 의도치 않게 활성화될 수 있으므로, Spring Boot 2.4+ 프로젝트에서는 `spring.config.import` 사용을 권장한다.

### 4.3 언제 멀티모듈을 도입해야 하는가

```
도입 권장 상황:
  → 동일 코드를 두 개 이상의 프로젝트에서 복붙하고 있을 때
  → API / 관리자 / 배치처럼 실행 단위가 명확히 분리될 때
  → 팀이 나뉘어 담당 영역만 독립적으로 개발해야 할 때

도입 미권장 상황:
  → 초기 MVP, 소규모 팀, 단일 서버로 충분한 경우
  → 모듈 간 경계가 불명확한 설계 초기 단계
```

---

## 5. 실전 예시 (Examples)

### 5.1 기본 Gradle 멀티모듈 구조

```
my-app/
├── settings.gradle          ← 모듈 등록
├── build.gradle             ← 공통 설정 (Java 버전, 공통 의존성)
│
├── module-core/
│   ├── build.gradle         ← core 전용 의존성
│   └── src/main/java/com.example.core/
│       ├── user/User.java
│       └── order/Order.java
│
├── module-api/
│   ├── build.gradle
│   └── src/main/java/com.example.api/
│       └── UserController.java
│
└── module-batch/
    ├── build.gradle
    └── src/main/java/com.example.batch/
        └── OrderBatch.java
```

```groovy
// settings.gradle
rootProject.name = 'my-app'
include(
    'module-core',
    'module-api',
    'module-admin',
    'module-batch'
)

// module-api/build.gradle
dependencies {
    implementation(project(":module-core"))
}
```

---

### 5.2 application.yml — spring.config.import 방식 (권장)

```yaml
# module-api/src/main/resources/application.yml
spring:
  config:
    import:
      - 'classpath:core-property.yml'   # module-core의 yml
      - 'classpath:db-property.yml'     # module-db의 yml

server:
  port: 8080
```

```yaml
# module-core/src/main/resources/core-property.yml
some-core:
  timeout: 3000
  retry: 3
```

> module-core의 yml은 Spring이 자동으로 읽지 않는다. module-core.jar가 module-api.jar에 포함되어 classpath에 올라가면, module-api의 yml에서 `spring.config.import`로 명시해야 읽힌다.

---

### 5.3 주의해야 할 패턴 (Anti-pattern)

```groovy
// ❌ 순환 의존성
// module-api/build.gradle
dependencies {
    implementation(project(":module-batch"))  // api → batch
}

// module-batch/build.gradle
dependencies {
    implementation(project(":module-api"))    // batch → api (순환!)
}
// → Gradle 빌드 시 "Circular dependency" 에러 발생
```

```groovy
// ✅ 단방향 의존성
// module-api/build.gradle
dependencies {
    implementation(project(":module-core"))   // api → core (단방향)
}

// module-batch/build.gradle
dependencies {
    implementation(project(":module-core"))   // batch → core (단방향)
}
// api와 batch는 서로 모름. core만 공유
```

---

## 6. 트러블슈팅 경험 (Troubleshooting)

### 문제 1. module-core의 yml이 읽히지 않음

- **증상**: module-core에 설정한 값이 애플리케이션 실행 시 null 또는 기본값으로 나옴.
- **원인**: Spring Boot는 실행 모듈의 application.yml만 자동으로 읽는다. module-core는 실행 주체가 아니므로 그 yml은 자동 로드되지 않음.
- **해결**: 실행 모듈(module-api)의 application.yml에 `spring.config.import`로 명시 추가.
- **교훈**: 멀티모듈에서 yml은 "어디에 파일이 있느냐"가 아니라 "어떤 모듈이 실행 주체냐"가 기준이다.

### 문제 2. 의도치 않은 모듈 간 참조가 컴파일 통과됨

- **증상**: module-batch에서 module-api의 Service를 import해도 컴파일 에러 없음.
- **원인**: module-batch의 build.gradle에 `implementation(project(":module-api"))`가 실수로 선언되어 있었음.
- **해결**: 의존성 선언 제거. ArchUnit 도입하여 의존 방향 규칙을 테스트 레벨에서 검증.
- **교훈**: build.gradle의 dependencies 블록이 아키텍처 경계이다. 의도하지 않은 의존성이 생기지 않도록 ArchUnit으로 CI에서 자동 검증하는 것이 좋다.

---

## 7. 실전 명령어 / 치트시트 (Cheat Sheet)

```bash
# 전체 빌드
./gradlew build

# 특정 모듈만 빌드
./gradlew :module-api:build

# 특정 모듈 테스트만 실행
./gradlew :module-core:test

# 특정 모듈 실행 가능 JAR 생성
./gradlew :module-api:bootJar

# 의존성 트리 확인
./gradlew :module-api:dependencies

# 모듈 간 의존 관계 확인 (태스크)
./gradlew projects

# 빌드 캐시 정리
./gradlew clean

# 특정 모듈 실행
./gradlew :module-api:bootRun
```

---

## 8. 깊게 이해하기 (Deep Dive)

### 8.1 Fat JAR 내부 구조

```
module-api.jar (Fat JAR)
├── BOOT-INF/
│   ├── classes/              ← module-api 컴파일 결과
│   │   └── com/example/api/UserController.class
│   └── lib/
│       ├── module-core-1.0.jar   ← module-core 통째로 포함
│       ├── spring-boot-3.x.jar
│       └── ...
└── META-INF/
    └── MANIFEST.MF           ← 메인 클래스 지정
```

module-core는 별도 실행 불가 JAR로 lib/ 안에 포함된다. 배포 단위는 module-api.jar 하나.

---

### 8.2 모듈 레이어링 패턴 (보강)

규모가 커질수록 모듈을 계층으로 나누는 패턴을 사용한다.

```
[Layered Multi-Module]

module-common    ← 유틸, 공통 상수, 예외 클래스 (의존성 없음)
     ↑
module-domain    ← Entity, Repository 인터페이스, Value Object
     ↑
module-application ← UseCase, Service (domain에만 의존)
     ↑
module-infrastructure ← JPA 구현체, 외부 API 클라이언트, Redis
     ↑
module-api / module-admin / module-batch  ← 실행 진입점

규칙: 아래 레이어는 위 레이어를 참조할 수 없음 (단방향)
```

이 패턴은 헥사고날 아키텍처(Ports & Adapters)와 함께 사용할 때 강력하다.

---

### 8.3 ArchUnit으로 의존성 규칙 테스트 강제화 (보강)

build.gradle 선언만으로는 충분하지 않다. 개발자가 실수로 잘못된 import를 쓸 수 있기 때문에, ArchUnit으로 CI 단계에서 아키텍처 규칙을 테스트한다.

```groovy
// module-core/build.gradle
testImplementation 'com.tngtech.archunit:archunit-junit5:1.2.1'
```

```java
// ArchitectureTest.java (module-core 테스트)
@AnalyzeClasses(packages = "com.example")
class ArchitectureTest {

    @ArchTest
    static final ArchRule domainShouldNotDependOnApplication =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("..application..");

    @ArchTest
    static final ArchRule coreShouldNotDependOnApi =
        noClasses()
            .that().resideInAPackage("..core..")
            .should().dependOnClassesThat()
            .resideInAPackage("..api..");
}
```

이 테스트가 CI에 포함되면 잘못된 의존 방향이 생기는 순간 빌드가 실패한다.

---

### 8.4 Gradle 빌드 캐시와 점진적 빌드 (보강)

멀티모듈에서 빌드 시간이 단축되는 이유는 두 가지다.

```
① 점진적 빌드 (Incremental Build)
   변경된 모듈과 그 모듈에 의존하는 모듈만 재빌드.
   module-core 변경 → module-api, module-admin, module-batch 재빌드
   module-api 변경  → module-api만 재빌드 (core, admin, batch 건드리지 않음)

② Gradle 빌드 캐시
   이전 빌드 결과를 캐시에 저장.
   동일한 입력이면 캐시된 결과 재사용 → 빌드 시간 수 배 단축

빌드 캐시 활성화 (gradle.properties):
  org.gradle.caching=true
  org.gradle.parallel=true
```

---

### 8.5 멀티모듈 vs MSA 비교 (보강)

```
멀티모듈:
  └── 하나의 루트 프로젝트, 하나의 Git 저장소 (모노레포)
  └── 모듈 간 통신 = 직접 메서드 호출 (같은 JVM)
  └── 배포 단위 = 각 모듈의 Fat JAR
  └── 모놀리스 → MSA 전환의 중간 단계로 활용 가능

MSA:
  └── 각 서비스가 완전히 독립된 저장소·프로세스·DB
  └── 서비스 간 통신 = REST, gRPC, 메시지 큐
  └── 배포 단위 = 독립 컨테이너/인스턴스
  └── 운영 복잡도 훨씬 높음

멀티모듈은 "한 JVM 안에서의 MSA스러운 구조"로 볼 수 있다.
모듈 간 의존성이 명확히 분리되어 있으면 나중에 MSA로 전환할 때
각 모듈을 별도 서비스로 분리하기 용이하다.
```

---

## 9. 나만의 요약 (My Summary)

```
멀티모듈의 핵심은 세 가지다.

1. module-core는 독립 JAR가 아니다
   실행 모듈(api, admin, batch)의 Fat JAR 안에 통째로 포함되어 패키징된다.
   배포 단위는 실행 모듈별 Fat JAR.

2. 의존성 선언 = 아키텍처 경계
   build.gradle의 implementation(project(":module-core"))가
   "이 모듈은 core만 알 수 있다"는 아키텍처 규칙이 된다.
   ArchUnit으로 이 규칙을 CI에서 자동 검증하면 더 강력하다.

3. yml은 실행 주체가 읽는다
   module-core의 yml은 자동으로 읽히지 않는다.
   실행 모듈에서 spring.config.import로 명시해야 한다.
```

**기억할 포인트 3가지:**
1. module-core는 독립 배포 단위가 아님 → 실행 모듈 Fat JAR에 포함
2. `spring.profiles.include`는 yml import가 아닌 profile 활성화 → 부작용 주의
3. 의존성은 항상 단방향(하위 → 상위) → 순환 참조는 Gradle이 빌드 시 감지

**다음에 헷갈릴 것 같은 부분:**
- module-core의 resources/ 하위 yml이 왜 자동으로 읽히지 않는지 (실행 주체 아니므로)
- `spring.profiles.include` 사용 시 `@Profile` 빈이 의도치 않게 활성화되는 경우
- Gradle include vs includeFlat 차이 (include는 디렉토리 내부, includeFlat은 같은 레벨)

---

## 10. 참고 자료 (References)

| 유형 | 제목 | 링크 | 메모 |
|------|------|------|------|
| 블로그 | 멀티모듈 프로젝트 왜, 어떻게 해야 할까 | https://techblog.gccompany.co.kr/멀티모듈-프로젝트-왜-그리고-어떻게-해야-할까-15be2b6733b8 | 실무 적용 사례 |
| 블로그 | 우아한 멀티모듈 | https://hyeon9mak.github.io/woowahan-multi-module/ | 우아한형제들 실무 패턴 |
| 블로그 | Spring Boot 멀티모듈 환경 변수 구성 | https://tech.kakaopay.com/post/spring-multi-module-environment-variable/ | 카카오페이 실무 yml 관리 |
| 블로그 | 왜 멀티모듈 프로젝트를 사용할까 | https://hudi.blog/why-use-multi-module/ | 도입 판단 기준 |
| GitHub | ihoneymon multi-module | https://github.com/ihoneymon/multi-module | 레퍼런스 구조 |
| 블로그 | Spring Boot 멀티모듈 아키텍처 | https://f-lab.kr/insight/spring-boot-multi-module-architecture-20260319 | 레이어링 패턴 |

---

*이 문서는 학습이 깊어질수록 계속 업데이트합니다.*
