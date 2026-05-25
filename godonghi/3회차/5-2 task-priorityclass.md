# 5-2 task-priorityclass 핵심 정리

## 1. 문제 핵심

`priority` 네임스페이스의 `busybox-logger` Deployment에 높은 우선순위를 부여한다.

해야 할 일:

```text
1. PriorityClass high-priority 생성
2. busybox-logger Deployment에 priorityClassName: high-priority 적용
3. busybox-logger rollout 성공 확인
4. 다른 Deployment는 수정하지 않음
```

---

## 2. 구축 명령어

```bash
kubectl create ns priority
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/priorityclass/deployment-others.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/priorityclass/deployment-busybox.yaml
```

---

## 3. PriorityClass 생성

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000000
globalDefault: false
description: "High priority class for user workloads"
```

적용:

```bash
kubectl apply -f high-priority.yaml
```

확인:

```bash
kubectl get priorityclass high-priority
```

---

## 4. Deployment 수정

```bash
kubectl edit deployment busybox-logger -n priority
```

`spec.template.spec` 아래에 추가:

```yaml
priorityClassName: high-priority
```

위치:

```yaml
spec:
  template:
    spec:
      priorityClassName: high-priority
      containers:
      - name: busybox
```

---

## 5. 확인 결과

롤아웃 확인:

```bash
kubectl rollout status deployment busybox-logger -n priority
```

정상 결과:

```text
deployment "busybox-logger" successfully rolled out
```

Pod 상태 확인:

```bash
kubectl get pod -n priority
```

결과:

```text
NAME                              READY   STATUS    RESTARTS   AGE
busybox-logger-66b4b75f4f-47569   1/1     Running   0          2m21s
busybox-logger-66b4b75f4f-hnv4r   1/1     Running   0          108s
busybox-logger-66b4b75f4f-xhzw8   1/1     Running   0          74s
others-748b8ff466-dpj8q           0/1     Pending   0          108s
others-748b8ff466-gks4f           0/1     Pending   0          74s
others-748b8ff466-j668r           0/1     Pending   0          2m21s
```

`busybox-logger`는 Running이고, `others`는 Pending 상태가 된다.

Deployment 적용 확인:

```bash
kubectl get deploy busybox-logger -n priority -o yaml | grep -A3 -B3 priorityClassName
```

결과:

```text
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      priorityClassName: high-priority
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
```

---

## 6. 정리

최종 상태:

```text
PriorityClass high-priority 존재
busybox-logger Deployment에 priorityClassName 적용
busybox-logger Pod 3개 Running
others Pod는 Pending
다른 Deployment는 수정하지 않음
```

---

## 7. 삭제 명령어

```bash
kubectl delete priorityclass high-priority
kubectl delete ns priority
```
