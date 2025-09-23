# k8s-cluster-build
**k8s 클러스터 구축 및 서비스 배포 기록 레포지토리입니다.** 

---
### 클러스터 초기 구성도

<img width="1133" height="527" alt="k8s 클러스터 초기구성도" src="https://github.com/user-attachments/assets/ba4dd414-4b1f-4110-adf0-8e4baa703170" />


**설명**:

**Masternode**: control-plane 역할 (API 서버, Scheduler, Controller Manager, etcd)

**Workernode**: Pod 실행 및 kube-proxy, Calico Node 운영

**클라이언트**는 Masternode의 API 서버를 통해 클러스터와 통신

<br>



## Kubernetes Cluster 설치 및 Calico CNI 구성 (Ubuntu 24.04 + Containerd)

### 환경 정보

| 노드         | 역할           | CPU/메모리 | IP 주소       |
|-------------|---------------|------------|---------------|
| masternode  | control-plane | 2 Core / 8GB RAM | 10.0.2.15 |
| workernode01| worker        | 2 Core / 4GB RAM | 10.0.2.20 |
| workernode02| worker        | 2 Core / 4GB RAM | 10.0.2.25 |

---

## 1. 사전 설치

### 1.1 Containerd 설치

```bash
sudo apt update && sudo apt install -y containerd
containerd --version
```
**설명**:

Kubernetes는 Container Runtime이 필요합니다.

여기서는 containerd를 설치했으며, 설치 후 버전을 확인합니다.


### 1.2 네트워크 모듈 및 커널 파라미터 설정
Bash

**필요한 커널 모듈을 영구적으로 로드하도록 설정**
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

---

**설정 즉시 적용**
```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

---

**쿠버네티스 네트워킹에 필요한 sysctl 파라미터 설정**
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
```

---

**재부팅 없이 변경사항 완전 적용**
```bash
sudo sysctl --system
```
**설명**:

이 설정은 Pod 간의 통신과 외부 네트워크 연결을 위해 반드시 필요한 설정입니다.

overlay, br_netfilter 모듈 로드:

overlay: Containerd와 같은 컨테이너 런타임이 사용하는 오버레이 파일 시스템을 위한 모듈입니다. 컨테이너의 레이어드 이미지를 효율적으로 관리합니다.

br_netfilter: 리눅스 브릿지를 통과하는 IPv4/IPv6 트래픽이 iptables 체인을 통과하도록 만들어 줍니다. 이를 통해 Pod 트래픽에 대한 방화벽 규칙(Network Policy 등)을 적용할 수 있게 됩니다.

sysctl 커널 파라미터 설정:

net.bridge.bridge-nf-call-iptables = 1: 브릿지로 연결된 네트워크 인터페이스를 통과하는 트래픽에 iptables 규칙이 적용되도록 활성화합니다.

net.ipv4.ip_forward = 1: IP 포워딩을 활성화합니다. 이 설정이 없으면 한 Pod에서 다른 노드에 있는 Pod로 트래픽을 전달(라우팅)할 수 없습니다.

---

### 1.3 필수 패키지 설치
```bash
sudo apt-get install -y apt-transport-https ca-certificates
```
**설명**:

HTTPS 리포지토리 접근 및 인증서 관리를 위해 필요한 패키지입니다.

Ubuntu 기본 apt install 만으로 설치 시, 버전 이슈가 있을 수 있어 apt-get 사용

---

### 1.4 Kubernetes 공식 GPG 키 다운로드 및 저장소 추가
```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
**설명**:

Kubernetes 패키지를 설치하기 위한 공식 GPG 키 등록

안정적인 패키지 설치를 위해 저장소를 추가

---

### 1.5 저장소 활성화 옵션 추가
```bash
sudo tee /etc/apt/apt.conf.d/99allow-insecure-repositories <<EOF
APT::Get::AllowUnauthenticated "true";
Acquire::AllowInsecureRepositories "true";
Acquire::AllowDowngradeToInsecureRepositories "true";
EOF
sudo apt-get update --allow-unauthenticated
```
**설명**:

인증 오류 발생 시 임시로 인증 무시 옵션 적용

---

# 2. 마스터 노드 초기화
```bash
코드 복사
sudo kubeadm init \
  --apiserver-advertise-address=10.0.2.15 \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket /run/containerd/containerd.sock
```

---

설명:

--**apiserver-advertise-address**: 클러스터 API 서버 접근용 IP 지정

--**pod-network-cidr**: Calico 설치 시 필요, Pod 통신용 네트워크 범위

--**cri-socket**: containerd 경로 지정

---

### 2.1 kubeconfig 설정
```bash
코드 복사
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
설명:

kubectl 명령어를 사용하기 위해 kubeconfig 설정

root 계정이 아닌 일반 사용자로 kubectl 실행 가능

---

# 3. CNI 설치 (Calico)
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```
**설명**:

Pod 간 통신을 위한 CNI(Container Network Interface) 설치

Calico는 BGP 기반 네트워크 플러그인으로 Pod-to-Pod 통신과 네트워크 정책 제공

---

### 3.1 네트워크 설정
/etc/hosts에 각 노드 IP와 호스트명 등록

```text
127.0.0.1 localhost
10.0.2.15 masternode
10.0.2.20 workernode01
10.0.2.25 workernode02
```
/etc/resolv.conf는 외부 DNS뿐만 아니라 CoreDNS ClusterIP 포함

```text
nameserver 10.96.0.10  # Kubernetes Cluster DNS
nameserver 8.8.8.8
nameserver 1.1.1.1
```
**설명**:

각 노드와 Pod가 서로 이름으로 통신 가능하게 설정

---

# 4. 워커 노드 클러스터 합류
```bash
sudo kubeadm join 10.0.2.15:6443 \
  --token <토큰> \
  --discovery-token-ca-cert-hash sha256:<해시>
```
**설명**:

TLS 인증을 통해 워커 노드를 클러스터에 합류

# 5. 상태 확인
### 5.1 노드 상태 확인
```bash
ubuntu@masternode:~$ kubectl get nodes -o wide
NAME           STATUS   ROLES           AGE   VERSION    INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
masternode     Ready    control-plane   23m   v1.30.14   10.0.2.15     <none>        Ubuntu 24.04.2 LTS   6.8.0-83-generic   containerd://1.7.27
workernode01   Ready    <none>          15m   v1.30.14   10.0.2.20     <none>        Ubuntu 24.04.2 LTS   6.8.0-83-generic   containerd://1.7.27
workernode02   Ready    <none>          15m   v1.30.14   10.0.2.25     <none>        Ubuntu 24.04.2 LTS   6.8.0-83-generic   containerd://1.7.27

```
모든 노드가 Ready 상태인지 확인

---

### 5.2 Pod 상태 확인
```bash
ubuntu@masternode:~$ kubectl get pods -A -o wide
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE    IP              NODE           NOMINATED NODE   READINESS GATES
kube-system   calico-kube-controllers-5b9b456c66-2h744   1/1     Running   0          19m    192.168.181.3   masternode     <none>           <none>
kube-system   calico-node-57jgp                          1/1     Running   0          19m    10.0.2.15       masternode     <none>           <none>
kube-system   calico-node-85scc                          1/1     Running   0          17m    10.0.2.20       workernode01   <none>           <none>
kube-system   calico-node-m7k2h                          1/1     Running   0          3m2s   10.0.2.25       workernode02   <none>           <none>
kube-system   coredns-55cb58b774-gkndd                   1/1     Running   0          24m    192.168.181.1   masternode     <none>           <none>
kube-system   coredns-55cb58b774-xbjz2                   1/1     Running   0          24m    192.168.181.2   masternode     <none>           <none>
kube-system   etcd-masternode                            1/1     Running   0          24m    10.0.2.15       masternode     <none>           <none>
kube-system   kube-apiserver-masternode                  1/1     Running   0          24m    10.0.2.15       masternode     <none>           <none>
kube-system   kube-controller-manager-masternode         1/1     Running   0          24m    10.0.2.15       masternode     <none>           <none>
kube-system   kube-proxy-44z46                           1/1     Running   0          17m    10.0.2.20       workernode01   <none>           <none>
kube-system   kube-proxy-fl8q8                           1/1     Running   0          17m    10.0.2.25       workernode02   <none>           <none>
kube-system   kube-proxy-w2h5s                           1/1     Running   0          24m    10.0.2.15       masternode     <none>           <none>
kube-system   kube-scheduler-masternode                  1/1     Running   0          24m    10.0.2.15       masternode     <none>           <none>
```
네임스페이스 내 CoreDNS, Calico, kube-proxy, control-plane Pod 정상 동작 확인

---

# 6. 네트워크 및 방화벽 관련 설명
**포트 6443 (API 서버): 클러스터 내부 NAT 환경에서는 방화벽 미설정 상태에서도 동작**

실제 외부 워커 노드와 통신하려면 반드시 6443 포트 허용 필요

Pod-to-Pod 통신: Calico 설치 시 지정한 --pod-network-cidr 범위 내에서 IP 할당, BGP 기반 라우팅

CoreDNS: ClusterIP 10.96.0.10로 클러스터 내부 DNS 제공, 외부 DNS와 병행 가능
<br>

---

# 7. 참고문서
kubeadm 문서: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/

Calico 문서: https://docs.projectcalico.org/
