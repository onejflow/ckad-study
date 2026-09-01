TASK
Update the Pod agent in the firewall namespace to use a NetworkPolicy allowing
the Pod to send and receive traffic only to and from the Pods front and db.
- All required NetworkPolicies have already been created.
- Must not create, modify or delete any NetworkPolicy

즉 문제는 
- NetworkPolicy(방화벽)는 이미 다 만들어져 있고 수정/생성/삭제가 불가능한 조건
- 따라서 기존 NetworkPolicy 설정을 조회(`-o yaml`)해서 어떤 라벨(Label)을 허용하고 있는지 파악한 뒤,
- `agent` 파드에 해당 라벨을 붙여주어(`kubectl label pod`) 통신이 되도록 만드는 문제임

---

## 문제 풀이 순서

### 1. 네임스페이스 내 파드 및 NetworkPolicy 목록 확인
`firewall` 네임스페이스에 있는 파드들의 현재 라벨과 이미 생성되어 있는 NetworkPolicy를 확인합니다.

```bash
# 파드 목록 및 라벨 확인
kubectl get pod -n firewall --show-labels

NAME      READY   STATUS    RESTARTS   AGE   LABELS
agent     1/1     Running   0          53m   app=agent
backend   1/1     Running   0          53m   app=backend
db        1/1     Running   0          53m   app=db
front     1/1     Running   0          53m   app=front

# NetworkPolicy 목록 확인
kubectl get netpol -n firewall

NAME           POD-SELECTOR     AGE
allow-all      allow-all=true   54m
allow-db       app=db           54m
allow-front    app=front        54m
default-deny   <none>           54m
```

---

### 2. 기존 NetworkPolicy 상세 설정 확인
NetworkPolicy를 수정할 수 없으므로, 어떤 라벨을 가진 파드를 대상으로 방화벽 정책이 걸려있는지 YAML로 확인

```bash
# YAML로 상세 조회
kubectl get netpol -n firewall -o yaml

# 또는 describe로 확인
kubectl describe netpol <네트워크정책-이름> -n firewall
```

```yaml
kubectl get netpol -n firewall -o yaml
apiVersion: v1
items:
- apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    creationTimestamp: "2026-09-01T13:48:36Z"
    generation: 1
    name: allow-all
    namespace: firewall
    resourceVersion: "27710481"
    uid: dfd2c1c7-9212-4c0e-b3c0-91cd1dfa8ae9
  spec:
    podSelector:
      matchLabels:
        allow-all: "true"
    policyTypes:
    - Ingress
    - Egress
- apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    creationTimestamp: "2026-09-01T13:48:36Z"
    generation: 1
    name: allow-db   # allow-db 정책
    namespace: firewall
    resourceVersion: "27710482"
    uid: a8ea83fa-0b5e-4851-9f7d-c26f01d7a1fb
  spec:
    egress:
    - to:
      - podSelector:
          matchLabels:
            allow-db: "true" # db 파드가 데이터 보낼 때 해당라벨이 있어야 가능
    ingress:
    - from:
      - podSelector:
          matchLabels:
            allow-db: "true" # db 파드로 들어오려면 해당 라벨있어야함
    podSelector:
      matchLabels:
        app: db # db 파드에 걸려있는 방화벽
    policyTypes:
    - Ingress
    - Egress
- apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    creationTimestamp: "2026-09-01T13:48:37Z"
    generation: 1
    name: allow-front   # allow-front 정책
    namespace: firewall
    resourceVersion: "27710486"
    uid: 36049ef2-1e56-471c-82f4-4f4c9c15ed1a
  spec:
    egress:
    - to:
      - podSelector:
          matchLabels:
            allow-front: "true" # <- front 파드가 데이터 보낼때 해당 라벨이 있어야함
    ingress:
    - from:
      - podSelector:
          matchLabels:
            allow-front: "true" # <- front 파드로 들어오려면 해당 라벨이 있어야함
    podSelector:
      matchLabels:
        app: front # <- front 파드에 걸려있는 방화벼
    policyTypes:
    - Ingress
    - Egress
- apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    creationTimestamp: "2026-09-01T13:48:37Z"
    generation: 1
    name: default-deny
    namespace: firewall
    resourceVersion: "27710483"
    uid: f2b0b404-2af6-468b-8fd8-98bcb06336c5
  spec:
    podSelector: {}
    policyTypes:
    - Ingress
kind: List
metadata:
  resourceVersion: ""
```


> 확인 사항
> - `spec.podSelector.matchLabels`에 `agent` 파드가 가져야 할 라벨 확인 (예: `allow-db: "true"`, `allow-front: "true"` )
> - `spec.ingress.from`: `front` 파드로부터의 유입 허용 라벨
> - `spec.egress.to`: `db` 파드로의 유출 허용 라벨

---

### 3. `agent` 파드에 라벨(Label) 부여
2번에서 확인한 `spec.podSelector`에 정의된 라벨을 `agent` 파드에 추가

```bash
# agent 파드에 라벨 추가
kubectl edit pod agent -n firewall 

---

### 4. 라벨 반영 여부 확인
`agent` 파드에 라벨이 제대로 적용되었는지 확인합니다.

```bash
kubectl get pod agent -n firewall --show-labels
```

---

### 5. 통신 테스트 및 검증
`agent` 파드에서 `front`와 `db` 파드로의 통신이 정상적으로 허용되는지 테스트

```bash
# 대상 파드들의 IP 확인
kubectl get pods -n firewall -o wide

# agent 파드 내부에서 front 및 db로 접속 테스트
kubectl exec -n firewall agent -it -- curl <front-pod-id>:8080/
kubectl exec -n firewall agent -it -- curl  <db-pod-ip>:<포트>/
```
