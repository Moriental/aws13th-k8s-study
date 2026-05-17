# 4-5. NetworkPolicy 후보 중 최소 권한 정책 선택

## 문제 요약

`frontend`와 `backend` Deployment가 서로 다른 네임스페이스에 존재한다.  
기존 `deny-all` NetworkPolicy는 유지한 채,  
`frontend`에서 `backend`로 필요한 통신만 허용하는 NetworkPolicy를 선택해 적용한다.

---

## 요구사항

- `frontend`와 `backend` 사이 통신 허용
- 가능한 한 가장 제한적인 정책 선택
- 기존 `deny-all` NetworkPolicy는 삭제하거나 수정하지 않음
- `~/netpol` 폴더의 후보 YAML 중 하나만 선택해 적용

---

## 핵심 개념

```text
1. 어느 Pod에 정책을 적용하는가?
   → spec.podSelector

2. 어느 Namespace에서 오는 트래픽을 허용하는가?
   → namespaceSelector

3. 그 Namespace 안에서도 어떤 Pod만 허용하는가?
   → podSelector
```

이번 문제에서는 다음 조건에 가장 정확히 맞는 정책을 골라야 한다.

```text
frontend namespace의 frontend Pod
        ↓
backend namespace의 backend Pod
```

---

## 1. 리소스 상태 확인

Namespace와 Pod label을 확인한다.

```bash
kubectl get ns frontend backend --show-labels
kubectl get pods -n frontend --show-labels
kubectl get pods -n backend --show-labels
```

확인 기준:

```text
frontend namespace label: name=frontend
frontend Pod label: app=frontend
backend Pod label: app=backend
```

---

### netpol1.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: netpol1
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            name: frontend
```

판단:

- `backend` 네임스페이스의 모든 Pod에 적용
- `frontend` 네임스페이스의 모든 Pod를 허용
- 통신은 가능하지만 범위가 너무 넓음

---

### netpol2.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: netpol2
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            name: frontend
        podSelector:
          matchLabels:
            app: frontend
```

판단:

- 대상 Pod를 `app=backend`로 정확히 제한
- 출발지도
  - `name=frontend` 네임스페이스
  - `app=frontend` Pod
  로 좁힘
- 필요한 통신만 최소 범위로 허용하므로 가장 적절함

---

### netpol3.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: netpol3
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            name: frontend
        podSelector:
          matchLabels:
            app: frontend
```

판단:

- 정책 대상이 `app=database`
- 실제 backend Pod label은 `app=backend`
- 따라서 backend Pod에 적용되지 않음

2가 가장 적합
---

##  NetworkPolicy 적용

```bash
kubectl apply -f ~/netpol/netpol2.yaml
```

---

## 5. 적용 확인

```bash
kubectl get netpol -n backend
```

확인 포인트:

- 기존 `deny-all` 정책이 그대로 존재하는지
- 새 정책 `netpol2`가 추가되었는지

deny all은 유지한채 netpol2 가 생성된걸 볼 수 있다.
<img width="882" height="126" alt="image" src="https://github.com/user-attachments/assets/6d18b137-2e04-4c72-8eef-66afcd6635a2" />
