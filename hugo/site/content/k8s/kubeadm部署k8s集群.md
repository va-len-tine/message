---
title: "kubeadm部署k8s集群"
date: 2022-06-02T23:25:00+08:00
draft: true


---

[官方部署文档](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)

### 一、环境准备

1. 准备两台全新的虚拟机

| 主机名 | IP             | 系统类型  | 配置 |
| ------ | -------------- | --------- | ---- |
| master | 192.168.66.100 | centos7.6 | 2c2g |
| node1  | 192.168.66.101 | centos7.6 | 4c4g |

2. 配置主机名

```bash
hostnamectl set-hostname master

[root@master ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.66.100 master
192.168.66.101 node1
```

3. 网卡配置

VMware网络适配器选择NAT模式，网卡配置为静态IP

```bash
[root@master ~]# cat /etc/sysconfig/network-scripts/ifcfg-ens33 
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens33
UUID=443eb4f6-645e-4eee-ab28-21f5cde8026f
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.66.100
NETMASK=255.255.255.0
GATEWAY=192.168.66.2
DNS1=8.8.8.8
DNS2=8.8.4.4
```

4. 配置yum源

[https://developer.aliyun.com/mirror/](https://developer.aliyun.com/mirror/)

```bash
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo
```

### 二、关闭防火墙、selinux、swap

1. 关闭防火墙

```bash
systemctl stop firewalld;systemctl disable firewalld
```

2. 关闭selinux

将SELINUX改为disabled，重启生效

```bash
[root@master ~]# cat /etc/selinux/config 

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

3. 关闭swap

注释最后一行，重启生效

```bash
[root@master ~]# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Fri Jun  3 00:34:12 2022
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=43aad901-8cc9-4a3f-a2f5-e42a78439098 /boot                   xfs     defaults        0 0
/dev/mapper/centos-home /home                   xfs     defaults        0 0
#/dev/mapper/centos-swap swap                    swap    defaults        0 0

```

### 三、配置iptables

[https://kubernetes.io/docs/setup/production-environment/container-runtimes/](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

### 四、安装docker容器运行时

[https://docs.docker.com/engine/install/centos/](https://docs.docker.com/engine/install/centos/)

[https://kubernetes.io/docs/setup/production-environment/container-runtimes/](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

1. 配置docker源

```bash
curl -o /etc/yum.repos.d/docker-ce.repo https://download.docker.com/linux/centos/docker-ce.repo
```

2. 安装docker

```bash
yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

3. 启动docker、containerd

```bash
systemctl start docker;systemctl enable docker;docker run hello-world
systemctl start containerd;systemctl enable containerd;
```

4. 配置cgroup driver

```bash
cat  >> /etc/docker/daemon.json << EOF
{
 "exec-opts":["native.cgroupdriver=systemd"]
}
EOF

systemctl restart docker
```


### 五、安装kubeadm、kubelet、kubectl

[https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

[https://www.orchome.com/10036](https://www.orchome.com/10036)

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
EOF

yum install -y kubelet kubeadm kubectl

systemctl start kubelet;systemctl enable kubelet
```

### 六、初始化集群（master）

1. 重启containerd

```bash
[root@master ~]# kubeadm init
[init] Using Kubernetes version: v1.24.1
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR CRI]: container runtime is not running: output: E0605 09:24:44.860979    7368 remote_runtime.go:925] "Status from runtime service failed" err="rpc error: code = Unimplemented desc = unknown service runtime.v1alpha2.RuntimeService"
time="2022-06-05T09:24:44+08:00" level=fatal msg="getting status of runtime: rpc error: code = Unimplemented desc = unknown service runtime.v1alpha2.RuntimeService"
, error: exit status 1
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
[root@master ~]# 
[root@master ~]# rm -rf /etc/containerd/config.toml 
[root@master ~]# systemctl restart containerd
```

2. 开启VPN，初始化集群

```bash
kubeadm init

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.66.100:6443 --token pxukys.1ptoihhkdxxdnv70 \
	--discovery-token-ca-cert-hash sha256:416230308cc2c9f98105ed161b729d9988595e4d5944e6f894293159e64b9dca 
```

3. 配置kubectl访问

```bash
mkdir ~/.kube ; cp /etc/kubernetes/admin.conf ~/.kube/config
```

4. 根据输出提示，将node节点加入集群

### 七、安装calico网络插件

[https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart](https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart)

[https://blog.csdn.net/weixin_44455125/article/details/124081178](https://blog.csdn.net/weixin_44455125/article/details/124081178)

```bash
kubectl apply -f https://projectcalico.docs.tigera.io/manifests/calico.yaml
```

### 八、去除master污点

[https://blog.csdn.net/shanghaibao123/article/details/123878175](https://blog.csdn.net/shanghaibao123/article/details/123878175)

```bash
kubectl describe node master | grep Taint
kubectl taint node master node-role.kubernetes.io/control-plane:NoSchedule-
```


### 九、配置kebectl

1. 自动补全

[https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#enable-shell-autocompletion](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#enable-shell-autocompletion)

```bash
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
alias k=kubectl
complete -F __start_kubectl k
```

2. 配置别名

```bash
alias kcd='kubectl config set-context --current --namespace '
```


3. 配置bash提示符

[https://github.com/jonmosco/kube-ps1](https://github.com/jonmosco/kube-ps1)

```bash
source /root/tool/kube-ps1/kube-ps1.sh
PS1='[\u@\h $(kube_ps1) \W]\$ '
```

### 九、安装dashboard插件

1. 安装helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

2. 安装dashboard

[https://artifacthub.io/packages/helm/k8s-dashboard/kubernetes-dashboard](https://artifacthub.io/packages/helm/k8s-dashboard/kubernetes-dashboard)

[https://github.com/kubernetes/dashboard](https://github.com/kubernetes/dashboard)

3. 修改服务类型为NodePort、绑定角色

```yaml
# 绑定集群角色：cluster-admin

# 修改服务类型
type: NodePort
ports:
- nodePort: 30000
  port: 443
  protocol: TCP
  targetPort: 8443
```

4. 创建访问token

```bash
kubectl create token kubernetes-dashboard --duration=36500h
```

5. 通过任意节点IP+Port访问

```bash
https://192.168.66.100:30000/
```

### 十、其他插件

1. 本地存储插件hostpath

[https://artifacthub.io/packages/helm/rimusz/hostpath-provisioner](https://artifacthub.io/packages/helm/rimusz/hostpath-provisioner)

2. Metrics Server

[https://artifacthub.io/packages/helm/metrics-server/metrics-server](https://artifacthub.io/packages/helm/metrics-server/metrics-server)

3. 负载均衡插件metallb

[https://artifacthub.io/packages/helm/bitnami/metallb](https://artifacthub.io/packages/helm/bitnami/metallb)

4. 资源监控Prometheus+Grafana

5. 日志采集ElasticSearch+Fluentd+Kibana