# 지식 정리 (Knowledge Note)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **주제** | K8s 환경 Observability 스택 — 로그(Loki) · 메트릭(Mimir) · 트레이스(Tempo) 수집 구조 |
| **분류** | 인프라 / Observability / Kubernetes |
| **키워드** | Alloy, Loki distributed, Tempo distributed, Mimir distributed, OTLP, Zipkin, Istio, istiod, xDS, OTel Java Agent, Grafana, B3 헤더, TraceID, 바이트코드 계측 |
| **학습 계기** | K8s 환경에서 Alloy가 수집한 로그·메트릭·트레이스가 각 백엔드에 어떻게 도달하는지 전체 파이프라인 파악 |
| **관련 업무 ID** | - |
| **최초 작성일** | 2026-05-07 |
| **최종 수정일** | 2026-05-07 |
| **숙련도** | 🌿 이해 |

---

## 1. 한 줄 요약 (TL;DR)

```
K8s 관측가능성 스택은 로그(Alloy → Loki) · 메트릭(OTel Java Agent → Alloy OTLP LB → Mimir) ·
트레이스(Envoy Zipkin + OTel Java Agent → Alloy OTLP LB → Alloy OTLP Collector → Tempo)
세 파이프라인이 각각 독립적으로 흐르되, 최종적으로 Grafana 한 곳에서 TraceID 기반 연동 조회된다.
alloy-otlp-loadbalancer가 ① Zipkin → OTLP 변환 ② Trace는 TraceID 해시 라우팅으로 Collector 전달
③ Metric은 Mimir로 직접 전송 — 세 역할을 한 컴포넌트에서 수행하는 것이 핵심 설계 포인트다.
```

---

## 2. 사전 지식 (Prerequisites)

- Kubernetes Pod, DaemonSet, Deployment, PersistentVolume 개념
- Helm 차트 기본 구조 (values.yaml)
- HTTP/gRPC 프로토콜 기본 이해
- Istio Service Mesh (사이드카 패턴) 개념
- JVM 클래스 로딩 기본 이해

---

## 3. 개념 설명 (Concept)

### 3.1 전체 아키텍처 개요

K8s 환경의 관측가능성 스택은 데이터 종류에 따라 세 파이프라인으로 분리된다.

| 데이터 종류 | 수집 에이전트 | 중계 | 저장 백엔드 | 조회 |
|---|---|---|---|---|
| **Log** | Alloy (DaemonSet) | — | Loki Distributed | Grafana (LogQL) |
| **Metric** | OTel Java Agent | Alloy OTLP LB (직접 전송) | Mimir Distributed | Grafana (PromQL) |
| **Trace** | OTel Java Agent + Envoy Sidecar | Alloy OTLP LB → Collector | Tempo Distributed | Grafana (TraceQL) |

### 3.2 Loki Distributed 컴포넌트 역할

Loki Distributed 모드는 컴포넌트별 역할을 명확히 분리한 마이크로서비스 아키텍처다.

**Gateway (게이트웨이)**
- 클러스터 외부 트래픽의 단일 진입점 (Reverse Proxy, 내부적으로 Nginx 사용)
- 쓰기 요청(`/loki/api/v1/push`) → Distributor로 라우팅
- 읽기 요청(`/loki/api/v1/query`) → Query Frontend로 라우팅
- 상태 없음(stateless)

```nginx
# gateway 내부 nginx.conf 예시
location = /loki/api/v1/push {
    proxy_pass http://loki-distributed-distributor:3100$request_uri;
}
```

**Distributor (분산기)**
- Loki **내부 로직**의 첫 번째 처리 컴포넌트 (외부 진입점은 Gateway)
- 수신 로그의 유효성 검증 (타임스탬프, 라벨 형식 등)
- 일관된 해시 링(Consistent Hash Ring)으로 Ingester 선택 후 전달
- 로그를 저장하지 않는 stateless 컴포넌트

> **Gateway vs Distributor 진입점 혼동 주의**
> Loki 공식 문서에서 Distributor를 "첫 번째 컴포넌트"라 표현하는 이유는
> **Loki 내부 로직 관점**에서의 표현이다.
> - 외부(네트워크) 관점 → **Gateway가 진입점**
> - Loki 내부 로직 관점 → **Distributor가 첫 번째 처리 컴포넌트**

**Querier (쿼리 실행기)**
- LogQL 쿼리를 실제로 실행
- flush되지 않은 최신 로그 → Ingester에서 조회
- 오래된 로그 → 오브젝트 스토리지(S3 등)에서 직접 조회
- 두 결과 병합 + 복제본 중복 제거 후 반환

**Query Frontend (쿼리 프론트엔드)**
- 대용량 쿼리를 시간 단위로 샤딩하여 여러 Querier에 병렬 분산
- 결과 캐싱 처리
- stateless 컴포넌트

**Query Scheduler (쿼리 스케줄러)**
- Query Frontend와 Querier 사이 쿼리 작업 큐(Queue) 관리
- 쿼리 요청의 공정한 분배 및 부하 조절
- Query Frontend에서 분리된 선택적 컴포넌트, 데이터 저장 없음

**실제 저장 담당 컴포넌트 (위 5개에는 없음)**
- **Ingester**: Distributor로부터 받은 로그를 메모리(WAL)에 버퍼링 → 일정 주기에 오브젝트 스토리지로 청크(Chunk) 단위 압축·저장
- **Compactor**: 오브젝트 스토리지에 저장된 청크를 주기적으로 최적화·병합

### 3.3 Loki 물리적 저장 구조

Loki 파드 내부 데이터 경로:

| 디렉토리 | 내용 |
|---|---|
| `/var/lib/loki/chunks` | 실제 로그 내용이 압축된 청크 파일 |
| `/var/lib/loki/index` | 로그 검색용 인덱스 (boltdb-shipper 방식) |
| `/var/lib/loki/wal` | 유실 방지용 임시 버퍼 (Write Ahead Log) |

프로덕션에서는 PV 대신 S3 같은 오브젝트 스토리지를 권장한다.
PV는 파드가 특정 노드에 묶이는 문제(node affinity)와 확장성 한계가 있기 때문이다.
`loki-distributed` Helm 차트 기준으로 Ingester 파드만 PV(WAL용)가 필요하고, 나머지 컴포넌트는 PV 불필요.

### 3.4 Loki 배포 모드 (같은 이미지, 다른 실행 모드)

Single Binary든 Distributed든 **동일한 `grafana/loki` 도커 이미지**를 사용한다.
`-target` 플래그(또는 `deploymentMode` 설정값)로 어떤 컴포넌트만 실행할지 결정하는 구조다.

```bash
# Monolithic (모든 컴포넌트 동시 실행)
grafana/loki -target=all

# Distributed - 컴포넌트별 분리 실행
grafana/loki -target=distributor
grafana/loki -target=ingester
grafana/loki -target=querier
grafana/loki -target=query-frontend
grafana/loki -target=query-scheduler
```

| 배포 모드 | 권장 환경 | 특징 |
|---|---|---|
| Monolithic | 소규모, 개발/테스트 | 모든 컴포넌트가 단일 파드 |
| Simple Scalable | 프로덕션 권장 (rough guide: 수 TB/day 이하) | 쓰기/읽기/백엔드 계층으로 분리 |
| Microservices | 대규모, 세밀한 스케일링 필요 | 컴포넌트별 독립 배포·스케일링 |

### 3.5 OTLP와 Zipkin의 역할 구분

| 구분 | OTLP | Zipkin |
|---|---|---|
| **종류** | 데이터 전송 **프로토콜** | 분산 추적 **백엔드 시스템** |
| **개발** | OpenTelemetry 프로젝트 | Twitter 개발 → 오픈소스 |
| **전송 방식** | gRPC(4317) / HTTP/JSON(4318) | HTTP/JSON (/api/v2/spans) |
| **데이터 종류** | Traces + Metrics + Logs | Traces만 처리 |
| **특징** | 벤더 중립, 어떤 백엔드에도 전송 가능 | Traces 시각화 UI 제공 |
| **비유** | 택배 표준 규격 박스 | 택배 추적 시스템 화면 |

OTLP와 Zipkin은 대립 관계가 아니라 함께 사용 가능하다.
OTel Collector가 OTLP로 수신 → Zipkin exporter로 내보내는 구성이 가능하다.

> **2026년 기준 신규 구축 시 참고**
> Zipkin 단독 사용보다 **Jaeger v2 + OTLP** 조합이 권장되는 추세.
> Jaeger v2는 OpenTelemetry Collector 위에 구축되어 OTLP 네이티브 지원.

### 3.6 istiod의 전체 역할

istiod는 v1.5 이후 Pilot + Citadel + Galley 세 컴포넌트가 하나로 통합된 컨트롤 플레인이다.
**사이드카 주입은 istiod 역할의 일부일 뿐**이며, 더 많은 역할을 수행한다.

| 역할 | 설명 | 원래 담당 컴포넌트 |
|---|---|---|
| 사이드카 주입 | MutatingAdmissionWebhook으로 Pod 스펙에 istio-init + istio-proxy 추가 | - |
| xDS 설정 배포 | Envoy와 gRPC 연결 유지, LDS/RDS/CDS/EDS 및 meshConfig를 실시간 push | Pilot |
| 서비스 디스커버리 | K8s Service/Endpoint 변경 Watch → EDS로 업스트림 목록 자동 갱신 | Pilot |
| 인증서 관리(mTLS) | SPIFFE 기반 ID 부여, mTLS 인증서 발급·갱신 | Citadel |
| 설정 유효성 검사 | VirtualService, DestinationRule 등 CRD 문법 오류 사전 차단 | Galley |

```
istiod (meshConfig 보관)
    ↓ xDS 프로토콜로 설정 배포
Envoy Sidecar (설정 수신 후 부팅)
    ↓ 트래픽 처리 중 Span 생성
alloy-otlp-collector:9411 (Zipkin 엔드포인트로 직접 전송)
```

> **핵심**: 사이드카 주입은 파드 생성 시 단 한 번 일어나지만,
> xDS를 통한 설정 동기화는 Envoy와 istiod가 gRPC 연결을 유지하며 상시 이루어진다.
> 그래서 파드 재시작 없이 VirtualService를 변경하면 즉시 트래픽 라우팅이 바뀐다.

**istiod에서 meshConfig를 설정하는 이유**

Envoy가 실제로 Span을 전송하지만 설정은 istiod에서 한다.
파드가 뜰 때 Envoy 사이드카가 istiod에 접속해 meshConfig.defaultConfig를 포함한 전체 설정을 xDS 프로토콜로 수신하기 때문이다.

- istiod = "운전 매뉴얼을 나눠주는 본사"
- Envoy = "매뉴얼대로 운전하는 기사"

**네임스페이스별 Helm 차트 분리 이유**

| 이유 | 설명 |
|---|---|
| Istio 차트 구조 | Gateway는 네임스페이스별 독립 배포 권장 (팀별 자체 관리) |
| 운영 격리 | 네임스페이스별 독립 배포·롤백, RBAC 권한 분리 |
| meshConfig 적용 범위 | istiod meshConfig는 클러스터 전체 적용 (파드 annotation으로 오버라이드 가능) |

```yaml
# 특정 파드만 다른 Zipkin 주소 사용 (네임스페이스 오버라이드)
annotations:
  proxy.istio.io/config: |
    tracing:
      zipkin:
        address: other-collector:9411
```

### 3.7 OTel Java Agent 바이트코드 계측 원리

`-javaagent` 옵션 하나만 추가하면 소스코드 수정 없이 자동 계측이 가능하다.
원리는 JVM의 **바이트코드 조작(Bytecode Instrumentation)**이다.

```
JVM 클래스 로딩
    ↓
Java Agent의 ClassFileTransformer 개입
    ↓
바이트코드에 계측 코드 삽입 (메모리 내 변환, 원본 .class 불변)
    ↓
수정된 클래스가 실행됨
```

OTel Java Agent가 자동 계측하는 대상:

| 계측 대상 | 수집 내용 |
|---|---|
| Spring MVC / WebFlux | HTTP 요청 서버 Span |
| RestTemplate / WebClient / OkHttp | 외부 API 호출 클라이언트 Span |
| gRPC | 서버/클라이언트 Span |
| JDBC | SQL 쿼리 DB Span (쿼리문 포함) |
| Kafka / RabbitMQ | 메시지 발행/소비 Span |
| JVM 런타임 | Heap, GC, Thread 메트릭 |

메트릭은 두 가지 경로로 수집된다:
- **JVM 자동 계측 메트릭**: Heap 사용량, GC, Thread 수 등 → OTLP로 전송
- **Micrometer 연동**: Spring Boot Actuator 메트릭을 OTel로 브리징

> **Micrometer 중복 수집 주의**: Spring Boot 앱에서 Micrometer(Actuator)도 함께 사용 중이면
> HTTP 메트릭이 두 곳에서 중복 수집될 수 있다.
> 이때는 Java Agent에서 메트릭을 비활성화하거나 수집 경로를 명확히 분리해야 한다.
> ```bash
> # Trace만 수집, 메트릭/로그는 비활성화
> -Dotel.metrics.exporter=none
> -Dotel.logs.exporter=none
> ```

**주입 방법 세 가지**

```dockerfile
# 방법 1: Dockerfile에 직접 추가
# 엔드포인트는 alloy-otlp-loadbalancer (Trace/Metric 분기를 loadbalancer가 처리)
ENTRYPOINT ["java", \
  "-javaagent:/otel/opentelemetry-javaagent.jar", \
  "-Dotel.service.name=my-service", \
  "-Dotel.exporter.otlp.endpoint=http://alloy-otlp-loadbalancer:4317", \
  "-jar", "/app.jar"]
```

```yaml
# 방법 2: K8s 환경변수로 주입 (이미지 수정 없이)
env:
  - name: JAVA_TOOL_OPTIONS
    value: "-javaagent:/otel/opentelemetry-javaagent.jar"
  - name: OTEL_SERVICE_NAME
    value: "my-service"
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://alloy-otlp-loadbalancer:4317"
```

```yaml
# 방법 3: OTel Operator로 자동 주입 (K8s 권장)
# 네임스페이스나 파드에 어노테이션만 달면 끝
annotations:
  instrumentation.opentelemetry.io/inject-java: "true"
```

### 3.8 Envoy Span 생성 원리와 B3 헤더 전파

**TraceID 생성 시점**

외부 요청이 Istio Ingress Gateway에 처음 도착할 때, 기존 트레이싱 헤더(`x-b3-traceid`, `traceparent`)가 없으면 Envoy가 새로운 128bit TraceID를 랜덤 생성한다. 이후 내부 서비스 간 호출에서는 이 TraceID를 그대로 전파한다.

**Span 구성 요소**

```
Span 구성 요소:
  - TraceID:      최초 생성된 TraceID (전체 요청 흐름에서 동일)
  - SpanID:       이 Envoy가 새로 생성하는 고유 ID
  - ParentSpanID: 업스트림에서 전달받은 SpanID (부모-자식 계층 구조)
  - 시작/종료 타임스탬프
  - HTTP 메서드, URL, 상태코드 등
```

**B3 헤더로 컨텍스트 전파**

```
X-B3-TraceId:     <동일한 TraceID>
X-B3-SpanId:      <이 Envoy의 SpanID>
X-B3-ParentSpanId: <이전 서비스의 SpanID>
X-B3-Sampled:     1
```

> **⚠️ 중요 제약: 앱이 헤더를 직접 전달해야 한다**
> Envoy는 인바운드/아웃바운드 트래픽을 각각 따로 본다.
> 서비스 A → 서비스 B 호출 시 Trace가 하나로 이어지려면,
> 애플리케이션 코드가 수신한 B3 헤더를 다음 호출에 그대로 전달(propagate)해야 한다.
> OTel Java Agent를 함께 쓰면 이 헤더 전파를 자동으로 처리해준다.

```
[Service A Envoy 수신] → B3 헤더 생성
        ↓
[Service A App] → B3 헤더를 그대로 Service B 호출에 포함
                  (OTel Java Agent가 자동 처리)
        ↓
[Service A Envoy 송신] → B3 헤더 포함해서 전송
        ↓
[Service B Envoy 수신] → 같은 TraceID로 새 Span 생성
```

### 3.9 alloy-otlp-loadbalancer: Zipkin → OTLP 변환 + TraceID 라우팅

**설정 구조 분석**

```hcl
# 1단계: Zipkin 프로토콜로 수신
otelcol.receiver.zipkin "istio" {
  endpoint = "0.0.0.0:9411"
  output {
    traces = [otelcol.exporter.loadbalancing.istio.input]
  }
}

# 2단계: OTLP로 변환해서 alloy-otlp-collector로 로드밸런싱
otelcol.exporter.loadbalancing "istio" {
  resolver {
    kubernetes {
      service = "alloy-otlp-collector"
      ports   = [9411]
    }
  }
  protocol {
    otlp { client { tls { insecure = true } } }
  }
}
```

**처리 흐름**

```
Istio Envoy Sidecar
    ↓ Zipkin 프로토콜 (HTTP POST /api/v2/spans, 포트 9411)
alloy-otlp-loadbalancer
    [otelcol.receiver.zipkin]
    → Zipkin JSON 포맷을 내부적으로 OTLP 데이터 모델(pdata)로 자동 변환
    ↓ OTLP gRPC (TraceID 기반 해시 라우팅)
alloy-otlp-collector (여러 파드 중 하나, 동일 TraceID는 항상 동일 파드)
    [otelcol.receiver.otlp]
    → 처리 후 Tempo Distributor로 전송
```

**TraceID 해시 라우팅이 필요한 이유**

단순 라운드로빈이 아닌 TraceID 해시 기반 라우팅을 쓰는 이유:

```
TraceID: abc123
  → hash(abc123) % N(파드 수) = 2번 파드로 고정 라우팅
```

같은 TraceID를 가진 모든 Span이 항상 동일한 Collector 파드로 모여야
Tail Sampling(수집 완료 후 샘플링 결정)이나 Span 조립이 정확하게 동작한다.

**kubernetes resolver를 쓰는 이유**

`kubernetes` resolver는 Service의 Endpoint(파드 IP 목록)를 직접 Watch한다.
ClusterIP를 거치면 kube-proxy가 임의로 분산시켜 버려서 일관된 TraceID 해시 라우팅이 불가능하다.

**alloy-otlp-loadbalancer의 세 가지 역할 요약**

1. **프로토콜 변환기**: Zipkin HTTP/JSON → OTLP gRPC (수신 즉시 자동 변환)
2. **TraceID 해시 로드밸런서**: 동일 TraceID는 항상 동일 Collector 파드로 라우팅 (Trace 전용)
3. **Metric 직접 전달**: 메트릭은 alloy-otlp-collector를 거치지 않고 mimir-distributed로 직접 전송

### 3.10 Istio에서 OTLP로 전환하는 방법

Istio 1.16+ 부터 OTLP 트레이서를 공식 지원한다.

**방법 1: meshConfig에서 openCensusAgent 방식 (레거시)**

```yaml
# istiod values.yaml
meshConfig:
  defaultConfig:
    tracing:
      openCensusAgent:              # Istio 내부 네이밍 quirk: 실제로는 OTLP gRPC 엔드포인트
        address: "alloy-otlp-collector:4317"
        context:
          - W3C_TRACE_CONTEXT       # traceparent/tracestate 헤더 사용
```

**방법 2: Telemetry API (Istio 1.22+ 권장)**

```yaml
# extension provider 등록 (meshConfig)
meshConfig:
  extensionProviders:
  - name: otel-tracing
    opentelemetry:
      service: alloy-otlp-collector.monitoring.svc.cluster.local
      port: 4317

---
# Telemetry 리소스로 적용
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: otel-tracing
  namespace: istio-system    # 전역 적용
spec:
  tracing:
  - providers:
    - name: otel-tracing
    randomSamplingPercentage: 100
```

| 비교 항목 | Zipkin | OTLP |
|---|---|---|
| Istio 지원 | 오래된 안정적 지원 | 1.16+ 공식 지원 |
| 헤더 형식 | B3 헤더 (`x-b3-traceid`) | W3C TraceContext (`traceparent`) |
| 변환 필요 | alloy-otlp-loadbalancer가 Zipkin→OTLP 변환 필요 | 불필요 (Java Agent와 동일 헤더 사용) |
| 통합 자연스러움 | 별도 헤더 변환 필요 | Envoy + Java Agent Trace가 자연스럽게 연결 |

---

## 4. 비교 분석 (Comparison)

### 4.1 Alloy 역할 비교

| 컴포넌트 | 배포 방식 | 역할 |
|---|---|---|
| alloy (로그 전용) | DaemonSet | 노드 `/var/log/pods/` 파일 테일링 → Loki push |
| alloy-otlp-loadbalancer | DaemonSet or Deployment | ① Zipkin 수신·OTLP 변환 ② Trace: TraceID 해시 라우팅 → Collector ③ Metric: Mimir로 직접 전송 |
| alloy-otlp-collector | Deployment (다중 파드) | OTLP 수신 → **Tempo로만 전송** (Trace 전용 컴포넌트) |

### 4.2 Loki 컴포넌트 저장 여부

| 컴포넌트 | 저장 여부 | stateful |
|---|---|---|
| Gateway | ✗ | stateless |
| Distributor | ✗ | stateless |
| Querier | ✗ | stateless |
| Query Frontend | ✗ | stateless |
| Query Scheduler | ✗ | stateless |
| **Ingester** | ✓ (WAL/PV) | **stateful** |
| **Compactor** | ✓ (최적화) | **stateful** |

### 4.3 Tempo vs Mimir 구조 비교

두 시스템 모두 Loki Distributed와 동일한 구조(Distributor → Ingester → 오브젝트 스토리지)를 따른다.

| 항목 | Tempo Distributed | Mimir Distributed |
|---|---|---|
| 데이터 종류 | Traces | Metrics |
| 수신 프로토콜 | OTLP gRPC | OTLP / Prometheus Remote Write |
| 조회 언어 | TraceQL | PromQL |
| 주요 조회 | TraceID로 서비스 간 호출 경로 시각화 | JVM 메트릭, HTTP 응답시간, 에러율 |

---

## 5. 실전 예시 (Examples)

### 5.1 전체 관측가능성 스택 통합 다이어그램

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║                              APPLICATION POD                                         ║
║                                                                                      ║
║  ┌─────────────────────────┐        ┌──────────────────────────────────────────┐    ║
║  │   Main Container (App)  │        │         Istio-proxy (Envoy Sidecar)      │    ║
║  │                         │        │                                          │    ║
║  │  OTel Java Agent (JVM   │        │  - 모든 인/아웃바운드 트래픽 가로채기     │    ║
║  │  바이트코드 자동 계측)    │        │  - Span 생성 (TraceID, SpanID, B3헤더)  │    ║
║  │  ├── Traces (OTLP)      │        │  - istiod로부터 xDS로 설정 수신          │    ║
║  │  └── Metrics (OTLP)     │        │  └── Traces (Zipkin HTTP /api/v2/spans)  │    ║
║  └────────────┬────────────┘        └───────────────────────┬──────────────────┘    ║
╚═══════════════╪═════════════════════════════════════════════╪════════════════════════╝
                │ OTLP gRPC (Traces + Metrics)                │ Zipkin HTTP (9411)
                │                                             │
                ▼                                             ▼
╔══════════════════════════════════════════════════════════════════════════════════════╗
║                    alloy-otlp-loadbalancer                                           ║
║                                                                                      ║
║  otelcol.receiver.otlp "app"          otelcol.receiver.zipkin "istio"                ║
║  └── OTLP gRPC/HTTP 수신              └── Zipkin HTTP 수신 (9411)                    ║
║       ├── Traces ─────────────────────────── ↓ 내부 OTLP 자동 변환                   ║
║       │                                                                              ║
║       │                              otelcol.exporter.loadbalancing "istio"          ║
║       │                              └── TraceID 해시 기반 라우팅                     ║
║       │                                  (kubernetes resolver → 파드 IP 직접)        ║
║       │   Traces: TraceID 해시 → 동일 Collector 파드 고정 라우팅 ──────────────────┐  ║
║       │                                                                             │  ║
║       └── Metrics: mimir-distributed 로 직접 전송 ──────────────────────────────┐  │  ║
╚════════════════════════════════════════════════════════════════════════════════╪═╪══╝
                                                                                │ │ OTLP gRPC (Traces only)
                                                   OTLP / Prometheus RW (Metrics)│ │
                                                                                │ ▼
                                                                                │ ╔═══════════════════════════════╗
                                                                                │ ║  alloy-otlp-collector          ║
                                                                                │ ║  (Deployment, 다중 파드)        ║
                                                                                │ ║                               ║
                                                                                │ ║  otelcol.receiver.otlp        ║
                                                                                │ ║  └── Trace 전용 수신           ║
                                                                                │ ║      라벨 추가, 배치 처리      ║
                                                                                │ ║      ↓ OTLP gRPC              ║
                                                                                │ ╚══════════════╤════════════════╝
                                                                                │                │ OTLP gRPC (Traces)
                                                                                ▼                ▼
╔══════════════════════════════╗                               ╔═══════════════════════════════╗
║    mimir-distributed          ║                               ║    tempo-distributed           ║
║                               ║                               ║                               ║
║  Distributor → Ingester       ║                               ║  Distributor → Ingester       ║
║       ↓ flush                 ║                               ║       ↓ flush                 ║
║  Object Storage (Metrics)     ║                               ║  Object Storage (Traces)      ║
║                               ║                               ║                               ║
║  조회: QFrontend → Querier    ║                               ║  조회: QFrontend → Querier    ║
╚═══════════════╤══════════════╝                               ╚══════════════╤════════════════╝
                │ PromQL                                                        │ TraceQL
                └───────────────────────────┬───────────────────────────────────┘
                                            │
╔══════════════════════════════════════════════════════════════════════════════════════╗
║                  로그 수집 파이프라인 (별도)                                           ║
║                                                                                      ║
║  [Application Pod stdout/stderr]                                                     ║
║        ↓ 노드 /var/log/pods/ 파일                                                    ║
║  alloy (DaemonSet, 로그 수집 전용)                                                    ║
║  └── 파일 테일링 → 라벨 추가 (namespace, pod, container)                              ║
║        ↓ HTTP Push (/loki/api/v1/push)                                               ║
║  loki-distributed-gateway (Nginx, 단일 진입점)                                       ║
║        ↓ URL 라우팅 (/push → Distributor)                                            ║
║  loki-distributed-distributor → loki-distributed-ingester → Object Storage           ║
║        (검증·분배)                   (버퍼링·flush)           (로그 Chunks 저장)       ║
║                                                                                      ║
║  조회: Query-frontend → Query-scheduler → Querier → Ingester / Object Storage        ║
╚══════════════════════════════════════════════════════════════════════════════════════╝
                │ LogQL
                ▼
╔══════════════════════════════════════════════════════════════════════════════════════╗
║                                    GRAFANA                                           ║
║                                                                                      ║
║  Data Sources:                                                                       ║
║  ├── Loki   (LogQL)   → loki-distributed-gateway   → 로그 조회                       ║
║  ├── Mimir  (PromQL)  → mimir-distributed-gateway  → 메트릭 조회                    ║
║  └── Tempo  (TraceQL) → tempo-distributed-gateway  → 트레이스 조회                  ║
║                                                                                      ║
║  [Trace → Logs 연동]    TraceID 클릭 → Loki에서 동일 TraceID 로그 연결               ║
║  [Trace → Metrics 연동] Span 클릭   → 해당 시간대 Mimir 메트릭 연결                  ║
╚══════════════════════════════════════════════════════════════════════════════════════╝
```

### 5.2 Istio meshConfig 설정 예시

```yaml
# istiod Helm values.yaml
meshConfig:
  defaultConfig:
    tracing:
      zipkin:
        address: alloy-otlp-collector:9411  # Zipkin 수신 포트
```

### 5.3 istiod 사이드카 주입 흐름

```
kubectl apply (Pod 생성 요청)
    ↓
API Server → MutatingAdmissionWebhook 호출 → istiod
    ↓
istiod가 istio-init + istio-proxy 추가한 수정된 Pod 스펙 반환
    ↓
수정된 스펙으로 파드 실행
    ↓
Envoy 부팅 후 istiod에 gRPC 연결 유지 (xDS 수신)
```

### 5.4 xDS 프로토콜 컴포넌트

```
istiod가 Envoy에 push하는 설정 종류:
  LDS (Listener Discovery)  : 어떤 포트를 리스닝할지
  RDS (Route Discovery)     : VirtualService 기반 라우팅 규칙
  CDS (Cluster Discovery)   : DestinationRule 기반 업스트림 서비스 목록
  EDS (Endpoint Discovery)  : 각 서비스의 실제 파드 IP 목록
  meshConfig                : Zipkin 주소, tracing 설정 등
```

---

## 6. 트러블슈팅 (Troubleshooting)

### 문제 1: Trace가 하나로 이어지지 않고 단절된다

**증상**: Grafana Tempo에서 서비스 A → B → C 호출 시 Trace가 끊기는 것처럼 보임

**원인**: 애플리케이션이 B3 헤더(`x-b3-traceid`, `x-b3-spanid`, `x-b3-parentspanid`)를 다음 서비스 호출 시 전달하지 않음

**해결책**:
- OTel Java Agent를 함께 사용하면 자동으로 헤더 전파 처리
- 수동 구현 시 수신한 B3 헤더를 그대로 downstream 호출에 포함

### 문제 2: Loki 파드 재시작 시 로그 유실

**증상**: Loki Ingester 파드 재시작 후 최근 로그가 사라짐

**원인**: WAL 경로(`/var/lib/loki/wal`)가 PV에 마운트되지 않아 파드 임시 디스크에만 저장됨

**해결책**: Ingester 파드에 PV 마운트 또는 프로덕션에서 S3 오브젝트 스토리지 백엔드 사용

### 문제 3: alloy-otlp-loadbalancer에서 Trace 누락

**증상**: Istio Envoy Span이 Tempo에 도달하지 않음

**체크 포인트**:
1. `otelcol.receiver.zipkin` endpoint가 `0.0.0.0:9411`인지 확인
2. Istio meshConfig의 zipkin.address가 alloy-otlp-loadbalancer 서비스 주소를 가리키는지 확인
3. `otelcol.exporter.loadbalancing`의 protocol이 `otlp`이고 collector의 해당 포트가 OTLP를 수신하도록 설정되었는지 확인

### 문제 4: OTel Java Agent 적용 후 메트릭 중복

**증상**: Grafana 대시보드에서 HTTP 요청 수 메트릭이 두 배로 집계됨

**원인**: Spring Boot Actuator(Micrometer)와 OTel Java Agent 양쪽에서 동일한 HTTP 메트릭 수집

**해결책**:
```bash
# Java Agent에서 메트릭 비활성화 (Micrometer 단독 사용)
-Dotel.metrics.exporter=none

# 또는 Java Agent에서 로그도 함께 비활성화
-Dotel.metrics.exporter=none
-Dotel.logs.exporter=none
```

---

## 7. 치트 시트 (Cheat Sheet)

```
# Loki 컴포넌트별 역할 한 줄 요약
Gateway          : Nginx 역방향 프록시, 단일 진입점 (외부 → 내부 라우팅)
Distributor      : 수신 검증 + 일관 해시 링으로 Ingester 선택 (stateless)
Ingester         : 메모리/WAL 버퍼링 → 오브젝트 스토리지 flush (stateful)
Querier          : LogQL 실행, Ingester + S3 병합 조회
Query Frontend   : 대용량 쿼리 샤딩, 캐싱
Query Scheduler  : 쿼리 큐 관리, 공정 분배

# istiod 핵심 역할
1. MutatingWebhook → 사이드카 주입 (파드 생성 시 1회)
2. xDS gRPC → Envoy 동적 설정 배포 (상시 연결 유지)
3. SPIFFE 인증서 발급/갱신 (mTLS)
4. CRD ValidatingWebhook (설정 오류 사전 차단)

# 전송 프로토콜 포트
OTLP gRPC : 4317
OTLP HTTP : 4318
Zipkin    : 9411

# Loki 배포 모드 전환
grafana/loki -target=all          # Monolithic
grafana/loki -target=distributor  # Distributed - Distributor만 실행
grafana/loki -target=ingester     # Distributed - Ingester만 실행

# OTel Java Agent 필수 설정값 (엔드포인트는 alloy-otlp-loadbalancer)
-Dotel.service.name=<서비스명>
-Dotel.exporter.otlp.endpoint=http://alloy-otlp-loadbalancer:4317
# Trace → Collector → Tempo / Metric → Mimir 분기는 loadbalancer가 처리

# B3 헤더 목록 (Zipkin 트레이싱)
X-B3-TraceId      : 전체 요청 흐름 식별자 (128bit)
X-B3-SpanId       : 현재 Span 식별자
X-B3-ParentSpanId : 부모 Span 식별자
X-B3-Sampled      : 샘플링 여부 (1=수집)
```

---

## 8. 심화 학습 (Deep Dive)

### 8.1 alloy-otlp-loadbalancer Zipkin 수신 포트의 혼동 포인트

현재 설정에서 resolver `ports = [9411]`로 되어 있지만, `protocol { otlp {} }`로 설정되어 있다.
이 의미는 **alloy-otlp-collector의 9411 포트에 OTLP gRPC로 전송**한다는 뜻이다.
Zipkin 포트번호(9411)를 그대로 재사용하고 있지만, 실제 프로토콜은 OTLP이므로
alloy-otlp-collector의 해당 포트 설정도 OTLP receiver로 구성되어 있어야 한다.

### 8.2 openCensusAgent 키워드 주의 (Istio 버전별 차이)

Istio meshConfig에서 OTLP 엔드포인트를 지정할 때 `openCensusAgent:`라는 키를 사용하는 것은
Istio 내부 API의 역사적 네이밍 quirk이다.
실제로는 OTel(OTLP) gRPC 엔드포인트로 전송하지만, OpenCensus gRPC 포맷으로 송신하는 경우도 있다.

- **Istio 1.16 ~ 1.21**: `openCensusAgent` 방식 사용
- **Istio 1.22+**: Telemetry API의 `opentelemetry` extension provider 방식 권장
  → Envoy와 Java Agent가 동일한 W3C `traceparent` 헤더 사용 → 별도 변환 없이 Trace 자연 연결

### 8.3 Envoy Sidecar와 OTel Java Agent의 계층적 계측 관계

두 컴포넌트는 서로 다른 레이어를 계측하므로 중복이 아닌 상호 보완 관계다.

```
[OTel Java Agent] → 애플리케이션 내부 계측
  - JDBC 쿼리 시간, 메서드 레벨 Span, 비즈니스 로직

[Envoy Sidecar] → 네트워크 레벨 계측
  - 서비스 간 HTTP/gRPC 요청 시간, 상태코드, 네트워크 레이턴시
```

두 Span이 하나의 Trace로 연결되려면 B3 헤더 또는 W3C TraceContext 헤더가
Envoy → App → Envoy 순서로 올바르게 전파되어야 한다.

### 8.4 Grafana LGTM 스택 통합 연동

Grafana에서 세 데이터 소스가 TraceID를 키로 서로 연동된다.

```
Tempo에서 TraceID 클릭
    → Grafana가 동일 TraceID로 Loki 쿼리 자동 실행
    → 해당 요청의 애플리케이션 로그 표시

Span에서 시간 범위 클릭
    → Grafana가 해당 시간대 Mimir PromQL 쿼리 자동 실행
    → JVM 메트릭, 에러율, 응답시간 메트릭 표시
```

이 연동이 동작하려면 Loki 로그에 TraceID가 라벨 또는 로그 본문에 포함되어 있어야 하고,
OTel Java Agent의 로그 계측이 활성화되어 있어야 한다.

---

## 9. 나만의 요약 (My Summary)

이 스택을 이해하는 핵심 시각은 **"누가 수집하고 누가 저장하는가"**를 명확히 분리하는 것이다.

- **Alloy** (로그전용·OTLP LB·Collector): 수집·변환·전달만, 저장 없음
- **Loki/Tempo/Mimir**: 수신 후 저장 담당, 내부는 모두 Distributor → Ingester → Object Storage 구조

Istio 트레이싱 파이프라인에서 헷갈리기 쉬운 포인트는 세 가지다:
1. **istiod는 Span을 전송하지 않는다** — 설정만 배포하고 Envoy가 실제 전송
2. **alloy-otlp-loadbalancer는 프로토콜 변환기이자 라우터다** — Zipkin을 OTLP로 변환하고, Trace는 Collector로, Metric은 Mimir로 직접 분기
3. **alloy-otlp-collector는 Trace 전용이다** — Metric은 loadbalancer에서 이미 Mimir로 직접 전송되므로 collector를 거치지 않음

OTel Java Agent의 바이트코드 계측은 소스코드를 건드리지 않지만,
Micrometer와 함께 쓸 때 메트릭 중복 수집이 발생할 수 있으므로
두 수집 경로를 명확히 분리하는 것이 중요하다.

---

## 10. 참고 자료 (References)

- [Grafana Alloy 공식 문서](https://grafana.com/docs/alloy/latest/)
- [Loki Architecture 공식 문서](https://grafana.com/docs/loki/latest/get-started/components/)
- [Tempo Distributed 공식 문서](https://grafana.com/docs/tempo/latest/)
- [Mimir Architecture 공식 문서](https://grafana.com/docs/mimir/latest/get-started/architecture/)
- [OpenTelemetry Java Agent 공식 문서](https://opentelemetry.io/docs/zero-code/java/agent/)
- [Istio Distributed Tracing 공식 문서](https://istio.io/latest/docs/tasks/observability/distributed-tracing/)
- [Istio Telemetry API (OpenTelemetry)](https://istio.io/latest/docs/tasks/observability/distributed-tracing/opentelemetry/)
- [otelcol.exporter.loadbalancing (Alloy)](https://grafana.com/docs/alloy/latest/reference/components/otelcol/otelcol.exporter.loadbalancing/)
