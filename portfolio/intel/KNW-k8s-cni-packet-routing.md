# 지식 정리 (Knowledge Note)

---

## 기본 정보

| 항목 | 내용 |
|------|------|
| **주제** | Kubernetes CNI, 패킷 라우팅, 오버레이 네트워크, EKS VPC CNI |
| **분류** | 인프라 / 네트워크 |
| **키워드** | CNI, DNAT, kube-proxy, iptables, veth pair, cni0, Flannel, VXLAN, VTEP, EKS, VPC CNI, ENI, NLB, ALB, ClusterIP, MetalLB, Ingress, Pod-to-Pod |
| **학습 계기** | 업무 중 필요 — CNI 역할 오해 교정 및 외부 트래픽이 파드까지 전달되는 경로 전반 정리 |
| **관련 업무 ID** | - |
| **최초 작성일** | 2026-04-20 |
| **최종 수정일** | 2026-04-20 |
| **숙련도** | 🌿 이해 |

---

## 1. 한 줄 요약 (TL;DR)

```
CNI는 Pod 생성 시 veth pair·IP·라우팅을 구성하고 끝낸다.
런타임 패킷 전달은 커널(iptables DNAT + 라우팅 테이블)이 담당하며,
EKS VPC CNI는 Pod IP = ENI Secondary IP이므로 오버레이 터널 없이
VPC 라우터가 직접 목적지 노드 ENI까지 전달한다.
```

---

## 2. 배경 지식 (Prerequisites)

- **필수 선행 지식**:
  - Linux 네트워크 네임스페이스, veth pair 개념
  - iptables/Netfilter 기본 (체인, DNAT)
  - AWS ENI(Elastic Network Interface), VPC 라우팅 개념

- **관련 개념과의 관계**:
  ```
  [외부 트래픽 진입 경로]
  외부 클라이언트
    └── LB (MetalLB / AWS NLB / AWS ALB)
          └── Worker Node 커널
                ├── iptables (kube-proxy가 관리)
                │     └── DNAT: ClusterIP → Pod IP
                └── CNI가 구성해 놓은 라우팅
                      └── veth pair → Pod 네트워크 네임스페이스

  [역할 분담]
  CNI         : Pod 생성 시 1회 — veth/IP/라우팅 구성
  kube-proxy  : iptables 룰 관리 — ClusterIP/NodePort DNAT
  커널(런타임): CNI가 만든 라우팅 + kube-proxy iptables 룰로 패킷 전달
  ```

---

## 3. 개념 설명 (Concept)

### 3.1 정의 (What)

```
CNI(Container Network Interface)는 컨테이너 런타임과 네트워크 플러그인
(Flannel, Calico, Cilium, AWS VPC CNI 등) 사이의 표준 인터페이스 규격이다.
kubelet이 Pod를 생성할 때 CNI 플러그인을 호출하고,
플러그인이 Pod의 네트워크 구성(veth, IP, 라우팅)을 담당한다.
```

### 3.2 존재 이유 (Why)

```
컨테이너는 격리된 네트워크 네임스페이스 안에 존재하기 때문에,
기본적으로 외부(호스트, 다른 Pod)와 통신할 수 없다.
CNI가 veth pair로 파드 네임스페이스와 호스트 네임스페이스를 연결하고,
IP를 부여하고, 라우팅 경로를 설정함으로써 Pod 간 / 외부 통신이 가능해진다.
```

### 3.3 동작 원리 (How it works)

#### Pod 생성 시 CNI가 수행하는 작업 (1회성)

```
① kubelet이 Pod 생성 요청
        ↓
② CNI 플러그인 호출
        ↓
③ veth pair 생성
   - 한 끝: Pod 네트워크 네임스페이스 안의 eth0
   - 다른 끝: 호스트 네임스페이스의 vethXXXX
        ↓
④ IPAM: Pod에 고유 IP 할당 (예: 10.244.2.15)
        ↓
⑤ 라우팅 설정
   - Pod 내부: default gateway 설정 (169.254.1.1 또는 cni0)
   - 호스트 라우팅 테이블에 "이 Pod IP → vethXXXX" 경로 추가
        ↓
⑥ 노드 간 경로 전파 (CNI 종류에 따라)
   - Flannel: VXLAN 터널 설정 (노드 간 VTEP 구성)
   - Calico:  BGP로 Pod CIDR 광고
   - EKS VPC CNI: 별도 터널 없음 (VPC 라우팅이 처리)

⚠️ 이후 런타임에 실제 패킷을 전달하는 것은 CNI가 아니라 커널이다.
   CNI가 구성해둔 라우팅 테이블 + iptables를 커널이 활용해 전달한다.
```

#### kube-proxy가 iptables를 관리하는 흐름

```
Pod 생성
  → kubelet이 Pod IP를 kube-apiserver에 업데이트
  → kube-proxy가 Endpoints 변경 감지
  → 각 노드 iptables에 규칙 추가:
      KUBE-SERVICES → KUBE-SVC-XXXX → KUBE-SEP-XXXX
      ClusterIP:Port → (DNAT) → Pod IP:Port
```

#### Pod → Service → Pod 흐름 (클러스터 내부)

```
[발신 Pod]
  ↓ 1. DNS 조회 → CoreDNS → ClusterIP 반환 (예: 10.96.0.50)
  ↓ 2. 패킷 생성 (목적지: 10.96.0.50)

[발신 Pod가 있는 노드 커널]
  ↓ 3. iptables PREROUTING/OUTPUT 체인 탐색
       ClusterIP 10.96.0.50 → DNAT → Pod IP 10.244.2.20
  ↓ 4. 라우팅 결정
       ├── 같은 노드: cni0 브릿지 → veth → 목적지 Pod
       └── 다른 노드: 오버레이 터널(flannel.1 등) → 목적지 노드 → veth → 목적지 Pod

✅ Ingress(lb-controller)는 전혀 관여하지 않는다.
   클러스터 내부 Pod 간 Service 호출은 DNS + iptables DNAT만으로 처리된다.
   ClusterIP는 실제 어떤 인터페이스에도 바인딩되지 않는 가상 IP이다.
```

### 3.4 구성 요소 / 핵심 용어 (Key Terms)

| 용어 | 설명 | 비고 |
|------|------|------|
| **CNI** | Pod 생성 시 네트워크 구성(veth/IP/라우팅)을 담당하는 플러그인 규격 | 런타임 패킷 전달은 담당 안 함 |
| **veth pair** | 두 네트워크 네임스페이스를 연결하는 가상 케이블 | 한 끝: 호스트(vethXXXX), 다른 끝: Pod(eth0) |
| **cni0** | Flannel 등 브릿지 기반 CNI가 만드는 Linux 브릿지 | vethXXXX들을 연결하는 허브 역할 |
| **kube-proxy** | 각 노드에서 iptables/IPVS 룰을 관리하는 컴포넌트 | DNAT 규칙 설치 담당 |
| **DNAT** | 목적지 IP:Port를 변환하는 NAT | ClusterIP → Pod IP 변환 |
| **ClusterIP** | 실제 인터페이스에 바인딩되지 않는 가상 IP | iptables 룰만으로 동작 |
| **VTEP** | VXLAN 터널의 종단점 인터페이스 | Flannel의 flannel.1 |
| **VXLAN** | L2 프레임을 UDP로 캡슐화하는 오버레이 기술 | Flannel이 사용, UDP 8472 포트 |
| **ENI** | AWS EC2의 가상 네트워크 인터페이스 | Secondary IP가 Pod IP로 사용 |
| **MetalLB** | 베어메탈 k8s용 LB — ARP/BGP로 VIP를 특정 노드에 고정 | VIP 보유 노드로 직접 도달 (순회 없음) |

---

## 4. 비교 및 구분 (Comparison)

### 4.1 CNI 방식별 노드 간 통신 비교

| CNI | 노드 간 통신 방식 | 특징 |
|-----|----------------|------|
| **Flannel (VXLAN)** | L2 프레임을 UDP 8472로 캡슐화 → 오버레이 터널 | 브릿지(cni0) + flannel.1(VTEP) |
| **Calico (BGP)** | BGP로 Pod CIDR 광고 → 언더레이 라우팅 직접 전달 | 오버레이 없음, 고성능 |
| **Cilium (eBPF)** | iptables 우회, eBPF 맵으로 처리 | 커널 레벨 최적화 |
| **AWS VPC CNI** | Pod IP = ENI Secondary IP → VPC 라우터가 직접 처리 | 오버레이 없음, 캡슐화 없음 |

### 4.2 일반 k8s(오버레이) vs EKS(VPC CNI) 홉 비교

```
[일반 k8s Flannel — 다른 노드 Pod로 갈 때]
발신 Pod
→ veth → cni0 브릿지        (홉 1: 로컬 브릿지)
→ flannel.1 VTEP (VXLAN 캡슐화)  (홉 2: 캡슐화)
→ 물리 NIC → 물리 네트워크       (홉 3: 언더레이 전송)
→ 목적지 노드 flannel.1 (역캡슐화)(홉 4: 역캡슐화)
→ cni0 브릿지 → veth            (홉 5: 로컬 브릿지)
→ 목적지 Pod

[EKS VPC CNI — 다른 노드 Pod로 갈 때]
발신 Pod
→ veth → 호스트 라우팅    (홉 1: 로컬 veth)
→ VPC 라우터              (홉 2: VPC 라우팅, 캡슐화 없음)
→ 목적지 노드 ENI → veth  (홉 3: 로컬 veth)
→ 목적지 Pod

✅ 결론: EKS가 홉이 적다.
두 방식 모두 물리 네트워크(VPC 라우터)를 거친다.
오버레이는 그 위에 캡슐화 홉을 추가로 얹는 구조이므로 EKS가 더 효율적이다.
```

| 항목 | 일반 k8s 오버레이 | EKS VPC CNI |
|------|-----------------|------------|
| VPC 라우터 경유 | ✅ 동일하게 경유 | ✅ 동일하게 경유 |
| VXLAN 캡슐화/역캡슐화 | ✅ 추가 발생 | ❌ 없음 |
| 브릿지(cni0) 경유 | ✅ 있음 | ❌ 없음 (veth 직접) |
| CPU 오버헤드 | 캡슐화 연산 추가 | 없음 |
| VPC Flow Logs 가시성 | ❌ 노드 IP만 보임 | ✅ Pod IP 그대로 보임 |
| Security Group Pod 단위 적용 | ❌ 불가 | ✅ 가능 |

### 4.3 EKS NLB IP mode vs Instance mode

| 항목 | IP mode | Instance mode |
|------|---------|--------------|
| Target Group 등록 | **Pod IP** 직접 등록 | EC2 인스턴스(NodePort) 등록 |
| 트래픽 경로 | NLB → Pod IP (VPC 라우팅 직접) | NLB → 노드 NodePort → kube-proxy iptables DNAT → Pod |
| iptables 경유 | ❌ 경유 안 함 | ✅ kube-proxy iptables 경유 |
| 불필요한 홉 | 없음 | Pod가 다른 노드에 있으면 추가 홉 발생 |
| Fargate 지원 | ✅ 가능 | ❌ 불가 |

---

## 5. 실전 예시 (Examples)

### 5.1 브릿지 기반 CNI(Flannel) 내부 패킷 흐름

```
[목적지 노드에 패킷 도달 후 Pod까지의 정확한 경로]

❌ 잘못된 이해:
  라우팅 테이블 → vethXXXX → Pod

✅ 올바른 경로 (브릿지 기반 CNI):
  라우팅 테이블 → cni0 (Linux 브릿지)
      ↓ 브릿지가 MAC 주소 기반으로 vethXXXX 선택
  vethXXXX → Pod 네트워크 네임스페이스의 eth0

이유: vethXXXX 자체에는 IP가 없기 때문에
      라우팅 테이블이 직접 veth를 목적지로 지정하지 않는다.
      라우팅은 cni0 브릿지까지, 그 이후는 L2 브릿지가 처리한다.
```

### 5.2 오버레이 터널 구조 (Flannel VXLAN)

```
오버레이 터널은 노드 ↔ 노드 간에 구성된다 (Pod ↔ Pod 간 직접 터널 ❌)

[Node 1]                           [Node 2]
Pod A (10.244.1.10)                Pod C (10.244.2.20)
  eth0 → veth → cni0(브릿지)     cni0(브릿지) → veth → eth0
                    ↓                  ↑
              flannel.1(VTEP)  ←──────── flannel.1(VTEP)
              (UDP 캡슐화)   VXLAN 터널  (UDP 역캡슐화)
                    ↓                  ↑
              eth0 (물리NIC) ─────── eth0 (물리NIC)
                          물리 네트워크

- flannel.1 같은 VTEP 인터페이스가 각 노드에 1개씩 생성됨
- N개 노드면 N×(N-1)개의 논리 경로(완전 메쉬)

[Node 1의 라우팅 테이블 예시]
10.244.2.0/24 via flannel.1   ← "Node 2의 Pod 대역은 flannel.1 터널로"
10.244.1.15/32 via cni0       ← "로컬 Pod는 브릿지로"
```

### 5.3 EKS Pod → Pod 흐름 (VPC CNI)

```
[Pod IP 구조]
Worker Node ENI (Primary IP: 192.168.1.101)
  ├── Secondary IP: 192.168.1.51 → Pod A
  ├── Secondary IP: 192.168.1.52 → Pod B
  └── Secondary IP: 192.168.1.53 → Pod C

[Pod A (10.0.1.15, Node 1) → Pod C (10.0.2.20, Node 2)]

1. Pod A eth0에서 패킷 출발 (목적지: 10.0.2.20)
   └── 내부 라우팅: default via 169.254.1.1 (ARP Proxy, 호스트 veth가 응답)
2. veth → Node 1 호스트 라우팅 테이블 확인
   └── "10.0.2.20은 로컬에 없음 → default gateway(VPC 라우터)로"
3. VPC 라우터가 10.0.2.20 확인
   └── "10.0.2.20은 Node 2의 ENI eni-xxx의 Secondary IP" → Node 2의 ENI로 전달
4. Node 2 수신
   └── 로컬 라우팅: "10.0.2.20/32 dev eniY" → veth → Pod C eth0 도착

캡슐화 없음, iptables DNAT 없음 (VPC 라우팅만 사용)
Pod는 목적지 Pod IP만 알면 된다. 어느 노드에 있는지 몰라도 된다.
```

### 5.4 외부 트래픽 흐름 전체 비교

#### MetalLB + Ingress(nginx) 흐름

```
외부 클라이언트
→ MetalLB VIP (ARP로 특정 노드에 고정 — 순회 없음)
→ 해당 노드 커널 iptables (NodePort 또는 ClusterIP 규칙)
→ ingress-nginx Pod (L7: URI/헤더 파싱 및 라우팅 결정)
  └── ingress-nginx가 새로운 TCP 커넥션 시작 (L7 프록시)
→ 매칭된 Service의 ClusterIP로 재전달
→ 커널 iptables: ClusterIP → Pod IP DNAT
→ CNI가 구성한 라우팅으로 목적지 노드 전달
→ 목적지 노드: cni0 브릿지 → veth → Pod 네임스페이스

⚠️ iptables는 L3/L4만 다룬다.
   L7 라우팅(URI, 헤더)은 ingress-nginx Pod 안에서만 처리된다.
```

#### EKS ALB IP mode 흐름

```
외부 클라이언트
→ AWS ALB (L7: TLS 종료, URI 라우팅)
→ Target Group에 등록된 Pod IP로 직접 전달
→ AWS VPC 라우터: Pod IP(= ENI Secondary IP) 보유 노드의 ENI로 전달
→ 노드 커널: Secondary IP → veth → Pod 네트워크 네임스페이스
  (iptables DNAT 불필요, CNI가 구성한 로컬 라우팅만 사용)
```

---

## 6. 트러블슈팅 경험 (Troubleshooting)

### 문제 1. "CNI가 런타임에 패킷을 전달한다"는 오해

- **증상**: "DNAT 이후 패킷을 목적지 Pod까지 전달하는 게 CNI의 역할"로 잘못 이해
- **원인**: CNI의 역할과 커널(iptables + 라우팅)의 역할을 혼동
- **해결**:
  ```
  역할 구분:
  CNI      → Pod 생성 시 1회 호출 → veth/IP/라우팅 구성 후 종료
  kube-proxy → iptables 룰 관리 (DNAT 규칙 설치)
  커널(런타임) → CNI가 만든 라우팅 + kube-proxy iptables로 패킷 전달

  DNAT는 kube-proxy(iptables/IPVS)의 역할이지 CNI의 역할이 아니다.
  ```
- **교훈**: CNI는 "준비"를 담당하고, 커널이 "실행"을 담당한다.

### 문제 2. "라우팅 테이블 → vethXXXX → Pod"로 잘못 이해

- **증상**: Flannel 같은 브릿지 기반 CNI에서 패킷이 라우팅 테이블에서 직접 veth로 간다고 오해
- **원인**: vethXXXX에 IP가 없다는 사실을 몰라서 발생
- **해결**:
  ```
  정확한 경로:
  라우팅 테이블 → cni0(Linux 브릿지) → vethXXXX → Pod eth0

  vethXXXX에는 IP가 없다.
  라우팅 테이블은 브릿지(cni0)까지만 안내하고,
  브릿지가 L2(MAC 주소) 기반으로 올바른 veth를 선택한다.
  ```
- **교훈**: L3 라우팅은 브릿지까지, 그 이후는 L2 브릿지의 MAC 학습이 처리한다.

### 문제 3. MetalLB 트래픽이 "노드를 순회한다"는 오해

- **증상**: MetalLB가 임의 워커 노드를 순회하면서 lb-controller 파드를 찾는다고 오해
- **원인**: MetalLB의 ARP/BGP 동작 방식 미이해
- **해결**:
  ```
  MetalLB는 VIP를 특정 노드의 ARP 응답으로 고정한다.
  → 클라이언트 요청이 VIP 보유 노드로 직접 전달된다 (순회 없음)
  ```
- **교훈**: MetalLB = "이 IP는 내 것" ARP 선언. L4 LB처럼 여러 노드를 프로브하지 않는다.

### 문제 4. "EKS가 일반 k8s보다 홉이 1개 더 생긴다"는 오해

- **증상**: EKS는 VPC 라우터를 경유하니 홉이 더 많다고 생각
- **원인**: 일반 k8s도 물리 네트워크(VPC 라우터와 동등)를 경유한다는 사실을 놓침
- **해결**:
  ```
  두 방식 모두 물리 네트워크(VPC 라우터에 해당하는 레이어)를 거친다.
  오버레이(Flannel)는 그 위에 VXLAN 캡슐화/역캡슐화 홉을 추가로 얹는다.
  → EKS VPC CNI가 실제로 홉이 더 적고 CPU 오버헤드도 없다.
  ```
- **교훈**: 오버레이 = "언더레이 위에 가상 레이어 추가". EKS는 언더레이(VPC)를 그대로 활용한다.

---

## 7. 실전 명령어 / 치트시트 (Cheat Sheet)

```bash
# ─────────────────────────────────────────────
# [CNI 구성 확인]
# ─────────────────────────────────────────────

# 노드의 라우팅 테이블 확인 (Pod IP → veth/브릿지 매핑)
ip route show

# 브릿지(cni0) 포트 목록 확인 (Flannel)
bridge link show

# veth pair 확인
ip link show type veth

# Pod 네트워크 네임스페이스에서 라우팅 확인
kubectl exec -n <ns> <pod> -- ip route
kubectl exec -n <ns> <pod> -- ip addr

# ─────────────────────────────────────────────
# [iptables DNAT 룰 확인 (kube-proxy)]
# ─────────────────────────────────────────────

# KUBE-SERVICES 체인: ClusterIP 진입점
sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers

# 특정 서비스의 DNAT 룰 확인
sudo iptables -t nat -L KUBE-SVC-XXXX -n

# Endpoint(SEP) 별 DNAT 룰
sudo iptables -t nat -L KUBE-SEP-XXXX -n

# 전체 iptables NAT 규칙 수 (규모 파악)
sudo iptables -t nat -L | wc -l

# ─────────────────────────────────────────────
# [Flannel VXLAN 터널 확인]
# ─────────────────────────────────────────────

# flannel.1 VTEP 인터페이스 확인
ip link show flannel.1
ip addr show flannel.1

# VXLAN ARP/FDB 테이블 (어느 노드에 어느 Pod가 있는지)
bridge fdb show dev flannel.1
ip neigh show dev flannel.1

# ─────────────────────────────────────────────
# [EKS VPC CNI 확인]
# ─────────────────────────────────────────────

# 노드의 ENI Secondary IP 목록 (Pod IP 범위)
aws ec2 describe-network-interfaces \
  --filters "Name=attachment.instance-id,Values=<instance-id>" \
  --query 'NetworkInterfaces[*].PrivateIpAddresses[*].PrivateIpAddress' \
  --output table

# 파드 IP가 ENI Secondary IP인지 확인
kubectl get pods -o wide -A | grep <node-name>

# ─────────────────────────────────────────────
# [NLB Target Group 상태 확인]
# ─────────────────────────────────────────────

# Target Group 헬스 상태 확인
aws elbv2 describe-target-health \
  --target-group-arn <tg-arn> \
  --query 'TargetHealthDescriptions[*].{Target:Target.Id,Port:Target.Port,Health:TargetHealth.State}'

# aws-load-balancer-controller 로그 확인
kubectl logs -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller \
  --tail=50
```

---

## 8. 깊게 이해하기 (Deep Dive)

### 8.1 Flannel 브릿지 기반 CNI의 패킷 전달 레이어별 처리

```
[L3 라우팅이 담당하는 영역]
목적지 Pod IP → "10.244.1.0/24 dev cni0" 경로로 cni0 브릿지로 전달

[L2 브릿지가 담당하는 영역]
cni0 브릿지가 MAC 주소 테이블(FDB) 기반으로 올바른 vethXXXX 선택
→ ARP로 Pod eth0의 MAC 주소를 학습해 FDB에 등록

[패킷 흐름 요약]
커널 L3 라우팅 → cni0(L2 브릿지, IP 없음) → vethXXXX(IP 없음) → Pod eth0(IP 있음)

vethXXXX에 IP가 없는 이유:
  veth는 "케이블"일 뿐이다. IP는 인터페이스에 부여하는 것이고,
  케이블 역할의 veth에는 IP가 불필요하다.
  cni0 브릿지가 스위치 역할을 하며 MAC으로 포워딩 결정을 내린다.
```

### 8.2 EKS VPC CNI의 169.254.1.1 ARP Proxy

```
[Pod 내부 라우팅 테이블]
default via 169.254.1.1 dev eth0

169.254.1.1은 실제로 존재하지 않는 링크-로컬 주소다.
Pod가 이 주소로 ARP 요청을 보내면,
호스트의 veth(Pod 측 대응 veth)가 ARP Proxy로 응답한다.
→ Pod는 항상 "게이트웨이 MAC = 호스트 veth MAC"으로 패킷을 보내고,
  호스트 라우팅 테이블이 이후 경로를 결정한다.

이 구조 덕분에 Pod는 목적지 Pod IP만 알면 되고,
어느 노드에 있는지 전혀 몰라도 된다.
```

### 8.3 ClusterIP의 정체

```
ClusterIP는 어떤 인터페이스에도 바인딩되지 않는 가상 IP다.

ping 10.96.0.50 (ClusterIP)  → 응답 없음 (정상)
curl http://10.96.0.50:80    → 정상 동작

이유:
kube-proxy가 설치한 iptables 룰이 ClusterIP로 향하는 패킷을
L3 라우팅 이전에 인터셉트해서 실제 Pod IP로 DNAT한다.
패킷은 ClusterIP를 목적지로 네트워크에 실제로 나가지 않는다.
DNAT 이후에는 실제 Pod IP로 라우팅된다.
```

### 8.4 Ingress의 역할 경계

```
[외부 트래픽 흐름에서 Ingress가 관여하는 구간]

외부 → LB → Worker Node (L4 도달) → ingress-nginx Pod
                                       ↑ 여기까지는 일반 iptables DNAT
                                       ↓
                                    L7 파싱 (URI, Host 헤더 등)
                                    새 TCP 커넥션 시작 → 백엔드 Service로 전달
                                       ↓
                                    커널 iptables DNAT (ClusterIP → Pod IP)
                                       ↓
                                    목적지 Pod

[Ingress가 관여하지 않는 구간]
Pod → Service → Pod (클러스터 내부 통신)
→ DNS + iptables DNAT만으로 처리, Ingress 전혀 없음

핵심: iptables(kube-proxy) = L3/L4 처리
      Ingress(nginx) = L7 처리 (새 커넥션으로 프록시)
```

---

## 9. 나만의 요약 (My Summary)

```
CNI는 Pod를 만들 때 딱 한 번 불린다. 그리고 뭘 만드느냐 하면
"Pod eth0 ↔ 호스트 vethXXXX" 케이블(veth pair),
IP 주소, 그리고 라우팅 테이블 항목이다.
그 이후엔 CNI가 할 일이 없다. 런타임에 패킷을 전달하는 건 커널이다.

패킷 전달 역할 분담을 외워두면 좋다:
  - kube-proxy → iptables 룰 관리 (ClusterIP→Pod IP DNAT)
  - CNI → 구조(veth/IP/라우팅) 설치
  - 커널 → 두 가지를 합쳐서 실제 전달

EKS VPC CNI의 핵심은 "Pod IP = ENI Secondary IP"라는 점 하나다.
이것 때문에 VPC 라우터가 어느 ENI에 Pod가 있는지 그냥 알고 있고,
오버레이(VXLAN) 없이 VPC 라우팅으로 직접 전달된다.
오버레이는 "언더레이 위에 가상 레이어를 한 겹 추가하는 것"이라
캡슐화 오버헤드가 생기는데, EKS는 이게 없다.

홉 개수 오해: EKS도 VPC 라우터를 거치지만, Flannel도 물리 네트워크를 거친다.
Flannel은 거기에 VXLAN 캡슐화 홉을 추가로 얹는다. EKS가 더 적다.
```

**기억할 포인트 6가지:**
1. CNI = Pod 생성 시 구조 설치, 런타임 패킷 전달 = 커널(iptables + 라우팅)
2. Flannel 브릿지 방식: 라우팅 → cni0 브릿지 → veth → Pod (veth에는 IP 없음)
3. 오버레이 터널은 노드↔노드 간 구성, Pod↔Pod 간 직접 터널 아님
4. ClusterIP는 가상 IP — 어떤 인터페이스에도 바인딩 안 됨, iptables가 인터셉트
5. EKS VPC CNI: Pod IP = ENI Secondary IP → VPC 라우터가 직접 처리, VXLAN 없음
6. Ingress는 "외부→클러스터" 진입 L7만 처리, 클러스터 내부 Pod 간 Service 호출엔 무관

**다음에 헷갈릴 것 같은 부분:**
- vethXXXX에 IP가 없는 이유 — veth는 케이블, L3 라우팅은 브릿지(cni0)까지만
- EKS 169.254.1.1 ARP Proxy — Pod 내 게이트웨이이지만 실제로 존재하지 않는 주소, 호스트 veth가 응답
- MetalLB 동작 — VIP 보유 노드 1개로 직접 도달, "노드 순회" 없음
- NLB IP mode에서는 iptables DNAT를 거치지 않음 (Instance mode는 거침)
- Pod → Service 호출에서 DNAT는 발신 노드 커널에서 즉시 처리, 패킷이 ClusterIP를 목적지로 실제 전송되지 않음

---

## 10. 참고 자료 (References)

| 유형 | 제목 | 링크 | 메모 |
|------|------|------|------|
| 공식 문서 | CNI Specification | https://github.com/containernetworking/cni | CNI 표준 규격 |
| 공식 문서 | AWS VPC CNI | https://docs.aws.amazon.com/eks/latest/best-practices/vpc-cni.html | EKS VPC CNI Best Practice |
| 공식 문서 | EKS NLB Network Load Balancing | https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html | NLB IP/Instance mode |
| 블로그 | Kubernetes Pod Networking Model | https://oneuptime.com/blog/post/2026-02-20-kubernetes-pod-networking-model/view | Pod 네트워킹 전반 |
| 블로그 | Flannel CNI 동작 원리 | https://techblog.ahnlabcloudmate.com/flannel-cni/ | Flannel VXLAN 구조 |
| 블로그 | EKS Networking Deep Dive | https://nxgcloud.com/2026/03/20/eks-networking-deep-dive-series-part-2-packet-journey/ | EKS 패킷 흐름 상세 |
| 공식 문서 | Kubernetes ClusterIP Allocation | https://kubernetes.io/docs/concepts/services-networking/cluster-ip-allocation/ | ClusterIP 가상 IP 개념 |

---

*이 문서는 학습이 깊어질수록 계속 업데이트합니다.*
