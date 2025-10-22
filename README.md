# svc-gateway
**svc-gateway**는 거래소 플랫폼의 **Public Entry Point**, 

즉 외부 트래픽이 처음 진입하는 **Edge Layer**이자, 

모든 사용자가 가장 먼저 마주하는 **Front Door** 역할을 합니다.

모든 Request는 이 Gateway를 통해 **Authentication, Rate Limiting,** **Routing,** **Observability  필터링** 단계를 거친 뒤 내부 서비스로 전달됩니다.

이 Gateway는 **Terraform**과 **ArgoCD** 기반으로 자동화되어 있으며, 

장애 발생 시에도 **5분 이내에 동일한 환경으로 복구하는 것을 목표로 합니다**.

---

## 주요 기능

**Zero-Trust Edge**

- **Istio Gateway + PeerAuthentication(mTLS STRICT)**
- **RequestAuthentication(JWT)** + **AuthorizationPolicy(aud/scope Claim Validation)**
- **EnvoyFilter Security Header Injection** (HSTS, CSP, Referrer-Policy)

**Rate-Limit & Resilience**

- Per-User / Per-IP Rate-Limit via Envoy + Lyft/RateLimit Service
- VirtualService Level의 **Retry / Timeout** 설정
- DestinationRule 기반 **Circuit Breaker / Outlier Detection**

**Operational Hardening**

- Health / Liveness / Readiness Probe
- **IRSA (IAM Role for ServiceAccount)** 적용으로 AWS 접근 제어
- NonRoot User, ReadOnlyRootFilesystem, Capabilities Drop 보안 설정

**Observability**

- **Prometheus ServiceMonitor**로 Metrics Scrape
- **Grafana Dashboard** + **Loki / Tempo** 연동
- **PrometheusRule SLO Alert** (P95 Latency, 5xx Error-Rate 모니터링)

---

## 역할과 흐름

1. 외부 사용자의 HTTPS Request가 **AWS ALB + Route53**을 통해 **Istio IngressGateway**로 유입됩니다.
2. Gateway는 **mTLS + JWT Validation**을 수행하여 신뢰 경계를 설정합니다.
3. **Envoy RateLimit Filter**가 IP / User 단위의 요청 빈도를 제어합니다.
4. Routing Rule에 따라 내부 Service로 트래픽이 전달됩니다.
    - `/api/v1/trade/*` → svc-trading-api (Order Handling)
    - `/api/v1/wallet/*` → svc-wallet (Deposit / Withdraw)
    - `/api/v1/risk/*` → svc-risk-control (Risk Evaluation)
5. 모든 Request Metric과 Log는 **Prometheus / Loki**로 Export되고,
    
    **Grafana Dashboard**에서 지연시간과 Error-Rate를 시각화합니다.
    

---

## 배포 순서

1. **Argo CD**의 **Sync-Wave 10** 단계에서 배포되어,
    
    **모든 내부 Service**보다 **먼저** 올라옵니다.
    
2. **Application Resource**들은 **gateway Namespace**에 존재합니다.
3. **EnvoyFilter**는 **istio-system Namespace**에서 IngressGateway에 적용됩니다.

---

## CI / CD Pipeline

**GitHub Actions**

- **Kubeconform**으로 K8s Manifest Schema 검증
- **Cosign**으로 Image Signature 검증
- **Conftest**로 Policy (mTLS / PodSecurity 등) 테스트

**Argo CD**

- **Auto-Sync + Self-Heal** 활성화
- 실패 시 Rollback 및 Slack Notification

---

## 운영 SLO (Service Level Objective)

| Metric | 목표(Target) |
| --- | --- |
| P95 Latency | ≤ 500ms |
| Error-Rate (5xx) | ≤ 2% |
| Availability | ≥ 99.99% |
| JWT Validation Fail-Rate | ≤ 0.1% |
| Rate-Limit Response (429) | ≤ 3% |

---

## 다른 Repository와의 연동 구조

| 구분 | 연결 Repo | 역할 |
| --- | --- | --- |
| **Inbound (상단 진입)** | AWS Route53 / ALB / ACM | DNS → TLS Termination → Gateway |
| **Outbound (하단 서비스)** | svc-trading-api / svc-wallet / svc-risk-control | 인증 후 내부 API로 Routing |
| **Infrastructure 관리** | infra-terraform | EKS, IAM(IRSA), ALB, Route53 코드 관리 |
| **GitOps 제어** | platform-argocd | Argo CD App-of-Apps 방식 배포 |
| **보안 정책** | policy-as-code | Kyverno / OPA로 mTLS, JWT, PodSecurity 규칙 검증 |
| **장애 / DR 대응** | tests-and-dr | ChaosMesh / Route53 Failover 테스트 |
| **모니터링 / 로그** | observability-stack | Prometheus / Loki / Tempo / Grafana Stack 통합 |

---
