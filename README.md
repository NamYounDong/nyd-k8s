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







# 🧱 쿠버네티스 - 단일서버(No Taint) 구조 정리

---

## 1️⃣ 디렉터리 구조

```text
~/k8s/
├── 00-namespaces.yaml
├── 01-rbac-jenkins.yaml
├── 10-edge-ingress-nginx.yaml
├── 20-cicd-jenkins.yaml
├── 30-infra-mariadb.yaml
├── 31-infra-redis.yaml
├── 32-infra-rabbitmq.yaml
├── 33-infra-kafka-kraft.yaml
├── 34-infra-kafka-ui.yaml
└── 35-infra-jupyter.yaml


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
  

1) 정의  
  - 쿠버네티스 안에서 리소스를 논리적으로 분리하는 “가상 공간”  
  

2) 역할  
  - 서비스 / 환경 / 팀 단위로 리소스 격리  
  - 이름 충돌 방지  
  - 권한 · 쿼터 · 정책을 묶어서 관리  
  


2️⃣ RBAC (Role-Based Access Control)  
  

1) 정의  
  - “누가(ServiceAccount)가  
     무엇을(Resource)  
     어디서(Namespace)  
     얼마나(Action)  
     할 수 있는가”를 정하는 규칙  
  

2) 역할  
  - Jenkins / 앱 / 운영자 권한 분리  
  - 보안 사고 최소화  
  - “필요한 권한만” 부여 (Least Privilege)  
  


3️⃣ Edge  
  

1) 정의  
  - 외부 트래픽이 쿠버네티스 클러스터로 들어오는 “입구”  
  

2) 역할  
  - 외부 요청 수신  
  - 내부 서비스로 라우팅  
  - SSL / TLS 종료  
  - 도메인 / 경로 기반 분기  
```


# ✅ 11 Jenkins + Kubernetes 배포 적용 절차 (실행 순서 & 체크포인트)
```text
────────────────────────────────────────
0️⃣ 노드 준비 (권한 설정)
────────────────────────────────────────
Jenkins가 사용하는 볼륨 디렉터리를  
호스트에 미리 생성하고 권한을 맞춰준다.  

(컨테이너 내부 Jenkins UID=1000 기준)

sudo mkdir -p /home/ubuntu/jenkins_home  
sudo chown -R 1000:1000 /home/ubuntu/jenkins_home  

────────────────────────────────────────
1️⃣ Namespace 생성
────────────────────────────────────────
쿠버네티스 리소스를 논리적으로 분리하기 위한  
기본 네임스페이스를 먼저 생성한다.  

kubectl apply -f 00-namespaces.yaml  

생성 확인:  
kubectl get ns | egrep 'edge|cicd|infra'  

────────────────────────────────────────
2️⃣ RBAC 적용 (Jenkins 권한)
────────────────────────────────────────
Jenkins ServiceAccount와  
배포에 필요한 최소 권한(Role/RoleBinding)을 적용한다.  

kubectl apply -f 01-rbac-jenkins.yaml  

확인:  
kubectl -n cicd get sa  
kubectl -n cicd get role,rolebinding  

────────────────────────────────────────
3️⃣ ingress-nginx 컨트롤러 설치
────────────────────────────────────────
외부 트래픽이 클러스터로 들어오는  
Entry Point(Ingress Controller)를 구성한다.  

kubectl apply -f 10-edge-ingress-nginx.yaml  

Pod 상태 확인 (Running 될 때까지 대기):  
kubectl -n edge get pod -w  

※ ingress-nginx-controller Pod가  
   Running 상태가 되어야 다음 단계 진행 가능  

────────────────────────────────────────
4️⃣ Jenkins 본체 배포
────────────────────────────────────────
Jenkins Deployment / Service / PVC를 배포한다.  

kubectl apply -f 20-cicd-jenkins.yaml  

확인 명령:  
kubectl -n cicd get pvc  
kubectl -n cicd get pod -w  
kubectl -n cicd get svc  

체크 포인트:  
- PVC 상태가 Bound 인지 확인  
- Jenkins Pod 상태가 Running 인지 확인  

────────────────────────────────────────
5️⃣ Jenkins Ingress 룰 적용
────────────────────────────────────────
외부 도메인을 통해  
Jenkins Web UI에 접근할 수 있도록 Ingress를 생성한다.  

kubectl apply -f 21-cicd-jenkins-ingress.yaml  

확인:  
kubectl -n cicd get ingress  

────────────────────────────────────────
6️⃣ 외부 접속 확인
────────────────────────────────────────
DNS 설정 필요:  

- jenkins.dm-nyd.shop  
- 서버 공인 IP를 가리키는 A 레코드 또는 A *  

브라우저 접속:  
http://jenkins.dm-nyd.shop  

※ 현재 단계에서는 SSL(TLS) 미적용 상태  

────────────────────────────────────────
7️⃣ Jenkins 초기 Admin 비밀번호 확인
────────────────────────────────────────
최초 접속 시 필요한  
Jenkins 초기 관리자 비밀번호를 확인한다.  

kubectl -n cicd exec -it deploy/jenkins --  
cat /var/jenkins_home/secrets/initialAdminPassword  

────────────────────────────────────────
✅ 정상 동작 체크리스트 (현재 단계 기준)
────────────────────────────────────────
[ ] edge 네임스페이스에 ingress-nginx Pod Running  
[ ] cicd 네임스페이스에 jenkins Pod Running  
[ ] cicd 네임스페이스에 jenkins-svc (ClusterIP) 존재  
[ ] cicd 네임스페이스에 jenkins-ingress 존재  
[ ] DNS가 서버 IP를 정상적으로 가리킴  
[ ] 브라우저 접속 시 Jenkins 화면 출력됨  
────────────────────────────────────────
```

# ✅ 12 SSL 인증서 적용
### *.dm-nyd.shop(와일드카드) 인증서는 “Ingress/HTTP-01”로는 발급이 안 되고, 반드시 DNS-01 방식으로만 Let’s Encrypt가 발급
### 자동 갱신(90일마다 자동) DNS 레코드를 자동으로 넣었다 빼는 DNS API 연동이 필요. (cert-manager가 TXT 레코드를 자동 생성/삭제)

## 0️⃣ 전제조건
### A. DNS는 와일드카드로 적용
- 호스팅사이트 레코드 A * 11.11.11.11 형식으로 등록
### B. 와일드카드 SSL 자동화는 DNS-01 + DNS API가 필요
- HTTP-01은 와일드카드 불가 
- DNS-01은 가능 
### 가비아는 별도 API를 제공하지 않으므로 CloudFlare 활용


### CloudFlare
```text
1) Cloudflare에 도메인 추가
Cloudflare 가입 / 로그인
“Add a Site”
도메인 입력: dm-nyd.shop
Free Plan 선택 (유료 선택 ❌ 필요 없음)
DNS 레코드 스캔 → 넘어감 (우리가 직접 설정할 거라 크게 신경 X)

2) 네임서버 변경 (가비아 → Cloudflare)
Cloudflare가 네임서버 2개를 보여줄 거야. 예:
alice.ns.cloudflare.com
bob.ns.cloudflare.com
가비아에서 해야 할 일
도메인 관리 → 네임서버 변경
위 Cloudflare 네임서버 2개로 교체
저장

⏱️ 전파 시간
보통 5~30분, 최대 수 시간
확인: nslookup dm-nyd.shop
Cloudflare 네임서버가 보이면 OK.

3) Cloudflare DNS 레코드 설정
Cloudflare → DNS 메뉴에서 아래처럼 설정
-------------------------------------------
Type: A
Name: *
IPv4: <네 EC2 공인 IP>
Proxy status: DNS only (회색 구름)  ✅ 중요
-------------------------------------------

📌 이유:
- cert-manager DNS-01이 TXT 레코드를 직접 다뤄야 함
- Proxy(오렌지 구름) 켜면 인증 과정 꼬일 수 있음

4) Cloudflare API Token 생성 (cert-manager용)
Cloudflare → My Profile → API Tokens → Create Token
템플릿
- Edit zone DNS
권한
- Zone → DNS → Edit
Zone Resources
- Include → Specific zone → dm-nyd.shop
생성 후 API Token 복사 (한 번만 보임)


5) cert-manager Secret 생성 (Cloudflare API 토큰 저장)
---------------------------------------------------------------
  (1) cert-manager 네임스페이스 생성
    kubectl create namespace cert-manager
    확인: kubectl get ns cert-manager
  (2) cert-manager 설치 (CRD → 본체 순서 중요)
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.crds.yaml
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml
    정상 기동 확인: kubectl -n cert-manager get pods
  (3) Secret 등록(cloudeflare api key 등록)
kubectl create secret generic cloudflare-api-token \
  -n cert-manager \
  --from-literal=api-token="여기에_Cloudflare_API_Token_전체문자열" \
  --dry-run=client -o yaml | kubectl apply -f -
---------------------------------------------------------------

6) ClusterIssuer 생성 (DNS-01 + Cloudflare)
kubectl apply -f cloudflare-dns-cluster-issue.yaml
```