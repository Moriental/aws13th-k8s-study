# 4-2 task-ingress 풀이 정리

## 1. 문제 번역

`echo-sound` 네임스페이스에 `echo`라는 새 Ingress 리소스를 생성하시오.

Service `echoserver-service`를 `http://example.org/echo` 경로로 노출하시오.

Service 포트는 `8080`을 사용하시오.

Service `echoserver-service`의 사용 가능 여부는 아래 명령어로 확인할 수 있으며, 결과는 `200`이 나와야 한다.

```bash
curl -o /dev/null -s -w "%{http_code}\n" http://example.org/echo
```


---

## 3. 기존 리소스 확인

```bash
kubectl get all -n echo-sound
kubectl describe svc echoserver-service -n echo-sound
```

확인할 것:

```text
Service 이름: echoserver-service
Service 포트: 8080
Pod Endpoint: 8080
```

---

## 4. Ingress YAML 작성

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo
  namespace: echo-sound
spec:
  rules:
  - host: example.org
    http:
      paths:
      - path: /echo
        pathType: Prefix
        backend:
          service:
            name: echoserver-service
            port:
              number: 8080
```

---
## 확인
```
[root@k8s-master ~]# kubectl get ingress -n echo-sound
NAME   CLASS   HOSTS         ADDRESS   PORTS   AGE
echo   nginx   example.org             80      25s
[root@k8s-master ~]# vi echo.yaml
[root@k8s-master ~]# kubectl describe ingress echo -n echo-sound
Name:             echo
Labels:           <none>
Namespace:        echo-sound
Address:          10.104.251.105
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host         Path  Backends
  ----         ----  --------
  example.org  
               /echo   echoserver-service:8080 (20.108.82.210:8080)
Annotations:   <none>
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    52s (x2 over 84s)  nginx-ingress-controller  Scheduled for sync
[root@k8s-master ~]# curl -o /dev/null -s -w "%{http_code}\n" http://example.org/echo
200
[root@k8s-master ~]# 

```
