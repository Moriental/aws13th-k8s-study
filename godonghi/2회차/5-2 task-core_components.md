# 5-2 task-core_components 풀이 정리

## 1. 문제 번역

kubeadm으로 구성된 클러스터가 새 머신으로 마이그레이션되었다.

정상 실행을 위해 설정 변경이 필요하다.

해야 할 일:

```text
머신 마이그레이션 중 깨진 단일 노드 클러스터를 수정해야 한다.
어떤 클러스터 컴포넌트가 깨졌는지 식별하고, 왜 깨졌는지 조사해야 한다.
폐기된 기존 클러스터는 외부 etcd 서버를 사용했다.
깨진 모든 클러스터 컴포넌트의 설정을 수정해야 한다.
변경 사항이 적용되도록 필요한 서비스와 컴포넌트를 재시작해야 한다.
마지막으로 클러스터, 단일 노드, 모든 Pod가 Ready 상태인지 확인해야 한다.
```

Quick Reference:

```text
Components
ETCD
```

---


---

## 3. 장애 원리

### kube-apiserver 장애

`kube-apiserver`는 Kubernetes API 서버다.

모든 `kubectl` 요청은 kube-apiserver를 통해 처리된다.

kube-apiserver는 클러스터 상태를 `etcd`에 저장하고 읽는다.

정상 단일 노드 클러스터에서는 etcd 주소가 로컬 etcd여야 한다.

정상 값:

```text
--etcd-servers=https://127.0.0.1:2379
```

문제 구축 후 잘못된 값:

```text
--etcd-servers=https://192.168.56.41:2379
```

의미:

```text
현재 노드의 etcd가 아니라
예전 외부 etcd 서버를 바라보고 있음
```

결과:

```text
kube-apiserver가 etcd에 연결하지 못함
API 서버 장애 또는 불안정
kubectl 명령 실패 가능
클러스터 전체 비정상
```

해결:

```text
192.168.56.41:2379
→ 127.0.0.1:2379
```

---

### kube-scheduler 장애

`kube-scheduler`는 새 Pod를 어떤 Node에 배치할지 결정하는 컴포넌트다.

문제 구축 시 kube-scheduler의 CPU request가 과하게 변경된다.

정상 값:

```text
cpu: 100m
```

잘못된 값:

```text
cpu: 4
```

단일 노드 클러스터에서 scheduler가 CPU 4개를 request하면, 노드 자원이 부족한 환경에서는 scheduler Pod가 정상적으로 뜨지 못할 수 있다.

문제에 추가된 주석도 힌트다.

```text
It is recommended to allocate kube-scheduler's resources to 10% of the worker node's resources.
```

의미:

```text
kube-scheduler 리소스는 worker node 리소스의 10% 정도로 설정하는 것이 권장된다.
```

해결:

```text
cpu: 4
→ cpu: 100m
```

---

## 4. 풀이 전 확인

kube-apiserver manifest 확인:

```bash
grep -- '--etcd-servers' /etc/kubernetes/manifests/kube-apiserver.yaml
```

잘못된 상태 예시:

```text
--etcd-servers=https://192.168.56.41:2379
```

kube-scheduler manifest 확인:

```bash
grep -A10 -B5 'resources:' /etc/kubernetes/manifests/kube-scheduler.yaml
```

잘못된 상태 예시:

```text
resources:
  requests:
    cpu: 4
```

Pod 상태 확인:

```bash
kubectl get pods -n kube-system
```

Node 상태 확인:

```bash
kubectl get nodes
```

---

## 5. 풀이 명령어

### kube-apiserver etcd 주소 수정

파일 직접 수정:

```bash
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

수정 전:

```yaml
- --etcd-servers=https://192.168.56.41:2379
```

수정 후:

```yaml
- --etcd-servers=https://127.0.0.1:2379
```

또는 명령어로 수정:

```bash
sudo sed -i 's|--etcd-servers=https://192.168.56.41:2379|--etcd-servers=https://127.0.0.1:2379|' /etc/kubernetes/manifests/kube-apiserver.yaml
```

---

### kube-scheduler CPU request 수정

파일 직접 수정:

```bash
sudo vi /etc/kubernetes/manifests/kube-scheduler.yaml
```

수정 전:

```yaml
cpu: 4
```

수정 후:

```yaml
cpu: 100m
```

또는 명령어로 수정:

```bash
sudo sed -i 's/cpu: 4/cpu: 100m/' /etc/kubernetes/manifests/kube-scheduler.yaml
```

---

### kubelet 재시작

Static Pod manifest를 수정하면 kubelet이 자동으로 control plane Pod를 다시 생성한다.

문제에서 필요한 서비스와 컴포넌트를 재시작하라고 했으므로 kubelet을 재시작한다.

```bash
sudo systemctl restart kubelet
```

---

## 6. 확인 명령어

kubelet 상태 확인:

```bash
sudo systemctl status kubelet --no-pager
```

kube-system Pod 확인:

```bash
kubectl get pods -n kube-system
```

확인할 주요 컴포넌트:

```text
etcd-k8s-master
kube-apiserver-k8s-master
kube-controller-manager-k8s-master
kube-scheduler-k8s-master
```

모두 `Running`이어야 한다.

Node Ready 확인:

```bash
kubectl get nodes
```

정상 예시:

```text
NAME         STATUS   ROLES           AGE   VERSION
k8s-master   Ready    control-plane    ...   ...
```

전체 Pod 확인:

```bash
kubectl get pods -A
```

모든 Pod가 정상 상태인지 확인한다.

설정값 확인:

```bash
grep -- '--etcd-servers' /etc/kubernetes/manifests/kube-apiserver.yaml
```

정상 값:

```text
--etcd-servers=https://127.0.0.1:2379
```

```bash
grep -A10 -B5 'resources:' /etc/kubernetes/manifests/kube-scheduler.yaml
```

정상 값:

```text
resources:
  requests:
    cpu: 100m
```

---

## 7. 핵심 정리

이 문제에서 깨진 컴포넌트:

```text
kube-apiserver
kube-scheduler
```

kube-apiserver 문제:

```text
etcd 서버 주소가 예전 외부 etcd 주소로 되어 있음
192.168.56.41:2379
```

수정:

```text
127.0.0.1:2379
```

kube-scheduler 문제:

```text
CPU request가 너무 큼
cpu: 4
```

수정:

```text
cpu: 100m
```

최종 성공 기준:

```text
kubelet active
kube-apiserver Running
kube-scheduler Running
etcd Running
node Ready
전체 Pod Ready
```

---

## 8. 정리 명령어

이 문제는 클러스터 핵심 컴포넌트를 정상 상태로 복구하는 문제라, 별도의 삭제 명령어는 없다.

정상 상태로 돌린 값 자체가 정리 완료 상태다.

확인만 하면 된다.

```bash
grep -- '--etcd-servers' /etc/kubernetes/manifests/kube-apiserver.yaml
grep -A10 -B5 'resources:' /etc/kubernetes/manifests/kube-scheduler.yaml
kubectl get nodes
kubectl get pods -A
```
