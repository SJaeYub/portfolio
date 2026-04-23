# KNW-istio-traffic-flow

> 학습한 내용을 나중에 언제 봐도 이해할 수 있도록 개념·원리·실제 사례를 함께 기록합니다.

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **주제** | Istio Service Mesh 트래픽 흐름 및 내부 동작 원리 |
| **분류** | 네트워크 / 인프라 / 플랫폼 |
| **키워드** | Istio, Envoy, sidecar, iptables, xDS, LDS, RDS, CDS, EDS, SDS, ADS, mTLS, SPIFFE, istiod, Pilot, Citadel, Mutating Webhook |
| **학습 계기** | 업무 중 필요 / 자기계발 |
| **관련 업무 ID** | — |
| **최초 작성일** | 2026-04-23 |
| **최종 수정일** | 2026-04-23 |
| **숙련도** | 🌿 이해 |

---

## 1. 한 줄 요약 (TL;DR)

```
Istio는 파드 내부에 Envoy 사이드카를 주입하고, istio-init이 파드 네임스페이스
iptables를 조작하여 모든 트래픽을 Envoy(15001/15006)로 강제 리다이렉트한다.
Envoy는 istiod로부터 xDS(CDS→EDS→LDS→RDS)로 실시간 설정을 받아 EDS로 목적지
파드 IP를 직접 결정하므로, kube-proxy의 ClusterIP DNAT를 우회하고 mTLS·관찰성·
트래픽 정책을 애플리케이션 코드 변경 없이 제공한다.
```

---

## 2. 배경 지식 (Prerequisites)

- **필수 선행 지식**:
  - Kubernetes Pod / Namespace / Service / Endpoints 개념
  - kube-proxy와 ClusterIP DNAT 동작 방식
  - Linux Network Namespace, veth pair, CNI 동작 원리
  - Linux iptables 기본 구조 (테이블/체인/룰)
  - Netfilter 훅 포인트 (PREROUTING / INPUT / FORWARD / OUTPUT / POSTROUTING)

- **관련 개념과의 위치**:
  ```
  [k8s Control Plane]
    └── kube-apiserver
          └── Admission Webhook → istiod (Mutating Webhook)
                └── sidecar 삽입

  [Node]
    ├── kube-proxy (노드 네트워크 NS iptables: ClusterIP DNAT)
    └── Pod 네트워크 NS
          ├── istio-init (iptables 룰 주입)
          ├── Envoy (istio-proxy) ← 이 문서의 핵심
          │     ├── 15001 (아웃바운드 핸들러)
          │     └── 15006 (인바운드 핸들러)
          └── App Container

  [istiod]
    ├── Pilot    → xDS (ADS gRPC 스트림)
    ├── Citadel  → SDS (인증서)
    └── Webhook  → 사이드카 삽입 / CRD 검증
  ```

---

## 3. 개념 설명 (Concept)

### 3.1 정의 (What)

```
Istio는 Service Mesh 솔루션으로, 마이크로서비스 간 통신에 대한
mTLS 보안, 트래픽 라우팅, 관찰성(metrics/tracing)을 애플리케이션 코드
수정 없이 인프라 레벨에서 제공한다.

핵심 메커니즘: 파드마다 Envoy 프록시를 사이드카로 삽입하고,
파드 내부 iptables를 조작하여 모든 트래픽이 Envoy를 반드시 경유하도록 강제.
```

### 3.2 존재 이유 (Why)

```
기존 k8s 환경에서의 문제:
- 서비스 간 mTLS 적용 → 앱마다 TLS 코드 필요
- 재시도/Circuit Breaker → 앱마다 개별 구현
- 분산 트레이싱 → 앱마다 trace context 전파 코드 필요
- 카나리 배포·트래픽 분산 → ingress/앱 레벨 복잡도 증가

Istio 도입 후:
- 모든 기능을 Envoy 사이드카가 투명하게 처리
- 앱은 평문 HTTP로 통신하고, Envoy 간 구간만 mTLS 암호화
- 트레이드오프: 인바운드·아웃바운드 각 1홉씩 추가 (레이턴시 소폭 증가)
```

### 3.3 동작 원리 (How it works)

#### 파드 생성 시 추가되는 작업

```
Step 1. Mutating Webhook
  - kubectl apply → kube-apiserver → istiod Mutating Webhook 호출
  - istiod가 파드 스펙에 두 가지 자동 삽입:
      a) istio-init  (Init 컨테이너)
      b) istio-proxy (Envoy 사이드카 컨테이너)

Step 2. istio-init 실행 (파드 기동 시 한 번만)
  - 파드 네트워크 네임스페이스 내부 iptables에 규칙 주입:
      아웃바운드: OUTPUT → ISTIO_OUTPUT → ISTIO_REDIRECT (포트 15001)
      인바운드:   PREROUTING → ISTIO_INBOUND → ISTIO_IN_REDIRECT (포트 15006)
      루프 방지:  UID 1337 패킷 → RETURN / 출발지 127.0.0.6 → RETURN

Step 3. Envoy 기동
  - istiod와 ADS(Aggregated Discovery Service) gRPC 스트림 수립
  - xDS 순서대로 설정 수신: CDS → EDS → LDS → RDS (+ SDS 독립)
```

#### 변경된 인바운드 패킷 흐름 (외부 → 목적지 파드)

```
외부 LB
  ↓
Istio IngressGateway(Envoy) ← L7 VirtualService 룰로 목적지 서비스 결정
  ↓
노드 iptables (kube-proxy) DNAT → 목적지 파드 IP 결정
  ↓
목적지 노드 veth → 파드 네트워크 NS 진입
  ↓
[파드 내부 iptables PREROUTING]
  → ISTIO_INBOUND → ISTIO_IN_REDIRECT
  → 포트 15006으로 리다이렉트
  ↓
Envoy 15006 (인바운드 핸들러)
  · mTLS 복호화, SPIFFE ID 검증
  · AuthorizationPolicy 적용
  · 텔레메트리(metrics, trace span) 기록
  · 출발지 IP를 127.0.0.6으로 변경 후 앱으로 전달
  ↓
[파드 내부 iptables ISTIO_OUTPUT]
  → 출발지 127.0.0.6 감지 → rule1 RETURN (인바운드 루프 방지)
  ↓
앱 컨테이너 수신 (평문 HTTP로 보임)
```

#### 변경된 아웃바운드 패킷 흐름 (파드 → 외부)

```
앱 컨테이너 (UID ≠ 1337) → ClusterIP:Port로 요청
  ↓
[파드 내부 iptables OUTPUT]
  → ISTIO_OUTPUT → ISTIO_REDIRECT
  → 포트 15001로 리다이렉트
  ↓
Envoy 15001 (아웃바운드 핸들러)
  · LDS → RDS → CDS → EDS 파이프라인으로 목적지 파드 IP 결정
  · mTLS 핸드셰이크 (SPIFFE X.509 SVID 사용)
  · 로드밸런싱·retry·circuit-breaker 정책 적용
  · 텔레메트리 기록
  · UID 1337로 패킷 재전송
  ↓
[파드 내부 iptables ISTIO_OUTPUT 재진입]
  → UID 1337 감지 → rule4 RETURN (아웃바운드 루프 방지)
  ↓
veth → 호스트 노드 네트워크 NS
  ↓
[노드 iptables PREROUTING]
  → KUBE-SERVICES 탐색: 목적지가 파드 IP(ClusterIP 아님) → 매칭 없음 통과
  ↓
[노드 라우팅 테이블] ← 핵심: 파드 CIDR → 목적지 노드 IP 결정
  → 예: 10.0.1.0/24 via 192.168.1.2 (Node B)
  ↓
물리 NIC → 목적지 노드로 전송 (mTLS 암호화 상태)
```

### 3.4 구성 요소 / 핵심 용어 (Key Terms)

#### Istio Control Plane (istiod)

| 컴포넌트 | 역할 | 비고 |
|---------|------|------|
| **Pilot** | k8s 리소스 → xDS 변환·push (트래픽 관리) | ADS gRPC 스트림 |
| **Citadel** | SPIFFE X.509 인증서 발급·rotate (CA 역할) | SDS 채널로 전달 |
| **Validation Webhook** | Istio CRD 문법·의미 유효성 검사 | kubectl apply 시 |
| **Mutating Webhook** | 파드 스펙에 sidecar·istio-init 자동 삽입 | 파드 생성 시 |

#### Istio Data Plane

| 컴포넌트 | 역할 | 비고 |
|---------|------|------|
| **istio-init** | 파드 NS iptables 룰 주입 (Init 컨테이너) | 파드 기동 시 1회 실행 후 종료 |
| **Envoy (15001)** | 아웃바운드 처리: LB·CB·retry·mTLS | istio-proxy 컨테이너 |
| **Envoy (15006)** | 인바운드 처리: mTLS 종료·AuthZ·텔레메트리 | 동일 컨테이너 |

#### xDS 프로토콜

| xDS | 이름 | 역할 | 관련 Istio 리소스 |
|-----|------|------|-------------------|
| **LDS** | Listener Discovery Service | 포트·프로토콜·FilterChain 정의 | — |
| **RDS** | Route Discovery Service | L7 라우팅 룰 (헤더·URI·가중치) | VirtualService |
| **CDS** | Cluster Discovery Service | 업스트림 서비스 그룹 정의, CB·OutlierDetection | DestinationRule |
| **EDS** | Endpoint Discovery Service | 실제 파드 IP:Port 목록 | k8s EndpointSlice |
| **SDS** | Secret Discovery Service | mTLS 인증서·키·CA 번들 | Citadel 발급 인증서 |
| **ADS** | Aggregated Discovery Service | 위 5가지를 단일 gRPC 스트림으로 통합 | — |

#### iptables 루프 방지 메커니즘

| 상황 | 감지 조건 | 룰 | 동작 |
|------|-----------|-----|------|
| Envoy 아웃바운드 재전송 | UID = 1337 | rule4 | RETURN (ISTIO_OUTPUT 탈출) |
| Envoy 인바운드 재전송 | 출발지 IP = 127.0.0.6 | rule1 | RETURN (ISTIO_OUTPUT 탈출) |

---

## 4. 비교 및 구분 (Comparison)

### 4.1 Istio 도입 전 vs 후 핵심 변화

| 구분 | Istio 없음 | Istio 있음 |
|------|-----------|-----------|
| **트래픽 처리 주체** | kube-proxy (노드 iptables) | kube-proxy + Envoy (파드 내부 iptables) |
| **L7 처리 위치** | lb-controller 파드 | IngressGateway(Envoy) + 각 파드 sidecar |
| **인바운드 경로** | veth → 앱 컨테이너 직접 | veth → 파드 iptables → 15006 Envoy → 앱 |
| **아웃바운드 경로** | 앱 → 노드 iptables | 앱 → 파드 iptables → 15001 Envoy → 노드 |
| **ClusterIP DNAT** | 노드 iptables KUBE-SEP 체인 | Envoy EDS로 파드 IP 직접 결정 (DNAT 우회) |
| **노드 IP 라우팅** | 거침 (필수) | 동일하게 거침 (필수) |
| **mTLS** | 없음 | Envoy 간 자동 mTLS |
| **홉 수** | 운동장 1바퀴 | 운동장 3바퀴 (비용) |

### 4.2 "노드 iptables 우회"의 정확한 의미

```
❌ 잘못된 이해: 노드 커널 iptables 전체를 건너뜀
✅ 정확한 의미: kube-proxy가 만든 KUBE-SERVICES → KUBE-SEP-xxx DNAT 체인만 미매칭

이유: Envoy가 EDS로 이미 파드 IP를 알고 직접 전송하므로
      KUBE-SERVICES 체인에서 어떤 룰도 히트되지 않을 뿐,
      노드 FORWARD 체인·라우팅 테이블은 반드시 거친다.
```

### 4.3 iptables vs 라우팅 테이블 역할 구분

| 레이어 | 담당 역할 | 처리 시점 |
|--------|----------|----------|
| **iptables** | 목적지 주소 변환 (DNAT), 허용/차단, 리다이렉트 | 주소 변환 |
| **라우팅 테이블** | "어떤 노드(NIC)로 내보낼까" 결정 | 라우팅 결정 |

```
패킷 흐름 순서:
iptables (nat PREROUTING/OUTPUT) → 라우팅 결정 → iptables (filter FORWARD) → iptables (nat POSTROUTING) → NIC
```

---

## 5. 실전 예시 (Examples)

### 5.1 Pod A → Pod B 전체 흐름 (Istio 환경)

| 단계 | 처리 주체 | 핵심 동작 |
|------|----------|----------|
| ① 앱 → iptables | 파드 A 커널 | ISTIO_REDIRECT → 15001 |
| ② Envoy (아웃) | 파드 A Envoy | EDS로 목적지 파드 IP 결정, mTLS 핸드셰이크, LB·정책 적용 |
| ③ iptables 통과 | 파드 A 커널 | UID 1337 감지 → RETURN (루프 방지) |
| ④ veth → 노드 | 호스트 커널 | 라우팅 테이블로 목적지 노드 결정 (DNAT 없음) |
| ⑤ veth → 파드 B | 목적지 노드 커널 | ISTIO_INBOUND → 15006 |
| ⑥ Envoy (인바) | 파드 B Envoy | mTLS 종료, 인증/인가, 텔레메트리 |
| ⑦ → 앱 컨테이너 | 파드 B 커널 | 출발지 127.0.0.6 → rule1 RETURN → 앱 전달 |

### 5.2 순수 k8s vs Istio iptables 체인 비교

| 통과 지점 | 순수 k8s | Istio |
|----------|---------|-------|
| 소스 파드 NS OUTPUT | 룰 없음, ACCEPT | ISTIO_OUTPUT → ISTIO_REDIRECT → 15001 |
| 소스 파드 NS OUTPUT (재진입) | 해당 없음 | UID 1337 감지 → RETURN |
| 소스 노드 PREROUTING | KUBE-SERVICES → **DNAT 적용** | KUBE-SERVICES → 매칭 없음, **DNAT 미적용** |
| 소스 노드 FORWARD | KUBE-FORWARD ACCEPT | KUBE-FORWARD ACCEPT (동일) |
| 소스 노드 POSTROUTING | MASQUERADE 조건 확인 | 동일 |
| 목적지 노드 FORWARD | ACCEPT | ACCEPT (동일) |
| 목적지 파드 NS PREROUTING | 룰 없음, ACCEPT | ISTIO_INBOUND → ISTIO_IN_REDIRECT → 15006 |
| 목적지 파드 NS OUTPUT (재진입) | 해당 없음 | 출발지 127.0.0.6 감지 → RETURN |

---

## 6. 트러블슈팅 경험 (Troubleshooting)

> 직접 경험한 사례가 생기면 추가 예정

---

## 7. 실전 명령어 / 치트시트 (Cheat Sheet)

```bash
# Envoy 엔드포인트 목록 확인 (ClusterIP가 아닌 파드 IP임을 확인)
istioctl proxy-config endpoints <pod-name>.<namespace>

# Envoy 리스너 목록 확인 (15001/15006 포트 등)
istioctl proxy-config listeners <pod-name>.<namespace>

# Envoy 클러스터 목록 확인
istioctl proxy-config clusters <pod-name>.<namespace>

# Envoy 라우트 목록 확인
istioctl proxy-config routes <pod-name>.<namespace>

# Envoy 시크릿(인증서) 확인
istioctl proxy-config secrets <pod-name>.<namespace>

# 파드 내부 iptables 룰 확인 (istio 룰 확인)
kubectl exec -n <namespace> <pod-name> -c istio-proxy -- iptables -t nat -L -n -v

# xDS 설정 덤프 (전체)
istioctl proxy-config all <pod-name>.<namespace> -o json

# Istio 설정 동기화 상태 확인
istioctl proxy-status

# mTLS 정책 확인
istioctl x check-inject -n <namespace>
```

---

## 8. 깊게 이해하기 (Deep Dive)

### 8.1 xDS 파이프라인 상세 및 의존 관계

#### 올바른 xDS push 순서: CDS → EDS → LDS → RDS

> ⚠️ **주의**: 일부 자료에서 "LDS → RDS → CDS → EDS" 순서로 기술하는 경우가 있으나 이는 잘못된 표현이다.
> Envoy는 Cluster가 정의되지 않은 상태에서 Route가 해당 Cluster를 참조하면 503 오류가 발생한다.
> 따라서 **CDS가 반드시 RDS보다 먼저** push되어야 한다.
> 올바른 의존 순서: CDS → EDS → LDS → RDS

```
istiod 변경 감지
  ↓
1. CDS push  ← Cluster(업스트림 서비스 그룹) 정의 먼저
  ↓
2. EDS push  ← Cluster에 속한 실제 파드 IP:Port 목록 전달
  ↓
3. LDS push  ← Listener(포트·필터체인) 정의
  ↓
4. RDS push  ← Route(L7 라우팅 룰): Cluster를 참조하므로 마지막
  ↓
5. SDS push  ← 언제든 독립적으로 rotate 가능 (인증서 만료 시)
```

#### 아웃바운드 Envoy 내부 처리 파이프라인 (15001 수신 후)

```
① LDS (Listener)
  - 패킷의 원래 목적지 IP:Port를 SO_ORIGINAL_DST로 읽어 FilterChain 선택
  - HTTP → http_connection_manager 필터
  - TCP  → tcp_proxy 필터

② RDS (Route)
  - VirtualService 룰을 Envoy Route 설정으로 변환
  - 헤더·URI·가중치 기반 라우팅, A/B 테스트, Canary 분기
  - 최종적으로 어떤 Cluster로 보낼지 결정
    예: outbound|9080||ratings.default.svc.cluster.local

③ CDS (Cluster)
  - Circuit Breaker 임계치, Outlier Detection, Connection Pool 설정
  - 로드밸런싱 알고리즘(Round Robin, Least Request 등) 결정
  - DestinationRule의 내용이 이 CDS 설정으로 변환됨

④ EDS (Endpoint)
  - 선택된 Cluster에 속한 실제 파드 IP:Port 목록에서 목적지 결정
  - istiod가 k8s EndpointSlice 변경 시 실시간 push
  - 가중치 기반 엔드포인트 선택, Locality-aware 라우팅 적용

⑤ Transport Socket (mTLS)
  - istiod(Citadel)에서 발급받은 SPIFFE X.509 SVID 인증서로 TLS 핸드셰이크
  - 상대 Envoy의 인증서 검증 (mutual authentication)
  - UID 1337로 패킷 재전송

⑥ 텔레메트리
  - 레이턴시·상태코드·서비스 정보를 Prometheus 형식 metrics로 기록
  - B3 헤더(x-b3-traceid) 전파 및 Span 생성
```

### 8.2 시나리오별 xDS 동작

#### 시나리오 1 — 새 파드 배포 시 (EDS push)

```
k8s API: Pod Created → Running → Ready
  ↓
istiod가 Endpoints 변경 감지 (List/Watch)
  ↓
해당 서비스를 바라보는 모든 Envoy에게
EDS push: "ratings-v1 파드 IP 10.0.1.5:9080 추가됨"
  ↓
각 Envoy 즉시 엔드포인트 목록 업데이트
→ 다음 요청부터 새 파드에 트래픽 전달 (재시작 불필요)
```

#### 시나리오 2 — VirtualService 배포 시 (Canary 라우팅)

```
kubectl apply -f virtualservice.yaml
  ↓
istiod가 VirtualService 객체 감지
  ↓
영향 받는 Envoy들에게 (올바른 순서로):
  CDS push: "outbound|9080|v2|ratings 클러스터 신규 등록"
  EDS push: "v2 클러스터 엔드포인트 목록 전달"
  RDS push: "ratings 서비스 요청 중 header:x-user=test이면 v2 Cluster로"
  ↓
이후 요청부터 헤더 기반 Canary 라우팅 즉시 적용
```

#### 시나리오 3 — mTLS 인증서 만료 전 Rotate 시 (SDS push)

```
istiod: 인증서 만료 80% 도달 감지
  ↓
새 X.509 인증서 발급 (SPIFFE SVID)
  ↓
SDS push: 해당 파드의 Envoy에게 새 인증서·키 전달
  ↓
Envoy: 기존 TLS 세션 유지하면서 새 인증서로 Hot Reload
→ 트래픽 중단 없음, 파드 재시작 불필요
```

### 8.3 istiod 구성요소 의존 관계 다이어그램

```
┌─────────────────────────────────────────────────────────────────────┐
│                         k8s API Server                              │
│   (Service, Endpoints, VirtualService, DestinationRule, Pod, SA...) │
└────────────────────────────┬────────────────────────────────────────┘
                             │ List/Watch
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                            istiod                                   │
│                                                                     │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────────┐   │
│  │   Pilot     │  │   Citadel    │  │  Validation/Mutating     │   │
│  │(xDS 변환·push)│  │(CA·인증서발급)│  │      Webhook            │   │
│  └──────┬──────┘  └──────┬───────┘  └────────────┬─────────────┘   │
│         │                │                        │                 │
│         │ ADS gRPC       │ SDS gRPC               │ 파드 스펙 변조   │
│         │ (CDS·EDS       │ (X.509                 │ (사이드카 삽입)  │
│         │  LDS·RDS)      │  SVID)                 │                 │
└─────────┼────────────────┼────────────────────────┼─────────────────┘
          │                │                        │
          ▼                ▼                        ▼
┌──────────────────────────────────────┐  ┌────────────────────────┐
│       Pod (사이드카 파드)              │  │     Pod 생성 시         │
│                                      │  │  istio-init 실행        │
│  ┌───────────────────────────────┐   │  │  → iptables 룰 주입     │
│  │    Envoy (istio-proxy)        │   │  └────────────────────────┘
│  │                               │   │
│  │  15001 (아웃바운드 핸들러)      │   │
│  │  └─ LDS→RDS→CDS→EDS          │   │
│  │     (목적지 파드 IP 결정)       │   │
│  │     (로드밸런싱·CB·retry)       │   │
│  │                               │   │
│  │  15006 (인바운드 핸들러)        │   │
│  │  └─ LDS → mTLS 종료           │   │
│  │     → AuthzPolicy 적용        │   │
│  │     → 텔레메트리 기록          │   │
│  │                               │   │
│  │  SDS: 인증서 보유·rotate       │   │
│  └───────────────────────────────┘   │
│                                      │
│  ┌────────────────────────────────┐  │
│  │       App Container            │  │
│  │   (UID ≠ 1337, 평문 HTTP)      │  │
│  └────────────────────────────────┘  │
│                                      │
│  ┌────────────────────────────────┐  │
│  │     파드 내부 iptables           │  │
│  │  OUT: → 15001 (UID≠1337 시)    │  │
│  │  OUT: RETURN  (UID=1337 시)    │  │
│  │  IN:  → 15006 (외부 진입 시)   │  │
│  │  IN:  RETURN  (src=127.0.0.6) │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
               │ veth pair
               ▼
┌──────────────────────────────────────┐
│          호스트 노드 커널              │
│   iptables FORWARD 체인              │
│   (KUBE-SERVICES DNAT은 매칭 안 됨)  │
│   → 라우팅 테이블로 목적지 노드 전달   │
└──────────────────────────────────────┘
```

### 8.4 iptables 3계층 구조 요약

| 계층 | 설명 |
|------|------|
| **테이블** | 목적별 분리: filter(방화벽), nat(주소변환), mangle(헤더조작), raw, security |
| **체인** | Netfilter 훅 포인트와 매핑: PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING |
| **룰** | 매칭 조건(Match) + 타겟(Target)으로 구성, 위에서부터 순서대로 탐색, 첫 매칭 즉시 종료 |

```
훅 포인트별 테이블 적용 순서:
PREROUTING:  raw → mangle → nat
INPUT:       mangle → filter
FORWARD:     mangle → filter
OUTPUT:      raw → mangle → nat → filter
POSTROUTING: mangle → nat
```

#### kube-proxy vs Istio iptables 사용 비교

| 사용 주체 | 테이블 | 체인 | 목적 |
|----------|--------|------|------|
| **kube-proxy** | nat | PREROUTING, OUTPUT | ClusterIP → 파드 IP DNAT |
| **kube-proxy** | nat | POSTROUTING | 노드 간 트래픽 MASQUERADE |
| **Istio istio-init** | nat | OUTPUT | 아웃바운드 → 15001 REDIRECT |
| **Istio istio-init** | nat | PREROUTING | 인바운드 → 15006 REDIRECT |
| **Istio istio-init** | nat | OUTPUT + owner match | UID 1337 → RETURN (루프 방지) |

> **포인트**: kube-proxy는 노드 네트워크 NS에, istio-init은 파드 네트워크 NS에 룰을 추가한다.
> 두 NS는 완전히 분리되므로 서로 간섭 없음.

---

## 9. 나만의 요약 (My Summary)

```
Istio = "파드 내부 우체국(Envoy)" + "집배원 명부(xDS)"

기존 k8s: 노드 iptables(kube-proxy)가 ClusterIP를 파드 IP로 바꿔준다.
Istio:    파드 내부 iptables가 모든 트래픽을 Envoy로 보내고,
          Envoy가 istiod한테서 받은 EDS 명부로 파드 IP를 직접 찾는다.
          노드 iptables는 거치지만 DNAT 룰은 매칭되지 않는다(명부가 이미 있으니까).

루프 방지 핵심:
  - Envoy 아웃 → UID 1337로 재전송 → ISTIO_OUTPUT rule4 RETURN
  - Envoy 인바 → 출발지 127.0.0.6으로 재전송 → ISTIO_OUTPUT rule1 RETURN
```

**기억할 포인트 3가지:**
1. xDS 올바른 push 순서: **CDS → EDS → LDS → RDS** (Cluster 정의가 Route보다 먼저)
2. "노드 iptables 우회"의 실체: FORWARD/라우팅 테이블은 거침, **ClusterIP DNAT 체인만 미매칭**
3. 파드 IP → 노드 IP 결정은 iptables가 아닌 **라우팅 테이블**이 담당 (CNI가 파드CIDR → 노드IP 등록)

**다음에 헷갈릴 것 같은 부분:**
- UID 1337(아웃바운드 루프 방지) vs 127.0.0.6(인바운드 루프 방지) 헷갈림 주의
- xDS push 순서를 LDS → RDS → CDS → EDS로 잘못 기억할 가능성 (실제: CDS → EDS → LDS → RDS)
- "노드 iptables 우회"를 노드 커널 전체 우회로 오해할 가능성

---

## 10. 참고 자료 (References)

| 유형 | 제목 | 링크 | 메모 |
|------|------|------|------|
| 공식 문서 | Istio Traffic Management | https://istio.io/latest/docs/concepts/traffic-management/ | |
| 공식 문서 | Istio Architecture | https://istio.io/latest/docs/ops/deployment/architecture/ | |
| 공식 문서 | Istio Sidecar | https://istio.io/latest/docs/reference/config/networking/sidecar/ | |
| 공식 문서 | Traffic for Ambient and Sidecar | https://istio.io/latest/blog/2023/traffic-for-ambient-and-sidecar/ | |
| GitHub | Istio Control and Data flow through Envoy | https://github.com/istio/istio/wiki/Control-and-Data-flow-through-Envoy | EDS 우회 설명 핵심 |
| 블로그 | Istio mTLS Pod Communication | https://oneuptime.com/blog/post/2026-01-08-istio-mtls-pod-communication/view | |
| 블로그 | Service Mesh Hops | https://makgol.com/archive/service-mesh-hops | 홉 수 비교 |
| 블로그 | Istio Life of a packet | https://gasidaseo.notion.site/Istio-Life-of-a-packet-6ad9808e14594296bf854dcc203cab71 | |

---

*이 문서는 학습이 깊어질수록 계속 업데이트합니다.*
