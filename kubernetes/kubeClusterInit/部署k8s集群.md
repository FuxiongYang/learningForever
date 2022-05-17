## 部署k8s集群
### ubuntu
-  环境准备

1.  ubuntu
-  安装k8s工具
```shell
apt-get update
apt-get install -y apt-transport-https ca-certificates curl

curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
sudo tee /etc/apt/sources.list.d/kubernetes.list <<EOF
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

apt-get update
apt-get install -y kubelet=1.20.15 kubeadm=1.20.15 kubectl=1.20.15

## 另外，你也可以指定版本安装
## apt-get install kubectl=1.21.3-00 kubelet=1.21.3-00 kubeadm=1.21.3-00
```
- 安装docker
```shell
apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common

curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

# apt-cache madison docker-ce

apt-get update
apt-get install -y docker-ce
```

- kubeadm部署
```shell
kubeadm init --kubernetes-version=v1.20.15 --pod-network-cidr=10.244.0.0/16 --image-repository=registry.aliyuncs.com/google_containers --apiserver-cert-extra-sans ${External_IP}

kubeadm init --apiserver-advertise-address 192.168.0.254 --pod-network-cidr=10.244.0.0/16

# 在master节点上初始化,主要指定k8s版本和网络参数
// 使用flannel网络插件
kubeadm init --kubernetes-version=v1.24.0 --pod-network-cidr=10.244.0.0/16

// 使用calio网络插件
kubeadm init --kubernetes-version=v1.24.0 --pod-network-cidr=192.168.0.0/16

init 常用主要参数：

--kubernetes-version: 指定Kubenetes版本，如果不指定该参数，会从google网站下载最新的版本信息。
--pod-network-cidr: 指定pod网络的IP地址范围，它的值取决于你在下一步选择的哪个网络网络插件，比如我在本文中使用的是flannel网络，需要指定为10.244.0.0/16。
--apiserver-advertise-address: 指定master服务发布的Ip地址，如果不指定，则会自动检测网络接口，通常是内网IP。
--feature-gates=CoreDNS: 是否使用CoreDNS，值为true/false，CoreDNS插件在1.10中提升到了Beta阶段，最终会成为Kubernetes的缺省选项。
--image-repository=registry.aliyuncs.com/google_containers
--apiserver-cert-extra-sans 生成的访问证书，需要外网或者额外网络访问的添加

# 添加新的IP证书
cp -r /etc/kubernetes /etc/kubernetes.bak
rm -rf /etc/kubernetes/pki/apiserver.* 
kubeadm init phase certs apiserver --apiserver-advertise-address ${Internal_IP} --apiserver-cert-extra-sans ${External_IP} 
kubeadm alpha certs renew admin.conf
kubectl -n kube-system delete pod -l component=kube-apiserver
```

- 国内环境下载k8s镜像
```shell
# 从国内（如阿里云）镜像源下载
vim pull_k8s_images.sh

#!/bin/bash
set -e 

kube_version=:v1.14.3
kube_images=(kube-proxy kube-scheduler kube-controller-manager kube-apiserver)
addon_images=(etcd-amd64:3.3.10 coredns:1.3.1 pause-amd64:3.1)

for imageName in ${kube_images[@]} ; do
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName-amd64$kube_version
  docker image tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName-amd64$kube_version k8s.gcr.io/$imageName$kube_version
  docker image rm registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName-amd64$kube_version
done

for imageName in ${addon_images[@]} ; do
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
  docker image tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
  docker image rm registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done

docker tag k8s.gcr.io/etcd-amd64:3.3.10 k8s.gcr.io/etcd:3.3.10
docker image rm k8s.gcr.io/etcd-amd64:3.3.10
docker tag k8s.gcr.io/pause-amd64:3.1 k8s.gcr.io/pause:3.1
docker image rm k8s.gcr.io/pause-amd64:3.1

# 执行权限
chmod +x pull_k8s_images.sh

./pull_k8s_images.sh
```

- 阿里云k8s组件镜像
```shell
k8s.gcr.io/kube-apiserver:v1.24.0
k8s.gcr.io/kube-controller-manager:v1.24.0
k8s.gcr.io/kube-scheduler:v1.24.0
k8s.gcr.io/kube-proxy:v1.24.0
k8s.gcr.io/pause:3.7
k8s.gcr.io/etcd:3.5.3-0
k8s.gcr.io/coredns/coredns:v1.8.6

docker pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.24.0         

docker pull registry.aliyuncs.com/google_containers/kube-proxy:v1.24.0          

docker pull registry.aliyuncs.com/google_containers/kube-controller-manager:v1.24.0 

docker pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.24.0        

docker pull registry.aliyuncs.com/google_containers/etcd:3.5.3-0        

docker pull registry.aliyuncs.com/google_containers/coredns:v1.8.6                                                  

docker pull registry.aliyuncs.com/google_containers/pause:3.7


docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.24.0 k8s.gcr.io/kube-apiserver:v1.24.0

docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.24.0 k8s.gcr.io/kube-proxy:v1.24.0

docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.24.0 k8s.gcr.io/kube-controller-manager:v1.24.0

docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.24.0 k8s.gcr.io/kube-scheduler:v1.24.0

docker tag registry.aliyuncs.com/google_containers/etcd:3.5.3-0  k8s.gcr.io/etcd:3.5.3-0 

docker tag registry.aliyuncs.com/google_containers/coredns:v1.8.6 k8s.gcr.io/coredns:v1.8.6

docker tag registry.aliyuncs.com/google_containers/pause:3.7 k8s.gcr.io/pause:3.7
```

- 部署节点

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- 安装网络组件

```shell
flannel: wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

calico:wget https://docs.projectcalico.org/v3.10/manifests/calico.yaml

kubectl apply -f [network].yaml
```


- 添加节点

```shell
#直接使用指令加入
 kubeadm join 192.168.56.101:6443 --token abcdeff.zlo1o2t9i8e53whd     --discovery-token-ca-cert-hash sha256:3613c3bc855b5d9e1555b99ebd21f48873d20353aeb8c2c64e73d8c7597d43f9
# token加入语句忘记了可以在master上使用下边命令进行生成
 kubeadm token create --print-join-command
```

```shell
# node 节点上生成 join-config.yml，并对主要的参数做修改
$ kubeadm config print join-defaults > join-config.yaml

# join-config.yaml 内容
 apiVersion: kubeadm.k8s.io/v1beta2
caCertPath: /etc/kubernetes/pki/ca.crt
discovery:
  bootstrapToken:
    apiServerEndpoint: kube-apiserver:6443 # 修改为master机器ip:port，192.168.56.101:6443
    token: abcdef.0123456789abcdef		   # 修改为真实token
    unsafeSkipCAVerification: true
  timeout: 5m0s
  tlsBootstrapToken: abcdef.0123456789abcdef # 修改为真实token
kind: JoinConfiguration
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: node01								# 根据实际节点名进行修改
  taints: null

#使用配置文件加入
$ kubeadm join  --config=join-config.yaml
```

## 卸载kubeadm
```shell
kubeadm reset
```