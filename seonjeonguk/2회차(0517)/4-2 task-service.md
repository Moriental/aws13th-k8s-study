# 4-2. Deployment 포트 노출 및 NodePort Service 생성

## 문제 요약

`sp-culator` 네임스페이스에 존재하는 기존 Deployment `front-end`를 수정하여  
`nginx` 컨테이너의 `80/TCP` 포트를 노출한다.

이후 새로운 Service `front-end-svc`를 생성하고,  
Pod의 `80/TCP` 포트를 `NodePort` 방식으로 노출한다. :contentReference[oaicite:0]{index=0}

---

## 요구사항

- Namespace: `sp-culator`
- 기존 Deployment: `front-end`
- 대상 컨테이너: `nginx`
- 컨테이너 포트: `80/TCP`
- 새 Service 이름: `front-end-svc`
- Service 타입: `NodePort`

---

## 핵심 개념

### 1. Deployment의 containerPort
컨테이너가 사용하는 포트를 명시한다.

1. 기존 Deployment 수정

```yaml
containers:
- image: nginx:1.14.2
  imagePullPolicy: IfNotPresent
  name: nginx
  ports:
  - containerPort: 80
    protocol: TCP
```

2. Service 매니페스트 작성

```yaml
apiVersion: v1
kind: Service
metadata:
  name: front-end-svc
  namespace: sp-culator
spec:
  type: NodePort
  selector:
    app: front-end
  ports:
    - port: 80
      targetPort: 80
```

주의할 점 : Service의 selector는 Deployment의 Pod label과 반드시 일치해야 함 (공식문서 복붙해서 하다가 까먹었음)

3. service 적용 후 확인

kubectl apply -f front-end-svc

kubectl get svc -n sp-culator

<img width="1110" height="744" alt="image" src="https://github.com/user-attachments/assets/0cbeae4f-1ee5-45fa-980b-b1343959df1d" />

