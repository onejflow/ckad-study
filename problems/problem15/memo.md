쿠버네티스에서 네임스페이르를 만들면 자동으로 추가 되는 리소스가 있음 
- defalut 라는 이름의 ServiceAccount
- kube-root-ca.crt 의 configmap

이후 deployment를 만들 때 Pod에 자동으로 채워지는 요소는

- serviceAccount: default
- serviceAccountName: default
- containers.volumeMounts
- volumes

컨테이너에 앱 입장에서 pod의 정보가 필요할 수 있어 kube-apiserver 호출로 자신의 pod를 조회하도록 되어 있음
- pod가 만들어진 이후에 할당된 정보, 한 deployment의 연결된 pod 라던지 정보를 알아야할때 필요
- kube-apiserver는 권한 조회를함 

아래 yaml의 볼륨 부분이 컨테이너에 생성되는건데 ca.crt, namespace, token을 쿠버네티스에서 알아서 만들어 주기에 api호출이 가능

```yaml
  volumes:
  - name: kube-api-access-6sgv4
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
```

단!, 조회시 token을 보고 발급자인 ServiceAccount 즉 defalut ServiceAccount에는 아무런 권한이 없기 때문에 task의 문제와 같은 에러가 발생

그래서 권한을 주기 위한 리소스를 추가로 만들어야함 -> Role, RoleBinding
- Role은 어떤 resource에 대해 어떤 액션을 취할지 설정, RoleBinding은 ServiceAccount와 Role연결 (이런걸 RBAC role based Access Control 이라 부름)



조회한 full yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cni.projectcalico.org/containerID: 9241c6020cc5e24f67185fa0551fcc4d7b28a0eba94d0feac0d0a65548a7250d
    cni.projectcalico.org/podIP: 20.110.126.47/32
    cni.projectcalico.org/podIPs: 20.110.126.47/32
  creationTimestamp: "2026-08-12T16:13:38Z"
  generateName: server-5976b7cd9f-
  labels:
    app: server
    pod-template-hash: 5976b7cd9f
  name: server-5976b7cd9f-zj59l
  namespace: beta
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: server-5976b7cd9f
    uid: 489585c8-b7fc-4f7f-bcb1-67e8ff25af58
  resourceVersion: "23755519"
  uid: 6e3328f6-e114-447e-bbd9-44f5c50b6ecd
spec:
  containers:
  - command:
    - sh
    - -c
    - while true; do kubectl get pods -n beta; sleep 10; done
    image: bitnami/kubectl
    imagePullPolicy: Always
    name: server
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-6sgv4
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: k8s-worker2
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: kube-api-access-6sgv4
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
```


kube-apiserver란?
