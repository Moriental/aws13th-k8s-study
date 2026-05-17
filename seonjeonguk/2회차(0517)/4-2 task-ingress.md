# 4-3. Ingress 생성 및 경로 기반 Service 노출

## 문제 요약

`echo-sound` 네임스페이스에 새로운 Ingress 리소스 `echo`를 생성한다.

Ingress를 통해  
`http://example.org/echo` 요청이  
Service `echoserver-service`의 `8080` 포트로 전달되도록 구성한다.

최종적으로 아래 명령어 실행 시 HTTP 상태코드 `200`이 출력되어야 한다.

```bash
curl -o /dev/null -s -w "%{http_code}\n" http://example.org/echo
```

---

## 요구사항

- Namespace: `echo-sound`
- Ingress 이름: `echo`
- Host: `example.org`
- Path: `/echo`
- Service 이름: `echoserver-service`
- Service Port: `8080`
- 최종 확인 결과: HTTP `200`

---

## 핵심 개념

### 1. Ingress의 Host와 Path

문제의 URL:

```text
http://example.org/echo
```

은 Ingress에서 다음처럼 나뉜다.

```text
Host = example.org
Path = /echo
```

---

### 2. Ingress는 Service로 요청을 전달한다

```text
Client
  ↓
http://example.org/echo
  ↓
Ingress echo
  ↓
Service echoserver-service:8080
  ↓
Pod
```

---

### 3. `ingressClassName`

Ingress Controller가 어떤 Ingress를 처리할지 결정한다.

먼저 현재 IngressClass를 확인한다.

```bash
kubectl get ingressclass
```

출력:

```bash
NAME    CONTROLLER
nginx   k8s.io/ingress-nginx
```

따라서 Ingress YAML에는 다음처럼 작성한다.

```yaml
ingressClassName: nginx
```

---

## 1. 문제 리소스 생성

문제에서 제공한 Deployment와 Service를 먼저 생성한다.

```bash
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/ingress/deployment.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/ingress/service.yaml
```

### 참고

`kubectl create -f URL`은  
내 디렉토리에 YAML 파일을 저장하는 것이 아니라,  
클러스터에 리소스만 생성한다.

---

## 2. 기존 Service 확인

Ingress가 연결할 Service 이름과 포트를 확인한다.

```bash
kubectl get svc echoserver-service -n echo-sound -o yaml
```

확인할 값:

```yaml
metadata:
  name: echoserver-service

spec:
  ports:
  - port: 8080
```

---

## 3. Ingress 매니페스트 작성

`ingress.yml` 파일을 작성한다.

```bash
vi ingress.yml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo
  namespace: echo-sound
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: example.org
    http:
      paths:
      - path: /echo
        pathType: Prefix
        backend:
          service:
            name: echoserver-service
            port:
              number: 8080
```

---

## 4. Ingress 적용

```bash
kubectl apply -f ingress.yml
```

---

## 5. Ingress 생성 확인

```bash
kubectl get ingress -n echo-sound
```

또는 자세히 확인한다.

```bash
kubectl describe ingress echo -n echo-sound
```

확인할 내용:

```text
Ingress Class: nginx
Host:          example.org
Path:          /echo
Backend:       echoserver-service:8080
```

# 트러블슈팅

## 1. IngressClass 이름 불일치

처음 작성한 YAML에서 다음처럼 작성했다.

```yaml
ingressClassName: nginx-example
```

하지만 실제 클러스터의 IngressClass는 다음과 같았다.

```bash
kubectl get ingressclass
```

```bash
NAME    CONTROLLER
nginx   k8s.io/ingress-nginx
```

따라서 다음처럼 수정했다.

```yaml
ingressClassName: nginx
```

---

## 2. `curl` 실행 결과가 `000`으로 출력됨

Ingress YAML을 수정한 뒤에도 다음 명령어 결과가 `000`으로 출력되었다.

```bash
curl -4 -o /dev/null -s -w "%{http_code}\n" http://example.org/echo
```

추가 확인 결과, Ingress Controller Pod가 정상 상태가 아니었다.

```bash
kubectl get pods -n ingress-nginx
```

```bash
ingress-nginx-controller-...   0/1   CrashLoopBackOff
```

---

## 3. Ingress Controller 로그 확인

```bash
kubectl logs -n ingress-nginx <ingress-nginx-controller-pod>
```

로그에서 다음 오류를 확인했다.

```text
Error loading shared library libpcre.so.1: Exec format error
NGINX master process died (127)
```

---

## 4. 동일 이미지로 임시 Pod 생성 후 직접 확인

Controller 이미지 자체 문제인지 확인하기 위해 같은 이미지를 사용하는 임시 Pod를 생성했다.

```bash
kubectl run ingress-image-check -n ingress-nginx \
  --image='registry.k8s.io/ingress-nginx/controller:v1.13.4@sha256:4042ae3c512c5d7bcf9682b0fdff96cd7b46a23dcbe15a762349094cd8087be7' \
  --restart=Never \
  --command -- /bin/sh -c 'sleep 3600'
```

컨테이너 내부에서 NGINX 실행을 확인했다.

```bash
kubectl exec -n ingress-nginx ingress-image-check -- /usr/bin/nginx -v
```

동일한 오류가 발생했다.

```text
Error loading shared library libpcre.so.1: Exec format error
```

---

## 5. 라이브러리 파일 손상 확인

```bash
kubectl exec -n ingress-nginx ingress-image-check -- sh -c '
find / -name "libpcre.so.1*" 2>/dev/null
ls -l /usr/lib/libpcre.so.1* 2>/dev/null || true
'
```

출력:

```bash
/usr/lib/libpcre.so.1
/usr/lib/libpcre.so.1.2.13

lrwxrwxrwx 1 root root 17 /usr/lib/libpcre.so.1 -> libpcre.so.1.2.13
-rwxr-xr-x 1 root root 0  /usr/lib/libpcre.so.1.2.13
```

`libpcre.so.1.2.13` 파일이 `0 byte`로 확인되었고,  
이로 인해 Ingress Controller 내부 NGINX가 정상 실행되지 못하고 있었다.

---

## 6. ingress-nginx Controller 복구

기존 실습 설정값은 유지하면서  
Ingress Controller 차트를 `4.15.1` 버전으로 업그레이드하였다.

### 복구용 values 파일 작성

```bash
cat > ~/ingress-nginx-recover-values.yaml <<'EOF'
controller:
  service:
    type: ClusterIP
    externalIPs:
      - 192.168.56.40
  ingressClassResource:
    name: nginx
    default: true
  extraEnvs:
    - name: TZ
      value: Asia/Seoul
  replicaCount: 1
  admissionWebhooks:
    enabled: false
EOF
```

### Helm 업그레이드

```bash
helm upgrade ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx \
  --version 4.15.1 \
  --reset-values \
  -f ~/ingress-nginx-recover-values.yaml
```

vagrant up 할때 helm 쪽에서 오류 발생해서 그것만 따로 shell 안에서 설치해서 했는데 버전이 뭐가 꼬엿나 봅니다. 이거 해결하는데 1시간 30분 걸렸습니다 하...
### 정상화 확인

```bash
kubectl get pods -n ingress-nginx
```

Ingress Controller가 정상 상태로 돌아온 뒤,  
최종 테스트 명령어에서 `200`을 확인하였다.

```bash
curl -4 -o /dev/null -s -w "%{http_code}\n" http://example.org/echo
```

```bash
200
```

결과 출력
<img width="1222" height="48" alt="image" src="https://github.com/user-attachments/assets/8a5be0e0-15a1-41a2-a3c7-1da601ac9116" />
