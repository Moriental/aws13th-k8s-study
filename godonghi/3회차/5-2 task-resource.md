# 5-2 task-resource 핵심 정리

## 1. 문제 핵심

`relative-fawn` 네임스페이스의 WordPress Deployment가 replicas 3개를 유지하면서 모든 Pod가 Running/Ready 상태가 되도록 리소스 requests를 조정하는 문제다.

기존 WordPress Pod는 requests가 너무 커서 3개 중 1개가 Pending 상태였다.

---

## 2. 구축 명령어

```bash
kubectl create ns relative-fawn
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/resource/deployment-mariadb.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/resource/deployment-wordpress.yaml
```

---

## 3. 문제 원인

노드 allocatable 확인:

```bash
kubectl describe node k8s-master | grep -A5 "Allocatable"
```

결과:

```text
Allocatable:
  cpu:                4
  memory:             3883424Ki
```

기존 WordPress requests:

```yaml
requests:
  cpu: "1"
  memory: 1000Mi
```

WordPress replicas가 3개이므로 WordPress만 계산해도:

```text
CPU: 1 x 3 = 3 CPU
Memory: 1000Mi x 3 = 3000Mi
```

여기에 MariaDB, kube-system, CNI, Ingress, Gateway, metrics-server 등 시스템 Pod와 노드 안정성을 위한 여유분이 필요해서 WordPress 3개가 모두 스케줄링되지 못했다.

---

## 4. 수정 내용

Deployment 수정:

```bash
kubectl edit deployment wordpress -n relative-fawn
```

`limits`는 그대로 두고 `requests`만 수정한다.

수정 전:

```yaml
resources:
  limits:
    cpu: "1"
    memory: 1000Mi
  requests:
    cpu: "1"
    memory: 1000Mi
```

수정 후:

```yaml
resources:
  limits:
    cpu: "1"
    memory: 1000Mi
  requests:
    cpu: 500m
    memory: 500Mi
```

수정 후 WordPress 3개 총 requests:

```text
CPU: 500m x 3 = 1500m = 1.5 CPU
Memory: 500Mi x 3 = 1500Mi
```

---

## 5. 롤링 업데이트가 멈춘 경우

수정 후 rollout 중 기존 Pod와 새 Pod가 동시에 떠서 자원이 부족할 수 있다.

이때 WordPress를 임시로 0개로 줄였다가 다시 3개로 올린다.

```bash
kubectl scale deployment wordpress -n relative-fawn --replicas=0
kubectl get deploy wordpress -n relative-fawn
```

정상 예시:

```text
wordpress   0/0
```

다시 3개로 올리기:

```bash
kubectl scale deployment wordpress -n relative-fawn --replicas=3
```

---

## 6. 확인 명령어

```bash
kubectl rollout status deployment wordpress -n relative-fawn
kubectl get deploy,pod -n relative-fawn
```

정상 결과:

```text
deployment "wordpress" successfully rolled out
```

```text
NAME                        READY   UP-TO-DATE   AVAILABLE
deployment.apps/mariadb     1/1     1            1
deployment.apps/wordpress   3/3     3            3

NAME                            READY   STATUS    RESTARTS
pod/mariadb-...                 1/1     Running   0
pod/wordpress-...               1/1     Running   0
pod/wordpress-...               1/1     Running   0
pod/wordpress-...               1/1     Running   0
```

requests 적용 확인:

```bash
kubectl describe deployment wordpress -n relative-fawn
```

정상 값:

```text
Requests:
  cpu:     500m
  memory:  500Mi
```

---

## 7. 핵심 정리

```text
원인:
WordPress requests가 너무 커서 replicas 3개가 모두 뜨지 못함

수정:
requests를 cpu 500m, memory 500Mi로 낮춤

주의:
limits는 변경하지 않음
replicas는 최종적으로 3 유지

롤아웃 꼬임 해결:
scale 0 → scale 3

최종 상태:
mariadb 1/1 Running
wordpress 3/3 Running
```

---

## 8. 삭제 명령어

```bash
kubectl delete ns relative-fawn
```
