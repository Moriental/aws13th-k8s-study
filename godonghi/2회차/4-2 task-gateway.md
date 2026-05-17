# 4-2 task-gateway 풀이 정리

## 1. 문제 번역

기존 웹 애플리케이션을 Ingress에서 Gateway API로 마이그레이션하시오.

HTTP 성공 상태를 유지해야 한다.

클러스터에는 `nginx`라는 GatewayClass가 설치되어 있다.

먼저 기존 Ingress 리소스 `web`의 TLS 및 listener 설정을 유지하면서, hostname `gateway.web.k8s.local`을 사용하는 Gateway `web-gateway`를 생성하시오.

다음으로 기존 Ingress 리소스 `web`의 라우팅 규칙을 유지하면서, hostname `gateway.web.k8s.local`을 사용하는 HTTPRoute `web-route`를 생성하시오.

Gateway API 설정은 아래 명령어로 테스트할 수 있다.

```bash
curl -k https://gateway.web.k8s.local
```

마지막으로 기존 Ingress 리소스 `web`을 삭제하시오.

---

## 4. Gateway YAML 작성

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    hostname: gateway.web.k8s.local
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: web-cert
```

---

## 5. HTTPRoute YAML 작성

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
  namespace: default
spec:
  parentRefs:
  - name: web-gateway
  hostnames:
  - gateway.web.k8s.local
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web
      port: 80
```
---

## 7. 기존 Ingress 삭제


```bash
kubectl delete ingress web
```

---

## 8. curl 테스트


```bash
[root@k8s-master ~]# curl -k https://gateway.web.k8s.local
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>

```

현재 hosts 확인:

```bash
grep gateway.web.k8s.local /etc/hosts
```

```text
192.168.56.40 gateway.web.k8s.local
```

`gateway.web.k8s.local`이 기존 Ingress/nginx 쪽 IP를 보고 있는 상태다.

Gateway IP로 직접 테스트:

```bash
curl -k --resolve gateway.web.k8s.local:443:10.106.84.48 https://gateway.web.k8s.local
```

정상 결과:

```text
hello
```

일반 curl도 성공시키려면 `/etc/hosts`에서 `gateway.web.k8s.local`을 Gateway IP로 변경한다.

```bash
sudo vi /etc/hosts
```

수정 전:

```text
192.168.56.40 gateway.web.k8s.local
```

수정 후:

```text
10.106.84.48 gateway.web.k8s.local
```

수정 후 확인:

```bash
[root@k8s-master ~]# grep gateway.web.k8s.local /etc/hosts
10.106.84.48 gateway.web.k8s.local
[root@k8s-master ~]# curl -k https://gateway.web.k8s.local
hello

```


