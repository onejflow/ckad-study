# CKAD 팁: YAML 템플릿 생성 & Heredoc(EOF) 활용법

---

## 1. YAML 템플릿 추출 (`--dry-run=client -o yaml`)

### 1-1. 기본 원리
* `--dry-run=client`: 클러스터에 실제 리소스를 생성하지 않고 터미널에서 문법/스펙만 시뮬레이션
* `-o yaml`: 생성 결과를 YAML 형식으로 터미널에 출력
* `> 파일명.yaml`: 출력된 내용을 파일로 저장

```bash
# 기본 형태
kubectl run <파드명> --image=<이미지명> --dry-run=client -o yaml > pod.yaml
```

### 1-2. 실전 단축키 설정 (시험 시작 필수 팁)
터미널 시작 시 환경 변수를 등록하면 타이핑 시간을 크게 줄일 수 있습니다.

```bash
# 환경 변수 등록
export do="--dry-run=client -o yaml"

# 사용 예시 ($do 붙이기)
k run my-pod --image=nginx $do > pod.yaml
k create deploy my-deploy --image=nginx --replicas=3 $do > deploy.yaml
```

---

### 1-3. 리소스별 템플릿 추출 원라이너 모음

#### ① Pod (파드)
Multi-container, Volume 마운트, Probe, Env 설정이 필요할 때 기본 틀 추출용으로 사용합니다.
```bash
kubectl run my-pod --image=nginx --port=80 $do > pod.yaml
```

#### ② Deployment (디플로이먼트)
```bash
kubectl create deployment my-deploy --image=nginx:1.14.2 --replicas=3 -n default $do > deploy.yaml
```

#### ③ Service (서비스)
```bash
# 1) 기존 Deployment에 연결된 NodePort 서비스 생성
kubectl expose deployment my-deploy --port=80 --target-port=80 --type=NodePort --name=web-service $do > svc-nodeport.yaml

# 2) ClusterIP 서비스 단독 생성
kubectl create service clusterip my-service --tcp=8080:80 $do > svc-clusterip.yaml
```

#### ④ Job & CronJob (잡 / 크론잡)
```bash
# Job
kubectl create job my-job --image=busybox $do -- /bin/sh -c "echo Hello; sleep 5" > job.yaml

# CronJob
kubectl create cronjob my-cron --image=busybox --schedule="*/5 * * * *" $do -- /bin/sh -c "date" > cron.yaml
```

#### ⑤ ConfigMap & Secret
```bash
# ConfigMap
kubectl create cm app-config --from-literal=APP_MODE=production $do > cm.yaml

# Secret (Generic)
kubectl create secret generic app-secret --from-literal=DB_PASSWORD=rootpass $do > secret.yaml
```

#### ⑥ RBAC (Role & RoleBinding)
```bash
# Role
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev $do > role.yaml

# RoleBinding (ServiceAccount와 연결)
kubectl create rolebinding read-pods --role=pod-reader --serviceaccount=dev:scraper-sa -n dev $do > rolebinding.yaml
```

#### ⑦ Ingress (인그레스)
```bash
kubectl create ingress my-ingress --rule="example.com/app=app-service:80" $do > ingress.yaml
```

---

### 1-4. 기존 리소스에서 YAML 추출하기
기존에 배포된 리소스를 복제하거나 수정하여 새 리소스를 만들 때 사용합니다.

```bash
kubectl get deploy current-web-deployment -n prod -o yaml > canary-web-deployment.yaml
```
> [!TIP]
> 기존 리소스에서 추출한 YAML에는 `status`, `uid`, `resourceVersion`, `creationTimestamp` 등의 시스템 메타데이터가 포함되어 있습니다. 새로 배포할 때는 해당 항목들을 삭제하거나 필요한 필드만 남겨서 배포해야 충돌이 발생하지 않습니다.

---

## 2. 임시 파일 없는 즉시 배포: Heredoc (`<<EOF`) & 다중 문서 (`---`)

### 2-1. Heredoc (`<<EOF`) 이란?
* 쉘(Bash/Zsh)에서 여러 줄의 텍스트를 파일로 저장하지 않고 명령어의 표준 입력(stdin)으로 직접 넘겨주는 문법입니다.
* `kubectl apply -f -`의 `-`는 표준 입력(stdin)으로부터 YAML을 읽어들이겠다는 의미입니다.

### 2-2. 다중 문서 구분자 (`---`)
* 하나의 YAML 스트림 안에서 **2개 이상의 독립된 쿠버네티스 리소스**를 동시에 정의할 때 사용합니다.
* 예: ServiceAccount + Role + RoleBinding을 한 번에 배포할 때 유용합니다.

---

### 2-3. 실전 예제

#### 예제 1: ServiceAccount + Role + RoleBinding 한 번에 배포
터미널에 아래 블록을 통째로 복사/붙여넣기 하면 임시 파일 생성 없이 즉시 배포됩니다.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: scraper-sa
  namespace: dev
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: scraper-role
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: scraper-binding
  namespace: dev
subjects:
- kind: ServiceAccount
  name: scraper-sa
  namespace: dev
roleRef:
  kind: Role
  name: scraper-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

#### 예제 2: NodePort Service 빠른 정의
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: front
spec:
  type: NodePort
  selector:
    app: nginx
    type: frontEnd
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
EOF
```

---

## 3. 요약 및 상황별 권장 패턴

| 상황 | 권장 방식 |
| :--- | :--- |
| 옵션이 단순한 기본 리소스 생성 | `kubectl create ...` 또는 `kubectl run ...` 명령형으로 즉시 생성 |
| Volume, Probe, 복잡한 설정 추가 필요 | `k run ... $do > test.yaml` 로 템플릿 뽑고 `vi test.yaml` 수정 후 `k apply -f` |
| 연관된 여러 리소스(SA + RBAC 등) 일괄 배포 | `kubectl apply -f - <<EOF` + `---` 구분자로 터미널 원터치 실행 |
