# 지식 정리 — Kubernetes 시크릿 관리: ESO vs CSI Driver

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **주제** | Kubernetes 외부 시크릿 관리 — External Secrets Operator(ESO) vs AWS Secrets Store CSI Driver(ASCP) |
| **분류** | 인프라 / 플랫폼 / 보안 |
| **키워드** | Kubernetes Secret, ESO, External Secrets Operator, CSI Driver, ASCP, AWS Secrets Manager, KMS, etcd, IRSA, secretObjects, 봉투 암호화 |
| **학습 계기** | EKS 환경에서 시크릿 관리 방식 도입 검토 |
| **최초 작성일** | 2026-04-09 |
| **최종 수정일** | 2026-04-09 |
| **숙련도** | 🌿 이해 |

---

## 1. 한 줄 요약 (TL;DR)

```
ESO는 외부 시크릿을 K8s Secret 오브젝트로 '동기화'하는 오퍼레이터고,
CSI Driver는 Pod에 직접 '파일로 마운트'하여 K8s Secret 오브젝트 자체를 만들지 않는다.
EKS + KMS 환경에서는 ESO의 etcd 보안 단점이 대부분 해소되지만,
'K8s Secret 오브젝트 자체가 존재하지 않아야 한다'는 요건이 있다면 CSI Driver가 유일한 답이다.
```

---

## 2. 배경 지식 (Prerequisites)

- **필수 선행 지식**:
  - Kubernetes Secret 오브젝트 개념
  - etcd (K8s 클러스터 상태 저장소) 역할
  - AWS Secrets Manager / IAM / IRSA 기본 개념
  - Pod 볼륨 마운트 개념

- **관련 개념 위치**:
  ```
  [AWS Secrets Manager]
        │
        ├── ESO (External Secrets Operator)
        │     └── K8s Secret 오브젝트 생성 → etcd 저장 → Pod 환경변수/볼륨으로 주입
        │
        └── ASCP (AWS Secrets Store CSI Driver)
              └── 노드 tmpfs에 직접 마운트 → Pod 볼륨으로 노출 (etcd 경유 없음)

  [EKS etcd]
        └── KMS CMK로 봉투 암호화(Envelope Encryption) 적용 가능
  ```

---

## 3. 개념 설명 (Concept)

### 3.1 External Secrets Operator (ESO)

**정의**
외부 시크릿 저장소(AWS Secrets Manager, HashiCorp Vault, GCP, Azure 등)에서 값을 가져와 **네이티브 K8s Secret 오브젝트로 동기화**하는 오픈소스 오퍼레이터.

**핵심 CRD**

| CRD | 역할 |
|-----|------|
| `SecretStore` | 특정 네임스페이스 내에서 접근할 외부 백엔드 연결 정보 정의 |
| `ClusterSecretStore` | 클러스터 전체에서 접근 가능한 공유 백엔드 연결 정보 |
| `ExternalSecret` | 어떤 백엔드에서, 어떤 키를, 어떤 K8s Secret으로 동기화할지 선언 |

**동작 흐름**
```
[ExternalSecret 정의]
      ↓ refreshInterval 주기마다
[ESO 오퍼레이터가 외부 백엔드 조회]
      ↓
[K8s Secret 오브젝트 생성/갱신 → etcd 저장]
      ↓
[Pod가 env: / volume으로 참조]
```

**주요 특성**
- 50개 이상의 외부 백엔드 지원 (멀티 클라우드 친화적)
- `refreshInterval` 설정으로 외부 변경사항 자동 반영
- GitOps(ArgoCD 등) 워크플로우와 자연스럽게 통합
- K8s Secret 오브젝트가 생성되므로 `env:`/`envFrom:` 모두 사용 가능

---

### 3.2 AWS Secrets Store CSI Driver (ASCP)

**정의**
AWS Secrets Manager의 시크릿을 **Pod의 파일 시스템(볼륨)으로 직접 마운트**하는 방식. K8s Secret 오브젝트를 생성하지 않으며, 데이터는 노드의 `tmpfs`(메모리 기반 가상 파일시스템)에만 존재한다.

**핵심 CRD**

| CRD | 역할 |
|-----|------|
| `SecretProviderClass` | 어떤 시크릿을 어떤 경로에 마운트할지 정의 |

**동작 흐름**
```
[Pod 생성 요청]
      ↓
[CSI DaemonSet이 해당 노드에서 직접 Secrets Manager 호출 (IRSA)]
      ↓
[노드 tmpfs에 시크릿 값 파일로 기록]
      ↓
[Pod 볼륨으로 마운트]
      ↓
[Pod 종료 시 tmpfs에서 즉시 소멸]
```

**주요 특성**
- K8s API 서버 / etcd 경유 없음 → K8s Secret 오브젝트 미생성
- IRSA(IAM Roles for Service Accounts) 기반 접근 제어
- `rotation reconciler` 기능으로 시크릿 자동 갱신
- 2025년 11월 EKS 공식 애드온으로 GA (수명주기 관리 용이)

---

### 3.3 EKS etcd와 KMS 봉투 암호화

**KMS가 정확히 하는 일 — 봉투 암호화(Envelope Encryption)**

KMS는 키가 키를 암호화하는 이중 구조를 사용한다.

```
[Secret 평문]
      ↓ DEK(Data Encryption Key)로 암호화  ← API 서버가 1회용 DEK 생성
[암호화된 Secret 데이터] → etcd 저장

[DEK]
      ↓ KEK(Key Encryption Key)로 암호화  ← AWS KMS CMK(Customer Managed Key)
[암호화된 DEK] → API 서버 캐시
```

단계별 설명:
1. API 서버 시작 시 DEK 시드를 생성, KMS CMK(KEK)로 암호화하여 캐싱
2. Secret 저장 시 KDF(Key Derivation Function)로 1회용 DEK를 파생
3. DEK로 Secret 평문 암호화 후 etcd에 저장
4. etcd에는 암호화된 데이터만 존재 → CMK 없이는 복호화 불가

**→ etcd나 EBS 볼륨이 물리적으로 탈취되어도 KMS CMK 없이는 해독 불가**

**EKS + KMS가 해소하는 것 vs 남는 차이**

| 구분 | EKS + KMS로 해소됨 | 여전히 차이 있음 |
|------|-------------------|----------------|
| etcd at-rest 탈취 | ✅ CMK 없이 복호화 불가 | - |
| EBS 물리 탈취 | ✅ 동일하게 암호화 | - |
| kubectl get secret | ❌ RBAC 실수 시 평문 노출 가능 | CSI는 오브젝트 자체 없음 |
| K8s Secret API 오브젝트 존재 | ❌ 여전히 존재 | CSI는 오브젝트 미생성 |
| RBAC 실수로 인한 노출 | ❌ ClusterRoleBinding 실수 위험 | CSI는 노출 대상 없음 |

---

## 4. 비교 및 구분 (Comparison)

### 4.1 ESO vs CSI Driver 전체 비교

| 구분 | ESO | CSI Driver (ASCP) |
|------|-----|-------------------|
| **동작 방식** | 외부 → K8s Secret 오브젝트 동기화 | 외부 → Pod tmpfs 직접 마운트 |
| **K8s Secret 생성** | 항상 생성 | 기본 없음 (`secretObjects` 옵션 시 생성) |
| **etcd 저장** | 있음 (KMS 적용 시 암호화) | 없음 |
| **환경변수(env:) 지원** | ✅ 자연스럽게 지원 | ❌ 기본 불가 (`secretObjects` 필요) |
| **자동 갱신** | `refreshInterval` 설정 | `rotation reconciler` |
| **멀티 클라우드** | ✅ 50개 이상 백엔드 | ❌ AWS 전용 (ASCP 기준) |
| **GitOps 통합** | ✅ 우수 | △ 제한적 |
| **EKS 애드온** | ❌ | ✅ (2025.11 GA) |
| **런타임 공격 표면** | K8s Secret 오브젝트 노출 가능 | 파일만 존재, kubectl 조회 불가 |
| **운영 복잡도** | 낮음 | 앱이 파일 읽도록 수정 필요 |

### 4.2 언제 무엇을 선택해야 하는가

```
EKS + KMS 환경에서 일반 워크로드
  → ESO 권장 (운영 편의성, 멀티 백엔드, GitOps 통합)

다음 중 하나라도 해당하면 CSI Driver 검토:
  - 컴플라이언스상 K8s API에 Secret 오브젝트가 존재하면 안 됨
  - 멀티테넌트 환경에서 RBAC 실수 가능성을 원천 제거하고 싶음
  - 보안 감사(Audit)에서 etcd Secret 오브젝트 존재 자체를 문제로 지적
  - PCI-DSS, HIPAA 등 고강도 컴플라이언스 요건

혼용 전략 (실무):
  일반 앱      → ESO
  보안 민감 앱 → CSI Driver
```

---

## 5. 실전 예시 (Examples)

### 5.1 ESO 기본 구성

```yaml
# 1. SecretStore: 어떤 AWS 계정/리전의 Secrets Manager를 쓸지
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-store
  namespace: my-app
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-northeast-2
      auth:
        jwt:
          serviceAccountRef:
            name: my-app-sa  # IRSA가 연결된 ServiceAccount

---
# 2. ExternalSecret: 어떤 시크릿을, 어떤 K8s Secret으로 동기화할지
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-db-secret
  namespace: my-app
spec:
  refreshInterval: 1h          # 1시간마다 외부와 동기화
  secretStoreRef:
    name: aws-secrets-store
    kind: SecretStore
  target:
    name: my-db-k8s-secret     # 생성될 K8s Secret 이름
  data:
    - secretKey: db-password   # K8s Secret의 키 이름
      remoteRef:
        key: prod/my-app/db    # Secrets Manager의 경로
        property: password     # JSON 시크릿 내 특정 필드
```

**→ 결과**: `my-db-k8s-secret`이라는 K8s Secret이 생성되고, 1시간마다 Secrets Manager 값으로 갱신됨.

---

### 5.2 CSI Driver 기본 구성

```yaml
# 1. SecretProviderClass: 어떤 시크릿을 어디서 가져올지
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: my-app-secrets
  namespace: my-app
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/my-app/db"   # Secrets Manager 경로
        objectType: "secretsmanager"
        jmesPath:
          - path: password
            objectAlias: db-password    # 마운트될 파일명

---
# 2. Pod: 볼륨으로 마운트
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: my-app-sa   # IRSA 연결된 SA
  containers:
    - name: app
      image: my-app:latest
      volumeMounts:
        - name: secrets-vol
          mountPath: "/mnt/secrets"   # 이 경로에 파일로 마운트됨
          readOnly: true
  volumes:
    - name: secrets-vol
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "my-app-secrets"
```

**→ 결과**: `/mnt/secrets/db-password` 파일이 생성됨. K8s Secret 오브젝트는 미생성.

---

### 5.3 CSI Driver — secretObjects 옵션 (K8s Secret 병행 생성)

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: my-app-secrets
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/my-app/db"
        objectType: "secretsmanager"
        jmesPath:
          - path: password
            objectAlias: db-password
  # ← 이 블록이 있을 때만 K8s Secret도 함께 생성됨
  secretObjects:
    - secretName: my-db-k8s-secret   # 생성될 K8s Secret 이름
      type: Opaque
      data:
        - objectName: db-password
          key: password
```

> ⚠️ **주의**: `secretObjects`를 선언해도 Pod에 볼륨 마운트가 먼저 되어야만 K8s Secret이 생성됨. Pod가 없으면 Secret도 없음.

---

### 5.4 주의해야 할 패턴 (Anti-pattern)

```yaml
# ❌ 고카디널리티 값을 SecretStore label로 사용
# (Loki 라벨 카디널리티와 같은 원리)
# ESO의 label에 동적 값(trace ID 등)을 넣는 경우 → 사용 안 함
```

```yaml
# ❌ CSI Driver를 쓰면서 secretObjects로 모든 시크릿에 K8s Secret 생성
# CSI Driver의 핵심 보안 원칙(K8s Secret 미생성)이 희석됨
secretObjects:
  - secretName: every-single-secret   # 남용 금지
```

```yaml
# ✅ secretObjects는 꼭 필요한 경우에만 제한적으로 사용
# 사용 기준:
# - 앱이 env: / envFrom: 방식만 지원하는 경우
# - Ingress TLS, imagePullSecrets 등 K8s 플랫폼이 직접 Secret을 요구하는 경우
# - 레거시 앱 마이그레이션 중간 단계 (장기적으로 제거 목표)
```

---

## 6. 트러블슈팅 경험 (Troubleshooting)

### 문제 1. CSI Driver — Pod는 뜨는데 시크릿 파일이 없는 경우

- **증상**: Pod가 Running 상태지만 `/mnt/secrets/` 경로에 파일 없음
- **원인**: IRSA 설정 누락 또는 IAM Policy 권한 부족
- **확인**:
  ```bash
  # CSI 드라이버 로그 확인
  kubectl logs -n kube-system -l app=secrets-store-csi-driver

  # Pod의 ServiceAccount에 IRSA가 연결됐는지 확인
  kubectl get sa <SA_NAME> -n <NAMESPACE> -o jsonpath='{.metadata.annotations}'
  ```
- **교훈**: IRSA 연결 여부와 IAM Policy(`secretsmanager:GetSecretValue`)를 먼저 확인

---

### 문제 2. ESO — ExternalSecret이 SecretSynced가 되지 않는 경우

- **증상**: `kubectl get externalsecret` 결과 STATUS가 `SecretSyncedError`
- **원인**: SecretStore의 인증 정보 오류 또는 Secrets Manager 경로 오타
- **확인**:
  ```bash
  kubectl describe externalsecret <NAME> -n <NAMESPACE>
  # Events 섹션에서 오류 메시지 확인
  ```
- **교훈**: `remoteRef.key` 경로는 AWS 콘솔에서 정확한 시크릿 이름을 복사해 사용

---

## 7. 실전 명령어 / 치트시트 (Cheat Sheet)

```bash
# ESO 관련
# ExternalSecret 동기화 상태 확인
kubectl get externalsecret -n <NAMESPACE>

# 수동으로 즉시 동기화 트리거 (annotation 방식)
kubectl annotate externalsecret <NAME> -n <NAMESPACE> \
  force-sync=$(date +%s) --overwrite

# SecretStore 연결 상태 확인
kubectl get secretstore -n <NAMESPACE>

# ESO 오퍼레이터 로그 확인
kubectl logs -n external-secrets -l app.kubernetes.io/name=external-secrets

# ---

# CSI Driver 관련
# SecretProviderClass 목록 확인
kubectl get secretproviderclass -n <NAMESPACE>

# CSI Driver DaemonSet 상태 확인
kubectl get daemonset -n kube-system secrets-store-csi-driver

# 특정 노드의 CSI Driver 로그 확인
kubectl logs -n kube-system -l app=secrets-store-csi-driver --field-selector spec.nodeName=<NODE>

# Pod에 마운트된 시크릿 확인 (Pod 내부)
ls -la /mnt/secrets/
cat /mnt/secrets/db-password

# ---

# KMS 봉투 암호화 설정 확인 (EKS)
aws eks describe-cluster --name <CLUSTER_NAME> \
  --query "cluster.encryptionConfig"
```

---

## 8. 깊게 이해하기 (Deep Dive)

### 8.1 KMS 봉투 암호화 계층 구조

```
레이어 1: EBS 볼륨 암호화 (디스크 레벨)
  └── AWS 관리형 KMS 키 또는 CMK로 EBS 볼륨 자체 암호화

레이어 2: etcd 봉투 암호화 (Secret 오브젝트 레벨)
  └── DEK(Data Encryption Key): API 서버가 생성, Secret 데이터 암호화
  └── KEK(Key Encryption Key): KMS CMK, DEK를 암호화

→ 레이어 1만 있으면: etcd 파일이 노출되어도 EBS 키 없이 못 읽음
→ 레이어 2 추가 시: API 서버 메모리에서 복호화된 Secret도 
  "저장된 상태"에서는 CMK 없이 복호화 불가
  (단, API 서버가 살아있는 동안 메모리 내 복호화 상태는 별개)
```

### 8.2 CSI Driver tmpfs의 보안 특성

```
tmpfs = 메모리 기반 가상 파일시스템
  - 디스크에 기록되지 않음 → 디스크 포렌식으로 복구 불가
  - Pod 종료 시 즉시 소멸 → 흔적 없음
  - 마운트된 Pod 이외에서는 접근 불가
  - kubectl exec로 다른 Pod에서 접근 불가
```

### 8.3 secretObjects의 생명주기 특성 (ESO와의 차이)

```
ESO:
  - K8s Secret이 생성되면 ExternalSecret이 삭제되어도 K8s Secret은 남을 수 있음
  - 클러스터에 영속적으로 존재

CSI Driver + secretObjects:
  - SecretProviderClass를 참조하는 Pod가 존재하는 동안만 K8s Secret 존재
  - Pod 전체 종료 시 K8s Secret도 삭제됨 (생명주기 연동)
  → ESO보다는 노출 기간이 제한적
```

### 8.4 버전별 주요 변경

| 버전/시기 | 변경 내용 |
|-----------|-----------|
| 2025년 11월 | AWS Secrets Store CSI Driver, EKS 공식 애드온 GA |
| K8s 1.28+ | CMK 미설정 시에도 AWS 관리형 키로 기본 봉투 암호화 적용 |

---

## 9. 나만의 요약 (My Summary)

```
핵심 비유:
  ESO  = 외부 금고에서 복사본을 꺼내 회사 금고(etcd)에 넣는 방식
         → 회사 금고가 털리면 노출 위험 (단, KMS로 잠금)
  CSI  = 외부 금고에서 꺼낸 내용물을 즉석에서 책상 위에 올려놓고 사용 후 바로 폐기
         → 회사 금고에 복사본이 아예 없음

EKS + KMS를 쓰면:
  "회사 금고는 매우 튼튼하게 잠겨 있으므로" ESO도 충분히 안전.
  다만 "금고 자체가 없어야 한다"는 규정이 있다면 CSI만 가능.
```

**기억할 포인트 3가지:**
1. CSI Driver의 핵심 강점은 **K8s Secret 오브젝트 자체가 존재하지 않는다**는 것 — `kubectl get secret`으로 조회할 것이 없음
2. KMS 봉투 암호화는 **저장(at-rest) 계층 보호** — 런타임에서 API로 Secret 오브젝트를 읽는 것은 별개 문제
3. `secretObjects`는 **환경변수가 꼭 필요한 앱, K8s 플랫폼이 직접 Secret을 요구하는 경우에만** 제한적으로 사용

**다음에 헷갈릴 것 같은 부분:**
- secretObjects를 쓰면 CSI Driver도 K8s Secret을 만든다 — 단, Pod가 살아있는 동안만 존재
- KMS가 있어도 `kubectl get secret -o yaml`로 평문 노출 가능 — KMS는 etcd at-rest 보호이고, API 접근 제어는 RBAC의 영역
- CSI Driver는 파일 마운트 방식이므로 앱이 환경변수가 아닌 파일을 읽도록 설계되어야 함

---

## 10. 참고 자료 (References)

| 유형 | 제목 | 링크 |
|------|------|------|
| 공식 문서 | AWS EKS - Secrets 관리 | https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/manage-secrets.html |
| 공식 문서 | AWS EKS - 봉투 암호화 | https://docs.aws.amazon.com/eks/latest/userguide/envelope-encryption.html |
| 공식 문서 | Secrets Store CSI Driver 공식 | https://secrets-store-csi-driver.sigs.k8s.io |
| 공식 문서 | AWS EKS 애드온 CSI Driver GA | https://aws.amazon.com/about-aws/whats-new/2025/11/amazon-eks-add-ons-aws-secrets-store-csi-driver-provider/ |
| 블로그 | ESO vs CSI Driver 비교 | https://www.kubeblog.com/kubernetes/secrets-store-csi-driver-vs-external-secrets-operator/ |
| 블로그 | EKS에서 시크릿 관리 선택기 | https://sienna1022.tistory.com/entry/k8s에서-Secret-관리는-어떻게-하는게-좋을까-External-Secret-Operator-선택기 |
| AWS 보안 블로그 | KMS EKS 봉투 암호화 | https://aws.amazon.com/blogs/containers/using-eks-encryption-provider-support-for-defense-in-depth/ |
| AWS 보안 가이드 | EKS 데이터 암호화 베스트 프랙티스 | https://aws.github.io/aws-eks-best-practices/ko/security/docs/data/ |

---

*이 문서는 학습이 깊어질수록 계속 업데이트합니다.*
