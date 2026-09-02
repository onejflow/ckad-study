# Task

- A deployment named **nginx**, running in **namespace cente**r is exposed via **Ingress web-ingress.** 
The deployment is supposed to be reachable at http://center.zone1.local but requesting this URL is currently returning an error. 

Identify and fix the problems by updating the associated resources so that the Deployment becomes externally reachable as planned. 

**Don’t modify the Deployment.** 
You can use the following command to test the Deployment. 
$ curl -vL http://center.zone1.local

# 문제풀이

!problem.png

- Ingress에 host가 누락되어 있음. 
rules 하위에 추가 필요

```jsx
rules: 
	host: "center.zone1.local" 
```

- Ingress에 service 명이 잘못 기재되어 있음.

```jsx
service: 
	name: web
```

- Service Selector의 key, value가 Deployment의 label과 매칭이 되지 않음.

```jsx
selector: 
	app: nginx
```

# 해답

!answer.png

- IngressClassName 동일해야 함.
- Ingress에서 Service로 연결하는 포트와 서비스에서 Deployment를 연결하는 포트가 일치해야 함.
