# 지식 정리 (Knowledge Note)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **주제** | Kubernetes 스케줄링 & 리소스 관리 (Node Affinity, Taint/Toleration, QoS, NetworkPolicy, PSA, HPA, Anti-Affinity) |
| **분류** | 인프라 / 플랫폼 / Kubernetes |
| **키워드** | Node Affinity, Taint, Toleration, QoS, LimitRange, NetworkPolicy, PSA, HPA, Metrics Server, KEDA, Anti-Affinity, topologyKey |
| **학습 계기** | Kubernetes 파드 스케줄링 제어 전략과 리소스 관리 정책 탐구 |
| **관련 업무 ID** | - |
| **최초 작성일** | 2026-04-27 |
| **최종 수정일** | 2026-04-27 |
| **숙련도** | 🌿 이해 |

---

## 1. 한 줄 요약 (TL;DR)

```
Kubernetes 스케줄링은 "파드가 어느 노드에 갈지"를 제어하는 다층 시스템이다.

파드 → 노드 끌어당김 : Node Affinity (레이블 조건 기반)
파드 ← 노드 밀어냄   : Taint & Toleration (배제 규칙 + 통행권)
파드 ↔ 파드 분산     : Anti-Affinity (같은 레이블끼리 멀리)

리소스 관리는 requests/limits → QoS 클래스 → 축출 우선순위 순으로 연결된다.
```

---

## 2. 배경 지식 (Prerequisites)

- **필수 선행 지식**:
  - Kubernetes 파드(Pod), 노드(Node), 레이블(Label) 기본 개념
  - kube-scheduler 역할 이해
  - Deployment / ReplicaSet 구조
  - Linux OOM Killer 기본 개념

- **관련 개념과의 관계**:
  ```
  [스케줄링 결정 레이어]
  1. Filtering (불가능한 노드 제거)
     └── nodeSelector, Node Affinity (required), Taint/Toleration
  2. Scoring (최적 노드 선택)
     └── Node Affinity (preferred, weight), Anti-Affinity (soft)
  3. 배치 완료 후 유지
     └── IgnoredDuringExecution (기존 파드는 재스케줄 안 함)

  [리소스 관리 레이어]
  requests/limits 설정
    └── QoS 클래스 자동 부여 (Guaranteed / Burstable / BestEffort)
          └── 노드 압박 시 축출 우선순위 결정
  LimitRange → 네임스페이스 기본값 강제
  ResourceQuota → 네임스페이스 전체 합산 제한
  ```

---

## 3. 개념 설명 (Concept)

### 3.1 Node Affinity

```
파드가 "어느 노드를 선호하거나 반드시 요구하는지"를 노드 레이블 기반으로 정의한다.
nodeSelector의 상위 호환으로 고급 연산자와 soft/hard 구분이 가능하다.
```

**두 가지 핵심 유형**

| 유형 | 의미 | 조건 불충족 시 |
|------|------|--------------|
| `requiredDuringSchedulingIgnoredDuringExecution` | Hard 룰 — 반드시 만족해야 함 | 스케줄링 거부 (Pending) |
| `preferredDuringSchedulingIgnoredDuringExecution` | Soft 룰 — 되도록 선호 | 조건 없는 노드에도 허용 |

> `IgnoredDuringExecution`: 파드가 이미 실행 중이면 노드 레이블이 변경되어도 그 노드에서 계속 실행됨.

**지원 Operator**

| 연산자 | 동작 |
|--------|------|
| `In` | 지정 값 목록 중 하나 이상 일치 |
| `NotIn` | 지정 값 목록에 해당 없음 |
| `Exists` | 키(key)가 존재하면 선택 (values 불필요) |
| `DoesNotExist` | 키가 없는 노드 선택 |
| `Gt` / `Lt` | 숫자형 값 초과 / 미만 비교 |

---

### 3.2 Taint & Toleration

```
Taint: 노드에 설정. "나는 아무나 못 받아" — 배제 규칙
Toleration: 파드에 설정. "나는 그 조건을 견딜 수 있어" — 통행권

둘은 항상 쌍으로 동작한다. key + value + effect가 모두 일치해야 매칭.
```

**3가지 Taint Effect**

| Effect | 의미 | 기존 실행 파드 영향 |
|--------|------|-------------------|
| `NoSchedule` | 신규 파드 스케줄링 거부 | 없음 |
| `PreferNoSchedule` | 신규 파드 스케줄링 비선호 (소프트) | 없음 |
| `NoExecute` | 신규 파드 거부 + 기존 파드 강제 퇴출 | 즉시 Eviction |

> `NoExecute`가 가장 강력. 실행 중인 파드도 퇴출하는 유일한 옵션.

---

### 3.3 Pod QoS 클래스

```
requests/limits 설정에 따라 Kubernetes가 자동으로 QoS 클래스를 부여.
노드 리소스 부족 시 QoS가 낮은 파드부터 먼저 축출(Evict).
```

| QoS 클래스 | 조건 | oom_score_adj | 축출 우선순위 |
|-----------|------|--------------|-------------|
| **Guaranteed** | 모든 컨테이너: requests == limits (CPU + 메모리 모두) | -997 (낮음) | 마지막 |
| **Burstable** | requests < limits이거나 일부 컨테이너만 설정 | 상대적 중간 | 중간 |
| **BestEffort** | requests/limits 모두 미설정 | 1000 (높음) | 가장 먼저 |

> oom_score_adj 값이 높을수록 OOM 발생 시 커널이 먼저 종료 대상으로 선택한다.

---

### 3.4 Anti-Affinity

```
Affinity(끌어당김)의 반대. 특정 파드나 노드를 서로 멀리 배치하도록 지시.
고가용성(HA) 확보와 리소스 분산이 목적.
```

**topologyKey** — "어느 단위로 분리할지" 지정

| topologyKey | 분리 단위 |
|------------|---------|
| `kubernetes.io/hostname` | 노드 단위 |
| `topology.kubernetes.io/zone` | 가용 영역(AZ) 단위 |
| `topology.kubernetes.io/region` | 리전 단위 |

---

### 3.5 NetworkPolicy

```
파드 간 또는 파드-외부 간 Ingress/Egress 트래픽을 L3/L4 수준에서 제어.
정책이 없으면 모든 통신 허용, 정책이 적용되면 명시된 것만 허용.
CNI 플러그인이 NetworkPolicy를 지원해야 동작.
```

**CNI별 NetworkPolicy 지원**

| CNI | 지원 여부 |
|-----|---------|
| Calico | ✅ 완전 지원 |
| Cilium | ✅ 완전 지원 (L7까지 가능) |
| Canal | ✅ 지원 |
| Flannel | ❌ 미지원 |
| AWS VPC CNI | ✅ EKS v1.25+ Network Policy 컨트롤러 별도 활성화 필요 |

---

### 3.6 Pod Security Admission (PSA)

```
Kubernetes v1.25 GA. 네임스페이스 레이블 기반으로 파드 보안 표준 준수 여부 검사.
PSP(PodSecurityPolicy)가 v1.25에서 완전히 제거되면서 공식 대체재로 도입.
레이블이 없으면 아무런 제한도 없음 (Opt-in 방식).
```

**3가지 보안 수준**

| 수준 | 의미 |
|------|------|
| `privileged` | 제한 없음 |
| `baseline` | 알려진 권한 상승 방지 (최소 보안) |
| `restricted` | 보안 모범 사례 강제 적용 |

**3가지 동작 모드**

| 모드 | 동작 |
|------|------|
| `enforce` | 위반 시 파드 생성 차단 |
| `audit` | 위반 내용을 감사 로그에 기록 |
| `warn` | 위반 시 사용자에게 경고 출력 |

---

### 3.7 HPA & Metrics Server

```
HPA: 파드 CPU/메모리 사용률에 따라 Replica 수를 자동 조절.
Metrics Server: HPA가 메트릭을 조회하기 위한 전용 경량 API 서버.
               오토스케일링·kubectl top 전용. 모니터링 솔루션이 아님.
```

**HPA 메트릭 조회 경로**

```
cAdvisor (kubelet 내장)
  ↓ 리소스 사용량 수집
Metrics Server
  ↓ Aggregation Layer로 k8s API에 노출
kube-apiserver (/apis/metrics.k8s.io)
  ↓ 메트릭 조회 (15초 주기)
HPA Controller
  ↓ 스케일링 결정
Deployment / ReplicaSet
```

---

### 3.8 구성 요소 / 핵심 용어 (Key Terms)

| 용어 | 설명 | 비고 |
|------|------|------|
| nodeSelector | 단순 Key-Value 노드 레이블 매칭 | Node Affinity의 구버전 |
| nodeSelectorTerms | required 조건 OR 관계 배열 | 배열 내 matchExpressions는 AND |
| weight | preferred affinity의 우선순위 (1~100) | 높을수록 우선 |
| Eviction | 노드 압박 시 파드를 강제 종료하는 동작 | QoS 낮은 순 |
| OOM Kill | 메모리 초과 시 Linux 커널이 프로세스 강제 종료 | CPU는 throttle, 메모리는 Kill |
| LimitRange | 네임스페이스 내 컨테이너 기본/최대 리소스 설정 | 미설정 파드에 자동 적용 |
| KEDA | Kubernetes Event-Driven Autoscaler. 외부 이벤트 기반 HPA | Kafka, SQS, Prometheus 등 |
| Topology Spread Constraints | Anti-Affinity보다 정교한 파드 분산 제어 | maxSkew 기반 |

---

## 4. 비교 및 구분 (Comparison)

### 4.1 Node Affinity vs Taint & Toleration vs Anti-Affinity

| 항목 | Node Affinity | Taint & Toleration | Pod Anti-Affinity |
|------|--------------|-------------------|--------------------|
| 설정 위치 | 파드에 설정 | 노드(Taint) + 파드(Toleration) | 파드에 설정 |
| 기준 | 노드 레이블 | Taint key/value/effect | 파드 레이블 |
| 방향 | 파드 → 노드 끌어당김 | 노드 → 파드 밀어냄 | 파드 ↔ 파드 멀리 |
| 주요 용도 | 특정 노드군에 파드 유인 | 노드 전용화, 보호 | 파드 분산, HA 확보 |

### 4.2 CPU vs 메모리 리소스 초과 동작

| 항목 | CPU | 메모리 |
|------|-----|--------|
| 자원 특성 | 압축 가능(compressible) | 압축 불가능(incompressible) |
| Limit 초과 시 | Throttle (성능 저하) | OOM Kill (즉시 종료) |
| requests 미설정 | 보장 없음 | 보장 없음 |

### 4.3 Metrics Server vs Prometheus

| 항목 | Metrics Server | Prometheus |
|------|---------------|-----------|
| 목적 | HPA/VPA + kubectl top 전용 | 범용 모니터링·알림 |
| 저장 방식 | 인메모리 (휘발성) | 시계열 DB (영속) |
| HPA 연동 | 직접 연동 | Prometheus Adapter 필요 |
| 메트릭 종류 | CPU/메모리만 | 모든 메트릭 |
| 운영 권장 | CPU/메모리 HPA | 비즈니스 메트릭 스케일링 |

### 4.4 HPA + KEDA 역할 분리

```
Metrics Server: CPU/메모리 기반 HPA + kubectl top
KEDA:           Kafka 메시지 수, SQS 큐 길이, Prometheus 커스텀 메트릭 등 이벤트 기반 스케일링

KEDA는 Metrics Server를 대체하지 않는다.
CPU/메모리 기반 스케일링에는 여전히 Metrics Server가 필요하다.

[KEDA 동작]
외부 이벤트 소스 (Kafka, SQS, Prometheus 등)
  ↓
KEDA (ScaledObject → HPA 자동 생성)
  ↓
HPA Controller → 파드 스케일링
  ↑
Metrics Server (CPU/메모리는 별도 제공)
```

### 4.5 Metrics Server — 클라우드별 기본 제공 여부 (오류 수정)

> **⚠️ 오류 수정**: 원문 "EKS, GKE, AKS 같은 매니지드 서비스는 Metrics Server가 기본 제공"은 부정확하다. EKS는 기본 제공하지 않는다.

| 클라우드 | Metrics Server 기본 제공 |
|---------|------------------------|
| **EKS (AWS)** | ❌ 기본 미포함 — 별도 설치 필수 |
| **GKE (Google)** | ✅ 기본 내장 |
| **AKS (Azure)** | ✅ 기본 내장 |
| kubeadm 클러스터 | ❌ 기본 미포함 — 별도 설치 필수 |

### 4.6 PSA 도입 전략

```
권장 순서:
  1단계: warn=restricted  → 경고 로그 수집, 영향도 파악
  2단계: audit=restricted → 감사 로그 기록
  3단계: enforce=restricted → 실제 차단 적용

처음부터 enforce=restricted를 적용하면 기존 파드가 갑자기 차단될 수 있음.
```

---

## 5. 실전 예시 (Examples)

### 5.1 Node Affinity Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  affinity:
    nodeAffinity:
      # 필수 조건: size=large 레이블 노드에만 스케줄링
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values:
            - large
      # 선호 조건: size=medium 노드 우선 (없으면 다른 노드 허용)
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1        # 높을수록 우선
        preference:
          matchExpressions:
          - key: size
            operator: In
            values:
            - medium
  containers:
  - name: nginx
    image: nginx
```

---

### 5.2 Taint & Toleration

```bash
# 노드에 Taint 추가
kubectl taint nodes node1 dedicated=gpu:NoSchedule

# Taint 제거 (끝에 - 추가)
kubectl taint nodes node1 dedicated=gpu:NoSchedule-
```

```yaml
# 파드에 Toleration 설정
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"   # Equal(기본) or Exists
    value: "gpu"
    effect: "NoSchedule"
  containers:
  - name: gpu-container
    image: nvidia/cuda:latest
```

---

### 5.3 Pod Anti-Affinity — 파드 분산 배치

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      affinity:
        podAntiAffinity:
          # Hard: 동일 노드에 같은 앱 파드 절대 금지
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: my-app
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: my-app
        image: my-app:latest
```

---

### 5.4 QoS 클래스별 리소스 설정 예시

```yaml
# ── Guaranteed (requests == limits 모두 동일) ──
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"

# ── Burstable (requests < limits) ──
resources:
  requests:
    cpu: "200m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"

# ── BestEffort (아무것도 미설정) ──
# resources 블록 없음
```

---

### 5.5 LimitRange — 네임스페이스 기본값 설정

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limit
  namespace: my-namespace
spec:
  limits:
  - type: Container
    default:          # limits 미설정 시 자동 적용
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:   # requests 미설정 시 자동 적용
      cpu: "200m"
      memory: "128Mi"
    max:              # 설정 가능한 limits 최대값
      cpu: "2"
      memory: "1Gi"
```

---

### 5.6 NetworkPolicy — 3계층 아키텍처 (Web → WAS → DB)

```yaml
# WAS 파드 정책: Web에서 8080 수신 허용, DB로 3306 송신 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: was-network-policy
spec:
  podSelector:
    matchLabels:
      tier: was
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: web
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: db
    ports:
    - protocol: TCP
      port: 3306
```

---

### 5.7 PSA — 네임스페이스 레이블 설정

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-secure-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

---

### 5.8 HPA Manifest

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50   # CPU requests 대비 50% 초과 시 스케일 아웃
```

---

### 5.9 주의해야 할 패턴 (Anti-pattern)

```yaml
# ❌ HPA 파드에 resources.requests 미설정
containers:
- name: app
  image: my-app:latest
  # resources 없음 → HPA가 % 계산 불가 → 스케일링 동작 안 함
```

```yaml
# ✅ requests 반드시 설정
containers:
- name: app
  image: my-app:latest
  resources:
    requests:
      cpu: "200m"
      memory: "128Mi"
```

---

```yaml
# ❌ required Anti-Affinity + 노드 부족 환경
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:  # Hard 룰
    - topologyKey: "kubernetes.io/hostname"
# → 노드 수 < Replica 수이면 일부 파드 Pending 상태에 빠짐
```

```yaml
# ✅ preferred 사용 (유연성 확보)
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:  # Soft 룰
    - weight: 100
      podAffinityTerm:
        topologyKey: "kubernetes.io/hostname"
```

---

## 6. 트러블슈팅 경험 (Troubleshooting)

### 문제 1. Pending 상태 파드 — Node Affinity 조건 불충족

- **증상**: 파드가 계속 Pending. `kubectl describe pod`에 "0/N nodes are available: N node(s) didn't match node affinity rules."
- **원인**: `requiredDuringScheduling` 조건을 충족하는 노드가 없음.
- **해결**: 노드에 레이블 추가 (`kubectl label node <node> size=large`) 또는 Affinity 조건을 `preferredDuringScheduling`으로 완화.
- **교훈**: required는 Hard 룰. 운영 환경에서 노드 레이블 관리가 선행되어야 한다.

### 문제 2. Taint가 있는데 파드가 배치됨

- **증상**: 특정 GPU 노드에 일반 파드가 스케줄링됨.
- **원인**: Toleration의 key, value, effect가 Taint와 정확히 일치하지 않거나, `operator: Exists`로 설정된 Toleration이 과하게 넓음.
- **해결**: Taint와 Toleration의 key/value/effect를 정확히 맞춤. Toleration을 최소한으로 좁힘.
- **교훈**: Toleration은 통행권이지 필수 배치 보장이 아님. Taint + Node Affinity를 함께 써야 "거부"와 "유인"을 동시에 달성.

### 문제 3. OOM Kill 반복 발생

- **증상**: 특정 파드가 주기적으로 재시작됨. `kubectl describe pod`에 OOM Kill 확인.
- **원인**: limits.memory가 실제 사용량보다 낮게 설정되어 있음.
- **해결**: `kubectl top pod`로 실제 사용량 확인 후 limits 상향 조정. QoS를 Burstable에서 Guaranteed로 전환 검토.
- **교훈**: limits.memory는 실제 피크 사용량 기반으로 설정해야 한다. 평균이 아닌 피크값이 기준.

### 문제 4. HPA가 스케일링하지 않음

- **증상**: CPU 100%인데 파드 수가 늘어나지 않음.
- **원인 1**: Metrics Server 미설치 또는 비정상 동작.
- **원인 2**: 파드에 resources.requests 미설정 → HPA가 사용률(%) 계산 불가.
- **해결**:
  ```bash
  kubectl top pods                            # Metrics Server 정상 여부 확인
  kubectl describe hpa <hpa-name>            # 이벤트 메시지 확인
  kubectl get pod <pod> -o yaml | grep -A5 resources  # requests 설정 확인
  ```
- **교훈**: HPA가 동작하려면 Metrics Server 설치 + resources.requests 설정이 동시에 필요.

### 문제 5. NetworkPolicy 적용 후 정상 트래픽 차단

- **증상**: NetworkPolicy 적용 후 서비스 간 통신 불가.
- **원인**: Default Deny All 정책만 있고 허용 정책이 없거나, namespaceSelector에 해당 네임스페이스 레이블이 없음.
- **해결**:
  ```bash
  kubectl label ns monitoring name=monitoring   # 네임스페이스 레이블 추가
  kubectl get networkpolicies -A               # 전체 정책 확인
  ```
- **교훈**: NetworkPolicy는 명시된 것만 허용. namespaceSelector 사용 시 해당 네임스페이스에 레이블이 반드시 있어야 함.

---

## 7. 실전 명령어 / 치트시트 (Cheat Sheet)

```bash
# ── Node Affinity / 노드 레이블 관리 ──
kubectl label node <node-name> size=large        # 레이블 추가
kubectl label node <node-name> size-             # 레이블 제거
kubectl get nodes --show-labels                  # 노드 레이블 확인

# ── Taint 관리 ──
kubectl taint nodes <node> dedicated=gpu:NoSchedule        # Taint 추가
kubectl taint nodes <node> dedicated=gpu:NoSchedule-       # Taint 제거
kubectl describe node <node> | grep -A5 Taints             # Taint 확인

# ── QoS 확인 ──
kubectl describe pod <pod> | grep "QoS Class"
kubectl get pod <pod> -o jsonpath='{.status.qosClass}'

# ── 리소스 사용량 ──
kubectl top nodes
kubectl top pods
kubectl top pods --containers                    # 컨테이너별 세분화

# ── HPA ──
kubectl get hpa
kubectl describe hpa <hpa-name>
kubectl autoscale deployment <deploy> --cpu-percent=50 --min=1 --max=10

# ── Metrics Server 설치 ──
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl get pods -n kube-system | grep metrics-server

# ── PSA 레이블 확인 ──
kubectl get namespaces --show-labels | grep "pod-security"
kubectl describe namespace <ns>

# ── PSA 레이블 적용 ──
kubectl label namespace <ns> \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest

# ── NetworkPolicy 확인 ──
kubectl get networkpolicies -A
kubectl describe networkpolicy <name>

# ── 스케줄링 이벤트 확인 ──
kubectl describe pod <pod> | grep -A10 Events
kubectl get events --field-selector reason=FailedScheduling
```

---

## 8. 깊게 이해하기 (Deep Dive)

### 8.1 kube-scheduler의 스케줄링 파이프라인

```
[스케줄링 파이프라인]

① Filtering (Filter 플러그인)
   불가능한 노드를 제거하는 단계
   - NodeResourcesFit: requests를 수용할 리소스 없는 노드 제거
   - NodeAffinity: required 조건 불충족 노드 제거
   - TaintToleration: Toleration 없이 Taint가 걸린 노드 제거
   - PodAntiAffinity: hard 조건에 위배되는 노드 제거

② Scoring (Score 플러그인)
   후보 노드들의 점수를 계산하는 단계
   - NodeAffinity: preferred 조건 weight 합산
   - PodAntiAffinity: soft 조건 만족도 점수
   - LeastAllocated: 리소스가 가장 여유 있는 노드 우선
   - ImageLocality: 이미지가 이미 있는 노드 우선

③ Binding
   최고 점수 노드에 파드 바인딩
```

---

### 8.2 Topology Spread Constraints — Anti-Affinity의 발전 (보강)

Anti-Affinity는 "특정 파드와 같은 노드를 피하라"는 단순 조건이다. 실제 분산 수를 제어하려면 **Topology Spread Constraints**가 더 정교하다.

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1                          # 노드 간 파드 수 차이 허용 최대값
    topologyKey: "kubernetes.io/hostname"
    whenUnsatisfiable: DoNotSchedule    # 조건 불충족 시: DoNotSchedule or ScheduleAnyway
    labelSelector:
      matchLabels:
        app: my-app
```

`maxSkew: 1`이면 노드 A에 3개, 노드 B에 4개는 가능하지만 노드 A에 2개, 노드 B에 4개는 불가(차이 = 2 > 1).

Anti-Affinity는 "같은 노드에 1개도 안 된다"는 절대 규칙이고, Topology Spread는 "최대 N개 차이 이내로 분산하라"는 상대 규칙이다.

---

### 8.3 Limits만 설정하고 Requests 생략 시 동작

```
limits만 설정, requests 생략 → Kubernetes가 자동으로 requests = limits 적용

결과:
  QoS 클래스 = Guaranteed
  스케줄러 기준 = limits 값 (보수적 스케줄링)
  → limits를 너무 크게 설정하면 Pending 발생 가능
```

---

### 8.4 PSA — 클러스터 전역 기본값 설정 (AdmissionConfiguration)

레이블 없는 네임스페이스도 보안 정책 적용을 원하면 kube-apiserver에 설정 파일을 지정한다.

```yaml
# /etc/kubernetes/psa-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "baseline"
      enforce-version: "latest"
      warn: "restricted"
      warn-version: "latest"
    exemptions:
      namespaces:
      - kube-system     # 시스템 네임스페이스 제외
      - kube-public
```

`kube-apiserver` 플래그: `--admission-control-config-file=/etc/kubernetes/psa-config.yaml`

---

### 8.5 Prometheus → HPA 연동 (Adapter 필요) (보강)

Prometheus 메트릭(예: 초당 요청 수, 큐 길이)으로 HPA를 구동하려면 **Prometheus Adapter**가 필요하다.

```
Prometheus → Prometheus Adapter → custom.metrics.k8s.io API → HPA

[메트릭 종류별 구성]
CPU/메모리 HPA        : Metrics Server
커스텀 메트릭 HPA     : Prometheus + Prometheus Adapter
외부 이벤트 기반 HPA  : KEDA (Kafka, SQS, Redis 등)
```

---

### 8.6 Control Plane 노드의 기본 Taint

Kubernetes 설치 시 Control Plane 노드에는 기본으로 아래 Taint가 설정된다.

```
node-role.kubernetes.io/control-plane:NoSchedule
```

이 덕분에 일반 파드가 Control Plane 노드에 배치되지 않는다. 단일 노드 환경(Minikube 등)에서는 이 Taint를 제거해야 일반 파드를 배치할 수 있다.

```bash
kubectl taint nodes <node> node-role.kubernetes.io/control-plane:NoSchedule-
```

---

## 9. 나만의 요약 (My Summary)

```
스케줄링 3대 축:

1. "파드가 특정 노드로 가고 싶다" → Node Affinity (레이블 조건)
2. "노드가 특정 파드를 거부한다" → Taint / Toleration (배제 + 통행권)
3. "파드끼리 서로 멀어져야 한다" → Anti-Affinity / Topology Spread

리소스 관리 핵심:
  requests: 스케줄러가 보장하는 최소값 (Filtering 기준)
  limits:   컨테이너가 넘을 수 없는 최대값
  → CPU 초과 = throttle, 메모리 초과 = OOM Kill
  → requests == limits → Guaranteed QoS → 안전하지만 노드 낭비 가능

HPA의 함정:
  Metrics Server 설치 + resources.requests 설정 → 둘 다 없으면 동작 안 함
  EKS는 Metrics Server 기본 미포함 → 별도 설치 필수

PSA의 함정:
  레이블 없으면 아무 제한 없음 (Opt-in)
  처음부터 enforce=restricted → 기존 파드 차단 위험
  → warn → audit → enforce 순으로 단계적 적용
```

**기억할 포인트 3가지:**
1. EKS는 Metrics Server를 기본 제공하지 않는다 (GKE/AKS는 기본 포함)
2. Anti-Affinity `required`는 노드 수 < Replica 수이면 Pending 위험 → `preferred` 검토
3. Taint/Toleration은 "거부 + 통행권" 쌍. 유인이 필요하면 반드시 Node Affinity를 병행

**다음에 헷갈릴 것 같은 부분:**
- nodeSelectorTerms 배열 내부의 matchExpressions는 AND, 배열 간은 OR 관계
- Topology Spread Constraints vs Pod Anti-Affinity 선택 기준 (절대 분리 vs 최대 편차)
- PSA warn/audit/enforce 세 모드 동시 적용 가능 여부 (가능함)

---

## 10. 참고 자료 (References)

| 유형 | 제목 | 링크 | 메모 |
|------|------|------|------|
| 공식 문서 | Node Affinity 공식 가이드 | https://kubernetes.io/ko/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/ | 공식 예제 |
| 공식 문서 | Pod QoS 공식 | https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-qos/ | QoS 기준 상세 |
| 공식 문서 | NetworkPolicy 공식 | https://kubernetes.io/ko/docs/concepts/services-networking/network-policies/ | 셀렉터 조합 |
| 공식 문서 | PSA 공식 | https://kubernetes.io/ko/docs/concepts/security/pod-security-admission/ | GA v1.25 |
| 공식 문서 | HPA Walkthrough | https://kubernetes.io/ko/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/ | 실습 가이드 |
| 블로그 | Kubernetes Anti-Affinity 이해 | https://nayoungs.tistory.com/entry/Kubernetes-k8s-어피니티Affinity와-안티-어피니티Anti-Affinity | Anti-Affinity 상세 |
| 블로그 | Taint & Toleration 실습 | https://velog.io/@pinion7/Kubernetes-Pod-배치전략-Taint와-Toleration에-대해-이해하고-실습해보기 | 실습 중심 |
| AWS 문서 | EKS Metrics Server 설치 | https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/metrics-server.html | EKS 설치 가이드 |
| AWS 문서 | EKS HPA | https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/horizontal-pod-autoscaler.html | EKS HPA 설정 |

---

*이 문서는 학습이 깊어질수록 계속 업데이트합니다.*
