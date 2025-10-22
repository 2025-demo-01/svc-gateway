 ### Resource와 NS
Gateway / VirtualService / PeerAuthentication / RequestAuthentication / ServiceMonitor / PrometheusRule / AuthorizationPolicy / NetworkPolicy / Deployment / Service
→ 모두 gateway(ns)
 EnvoyFilter(ratelimit, 보안헤더 등) → istio-system(ns)
통과 신호: kubectl -n istio-system get envoyfilter 에서 resource 보임
-> EnvoyFilter를 gateway에 두면 매칭 안됨

### Gateway ↔ VirtualService 정합성 
gateway.yaml의 spec.servers[*].hosts와 virtualservice.yaml의 spec.hosts 도메인 동일
redentialName(예: acm-tls-cert)가 실제 Secret/ACM과 매칭
통과 신호: ALB HTTPS 443 정상, 브라우저/curl -v 인증서 OK
-> LB 생성 직후 2~3분 전파 지연 있을 수 있음

### 인증/인가 경계
requestauthn.yaml의 issuer, audiences, jwksUri 값 설정
있다면 authorizationpolicy.yaml 의 requestPrincipals/claims 조건이 팀내 규칙과 매칭
잘못된/누락 JWT → 401 Unauthorized
올바른 JWT, scope 불일치 → 403 Forbidden
올바른 JWT + scope 일치 → 200 OK

### mTLS 기본값
peerauth-strict.yaml → mode: STRICT
테스트: 사이드카 없는 Pod로 직접 호출 시 실패(정상)

### RateLimit 경계 
짧은 시간에 과도 호출 → 429 Too Many Requests + Retry-After 헤더
정상 호출 → 200/4xx/5xx 응답, 429 비발생

### Metric/Log/Trace
metadata.namespace: gateway / spec.selector.matchLabels.app: svc-gateway
namespaceSelector.matchNames: ["gateway"]
통과 신호: Prometheus에서 target UP / Grafana 패널에 지표 표시
Deployment에 프로브/리소스/보안컨텍스트/메트릭 어노테이션 적용
통과 신호: kubectl -n gateway get pods → READY 1/1 안정
/metrics 스크랩 OK

### NetworkPolicy 
networkpolicy.yaml 이 ingress/egress 최소허용으로 설정
통과 신호: 허용된 경로만 통신 OK (불필요한 동서 트래픽 차단)

### CI Gate 
GitHub Actions → kubeconform 통과
cosign verify / conftest 규칙 통과
통과 신호: Actions 체크 / Log에 “Summary: … OK”

#### Smoke Test 
##### 네임스페이스/리소스
kubectl get ns gateway istio-system
kubectl -n gateway get deploy,svc,gw,vs,peerauth,requestauthentication,servicemonitor,prometheusrule,authorizationpolicy,networkpolicy
kubectl -n istio-system get envoyfilter

##### 라우팅/인증
HOST=api.exchange.example.com
curl -sIk "https://${HOST}/api/v1/trade/health"         # 200(무인증 허용 구간이면)
curl -sIk "https://${HOST}/api/v1/trade/orders"         # 401 기대 (JWT 미포함)
curl -sIk -H "Authorization: Bearer $GOOD_JWT" "https://${HOST}/api/v1/trade/orders"  # 200
curl -sIk -H "Authorization: Bearer $BAD_SCOPE_JWT" "https://${HOST}/api/v1/trade/orders"  # 403

# 레이트리밋 (1분에 N회 반복)
for i in {1..400}; do curl -s -o /dev/null -w "%{http_code}\n" "https://${HOST}/api/v1/trade/ping"; done
# 429와 Retry-After 헤더 확인
curl -sI "https://${HOST}/api/v1/trade/ping" | grep -i retry-after

## 설명
- 네임스페이스 전략: 앱 리소스는 `gateway`, EnvoyFilter는 `istio-system`으로 분리(ingressgateway 워크로드 매칭 규칙에 따름).
- CI 품질 게이트: kubeconform + (선택) cosign verify + conftest로 스키마/서명/정책 위반을 차단합니다.
- 엣지 보안: mTLS STRICT, JWT 검증 + AuthorizationPolicy(aud/scope), Envoy RateLimit(429 + Retry-After), 보안 헤더(HSTS/CSP).
- 다운스트림 보호: VirtualService 재시도·타임아웃, DestinationRule 서킷브레이커·아웃라이어 제거.

svc-trading-api를 붙여서 /orders 접수 → Kafka produce → Aurora 기록 → Flagger canary까지 한 번에 묶자.
