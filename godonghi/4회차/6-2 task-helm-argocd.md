# 6-2 task-helm-argocd 핵심 정리

## 1. 문제 핵심

Argo CD를 Helm으로 설치하는 문제다.

해야 할 일:

```text
1. Argo CD 공식 Helm repo를 argo 이름으로 추가
2. Argo CD CRD는 이미 설치되어 있으므로 Helm chart에서 CRD 설치 비활성화
3. chart version 8.6.4로 helm template 생성
4. template 결과를 ~/argo-helm.yaml에 저장
5. 같은 버전과 같은 설정으로 Helm install
6. release 이름은 argocd
7. namespace는 argocd
```

---

## 2. 문제 구축 명령어

Argo CD CRD를 미리 설치한다.

```bash
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/helm/argocd-3.1.8/application-crd.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/helm/argocd-3.1.8/applicationset-crd.yaml
kubectl create -f https://raw.githubusercontent.com/kubetm/exam-c/main/helm/argocd-3.1.8/appproject-crd.yaml
```

확인:

```bash
kubectl get crd | grep argoproj
```

확인 결과:

```text
applications.argoproj.io                                2026-06-06T16:10:54Z
applicationsets.argoproj.io                             2026-06-06T16:10:55Z
appprojects.argoproj.io                                 2026-06-06T16:10:55Z
```

---

## 3. 풀이 명령어

Namespace 생성:

```bash
kubectl create ns argocd
```

Helm repo 추가:

```bash
helm repo add argo https://argoproj.github.io/argo-helm/
```

Helm repo 업데이트:

```bash
helm repo update
```

Helm template 생성 및 저장:

```bash
helm template argocd argo/argo-cd \
  --version 8.6.4 \
  --namespace argocd \
  --set crds.install=false \
  > ~/argo-helm.yaml
```

Helm install:

```bash
helm install argocd argo/argo-cd \
  --version 8.6.4 \
  --namespace argocd \
  --set crds.install=false
```

---

## 4. 확인 명령어

```bash
ls -l ~/argo-helm.yaml
helm list -n argocd
kubectl get pods -n argocd
kubectl get crd | grep argoproj
```

확인 결과:

```text
-rw-r--r--. 1 root root 107457 Jun  7 01:12 /root/argo-helm.yaml
```

```text
NAME    NAMESPACE   REVISION   UPDATED                                  STATUS    CHART          APP VERSION
argocd  argocd      1          2026-06-07 01:12:59.426953972 +0900 KST   deployed  argo-cd-8.6.4  v3.1.8
```

```text
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          2m44s
argocd-applicationset-controller-7dbb95fc56-8t9bs   1/1     Running   0          2m44s
argocd-dex-server-69d8df8478-srd8z                  1/1     Running   0          2m44s
argocd-notifications-controller-6fffcd649-fkvjc     1/1     Running   0          2m44s
argocd-redis-658b7bf8c9-dtqk8                       1/1     Running   0          2m44s
argocd-repo-server-76cf7c8749-7z4tn                 1/1     Running   0          2m44s
argocd-server-5f57dc6487-wxsfq                      1/1     Running   0          2m44s
```

```text
applications.argoproj.io                                2026-06-06T16:10:54Z
applicationsets.argoproj.io                             2026-06-06T16:10:55Z
appprojects.argoproj.io                                 2026-06-06T16:10:55Z
```

---

## 5. 핵심 개념

Helm 용어:

```text
Repository = Helm chart 저장소
Chart = 설치 패키지
Release = chart를 설치해서 생성된 인스턴스
```

이 문제에서 사용한 값:

```text
repo name: argo
repo URL: https://argoproj.github.io/argo-helm/
chart: argo/argo-cd
release name: argocd
namespace: argocd
chart version: 8.6.4
```

CRD는 이미 설치되어 있으므로 Helm chart가 CRD를 다시 설치하지 않도록 한다.

```bash
--set crds.install=false
```

---

## 6. UI 안내 메시지

`helm install` 후 Argo CD UI 접속 안내가 출력되지만, 문제에서 UI 접근 설정은 필요 없다고 했다.

```text
You do not need to configure access to the Argo CD server UI.
```

따라서 port-forward나 admin password 확인은 하지 않아도 된다.

---

## 7. 최종 성공 기준

```text
~/argo-helm.yaml 파일 존재
helm release argocd 상태 deployed
chart version argo-cd-8.6.4
argocd namespace의 Pod 전부 Running
argoproj CRD 3개 존재
crds.install=false 사용
```

---
kubectl delete ns argocd
rm -f ~/argo-helm.yaml
```
