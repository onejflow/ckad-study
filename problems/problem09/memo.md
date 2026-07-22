# Kubernetes 배포 전략 & Canary Deployment 메모

## 1. 카나리 배포 (Canary Deployment)의 기술적 개념

### 카나리 배포란?
- 전체 사용자에게 신규 버전을 한 번에 배포하는 위험을 줄이기 위해, **실서버(Production) 트래픽 중 극히 일부(예: 10%~20%)만 신규 버전(Canary Pod)으로 흘려보내 사전에 안전하게 검증**하는 배포 방식

---

## 2. 쿠버네티스와 카나리 배포의 관계 (수작업 필요성)

### 카나리 배포는 쿠버네티스의 기본(Built-in) 기능이 아님
* 순수 쿠버네티스 기본 오브젝트(Deployment, Service)만으로 카나리 배포를 하려면 수작업 필요
  1. 기존 버전 Deployment와 별도로 신규 버전 카나리 Deployment를 추가로 생성해야 함
  2. 쿠버네티스 `Service`는 라벨 셀렉터(Label Selector)에 매칭되는 Pod들에게 Pod 개수 비례(Round-Robin)로 트래픽을 분산
  3. 따라서 원하는 트래픽 비율(예: 80% vs 20%)을 맞추기 위해 두 Deployment의 `replicas` 파드 개수 비율(8:2)을 수동으로 계산하여 설정od
    - 같은 사용자더라도 호출 시 마다 다르게 할 수 있어 로직이 다르게 처리될수 있기도 하고 새로운 pod에서는 DB스키마가 변경될 경우도 있고해서 실무에서 사용하기 까다로움??

> **참고 ( 카나리 자동화)**:
> 수동 파드 개수 조절 대신 세밀한 트래픽 가중치 분산(예: 1%만 카나리로 전송) 및 자동 롤백을 구현하기 위해서는 **Istio, Linkerd** 같은 Service Mesh나 **Argo Rollouts, Flagger** 같은 외부 배포 컨트롤러 도구가 쓰일 수 있다고함 (argo workflow처럼)

---

## 3. 쿠버네티스의 기본 배포 전략 (Deployment Strategy)

쿠버네티스 Deployment 스펙(`spec.strategy.type`)에서 기본 제공하는 배포 전략은 크게 **2가지**입니다.

### ① RollingUpdate (롤링 업데이트) - ★ 기본값 (Default)
* **방식**: 기존 파드를 하나씩 순차적으로 제거하면서, 동시에 신규 파드를 하나씩 순차적으로 생성하는 **무중단 배포** 방식입니다.
* **장점**: 서비스 중단(Downtime) 없이 배포가 진행됩니다.
* **주요 옵션**:
  * `maxSurge`: 배포 도중 지정된 replicas 수를 초과하여 최대로 추가 생성할 수 있는 파드의 개수/비율 (기본값: 25%)
  * `maxUnavailable`: 배포 도중 일시적으로 사용할 수 없는 상태가 되어도 되는 파드의 개수/비율 (기본값: 25%)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
```

### ② Recreate (재생성)
* **방식**: 기존 버전의 모든 파드를 **일시에 전부 종료(삭제)**한 후, 신규 버전의 파드를 일괄 생성합니다.
* **단점**: 기존 파드가 켜지고 신규 파드가 뜰 때까지 순간적인 **서비스 중단(Downtime)**이 발생합니다.
* **사용 이유**: 구버전과 신버전 파드가 동시에 떠서 같은 DB나 파일 시스템을 접근할 때 충돌이 우려되는 경우(예: DB 마이그레이션 필수 작업)에 사용됩니다.

```yaml
spec:
  strategy:
    type: Recreate
```

---

## 4. 관련 공식 문서

* [Kubernetes Deployments - Canary Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#canary-deployment)
* [Kubernetes Workload Management - Canary Deployments](https://kubernetes.io/docs/concepts/workloads/management/#canary-deployments)
* [kubectl rollout Reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)

---

## 5. kubectl rollout 

rollout`은 **Deployment, DaemonSet, StatefulSet**처럼 배포 및 파드 생성을 관리하는 워크로드 컨트롤러 오브젝트들의 배포 진행
  상태, 이력(Revision), 롤백 등을 모니터링하고 조작하는 kubectl 하위 명령어(sub-command)

| 명령어 | 설명 | 사용 예시 |
| :--- | :--- | :--- |
| **status** | 리소스의 배포 진행 상태를 실시간으로 모니터링합니다. | `kubectl rollout status deploy/app2 -n test` |
| **history** | 리소스의 배포 이력(Revision) 목록 및 세부 변경 사항을 확인합니다. | `kubectl rollout history deploy/app2 -n test` |
| **undo** | 배포에 문제가 있을 때 이전 버전(또는 특정 revision)으로 롤백합니다. | `kubectl rollout undo deploy/app2 -n test` |
| **pause** | 진행 중인 롤아웃을 일시 정지합니다. | `kubectl rollout pause deploy/app2 -n test` |
| **resume** | 일시 정지된 롤아웃을 다시 재개합니다. | `kubectl rollout resume deploy/app2 -n test` |
| **restart** | 리소스의 파드(Pod)들을 순차적으로 재시작(재배포)합니다. | `kubectl rollout restart deploy/app2 -n test` |
