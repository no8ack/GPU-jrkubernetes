# jrkubernetes

# 1. 작업자 생성

$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt-get install linux-image-extra-virtual
$ sudo reboot

$ sudo useradd -s /bin/bash -m k8s-admin
$ sudo passwd k8s-admin
$ sudo usermod -aG sudo k8s-admin

# 2. 도커 설치(master, client 둘다)

1) 도커 삭제

$ sudo apt-get remove docker docker-engine docker.i docker-ce docker-ce-cli

2) 도커 설치 

$ sudo apt-get update
$ sudo apt-get install \
  apt-transport-https \
  ca-certificates \
  curl \
  software-properties-common
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"
$ sudo apt-get update
$ sudo apt-cache madison docker-ce
docker-ce | 5:18.09.5~3-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 5:18.09.4~3-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 5:18.09.3~3-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 5:18.09.2~3-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 5:18.09.1~3-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 5:18.09.0~3-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 18.06.3~ce~3-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 18.06.2~ce~3-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 18.06.1~ce~3-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 18.06.0~ce~3-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 18.03.1~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 18.03.0~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 17.12.1~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 17.12.0~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 17.09.1~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 17.09.0~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 17.06.2~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 17.06.1~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 17.06.0~ce-0~ubuntu | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 17.03.3~ce-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 17.03.2~ce-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 17.03.1~ce-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages
docker-ce | 17.03.0~ce-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu xenial/stable amd64 Packages

$ sudo apt-get install docker-ce=18.06.3~ce~3-0~ubuntu # validated 버전 18.06('19 2/20)
현재로는 이 버전이 stable 이다.

$ sudo usermod -aG docker k8s-admin

Docker Service의 cgroup driver를 systemd로 변경
  이 부분은 permission 에러가 날 수 있으니 sudo vi로 해서 새로 생성할 것.
$ cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

- Docker Service 설정 디렉토리 생성

$ mkdir -p /etc/systemd/system/docker.service.d

- Docker Service 재시작

$ systemctl daemon-reload
$ systemctl restart docker


# 3. k8s 설치 (master, client 둘다)

마스터 ID : test 
서버 IP : 220.67.133.XXX

$ sudo apt install apt-transport-https 
$ sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
$ sudo add-apt-repository "deb https://apt.kubernetes.io/ kubernetes-$(lsb_release -cs) main" 
$ sudo apt update 

$ sudo apt install kubelet kubeadm kubectl kubernetes-cni
$ sudo apt install kubelet=1.14.1-00 kubeadm=1.14.1-00 kubectl=1.14.1-00 kubernetes-cni=0.7.5-00
쿠버네티스 버전을 다르게 하려면 위와 같이 하면 된다.

Swap을 사용하는 경우 k8s 클러스터가 구동되지 않습니다.
swapoff 명령어를 사용하거나 /etc/fstab 파일에서 swap부분을 주석처리(재부팅 필요)합니다.

$ swapoff -a && sed -i '/swap/d' /etc/fstab 
$ sudo reboot

3-1) 마스터만 설치
root@node1:~# export API_ADDR="220.67.133.XXX" # Master 서버 외부 IP
root@node1:~# export DNS_DOMAIN="k8s.local"
root@node1:~# export POD_NET="10.100.0.0/16" # k8s 클러스터 POD Network CIDR -> 이거 calico 서버 ip
root@node1:~# sysctl net.bridge.bridge-nf-call-iptables=1
root@node1:~# kubeadm init --pod-network-cidr ${POD_NET} \
--apiserver-advertise-address ${API_ADDR} \
--service-dns-domain "${DNS_DOMAIN}"  \
--kubernetes-version=v1.15.3   (잘못 설치했는지 패스 잘못잡아서 직접 설정해줌)

xxxxxxxxx 뭐라뭐라 나오고 마지막에

kubeadm join 192.168.56.2:6443 --token kakych.0ebc0g9vbl6xo8hp \
    --discovery-token-ca-cert-hash sha256:8a0feaa2cea150cadb56f36f39c18cc5650852faee252092d39298ccda928344

이거 나오는데 이거 저장해야합니다. client가 서버에 접속하는 passwd 입니다.
k8s 클러스터 관리 사용자 계정인 k8s-admin 으로 서버에 로그인 합니다.



- k8s Master 노드 접속하기 위한 설정
k8s-admin@node1:~$ \rm -rf $HOME/.k8s
k8s-admin@node1:~$ mkdir -p $HOME/.k8s
k8s-admin@node1:~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.k8s/config
k8s-admin@node1:~$ sudo chown $(id -u):$(id -g) $HOME/.k8s/config
k8s-admin@node1:~$ export KUBECONFIG=$HOME/.k8s/config
k8s-admin@node1:~$ echo "export KUBECONFIG=$HOME/.k8s/config" | tee -a ~/.bashrc

이건 꼭 해줘야됨. kubeadm reset으로 리셋했을 때, 이거 안해주면 calico 접속할 때 authentication에서 에러남







