# Control Plane 장애 복구 문제 정리

이 문제는 너무 어려워서 다 ai 한테 물어보고 따라하면서 실습하는 식으로 했습니다..


## 문제 요약

머신 마이그레이션 과정에서 망가진  
단일 노드 kubeadm 클러스터를 복구하는 문제이다.

깨진 원인을 직접 찾고,  
관련 설정 파일을 수정한 뒤  
클러스터와 모든 Pod가 정상 상태가 되도록 복구해야 한다.

---

## 요구사항

- 깨진 클러스터 구성 요소 확인
- 장애 원인 조사
- 잘못된 설정 수정
- 필요한 서비스 및 컴포넌트 재시작
- 최종적으로
  - Node `Ready`
  - 모든 Pod 정상 상태
  확인

---

## 핵심 개념

### 1. kube-apiserver

`kubectl` 명령은 모두 `kube-apiserver`를 통해 처리된다.

따라서 `kube-apiserver`가 죽으면:

```bash
kubectl get nodes
kubectl get pods -A
```

같은 명령도 제대로 동작하지 않을 수 있다.

---

### 2. etcd

`etcd`는 Kubernetes 클러스터의 상태 데이터를 저장하는 저장소이다.

```text
Pod, Service, Node, ConfigMap 등
클러스터 주요 정보가 etcd에 저장됨
```

`kube-apiserver`는 반드시 etcd와 연결되어 있어야 한다.

---

### 3. Static Pod

kubeadm으로 구성한 Control Plane 컴포넌트는  
`/etc/kubernetes/manifests/` 아래 YAML 파일을 기준으로 실행된다.

```text
kube-apiserver
kube-scheduler
kube-controller-manager
etcd
```

이 디렉토리의 파일을 수정하면  
`kubelet`이 변경을 감지해 해당 Static Pod를 다시 반영한다.

---

## 1. 문제 구축으로 망가진 지점 이해

### 1) kube-apiserver의 etcd 연결 주소 변경

문제 구축 과정에서 다음 값이 바뀌었다.

```text
정상:
--etcd-servers=https://127.0.0.1:2379

비정상:
--etcd-servers=https://192.168.56.41:2379
```

즉, 현재 단일 노드에서 사용해야 하는 로컬 etcd가 아니라  
존재하지 않는 외부 etcd 주소를 보게 되어  
`kube-apiserver`가 정상 동작하지 못한다.

---

### 2) kube-scheduler의 CPU request 증가

문제 구축 과정에서 다음 값도 바뀌었다.

```text
정상:
cpu: 100m

비정상:
cpu: 4
```

이는 `kube-scheduler`가 CPU 4개를 요청하도록 바뀐 상태이다.  
불필요하게 큰 리소스 요청으로 인해 정상 실행에 문제가 생길 수 있다.

---

## 2. 클러스터 상태 확인

먼저 일반적인 상태 확인을 시도한다.

```bash
kubectl get nodes
kubectl get pods -A
```

### 확인 포인트

- 명령이 정상 실행되지 않는지
- API Server 연결 오류가 발생하는지
- Node 상태가 `NotReady`인지

---

## 3. Control Plane 컨테이너 상태 확인

`kubectl`이 제대로 동작하지 않으면  
컨테이너 런타임을 직접 확인한다.

```bash
sudo crictl ps -a | grep kube
```

### 확인 포인트

특히 다음 컴포넌트를 본다.

```text
kube-apiserver
kube-scheduler
```

- 반복 재시작 중인지
- Exited 상태인지
- Running 상태가 아닌지 확인한다

---

## 4. kube-apiserver 문제 확인

### 1) kube-apiserver 컨테이너 찾기

```bash
sudo crictl ps -a | grep kube-apiserver
```

### 2) 로그 확인

```bash
sudo crictl logs <kube-apiserver-container-id>
```

### 예상 단서

다음과 같은 메시지를 통해 etcd 연결 문제를 확인할 수 있다.

```text
etcd
192.168.56.41:2379
connection refused
context deadline exceeded
```

---

## 5. kube-apiserver manifest 직접 수정

### 1) 현재 설정 확인

```bash
sudo grep etcd-servers /etc/kubernetes/manifests/kube-apiserver.yaml
```

비정상 상태라면 다음처럼 보인다.

```text
--etcd-servers=https://192.168.56.41:2379
```

---

### 2) 파일 직접 편집

```bash
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

아래 항목을 찾는다.

```yaml
- --etcd-servers=https://192.168.56.41:2379
```

이를 다음처럼 수정한다.

```yaml
- --etcd-servers=https://127.0.0.1:2379
```

---

## 6. kube-scheduler 문제 확인

### 1) 현재 resource request 확인

```bash
sudo grep -A 8 -B 3 requests /etc/kubernetes/manifests/kube-scheduler.yaml
```

비정상 상태라면 아래처럼 보인다.

```yaml
resources:
  requests:
    cpu: 4
```

---

## 7. kube-scheduler manifest 직접 수정

```bash
sudo vi /etc/kubernetes/manifests/kube-scheduler.yaml
```

다음 값을 찾는다.

```yaml
cpu: 4
```

아래처럼 수정한다.

```yaml
cpu: 100m
```

---

## 8. 변경 사항 반영

Static Pod manifest는 kubelet이 감시하므로  
파일을 저장하면 자동으로 반영된다.

그래도 문제에서 필요한 서비스와 컴포넌트 재시작을 요구하므로  
kubelet을 재시작해 적용을 확실히 한다.

```bash
sudo systemctl restart kubelet
```

---

## 9. 복구 상태 확인

### 1) Node 상태 확인

```bash
kubectl get nodes
```

정상 예시:

```text
NAME         STATUS   ROLES           AGE   VERSION
k8s-master   Ready    control-plane   ...   ...
```

---

### 2) 모든 Pod 상태 확인

```bash
kubectl get pods -A
```

확인 포인트:

- Control Plane Pod 정상
- 시스템 Pod 정상
- 전체 Pod가 `Running` 또는 정상 상태

---

## 10. 최종 확인 명령어

```bash
kubectl get nodes
kubectl get pods -A
```

추가로 설정이 정상적으로 수정됐는지 확인할 수 있다.

```bash
sudo grep etcd-servers /etc/kubernetes/manifests/kube-apiserver.yaml
sudo grep -A 8 -B 3 requests /etc/kubernetes/manifests/kube-scheduler.yaml
```

정상값:

```text
--etcd-servers=https://127.0.0.1:2379
cpu: 100m
```

---



실제 출력 값
<img width="748" height="88" alt="image" src="https://github.com/user-attachments/assets/df384cca-0535-4b66-a0bb-e73de0fd90b0" />


## 풀이 핵심 정리

```text
1. kubectl 명령이 안 되면 kube-apiserver 장애를 의심한다.
2. crictl로 Control Plane 컨테이너 상태를 확인한다.
3. kube-apiserver 로그에서 etcd 연결 문제를 확인한다.
4. kube-apiserver manifest의 etcd 주소를 로컬 etcd로 복구한다.
5. kube-scheduler manifest에서 비정상적인 CPU request를 원래 값으로 복구한다.
6. kubelet을 재시작하고 Node와 Pod 상태를 확인한다.
```
