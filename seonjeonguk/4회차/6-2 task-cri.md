# CKA CRI 문제 정리 - Docker 환경에서 cri-dockerd와 sysctl 설정

## 문제 요약

이 문제는 Kubernetes 리소스를 생성하는 문제가 아니라, Linux 시스템을 Kubernetes에서 사용할 수 있도록 준비하는 문제이다.

문제에서 Docker는 이미 설치되어 있다고 되어 있다.
하지만 kubeadm이 Docker를 container runtime으로 사용할 수 있도록 `cri-dockerd`를 설치하고, 필요한 커널 파라미터를 설정해야 한다.

---

## 문제 요구사항

해야 할 작업은 크게 2가지이다.

### 1. cri-dockerd 설정

```text
1. ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb 패키지를 설치한다.
2. Debian package는 dpkg로 설치한다.
3. cri-docker service를 enable 한다.
4. cri-docker service를 start 한다.
```

### 2. system parameter 설정

다음 커널 파라미터를 설정해야 한다.

```text
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
net.netfilter.nf_conntrack_max = 131072
```

---

## 최종 요구사항 한 줄 정리

```text
Docker는 이미 설치되어 있으므로 cri-dockerd만 dpkg로 설치하고,
cri-docker.service를 enable/start 한 뒤,
kubeadm이 요구하는 sysctl 값을 설정하고 적용한다.
```

---

## 공식문서에서 봐야 하는 부분

문제에서 참고하라고 한 공식문서:

```text
https://kubernetes.io/docs/setup/production-environment/container-runtimes/
```

이 문서에서 봐야 하는 부분은 크게 4곳이다.

---

## 1. Container Runtimes 첫 부분

공식문서 첫 부분에서는 Kubernetes 노드가 Pod를 실행하려면 container runtime이 필요하다고 설명한다.

또한 Kubernetes는 CRI(Container Runtime Interface)를 통해 container runtime과 통신한다.

이 내용을 문제와 연결하면 다음과 같다.

```text
문제: Docker is already installed, but you need to configure it for kubeadm.

의미:
Docker는 이미 설치되어 있지만,
kubeadm/Kubernetes가 Docker를 container runtime으로 사용하려면
CRI와 연결되는 구성이 필요하다.
```

즉, 이 문제는 Docker를 새로 설치하는 문제가 아니다.

Docker와 Kubernetes 사이를 연결해주는 `cri-dockerd`를 설치하는 문제이다.

---

## 2. Docker Engine 섹션

공식문서의 `Docker Engine` 섹션을 보면, Docker Engine은 Kubernetes의 CRI를 직접 구현하지 않으므로 `cri-dockerd` adapter가 필요하다고 설명한다.

이 내용을 문제와 연결하면 다음과 같다.

```text
문제: Docker is already installed
공식문서: Docker Engine을 Kubernetes와 함께 사용하려면 cri-dockerd adapter가 필요함

따라서:
Docker 설치 X
cri-dockerd 설치 O
```

문제에서 설치할 패키지 파일을 직접 제공한다.

```text
~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
```

그리고 Debian package는 `dpkg`로 설치하라고 되어 있다.

따라서 설치 명령어는 다음과 같다.

```bash
sudo dpkg -i ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
```

---

## 3. Install and configure prerequisites 섹션

공식문서에는 Kubernetes 노드를 구성하기 전에 필요한 Linux 설정이 나온다.

특히 Kubernetes 네트워킹이 정상 동작하려면 일부 kernel parameter 설정이 필요할 수 있다.

이 내용을 문제와 연결하면 다음과 같다.

```text
문제: Configure these system parameters
공식문서: Kubernetes networking을 위해 sysctl parameter 설정이 필요함

따라서:
문제에서 준 4개 값을 sysctl 설정 파일에 작성하고 적용해야 한다.
```

---

## 4. Enable IPv4 packet forwarding 섹션

공식문서에서 가장 직접적으로 참고할 부분이다.

공식문서는 sysctl 값을 파일로 작성하고, `sysctl --system`으로 적용하는 흐름을 보여준다.

공식문서의 기본 흐름:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
sysctl net.ipv4.ip_forward
```

이 문제에서는 `net.ipv4.ip_forward` 하나만 설정하는 것이 아니라, 문제에서 제시한 4개 값을 모두 설정해야 한다.

따라서 공식문서의 형식을 가져오고, 값은 문제에서 준 값을 넣는다.

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
net.netfilter.nf_conntrack_max = 131072
EOF
```

적용:

```bash
sudo sysctl --system
```

확인:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv6.conf.all.forwarding
sysctl net.ipv4.ip_forward
sysctl net.netfilter.nf_conntrack_max
```

---

## 시험장에서 문제를 보는 방식

문제를 보면 먼저 키워드를 뽑는다.

```text
Prepare a Linux system for Kubernetes
Docker is already installed
configure it for kubeadm
Set up cri-dockerd
Install Debian package using dpkg
Enable and start cri-docker service
Configure system parameters
```

이걸 작업으로 바꾸면 다음과 같다.

```text
1. Kubernetes 리소스 문제가 아니라 OS 설정 문제이다.
2. Docker는 이미 설치되어 있으므로 Docker 설치는 하지 않는다.
3. kubeadm이 Docker를 쓰도록 cri-dockerd를 설치한다.
4. 문제에서 준 .deb 파일을 dpkg로 설치한다.
5. cri-docker.service를 enable/start 한다.
6. 문제에서 준 sysctl 값을 /etc/sysctl.d/*.conf에 작성한다.
7. sysctl --system으로 적용한다.
8. service 상태와 sysctl 값을 확인한다.
```

---

## 실제 풀이 순서

## 1. 지정된 host 접속

문제 상단에 다음과 같이 되어 있으면 먼저 접속한다.

```bash
ssh cka0001
```

접속:

```bash
ssh cka0001
```

---

## 2. 패키지 파일 확인

문제에서 설치하라고 한 파일이 실제로 있는지 확인한다.

```bash
ls -l ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
```

정상이라면 파일이 보여야 한다.

---

## 3. cri-dockerd 설치

문제에서 Debian package는 `dpkg`로 설치하라고 했다.

```bash
sudo dpkg -i ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
```

설치 후 systemd unit을 확인할 수 있다.

```bash
systemctl list-unit-files | grep cri
```

예상되는 service:

```text
cri-docker.service
cri-docker.socket
```

---

## 4. cri-docker service enable/start

문제에서 요구한 것은 다음이다.

```text
Enable and start the cri-docker service.
```

따라서 다음 명령어를 사용한다.

```bash
sudo systemctl enable --now cri-docker.service
```

이 명령어는 아래 두 명령어를 합친 것이다.

```bash
sudo systemctl enable cri-docker.service
sudo systemctl start cri-docker.service
```

확인:

```bash
systemctl is-enabled cri-docker.service
systemctl is-active cri-docker.service
```

정상 출력:

```text
enabled
active
```

상세 확인:

```bash
systemctl status cri-docker.service --no-pager
```

---

## 5. br_netfilter 모듈 로드

`net.bridge.bridge-nf-call-iptables` 설정을 위해 `br_netfilter` 모듈이 필요할 수 있다.

현재 세션에서 로드:

```bash
sudo modprobe br_netfilter
```

재부팅 후에도 로드되도록 설정:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
```

확인:

```bash
lsmod | grep br_netfilter
```

출력이 있으면 로드된 것이다.

---

## 6. sysctl 설정 파일 작성

문제에서 요구한 4개 system parameter를 설정 파일에 작성한다.

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
net.netfilter.nf_conntrack_max = 131072
EOF
```

파일 이름은 꼭 `99-kubernetes-cri.conf`일 필요는 없다.
중요한 것은 `/etc/sysctl.d/` 아래에 설정을 남기는 것이다.

---

## 7. sysctl 설정 적용

작성한 sysctl 설정을 적용한다.

```bash
sudo sysctl --system
```

특정 파일만 적용하고 싶다면 다음도 가능하다.

```bash
sudo sysctl -p /etc/sysctl.d/99-kubernetes-cri.conf
```

시험에서는 `sudo sysctl --system`이 편하다.

---

## 8. 설정값 확인

문제에서 요구한 값이 실제로 적용되었는지 확인한다.

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv6.conf.all.forwarding
sysctl net.ipv4.ip_forward
sysctl net.netfilter.nf_conntrack_max
```

정상 출력:

```text
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
net.netfilter.nf_conntrack_max = 131072
```

한 번에 확인할 수도 있다.

```bash
sysctl net.bridge.bridge-nf-call-iptables net.ipv6.conf.all.forwarding net.ipv4.ip_forward net.netfilter.nf_conntrack_max
```

---

## 최종 정답 명령어 모음

시험장에서 바로 입력할 수 있는 흐름은 다음과 같다.

```bash
ssh cka0001
```

```bash
ls -l ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
```

```bash
sudo dpkg -i ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
```

```bash
sudo systemctl enable --now cri-docker.service
```

```bash
sudo modprobe br_netfilter
```

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
```

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
net.netfilter.nf_conntrack_max = 131072
EOF
```

```bash
sudo sysctl --system
```

확인:

```bash
systemctl is-enabled cri-docker.service
systemctl is-active cri-docker.service
```

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv6.conf.all.forwarding
sysctl net.ipv4.ip_forward
sysctl net.netfilter.nf_conntrack_max
```

---

## 에러 상황 1. cri-docker.service가 없다고 나오는 경우

설치 후 service 이름을 확인한다.

```bash
systemctl list-unit-files | grep cri
```

또는 상태를 확인한다.

```bash
systemctl status cri-docker --no-pager
systemctl status cri-dockerd --no-pager
```

다만 문제에서 `cri-docker service`라고 했으므로 기본적으로는 다음 명령어를 먼저 사용한다.

```bash
sudo systemctl enable --now cri-docker.service
```

---

## 에러 상황 2. bridge 관련 sysctl 적용 에러

예를 들어 다음과 같은 에러가 날 수 있다.

```text
cannot stat /proc/sys/net/bridge/bridge-nf-call-iptables
```

이 경우 `br_netfilter` 모듈이 로드되지 않았을 가능성이 있다.

해결:

```bash
sudo modprobe br_netfilter
lsmod | grep br_netfilter
sudo sysctl --system
```

---

## 에러 상황 3. dpkg 설치 중 의존성 문제

`dpkg -i` 중 의존성 에러가 발생할 수 있다.

일반적으로는 다음 명령어로 의존성을 고칠 수 있다.

```bash
sudo apt-get install -f
```

다만 시험 문제에서 별도로 의존성 패키지를 제공하지 않았고, 패키지가 정상적으로 준비되어 있다면 보통 `dpkg -i`만으로 설치된다.

---

## 하지 말아야 할 것

### 1. Docker를 새로 설치하지 않기

문제에 이미 Docker가 설치되어 있다고 되어 있다.

```text
Docker is already installed
```

따라서 Docker 설치 작업은 하지 않는다.

---

### 2. kubeadm init 하지 않기

이 문제는 Linux system prepare 문제이다.

아래 명령어는 요구사항이 아니다.

```bash
sudo kubeadm init
```

---

### 3. containerd 설정 문제로 착각하지 않기

공식문서에는 containerd 설정도 많이 나온다.

하지만 이 문제는 Docker가 이미 설치되어 있고, `cri-dockerd`를 설치하라고 직접 지정했다.

따라서 containerd 설정 파일을 수정하는 문제가 아니다.

---

### 4. sysctl 값을 임시로만 설정하고 끝내지 않기

아래처럼 직접 값을 설정할 수도 있다.

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

하지만 이 방식은 임시 설정이다.

시험에서는 보통 재부팅 후에도 유지될 수 있도록 `/etc/sysctl.d/*.conf` 파일에 남기는 것이 안전하다.

---

## 최종 확인 기준

채점 기준으로 보면 아래 상태가 중요하다.

```text
1. cri-dockerd Debian package가 설치되어 있음
2. cri-docker.service가 enabled 상태
3. cri-docker.service가 active 상태
4. net.bridge.bridge-nf-call-iptables = 1
5. net.ipv6.conf.all.forwarding = 1
6. net.ipv4.ip_forward = 1
7. net.netfilter.nf_conntrack_max = 131072
```

확인 명령어:

```bash
systemctl is-enabled cri-docker.service
systemctl is-active cri-docker.service
```

```bash
sysctl net.bridge.bridge-nf-call-iptables net.ipv6.conf.all.forwarding net.ipv4.ip_forward net.netfilter.nf_conntrack_max
```

---

## 시험장에서의 판단 흐름

```text
Prepare Linux system for Kubernetes
→ Kubernetes 리소스 문제가 아니라 OS 설정 문제

Docker already installed
→ Docker 설치 X

configure it for kubeadm
→ Docker를 kubeadm이 쓰게 cri-dockerd 필요

Install Debian package using dpkg
→ sudo dpkg -i ~/cri-dockerd_...

Enable and start cri-docker service
→ sudo systemctl enable --now cri-docker.service

Configure system parameters
→ /etc/sysctl.d/*.conf 작성
→ sudo sysctl --system
→ sysctl 값 확인
```

---

## 한 줄 핵심 암기

```text
Docker는 이미 있으므로 cri-dockerd만 dpkg로 설치하고,
cri-docker.service를 enable/start 한 뒤,
문제에서 준 sysctl 값을 /etc/sysctl.d에 저장하고 sysctl --system으로 적용한다.
```
