# 4-2 task-service 풀이 정리

## 1. 문제 번역

`sp-culator` 네임스페이스에 있는 기존 Deployment `front-end`를 재구성하여, 기존 컨테이너 `nginx`의 `80/tcp` 포트를 노출하시오.

컨테이너 포트 `80/tcp`를 노출하는 `front-end-svc`라는 새 Service를 생성하시오.

새 Service가 개별 Pod들을 `NodePort`를 통해서도 노출하도록 구성하시오.


---

## 3. 기존 상태 확인

```bash
kubectl get all -n sp-culator
kubectl describe deployment front-end -n sp-culator
```

처음 상태에서는 Deployment의 컨테이너 포트가 없다.

```text
Containers:
 nginx:
  Image: nginx:1.14.2
  Port:  <none>
```

---

## 4. Deployment 수정

기존 Deployment `front-end` 안의 컨테이너 `nginx`에 `80/TCP` 포트를 추가한다.



아래 위치에 `ports` 항목을 추가한다.

```yaml
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
          protocol: TCP
```

---

## 서비스 리소스 생성
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
  - name: nginx
    protocol: TCP
    port: 80
    targetPort: 80
```


---

## 최종 확인
```text
Name:        front-end-svc
Namespace:   sp-culator
Selector:    app=front-end
Type:        NodePort
Port:        nginx 80/TCP
TargetPort:  80/TCP
NodePort:    nginx 30130/TCP
Endpoints:   20.108.82.254:80,20.108.82.255:80,20.108.82.249:80
```



```text
NAME            ENDPOINTS                                            AGE
front-end-svc   20.108.82.249:80,20.108.82.254:80,20.108.82.255:80   4m
```
