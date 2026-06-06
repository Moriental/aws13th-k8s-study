# CKA CNI 설치 문제 정리 - Calico 선택

## 문제 요약

Kubernetes 클러스터에 CNI(Container Network Interface)를 설치하고 설정하는 문제이다.

문제에서는 두 가지 CNI 선택지를 제공한다.

1. Flannel
2. Calico

하지만 요구사항에 `NetworkPolicy enforcement`가 포함되어 있으므로, 이 문제에서는 Calico를 선택하는 것이 적절하다.

---

## 문제 요구사항

문제에서 요구하는 CNI 조건은 다음과 같다.

```text
The CNI you choose must:
• Let Pods communicate with each other
• Support Network Policy enforcement
• Install from manifest files (do not use Helm)
```

뜻:

선택한 CNI는 반드시 다음 조건을 만족해야 한다.

1. Pod 간 통신이 가능해야 한다.
2. NetworkPolicy 적용을 지원해야 한다.
3. Helm이 아니라 manifest 파일로 설치해야 한다.

---

## 핵심 판단

이 문제의 핵심은 `NetworkPolicy enforcement` 조건이다.

Flannel은 기본적인 Pod 네트워크 연결에는 사용할 수 있지만, 일반적인 Flannel 단독 설치는 NetworkPolicy enforcement를 지원하는 선택지로 보기 어렵다.

반면 Calico는 Pod 네트워킹과 NetworkPolicy enforcement를 모두 지원한다.

따라서 이 문제에서는 Calico를 선택하는 것이 안전하다.

```text
Pod communication만 필요하면 Flannel도 가능
NetworkPolicy enforcement까지 필요하면 Calico 선택
```

---

## 최종 요구사항 한 줄 정리

```text
Flannel이 아니라 Calico를 manifest 방식으로 설치해서,
Pod 간 통신과 NetworkPolicy enforcement가 가능한 CNI 환경을 구성한다.
```

---

## 시험장에서 접근하는 사고방식

문제를 보면 먼저 조건을 뽑는다.

```text
CNI 설치 문제
Flannel 또는 Calico 선택 가능
Pod 간 통신 필요
NetworkPolicy enforcement 필요
Helm 사용 금지
Manifest 파일로 설치
```

그다음 이렇게 판단한다.

```text
NetworkPolicy enforcement 조건 있음
→ Flannel 단독 설치는 부적절
→ Calico 선택
→ Helm 금지이므로 kubectl apply -f 방식 사용
```

---

## 실습 환경에서 이미 Calico가 설치되어 있던 경우

실습 중 다음 명령어를 실행했다.

```bash
kubectl get ds -A
```

출력 결과:

```text
NAMESPACE       NAME              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
calico-system   calico-node       1         1         1       1            1           kubernetes.io/os=linux   21d
calico-system   csi-node-driver   1         1         1       1            1           kubernetes.io/os=linux   21d
kube-system     kube-proxy        1         1         1       1            1           kubernetes.io/os=linux   21d
```

이 출력에서 중요한 부분은 다음이다.

```text
calico-system   calico-node       READY 1
calico-system   csi-node-driver   READY 1
```

즉, 현재 클러스터에는 이미 Calico CNI가 설치되어 있다.

따라서 이 상황에서는 Flannel을 추가로 설치하면 안 된다.

CNI는 보통 하나의 클러스터에 하나만 사용하는 것이 원칙이다.
이미 Calico가 있는데 Flannel을 추가 설치하면 네트워크가 꼬일 수 있다.

---

## 이미 Calico가 설치된 경우 해야 할 일

이미 Calico가 설치되어 있다면 새로 설치하지 말고 상태를 확인한다.

### 1. DaemonSet 확인

```bash
kubectl get ds -A
```

확인할 부분:

```text
calico-system   calico-node       READY 1
```

### 2. Calico Pod 확인

```bash
kubectl get pods -n calico-system
```

정상 기준:

```text
calico-node-...              1/1   Running
calico-kube-controllers-...  1/1   Running
csi-node-driver-...          1/1   Running
```

### 3. Node Ready 확인

```bash
kubectl get nodes
```

정상 기준:

```text
STATUS = Ready
```

### 4. 전체 Pod 확인

```bash
kubectl get pods -A
```

확인할 부분:

```text
CoreDNS Running
Calico 관련 Pod Running
Node Ready
```

---

## 이미 설치되어 있을 때 결론

```text
Calico가 이미 설치되어 있음
→ NetworkPolicy enforcement 조건 만족 가능
→ Flannel 추가 설치 금지
→ Calico Pod와 Node Ready 상태만 확인
```

이 경우 문제 풀이 방향은 다음과 같다.

```text
설치 명령을 다시 치는 문제가 아니라,
이미 조건을 만족하는 CNI가 설치되어 있는지 확인하는 문제로 판단한다.
```

---

## 실제 시험장에서 CNI가 설치되어 있지 않은 경우

실제 시험에서는 CNI가 미리 설치되어 있지 않을 수 있다.

따라서 먼저 CNI 존재 여부를 확인해야 한다.

### 1. 지정된 host 접속

문제 상단에 다음과 같이 되어 있으면 먼저 해당 host로 접속한다.

```bash
ssh cka0001
```

접속 후 클러스터 상태 확인:

```bash
kubectl get nodes
```

CNI가 없으면 Node가 `NotReady`일 수 있다.

---

### 2. 기존 CNI 설치 여부 확인

```bash
kubectl get ds -A
```

또는:

```bash
kubectl get pods -A | egrep 'calico|flannel|cilium|weave'
```

확인 기준:

```text
calico-system 네임스페이스에 calico-node가 있으면 Calico 설치됨
kube-flannel 네임스페이스에 kube-flannel-ds가 있으면 Flannel 설치됨
아무것도 없으면 CNI 미설치 가능성 높음
```

---

## CNI가 없는 경우 Calico 설치

문제에서 Calico manifest URL이 제공된다.

```text
https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/tigera-operator.yaml
```

문제 조건에 `do not use Helm`이 있으므로 Helm을 사용하면 안 된다.

잘못된 예:

```bash
helm install calico ...
```

올바른 방식:

```bash
kubectl apply -f <manifest-url>
```

---

## 1단계. Tigera Operator 설치

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/tigera-operator.yaml
```

확인:

```bash
kubectl get pods -n tigera-operator
```

정상 예시:

```text
tigera-operator-...   1/1   Running
```

---

## 2단계. Calico custom-resources.yaml 준비

Calico operator 방식은 operator만 설치하고 끝나는 것이 아니라, Calico 설치 설정을 담은 custom resource를 적용해야 한다.

공식 Calico 설치 흐름에서는 `custom-resources.yaml`을 함께 적용한다.

```bash
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/custom-resources.yaml
```

파일 확인:

```bash
ls -l custom-resources.yaml
```

---

## 3단계. Pod CIDR 확인

Calico custom-resources.yaml에는 기본 IPPool CIDR이 들어 있다.

이 값이 현재 클러스터의 Pod CIDR과 맞아야 한다.

현재 Node의 Pod CIDR 확인:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.podCIDR}{"\n"}{end}'
```

예시 1:

```text
k8s-master 192.168.0.0/24
k8s-worker1 192.168.1.0/24
```

이 경우 전체 Pod CIDR은 `192.168.0.0/16` 계열일 가능성이 높다.

예시 2:

```text
k8s-master 10.244.0.0/24
k8s-worker1 10.244.1.0/24
```

이 경우 전체 Pod CIDR은 `10.244.0.0/16` 계열일 가능성이 높다.

---

## 4단계. custom-resources.yaml의 CIDR 확인

```bash
grep -n "cidr:" custom-resources.yaml
```

예상:

```text
cidr: 192.168.0.0/16
```

만약 현재 클러스터 Pod CIDR이 `10.244.0.0/16` 계열이면 파일을 수정해야 한다.

```bash
vi custom-resources.yaml
```

수정 예시:

```yaml
cidr: 10.244.0.0/16
```

또는 sed 사용:

```bash
sed -i 's#192.168.0.0/16#10.244.0.0/16#' custom-resources.yaml
```

주의:

```text
무조건 10.244.0.0/16으로 바꾸는 것이 아니다.
현재 클러스터의 Pod CIDR을 확인하고 그에 맞춰야 한다.
```

---

## 5단계. Calico custom resources 적용

```bash
kubectl apply -f custom-resources.yaml
```

이제 Calico가 실제로 설치된다.

확인:

```bash
kubectl get pods -n calico-system
```

또는:

```bash
kubectl get pods -A | grep calico
```

정상 예시:

```text
calico-node-...              1/1   Running
calico-kube-controllers-...  1/1   Running
calico-typha-...             1/1   Running
csi-node-driver-...          1/1   Running
```

---

## 6단계. Node Ready 확인

```bash
kubectl get nodes
```

정상 기준:

```text
STATUS = Ready
```

CNI 설치 직후에는 조금 시간이 걸릴 수 있다.

상태를 반복 확인한다.

```bash
kubectl get pods -n calico-system
kubectl get nodes
```

---

## Pod 간 통신 확인

문제 요구사항 중 하나는 Pod 간 통신 가능이다.

학습할 때는 간단한 테스트를 해보는 것이 좋다.

### 테스트 namespace 생성

```bash
kubectl create ns cni-test
```

### nginx Pod 생성

```bash
kubectl run web -n cni-test --image=nginx --labels=app=web --port=80
```

### Service 생성

```bash
kubectl expose pod web -n cni-test --port=80
```

### Pod 상태 확인

```bash
kubectl get pods -n cni-test -o wide
```

### busybox로 접근 테스트

```bash
kubectl run test-client -n cni-test --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- web
```

nginx HTML이 나오면 Pod 간 통신이 되는 것이다.

---

## NetworkPolicy enforcement 확인

문제의 핵심 요구사항은 NetworkPolicy enforcement 지원이다.

Calico를 선택한 이유가 바로 이것이다.

### 1. web Pod로 들어오는 ingress 차단

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-web-ingress
  namespace: cni-test
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
EOF
```

이 정책은 `app=web` Pod로 들어오는 ingress 트래픽을 기본 차단한다.

접속 테스트:

```bash
kubectl run test-client -n cni-test --rm -it --image=busybox:1.36 --restart=Never -- wget --timeout=3 -qO- web
```

정상적으로 NetworkPolicy가 적용되면 timeout 또는 실패가 발생한다.

---

### 2. 특정 label이 있는 Pod만 허용

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-from-allowed-client
  namespace: cni-test
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: allowed
    ports:
    - protocol: TCP
      port: 80
EOF
```

이제 `access=allowed` label이 있는 Pod만 web Pod에 접근할 수 있다.

테스트:

```bash
kubectl run allowed-client -n cni-test --rm -it \
  --image=busybox:1.36 \
  --labels=access=allowed \
  --restart=Never \
  -- wget -qO- web
```

nginx HTML이 나오면 NetworkPolicy enforcement가 정상적으로 동작하는 것이다.

테스트 후 정리:

```bash
kubectl delete ns cni-test
```

---

## 최종 정답 명령어 흐름 - CNI가 없는 경우

실제 시험에서 CNI가 없다고 판단되면 다음 흐름으로 진행한다.

```bash
ssh cka0001
```

```bash
kubectl get nodes
kubectl get ds -A
kubectl get pods -A | egrep 'calico|flannel|cilium|weave'
```

Calico 설치:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/tigera-operator.yaml
```

custom resources 다운로드:

```bash
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/custom-resources.yaml
```

Pod CIDR 확인:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.podCIDR}{"\n"}{end}'
```

custom-resources.yaml CIDR 확인:

```bash
grep -n "cidr:" custom-resources.yaml
```

필요 시 수정:

```bash
vi custom-resources.yaml
```

또는 예를 들어 Pod CIDR이 `10.244.0.0/16`인 경우:

```bash
sed -i 's#192.168.0.0/16#10.244.0.0/16#' custom-resources.yaml
```

적용:

```bash
kubectl apply -f custom-resources.yaml
```

확인:

```bash
kubectl get pods -n tigera-operator
kubectl get pods -n calico-system
kubectl get nodes
```

---

## 최종 정답 명령어 흐름 - 이미 Calico가 있는 경우

이미 다음과 같이 Calico가 설치되어 있다면:

```text
calico-system   calico-node       READY 1
calico-system   csi-node-driver   READY 1
```

추가 설치하지 않는다.

확인만 수행한다.

```bash
kubectl get nodes
kubectl get pods -n calico-system
kubectl get ds -A
kubectl get pods -A
```

정상 기준:

```text
calico-node Running
Node Ready
CoreDNS Running
```

---

## 자주 하는 실수

### 실수 1. Flannel 선택

문제에 Flannel manifest가 있어서 Flannel을 설치하고 싶어질 수 있다.

하지만 요구사항에 다음이 있다.

```text
Support Network Policy enforcement
```

이 조건 때문에 Calico를 선택하는 것이 안전하다.

---

### 실수 2. Helm 사용

문제에 다음이 있다.

```text
Install from manifest files (do not use Helm)
```

따라서 Helm을 사용하면 안 된다.

잘못된 예:

```bash
helm install calico ...
```

올바른 예:

```bash
kubectl apply -f <manifest-url>
```

---

### 실수 3. 이미 CNI가 있는데 다른 CNI 추가 설치

이미 Calico가 설치되어 있는데 Flannel을 추가 설치하면 안 된다.

```text
Calico 있음 → Flannel 설치 X
Flannel 있음 → Calico를 덮어쓰기 전에 문제 조건과 상태를 신중히 확인
CNI 없음 → 조건에 맞는 CNI 설치
```

이 문제에서는 NetworkPolicy 조건 때문에 Calico가 맞다.

---

### 실수 4. Tigera Operator만 설치하고 끝내기

아래 명령어만 실행하고 끝내면 부족할 수 있다.

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/tigera-operator.yaml
```

이 명령어는 Tigera Operator 설치이다.

Calico 설치 구성을 위해 보통 custom resource 적용이 필요하다.

따라서 다음까지 진행한다.

```bash
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/custom-resources.yaml
kubectl apply -f custom-resources.yaml
```

---

### 실수 5. Pod CIDR 불일치

custom-resources.yaml의 CIDR과 클러스터 Pod CIDR이 맞지 않으면 Node가 Ready가 되지 않거나 Pod 네트워크가 정상 동작하지 않을 수 있다.

확인:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.podCIDR}{"\n"}{end}'
grep -n "cidr:" custom-resources.yaml
```

---

## 최종 확인 기준

### 1. Calico operator 확인

```bash
kubectl get pods -n tigera-operator
```

정상:

```text
tigera-operator-...   1/1   Running
```

### 2. Calico system 확인

```bash
kubectl get pods -n calico-system
```

정상:

```text
calico-node-...              1/1   Running
calico-kube-controllers-...  1/1   Running
csi-node-driver-...          1/1   Running
```

### 3. Node Ready 확인

```bash
kubectl get nodes
```

정상:

```text
STATUS = Ready
```

### 4. 전체 Pod 확인

```bash
kubectl get pods -A
```

CoreDNS와 Calico 관련 Pod가 Running인지 확인한다.

---

## 시험장에서 최종 판단 흐름

```text
1. 문제에서 CNI 설치 요구 확인
2. 선택지 확인: Flannel / Calico
3. 요구사항 확인: NetworkPolicy enforcement
4. Calico 선택
5. 기존 CNI 확인
   - Calico가 이미 있으면 재설치하지 않고 상태 확인
   - CNI가 없으면 Calico manifest 설치
6. Helm 금지 조건 확인
7. kubectl apply -f 방식으로 설치
8. Node Ready, Calico Pod Running 확인
```

---

## 한 줄 핵심 암기

```text
NetworkPolicy enforcement 조건이 있으면 Calico를 선택한다.
CNI가 이미 설치되어 있으면 추가 설치하지 말고 상태를 확인한다.
CNI가 없으면 Helm이 아니라 manifest로 Calico를 설치한다.
```
