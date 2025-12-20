# ✅ 1. EC2 준비
## 스펙
> t3.medium (2 vCPU / 4GB RAM)
> Ubuntu 22.04 LTS 권장

## 보안그룹 열어야 할 포트
포트	용도
22 : SSH
6443 : Kubernetes API 서버
80, 443 : 나중에 NPM 또는 Ingress로 외부 서비스
30000–32767 : NodePort 기본 포트 범위 (선택)
나중에 Ingress Controller 쓰면 80/443 정도만 쓰게 됨.

# ✅ 2. OS 기본 설정
접속 후:
sudo apt update \\\&\\\& sudo apt upgrade -y
sudo timedatectl set-timezone Asia/Seoul

# ✅ 3. swap 완전 OFF (K8s 필수)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab


#✅ 4. 커널 모듈 + sysctl 설정
K8s + containerd가 정상 동작하려면 필수.
## 4-1. 커널 모듈 등록
### cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
> br\_netfilter
> overlay
> EOF
### sudo modprobe br\_netfilter
### sudo modprobe overlay

## 4-2. 네트워크 셋팅
### cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
> net.bridge.bridge-nf-call-iptables=1
> net.bridge.bridge-nf-call-ip6tables=1
> net.ipv4.ip\_forward=1
> EOF
### sudo sysctl --system

### Kubernetes는 Linux 커널 기능 몇 가지를 필수로 요구함.
### ✔ 브릿지 네트워크 트래픽을 iptables로 전달
### net.bridge.bridge-nf-call-iptables = 1
### net.bridge.bridge-nf-call-ip6tables = 1
### → Pod 간 통신, CNI 네트워크(flannel, calico 등) 동작을 위해 필요

### ✔ IP 포워딩 허용
### net.ipv4.ip\_forward = 1
### → Pod → Node → 외부로 트래픽이 나가기 위해 필요
## 이 값을 적용하지 않으면:
### Pod 네트워크 연결 안 됨
### kube-proxy, CNI 설치 실패
### 외부 연결 불가
### 노드 NotReady 발생
### 같은 문제가 생김.
### 그래서 K8s 설치 전에 반드시 커널 파라미터를 미리 설정해두는 것이 정석 절차임.

# ✅ 5. containerd 설치 + systemd cgroup 설정
## 5-1. 설치
### sudo apt install -y containerd

## 5-2. 기본 설정파일 생성
### sudo mkdir -p /etc/containerd
### sudo containerd config default | sudo tee /etc/containerd/config.toml

## 5-3. systemd cgroup 활성화 (K8s 필수)
### sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

## 5-4. 재시작 + 활성화
### sudo systemctl restart containerd
### sudo systemctl enable containerd

## ✅ 6. kubeadm / kubelet / kubectl 설치
### 6-1. Kubernetes apt repo 등록
### sudo apt install -y apt-transport-https ca-certificates curl gpg
### sudo mkdir -p /etc/apt/keyrings
### curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \\
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg (그대로 줄바꿈까지 다 복사해서 쓸것)
### echo "deb \[signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \\
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" \\
  | sudo tee /etc/apt/sources.list.d/kubernetes.list  (그대로 줄바꿈까지 다 복사해서 쓸것)
### sudo apt update

## 6-2. K8s 패키지 설치 + 버전 고정
### sudo apt install -y kubelet kubeadm kubectl
### sudo apt-mark hold kubelet kubeadm kubectl

# ✅ 7. kubeadm init (클러스터 생성)
## 7-1. EC2 프라이빗 IP 확인
### hostname -I
예시: 172.31.15.100
### 7-2. kubeadm init 실행
### Calico의 기본 pod CIDR(192.168.0.0/16) 기준으로 설정: 
### sudo kubeadm init \\
  --apiserver-advertise-address=172.31.15.100 \\
  --pod-network-cidr=192.168.0.0/16     (그대로 줄바꿈까지 다 복사해서 쓸것)
###   --apiserver-advertise-address=172.31.15.100 \\ 여기에 hostname -I 로 조회된 주소 입력

### ⚠️ 여기 IP는 반드시 너 EC2의 프라이빗 IP 넣어야 함
### 퍼블릭 IP 넣으면 절대 안 됨.
### 성공 메시지가 뜨면:
### admin.conf 경로 안내
### 워커 노드 조인 명령 출력됨 (지금은 무시)


# ✅ 8. kubectl 사용 준비
### mkdir -p $HOME/.kube
### sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
### sudo chown $(id -u):$(id -g) $HOME/.kube/config
### 테스트: kubectl get nodes
### 현재는 노드 상태가 NotReady → 네트워크 플러그인 설치 후 Ready 로 바뀜.

# ✅ 9. CNI(네트워크 플러그인) 설치 – Calico 
## 쿠버네티스에서 Pod들이 서로 통신하고, 외부와 통신할 수 있게 네트워크를 만들어주는 “네트워크 담당 플러그인”.
## 쿠버네티스 자체에는 “네트워크 기능”이 없다.
\- Pod에 IP를 어떻게 줄지?
\- Pod끼리 어떻게 통신할지?
\- 노드 간 네트워크는 어떻게 연결할지?
\- 외부 인터넷과 통신할 때 NAT은 누가 처리할지?
\- 이런 걸 쿠버네티스 본체는 아무것도 안 해줌.

### kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

### 잠시 후 확인:
### kubectl get pods -n kube-system
### kubectl get nodes
### 노드가 Ready 로 바뀜 → 네트워크 정상.

# ✅ 10. 싱글 노드용 taint 제거 (서버 한대인 경우)
### 기본적으로 Control Plane은 Pod 스케줄링이 막혀있다.
### kubectl taint nodes --all node-role.kubernetes.io/control-plane-
### control-plane(쿠버네티스 두뇌 같은 기능) 기능을 제거 하는게 아니라 "역할의 제한을 해제하는 것"
### control-plane 서버(노드) - 워커 서버(노드) 와 같이 연결형태로 활용되는 것이 안정성 측면에서 합당한 구조이기 때문에 control-plane 노드인 서버에 POD를 올릴수 없게되어 있는 제약이 걸려 있는데 이를 해제하여 서버 한대에서 control-plane, 워커 모두 활용할 수 있도록 처리.

### 확인: kubectl describe node | grep -i taint
### 아무 것도 안 뜨면 OK
# Kubernetes 기본 골격 구축 완료







### 쿠버네티스 구조
## 1) 디렉터리 구조
~/k8s/
  00-namespaces.yaml
  01-rbac-jenkins.yaml
  10-edge-ingress-nginx.yaml
  20-cicd-jenkins.yaml
  30-infra-mariadb.yaml
  31-infra-redis.yaml
  32-infra-rabbitmq.yaml
  33-infra-kafka-kraft.yaml
  34-infra-kafka-ui.yaml
  35-infra-jupyter.yaml

📐 단계별 의미 (네 파일 기준)
🔹 00-09 : 클러스터 “기반 뼈대”
    - Namespace 없으면 → 이후 모든 리소스 생성 불가
    - RBAC 없으면 → ServiceAccount / Jenkins 접근 불가
🔹 10-19 : Edge / Gateway / Entry Point
    - 외부 트래픽 진입 지점
    - Nginx Proxy Manager / Ingress / Gateway API 등
🔹 20-29 : CI/CD · 플랫폼 레벨
🔹 30-39 : Infra (DB / MQ / Cache / 분석 도구)
    - 애플리케이션이 의존하는 서비스

1️⃣ Namespace
1) 정의 : 쿠버네티스 안에서 리소스를 논리적으로 분리하는 “가상 공간”
2) 역할 :
    - 서비스/환경/팀 단위로 리소스 격리
    - 이름 충돌 방지
    - 권한·쿼터·정책을 묶어서 관리

2️⃣ RBAC (Role-Based Access Control)
1) 정의 : “누가(ServiceAccount)가 / 무엇을(Resource) / 어디서(Namespace) / 얼마나(Action) 할 수 있는가”를 정하는 규칙
2) 역할 :
    - Jenkins / 앱 / 운영자 권한 분리
    - 보안 사고 최소화
    - “필요한 권한만” 부여 (Least Privilege)

3️⃣ Edge
1) 정의 : 외부 트래픽이 쿠버네티스 클러스터로 들어오는 “입구”
2) 역할 :
    - 외부 요청 수신
    - 내부 서비스로 라우팅
    - SSL/TLS 종료
    - 도메인/경로 기반 분기

