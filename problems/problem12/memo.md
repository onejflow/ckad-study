limitrange란? 

쿠버네티스 네임스페이스(Namespace) 내에서 생성되는 파드(Pod)나 컨테이너의 자원(CPU, 메모리, 스토리지) 요청량(Requests)과 제한량(Limits)의 최소·최대 크기를 강제로 제한하는 정책

HPA가 트래픽 변화에 따른 파드 개수 자동조절이고 deployment와 함께 쓴다면며 limitrange는 네임스페이스 내 자원 사용 강제, limitrange는 개별 파드 및 컨테이너에 적용

ResourceQuota는?
ResourceQuota는 쿠버네티스 네임스페이스(Namespace) 단위로 사용할 수 있는 컴퓨팅 자원의 총량과 오브젝트의 생성 개수를 제한하는 정책

- 컴퓨팅 자원 총량 제한: 해당 네임스페이스에 속한 모든 파드의 CPU 및 메모리 요청량(Requests)과 제한량(Limits)의 총합을 제한합니다 [Resource Quotas].
- 스토리지 총량 제한: 네임스페이스 내에서 생성할 수 있는 PersistentVolumeClaim(PVC)의 전체 용량 합계를 제한합니다 [Resource Quotas].
- 오브젝트 개수 제한: 특정 네임스페이스 안에서 만들 수 있는 파드(Pod), 서비스(Service), 디플로이먼트(Deployment), 컨피그맵(ConfigMap) 등의 최대 개수를 묶어둡니다 [Resource Quotas]


ResourceQuota로 CPU나 메모리 총량을 제한하면, 해당 네임스페이스에 배포되는 모든 파드는 반드시 자신들의 requests와 limits(컴퓨팅 자원 설정)를 명시해야 함. 
자원 설정이 없는 파드는 생성이 거부됨 

- 팁: 이 문제를 해결하기 위해 앞서 언급한 LimitRange를 함께 사용. 개발자가 자원 설정을 깜빡하더라도 LimitRange가 기본값(Default)을 자동으로 넣어주므로, ResourceQuota에 걸려 파드 생성이 거절되는 일을 막을 수 있음

