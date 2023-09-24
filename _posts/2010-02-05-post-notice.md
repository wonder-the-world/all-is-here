---
title: "Post: aliyun install k8s"
categories:
  - 云原生技术
tags:
  - cloud native 
---

##  资源准备
1. 三台服务器： master 1 台，2 台 work node

## 安装步骤
1. 添加`yum` 源
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
2. 安装`kubelet`,`kubeadm`,`kubectl`
```
#安装
yum install -y kubelet kubeadm kubectl
# 设置kubelet开机启动
systemctl enable kubelet && systemctl start kubelet
```
### 安装docker
按照如下步骤安装docker 相关依赖
```
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum-config-manager --disable docker-ce-edge
yum-config-manager --disable docker-ce-test

# Step 3: 更新并安装 Docker-CE
sudo yum makecache fast
sudo yum -y install docker-ce

# Step 4: 开启Docker服务
sudo service docker start

# Step 5: 更改cgroup driver
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
```
### 拉取k8s 相关镜像
google和docker似乎是有意要对着干的，虽然阿里云也有docker registry的加速器，但是google并没有将kubernetes的镜像放到docker hub上。所以，我们需要先使用脚本，从阿里云的google_containers命名空间下载对应的克隆镜像，然后再通过docker tag将其labels修改为kubeadm生成的static-pod yaml文件对应的镜像标签。从而欺骗kubeadm，所有镜像都已经ready了，不用再去公网上拉取了。
```
#/usr/bin/bash

for i in `kubeadm config images list`; do 
  imageName=${i#k8s.gcr.io/}
  docker pull registry.aliyuncs.com/google_containers/$imageName
  docker tag registry.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
  docker rmi registry.aliyuncs.com/google_containers/$imageName
done;
```
### 安装网络

采用 flanneel 网络
```
kubectl apply -f https://github.com/flannel-io/flannel/blob/master/Documentation/kube-flannel.yml
```
### kubeadm init 初始化
执行如下命令完成master 节点 的初始化
```
kubeadm init --kubernetes-version=$(kubeadm version -o short)  --pod-network-cidr=10.244.0.0/16
```
### 加入node节点

```
# 在master 节点上执行如下命令后的输出，在node 节点上去执行
kubeadm token create --print-join-command
# 输出如下
kubeadm join 172.22.229.152:6443 --token 3fmx3o.r80t7quc8yimvbj4 --discovery-token-ca-cert-hash sha256:6223679b37cd252362f946454a5ef34ef1a6ee3ad028e5e499e6c0823d6fcfe1 


`kubeadm init`/ `kubeadm join` 之后，在`/etc/kubenetes/` 目录下会生成kubelet.conf 文件
 mv /etc/kubernetes/kubelet.conf /etc/kubernetes/admin.conf
 

```