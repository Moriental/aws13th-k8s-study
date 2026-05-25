# Sidecar Container를 이용한 Kubernetes Logging 구성

## 문제 요약

기존 `synergy-deployment` Deployment에 sidecar 컨테이너를 추가하여 레거시 애플리케이션 로그를 Kubernetes 기본 로깅 방식인 `kubectl logs`로 확인할 수 있게 만드는 문제이다.

기존 애플리케이션 컨테이너는 `/var/log/synergy-deployment.log` 파일에 로그를 기록하고 있다.

새로 추가할 sidecar 컨테이너는 같은 로그 파일을 `tail -f`로 읽어 표준 출력으로 내보내야 한다.

이를 위해 기존 컨테이너와 sidecar 컨테이너가 `/var/log` 경로를 공유하도록 shared volume을 설정해야 한다.

## 요구사항

- 올바른 노드에 접속한다.
- 기존 Deployment `synergy-deployment`를 수정한다.
- 기존 Pod에 sidecar 컨테이너를 추가한다.
- sidecar 컨테이너 이름은 `sidecar`로 설정한다.
- sidecar 컨테이너 이미지는 `busybox:stable`을 사용한다.
- sidecar 컨테이너는 다음 명령을 실행해야 한다.

```bash
/bin/sh -c "tail -n+1 -f /var/log/synergy-deployment.log"
```

- `/var/log`에 마운트되는 shared volume을 사용한다.
- 기존 메인 컨테이너와 sidecar 컨테이너가 같은 `/var/log` 볼륨을 공유해야 한다.
- 기존 컨테이너의 설정은 필요한 volumeMount 추가 외에는 수정하지 않는다.
- 최종적으로 Pod가 `2/2 Running` 상태가 되어야 한다.
- `kubectl logs -c sidecar`로 로그를 확인할 수 있어야 한다.

## 핵심 개념

Sidecar 컨테이너는 같은 Pod 안에서 메인 컨테이너를 보조하는 컨테이너이다.

이 문제에서는 메인 컨테이너가 로그 파일에 로그를 쓰고, sidecar 컨테이너가 그 로그 파일을 읽어서 표준 출력으로 내보낸다.

Kubernetes의 `kubectl logs`는 컨테이너의 표준 출력을 확인하는 명령이므로, sidecar가 파일 로그를 표준 출력으로 흘려보내면 `kubectl logs`로 로그 확인이 가능해진다.

## 구조

```text
synergy-deployment Pod
├── synergy-app 컨테이너
│   └── /var/log/synergy-deployment.log 에 로그 기록
│
└── sidecar 컨테이너
    └── /var/log/synergy-deployment.log 를 tail -f로 읽음
```

두 컨테이너가 같은 로그 파일을 보기 위해 shared volume을 사용한다.

```text
emptyDir volume
        ↓
/var/log 로 메인 컨테이너에 mount
/var/log 로 sidecar 컨테이너에 mount
```

## 1. 올바른 노드 접속

문제에서 지정한 노드로 접속한다.

```bash
ssh cka0001
```

잘못된 노드에서 작업하면 점수를 받을 수 없으므로 먼저 접속 위치를 확인한다.

## 2. Deployment 확인

먼저 `synergy-deployment`가 어느 네임스페이스에 있는지 확인한다.

```bash
kubectl get deploy -A | grep synergy-deployment
```

이번 실습에서는 `default` 네임스페이스에 존재했다.

```bash
kubectl get deploy synergy-deployment -n default
```

Deployment의 YAML 구조를 확인한다.

```bash
kubectl get deployment synergy-deployment -n default -o yaml
```

기존 구조는 다음과 같다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: synergy-deployment
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: synergy
  template:
    metadata:
      labels:
        app: synergy
    spec:
      containers:
      - command:
        - sh
        - -c
        - while true; do echo "logging" >> /var/log/synergy-deployment.log; sleep 2; done
        image: busybox:stable
        imagePullPolicy: IfNotPresent
        name: synergy-app
        resources: {}
```

기존 컨테이너 `synergy-app`는 이미 다음 파일에 로그를 쓰고 있다.

```text
/var/log/synergy-deployment.log
```

## 3. Deployment 수정

Deployment를 수정한다.

```bash
kubectl edit deployment synergy-deployment -n default
```

수정해야 할 내용은 3가지이다.

```text
1. 기존 synergy-app 컨테이너에 volumeMounts 추가
2. sidecar 컨테이너 추가
3. Pod spec에 emptyDir volume 추가
```

## 4. 기존 컨테이너에 volumeMount 추가

기존 `synergy-app` 컨테이너에 `/var/log` 경로로 volumeMount를 추가한다.

```yaml
volumeMounts:
- name: log-volume
  mountPath: /var/log
```

기존 컨테이너의 image, command, name 등은 수정하지 않는다.

수정 후 기존 컨테이너 부분은 다음과 같은 형태가 된다.

```yaml
containers:
- command:
  - sh
  - -c
  - while true; do echo "logging" >> /var/log/synergy-deployment.log; sleep 2; done
  image: busybox:stable
  imagePullPolicy: IfNotPresent
  name: synergy-app
  resources: {}
  terminationMessagePath: /dev/termination-log
  terminationMessagePolicy: File
  volumeMounts:
  - name: log-volume
    mountPath: /var/log
```

## 5. sidecar 컨테이너 추가

기존 컨테이너 아래에 같은 레벨로 sidecar 컨테이너를 추가한다.

```yaml
- name: sidecar
  image: busybox:stable
  command:
  - /bin/sh
  - -c
  - tail -n+1 -f /var/log/synergy-deployment.log
  volumeMounts:
  - name: log-volume
    mountPath: /var/log
```

중요한 점은 sidecar가 기존 컨테이너 안에 들어가는 것이 아니라, `containers` 목록에 새로운 항목으로 추가되어야 한다는 것이다.

잘못된 예시:

```yaml
containers:
- name: synergy-app
  image: busybox:stable
  sidecar:
    image: busybox:stable
```

올바른 예시:

```yaml
containers:
- name: synergy-app
  image: busybox:stable

- name: sidecar
  image: busybox:stable
```

## 6. emptyDir volume 추가

`volumes`는 `containers`와 같은 레벨에 작성한다.

```yaml
volumes:
- name: log-volume
  emptyDir: {}
```

`volumes`를 컨테이너 안쪽에 넣으면 안 된다.

올바른 위치:

```yaml
spec:
  template:
    spec:
      containers:
      - name: synergy-app
      - name: sidecar
      volumes:
      - name: log-volume
        emptyDir: {}
```

## 7. 최종 YAML 구조

최종적으로 `spec.template.spec` 아래는 다음과 같은 구조가 되어야 한다.

```yaml
template:
  metadata:
    labels:
      app: synergy
  spec:
    containers:
    - command:
      - sh
      - -c
      - while true; do echo "logging" >> /var/log/synergy-deployment.log; sleep 2; done
      image: busybox:stable
      imagePullPolicy: IfNotPresent
      name: synergy-app
      resources: {}
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
      volumeMounts:
      - name: log-volume
        mountPath: /var/log

    - name: sidecar
      image: busybox:stable
      command:
      - /bin/sh
      - -c
      - tail -n+1 -f /var/log/synergy-deployment.log
      volumeMounts:
      - name: log-volume
        mountPath: /var/log

    volumes:
    - name: log-volume
      emptyDir: {}

    dnsPolicy: ClusterFirst
    restartPolicy: Always
    schedulerName: default-scheduler
```

## 8. 롤아웃 확인

수정 후 Deployment가 정상적으로 롤아웃되는지 확인한다.

```bash
kubectl rollout status deployment/synergy-deployment -n default
```

정상 출력 예시:

```bash
deployment "synergy-deployment" successfully rolled out
```

## 9. Pod 상태 확인

Pod 상태를 확인한다.

```bash
kubectl get pods -n default
```

sidecar 컨테이너가 추가되었으므로 READY 값이 `2/2`가 되어야 한다.

정상 예시:

```bash
NAME                                  READY   STATUS    RESTARTS   AGE
synergy-deployment-xxxxxxxxxx-xxxxx   2/2     Running   0          1m
```

`2/2`의 의미는 다음과 같다.

```text
synergy-app 컨테이너 1개
sidecar 컨테이너 1개
총 2개의 컨테이너가 모두 Ready
```

## 10. 컨테이너 목록 확인

Pod 안에 컨테이너가 정상적으로 2개 들어갔는지 확인한다.

```bash
kubectl get pod <pod-name> -n default \
  -o jsonpath='{.spec.containers[*].name}{"\n"}'
```

정상 출력:

```bash
synergy-app sidecar
```

## 11. sidecar 로그 확인

sidecar 컨테이너의 로그를 확인한다.

```bash
kubectl logs <pod-name> -c sidecar -n default
```

또는 Deployment 기준으로 확인할 수도 있다.

```bash
kubectl logs deployment/synergy-deployment -c sidecar -n default
```

정상이라면 다음과 같이 `logging`이 출력된다.

```bash
logging
logging
logging
```

계속 로그를 보고 싶으면 `-f` 옵션을 사용할 수 있다.

```bash
kubectl logs deployment/synergy-deployment -c sidecar -n default -f
```

## 트러블슈팅

### 1. Pod READY가 1/2 또는 0/2인 경우

Pod 상태를 먼저 확인한다.

```bash
kubectl get pods -n default
```

문제가 있는 Pod를 describe 한다.

```bash
kubectl describe pod <pod-name> -n default
```

확인할 부분은 아래 `Events`이다.

```text
Events:
  ...
```

### 2. sidecar가 CrashLoopBackOff 되는 경우

sidecar 명령어가 잘못되었을 가능성이 있다.

문제에서 요구한 명령어는 다음과 같다.

```bash
/bin/sh -c "tail -n+1 -f /var/log/synergy-deployment.log"
```

YAML에서는 다음처럼 작성한다.

```yaml
command:
- /bin/sh
- -c
- tail -n+1 -f /var/log/synergy-deployment.log
```

### 3. sidecar 로그가 안 나오는 경우

먼저 sidecar 컨테이너 로그를 확인한다.

```bash
kubectl logs <pod-name> -c sidecar -n default
```

로그가 비어 있거나 파일을 찾을 수 없다는 메시지가 나오면 shared volume 설정을 확인한다.

기존 컨테이너와 sidecar 컨테이너에 둘 다 같은 volumeMount가 있어야 한다.

```yaml
volumeMounts:
- name: log-volume
  mountPath: /var/log
```

그리고 Pod spec 아래에 같은 이름의 volume이 있어야 한다.

```yaml
volumes:
- name: log-volume
  emptyDir: {}
```

### 4. volumes 위치 오류

`volumes`는 `containers` 안쪽이 아니라 `containers`와 같은 레벨이다.

잘못된 예시:

```yaml
containers:
- name: synergy-app
  image: busybox:stable
  volumes:
  - name: log-volume
    emptyDir: {}
```

올바른 예시:

```yaml
spec:
  containers:
  - name: synergy-app
  - name: sidecar
  volumes:
  - name: log-volume
    emptyDir: {}
```

### 5. 기존 컨테이너 설정을 건드린 경우

문제에서 기존 컨테이너의 specification은 필요한 추가 설정 외에는 수정하지 말라고 했다.

따라서 기존 컨테이너에서 수정해도 되는 것은 사실상 `/var/log`를 공유하기 위한 `volumeMounts` 추가뿐이다.

건드리면 안 되는 대표 항목:

```text
image
command
args
name
resources
ports
```

## 실제 출력 결과

Deployment 확인:

```bash
kubectl get deploy synergy-deployment -n default
```

Deployment 수정:

```bash
kubectl edit deployment synergy-deployment -n default
```

롤아웃 확인:

```bash
kubectl rollout status deployment/synergy-deployment -n default
```

Pod 확인:

```bash
kubectl get pods -n default
```

정상 목표:

```bash
NAME                                  READY   STATUS    RESTARTS   AGE
synergy-deployment-xxxxxxxxxx-xxxxx   2/2     Running   0          1m
```

컨테이너 이름 확인:

```bash
kubectl get pod <pod-name> -n default \
  -o jsonpath='{.spec.containers[*].name}{"\n"}'
```

실제 출력:
<img width="1500" height="709" alt="스크린샷 2026-05-25 184909" src="https://github.com/user-attachments/assets/ee635db8-f87e-48d1-a845-6a20e9743981" />



## 정리

이 문제의 핵심은 기존 애플리케이션 컨테이너가 파일로 남기는 로그를 sidecar 컨테이너가 읽어서 표준 출력으로 내보내도록 만드는 것이다.

이를 위해 두 컨테이너가 같은 `/var/log` 경로를 공유해야 한다.

```text
기존 컨테이너가 /var/log/synergy-deployment.log에 로그 기록
        ↓
emptyDir shared volume으로 /var/log 공유
        ↓
sidecar 컨테이너가 같은 로그 파일을 tail -f
        ↓
kubectl logs -c sidecar로 로그 확인 가능
컨테이너 목록에 synergy-app, sidecar 존재
kubectl logs -c sidecar로 logging 출력 확인
```
