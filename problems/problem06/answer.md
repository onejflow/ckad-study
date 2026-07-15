# Deployment 업데이트

deployment 조회
- kubectl get deployment -n front -o wide

deployment 업데이트
- kubectl edit deployment web-deployment -n front 
- vi로 수정

```
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2026-07-14T15:15:35Z"
  generation: 1
  name: web-deployment
  namespace: front
  resourceVersion: "18195076"
  uid: b75b59b8-7972-4c80-8630-b8aa952a9058
spec:
  progressDeadlineSeconds: 600
  replicas: 3 #변경
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
      # type: frontend # 추가했었는데 오류가 뜸 
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
```

셀렉터는 최초 deployment를 만들때만 줄 수 있는 값. 이후 부터는 만들어진 pod와 연결을 유지하기 위해 수정이 불가능한 영역

```
# deployments.apps "web-deployment" was not valid:
# * spec.template.metadata.labels: Invalid value: map[string]string{"app":"nginx"}: `selector` does not match template `labels`
# * spec.selector: Invalid value: v1.LabelSelector{MatchLabels:map[string]string{"app":"nginx", "type":"frontend"}, MatchExpressions:[]v1.Label
SelectorRequirement(nil)}: field is immutable
```

아래와 같이 spec.template.metadata.labels 에 추가했어야함

```
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2026-07-15T15:15:35Z"
  generation: 1
  name: web-deployment
  namespace: front
  resourceVersion: "18195076"
  uid: b75b59b8-7972-4c80-8630-b8aa952a9058
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
        type: frontEnd
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 1
  conditions:
  ...
```

# 노드포트 서비스 생성

공식문서에 service 검색 후 가장 위 클릭, nodeport 섹션 찾아서 참고

vi로 yaml 만든뒤 

```yaml
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
```

kubectl apply -f xxx.yaml 또는 yaml작성하지 않고

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
EOF


# 서비스 조회
- kubectl get svc -n front

# Pod와 연결 확인
- kubedctl get -n front endpointslices

# 서비스 노드포트란? 
서비스는 pod들을 외부나 내부에서 접근 가능하게 해주는 네트워크 추상화
- 노드포트: 클러스터의 각 노드(Node)의 특정 포트를 열어서, 외부에서 접근 가능하게 하는 방식
- 클러스터IP: 클러스터 내부에서만 접근가능
- LoadBalance: 클라우드 운영 등에서 활용
- ExternalName: 외부 API연결