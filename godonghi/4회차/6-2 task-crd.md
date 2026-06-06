# 6-2 task-crd 핵심 정리

## 1. 문제 핵심

cert-manager가 클러스터에 배포되어 있는지 확인하고, cert-manager 관련 CRD 목록과 `Certificate.spec` 문서를 파일로 저장하는 문제다.

생성해야 할 파일:

```text
~/resources.yaml
~/subject.yaml
```

---

## 2. 구축 확인

이 문제는 cert-manager가 이미 설치되어 있는 상태를 사용한다.

```bash
kubectl get ns cert-manager
kubectl get pods -n cert-manager
kubectl get crd | grep cert-manager
```

확인 결과:

```text
NAME           STATUS   AGE
cert-manager   Active   41d

NAME                                       READY   STATUS    RESTARTS      AGE
cert-manager-5776fd696b-kkbfb              1/1     Running   5 (22m ago)   15d
cert-manager-cainjector-5dc4bf4cdf-6r58b   1/1     Running   5 (22m ago)   15d
cert-manager-webhook-6ff7dcb868-c8zt4      1/1     Running   5 (22m ago)   15d

certificaterequests.cert-manager.io                     2026-04-26T05:10:25Z
certificates.cert-manager.io                            2026-04-26T05:10:25Z
challenges.acme.cert-manager.io                         2026-04-26T05:10:25Z
clusterissuers.cert-manager.io                          2026-04-26T05:10:25Z
issuers.cert-manager.io                                 2026-04-26T05:10:25Z
orders.acme.cert-manager.io                             2026-04-26T05:10:25Z
```

---

## 3. 필요한 개념

CRD는 `CustomResourceDefinition`이다.

```text
Kubernetes에 새로운 리소스 종류를 추가하는 설계도
```

cert-manager가 설치되면 `Certificate`, `Issuer`, `ClusterIssuer` 같은 Custom Resource를 사용할 수 있도록 CRD가 등록된다.

`kubectl get crd`는 클러스터에 등록된 CRD 목록을 보여준다.

`kubectl explain`은 리소스 필드 설명서를 보여준다.

---

## 4. 풀이 명령어

cert-manager CRD 목록을 `~/resources.yaml`에 저장한다.

문제에서 기본 출력 형식을 사용하라고 했으므로 `-o yaml` 같은 출력 옵션을 쓰지 않는다.

```bash
kubectl get crd | grep cert-manager > ~/resources.yaml
```

`Certificate` Custom Resource의 `spec` 필드 문서를 `~/subject.yaml`에 저장한다.

```bash
kubectl explain certificate.spec > ~/subject.yaml
```

---

## 5. 확인 명령어

```bash
ls -l ~/resources.yaml ~/subject.yaml
cat ~/resources.yaml
head -30 ~/subject.yaml
```

확인 결과:

```text
-rw-r--r--. 1 root root  462 Jun  7 00:53 /root/resources.yaml
-rw-r--r--. 1 root root 8559 Jun  7 00:54 /root/subject.yaml
```

`resources.yaml` 내용:

```text
certificaterequests.cert-manager.io                     2026-04-26T05:10:25Z
certificates.cert-manager.io                            2026-04-26T05:10:25Z
challenges.acme.cert-manager.io                         2026-04-26T05:10:25Z
clusterissuers.cert-manager.io                          2026-04-26T05:10:25Z
issuers.cert-manager.io                                 2026-04-26T05:10:25Z
orders.acme.cert-manager.io                             2026-04-26T05:10:25Z
```

`subject.yaml` 내용 앞부분:

```text
GROUP:      cert-manager.io
KIND:       Certificate
VERSION:    v1

FIELD: spec <Object>


DESCRIPTION:
    Specification of the desired state of the Certificate resource.
    https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
    
FIELDS:
  additionalOutputFormats	<[]Object>
    Defines extra output formats of the private key and signed certificate chain
    to be written to this Certificate's target Secret.

  commonName	<string>
    Requested common name X509 certificate subject attribute.
    More info: https://datatracker.ietf.org/doc/html/rfc5280#section-4.1.2.6

  dnsNames	<[]string>
    Requested DNS subject alternative names.
```

---

## 6. 핵심 주의사항

`~/resources.yaml`은 파일 이름이 `.yaml`이지만, 문제에서 기본 출력 형식을 쓰라고 했으므로 표 형태 출력이 저장되는 것이 맞다.

하면 안 되는 명령어:

```bash
kubectl get crd -o yaml > ~/resources.yaml
kubectl get crd -o json > ~/resources.yaml
kubectl get crd -o wide > ~/resources.yaml
```

정답 형태:

```bash
kubectl get crd | grep cert-manager > ~/resources.yaml
```

---

## 7. 최종 성공 기준

```text
cert-manager namespace 존재
cert-manager Pod 3개 Running
~/resources.yaml 파일 존재
~/subject.yaml 파일 존재
resources.yaml에 cert-manager 관련 CRD 목록 저장됨
subject.yaml에 Certificate spec 문서 저장됨
resources.yaml 생성 시 -o 옵션 사용하지 않음
```

---
