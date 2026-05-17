# 4-2 task-networkpolicy 풀이 정리

## 1. 문제 번역

`frontend`와 `backend` Deployment가 각각 별도의 네임스페이스인 `frontend`, `backend`에 존재한다.

두 Deployment는 서로 통신해야 한다.

`frontend`와 `backend` Deployment를 확인하여 통신 요구사항을 분석하시오.

`~/netpol` 폴더 안에 있는 NetworkPolicy YAML 파일들 중 하나를 선택하여 적용하시오.

선택한 NetworkPolicy는 다음 조건을 만족해야 한다.

```text
frontend와 backend 사이의 통신을 허용해야 함
가능한 한 제한적이어야 함
기존 deny-all NetworkPolicy를 삭제하거나 변경하면 안 됨
```

이 규칙을 지키지 않으면 감점 또는 0점 처리될 수 있다.

---


---

## 3. 기존 리소스 확인

Pod 라벨 확인:

```bash
kubectl get pod -n frontend --show-labels
kubectl get pod -n backend --show-labels
```

확인된 라벨:

```text
frontend Pod: app=frontend
backend Pod: app=backend
```

기존 NetworkPolicy 확인:

```bash
kubectl get networkpolicy -n backend
```

기존 상태:

```text
NAME       POD-SELECTOR
deny-all   <none>
```

`deny-all`은 기존 차단 정책이므로 삭제하거나 수정하면 안 된다.

---

## 4. 후보 파일 확인

```bash
ls ~/netpol
```

출력:

```text
netpol1.yaml  netpol2.yaml  netpol3.yaml
```

---

## 5. netpol1.yaml 내용과 해석

확인 명령어:

```bash
cat ~/netpol/netpol1.yaml
```

내용:

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

해석:

```text
NetworkPolicy 이름: netpol1
적용 네임스페이스: backend
대상 Pod: backend 네임스페이스의 모든 Pod
허용 방향: Ingress
허용 출발지: name=frontend 라벨이 붙은 namespace
```

`podSelector: {}`는 해당 네임스페이스의 모든 Pod를 의미한다.

문제점:

```text
backend 네임스페이스의 모든 Pod를 대상으로 하므로 너무 넓다.
문제에서 요구한 “가능한 한 제한적” 조건에 가장 잘 맞지는 않는다.
```

---

## 6. netpol2.yaml 내용과 해석

확인 명령어:

```bash
cat ~/netpol/netpol2.yaml
```

내용:

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

해석:

```text
NetworkPolicy 이름: netpol2
적용 네임스페이스: backend
대상 Pod: backend 네임스페이스의 app=backend Pod
허용 방향: Ingress
허용 출발지 namespace: name=frontend
허용 출발지 Pod: app=frontend
```

의미:

```text
frontend 네임스페이스 안의 app=frontend Pod에서
backend 네임스페이스 안의 app=backend Pod로 들어오는 트래픽만 허용한다.
```

이 파일이 정답인 이유:

```text
현재 frontend Pod 라벨은 app=frontend
현재 backend Pod 라벨은 app=backend
대상 Pod와 출발지 Pod를 모두 라벨로 제한함
후보 중 가장 제한적임
기존 deny-all을 삭제하거나 수정하지 않음
```

---

## 7. netpol3.yaml 내용과 해석

확인 명령어:

```bash
cat ~/netpol/netpol3.yaml
```

내용:

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

해석:

```text
NetworkPolicy 이름: netpol3
적용 네임스페이스: backend
대상 Pod: backend 네임스페이스의 app=database Pod
허용 방향: Ingress
허용 출발지 namespace: name=frontend
허용 출발지 Pod: app=frontend
```

문제점:

```text
현재 backend Pod 라벨은 app=backend이다.
netpol3는 app=database Pod를 대상으로 한다.
따라서 실제 backend Pod에 적용되지 않으므로 정답이 아니다.
```

---

## 8. 확인

정답은 `netpol2.yaml`.

```bash
kubectl apply -f ~/netpol/netpol2.yaml
```

정상 출력:

```[root@k8s-master ~]# kubectl get networkpolicy -n backend
NAME       POD-SELECTOR   AGE
deny-all   <none>         6m5s
netpol2    app=backend    51s
[root@k8s-master ~]# kubectl describe networkpolicy netpol2 -n backend
Name:         netpol2
Namespace:    backend
Created on:   2026-05-17 18:50:52 +0900 KST
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     app=backend
  Allowing ingress traffic:
    To Port: <any> (traffic allowed to all ports)
    From:
      NamespaceSelector: name=frontend
      PodSelector: app=frontend
  Not affecting egress traffic
  Policy Types: Ingress
[root@k8s-master ~]# ^C
```
