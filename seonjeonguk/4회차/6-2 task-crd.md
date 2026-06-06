# CKA CRD 문제 정리 - cert-manager CRD 목록 및 Certificate 필드 문서 추출

## 문제 요약

이 문제는 cert-manager를 새로 설치하는 문제가 아니다.

이미 클러스터에 배포되어 있는 cert-manager application을 확인하고, cert-manager 관련 CRD 목록과 `Certificate` Custom Resource의 `spec.subject` 필드 문서를 파일로 저장하는 문제이다.

최종적으로 만들어야 하는 파일은 2개이다.

```bash
~/resources.yaml
~/subject.yaml
```

---

## 문제 요구사항

문제에서 요구하는 작업은 다음과 같다.

```text
1. 올바른 host에 접속한다.
2. cert-manager application이 클러스터에 배포되어 있는지 확인한다.
3. 모든 cert-manager CRD 목록을 kubectl 기본 출력 형식으로 조회해서 ~/resources.yaml에 저장한다.
4. Certificate Custom Resource의 spec.subject 필드 문서를 kubectl로 추출해서 ~/subject.yaml에 저장한다.
```

---

## 핵심 주의사항

문제에서 가장 중요한 조건은 다음이다.

```text
Make sure kubectl's default output format and use kubectl to list CRDs.
Do not set an output format.
```

뜻:

`~/resources.yaml` 파일을 만들 때는 `kubectl get crd`의 기본 출력 형식을 사용해야 한다.

따라서 아래 명령어는 사용하면 안 된다.

```bash
kubectl get crd -o yaml
kubectl get crd -o json
kubectl get crd -o name
```

파일 이름이 `resources.yaml`이지만, YAML 형식으로 저장하라는 뜻이 아니다.

문제에서 `Do not set an output format`이라고 했으므로, `kubectl get crd` 기본 table 출력 그대로 저장해야 한다.

---

## 최종 요구사항 한 줄 정리

```text
cert-manager가 배포되어 있는지 확인한 뒤,
cert-manager 관련 CRD 목록을 기본 출력 형식으로 ~/resources.yaml에 저장하고,
Certificate 리소스의 spec.subject 필드 설명을 kubectl explain으로 ~/subject.yaml에 저장한다.
```

---

## 시험장에서 문제를 보는 방식

문제에서 뽑아야 할 키워드는 다음과 같다.

```text
Verify cert-manager application
Create a list of all cert-manager CRDs
Save it to ~/resources.yaml
Use kubectl default output format
Do not set an output format
Extract documentation for subject specification field
Certificate Custom Resource
Save it to ~/subject.yaml
```

이걸 명령어로 바꾸면 다음과 같다.

```text
cert-manager 확인
→ kubectl get pods -n cert-manager
→ kubectl get deploy -n cert-manager

cert-manager CRD 목록 저장
→ kubectl get crd | awk 'NR==1 || /cert-manager.io/' > ~/resources.yaml

Certificate.spec.subject 문서 저장
→ kubectl explain certificate.spec.subject > ~/subject.yaml
```

---

## 1. 지정된 host 접속

문제 상단에 다음과 같은 문구가 있으면 먼저 해당 host로 접속한다.

```bash
ssh cka0001
```

접속 후 클러스터 접근이 되는지 확인한다.

```bash
kubectl get nodes
```

이 단계는 내가 문제를 풀어야 하는 올바른 클러스터에 접속했는지 확인하는 과정이다.

---

## 2. cert-manager application 확인

문제에 다음과 같이 되어 있다.

```text
Verify the cert-manager application which has been deployed in the cluster.
```

뜻:

cert-manager를 설치하라는 것이 아니라, 이미 배포되어 있는 cert-manager를 확인하라는 뜻이다.

먼저 namespace를 확인한다.

```bash
kubectl get ns
```

`cert-manager` namespace가 있으면 Pod를 확인한다.

```bash
kubectl get pods -n cert-manager
```

정상이라면 보통 다음과 같은 Pod들이 보인다.

```text
cert-manager-...
cert-manager-cainjector-...
cert-manager-webhook-...
```

Deployment 기준으로도 확인한다.

```bash
kubectl get deploy -n cert-manager
```

정상 예시:

```text
NAME                      READY   UP-TO-DATE   AVAILABLE
cert-manager              1/1     1            1
cert-manager-cainjector   1/1     1            1
cert-manager-webhook      1/1     1            1
```

이 단계는 문제의 `Verify` 조건을 만족하기 위한 확인 과정이다.

---

## 3. cert-manager CRD 목록 확인

CRD 전체 목록은 다음 명령어로 확인한다.

```bash
kubectl get crd
```

cert-manager 관련 CRD만 확인하려면 다음처럼 필터링한다.

```bash
kubectl get crd | grep cert-manager.io
```

cert-manager 관련 CRD는 보통 다음과 같다.

```text
certificaterequests.cert-manager.io
certificates.cert-manager.io
challenges.acme.cert-manager.io
clusterissuers.cert-manager.io
issuers.cert-manager.io
orders.acme.cert-manager.io
```

여기서 주의할 점은 다음 두 CRD도 cert-manager 관련이라는 것이다.

```text
challenges.acme.cert-manager.io
orders.acme.cert-manager.io
```

따라서 `grep cert-manager.io`로 필터링하면 `cert-manager.io` 계열과 `acme.cert-manager.io` 계열을 모두 잡을 수 있다.

---

## 4. cert-manager CRD 목록을 ~/resources.yaml에 저장

문제 조건상 `kubectl get crd` 기본 출력 형식을 사용해야 한다.

가장 단순한 명령어는 다음과 같다.

```bash
kubectl get crd | grep cert-manager.io > ~/resources.yaml
```

다만 이 방식은 header가 빠진다.

시험장에서 조금 더 깔끔하게 하려면 header까지 남기는 아래 명령어를 추천한다.

```bash
kubectl get crd | awk 'NR==1 || /cert-manager.io/' > ~/resources.yaml
```

이 명령어의 의미:

```text
kubectl get crd
→ CRD를 기본 출력 형식으로 조회한다.

awk 'NR==1 || /cert-manager.io/'
→ 첫 번째 줄 header는 남긴다.
→ cert-manager.io가 들어간 CRD만 남긴다.

> ~/resources.yaml
→ 결과를 파일로 저장한다.
```

확인:

```bash
cat ~/resources.yaml
```

정상 예시:

```text
NAME                                  CREATED AT
certificaterequests.cert-manager.io   ...
certificates.cert-manager.io          ...
challenges.acme.cert-manager.io       ...
clusterissuers.cert-manager.io        ...
issuers.cert-manager.io               ...
orders.acme.cert-manager.io           ...
```

---

## 5. Certificate 리소스의 spec.subject 필드 문서 확인

문제 문장은 다음과 같다.

```text
Using kubectl, extract the documentation for the subject specification field
of the Certificate Custom Resource and save it to ~/subject.yaml.
```

여기서 말하는 `subject specification field`는 다음 필드를 의미한다.

```text
Certificate.spec.subject
```

Kubernetes에서 리소스 필드 설명을 볼 때는 `kubectl explain`을 사용한다.

먼저 Certificate 리소스가 있는지 확인할 수 있다.

```bash
kubectl api-resources | grep -i certificate
```

출력 예시:

```text
certificates   cert-manager.io/v1   true   Certificate
```

이제 `spec.subject` 필드 설명을 확인한다.

```bash
kubectl explain certificate.spec.subject
```

정상 출력 예시:

```text
KIND:       Certificate
VERSION:    cert-manager.io/v1

FIELD: subject <Object>

DESCRIPTION:
...
```

---

## 6. spec.subject 필드 문서를 ~/subject.yaml에 저장

정답 명령어는 다음과 같다.

```bash
kubectl explain certificate.spec.subject > ~/subject.yaml
```

확인:

```bash
cat ~/subject.yaml
```

정상적으로 저장되었다면 다음과 같은 내용이 보인다.

```text
KIND:       Certificate
VERSION:    cert-manager.io/v1

FIELD: subject <Object>

DESCRIPTION:
...
```

---

## 7. kubectl explain 명령어가 안 될 때

환경에 따라 singular 이름인 `certificate`가 안 먹을 수 있다.

그럴 때는 먼저 리소스 이름을 확인한다.

```bash
kubectl api-resources | grep -i certificate
```

출력에서 리소스 이름이 `certificates`로 보이면 다음을 시도한다.

```bash
kubectl explain certificates.spec.subject
```

그래도 안 되면 API group까지 붙여서 시도한다.

```bash
kubectl explain certificates.cert-manager.io.spec.subject
```

성공하는 명령어를 찾으면 그 명령어를 파일로 저장한다.

예시:

```bash
kubectl explain certificates.cert-manager.io.spec.subject > ~/subject.yaml
```

---

## 최종 풀이 명령어 흐름

시험장에서 바로 입력할 수 있는 전체 흐름은 다음과 같다.

```bash
ssh cka0001
```

```bash
kubectl get nodes
```

cert-manager 확인:

```bash
kubectl get ns
kubectl get pods -n cert-manager
kubectl get deploy -n cert-manager
```

cert-manager CRD 목록 저장:

```bash
kubectl get crd | awk 'NR==1 || /cert-manager.io/' > ~/resources.yaml
```

Certificate `spec.subject` 필드 문서 저장:

```bash
kubectl explain certificate.spec.subject > ~/subject.yaml
```

최종 확인:

```bash
cat ~/resources.yaml
cat ~/subject.yaml
ls -l ~/resources.yaml ~/subject.yaml
```

---

## 최종 확인 기준

### 1. resources.yaml 확인

```bash
cat ~/resources.yaml
```

정상적으로 cert-manager 관련 CRD 목록이 저장되어 있어야 한다.

확인할 CRD 예시:

```text
certificaterequests.cert-manager.io
certificates.cert-manager.io
challenges.acme.cert-manager.io
clusterissuers.cert-manager.io
issuers.cert-manager.io
orders.acme.cert-manager.io
```

중요:

`resources.yaml`은 YAML 형식일 필요가 없다.

문제에서 `Do not set an output format`이라고 했으므로, `kubectl get crd` 기본 table 출력이면 된다.

---

### 2. subject.yaml 확인

```bash
cat ~/subject.yaml
```

정상적으로 Certificate의 `spec.subject` 필드 설명이 저장되어야 한다.

확인할 키워드:

```text
KIND: Certificate
FIELD: subject
DESCRIPTION
```

---

## 자주 하는 실수

### 실수 1. cert-manager를 새로 설치하려고 함

이 문제는 cert-manager 설치 문제가 아니다.

문제에 `Verify the cert-manager application`이라고 되어 있으므로, 이미 배포된 cert-manager를 확인하는 문제이다.

따라서 Helm이나 manifest로 cert-manager를 새로 설치할 필요가 없다.

---

### 실수 2. resources.yaml을 YAML 형식으로 저장함

잘못된 예:

```bash
kubectl get crd -o yaml | grep cert-manager.io > ~/resources.yaml
```

문제에서 분명히 `Do not set an output format`이라고 했다.

따라서 `-o yaml`, `-o json`, `-o name` 등을 붙이면 안 된다.

올바른 예:

```bash
kubectl get crd | awk 'NR==1 || /cert-manager.io/' > ~/resources.yaml
```

---

### 실수 3. kubectl describe를 사용함

필드 문서를 추출하는 문제이므로 `kubectl describe`가 아니라 `kubectl explain`이 맞다.

잘못된 접근:

```bash
kubectl describe certificate
```

올바른 접근:

```bash
kubectl explain certificate.spec.subject
```

---

### 실수 4. subject 필드 위치를 잘못 이해함

문제의 `subject specification field`는 다음을 의미한다.

```text
Certificate.spec.subject
```

따라서 다음 명령어를 사용한다.

```bash
kubectl explain certificate.spec.subject
```

---

### 실수 5. acme.cert-manager.io CRD를 놓침

cert-manager CRD에는 다음과 같은 ACME 관련 CRD도 포함된다.

```text
challenges.acme.cert-manager.io
orders.acme.cert-manager.io
```

따라서 `grep cert-manager.io`로 필터링하면 이 CRD들까지 포함할 수 있다.

---

## 시험장에서의 판단 흐름

```text
1. 문제에서 Verify라고 했는지 확인한다.
   → 설치 문제가 아니라 확인 문제이다.

2. CRD 목록 저장 파일을 확인한다.
   → ~/resources.yaml

3. 출력 형식 조건을 확인한다.
   → kubectl 기본 출력 형식
   → -o 옵션 사용 금지

4. cert-manager 관련 CRD만 필터링한다.
   → kubectl get crd | awk 'NR==1 || /cert-manager.io/'

5. Certificate의 subject 필드 문서를 추출한다.
   → kubectl explain certificate.spec.subject

6. 결과를 파일로 저장한다.
   → > ~/resources.yaml
   → > ~/subject.yaml
```

---

## 한 줄 핵심 암기

```text
CRD 목록은 kubectl get crd 기본 출력으로 저장하고,
CRD 필드 문서는 kubectl explain으로 저장한다.
resources.yaml에는 -o 옵션 금지,
subject.yaml에는 kubectl explain certificate.spec.subject 사용.
```

---

## 최종 암기 명령어

```bash
kubectl get crd | awk 'NR==1 || /cert-manager.io/' > ~/resources.yaml
kubectl explain certificate.spec.subject > ~/subject.yaml
```
