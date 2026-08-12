TASK
A pod within the Deployment named server and in namespace beta is logging
errors.
Look at the logs and identify error messages.
Find errors, including: User "system:serviceaccount:beta:default" cannot list resource
"pods" [...] in the namespace beta.
Update the Deployment server to resolve the errors in the logs of the Pod.
Path: ~/role/server.yaml


---

pod 로그 조회 
`kubectl logs -n beta server-5976b7cd9f-zj59l`

서비스 어카운트 조회
`kubectl get sa -n beta`

role, rolebinding 조회도 
`kubectl get role -n beta`
`kubectl get rolebinding -n beta`


deployment에 service account로 backend-sa 수정

`kubectl edit deployment server -n beta`


root@k8s-master:~# kubectl get pod -n beta
NAME                      READY   STATUS        RESTARTS   AGE
server-5976b7cd9f-zj59l   1/1     Terminating   0          6h36m
server-5bf5fcd9d7-s7s6g   1/1     Running       0          15s

새로 뜬거 확인후 다시 pod 로그 조회

root@k8s-master:~# kubectl get pod -n beta
NAME                      READY   STATUS        RESTARTS   AGE
server-5976b7cd9f-zj59l   1/1     Terminating   0          6h36m
server-5bf5fcd9d7-s7s6g   1/1     Running       0          15s
root@k8s-master:~# kubectl logs server-5bf5fcd9d7-s7s6g -n beta
NAME                      READY   STATUS        RESTARTS   AGE
server-5976b7cd9f-zj59l   1/1     Terminating   0          6h36m
server-5bf5fcd9d7-s7s6g   1/1     Running       0          2s
NAME                      READY   STATUS        RESTARTS   AGE
server-5976b7cd9f-zj59l   1/1     Terminating   0          6h36m
server-5bf5fcd9d7-s7s6g   1/1     Running       0          12s
NAME                      READY   STATUS        RESTARTS   AGE
server-5976b7cd9f-zj59l   1/1     Terminating   0          6h36m
server-5bf5fcd9d7-s7s6g   1/1     Running       0          22s
NAME                      READY   STATUS        RESTARTS   AGE
server-5976b7cd9f-zj59l   0/1     Terminating   0          6h36m
server-5bf5fcd9d7-s7s6g   1/1     Running       0          32s
NAME                      READY   STATUS    RESTARTS   AGE
server-5bf5fcd9d7-s7s6g   1/1     Running   0          42s
NAME                      READY   STATUS    RESTARTS   AGE
server-5bf5fcd9d7-s7s6g   1/1     Running   0          52s
NAME                      READY   STATUS    RESTARTS   AGE
server-5bf5fcd9d7-s7s6g   1/1     Running   0          62s
NAME                      READY   STATUS    RESTARTS   AGE
server-5bf5fcd9d7-s7s6g   1/1     Running   0          72s
NAME                      READY   STATUS    RESTARTS   AGE
server-5bf5fcd9d7-s7s6g   1/1     Running   0          82s
NAME                      READY   STATUS    RESTARTS   AGE
server-5bf5fcd9d7-s7s6g   1/1     Running   0          93s