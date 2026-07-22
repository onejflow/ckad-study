# Canary Deployment (카나리 배포)

## TASK
You are asked to prepare a Canary deployment for testing a new application release.
A Service named `web-service` in the `prod` namespace points to 5 pods created by the Deployment named `current-web-deployment`
- path: `~/canary/current-web-deployment.yaml`

Create Canary Deployment named `canary-web-deployment`, in the same namespace.
- A maximum number of 10 pods run in the `prod` namespace.
- 20% of the `web-service` traffic goes to the `canary-web-deployment` pods.
- Testing : `curl http://localhost:30020`

---

## 풀이

- deployment 조회
  - `kubectl get deploy -n prod`
  - `kubectl get svc -n prod`

 
- 비율 계산:
  - 카나리 파드 수:  2 개 (`canary-web-deployment` replicas: 2)
  - 기존 파드 수: 8 개 (`current-web-deployment` replicas: 8)

---

- Current Deployment의 Replica 수 변경
  - `kubectl edit -n prod deploy current-web-deployment`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: current-web-deployment
  namespace: prod
spec:
  replicas: 8 # 5 -> 8로 수정
```

---

- Canary Deployment 매니페스트 작성 및 배포

기존 파일(`~/canary/current-web-deployment.yaml`)을 복사
- `cp current-web-deployment.yaml canary-web-deployment.yaml`


```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: canary-web-deployment # 카나리로 수정
  namespace: prod
spec:
  replicas: 2 # 2로 수정
  selector:
    matchLabels:
      name: canary # canary 로 수정
      app: web 
  template:
    metadata:
      labels:
        name: canary # canary로 수정
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.14.2
          ports:
            - containerPort: 80
          volumeMounts:
            - mountPath: /usr/share/nginx/html
              name: volume
      initContainers:
        - name: init
          image: busybox:stable
          command:
            - sh
            - -c
            - |
               echo $POD_NAME > /usr/share/nginx/html/index.html
```

매니페스트 적용:
- `kubectl apply -f canary-web-deployment.yaml`


---

테스트
- curl http://localhost:30020

