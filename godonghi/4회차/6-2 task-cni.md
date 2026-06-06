# 6-2 task-cni 핵심 정리

## 1. 문제 핵심

요구사항을 만족하는 CNI를 선택해 설치하고 구성하는 문제다.

조건:

```text
Pod 간 통신 가능
NetworkPolicy enforcement 지원
Helm 사용 금지
manifest 파일로 설치
```

Flannel과 Calico 중 선택할 수 있지만, NetworkPolicy 지원 조건 때문에 Calico를 선택한다.

---

## 2. 필요한 개념

CNI는 Kubernetes Pod 네트워크를 담당하는 플러그인이다.

주요 역할:

```text
Pod IP 할당
Pod 간 통신
Node 간 Pod 통신
NetworkPolicy 적용
```

Flannel:

```text
설치는 단순함
Pod 간 통신 가능
기본적으로 NetworkPolicy enforcement 지원 안 함
```

Calico:

```text
Pod 간 통신 가능
NetworkPolicy enforcement 지원
manifest 설치 가능
```

따라서 이 문제에서는 Calico가 더 적절하다.

---

## 3. 풀이 명령어

Calico operator manifest 적용:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/tigera-operator.yaml
```

Calico custom resources 적용:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/custom-resources.yaml
```

---

## 4. 확인 명령어

```bash
kubectl get pods -n tigera-operator
kubectl get pods -n calico-system
kubectl get nodes
kubectl get pods -A
kubectl api-resources | grep -i networkpolicy
```

확인 결과:

```text
calico-system     calico-apiserver-68b4f6d4f4-l94cd          1/1     Running
calico-system     calico-apiserver-68b4f6d4f4-mj8th          1/1     Running
calico-system     calico-kube-controllers-77d588f9c4-f8mmn   1/1     Running
calico-system     calico-node-9jj59                          1/1     Running
calico-system     calico-typha-7f97474654-ghlg4              1/1     Running
```

Node 상태:

```text
NAME         STATUS   ROLES           AGE   VERSION
k8s-master   Ready    control-plane   41d   v1.34.3
```

NetworkPolicy 지원 확인:

```text
networkpolicies                     netpol                                          networking.k8s.io/v1                true         NetworkPolicy
networkpolicies                     cnp,caliconetworkpolicy,caliconetworkpolicies   projectcalico.org/v3                true         NetworkPolicy
globalnetworkpolicies                                                               crd.projectcalico.org/v1            false        GlobalNetworkPolicy
globalnetworkpolicies               gnp,cgnp,calicoglobalnetworkpolicies            projectcalico.org/v3                false        GlobalNetworkPolicy
```

---

## 5. 발생한 이슈

`tigera-operator` Pod가 `CrashLoopBackOff` 상태가 됐다.

확인:

```bash
kubectl logs -n tigera-operator deploy/tigera-operator
```

로그 핵심:

```text
Failed to ensure CRDs are created
cannot create resource "customresourcedefinitions"
User "system:serviceaccount:tigera-operator:tigera-operator" cannot create resource "customresourcedefinitions"
```

권한 확인:

```bash
kubectl auth can-i create customresourcedefinitions \
  --as=system:serviceaccount:tigera-operator:tigera-operator
```

결과:

```text
no
```

원인:

```text
이미 Calico가 설치되어 정상 동작 중인 환경에서 tigera-operator를 재적용했다.
새로 뜬 tigera-operator가 CRD 생성을 시도했지만 RBAC 권한이 없어 CrashLoopBackOff가 발생했다.
Calico CNI 자체 문제는 아니다.
```

---

## 6. 조치

Calico 본체는 정상 동작 중이므로 CrashLoopBackOff 상태인 `tigera-operator` 네임스페이스만 정리했다.

```bash
kubectl delete ns tigera-operator
```

정리 후 확인:

```bash
kubectl get pods -A
kubectl get nodes
kubectl get pods -n calico-system
kubectl api-resources | grep -i networkpolicy
```

정리 후 상태:

```text
calico-system Pod들 Running
node Ready
NetworkPolicy 리소스 확인됨
tigera-operator CrashLoopBackOff 없음
```

---

## 7. 최종 성공 기준

```text
Calico CNI 정상 동작
calico-node Running
calico-kube-controllers Running
Node Ready
NetworkPolicy 지원 확인
CrashLoopBackOff 없음
```

---
