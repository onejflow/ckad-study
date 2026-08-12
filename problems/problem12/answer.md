문제

TASK
A Deployment named mysql in the database Namespace fails to start because its
container runs out of memory.
Modify a mysql Deployment in the database namespace
- requests 128Mi of memory
- limits the memory to half the maximum memory for namespace


pod조회
`kubectl get pod -n database`
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
mysql   0/1     1            0           33s

NAME                     READY   STATUS             RESTARTS       AGE
mysql-74977b8c85-6znk8   0/1     CrashLoopBackOff   6 (3m8s ago)   9m15s


limitrange 조회
`kubectl get limitrange -n database`


deployment 수정
`kubectl edit deployment mysql -n database`

이후 비어있는 resources: {} 를 수정

```yaml
    spec:
      containers:
      - image: 1pro/app
        imagePullPolicy: Always
        name: mysql
        ports:
        - containerPort: 8080
          protocol: TCP
        resources:
          limits:
            memory: 320Mi
          requests:
            memory: 128Mi
```

그 뒤 확인

`kubectl get pod -n database`
NAME                     READY   STATUS    RESTARTS   AGE
mysql-66f8f5d5d7-phnlh   1/1     Running   0          74s