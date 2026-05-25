# PriorityClass를 이용한 Deployment 우선순위 설정

## 문제 요약

`priority` 네임스페이스에 있는 `busybox-logger` Deployment가 새로 생성한 `high-priority` PriorityClass를 사용하도록 수정하는 문제이다.

`high-priority` PriorityClass는 기존 사용자 정의 PriorityClass 중 가장 높은 값보다 1 작은 값으로 생성해야 한다.

또한 `busybox-logger` Deployment가 새 PriorityClass를 적용한 상태로 정상 롤아웃되어야 한다.

## 요구사항

- 올바른 노드에 접속한다.
- `high-priority` 이름의 PriorityClass를 생성한다.
- PriorityClass의 값은 기존 사용자 정의 PriorityClass 중 가장 높은 값보다 1 작게 설정한다.
- `priority` 네임스페이스의 `busybox-logger` Deployment만 수정한다.
- `busybox-logger` Deployment의 Pod template에 `priorityClassName: high-priority`를 설정한다.
- 다른 Deployment는 수정하지 않는다.
- 최종적으로 `busybox-logger` Deployment가 정상 롤아웃되었는지 확인한다.

## 핵심 개념

PriorityClass는 Pod의 우선순위를 지정하는 리소스이다.

PriorityClass를 생성했다고 해서 기존 Pod에 자동으로 적용되는 것은 아니다.

Deployment에서 PriorityClass를 적용하려면 기존 Pod를 직접 수정하는 것이 아니라 Deployment의 Pod template에 `priorityClassName`을 추가해야 한다.

```text
PriorityClass 생성
        ↓
Deployment의 Pod template 수정
        ↓
새 ReplicaSet / 새 Pod 생성
        ↓
새 Pod에 priorityClassName 적용
```

중요한 위치는 다음과 같다.

```yaml
spec:
  template:
    spec:
      priorityClassName: high-priority
      containers:
      - name: busybox
```

`priorityClassName`은 컨테이너 내부가 아니라 `containers`와 같은 레벨에 작성해야 한다.

---

## 1. 올바른 노드 접속

문제에서 지정한 노드로 접속한다.

```bash
ssh cka0001
```

잘못된 노드에서 작업하면 점수를 받을 수 없으므로 먼저 접속 위치를 확인한다.

---

## 2. 기존 PriorityClass 확인

먼저 현재 존재하는 PriorityClass를 확인한다.

```bash
kubectl get priorityclass
```

또는 짧게 다음 명령도 사용할 수 있다.

```bash
kubectl get pc
```

출력 예시:

```bash
NAME                      VALUE        GLOBAL-DEFAULT   AGE   PREEMPTIONPOLICY
system-cluster-critical   2000000000   false            9d    PreemptLowerPriority
system-node-critical      2000001000   false            9d    PreemptLowerPriority
```

`system-cluster-critical`, `system-node-critical`은 Kubernetes 시스템용 PriorityClass이다.

문제에서 말하는 `user-defined priority class`는 사용자가 만든 PriorityClass를 의미한다.

---

## 3. high-priority PriorityClass 생성

실습에서는 `high-priority` PriorityClass를 생성하였다.

```bash
kubectl create priorityclass high-priority \
  --value=10000 \
  --global-default=false \
  --description="PriorityClass for user-workloads"
```

생성 확인:

```bash
kubectl get priorityclass
```

특정 PriorityClass만 확인:

```bash
kubectl describe priorityclass high-priority
```

확인해야 할 부분:

```text
Name:   high-priority
Value:  10000
```

---

## 4. Deployment 상태 확인

`priority` 네임스페이스의 Deployment를 확인한다.

```bash
kubectl get deploy -n priority
```

출력 예시:

```bash
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
busybox-logger   0/3     3            0           8m21s
others           3/3     3            3           8m28s
```

현재 `busybox-logger`는 정상 실행되지 못하고 있고, `others`가 실행 중인 상태이다.

문제에서는 `busybox-logger`에 높은 우선순위를 부여하면 다른 Deployment의 Pod가 evicted 될 수 있다고 안내하고 있다.

단, 다른 Deployment 자체는 수정하면 안 된다.

---

## 5. busybox-logger Deployment 수정

`busybox-logger` Deployment의 Pod template에 `priorityClassName: high-priority`를 추가한다.

```bash
kubectl edit deployment busybox-logger -n priority
```

수정해야 하는 위치:

```yaml
spec:
  template:
    spec:
      priorityClassName: high-priority
      containers:
      - command:
        - sh
        - -c
        - echo "Hello, Kubernetes!" && sleep 3600
        image: busybox
        imagePullPolicy: Always
        name: busybox
```

핵심은 다음 위치이다.

```text
spec.template.spec.priorityClassName
```

잘못된 위치 예시:

```yaml
containers:
- name: busybox
  priorityClassName: high-priority
```

이 위치는 틀렸다. `priorityClassName`은 컨테이너 설정이 아니라 Pod spec 설정이다.

---

## 6. patch 명령으로 수정하는 방법

시험에서 빠르게 처리하려면 `kubectl patch`를 사용할 수도 있다.

```bash
kubectl patch deployment busybox-logger -n priority \
  --type='merge' \
  -p '{"spec":{"template":{"spec":{"priorityClassName":"high-priority"}}}}'
```

이 명령은 Deployment의 Pod template에 다음 내용을 추가하는 것과 같다.

```yaml
spec:
  template:
    spec:
      priorityClassName: high-priority
```

연습할 때는 `kubectl edit`로 구조를 익히고, 시험에서는 상황에 따라 `patch`를 사용하면 된다.

---

## 7. 롤아웃 상태 확인

Deployment가 정상적으로 롤아웃되었는지 확인한다.

```bash
kubectl rollout status deployment/busybox-logger -n priority
```

정상 출력 예시:

```bash
deployment "busybox-logger" successfully rolled out
```

---

## 8. 최종 확인

Deployment 상태 확인:

```bash
kubectl get deploy -n priority
```

정상 목표:

```bash
NAME             READY   UP-TO-DATE   AVAILABLE
busybox-logger   3/3     3            3
```

Deployment에 PriorityClass가 적용되었는지 확인:

```bash
kubectl get deployment busybox-logger -n priority \
  -o jsonpath='{.spec.template.spec.priorityClassName}{"\n"}'
```

정상 출력:

```bash
high-priority
```

Pod에도 적용되었는지 확인:

```bash
kubectl get pods -n priority
```

`busybox-logger` Pod 이름을 확인한 뒤 다음 명령을 실행한다.

```bash
kubectl get pod <busybox-logger-pod-name> -n priority \
  -o jsonpath='{.spec.priorityClassName}{"\n"}'
```

정상 출력:

```bash
high-priority
```

---

## 실제 출력 결과

<img width="1469" height="597" alt="image" src="https://github.com/user-attachments/assets/ce7d1cc0-f00d-4fcb-851d-f1bc4402fc52" />
