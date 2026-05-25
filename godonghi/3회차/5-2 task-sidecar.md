# 5-2 task-sidecar 핵심 정리

## 1. 문제 핵심

기존 Deployment `synergy-deployment`의 메인 컨테이너는 로그를 stdout이 아니라 파일에 쓴다.

```bash
/var/log/synergy-deployment.log
```

목표는 sidecar 컨테이너를 추가해서 이 파일 로그를 `kubectl logs`로 볼 수 있게 만드는 것이다.

구조:

```text
synergy-app 컨테이너 → /var/log/synergy-deployment.log에 로그 기록
sidecar 컨테이너 → 같은 로그 파일을 tail -f 해서 stdout으로 출력
kubectl logs -c sidecar → 로그 확인
```

---

## 2. 구축 명령어

```bash
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/sidecar/deployment.yaml
```

---

## 3. 수정할 내용

Deployment 수정:

```bash
kubectl edit deployment synergy-deployment
```

`spec.template.spec` 아래를 수정한다.

최종 핵심 형태:

```yaml
spec:
  template:
    spec:
      containers:
      - command:
        - sh
        - -c
        - while true; do echo "logging" >> /var/log/synergy-deployment.log; sleep
          2; done
        image: busybox:stable
        imagePullPolicy: IfNotPresent
        name: synergy-app
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/log
          name: log-volume

      - command:
        - /bin/sh
        - -c
        - tail -n+1 -f /var/log/synergy-deployment.log
        image: busybox:stable
        imagePullPolicy: IfNotPresent
        name: sidecar
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/log
          name: log-volume

      volumes:
      - emptyDir: {}
        name: log-volume
```

---

## 4. 왜 이렇게 하는가?

`kubectl logs`는 컨테이너의 stdout/stderr 로그를 본다.

하지만 기존 `synergy-app`은 로그를 파일에 쓴다.

```bash
echo "logging" >> /var/log/synergy-deployment.log
```

그래서 sidecar가 같은 파일을 읽어서 stdout으로 출력하게 한다.

```bash
tail -n+1 -f /var/log/synergy-deployment.log
```

두 컨테이너가 같은 로그 파일을 봐야 하므로 `emptyDir` 볼륨을 만들고 둘 다 `/var/log`에 마운트한다.

```text
synergy-app /var/log
        ↓
    log-volume
        ↑
sidecar /var/log
```

---

## 5. 확인 명령어

롤아웃 확인:

```bash
kubectl rollout status deployment synergy-deployment
```

Pod 확인:

```bash
kubectl get pod -l app=synergy
```

정상 예시:

```text
READY 2/2
```

컨테이너 이름 확인:

```bash
kubectl get deploy synergy-deployment -o jsonpath='{.spec.template.spec.containers[*].name}{"\n"}'
```

정상 출력:

```text
synergy-app sidecar
```

sidecar 로그 확인:

```bash
POD=$(kubectl get pod -l app=synergy -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD -c sidecar
```

정상 출력 예시:

```text
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
logging
```

계속 따라보기:

```bash
kubectl logs -f $POD -c sidecar
```

---

## 6. 정리 명령어

```bash
kubectl delete deploy synergy-deployment
```
