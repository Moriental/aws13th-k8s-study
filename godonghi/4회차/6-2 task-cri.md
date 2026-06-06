# 6-2 task-cri 핵심 정리

## 1. 문제 핵심

Kubernetes를 위해 Linux 시스템을 준비하는 문제다.

해야 할 일:

```text
1. cri-dockerd deb 패키지 설치
2. cri-docker 서비스 enable/start
3. Kubernetes용 sysctl 커널 파라미터 설정
```

문제에서 요구한 패키지:

```text
~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
```

---

## 2. 필요한 개념

Kubernetes는 컨테이너 런타임과 통신할 때 CRI(Container Runtime Interface)를 사용한다.

Docker는 Kubernetes에서 직접 런타임으로 붙지 않고 `cri-dockerd`를 통해 CRI로 연결된다.

구조:

```text
kubelet
  ↓ CRI
cri-dockerd
  ↓
Docker Engine
```

`dpkg -i`는 Debian/Ubuntu 계열에서 `.deb` 패키지를 설치하는 명령어다.

`systemctl enable --now`는 서비스를 부팅 시 자동 시작으로 등록하고 즉시 시작한다.

`sysctl`은 Linux 커널 파라미터를 설정하는 명령어다.

---

## 3. 공식문서 기반 풀이 방식

Kubernetes 공식문서의 Container Runtimes 문서에는 sysctl 값을 `/etc/sysctl.d/k8s.conf`에 저장하고 `sysctl --system`으로 적용하는 구조가 나온다.

공식문서 구조:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

문제에서는 설정해야 할 값이 4개로 주어진다.

```text
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
net.netfilter.nf_conntrack_max = 131072
```

따라서 공식문서의 구조에 문제에서 요구한 4개 값을 넣어서 작성한다.

---

## 4. 환경 확인

문제 원문은 Ubuntu Jammy용 deb 패키지를 전제로 한다.

확인 명령어:

```bash
cat /etc/os-release
which dpkg
ls -l ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
docker --version
systemctl status docker --no-pager
which kubeadm
which kubelet
systemctl status kubelet --no-pager
```

처음에는 deb 파일이 없어서 설치가 실패했다.

```text
dpkg: error: cannot access archive '/root/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb': No such file or directory
Failed to enable unit: Unit file cri-docker.service does not exist.
```

원인:

```text
cri-dockerd deb 파일이 없음
cri-dockerd가 설치되지 않았으므로 cri-docker.service도 없음
```

---

## 5. 실습 환경 준비

deb 파일 다운로드:

```bash
cd ~

wget -O cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb \
https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.9/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
```

확인:

```bash
ls -l ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
```

확인 결과:

```text
-rw-r--r-- 1 root root 11108590 Jan  2  2024 /root/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
```

---

## 6. 풀이 명령어

cri-dockerd 설치:

```bash
sudo dpkg -i ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
```

cri-docker 서비스 enable/start:

```bash
sudo systemctl enable --now cri-docker
```

설치 결과:

```text
Selecting previously unselected package cri-dockerd.
Unpacking cri-dockerd (0.3.9~3-0~ubuntu-jammy) ...
Setting up cri-dockerd (0.3.9~3-0~ubuntu-jammy) ...
Created symlink /etc/systemd/system/multi-user.target.wants/cri-docker.service → /usr/lib/systemd/system/cri-docker.service.
Created symlink /etc/systemd/system/sockets.target.wants/cri-docker.socket → /usr/lib/systemd/system/cri-docker.socket.
```

sysctl 설정:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
net.netfilter.nf_conntrack_max = 131072
EOF

sudo sysctl --system
```

---

## 7. 확인 명령어

서비스 상태 확인:

```bash
systemctl status cri-docker --no-pager
systemctl is-enabled cri-docker
```

확인 결과:

```text
● cri-docker.service - CRI Interface for Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/cri-docker.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-06-06 17:29:08 UTC
TriggeredBy: ● cri-docker.socket
   Main PID: 5255 (cri-dockerd)

enabled
```

sysctl 값 확인:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv6.conf.all.forwarding
sysctl net.ipv4.ip_forward
sysctl net.netfilter.nf_conntrack_max
```

확인 결과:

```text
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
net.netfilter.nf_conntrack_max = 131072
```

---

## 8. 최종 성공 기준

```text
cri-dockerd deb 패키지 설치 완료
cri-docker.service active (running)
cri-docker enabled
net.bridge.bridge-nf-call-iptables = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
net.netfilter.nf_conntrack_max = 131072
```

---
