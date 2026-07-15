시험환경 가이드 tip

터미널에 리소스 생성 관련 명령어 옮길때 외우고 활용하기 

```sh
# k apply -f - <<EOF
apiVersion: batch/v1
kind: CronJob
metadata:
    name: batch
    nmaespace: prod
spec:
    schedule: "* * * * *"
    jobTemplate:
        spec
...
# EOF
```

