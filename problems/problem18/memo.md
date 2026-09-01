# NetworkPolicy란?

쿠버네티스 파드(Pod)에 적용하는 일종의 **방화벽(네트워크 보안 규칙)**

- 기본 상태: 쿠버네티스의 파드들은 기본적으로 서로 아무런 제약 없이 전부 통신 가능함 (All Allow)
- NetworkPolicy 적용 시: 파드에 NetworkPolicy가 하나라도 연결되는 순간 해당 파드는 **격리(Isolated)** 상태로 변경됨
- 즉, 화이트리스트(Whitelist) 방식으로 동작하여 명시적으로 허용한 트래픽만 통과하고 나머지는 전부 차단됨

---

# Ingress 와 Egress 개념

- **Ingress (인그레스)**: 외부나 다른 파드에서 **내 파드로 들어오는** 트래픽 (Inbound)
  - 예: 프론트엔드 파드가 내 백엔드 파드로 요청을 보낼 때 (내 파드 기준 Ingress)
- **Egress (이그레스)**: 내 파드에서 다른 파드나 외부로 **나가는** 트래픽 (Outbound)
  - 예: 내 파드가 DB 파드에 데이터를 요청하러 나갈 때 (내 파드 기준 Egress)

---

# NetworkPolicy YAML 구조

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-network-policy
  namespace: firewall
spec:
  # 1. 방화벽 적용 대상 파드 지정 (어떤 파드를 보호/제어할 것인가?)
  podSelector:
    matchLabels:
      app: agent

  # 2. 적용할 정책 타입 (Ingress, Egress 둘 다 적용할지 하나만 할지)
  policyTypes:
  - Ingress
  - Egress

  # 3. 들어오는 트래픽(Ingress) 허용 규칙
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: front         # front 라벨을 가진 파드에서 들어오는 것만 허용
    ports:
    - protocol: TCP
      port: 80

  # 4. 나가는 트래픽(Egress) 허용 규칙
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: db            # db 라벨을 가진 파드로 나가는 것만 허용
    ports:
    - protocol: TCP
      port: 3306
```

---

# from / to 작성 시 주의점 (OR vs AND)

대시(`-`)의 위치에 따라 조건이 완전히 달라지므로 주의해야 함

### 1) OR 조건 (대시가 각각 따로 붙은 경우)
- 네임스페이스가 `dev`인 파드 **OR** `app: front` 라벨을 가진 파드 (둘 중 하나만 만족해도 허용)

```yaml
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          env: dev
    - podSelector:
        matchLabels:
          app: front
```

### 2) AND 조건 (대시 하나 아래에 같이 묶인 경우)
- `env: dev` 네임스페이스 안에 있으면서 **동시에** `app: front` 라벨을 가진 파드만 허용

```yaml
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          env: dev
      podSelector:
        matchLabels:
          app: front
```

---

# 문제 풀이 순서

1. 기존 NetworkPolicy 조회
`kubectl get netpol -n firewall`
`kubectl get netpol <정책이름> -n firewall -o yaml`

2. YAML 내의 `podSelector`와 `ingress`/`egress` 라벨 확인
- `agent` 파드에 어떤 라벨이 있어야 방화벽 정책에 걸리는지 확인

3. `agent` 파드에 라벨 적용
`kubectl label pod agent -n firewall app=agent` (필요한 키=값 부여)

4. 통신 테스트
`kubectl exec agent -n firewall -- curl <front-ip>`
`kubectl exec agent -n firewall -- curl <db-ip>`
