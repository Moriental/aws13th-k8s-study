# CKA Helm 문제 정리 - Argo CD 설치

## 문제 요약

Argo CD를 Helm으로 설치하는 문제이다.

핵심은 다음 3가지이다.

1. Argo CD 공식 Helm repository를 `argo`라는 이름으로 추가한다.
2. Argo CD Helm chart version `8.6.4`를 template으로 생성해서 `~/argo-helm.yaml`에 저장한다.
3. 같은 조건으로 Argo CD를 Helm install 한다.

중요 조건은 다음과 같다.

| 항목             | 값                                       |
| -------------- | --------------------------------------- |
| Helm repo name | `argo`                                  |
| Repo URL       | `https://argoproj.github.io/argo-helm/` |
| Chart          | `argo/argo-cd`                          |
| Chart version  | `8.6.4`                                 |
| Release name   | `argocd`                                |
| Namespace      | `argocd`                                |
| Template 저장 파일 | `~/argo-helm.yaml`                      |
| CRD 설치 여부      | 설치하지 않음                                 |
| UI 접속 설정       | 필요 없음                                   |

---

## 문제 핵심 문장

```text
The Argo CD CRDs have already been pre-installed in the cluster.
Configure the chart to not install CRDs.
```

뜻:

이미 Argo CD CRD가 클러스터에 설치되어 있으므로, Helm chart가 CRD를 다시 설치하면 안 된다.

따라서 반드시 아래 옵션을 사용해야 한다.

```bash
--set crds.install=false
```

---

## 시험장에서 접근하는 순서

## 1. 지정된 host 접속

문제 상단에 다음과 같은 문구가 있으면 먼저 접속한다.

```bash
ssh cka0001
```

확인:

```bash
kubectl get nodes
```

---

## 2. Helm repo 추가

문제에서 repo 이름과 주소를 이미 제공한다.

```text
Add the official Argo CD Helm repository with the name argo.
Repository Path: https://argoproj.github.io/argo-helm/
```

Helm repo 추가 기본 형식은 다음과 같다.

```bash
helm repo add <repo-name> <repo-url>
```

문제 값을 넣으면 다음과 같다.

```bash
helm repo add argo https://argoproj.github.io/argo-helm/
helm repo update
```

확인:

```bash
helm repo list
```

---

## 3. chart 이름 확인

기억이 안 나면 repo에서 검색한다.

```bash
helm search repo argo
```

Argo CD chart 이름은 다음과 같다.

```bash
argo/argo-cd
```

구조는 다음과 같다.

```text
<repo-name>/<chart-name>
```

이번 문제에서는 다음과 같다.

```text
argo/argo-cd
```

---

## 4. namespace 생성

문제에서 `argocd` namespace에 설치하라고 했으므로 namespace가 필요하다.

안전한 명령어:

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
```

확인:

```bash
kubectl get ns argocd
```

---

## 5. Helm template 생성

문제 요구사항은 다음과 같다.

```text
Generate a helm template of the Argo CD Helm chart version 8.6.4
for the argocd namespace and save it to ~/argo-helm.yaml.
Configure the chart to not install CRDs.
```

Helm template 기본 형식:

```bash
helm template <release-name> <repo/chart> --version <version> -n <namespace> --set <key>=<value> > <file>
```

이번 문제 정답:

```bash
helm template argocd argo/argo-cd \
  --version 8.6.4 \
  -n argocd \
  --set crds.install=false \
  > ~/argo-helm.yaml
```

한 줄 버전:

```bash
helm template argocd argo/argo-cd --version 8.6.4 -n argocd --set crds.install=false > ~/argo-helm.yaml
```

확인:

```bash
ls -l ~/argo-helm.yaml
head ~/argo-helm.yaml
```

CRD가 포함되지 않았는지 확인:

```bash
grep -n "kind: CustomResourceDefinition" ~/argo-helm.yaml
```

정상이라면 아무 결과가 나오지 않는 것이 좋다.

---

## 6. Helm install 수행

주의할 점:

`helm install`에는 `> ~/argo-helm.yaml`를 붙이면 안 된다.

잘못된 예:

```bash
helm install argocd argo/argo-cd --version 8.6.4 -n argocd --set crds.install=false > ~/argo-helm.yaml
```

올바른 예:

```bash
helm install argocd argo/argo-cd \
  --version 8.6.4 \
  -n argocd \
  --set crds.install=false
```

한 줄 버전:

```bash
helm install argocd argo/argo-cd --version 8.6.4 -n argocd --set crds.install=false
```

---

## template과 install 차이

## helm template

```bash
helm template ...
```

`helm template`은 실제 설치가 아니다.

설치될 YAML을 화면에 출력하는 명령어이다.

문제에서 파일 저장을 요구하면 `>`를 사용해서 파일로 저장한다.

이번 문제에서는 다음 명령어로 template 파일을 만든다.

```bash
helm template argocd argo/argo-cd --version 8.6.4 -n argocd --set crds.install=false > ~/argo-helm.yaml
```

---

## helm install

```bash
helm install ...
```

`helm install`은 실제 클러스터에 chart를 설치하는 명령어이다.

Deployment, Service, ConfigMap, Secret 등이 생성된다.

`helm install`에는 파일 저장 리다이렉션 `>`를 붙이지 않는다.

```bash
helm install argocd argo/argo-cd --version 8.6.4 -n argocd --set crds.install=false
```

---

## 자주 하는 실수 1 - 버전 오타

잘못된 명령어:

```bash
--version 8.64
```

정답:

```bash
--version 8.6.4
```

에러 예시:

```text
Error: chart "argo-cd" matching 8.64 not found in argo index.
```

원인:

`8.64`라는 chart version은 없고, 문제에서 요구한 버전은 `8.6.4`이다.

---

## 자주 하는 실수 2 - install에 리다이렉션 붙이기

잘못된 명령어:

```bash
helm install argocd argo/argo-cd --version 8.6.4 -n argocd --set crds.install=false > ~/argo-helm.yaml
```

문제점:

`~/argo-helm.yaml` 파일 저장은 `helm template`에서 하는 것이다.

`helm install`은 실제 설치 명령이다.

따라서 install에는 `>`를 붙이지 않는다.

올바른 순서:

```bash
helm template argocd argo/argo-cd --version 8.6.4 -n argocd --set crds.install=false > ~/argo-helm.yaml
helm install argocd argo/argo-cd --version 8.6.4 -n argocd --set crds.install=false
```

---

## 자주 하는 실수 3 - 설치 중 Ctrl+C

설치 중 `Ctrl+C`를 누르면 다음과 같은 에러가 날 수 있다.

```text
Error: INSTALLATION FAILED: context canceled
```

이후 다시 install을 하면 다음 에러가 날 수 있다.

```text
Error: INSTALLATION FAILED: cannot re-use a name that is still in use
```

뜻:

설치가 중간에 끊겼지만 Helm release 이름 `argocd`가 이미 등록되어 있는 상태이다.

해결:

```bash
helm list -n argocd -a
helm uninstall argocd -n argocd
helm list -n argocd -a
```

그다음 다시 설치한다.

```bash
helm install argocd argo/argo-cd --version 8.6.4 -n argocd --set crds.install=false
```

---

## 최종 정답 명령어 모음

시험장에서 바로 입력할 수 있는 최종 흐름은 다음과 같다.

```bash
ssh cka0001
```

```bash
helm repo add argo https://argoproj.github.io/argo-helm/
helm repo update
```

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
```

```bash
helm template argocd argo/argo-cd --version 8.6.4 -n argocd --set crds.install=false > ~/argo-helm.yaml
```

```bash
helm install argocd argo/argo-cd --version 8.6.4 -n argocd --set crds.install=false
```

확인:

```bash
helm list -n argocd
kubectl get pods -n argocd
ls -l ~/argo-helm.yaml
head ~/argo-helm.yaml
grep -n "kind: CustomResourceDefinition" ~/argo-helm.yaml
```

---

## 최종 확인 기준

<img width="2854" height="1169" alt="image" src="https://github.com/user-attachments/assets/f6fcd09b-ef59-406f-8cc5-cbaa331ccbdd" />


## 한 줄 핵심 암기

```text
template은 파일 저장, install은 실제 설치.
CRD가 이미 있으면 --set crds.install=false.
install에는 > ~/argo-helm.yaml 붙이지 말 것.
```
