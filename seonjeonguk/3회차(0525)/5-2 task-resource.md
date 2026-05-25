# WordPress Deployment Resource Requests 조정

## 문제 요약

`relative-fawn` 네임스페이스에 있는 WordPress 애플리케이션의 Pod resource requests를 조정하는 문제이다.

WordPress는 최종적으로 3개의 replica를 유지해야 하며, 모든 Pod가 `Running` 및 `Ready` 상태가 되어야 한다.

주어진 전체 리소스는 다음과 같다.

```text
CPU: 1
Memory: 1024Mi
```

이 리소스를 3개의 WordPress Pod가 공평하게 사용할 수 있도록 나누되, 노드가 안정적으로 동작할 수 있도록 약간의 여유 리소스를 남겨야 한다.

## 요구사항

- 올바른 노드에 접속한다.
- `relative-fawn` 네임스페이스의 WordPress Deployment를 확인한다.
- WordPress Pod의 resource requests를 조정한다.
- 3개의 Pod가 공평하게 CPU와 Memory를 사용할 수 있도록 설정한다.
- 노드 안정성을 위해 약간의 여유 리소스를 남긴다.
- 일반 containers와 init containers가 있다면 같은 requests 값을 사용한다.
- resource limits는 변경하지 않아도 된다.
- 최종적으로 WordPress Deployment가 3 replicas를 유지해야 한다.
- 모든 WordPress Pod가 `Running` 및 `Ready` 상태여야 한다.

## 핵심 개념

Kubernetes에서 `resources.requests`는 Pod가 스케줄링될 때 필요한 최소 리소스를 의미한다.

현재 WordPress Pod 하나가 CPU 1개와 Memory 1000Mi를 요청하고 있으면, 전체 리소스가 CPU 1, Memory 1024Mi인 환경에서 replica 3개가 모두 뜰 수 없다.

따라서 Pod 3개가 모두 실행될 수 있도록 각 Pod의 requests 값을 줄여야 한다.

```text
전체 CPU 1 = 1000m
1000m / 3 = 약 333m

전체 Memory 1024Mi
1024Mi / 3 = 약 341Mi
```

하지만 문제에서 노드 안정성을 위해 여유 리소스를 남기라고 했으므로, 각 Pod에 다음과 같이 설정한다.

```yaml
requests:
  cpu: 300m
  memory: 300Mi
```

이렇게 설정하면 전체 요청량은 다음과 같다.

```text
CPU: 300m × 3 = 900m
Memory: 300Mi × 3 = 900Mi
```

즉, CPU는 약 100m, Memory는 약 124Mi 정도 여유가 남는다.

## 리소스 흐름

```text
전체 노드 리소스
CPU 1000m / Memory 1024Mi
        ↓
WordPress Pod 3개에 분배
        ↓
Pod 1개당 requests: cpu 300m, memory 300Mi
        ↓
3개 Pod 전체 requests: cpu 900m, memory 900Mi
        ↓
노드 안정성을 위한 여유 리소스 확보
```

## 1. 올바른 노드 접속

문제에서 지정한 노드로 접속한다.

```bash
ssh cka0001
```

잘못된 노드에서 작업하면 점수를 받을 수 없으므로 먼저 접속 위치를 확인한다.

## 2. 현재 Deployment 확인

`relative-fawn` 네임스페이스의 Deployment를 확인한다.

```bash
kubectl get deploy -n relative-fawn
```

출력 예시:

```bash
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
mariadb     1/1     1            1           19m
wordpress   3/3     2            3           19m
```

Pod 상태도 확인한다.

```bash
kubectl get pods -n relative-fawn
```

## 3. WordPress Deployment 수정

WordPress Deployment를 수정한다.

```bash
kubectl edit deployment wordpress -n relative-fawn
```

기존 설정은 다음과 같이 Pod 하나가 너무 많은 리소스를 요청하고 있다.

```yaml
resources:
  limits:
    cpu: "1"
    memory: 1000Mi
  requests:
    cpu: "1"
    memory: 1000Mi
```

이를 다음과 같이 수정한다.

```yaml
resources:
  limits:
    cpu: "1"
    memory: 1000Mi
  requests:
    cpu: 300m
    memory: 300Mi
```

`limits`는 문제에서 변경하지 않아도 된다고 했으므로 그대로 둔다.

수정해야 하는 위치는 다음과 같다.

```yaml
spec:
  template:
    spec:
      containers:
      - image: 1pro/exam
        name: wordpress
        resources:
          limits:
            cpu: "1"
            memory: 1000Mi
          requests:
            cpu: 300m
            memory: 300Mi
```

## 4. initContainers 확인

문제에서 containers와 init containers에 같은 requests 값을 사용하라고 했으므로, `initContainers`가 있는지 확인한다.

```bash
kubectl get deployment wordpress -n relative-fawn -o yaml | grep -A20 "initContainers"
```

만약 `initContainers`가 있다면, init container의 requests도 동일하게 설정한다.

```yaml
initContainers:
- name: example-init
  resources:
    requests:
      cpu: 300m
      memory: 300Mi
```

이번 실습에서는 WordPress YAML에 `initContainers`가 보이지 않았으므로 별도로 추가하지 않았다.

## 5. 롤아웃 확인

수정 후 Deployment 롤아웃 상태를 확인한다.

```bash
kubectl rollout status deployment/wordpress -n relative-fawn
```

정상 상태라면 다음과 같이 출력된다.

```bash
deployment "wordpress" successfully rolled out
```

Deployment 상태도 확인한다.

```bash
kubectl get deploy -n relative-fawn
```

정상 목표:

```bash
NAME        READY   UP-TO-DATE   AVAILABLE
mariadb     1/1     1            1
wordpress   3/3     3            3
```

Pod 상태도 확인한다.

```bash
kubectl get pods -n relative-fawn
```

정상 목표:

```bash
NAME                         READY   STATUS    RESTARTS   AGE
mariadb-...                  1/1     Running   0          ...
wordpress-...                1/1     Running   0          ...
wordpress-...                1/1     Running   0          ...
wordpress-...                1/1     Running   0          ...
```

## 6. 실제 적용된 requests 확인

WordPress Deployment에 requests 값이 정상 반영되었는지 확인한다.

```bash
kubectl get deployment wordpress -n relative-fawn \
  -o jsonpath='{.spec.template.spec.containers[*].resources.requests}{"\n"}'
```

정상 출력 예시:

```bash
{"cpu":"300m","memory":"300Mi"}
```

Pod에도 적용되었는지 확인하려면 WordPress Pod 이름을 확인한 뒤 다음 명령을 실행한다.

```bash
kubectl get pod <wordpress-pod-name> -n relative-fawn \
  -o jsonpath='{.spec.containers[*].resources.requests}{"\n"}'
```

정상 출력 예시:

```bash
{"cpu":"300m","memory":"300Mi"}
```

## 트러블슈팅: rollout이 끝나지 않고 Pending Pod가 생긴 경우

수정 후 다음과 같이 rollout이 실패할 수 있다.

```bash
[cka0001@k8s-master ~]$ kubectl rollout status deployment/wordpress -n relative-fawn
Waiting for deployment "wordpress" rollout to finish: 2 out of 3 new replicas have been updated...
error: deployment "wordpress" exceeded its progress deadline
```

Pod 상태를 보면 기존 ReplicaSet의 Pod와 새 ReplicaSet의 Pod가 섞여 있고, 새 Pod 하나가 Pending 상태일 수 있다.

```bash
[cka0001@k8s-master ~]$ kubectl get pods -n relative-fawn
NAME                         READY   STATUS    RESTARTS   AGE
mariadb-77b7cfc86c-xrchg     1/1     Running   0          20m
wordpress-6f4fc6c46f-84v9p   1/1     Running   0          20m
wordpress-6f4fc6c46f-j2ckf   1/1     Running   0          20m
wordpress-86c948c57b-gdbbb   1/1     Running   0          11m
wordpress-86c948c57b-hpbkr   0/1     Pending   0          11m
```

이 경우 원인은 RollingUpdate 방식 때문이다.

Deployment는 기본적으로 기존 Pod를 한 번에 모두 지우지 않고, 새 Pod를 만들면서 순차적으로 교체한다.

하지만 리소스가 부족한 환경에서는 기존 Pod가 남아 있는 상태에서 새 Pod를 추가로 만들려고 하면서 Pending이 발생할 수 있다.

문제에서도 resource requests를 수정하는 동안 WordPress Deployment를 임시로 0 replicas로 줄이는 것이 도움이 될 수 있다고 안내하고 있다.

## 7. WordPress를 임시로 0개로 줄이기

WordPress Deployment를 잠시 0 replicas로 줄인다.

```bash
kubectl scale deployment wordpress -n relative-fawn --replicas=0
```

Pod가 삭제되는지 확인한다.

```bash
kubectl get pods -n relative-fawn
```

WordPress Pod가 모두 사라질 때까지 기다린다.

`mariadb`는 문제의 직접 수정 대상이 아니므로 건드리지 않는다.

## 8. 다시 3 replicas로 확장

WordPress Deployment를 다시 3 replicas로 확장한다.

```bash
kubectl scale deployment wordpress -n relative-fawn --replicas=3
```

롤아웃 상태를 다시 확인한다.

```bash
kubectl rollout status deployment/wordpress -n relative-fawn
```

Deployment 상태 확인:

```bash
kubectl get deploy -n relative-fawn
```

Pod 상태 확인:

```bash
kubectl get pods -n relative-fawn
```

최종 목표는 다음과 같다.

```bash
NAME        READY   UP-TO-DATE   AVAILABLE
mariadb     1/1     1            1
wordpress   3/3     3            3
```

```bash
NAME                         READY   STATUS    RESTARTS   AGE
mariadb-...                  1/1     Running   0          ...
wordpress-...                1/1     Running   0          ...
wordpress-...                1/1     Running   0          ...
wordpress-...                1/1     Running   0          ...
```

## 9. 그래도 Pending이면 확인할 것

만약 다시 3 replicas로 올렸는데도 Pod가 Pending이면 Pending Pod의 이벤트를 확인한다.

```bash
kubectl describe pod <pending-wordpress-pod-name> -n relative-fawn
```

아래쪽 `Events`에서 다음과 같은 메시지를 확인한다.

```text
Insufficient cpu
Insufficient memory
```

만약 `Insufficient cpu` 또는 `Insufficient memory`가 계속 발생하면 requests 값이 아직 큰 것이다.

이 경우 현재 노드 상황에 맞게 requests 값을 조금 더 낮춰야 한다.

예시:

```yaml
requests:
  cpu: 200m
  memory: 200Mi
```

하지만 문제의 조건상 3개의 Pod에 공평하게 리소스를 나누고 약간의 여유를 남기는 목적이라면, 일반적으로는 다음 값이 적절하다.

```yaml
requests:
  cpu: 300m
  memory: 300Mi
```

## 실제 출력 결과

<img width="1261" height="351" alt="image" src="https://github.com/user-attachments/assets/85bed729-0722-407c-8ee5-a8f7b513224c" />

