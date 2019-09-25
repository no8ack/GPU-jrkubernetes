## Kubernetes


1. **작업자 생성**
```
$ sudo apt-get update
```
```
$ sudo apt-get upgrade
```
```
$ sudo apt-get install linux-image-extra-virtual
```
```
$ sudo reboot
```
```
$ sudo useradd -s /bin/bash -m k8s-admin
```
```
$ sudo passwd k8s-admin
```
```
$ sudo usermod -aG sudo k8s-admin
```
2. **docker 설치 (master, client) & Nvidia-docker2  설치**

```
$ sudo apt-get update
```
```
$ sudo apt-get install \
   apt-transport-https \
   ca-certificates \
   curl \
   software-properties-common
```
```
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
```
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```
```
$ sudo apt-get update
```
```
$ sudo apt-cache madison docker-ce
```

  이 명령어를 입력하면 다운 받을 수 있는 docker 버전들이 나옵니다.

  ```
$ sudo apt-get install docker-ce=18.06.3~ce~3-0~ubuntu
```
```
$ sudo usermod -aG docker k8s-admin
```

  Docker Service의 cgroup driver를 systemd로 변경해야합니다.
이 부분은 permission 에러가 날 수 있으니 sudo vi로 해서 새로 생성하는 것이 좋습니다.

  ```
$ vi /etc/docker/daemon.json
  ```
```
{
    "exec-opts":["native.cgroupdriver=systemd"],
        "log-driver":"json-file",
        "log-opts":{
            "max-size":"100m"
        },
    "storage-driver":"overlay2",

    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```

  위의 내용을 daemon.json에 추가하면 됩니다.

  ```
$ mkdir -p /etc/systemd/system/docker.service.d
```
  ```
$ systemctl daemon-reload
  ```
```
$ systemctl restart docker
```
**- Nvidia docker 설치**
```
$ curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | \
  sudo apt-key add -
```
```
$ curl -s -L https://nvidia.github.io/nvidia-docker/ubuntu16.04/amd64/nvidia-docker.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-docker.list
```
```
$ sudo apt-get update
```
```
$ sudo apt-get install -y nvidia-docker2
```
```
$ sudo pkill -SIGHUP dockerd
```
설치가 완료되었으면 잘 깔렸는지 확인하기 위해서 CUDA 이미지를 통해 smi를 실행해봅니다.
```
$ docker pull nvidia/cuda
```
```
$ docker run --runtime=nvidia --rm -ti nvidia/cuda nvidia-smi
```
3. **k8s 설치 (master, client)**
```
$ sudo apt install apt-transport-https 
```
```
$ sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
```
```
$ sudo add-apt-repository "deb https://apt.kubernetes.io/ kubernetes-$(lsb_release -cs) main" 
```
```
$ sudo apt update
```
```
$ sudo apt install kubelet kubeadm kubectl kubernetes-cni
```
  만약 kubelet, kubeadm, kubctl의 버전을 낮추고 싶으면 아래와 같이 하면 됩니다.

  ```
$ sudo apt install kubelet=1.14.1-00 kubeadm=1.14.1-00 kubectl=1.14.1-00 kubernetes-cni=0.7.5-00
```

  ```
$ which kubelet
``` 
  /usr/bin/kubelet
```
$ which kubeadm
``` 
/usr/bin/kubeadm

  Swap을 사용하는 경우 k8s 클러스터가 구동되지 않습니다.
swapoff 명령어를 사용하거나 /etc/fstab 파일에서 swap부분을 주석처리(재부팅 필요)합니다.

  ```
$ swapoff -a && sed -i '/swap/d' /etc/fstab
```
```
$ sudo reboot
```
4. ** Master node**
```
$ root@node1:~# export API_ADDR="127.0.0.1" # Master 서버 외부 IP
```
```
$ root@node1:~# export DNS_DOMAIN="k8s.local"
```
```
$ root@node1:~# export POD_NET="10.100.0.0/16" # k8s 클러스터 POD Network CIDR
```
```
$ root@node1:~# sysctl net.bridge.bridge-nf-call-iptables=1
```
```
$ root@node1:~# kubeadm init --pod-network-cidr ${POD_NET} \
--apiserver-advertise-address ${API_ADDR} \
--service-dns-domain "${DNS_DOMAIN}" \
--kubernetes-version=v1.15.3
```

  init을 하고 나면 뒤에 kubeadm join 192.168.56.2:6443 --token xxxxx라고 나옵니다.
kubeadm 부터 끝까지 따로 저장해야 합니다. node들이 서버에 접속할 때 사용됩니다.
5. ** k8s 클러스터 관리 사용자 계정인 k8s-admin 으로 서버에 로그인. **

 ```
$ k8s-admin@node1:~#
```
```
$ k8s-admin@node1:~# \rm -rf $HOME/.k8s
```
```
$ k8s-admin@node1:~# mkdir -p $HOME/.k8s
```
```
$ k8s-admin@node1:~# sudo cp -i /etc/kubernetes/admin.conf $HOME/.k8s/config
```
```
$ k8s-admin@node1:~# sudo chown $(id -u):$(id -g) $HOME/.k8s/config
```
```
$ k8s-admin@node1:~# export KUBECONFIG=$HOME/.k8s/config
```
```
$ k8s-admin@node1:~# echo "export KUBECONFIG=$HOME/.k8s/config" | tee -a ~/.bashrc
```
**- Calico Pod Network 배포하기 **
```
$ k8s-admin@node1:~# wget https://docs.projectcalico.org/v3.5/getting-started/kubernetes/installation/hosted/etcd.yaml
```
```
$ k8s-admin@node1:~# wget https://docs.projectcalico.org/v3.5/getting-started/kubernetes/installation/hosted/calico.yaml
```
```
$ k8s-admin@node1:~# vi calico.yaml
```
 - name: CALICO_IPV4POOL_CIDR <br>
    　　　　　　　 value: "10.100.0.0/16" # POD Network CIDR로 변경합니다
              
 ```
$ k8s-admin@node1:~# kubectl apply -f etcd.yaml
```
```
$ k8s-admin@node1:~# kubectl apply -f calico.yaml
```
```
$ k8s-admin@node1:~# watch kubectl get pods --all-namespaces
```

 | NAMESPACE | NAME | READY | STATUS |
|--------|--------|--------|--------|
|kube-system |calico-etcd-wgzps| 1/1 | Running |

 Calico Pod이 정상적으로 실행된 경우 STATUS 값이 Running으로 출력됩니다. 오류가 발생한 경우 kubectl describe 명령어를 사용하여 해당 컨테이너의 상태를 확인하여 조치를 취합니다.

 Master 노드에도 Pod을 배포하려면 아래와 같은 명령어를 실행합니다.

 ```
$ k8s-admin@master:~# kubectl taint nodes --all node-role.kubernetes.io/master-node/node1 untainted
```
Nvidia device를 사용하기 위해서 pod을 추가해줍니다.
```
$ kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/1.0.0-beta/nvidia-device-plugin.yml
```
6. **Worker node**

 아까 저장했던  kubeadm join 192.168.56.2:6443 --token xxxxx을 입력하면 서버로 접속이 됩니다.
```
$ root@node2:~# kubeadm join 192.168.56.2:6443 --token kakych.0ebc0g9vbl6xo8hp \
    --discovery-token-ca-cert-hash sha256:8a0feaa2cea150cadb56f36f39c18cc5650852faee252092d39298ccda928344
```

 Master 노드에서 아래 명령어로 노드 상태를 확인합니다. (master node에서만 확인가능.)

 ```
$ k8s-admin@node1:~# kubectl get no
```

 | NAME | STATUS | ROLES | AGE | VERSION |
|--------|--------|--------|--------|--------|
|node1|READY| master | 20m | v1.15.3 |
|node2|READY| none | 11m 21s | v1.15.3 |



