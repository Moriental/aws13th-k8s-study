# 4-4. Ingress를 Gateway API로 마이그레이션

## 문제 요약

기존 Ingress 리소스 `web`의 설정을 유지한 채  
Gateway API 방식으로 마이그레이션한다.

- Gateway `web-gateway` 생성
- HTTPRoute `web-route` 생성
- 기존 Ingress `web` 삭제

---

## 요구사항

- GatewayClass: `nginx`
- Gateway 이름: `web-gateway`
- Gateway Hostname: `gateway.web.k8s.local`
- 기존 Ingress의 TLS 설정 유지
- HTTPRoute 이름: `web-route`
- 기존 Ingress의 라우팅 규칙 유지
- 최종적으로 기존 Ingress `web` 삭제

---

## 핵심 개념

기존 Ingress의 역할을 Gateway API에서는 두 리소스로 나눈다.

```text
Ingress
 ├─ TLS / 외부 Listener 설정 → Gateway
 └─ Host / Path / Backend 라우팅 → HTTPRoute
```

---

## 1. 기존 Ingress 설정 확인

```bash
kubectl get ingress web -n default -o yaml
```

기존 Ingress에서 확인한 값:

- TLS Secret: `web-cert`
- Host: `ingress.web.k8s.local`
- Path: `/`
- Backend Service: `web`
- Service Port: `80`

---

## 2. Gateway 생성

`gateway.yml`

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
    protocol: HTTPS
    port: 443
    hostname: "gateway.web.k8s.local"
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: web-cert
    allowedRoutes:
      namespaces:
        from: Same
```

적용:

```bash
kubectl apply -f gateway.yml
```

---

## 3. HTTPRoute 생성

`httproute.yml`

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
  - "gateway.web.k8s.local"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web
      port: 80
```

적용:

```bash
kubectl apply -f httproute.yml
```

---

## 4. Gateway / HTTPRoute 상태 확인

```bash
kubectl get gateway web-gateway -n default
kubectl describe gateway web-gateway -n default
kubectl describe httproute web-route -n default
```

확인 포인트:

```text
Gateway Programmed: True
Attached Routes: 1
HTTPRoute Accepted: True
ResolvedRefs: True
```

---

## 5. Gateway API 동작 확인

실습 환경에서는 Gateway가 생성한 Service의 ClusterIP로 직접 확인하였다.

```bash
curl -k --resolve gateway.web.k8s.local:443:10.100.43.246 \
https://gateway.web.k8s.local
```

출력:
<img width="1396" height="86" alt="image" src="https://github.com/user-attachments/assets/62baf67f-24e2-48df-9352-10491a9225f3" />


[cka0001@k8s-master 2회차]$ kubectl delete ingress web -n default
ingress.networking.k8s.io "web" deleted from default namespace
[cka0001@k8s-master 2회차]$ kubectl get ingress web -n default
Error from server (NotFound): ingresses.networking.k8s.io "web" not found

삭제까지 완료
