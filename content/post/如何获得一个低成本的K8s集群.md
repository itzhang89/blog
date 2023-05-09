---
title: "如何获得一个低成本的K8S集群"
date: 2023-04-30T12:28:33+08:00 
draft: true
categories:
    - devops
tags:
    - k8s
---

最近一直在玩Kubernetes（K8s），我想把之前白嫖的甲骨文主机、国内的一台腾讯云主机和家里的淘汰电脑整合到一起，组成一个K8s集群。这样，我既可以方便地使用和管理这些计算资源，又可以节省成本。

在这个场景下，有许多网络方案可供选择，我选择了通过自建headscale来建立跨区域VPN，使得这些分散的主机可以互相访问。在这种情况下，他们能够通过各自的**内网网卡IP地址实现互相的访问**。

涉及到的所有主机信息和角色分配如下：

1. 甲骨文免费的2台AMD64 主机 Ubuntu22.02TLS系统 , 1C1G配置，1 个安装 Nginx （主机名oci1）； 1 个安装Headscale的服务端（主机名oci2）。
2. 甲骨文免费的ARM64的主机主机 Ubuntu22.02TLS系统， 2C12G配置，1个作为K8s 的控制节点（主机名oci3），1个作为普通的工作节点（主机名oci4）
3. 腾讯云主机，作为工作节点（主机名tennet01），Ubuntu18.04TLS系统 ,2C4G8M带宽
4. 家里的主机作为工作节点（主机名ubuntu01），20.04.4 LTS系统，8C16G配置，500M电信家用带宽。

## 点对点的网络设置

![网络拓扑结构](https://ipic.ilovestudy.top/blog/20230330/image-20230330185830292.png)

在上述配置中，主要目的是连接三个网络，包括 Oracle Cloud 的内网网段（10.0.0.0/24）、家里内网网段（192.168.50.0/24）以及腾讯云内网网段（10.0.16.0/24），从而实现这些网段之间的地址互相访问。

### 安装Headscale控制端

需要手动安装headscale控制端。

1. 如果你的配置和我的一样，你需要先准备好 Headscale 服务的域名和证书。你可以参考我放在 Github Gist 上的脚本快速安装并配置
   Nginx，详见[安装 headscale 服务端脚本和 nginx 反向代理配置文件](https://gist.github.com/yutianaiqingtian/a26ad1ad6574e46c00ab43d88d0b7a6d)。
2. 如果你有其他的想法，可以参考官方 Github
   网站上提供的步骤 [juanfont/headscale](https://github.com/juanfont/headscale/blob/main/docs/running-headscale-container.md)，它支持使用
   Docker 容器部署，同时可以通过 Caddy 实现反向代理等多种工具。

> 需要注意的是，Headscale 是 tailscale 控制端的开源替代品。如果你不想自建 Headscale，可以考虑直接使用 tailscale 商业版本。这个版本免费用户有 20 台设备限制和一个子网网段的限制，如果你的路由需求没有超出一个子网网段，那么 tailscale 应该可以满足你的需求。例如，在一个云服务提供商的云主机上加上家里局域网内的主机，你只需要广播（advertise）家里的路由网段就可以实现连接。
>
> 需要提醒的是，使用 Headscale 作为控制端需要注意，毕竟它是开源软件，所以 Headscale 软件的 Bug 相对较多。因此，有些删除操作或重复操作可能会导致系统崩溃，所以你需要清除 Headscale 控制端的所有数据，并有准备在重装之前的心理准备。

### 安装Tailscale客户端

我这边分配的给的headscale VPN的内网地址网段为`10.0.10.0/24`。安装好headscale服务端之后 ，把需要接入Headscale的主机上安装tailscale 客户端，然后将3个内网的某一台主机配置成Subnet
router（网关的形式）。通过某一台主机代理局域网内的流量。各个内网地址的网关配置如下

1. Oracle的中，选中OCI3的主机，作为Subnet router（网关）；
2. 腾讯云和家里只有一台主机，只能把他们都作为Subnet router（网关）。

之所以这么选择是因为，所有的**作为Subnet router（网关）的角色的主机，都会广播出去自身的网卡的IP地址（OCI3 `10.0.0.3`， tencent01 `10.0.16.2`,
ubuntu01 `192.168.50.2` ）, 并且网关彼此之间能够通过自身的IP地址互相访问。**

> 理论上在一个网段下，随便选中一个主机作为网关都可以，但是在配置的过程中，甲骨文云的主机，内网之间始终没法通过作为网关的主机路由到其它网段。 比如，上面的配置中，我在oci1中，添加路由 `ip route add 192.168.50.0/24 via 10.0.10.3` ，想实现 oci1 中访问 192.168.50.2 主机的时候通过网关 oci3 访问。无论我怎么修改路由遍的优先级，都无法实现oci1 访问 `192.168.50.2` 地址的下一跳是 oci3 主机。 我通过 `ip route get 192.168.50.2` 返回的是我添加的跳转路由，但是实际上通过 `traceroute 192.168.50.2` 的时候，显示下一跳总是不是 `10.0.10.3` ，依旧从默认的路由表走出。有大神知道原因的还麻烦告知一下

3个选中作为Subnet router（网关）的角色的主机的实际配置如下：

1. 安装Tailscale客户端，其它客户端安装参考[官网]([Download · Tailscale](https://tailscale.com/download/linux))。

   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   ```

2. 配置允许IPv4 和 IPv6 的地址转发

   ```bash
   echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
   echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf
   sudo sysctl -p /etc/sysctl.conf
   ```

3. 往Headscale中控制端中注册主机信息

   ```bash
   # oci3 主机上运行
   tailscale up --login-server=https://headscale.xxx.com --accept-dns=false --accept-routes --snat-subnet-routes=false  --advertise-routes=10.0.0.0/24 --force-reauth=true
   # tencent01 主机上运行
   tailscale up --login-server=https://headscale.xxx.com --accept-dns=false --accept-routes --snat-subnet-routes=false  --advertise-routes=10.0.16.0/24 --force-reauth=true
   # ubuntu01 主机上运行
   tailscale up --login-server=https://headscale.xxx.com --accept-dns=false --accept-routes --snat-subnet-routes=false  --advertise-routes=192.168.50.0/24 --force-reauth=true
   ```

   其中 `--login-server` 中指定你的headscale服务端地址；`--accept-dns` 最好设置为false，不然会覆盖到主机上的DNS信息，不推荐；`--accept-routes`
   一定要选中，不然不会接受到其它广播的网段信息；` --advertise-routes` 表示当前主机作为Subnet router（网关），广播的路由网段信息；`--snat-subnet-routes=false`
   表示再做网关的时候，是不是开启源地址的nat 转换，建议不开启；`--force-reauth=true` 表示是不是开始往Headscale控制重新认证主机信息。

4. 步骤3中，需要在 Headscale的控制端同意，并且开启路由信息，在 headscale 控制端（oci2 主机）上

   ```bash
   # 依次注册 oci3， tencent01，ubuntu01 主机信息
   headscale nodes register --user default --key nodekey:edd5db38ab301159fxxxxxxxxxxxxxxxaa34e602fc65
   ```

   其中`--user`  表示所属的user信息，每个用户互相隔离的；也可以认为是 TELNET 的作用；nodekey 是你在第3步骤注册的生成的。

   ```bash
   # 查看注册的Node 信息
   root@oci2:/home/ubuntu# headscale nodes list
   ID | Hostname  | Name      | MachineKey | NodeKey | User | IP addresses                 | Ephemeral | Last seen           | Expiration          | Online  | Expired
   1  | oci3      | oci3      | [365vf]    | [DXRGM] | k8s  | 10.0.10.1, fd7a:115c:a1e0::1 | false     | 2023-03-30 14:06:49 | 0001-01-01 00:00:00 | online  | no
   2  | ubuntu01  | ubuntu01  | [9/kRk]    | [MYZ6Z] | k8s  | 10.0.10.2, fd7a:115c:a1e0::2 | false     | 2023-03-30 14:06:37 | 0001-01-01 00:00:00 | online  | no
   4  | tencent01 | tencent01 | [tjCa0]    | [7dXbO] | k8s  | 10.0.10.3, fd7a:115c:a1e0::3 | false     | 2023-03-30 14:07:04 | 0001-01-01 00:00:00 | online  | no
   # 查看广播的路由信息，需要所有路由信息都enabled
   root@oci2:/home/ubuntu# headscale routes list
   ID | Machine   | Prefix          | Advertised | Enabled | Primary
   1  | oci3      | 10.0.0.0/24     | true       | true    | true
   2  | ubuntu01  | 192.168.50.0/24 | true       | true    | true
   3  | tencent01 | 10.0.16.0/24    | true       | true    | true
   ```

5. 配置Subnet router（网关）的角色以将最大分段大小 (MSS) 限制在最大传输单元 (MTU) 上，其中`-o` 后的输入网卡信息，根据实际情况修改。

   ```bash
   iptables -t mangle -A FORWARD -i tailscale0 -o eth0 -p tcp -m tcp \
   --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
   ```

Tailscale 官网中，针对这一部分的原理详细说明，可以参考 [Site-to-site networking · Tailscale](https://tailscale.com/kb/1214/site-to-site/).

## K8s 的安装

### 安装K8s的主机上的基本设置

#### 1. 设置各个主机的名称，并在台主机上补充hosts信息

将每台主机的名称修改为自己方便记忆的名称，比如我这边以oci1,2,..,n 来进行编号。通过 `sudo hostnamectl set-hostname "oci1"`
这种方式来设置4台主机的hostname。然后修改每台主机的 `/etc/hosts` 文件，补充每台主机的ip地址和域名的映射关系。

```bash
# 没台主机上分别执行
$ sudo hostnamectl set-hostname "oci1"
$ sudo tee /etc/hosts <<EOF
10.0.0.1 oci1
10.0.0.2 oci2
10.0.0.3 oci3
10.0.0.4 oci4
10.0.16.2 tencent01
192.168.50.2 ubuntu01
EOF
```

#### 2. 禁用 swap 并添加内核设置

云主机默认是没有开启的，可以运行下 `sudo free -h`命令查看下，如果swap 空间为0 ，则表示没有开启；如有有开启，可以运行如下命令关闭。

```bash
$ sudo swapoff -a
$ sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

每台主机上都运行如下命令，保证开启系统加载覆盖网络和网络桥接驱动，支持Kubernetes集群中的网络配置。

```bash
$ sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
$ sudo modprobe overlay
$ sudo modprobe br_netfilter
```

开启Linux内核的IP转发功能，并开启iptables的bridge-nf功能，以便于Kubernetes集群内部的Pod可以正常通信。

```bash
$ sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

加载上诉配置，使其生效

```bash
$ sudo sysctl --system
```

#### 3. 安装 containerd Runtimes

安装支持多种 Container
Runtimes，这里选择 [containerd](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd)。
对其它的感兴趣可以查看[官网container-runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

安装一些依赖

```bash
$ sudo apt-get update
$ sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

添加和启用官方的 docker 存储库

```bash
$ sudo mkdir -m 0755 -p /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

现在，运行以下 apt 命令来安装 containerd

```bash
$ sudo apt update
$ sudo apt install -y containerd.io
```

配置 containerd 以便它开始使用 systemd 作为 cgroup。

```bash
$ containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
$ sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```

重启并启用容器服务

```bash
$ sudo systemctl restart containerd
$ sudo systemctl enable containerd
```

#### 4. 为 Kubernetes 添加 apt 存储库

执行以下命令为 Kubernetes 添加 apt 存储库

```bash
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```

> 如果是国内环境，可以参考 https://blog.csdn.net/wuyundong123/article/details/116227442

#### 5 安装 Kubernetes 组件 Kubectl、kubeadm 和 kubelet

在所有节点上安装 kubectl、kubelet 和 Kubeadm 实用程序等 Kubernetes 组件。运行以下命令集，

```
$ sudo apt update
$ sudo apt install -y kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl
```

####  

### 初始化K8s集群

现在，我们都准备好初始化 Kubernetes 集群了。仅从主节点运行以下 Kubeadm 命令。

1. 准备一个` kubeadm-config.yaml` 文件，用来初始化控制节点

   ```yaml
   apiVersion: kubeadm.k8s.io/v1beta3
   kind: InitConfiguration
   localAPIEndpoint:
     advertiseAddress: 0.0.0.0
     bindPort: 6443
   networking:
     podSubnet: 10.244.0.0/16 # 因为使用 flannl 网络插件默认使用这个网段的
   ---
   apiVersion: kubeadm.k8s.io/v1beta3
   kind: ClusterConfiguration
   apiServer:
     certSANs:
       - "k8s.xxx.com" # 给控制节点申请的域名，如果没有则通过IP连接
       - "控制节点公网IP地址"
       - "10.0.10.1" # tailscale 注册分配的VPN内的IP地址（可选）
       - "10.0.0.3" # 内网IP地址
   controlPlaneEndpoint: "k8s.ilovestudy.top:6443"
   networking:
     podSubnet: 10.244.0.0/16
   ```

   通过在控制节点中，运行 `kubeadm init --config ./kubeadm-config.yaml`  来初始化，初始化成功之后，控制台会打印出工作节点加入集群的语句，保存下来依次往节点中执行就可以。

2. 在工作节点中运行如下命令来添加到K8s集群中

   ```bash
   kubeadm join k8s.xxx.com:6443 --token s6o14o.xxxxxxxxx   --discovery-token-ca-cert-hash sha256:339ef40611b32ccc5b7238b33a2e9a1811e6542388fb1adxxxxxxxxxxd
   ```

3. 应用flannel 网络插件

   ```bash
   kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
   ```

4. 下载和安装cni插件，这里面需要根据服务的架构，下载相应的版本，比如这里就是arm64的版本。

   ```
   mkdir -p /opt/cni/bin
   curl -O -L https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-arm64-v1.2.0.tgz
   tar -C /opt/cni/bin -xzf cni-plugins-linux-arm64-v1.2.0.tgz
   ```

5. 如果想资源紧张，想让控制节点页参与任务调度，运行如下命令，去除污点

   ```bash
   kubectl taint nodes oci3  node-role.kubernetes.io/control-plane:NoSchedule-
   ```

## 其它你可能需要的操作

### 如何重置一下K8s集群

```bash
# 删除节点
kuebctl delete node node1 node2...
# 重置节点上运行
sudo kubeadm reset -f
# 删除 kubelet 状态：
sudo rm -rf /var/lib/kubelet/*
# 删除 cni 状态：
sudo rm -rf /var/lib/cni/*
sudo rm -rf /etc/cni/*
# 删除 kubeconfig 文件：
sudo rm -rf $HOME/.kube/config
# 重新设置iptables， 在执行iptables -F，一定确保iptables -P INPUT ACCEPT，否则可能会导致无法重连服务器
apt-get install iptables-persistent -y
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
iptables -F
```

### 如何重新安装flannel

```bash
#第一步，在master节点删除flannel
kubectl delete -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

#第二步，在node节点清理flannel网络留下的文件
ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1
rm -rf /var/lib/cni/
rm -f /etc/cni/net.d/*
```

### 国内环境如何安装K8S

由于国内的网络环境，导致有些镜像或者安装包的下载会非常的慢，甚至是无法下载。 最简单的方法当然就是找一个能访问外网的代理；通过代理来安装所需要的软件和镜像。没有代理的话，可以通过下面的方式来加速安装。

1. 通过国内源下载安装 kubelet, kubeadm, kubectl 三件套

   ```bash
   # 添加并信任APT证书
   curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
   # 添加源地址
   add-apt-repository "deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main"
   # 更新源并安装最新版 kubenetes
   sudo apt update && apt install -y kubelet kubeadm kubectl
   ```

2. 添加国内源来下载containerd

   ```bash
   # 添加GPG密钥
   curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | apt-key add -
   # 添加repo信息
   sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
   # 更新并且安装
   sudo apt update && sudo apt install -y containerd.io
   ```

3. 拉取镜像的时候，通过国内源获取，然后rename
   镜像名称。参考[国内拉取google kubernetes镜像](https://gist.github.com/qwfys/aec4d2ab79281aeafebdb40b22d0b748)。还有一种方式是在运行 `kubadm init`
   的时候，指定国内的`--image-repository`,
   比如 `kubeadm init --image-repository="registry.aliyuncs.com/google_containers" --config kubeadm-config.yaml`
   指定国内阿里源。这种方式有些时候镜像不会是最新的。

4. 如果有代理的话，设置好代理后，可以通过如下的命令手动拉取镜像。 这里需要注意，就是最新版的K8s集群，需要需要指定 ` -n k8s.io ` 的namespace，不然的话。拉取会放在默认的default
   namespace中，拉取了，还是没有正常使用。

   ```
   # 查看需要哪些镜像
   kubeadm config images list
   # 根据这些镜像依次拉取镜像
   ctr -n k8s.io images pull registry.k8s.io/kube-proxy:v1.26.0
   ```

### 如何更新你的部分CNI插件和 CNI配置文件

当我们需要对某个节点进行更新或者维护的时候，可以运行如下命令，来让这个节点上的一些Pod 驱逐出去，重新调度到另外的可用节点上。

```bash
kubectl drain --ignore-daemonsets <node name>
```

维护结束后可以运行如下命令，重新添加

```bash
kubectl uncordon <node name>
```

### 如何重新获得加入节点的Token信息

如果一段时间过后，想加入新的节点，可以运行如下的操作

```bash
# 在控制节点中检查下，是不是还有token有效
kubeadm token list | awk '/The default bootstrap token/ { print $1; }'
# 没有话运行如下命令生成一个
kubeadm token create
# 通过下面的命令或者证书的hash值
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
# 根据生成的token和证书hash值组合加入集群的命令
kubeadm join --skip-preflight-checks --token {{token}} --discovery-token-ca-cert-hash sha256:{{hash}} master_ip:6443
```

## 总结

上面这种情况下，搭建的K8s，在国内的访问速度去测试约100ms左右，还是相当不错的存在。其中值得注意就是，有2个arm架构的节点，使用一些镜像的时候，一定要注意是否支持arm 架构，如果不支持的话，需要单独指定调度到AMD 的主机上。

最后放一下K8s中节点连接的截图

```bash
root@oci3:/home/ubuntu# kgno -o wide
NAME        STATUS     ROLES           AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
oci3        Ready      control-plane   31h   v1.26.3   10.0.0.3         <none>        Ubuntu 22.04.1 LTS   5.15.0-1030-oracle   containerd://1.6.18
oci4        Ready      <none>          24h   v1.26.3   10.0.0.4         <none>        Ubuntu 22.04.1 LTS   5.15.0-1029-oracle   containerd://1.6.18
tencent01   Ready      <none>          31h   v1.26.3   10.0.16.2        <none>        Ubuntu 18.04.6 LTS   4.15.0-159-generic   containerd://1.6.18
ubuntu01    NotReady   <none>          25h   v1.26.0   192.168.50.2     <none>        Ubuntu 20.04.4 LTS   5.15.0-46-generic    containerd://1.6.8
```

## 参考资料

---

1.
官方的K8s安装教程 [Creating a cluster with kubeadm | Kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
2. [kubeadm 部署的 k8s 增加 ip 并重新生成证书-阿里云开发者社区](https://developer.aliyun.com/article/1094847)
3. [ssl - How can I add an additional IP / hostname to my Kubernetes certificate? - DevOps Stack Exchange](https://devops.stackexchange.com/questions/9483/how-can-i-add-an-additional-ip-hostname-to-my-kubernetes-certificate)
