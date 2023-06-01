---
layout: post
title:  "搭建Kubernetes高可用集群"
date:   2023-07-10 20:00:00 +0800
categories: Kubernetes
---


# prepare

## 操作系统prepare

**禁用swap**：swapoff -a

**查看主机名称**：cat /proc/sys/kernel/hostname  、 hostname -i  、 uname -n

**设置主机名称**：hostnamectl set-hostname test-slave1

**mac地址**： `ip link | awk '{print $2}'`

**检查product_uuid是否相同**: dmidecode -s system-uuid

**时间同步**：
```
yum  -y  install  ntp  ntpdate   
ntpdate  cn.pool.ntp.org
```

## docker/containerd/kubelet 安装

### 快速的安装docker与containerd

> https://docs.docker.com/engine/install/centos/

```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
export VERSION_STRING=3:24.0.2-1.el7
sudo yum -y install docker-ce-$VERSION_STRING docker-ce-cli-$VERSION_STRING containerd.io docker-buildx-plugin docker-compose-plugin
mkdir -p /etc/docker
touch /etc/docker/daemon.json
cat > /etc/docker/daemon.json<<EOF
{ "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"], "exec-opts": ["native.cgroupdriver=systemd"] }
EOF
systemctl enable docker
systemctl start docker 
systemctl status docker 

```

在安装好docker后，containerd也相应的安装好，他们的关系如下：
> 可以参考一文搞懂containerd：https://www.qikqiak.com/post/containerd-usage/

![img]({{ site.url }}/assets/k8s-ha-img1.png)
同时连接docker和containerd的客户端docker、ctr也安装好了。

### 安装kubelet并配置镜像源
> https://developer.aliyun.com/mirror/kubernetes/?spm=a2c6h.25603864.0.0.344135dcps44Yf

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
setenforce 0
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet && systemctl start kubelet
```

gpg 内部部署软件可以关闭检查。
> gpgcheck=0   repo_gpgcheck=0

在安装好后，也会自动安装好作为CRI的客户端的crictl。

### containerd的配置修改(尤其unix socket)

> https://forum.linuxfoundation.org/discussion/862825/kubeadm-init-error-cri-v1-runtime-api-is-not-implemented

#### containerd服务配置
在安装好后，需要修改containerd的配置。docker中，默认contianerd的是不使用unix socket的，需要在配置文件中修改。因为在本文笔者安装的Kubernetes版本较新，已不支持docker作为CRI，所以一定要配置containerd作为CRI(当然可以选择其他的CRI，只不过containerd是最流行的)。plugin配置修改：
Like this, or with your favorite editor: sudo nano /etc/containerd/config.toml and comment out the line that says disabled_plugins = ["cri"].
同时也要配置镜像加速，如sandbox的配置。可通过 containerd config default > /etc/containerd/config.toml 来导出containerd全量的配置。以下为containerd修改好镜像源的config配置：

```
#   Copyright 2018-2022 Docker Inc.

#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at

#       http://www.apache.org/licenses/LICENSE-2.0

#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.

#disabled_plugins = ["cri"]

#root = "/var/lib/containerd"
#state = "/run/containerd"
#subreaper = true
#oom_score = 0

#[grpc]
#  address = "/run/containerd/containerd.sock"
#  uid = 0
#  gid = 0

#[debug]
#  address = "/run/containerd/debug.sock"
#  uid = 0
#  gid = 0
#  level = "info"
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6"
    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = ""
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://b9pmyelo.mirror.aliyuncs.com"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
          endpoint = ["registry.aliyuncs.com/google_containers"]
```

CRI的接口：https://github.com/kubernetes/kubernetes/blob/release-1.5/pkg/kubelet/api/v1alpha1/runtime/api.proto ,其中包括关于ImageService和RuntimeService

重启containerd：
```
systemctl enable containerd
systemctl restart containerd
systmectl status containerd
```

#### crictl配置修改

crictl是cri的客户端，ctr是containerd的客户端

```
cat > /etc/crictl.yaml<<'EOF'
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: true
EOF
```

#### containerd 可能遇到的问题
> crictl 的配置文件是 /etc/crictl.yaml

containerd.sock有两个，分别是：(to understand)
/run/containerd/containerd.sock
/run/docker/containerd/containerd.sock

```
runtime-endpoint: "unix:///run/containerd/containerd.sock"
image-endpoint: ""
timeout: 0
debug: false
pull-image-on-create: false
disable-pull-on-run: false
```

当配置 /run/docker/containerd/containerd.sock的时候，会报错crictl images:

> FATA[0000] validate service connection: CRI v1 image API is not implemented for endpoint "unix:///run/docker/containerd/containerd.sock": rpc error: code = Unimplemented desc = unknown service runtime.v1.ImageService

关于sock选择的问题：(之前默认是docker-shim，后来给去掉了)
https://github.com/kubernetes-sigs/cri-tools/issues/1089

https://github.com/kubernetes-sigs/cri-tools/issues/1041

https://github.com/kubernetes-sigs/cri-tools/issues/870


# kubeadm init
在control-plane初始化前，我们需要给控制面配置一个VIP来保证所有的kubelet都连接的是VIP。下面的172.16.115.80就是VIP。

## keepalived配置

> keepalived的配置： https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#options-for-software-load-balancing

```
${STATE} is MASTER for one and BACKUP for all other hosts, hence the virtual IP will initially be assigned to the MASTER.
${INTERFACE} is the network interface taking part in the negotiation of the virtual IP, e.g. eth0.
${ROUTER_ID} should be the same for all keepalived cluster hosts while unique amongst all clusters in the same subnet. Many distros pre-configure its value to 51.
${PRIORITY} should be higher on the control plane node than on the backups. Hence 101 and 100 respectively will suffice.
${AUTH_PASS} should be the same for all keepalived cluster hosts, e.g. 42
${APISERVER_VIP} is the virtual IP address negotiated between the keepalived cluster hosts.

export STATE=BACKUP
export PRIORITY=100

export INTERFACE=ens192
export ROUTER_ID=51
export AUTH_PASS=42
export APISERVER_VIP=172.16.115.80

mkdir -p /etc/keepalived
cat>/etc/keepalived/keepalived.conf<<EOF
! /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state ${STATE}
    interface ${INTERFACE}
    virtual_router_id ${ROUTER_ID}
    priority ${PRIORITY}
    authentication {
        auth_type PASS
        auth_pass ${AUTH_PASS}
    }
    virtual_ipaddress {
        ${APISERVER_VIP}
    }
    track_script {
        check_apiserver
    }
}
EOF

systemctl enable keepalived
systemctl start keepalived
systemctl status keepalived

export APISERVER_VIP=172.16.115.80
export APISERVER_DEST_PORT=6443
cat>/etc/keepalived/check_apiserver.sh<<EOF
#!/bin/sh

errorExit() {
    echo "*** $*" 1>&2
    exit 1
}

curl --silent --max-time 2 --insecure https://localhost:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://localhost:${APISERVER_DEST_PORT}/"
if ip addr | grep -q ${APISERVER_VIP}; then
    curl --silent --max-time 2 --insecure https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/"
fi
EOF
```

## Kubernetes HA 初始化

> 使用kubeadm创建集群：https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
> kubeadm的HA部署： https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/high-availability/
> 高可用拓扑选项：https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/ha-topology/#stacked-etcd-topology 

官方介绍Kubernetes HA主要分为两种方式，第一种是使用具有堆叠的控制平面节点，第二种是使用外部etcd集群。第一种适用于较小规模集群，本人使用的就是第一种：

> sudo kubeadm init --control-plane-endpoint "172.16.115.80:6443" --upload-certs --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers  --service-cidr=10.1.0.0/16 --pod-network-cidr=10.244.0.0/16

初始化成功会有以下提示：

```
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.16.115.33:6443 --token 5hfvtc.6h86fglvh3miodtf \
        --discovery-token-ca-cert-hash sha256:8e194e7a345a99eeb09bacaadaaad034b91868e0551bb6c01158e30ecdbdf1f3
        
        
kubeadm join 172.16.115.33:6443 --token 5hfvtc.6h86fglvh3miodtf  --control-plane \
        --discovery-token-ca-cert-hash sha256:8e194e7a345a99eeb09bacaadaaad034b91868e0551bb6c01158e30ecdbdf1f3
```
根据提示对node进行添加。

## kubeadm command

kubeadm有很多命令可以直接使用，以下几个印象深刻。
- kubeadm token create --print-join-command  ：生成加入Kubernetes的命令

# calico安装

calico直接按照官方文档安装即可:
https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

## calico报错解决
解决了防火墙的问题，calico就不再报错了.coredns的报错在kubectl上查询不到： 应该是防火墙的问题，清空策略
```
sudo iptables -P INPUT ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -P FORWARD ACCEPT
sudo iptables -F
```

# rancher部署
> https://ranchermanager.docs.rancher.com/zh/v2.7/getting-started/quick-start-guides/deploy-rancher-manager/helm-cli


在helm pull 后，在Chart.yaml修改版本限制（我修改成了kubeVersion: < 1.28.0-0），并配置global.cattle.psp.enabled 设置为 false
```
helm install rancher ./rancher --namespace cattle-system   --set hostname=172.16.115.33.sslip.io   --set replicas=1   --set bootstrapPassword=admin --set global.cattle.psp.enabled=false
```

# kubernetes dashboard
> https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/

1. 拉取服务

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

2. 修改Service的nodePort，https对外暴露 （443的端口对外暴露）
3. 生成管理员token


```
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard
kubectl create clusterrolebinding dashboard-admin-rb --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin

cat > dashboard-admin-token.yaml<<EOF 
apiVersion: v1 
kind: Secret 
metadata: 
  name: dashboard-admin-secret 
  namespace: kubernetes-dashboard 
  annotations: 
    kubernetes.io/service-account.name: dashboard-admin 
type: kubernetes.io/service-account-token 
EOF

kubectl apply -f dashboard-admin-token.yaml

kubectl describe secret dashboard-admin-secret -n kubernetes-dashboard
```

4. 登录页面

# docker,ctr,crictl 常用命令

docker、 ctr、crictl常用命令

```
命令 docker ctr（containerd） crictl（kubernetes）
命令 docker ctr（containerd） crictl（kubernetes）
查看运行的容器 docker ps ctr task ls/ctr container ls crictl ps
查看镜像 docker images ctr image ls crictl images
查看容器日志 docker logs 无 crictl logs
查看容器数据信息 docker inspect ctr container info crictl inspect
查看容器资源 docker stats 无 crictl stats
启动/关闭已有的容器 docker start/stop ctr task start/kill crictl start/stop
运行一个新的容器 docker run ctr run 无（最小单元为pod）
修改镜像标签 docker tag ctr image tag 无
创建一个新的容器 docker create ctr container create crictl create
导入镜像 docker load ctr image import 无
导出镜像 docker save ctr image export 无
删除容器 docker rm ctr container rm crictl rm
删除镜像 docker rmi ctr image rm crictl rmi
拉取镜像 docker pull ctr image pull ctictl pull
推送镜像 docker push ctr image push 无
在容器内部执行命令 docker exec 无 crictl exec
更多命令操作，可以直接在命令行输入命令查看帮助。

```
ctr -n=k8s.io  c ls 可以看到对应的容器