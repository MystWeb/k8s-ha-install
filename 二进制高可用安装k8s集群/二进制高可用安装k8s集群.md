# 二进制高可用安装k8s集群

视频课程地址：https://ke.qq.com/course/2738602、https://edu.51cto.com/course/23845.html

# 第一章  安装前必读

本文档适用于k8s 1.24+

请不要使用带中文的服务器和克隆的虚拟机

生产环境建议使用二进制安装方式

请将该文档复制一份，然后进行更改安装，并记录每一个步骤的返回信息，有问题可以直接发送部署文档进行问答，解决更加迅速

k8s-master、etcd集群一定要采用奇数，不要采用偶数的配置，否则容易产生脑裂问题

## 1.1  生产环境可部署Kubernetes集群的两种方式

- kubeadm

Kubeadm是一个K8s部署工具，提供kubeadm init和kubeadm join，用于快速部署Kubernetes集群。

官方地址：https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/

- 二进制包

从github下载发行版的二进制包，手动部署每个组件，组成Kubernetes集群。

Kubeadm降低部署门槛，但屏蔽了很多细节，遇到问题很难排查。如果想更容易可控，而且Pod部署到达一定程度会遇到瓶颈，官方还建议每年需要更新一次Kubeadm。

推荐使用二进制包部署Kubernetes集群，虽然手动部署麻烦点，期间可以学习很多工作原理，也利于后期维护。

## 1.2  安装要求

在开始之前，部署Kubernetes集群机器需要满足以下几个条件：

- 一台或多台机器，操作系统 CentOS7.x-86_x64
- 硬件配置：2GB或更多RAM，2个CPU或更多CPU，硬盘30GB或更多
- 集群中所有机器之间网络互通
- 可以访问外网，需要拉取镜像，如果服务器不能上网，需要提前下载镜像并导入节点
- 禁止swap分区

# 第二章  安装说明

本文章将采用CentOS 7.9二进制方式安装高可用k8s 1.24+，其中Master节点3台，Node节点2台，高可用工具采用HAProxy + KeepAlived，相对于其他版本，二进制安装方式并无太大区别，只需要区分每个组件版本的对应关系即可。

生产环境中，建议使用小版本大于5的Kubernetes版本，比如1.23.5以后的才可用于生产环境。

# 第三章  集群安装

## 3.1  基本环境配置

K8S官网：https://kubernetes.io/docs/setup/

主机信息，服务器IP地址不能设置成dhcp，要配置成静态IP。

VIP（虚拟IP）不要和公司内网IP重复，首先去ping一下，不通才可用。VIP需要和主机在同一个局域网内！

公有云的话，VIP为公有云的负载均衡的IP，比如阿里云的SLB地址，腾讯云的ELB地址，注意公有云的负载均衡都是内网的负载均衡。

表1-1  高可用Kubernetes集群规划

> 注意：不要将k8s-master01的主机名设置为k8s-master-lb

|      主机名       |        IP地址         |       角色        |     配置      |
| :---------------: | :-------------------: | :---------------: | :-----------: |
| k8s-master01 ~ 03 | 192.168.100.151 ~ 153 | Master/Worker节点 |   2C2G 40G    |
|  k8s-node01 ~ 02  | 192.168.100.154 ~ 155 |    Worker节点     |   2C2G 40G    |
|   k8s-master-lb   |    192.168.100.160    |        VIP        | VIP不占用机器 |

> 注意：请统一替换这些网段，Pod网段和service和宿主机网段不要重复！！！

| 主机名        | IP地址                | 说明             | 组件                                                         |
| ------------- | --------------------- | ---------------- | ------------------------------------------------------------ |
| k8s-master01  | 192.168.100.151       | master节点 * 3   | docker，containerd，etcd，keepalived，haproxy，kube-apiserver，kube-controller-manager，kube-proxy，kube-scheduler，calico，coredns，metrics-server，kubernetes-dashboard |
| k8s-master02  | 192.168.100.152       |                  | docker，containerd，etcd，keepalived，haproxy，kube-apiserver，kube-controller-manager，kube-proxy，kube-scheduler，calico，coredns，metrics-server，kubernetes-dashboard |
| k8s-master03  | 192.168.100.153       |                  | docker，containerd，etcd，keepalived，haproxy，kube-apiserver，kube-controller-manager，kube-proxy，kube-scheduler，calico，coredns，metrics-server，kubernetes-dashboard |
| k8s-master-lb | 192.168.100.160 (VIP) | keepalived虚拟IP | VIP不占用机器                                                |
| k8s-node01    | 192.168.100.154       | worker节点 * 2   | docker，containerd，kubelet，kube-proxy                      |
| k8s-node02    | 192.168.100.155       |                  | docker，containerd，kubelet，kube-proxy                      |

> 注意：宿主机网段、K8s Service网段、Pod网段不能重复，具体看课程资料的【安装前必看】集群安装网段划分
>
> 在线IP地址/子网掩码计算与转换工具：http://tools.jb51.net/aideddesign/ip_net_calc/
>
> Pod、Service网段全局替换时，建议只替换前三段：10.96.0，安装过程中检查是否有疏漏

| 配置信息                            | 备注                                     |
| ----------------------------------- | ---------------------------------------- |
| 系统版本                            | CentOS 7.9（内核：5.15.82）              |
| Docker                              | 20.10.x                                  |
| Kubernetes                          | 1.25.5                                   |
| Pod网段                             | 172.16.0.0/12                            |
| Service网段                         | 10.96.0.0/12                             |
| 宿主机网段：inet 192.168.100.151/24 | 注意Pod、Service网段不要与宿主机网段重复 |

虚拟机环境：

![img](二进制高可用安装k8s集群.assets/wps1.jpg) 

![img](二进制高可用安装k8s集群.assets/wps2.jpg) 

## 3.2 系统基础配置

```sh
设置静态IP：
# 备份网卡配置（备份的网卡需设置：ONBOOT="yes"）
cp -p /etc/sysconfig/network-scripts/ifcfg-ens32 /etc/sysconfig/network-scripts/ifcfg-ens32.bak$(date '+%Y%m%d%H%M%S')
# 编辑网卡配置
vi /etc/sysconfig/network-scripts/ifcfg-ens32
# ————————start————————
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static" # 设置为静态IP 默认dhcp -修改
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens32"
# Version 1 UUID生成：https://www.uuidgenerator.net
UUID="7d747022-728e-40b4-9ffa-c955327e2522" # UUID -修改
DEVICE="ens32"
ONBOOT="yes"
IPADDR="192.168.100.151" # IP地址 -增加
GATEWAY="192.168.100.1" # 网关 -增加
DNS1="192.168.100.1" # DNS服务器1 -增加（如需局域网服务器，注释该选项）
DNS2="114.114.114.114" # DNS服务器2 -增加（如需局域网服务器，注释该选项）
# ————————end————————

# CentOS Linux 7 重启网络
service network restart

#重启网卡(下面的三条命令都可以)
nmcli c up ens32
nmcli d reapply ens32
nmcli d connect ens32

CentOS 7 安装yum源：
cp -p /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak$(date '+%Y%m%d%H%M%S')
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo

必备工具安装：
yum install -y epel-release yum-utils device-mapper-persistent-data lvm2

# docker源
sudo yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo \
    --add-repo \
    https://nvidia.github.io/nvidia-docker/centos7/nvidia-docker.repo

# Add End Point Package Repository（CentOS 7上安装新版本 Git 最快的方法是通过End Point库）
# 其他版本的 CentOS 替换成对应版本安装，可参考：https://packages.endpointdev.com
yum install https://packages.endpointdev.com/rhel/7/os/x86_64/endpoint-repo.x86_64.rpm

yum clean all && yum -y makecache && yum repolist
yum -y update && yum -y upgrade && reboot

常用工具安装：
yum -y install wget jq psmisc vim net-tools telnet yum-utils \
  device-mapper-persistent-data lvm2 \
  git lrzsz unzip zip tree sysstat pciutils \
  nginx gcc kernel-devel dkms nmon ansible \
  expect lsof screen

所有节点关闭防火墙、selinux、dnsmasq、swap。服务器配置如下：
systemctl disable --now firewalld # 关闭防火墙
systemctl disable --now dnsmasq # 关闭dnsmasq
systemctl disable --now NetworkManager # 关闭NetworkManager

关闭selinux：
cp -p /etc/selinux/config /etc/selinux/config.bak$(date '+%Y%m%d%H%M%S')
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config # 永久
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/sysconfig/selinux # 永久
setenforce 0  # 临时

关闭swap：
cp -p /etc/fstab /etc/fstab.bak$(date '+%Y%m%d%H%M%S')
swapoff -a && sysctl -w vm.swappiness=0 # 临时
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab # 永久

根据规划设置主机名：
hostnamectl set-hostname <hostname>

所有节点配置hosts，修改/etc/hosts如下：
cp -p /etc/hosts /etc/hosts.bak$(date '+%Y%m%d%H%M%S')
vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.100.151 k8s-master01
192.168.100.152 k8s-master02
192.168.100.153 k8s-master03
192.168.100.160 k8s-master-lb # 如果不是高可用集群，该IP为Master01的IP
192.168.100.154 k8s-node01
192.168.100.155 k8s-node02

所有节点同步时间。时间同步配置如下：
rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm && yum install ntpdate -y
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo 'Asia/Shanghai' >/etc/timezone
ntpdate time2.aliyun.com
加入到crontab
crontab -e
*/5 * * * * /usr/sbin/ntpdate time2.aliyun.com

所有节点配置limit：
ulimit -SHn 65535

vim /etc/security/limits.conf
# 末尾添加如下内容（粘贴后检查前面是否有#）
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited

基本配置时间+用户+IP设置：
vi /etc/profile

# 基本配置时间+用户+ip设置
HISTFILESIZE=2000 # 默认保存命令条数
HISTSIZE=2000	   # 使用命令时输出的记录数
IP=`who -u am i 2>/dev/null| awk '{print $NF}'|sed -e 's/[()]//g'` # 获取客户端IP
if [ -z $IP ] # IP长度为零时则赋值本机主机名
then
IP=`hostname`
fi
HISTTIMEFORMAT="%F %T $IP:`whoami` " # 设置history输出格式
export HISTTIMEFORMAT

source /etc/profile
```

所有节点升级系统并重启，此处升级没有升级内核，下节会单独升级内核：

```bash
yum update -y --exclude=kernel* && reboot #CentOS7需要升级，CentOS8可以按需升级系统

# 清除history历史记录（备份虚拟机）
echo > /root/.bash_history && history -r && poweroff
```

Master01节点免密钥登录其他节点，安装过程中生成配置文件和证书均在Master01上操作，集群管理也在Master01上操作，阿里云或者AWS上需要单独一台kubectl服务器。密钥配置如下：

```bash
[root@k8s-master01 ~] ssh-keygen -t rsa
[root@k8s-master01 ~] for i in k8s-master01 k8s-master02 k8s-master03 k8s-node01 k8s-node02;do ssh-copy-id -i .ssh/id_rsa.pub $i;done

Master01下载安装文件：
[root@k8s-master01 ~]# cd /root/ ; git clone https://github.com/MystWeb/k8s-ha-install.git
```

## 3.3  内核升级配置

CentOS7 需要升级内核至4.18+，本地升级的版本为5.15.82

在master01节点下载内核：

- Linux内核档案：https://kernel.org/

- 参考链接：https://teddysun.com/648.html
- Linux内核仓库：https://dl.lamp.sh/kernel/el7/

```sh
# 所有节点创建内核升级文件存放目录
mkdir -p /opt/installation_package/kernel && cd /opt/installation_package/kernel
# Docker官方经过测试-LTS内核版本：4.19
wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm
wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm
# 最新稳定内核版本：6.0.10
wget https://elrepo.org/linux/kernel/el7/x86_64/RPMS/kernel-ml-devel-6.0.10-1.el7.elrepo.x86_64.rpm
wget https://elrepo.org/linux/kernel/el7/x86_64/RPMS/kernel-ml-6.0.10-1.el7.elrepo.x86_64.rpm
# 最新LTS内核版本：5.15.82
wget https://dl.lamp.sh/kernel/el7/kernel-ml-devel-5.15.82-1.el7.x86_64.rpm
wget https://dl.lamp.sh/kernel/el7/kernel-ml-5.15.82-1.el7.x86_64.rpm
```

从master01节点传到其他节点：

```sh
[root@k8s-master01 kernel]# for i in k8s-master02 k8s-master03 k8s-node01 k8s-node02;do scp kernel-ml*.rpm $i:/opt/installation_package/kernel ; done
```

所有节点安装内核

```sh
cd /opt/installation_package/kernel && yum -y localinstall kernel-ml*
```

成功安装后，执行以下命令查看本机已经存在的内核列表：

```bash
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /boot/grub2/grub.cfg
```

```bash
0 : CentOS Linux (5.15.82-1.el7.x86_64) 7 (Core)
1 : CentOS Linux (3.10.0-1160.80.1.el7.x86_64) 7 (Core)
2 : CentOS Linux (3.10.0-1160.el7.x86_64) 7 (Core)
3 : CentOS Linux (0-rescue-f2c0680f2af64e878529fece3b92e36f) 7 (Core)
```

所有节点更改内核启动顺序

```sh
grub2-set-default 0 && grub2-mkconfig -o /etc/grub2.cfg
grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"
```

检查默认内核是不是5.15.82

```sh
grubby --default-kernel
```

```bash
/boot/vmlinuz-5.15.82-1.el7.x86_64
```

所有节点重启，然后检查内核是不是5.15.82

```sh
reboot
uname -a && rm -rf /opt/installation_package/kernel
```

```bash
Linux k8s-master01 5.15.82-1.el7.x86_64 #1 SMP Thu Dec 8 13:49:20 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
```

所有节点安装ipvsadm：

> ipvsadm工具为方便查看k8s节点中ipvs的配置，路由配置、转发配置等。

```sh
yum -y install ipvsadm ipset sysstat conntrack libseccomp
```

所有节点配置ipvs模块，在内核4.19+版本`nf_conntrack_ipv4`已经改为`nf_conntrack`， 4.18以下使用`nf_conntrack_ipv4`即可：

```ini
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
# 内核4.19+版本nf_conntrack_ipv4改为nf_conntrack
modprobe -- nf_conntrack
```

```sh
vim /etc/modules-load.d/ipvs.conf
```

```ini
ip_vs
ip_vs_lc
ip_vs_wlc
ip_vs_rr
ip_vs_wrr
ip_vs_lblc
ip_vs_lblcr
ip_vs_dh
ip_vs_sh
ip_vs_fo
ip_vs_nq
ip_vs_sed
ip_vs_ftp
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
```

```bash
systemctl enable --now systemd-modules-load.service
```

检查是否加载：

```sh
lsmod | grep -e ip_vs -e nf_conntrack
```

```bash
ip_vs_ftp              16384  0 
nf_nat                 49152  1 ip_vs_ftp
ip_vs_sed              16384  0 
ip_vs_nq               16384  0 
ip_vs_fo               16384  0 
ip_vs_sh               16384  0 
ip_vs_dh               16384  0 
ip_vs_lblcr            16384  0 
ip_vs_lblc             16384  0 
ip_vs_wrr              16384  0 
ip_vs_rr               16384  0 
ip_vs_wlc              16384  0 
ip_vs_lc               16384  0 
ip_vs                 159744  24 ip_vs_wlc,ip_vs_rr,ip_vs_dh,ip_vs_lblcr,ip_vs_sh,ip_vs_fo,ip_vs_nq,ip_vs_lblc,ip_vs_wrr,ip_vs_lc,ip_vs_sed,ip_vs_ftp
nf_conntrack          159744  2 nf_nat,ip_vs
nf_defrag_ipv6         24576  2 nf_conntrack,ip_vs
nf_defrag_ipv4         16384  1 nf_conntrack
libcrc32c              16384  4 nf_conntrack,nf_nat,xfs,ip_vs
```

开启一些k8s集群中必须的内核参数，所有节点配置k8s内核：

> 注意：EOF创建配置文件时，配置文件内容的特殊符号：$，前面需要添加转义符：\

```sh
cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720

net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
EOF
```

```bash
sysctl --system
```

所有节点配置完内核后，重启服务器，保证重启后内核依旧加载

```sh
reboot
lsmod | grep --color=auto -e ip_vs -e nf_conntrack
```

# 第四章  K8s组件&Runtime安装

本节主要安装的是集群中用到的各种组件，比如Docker-CE、Kubernetes各组件等。

## 4.1 Docker、Containerd安装

所有节点安装Docker、Containerd

> 安装Docker时，会关联安装Containerd，后续可能会用到Docker构建镜像，但是推荐以Containerd为主！

```sh
# 方式一：安装最新版本的Docker Engine和容器（推荐）
sudo yum -y install docker-ce docker-ce-cli containerd.io nvidia-container-toolkit nvidia-docker2 bash-completion

# 方式二：安装特定版本的Docker Engine
	# a.列出并排序您存储库中可用的版本，然后选择并安装。此示例按版本号（从高到低）对结果进行排序
	yum list docker-ce --showduplicates | sort -r
	# b.通过其完全合格的软件包名称安装特定版本，该软件包名称是软件包名称（docker-ce）加上版本字符串（第二列），从第一个冒号（:）一直到第一个连字符，并用连字符（-）分隔。例如，docker-ce-18.09.1
sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
# 注意：Docker已安装但尚未启动。docker创建该组，但没有用户添加到该组。
```

### 4.1.1 Docker配置

```sh
#cat <<EOF>> /etc/docker/daemon.json
sudo cat > /etc/docker/daemon.json <<EOF
{
    "bip": "172.17.0.1/24",
    "data-root": "/var/lib/docker",
    "storage-driver": "overlay2",
    "insecure-registries":["http://192.168.100.150:8082"],
    "exec-opts": ["native.cgroupdriver=systemd"],
    "registry-mirrors": [
        "https://docker.nju.edu.cn",
        "https://hub-mirror.c.163.com",
        "https://registry.cn-hangzhou.aliyuncs.com"
    ],
    "max-concurrent-downloads": 10,
    "max-concurrent-uploads": 20,
    "live-restore": true,
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "500m",
        "max-file": "3"
    },
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
EOF
```

> 通常情况下无需启动Docker，只需要配置和启动Containerd即可。

```bash
systemctl daemon-reload && systemctl enable --now docker
```

## 4.2  Containerd作为Runtime（K8s版本≥1.24）

首先配置Containerd所需的模块（所有节点）：

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

所有节点加载模块：

```bash
modprobe -- overlay && modprobe -- br_netfilter
```

所有节点，配置Containerd所需的内核：

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

所有节点加载内核：

```bash
sysctl --system
```

所有节点配置Containerd的配置文件：

```bash
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
```

所有节点将Containerd的Cgroup改为Systemd：

```bash
vim /etc/containerd/config.toml
```

找到`containerd.runtimes.runc.options`，添加`SystemdCgroup = true`**（如果已存在直接修改，否则会报错）**，如下图所示：

![image-20221125162637293](二进制高可用安装k8s集群.assets/image-20221125162637293.png)

所有节点将`sandbox_image`的Pause镜像改成符合自己版本的地址：

registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9

参考链接：https://github.com/AliyunContainerService/k8s-for-docker-desktop/blob/master/images.properties

![image-20221214135136497](二进制高可用安装k8s集群.assets/image-20221214135136497.png)

所有节点启动Containerd，并配置开机自启动：

```bash
systemctl daemon-reload && systemctl enable --now containerd
```

所有节点配置crictl客户端连接的运行时（Runtime）位置：

```bash
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

### 4.2.1  Containerd配置

> containerd 实现了 kubernetes 的 Container Runtime Interface (CRI) 接口，提供容器运行时核心功能，如镜像管理、容器管理等，相比 dockerd 更加简单、健壮和可移植。
>
> 从docker过度还是需要一点时间慢慢习惯的，今天来探讨containerd 如何从无域名与权威证书的私有仓库harbor，下载镜像！
>
> containerd 不能像docker一样 `docker login harbor.example.com` 登录到镜像仓库,无法从harbor拉取到镜像。

修改Containerd配置文件

```bash
vim /etc/containerd/config.toml
```

- [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]：镜像仓库源地址
- endpoint = ["https://registry-1.docker.io"]：镜像仓库代理地址
- insecure_skip_verify = true：是否跳过安全认证
- [plugins."io.containerd.grpc.v1.cri".registry.configs."192.168.100.150:8082".auth]：私有镜像仓库授权认证
  - 配置私有镜像仓库账号密码后，k8s Pod拉取镜像无需创建Secrets，Deployment也无需配置Secrets

- 配置文件参考：https://github.com/containerd/containerd/blob/main/docs/cri/registry.md

```toml
    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = ""

      [plugins."io.containerd.grpc.v1.cri".registry.auths]

      [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."192.168.100.150:8082".tls]
          insecure_skip_verify = true  # 是否跳过安全认证
        [plugins."io.containerd.grpc.v1.cri".registry.configs."192.168.100.150:8082".auth]
          username = "admin"
          password = "proaim@2013"
      [plugins."io.containerd.grpc.v1.cri".registry.headers]

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://registry-1.docker.io"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."192.168.100.150:8082"]
          endpoint = ["http://192.168.100.150:8082"]
```

拉取和查看镜像

```bash
ctr -n k8s.io image pull 192.168.100.150:8082/proaim/proaim-trinity-service:RELEASE-1.2.0-fc67c4d5 --plain-http --user admin:Harbor12345
ctr -n k8s.io image ls
```

## 4.3  Docker-Composer安装

Linux 上我们可以从 Github 上下载它的二进制包来使用，最新发行的版本地址：https://github.com/docker/compose/releases。

运行以下命令以下载 Docker Compose 的当前稳定版本：

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/v2.14.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

Docker Compose 存放在 GitHub，不太稳定。

你可以也通过执行下面的命令，高速安装 Docker Compose。

```bash
curl -L https://get.daocloud.io/docker/compose/releases/download/v2.14.2/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
```

将可执行权限应用于二进制文件并创建软链：

```bash
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

测试是否安装成功

```bash
docker-compose version
Docker Compose version v2.14.2
```

## 4.4  K8s及etcd安装

Master01下载kubernetes安装包

```sh
[root@k8s-master01 ~]# mkdir -p /opt/installation_package/{kubernetes,etcd} && cd /opt/installation_package/kubernetes && wget https://dl.k8s.io/v1.25.5/kubernetes-server-linux-amd64.tar.gz
```

注意目前版本是1.25.5，安装时需要下载最新的1.25.x版本：https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.25.md

打开页面后点击：

![image-20221214095650385](二进制高可用安装k8s集群.assets/image-20221214095650385.png)

**以下操作都在master01执行**

下载etcd安装包

```sh
[root@k8s-master01 ~]# cd /opt/installation_package/etcd && wget https://github.com/etcd-io/etcd/releases/download/v3.5.6/etcd-v3.5.6-linux-amd64.tar.gz
```

解压kubernetes安装文件

```sh
[root@k8s-master01 ~]# cd /opt/installation_package/kubernetes && tar -xf kubernetes-server-linux-amd64.tar.gz  --strip-components=3 -C /usr/local/bin kubernetes/server/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy}
```

解压etcd安装文件

```sh
[root@k8s-master01 ~]# cd /opt/installation_package/etcd && tar -zxvf etcd-v3.5.6-linux-amd64.tar.gz --strip-components=1 -C /usr/local/bin etcd-v3.5.6-linux-amd64/etcd{,ctl}
```

版本查看

```sh
[root@k8s-master01 ~]# kubelet --version
Kubernetes v1.25.5
[root@k8s-master01 ~]# etcdctl version
etcdctl version: 3.5.6
API version: 3.5
```

将组件发送到其他节点

```sh
MasterNodes='k8s-master02 k8s-master03'
WorkNodes='k8s-node01 k8s-node02'
for NODE in $MasterNodes; do echo $NODE; scp /usr/local/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy} $NODE:/usr/local/bin/; scp /usr/local/bin/etcd* $NODE:/usr/local/bin/; done
for NODE in $WorkNodes; do scp /usr/local/bin/kube{let,-proxy} $NODE:/usr/local/bin/; done
```

所有节点创建/opt/cni/bin目录

```sh
mkdir -p /opt/cni/bin
```

Master01节点切换到1.25.x分支（其他版本可以切换到其他分支，.x即可，不需要更改为具体的小版本）

```sh
cd /opt/installation_package && git clone https://github.com/MystWeb/k8s-ha-install.git
cd /opt/installation_package/k8s-ha-install && git checkout manual-installation-v1.25.x
```

## 4.5  命令自动补全

**安装 bash-completion**

```sh
sudo yum install -y bash-completion

安装完成之后重启系统或者重新登录 shell。如果安装成功。键入 docker p 后，再 Tab 键，系统显示如下：
pause   plugin  port    ps      pull    push
```

**Docker Composer 命令自动补全**

```sh
sudo curl -L https://raw.githubusercontent.com/docker/compose/1.27.4/contrib/completion/bash/docker-compose -o /etc/bash_completion.d/docker-compose
source /etc/bash_completion.d/docker-compose
```

**Containerd Ctr 命令自动补全**

```bash
curl -L https://raw.githubusercontent.com/containerd/containerd/main/contrib/autocomplete/ctr -o /etc/bash_completion.d/ctr # ctr自动补全
```

**K8s-Master节点 命令自动补全**

```sh
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

**Helm 命令自动补全**

```bash
helm completion bash > .helmrc && echo "source .helmrc" >> .bashrc
```

# 第五章  生成证书

二进制安装最关键步骤，一步错误全盘皆输，一定要注意每个步骤都要是正确的

> 注意：kubeadm集群入口默认端口：16443，此处为区分，使用了8443端口，如果想要使用16443或公有云是6443等其它端口，需同步修改"高可用配置"章节的关联端口

Master01下载生成证书工具（下载不成功可以通过浏览器或百度网盘下载）

GitHub直链下载

- cfssl：https://github.com/cloudflare/cfssl/releases/download/v1.6.3/cfssl_1.6.3_linux_amd64
- cfssljson：https://github.com/cloudflare/cfssl/releases/download/v1.6.3/cfssljson_1.6.3_linux_amd64

```sh
wget "https://pkg.cfssl.org/R1.2/cfssl_linux-amd64" -O /usr/local/bin/cfssl
wget "https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64" -O /usr/local/bin/cfssljson

chmod +x /usr/local/bin/cfssl*
```

## 5.1  etcd证书

所有Master节点创建etcd证书目录

```sh
mkdir /etc/etcd/ssl -p
```

所有节点创建kubernetes相关目录

```sh
mkdir -p /etc/kubernetes/pki
```

Master01节点生成etcd证书

生成证书的CSR文件：证书签名请求文件，配置了一些域名、公司、单位

```sh
[root@k8s-master01 pki]# cd /opt/installation_package/k8s-ha-install/pki

# 生成etcd CA证书和CA证书的key
cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare /etc/etcd/ssl/etcd-ca
```

注意：-hostname参数：如果etcd是单独集群，需修改k8s-master*，ip = etcd-node，etcd-ip（etcd集群节点主机名称和ip地址）

```bash
# 支持域名，预留多个IP地址方便后期扩容，而不需要重新生成证书
[root@k8s-master01 pki]# cfssl gencert \
   -ca=/etc/etcd/ssl/etcd-ca.pem \
   -ca-key=/etc/etcd/ssl/etcd-ca-key.pem \
   -config=ca-config.json \
   -hostname=127.0.0.1,k8s-master01,k8s-master02,k8s-master03,192.168.100.151,192.168.100.152,192.168.100.153 \
   -profile=kubernetes \
   etcd-csr.json | cfssljson -bare /etc/etcd/ssl/etcd
```

执行结果

```bash
2022/12/14 14:07:54 [INFO] generate received request
2022/12/14 14:07:54 [INFO] received CSR
2022/12/14 14:07:54 [INFO] generating key: rsa-2048
2022/12/14 14:07:55 [INFO] encoded CSR
2022/12/14 14:07:55 [INFO] signed certificate with serial number 578142582718517357263811745144450416168339986243
```

将证书复制到其他节点

```sh
MasterNodes='k8s-master02 k8s-master03'
#WorkNodes='k8s-node01 k8s-node02'

for NODE in $MasterNodes; do
     ssh $NODE "mkdir -p /etc/etcd/ssl"
     for FILE in etcd-ca-key.pem  etcd-ca.pem  etcd-key.pem  etcd.pem; do
       scp /etc/etcd/ssl/${FILE} $NODE:/etc/etcd/ssl/${FILE}
     done
 done
```

## 5.2  k8s组件证书

**Master01生成kubernetes证书**

> 由于k8s证书比较多，所以通过ca-csr.json生成统一的根证书

```sh
[root@k8s-master01 pki]# cd /opt/installation_package/k8s-ha-install/pki

cfssl gencert -initca ca-csr.json | cfssljson -bare /etc/kubernetes/pki/ca
```

生成apiserver的证书

> 注意：10.96.0.0是k8s service的网段，如果说需要更改k8s service网段，需要同步更改10.96.0.1
>
> 10.96.0.1：k8s service网段10.96.0.0/12，通过"网络和IP地址计算器"工具计算出的第一个可用IP
>
> 如果不是高可用集群，192.168.100.160（VIP）需改为Master01的主机IP

```sh
[root@k8s-master01 pki]# cfssl gencert -ca=/etc/kubernetes/pki/ca.pem -ca-key=/etc/kubernetes/pki/ca-key.pem -config=ca-config.json -hostname=10.96.0.1,192.168.100.160,127.0.0.1,kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.default.svc.cluster.local,192.168.100.151,192.168.100.152,192.168.100.153 -profile=kubernetes apiserver-csr.json | cfssljson -bare /etc/kubernetes/pki/apiserver
```

生成apiserver的聚合证书。

> 聚合证书限制一些比较小的权限提供第三方工具（如：Metrics Server）使用。
>
> 它会判断请求的请求头，然后再进行授权（Requestheader-client-xxx requestheader-allowwd-xxx:aggerator）
>
> 如果直接使用管理员的证书，权限过大，可能存在安全隐患！

```sh
cfssl gencert -initca front-proxy-ca-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-ca

cfssl gencert -ca=/etc/kubernetes/pki/front-proxy-ca.pem -ca-key=/etc/kubernetes/pki/front-proxy-ca-key.pem -config=ca-config.json -profile=kubernetes front-proxy-client-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-client
```

返回结果（忽略警告）

```sh
2022/12/14 14:09:07 [INFO] generate received request
2022/12/14 14:09:07 [INFO] received CSR
2022/12/14 14:09:07 [INFO] generating key: rsa-2048
2022/12/14 14:09:07 [INFO] encoded CSR
2022/12/14 14:09:07 [INFO] signed certificate with serial number 279637024862091547792911851849398285103473545833
2022/12/14 14:09:07 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

生成controller-manage的证书

> controller-manager需要连接apiserver，它的权限是由controller-manager.kubeconfig控制

```sh
cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   manager-csr.json | cfssljson -bare /etc/kubernetes/pki/controller-manager

# 注意：如果不是高可用集群，192.168.100.160:8443（VIP）需改为Master01的主机IP，8443改为apiserver的端口，默认是6443
# set-cluster：设置一个集群项
kubectl config set-cluster kubernetes \
     --certificate-authority=/etc/kubernetes/pki/ca.pem \
     --embed-certs=true \
     --server=https://192.168.100.160:8443 \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

# 设置一个环境项，一个上下文（设置一个用户）
kubectl config set-context system:kube-controller-manager@kubernetes \
    --cluster=kubernetes \
    --user=system:kube-controller-manager \
    --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

# set-credentials 设置一个用户项（设置用户用哪个证书连接kubeconfig）
# 证书文件会绑定组织/ CN（COMMON NAME）等，会把授权绑定到CN上，所以就有了对应的权限。
kubectl config set-credentials system:kube-controller-manager \
     --client-certificate=/etc/kubernetes/pki/controller-manager.pem \
     --client-key=/etc/kubernetes/pki/controller-manager-key.pem \
     --embed-certs=true \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

# 使用某个环境当做默认环境（kubeconfig支持配置多个集群）
# 每个集群都有一个用户（名称：system:kube-controller-manager）
# 每个用户都有自己用到的证书（名称对应的证书文件：/etc/kubernetes/controller-manager.kubeconfig）
kubectl config use-context system:kube-controller-manager@kubernetes \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
```

生成scheduler的证书

```bash
cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   scheduler-csr.json | cfssljson -bare /etc/kubernetes/pki/scheduler

# 注意：如果不是高可用集群，192.168.100.160:8443（VIP）需改为Master01的主机IP，8443改为apiserver的端口，默认是6443
kubectl config set-cluster kubernetes \
     --certificate-authority=/etc/kubernetes/pki/ca.pem \
     --embed-certs=true \
     --server=https://192.168.100.160:8443 \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
     --client-certificate=/etc/kubernetes/pki/scheduler.pem \
     --client-key=/etc/kubernetes/pki/scheduler-key.pem \
     --embed-certs=true \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

kubectl config set-context system:kube-scheduler@kubernetes \
     --cluster=kubernetes \
     --user=system:kube-scheduler \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

kubectl config use-context system:kube-scheduler@kubernetes \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig
```

生成管理员证书

```bash
cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   admin-csr.json | cfssljson -bare /etc/kubernetes/pki/admin

# 注意：如果不是高可用集群，192.168.100.160:8443（VIP）需改为Master01的主机IP，8443改为apiserver的端口，默认是6443
kubectl config set-cluster kubernetes --certificate-authority=/etc/kubernetes/pki/ca.pem --embed-certs=true --server=https://192.168.100.160:8443 --kubeconfig=/etc/kubernetes/admin.kubeconfig

kubectl config set-credentials kubernetes-admin --client-certificate=/etc/kubernetes/pki/admin.pem --client-key=/etc/kubernetes/pki/admin-key.pem --embed-certs=true --kubeconfig=/etc/kubernetes/admin.kubeconfig

kubectl config set-context kubernetes-admin@kubernetes --cluster=kubernetes --user=kubernetes-admin --kubeconfig=/etc/kubernetes/admin.kubeconfig

kubectl config use-context kubernetes-admin@kubernetes --kubeconfig=/etc/kubernetes/admin.kubeconfig
```

创建ServiceAccount Key -> secret

> 创建一个serviceAccount时，会生成含有token的secret，而secret的token生成是由controllerManager基于密钥对与证书生成

```sh
openssl genrsa -out /etc/kubernetes/pki/sa.key 2048
```

返回结果

```sh
Generating RSA private key, 2048 bit long modulus
...................+++
..........................+++
e is 65537 (0x10001)
```

```sh
openssl rsa -in /etc/kubernetes/pki/sa.key -pubout -out /etc/kubernetes/pki/sa.pub
```

发送证书至其他节点

```sh
for NODE in k8s-master02 k8s-master03; do
	for FILE in $(ls /etc/kubernetes/pki | grep -v etcd); do
		scp /etc/kubernetes/pki/${FILE} $NODE:/etc/kubernetes/pki/${FILE}
	done
	for FILE in admin.kubeconfig controller-manager.kubeconfig scheduler.kubeconfig; do
		scp /etc/kubernetes/${FILE} $NODE:/etc/kubernetes/${FILE}
	done
done
```

查看证书文件（共计23个证书文件）

```sh
[root@k8s-master01 pki]# ls /etc/kubernetes/pki/ && ls /etc/kubernetes/pki/ |wc -l
admin.csr          apiserver.pem           controller-manager-key.pem  front-proxy-client.csr      scheduler.csr
admin-key.pem      ca.csr                  controller-manager.pem      front-proxy-client-key.pem  scheduler-key.pem
admin.pem          ca-key.pem              front-proxy-ca.csr          front-proxy-client.pem      scheduler.pem
apiserver.csr      ca.pem                  front-proxy-ca-key.pem      sa.key
apiserver-key.pem  controller-manager.csr  front-proxy-ca.pem          sa.pub
23
```

# 第六章  Kubernetes系统组件配置

## 6.1  Etcd配置

etcd配置大致相同，注意修改每个Master节点的etcd配置的主机名和IP地址

### 6.1.1  Master01

```sh
vim /etc/etcd/etcd.config.yml
```

```sh
name: 'k8s-master01'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://192.168.100.151:2380'
listen-client-urls: 'https://192.168.100.151:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://192.168.100.151:2380'
advertise-client-urls: 'https://192.168.100.151:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'k8s-master01=https://192.168.100.151:2380,k8s-master02=https://192.168.100.152:2380,k8s-master03=https://192.168.100.153:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false
```

### 6.1.2  Master02

```sh
vim /etc/etcd/etcd.config.yml
```

```sh
name: 'k8s-master02'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://192.168.100.152:2380'
listen-client-urls: 'https://192.168.100.152:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://192.168.100.152:2380'
advertise-client-urls: 'https://192.168.100.152:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'k8s-master01=https://192.168.100.151:2380,k8s-master02=https://192.168.100.152:2380,k8s-master03=https://192.168.100.153:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false
```

### 6.1.3  Master03

```sh
vim /etc/etcd/etcd.config.yml
```

```sh
name: 'k8s-master03'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://192.168.100.153:2380'
listen-client-urls: 'https://192.168.100.153:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://192.168.100.153:2380'
advertise-client-urls: 'https://192.168.100.153:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'k8s-master01=https://192.168.100.151:2380,k8s-master02=https://192.168.100.152:2380,k8s-master03=https://192.168.100.153:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false
```

### 6.1.4  创建Service

所有Master节点创建etcd的service启动文件

```sh
vim /usr/lib/systemd/system/etcd.service
```

```sh
[Unit]
Description=Etcd Service
Documentation=https://coreos.com/etcd/docs/latest/
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd --config-file=/etc/etcd/etcd.config.yml
Restart=on-failure
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
Alias=etcd3.service
```

所有Master节点创建etcd的证书目录

```sh
mkdir /etc/kubernetes/pki/etcd && ln -s /etc/etcd/ssl/* /etc/kubernetes/pki/etcd/
systemctl daemon-reload && systemctl enable --now etcd
```

查看etcd启动日志

```sh
systemctl status etcd && tail -f /var/log/messages
```

查看etcd状态

```sh
export ETCDCTL_API=3
etcdctl --endpoints="192.168.100.153:2379,192.168.100.152:2379,192.168.100.151:2379" --cacert=/etc/kubernetes/pki/etcd/etcd-ca.pem --cert=/etc/kubernetes/pki/etcd/etcd.pem --key=/etc/kubernetes/pki/etcd/etcd-key.pem  endpoint status --write-out=table
```

![image-20221128194217209](二进制高可用安装k8s集群.assets/image-20221128194217209.png)

# 第七章  高可用配置

> 高可用配置（注意：如果不是高可用集群，haproxy和keepalived无需安装）
>
> 如果在云上安装也无需执行此章节的步骤，可以直接使用云上的lb，比如阿里云slb，腾讯云elb等
>
> 公有云要用公有云自带的负载均衡，比如阿里云的SLB，腾讯云的ELB，用来替代haproxy和keepalived，因为公有云大部分都是不支持keepalived的，另外如果用阿里云的话，kubectl控制端不能放在master节点，推荐使用腾讯云，因为阿里云的slb有回环的问题，也就是slb代理的服务器不能反向访问SLB，但是腾讯云修复了这个问题。
>
> 注意：kubeadm集群入口默认端口：16443，此处为区分，使用了8443端口，如果想要使用16443或公有云是6443等其它端口，需同步修改"生成证书"章节的关联端口

Slb -> haproxy -> apiserver

所有Master节点安装keepalived和haproxy

```sh
yum -y install keepalived haproxy
```

所有Master配置HAProxy，配置一样

```sh
cp -p /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak$(date '+%Y%m%d%H%M%S')
vim /etc/haproxy/haproxy.cfg
```

```sh
global
  maxconn  2000
  ulimit-n  16384
  log  127.0.0.1 local0 err
  stats timeout 30s

defaults
  log global
  mode  http
  option  httplog
  timeout connect 5000
  timeout client  50000
  timeout server  50000
  timeout http-request 15s
  timeout http-keep-alive 15s

frontend k8s-master
  bind 0.0.0.0:8443
  bind 127.0.0.1:8443
  mode tcp
  option tcplog
  tcp-request inspect-delay 5s
  default_backend k8s-master

backend k8s-master
  mode tcp
  option tcplog
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  server k8s-master01    192.168.100.151:6443  check
  server k8s-master02    192.168.100.152:6443  check
  server k8s-master03    192.168.100.153:6443  check
```

## 7.1  Master01 keepalived

所有Master节点配置KeepAlived，配置不一样，注意区分 [root@k8s-master01 pki]# vim /etc/keepalived/keepalived.conf ，注意每个节点的IP和网卡（interface参数）

所有Master节点备份KeepAlived默认配置

```bash
cp -p /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak$(date '+%Y%m%d%H%M%S')
```

如果公司服务器还有其他keepalived，注意（virtual_router_id参数）不可以重复

```bash
[root@k8s-master01 ~]# vim /etc/keepalived/keepalived.conf
```

```sh
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2
    rise 1
}
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    mcast_src_ip 192.168.100.151
    virtual_router_id 51
    priority 101
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.100.160
    }
    track_script {
       chk_apiserver
    }
}
```

## 7.2  Master02 keepalived

```sh
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2
    rise 1
}
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    mcast_src_ip 192.168.100.152
    virtual_router_id 51
    priority 100
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.100.160
    }
    track_script {
       chk_apiserver
    }
}
```

## 7.3  Master03 keepalived

```sh
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2
    rise 1
}
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    mcast_src_ip 192.168.100.153
    virtual_router_id 51
    priority 100
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.100.160
    }
    track_script {
       chk_apiserver
    }
}
```

## 7.4  健康检查配置

所有master节点

```sh
[root@k8s-master01 keepalived]# vim /etc/keepalived/check_apiserver.sh
```

```sh
#!/bin/bash

err=0
for k in $(seq 1 3); do
  check_code=$(pgrep haproxy)
  if [[ $check_code == "" ]]; then
    err=$(expr $err + 1)
    sleep 1
    continue
  else
    err=0
    break
  fi
done

if [[ $err != "0" ]]; then
  echo "systemctl stop keepalived"
  /usr/bin/systemctl stop keepalived
  exit 1
else
  exit 0
fi
```

```sh
chmod +x /etc/keepalived/check_apiserver.sh
```

所有master节点启动haproxy和keepalived

```sh
systemctl daemon-reload && systemctl enable --now haproxy && systemctl enable --now keepalived
```

VIP测试

```sh
[root@k8s-master01 network-scripts]# ping 192.168.100.160 -c 4

PING 192.168.100.160 (192.168.100.160) 56(84) bytes of data.
64 bytes from 192.168.100.160: icmp_seq=1 ttl=64 time=0.016 ms
64 bytes from 192.168.100.160: icmp_seq=2 ttl=64 time=0.025 ms
64 bytes from 192.168.100.160: icmp_seq=3 ttl=64 time=0.015 ms
64 bytes from 192.168.100.160: icmp_seq=4 ttl=64 time=0.020 ms

--- 192.168.100.160 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3062ms
rtt min/avg/max/mdev = 0.015/0.019/0.025/0.004 ms
```

重要：如果安装了keepalived和haproxy，需要测试keepalived是否是正常的

```sh
[root@k8s-master01 network-scripts]# telnet 192.168.100.160 8443

Trying 192.168.100.160...
Connected to 192.168.100.160.
Escape character is '^]'.
Connection closed by foreign host.
```

- 如果ping不通且telnet没有出现 ]，则认为VIP不可以，不可在继续往下执行，需要排查keepalived的问题，比如防火墙和selinux，haproxy和keepalived的状态，监听端口等

- 所有节点查看防火墙状态必须为disable和inactive：systemctl status firewalld

- 所有节点查看selinux状态，必须为disable：getenforce

- master节点查看haproxy和keepalived状态：systemctl status keepalived haproxy

```bash
[root@k8s-master01 network-scripts]# systemctl status keepalived haproxy

● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-12-14 14:20:07 CST; 1min 37s ago
  Process: 15344 ExecStart=/usr/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 15345 (keepalived)
    Tasks: 3
   Memory: 3.1M
   CGroup: /system.slice/keepalived.service
           ├─15345 /usr/sbin/keepalived -D
           ├─15346 /usr/sbin/keepalived -D
           └─15347 /usr/sbin/keepalived -D

Dec 14 14:20:11 k8s-master01 Keepalived_vrrp[15347]: Sending gratuitous ARP on eth0 for 192.168.100.160
Dec 14 14:20:11 k8s-master01 Keepalived_vrrp[15347]: Sending gratuitous ARP on eth0 for 192.168.100.160
Dec 14 14:20:11 k8s-master01 Keepalived_vrrp[15347]: Sending gratuitous ARP on eth0 for 192.168.100.160
Dec 14 14:20:11 k8s-master01 Keepalived_vrrp[15347]: Sending gratuitous ARP on eth0 for 192.168.100.160
Dec 14 14:20:16 k8s-master01 Keepalived_vrrp[15347]: Sending gratuitous ARP on eth0 for 192.168.100.160
Dec 14 14:20:16 k8s-master01 Keepalived_vrrp[15347]: VRRP_Instance(VI_1) Sending/queueing gratuitous ARPs on et....160
Dec 14 14:20:16 k8s-master01 Keepalived_vrrp[15347]: Sending gratuitous ARP on eth0 for 192.168.100.160
Dec 14 14:20:16 k8s-master01 Keepalived_vrrp[15347]: Sending gratuitous ARP on eth0 for 192.168.100.160
Dec 14 14:20:16 k8s-master01 Keepalived_vrrp[15347]: Sending gratuitous ARP on eth0 for 192.168.100.160
Dec 14 14:20:16 k8s-master01 Keepalived_vrrp[15347]: Sending gratuitous ARP on eth0 for 192.168.100.160

● haproxy.service - HAProxy Load Balancer
   Loaded: loaded (/usr/lib/systemd/system/haproxy.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-12-14 14:20:07 CST; 1min 38s ago
 Main PID: 15322 (haproxy-systemd)
    Tasks: 3
   Memory: 2.3M
   CGroup: /system.slice/haproxy.service
           ├─15322 /usr/sbin/haproxy-systemd-wrapper -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
           ├─15328 /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds
           └─15329 /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds

Dec 14 14:20:07 k8s-master01 systemd[1]: Started HAProxy Load Balancer.
Dec 14 14:20:07 k8s-master01 haproxy-systemd-wrapper[15322]: haproxy-systemd-wrapper: executing /usr/sbin/haprox...-Ds
Dec 14 14:20:07 k8s-master01 haproxy-systemd-wrapper[15322]: [WARNING] 347/142007 (15328) : config : frontend 'G...ed.
Hint: Some lines were ellipsized, use -l to show in full.
```

- master节点查看监听端口：netstat -lntp

```bash
[root@k8s-master01 network-scripts]# netstat -lntp

Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:2379          0.0.0.0:*               LISTEN      1018/etcd           
tcp        0      0 192.168.100.151:2379    0.0.0.0:*               LISTEN      1018/etcd           
tcp        0      0 192.168.100.151:2380    0.0.0.0:*               LISTEN      1018/etcd           
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1021/sshd           
tcp        0      0 0.0.0.0:8443            0.0.0.0:*               LISTEN      5746/haproxy        
tcp        0      0 127.0.0.1:8443          0.0.0.0:*               LISTEN      5746/haproxy        
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      1117/master         
tcp        0      0 127.0.0.1:34355         0.0.0.0:*               LISTEN      1026/containerd     
tcp6       0      0 ::1:25                  :::*                    LISTEN      1117/master         
tcp6       0      0 :::22                   :::*                    LISTEN      1021/sshd 
```

# 第八章  Kubernetes组件配置

所有节点创建相关目录

```sh
mkdir -p /etc/kubernetes/manifests/ /etc/systemd/system/kubelet.service.d /var/lib/kubelet /var/log/kubernetes
```

## 8.1  Apiserver

所有Master节点创建kube-apiserver的service启动文件

> 注意：如果不是高可用集群，192.168.100.160:8443（VIP）需改为Master01的主机IP
>
> 本文档使用的k8s service网段为10.96.0.0/12，该网段不能和宿主机的网段、Pod网段的重复，请按需修改

### 8.1.0.1  Master01配置

```sh
vim /usr/lib/systemd/system/kube-apiserver.service
```

```sh
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
      --v=2  \
      --logtostderr=true  \
      --allow-privileged=true  \
      --bind-address=0.0.0.0  \
      --secure-port=6443  \
      --advertise-address=192.168.100.151 \
      --service-cluster-ip-range=10.96.0.0/12  \
      --service-node-port-range=30000-32767  \
      --etcd-servers=https://192.168.100.151:2379,https://192.168.100.152:2379,https://192.168.100.153:2379 \
      --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem  \
      --etcd-certfile=/etc/etcd/ssl/etcd.pem  \
      --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem  \
      --client-ca-file=/etc/kubernetes/pki/ca.pem  \
      --tls-cert-file=/etc/kubernetes/pki/apiserver.pem  \
      --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem  \
      --kubelet-client-certificate=/etc/kubernetes/pki/apiserver.pem  \
      --kubelet-client-key=/etc/kubernetes/pki/apiserver-key.pem  \
      --service-account-key-file=/etc/kubernetes/pki/sa.pub  \
      --service-account-signing-key-file=/etc/kubernetes/pki/sa.key  \
      --service-account-issuer=https://kubernetes.default.svc.cluster.local \
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
      --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
      --feature-gates=LegacyServiceAccountTokenNoAutoGeneration=false \
      --authorization-mode=Node,RBAC  \
      --enable-bootstrap-token-auth=true  \
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \
      --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \
      --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \
      --requestheader-allowed-names=aggregator  \
      --requestheader-group-headers=X-Remote-Group  \
      --requestheader-extra-headers-prefix=X-Remote-Extra-  \
      --requestheader-username-headers=X-Remote-User
      # --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

### 8.1.0.2  Master02配置

```sh
vim /usr/lib/systemd/system/kube-apiserver.service
```

```sh
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
      --v=2  \
      --logtostderr=true  \
      --allow-privileged=true  \
      --bind-address=0.0.0.0  \
      --secure-port=6443  \
      --advertise-address=192.168.100.152 \
      --service-cluster-ip-range=10.96.0.0/12  \
      --service-node-port-range=30000-32767  \
      --etcd-servers=https://192.168.100.151:2379,https://192.168.100.152:2379,https://192.168.100.153:2379 \
      --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem  \
      --etcd-certfile=/etc/etcd/ssl/etcd.pem  \
      --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem  \
      --client-ca-file=/etc/kubernetes/pki/ca.pem  \
      --tls-cert-file=/etc/kubernetes/pki/apiserver.pem  \
      --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem  \
      --kubelet-client-certificate=/etc/kubernetes/pki/apiserver.pem  \
      --kubelet-client-key=/etc/kubernetes/pki/apiserver-key.pem  \
      --service-account-key-file=/etc/kubernetes/pki/sa.pub  \
      --service-account-signing-key-file=/etc/kubernetes/pki/sa.key  \
      --service-account-issuer=https://kubernetes.default.svc.cluster.local \
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
      --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
      --feature-gates=LegacyServiceAccountTokenNoAutoGeneration=false \
      --authorization-mode=Node,RBAC  \
      --enable-bootstrap-token-auth=true  \
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \
      --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \
      --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \
      --requestheader-allowed-names=aggregator  \
      --requestheader-group-headers=X-Remote-Group  \
      --requestheader-extra-headers-prefix=X-Remote-Extra-  \
      --requestheader-username-headers=X-Remote-User
      # --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

### 8.1.0.3  Master03配置

```sh
vim /usr/lib/systemd/system/kube-apiserver.service
```

```sh
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
      --v=2  \
      --logtostderr=true  \
      --allow-privileged=true  \
      --bind-address=0.0.0.0  \
      --secure-port=6443  \
      --advertise-address=192.168.100.153 \
      --service-cluster-ip-range=10.96.0.0/12  \
      --service-node-port-range=30000-32767  \
      --etcd-servers=https://192.168.100.151:2379,https://192.168.100.152:2379,https://192.168.100.153:2379 \
      --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem  \
      --etcd-certfile=/etc/etcd/ssl/etcd.pem  \
      --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem  \
      --client-ca-file=/etc/kubernetes/pki/ca.pem  \
      --tls-cert-file=/etc/kubernetes/pki/apiserver.pem  \
      --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem  \
      --kubelet-client-certificate=/etc/kubernetes/pki/apiserver.pem  \
      --kubelet-client-key=/etc/kubernetes/pki/apiserver-key.pem  \
      --service-account-key-file=/etc/kubernetes/pki/sa.pub  \
      --service-account-signing-key-file=/etc/kubernetes/pki/sa.key  \
      --service-account-issuer=https://kubernetes.default.svc.cluster.local \
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
      --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
      --feature-gates=LegacyServiceAccountTokenNoAutoGeneration=false \
      --authorization-mode=Node,RBAC  \
      --enable-bootstrap-token-auth=true  \
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \
      --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \
      --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \
      --requestheader-allowed-names=aggregator  \
      --requestheader-group-headers=X-Remote-Group  \
      --requestheader-extra-headers-prefix=X-Remote-Extra-  \
      --requestheader-username-headers=X-Remote-User
      # --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

### 8.1.0.4  启动apiserver

所有Master节点开启kube-apiserver

```sh
systemctl daemon-reload && systemctl enable --now kube-apiserver
```

检测kube-server状态

```sh
systemctl status kube-apiserver
```

```sh
● kube-apiserver.service - Kubernetes API Server
   Loaded: loaded (/usr/lib/systemd/system/kube-apiserver.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-11-30 16:55:33 CST; 44s ago
```

查看系统日志信息

```sh
tail -f /var/log/messages
```

系统日志的这些提示可以忽略

如何区分是否是报错信息：[时间] [主机名] [kube-apiserver]: EXXXX

```sh
Jan 20 14:10:49 k8s-master01 kube-apiserver: I0120 14:10:49.839384    9156 clientconn.go:948] ClientConn switching balancer to "pick_first"
Jan 20 14:10:49 k8s-master01 kube-apiserver: I0120 14:10:49.839463    9156 balancer_conn_wrappers.go:78] pickfirstBalancer: HandleSubConnStateChange: 0xc00a4d3f40, {CONNECTING <nil>}
Jan 20 14:10:49 k8s-master01 kube-apiserver: I0120 14:10:49.843523    9156 balancer_conn_wrappers.go:78] pickfirstBalancer: HandleSubConnStateChange: 0xc00a4d3f40, {READY <nil>}
Jan 20 14:10:49 k8s-master01 kube-apiserver: I0120 14:10:49.844082    9156 controlbuf.go:508] transport: loopyWriter.run returning. connection error: desc = "transport is closing"
```

## 8.2  ControllerManager

所有Master节点创建kube-controller-manager的service启动文件

注意本文档使用的k8s Pod网段为172.16.0.0/12，该网段不能和宿主机的网段、k8s Service网段的重复，请按需修改

```sh
vim /usr/lib/systemd/system/kube-controller-manager.service
```

```sh
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
      --v=2 \
      --logtostderr=true \
      --root-ca-file=/etc/kubernetes/pki/ca.pem \
      --cluster-signing-cert-file=/etc/kubernetes/pki/ca.pem \
      --cluster-signing-key-file=/etc/kubernetes/pki/ca-key.pem \
      --service-account-private-key-file=/etc/kubernetes/pki/sa.key \
      --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig \
      --feature-gates=LegacyServiceAccountTokenNoAutoGeneration=false \
      --leader-elect=true \
      --use-service-account-credentials=true \
      --node-monitor-grace-period=40s \
      --node-monitor-period=5s \
      --pod-eviction-timeout=2m0s \
      --controllers=*,bootstrapsigner,tokencleaner \
      --allocate-node-cidrs=true \
      --cluster-cidr=172.16.0.0/12 \
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem \
      --node-cidr-mask-size=24
      
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

所有Master节点启动kube-controller-manager

```sh
systemctl daemon-reload && systemctl enable --now kube-controller-manager
```

查看启动状态

```sh
systemctl status kube-controller-manager
```

```sh
● kube-controller-manager.service - Kubernetes Controller Manager
   Loaded: loaded (/usr/lib/systemd/system/kube-controller-manager.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-12-14 14:27:36 CST; 3s ago
```

查看系统日志信息

```sh
tail -f /var/log/messages
```

## 8.3  Scheduler

所有Master节点创建kube-scheduler的service启动文件

```sh
vim /usr/lib/systemd/system/kube-scheduler.service
```

```sh
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
      --v=2 \
      --logtostderr=true \
      --leader-elect=true \
      --authentication-kubeconfig=/etc/kubernetes/scheduler.kubeconfig \
      --authorization-kubeconfig=/etc/kubernetes/scheduler.kubeconfig \
      --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

所有Master节点启动kube-controller-manager

```sh
systemctl daemon-reload && systemctl enable --now kube-scheduler
```

查看启动状态

```sh
systemctl status kube-scheduler
```

```sh
● kube-scheduler.service - Kubernetes Scheduler
   Loaded: loaded (/usr/lib/systemd/system/kube-scheduler.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-12-14 14:28:38 CST; 2s ago
```

查看系统日志信息

```sh
tail -f /var/log/messages
```

# 第九章  TLS Bootstrapping配置

> ControllerManager、Scheduler需要和Apiserver通信，拥有对应的kubeconfig文件，没有主机绑定关系，每个节点的证书一样。
>
> Kubelet、kube-proxy也需要和Apiserver通信，有主机绑定关系（主机名、IP地址），每个集群的证书都是不一样的，所以证书无法统一生成。如果手动生成证书，可能会有几百上千个节点，每个节点都需要配置一遍，而主节点只有几个，可以手动关联证书，但Kubelet手动去管理的话就非常麻烦了。
>
> 我们用到了Bootstrapping的功能，它可以为我们的工作节点自动颁发证书，首先生成一个bootstrap的kubeconfig文件（bootstrap-kubelet.kubeconfig），它的权限是为了生成我们（kubelet）的证书，Kubelet在启动的时候指定这个 bootstrap-kubelet.kubeconfig 文件，会自动申请一个证书，然后ControllerManager会自动颁发这个证书

在Master01创建bootstrap

> 注意：如果不是高可用集群，192.168.100.160:8443（VIP）需改为Master01的主机IP，8443改为apiserver的端口，默认是6443

```sh
cd /opt/installation_package/k8s-ha-install/bootstrap

kubectl config set-cluster kubernetes --certificate-authority=/etc/kubernetes/pki/ca.pem --embed-certs=true --server=https://192.168.100.160:8443 --kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig
# token需要与bootstrap.secret.yaml的token-id和token-secret保持一致（默认可不改）
kubectl config set-credentials tls-bootstrap-token-user --token=c8ad9c.2e4d610cf3e7426e --kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig
kubectl config set-context tls-bootstrap-token-user@kubernetes --cluster=kubernetes --user=tls-bootstrap-token-user --kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig
kubectl config use-context tls-bootstrap-token-user@kubernetes --kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig
```

注意：如果要修改bootstrap.secret.yaml的token-id和token-secret，需要保证下图红圈内的字符串一致的，并且位数是一样的。还要保证上个命令的黄色字体：c8ad9c.2e4d610cf3e7426e与你修改的字符串要一致

![image-20221130175436418](二进制高可用安装k8s集群.assets/image-20221130175436418.png)

```sh
# 授权k8s-master01节点kubectl客户端管理员权限（其它节点想要kubectl客户端管理集群，也需复制kubeconfig文件）
[root@k8s-master01 bootstrap]# mkdir -p /root/.kube ; cp /etc/kubernetes/admin.kubeconfig /root/.kube/config

# 可以正常查询集群状态，才可以继续往下，否则不行，需要排查k8s组件是否有故障
[root@k8s-master01 bootstrap]# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
scheduler            Healthy   ok                              
controller-manager   Healthy   ok                              
etcd-2               Healthy   {"health":"true","reason":""}   
etcd-1               Healthy   {"health":"true","reason":""}   
etcd-0               Healthy   {"health":"true","reason":""}

[root@k8s-master01 bootstrap]# kubectl create -f bootstrap.secret.yaml
secret/bootstrap-token-c8ad9c created
clusterrolebinding.rbac.authorization.k8s.io/kubelet-bootstrap created
clusterrolebinding.rbac.authorization.k8s.io/node-autoapprove-bootstrap created
clusterrolebinding.rbac.authorization.k8s.io/node-autoapprove-certificate-rotation created
clusterrole.rbac.authorization.k8s.io/system:kube-apiserver-to-kubelet created
clusterrolebinding.rbac.authorization.k8s.io/system:kube-apiserver created
```

> 使用/root/.kube/config访问集群解决：
>
> kubectl get node：The connection to the server localhost:8080 was refused - did you specify the right host or port?

# 第十章  Node节点配置

## 10.1  复制证书

Master01节点复制证书至Node节点

```sh
cd /etc/kubernetes/
```

```sh
for NODE in k8s-master02 k8s-master03 k8s-node01 k8s-node02; do
    ssh $NODE mkdir -p /etc/kubernetes/pki
    for FILE in pki/ca.pem pki/ca-key.pem pki/front-proxy-ca.pem bootstrap-kubelet.kubeconfig; do
        scp /etc/kubernetes/$FILE $NODE:/etc/kubernetes/${FILE}
    done
done
```

执行结果：

```sh
ca.pem                                                100% 1363     1.3MB/s   00:00
ca-key.pem                                            100% 1675     2.1MB/s   00:00
front-proxy-ca.pem                                    100% 1094    20.9KB/s   00:00
bootstrap-kubelet.kubeconfig                          100% 2238     2.7MB/s   00:00
ca.pem                                                100% 1363     1.4MB/s   00:00
ca-key.pem                                            100% 1675    28.5KB/s   00:00
front-proxy-ca.pem                                    100% 1094     1.0MB/s   00:00
bootstrap-kubelet.kubeconfig                          100% 2238     2.8MB/s   00:00
ca.pem                                                100% 1363     1.8MB/s   00:00
ca-key.pem                                            100% 1675     2.3MB/s   00:00
front-proxy-ca.pem                                    100% 1094     1.5MB/s   00:00
bootstrap-kubelet.kubeconfig                          100% 2238     2.9MB/s   00:00
ca.pem                                                100% 1363     1.6MB/s   00:00
ca-key.pem                                            100% 1675    57.9KB/s   00:00
front-proxy-ca.pem                                    100% 1094     1.4MB/s   00:00
bootstrap-kubelet.kubeconfig                          100% 2238   531.7KB/s   00:00
```

## 10.2  Kubelet配置

所有节点创建相关目录

```sh
mkdir -p /var/lib/kubelet /var/log/kubernetes /etc/systemd/system/kubelet.service.d /etc/kubernetes/manifests/
```

所有节点创建kubelet的service启动文件

```sh
vim /usr/lib/systemd/system/kubelet.service
```

```sh
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kubelet

Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
```

所有节点创建kubelet的service配置文件

```sh
vim /etc/systemd/system/kubelet.service.d/10-kubelet.conf
```

```sh
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig"
Environment="KUBELET_SYSTEM_ARGS=--container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
Environment="KUBELET_CONFIG_ARGS=--config=/etc/kubernetes/kubelet-conf.yml"
Environment="KUBELET_EXTRA_ARGS=--node-labels=node.kubernetes.io/node='' "
ExecStart=
ExecStart=/usr/local/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_SYSTEM_ARGS $KUBELET_EXTRA_ARGS
```

所有节点创建kubelet的配置文件

> 注意：如果更改了k8s Service网段，需要更改kubelet-conf.yml的clusterDNS配置，改成k8s Service网段的第十个IP地址（如10.96.0.10）

```sh
vim /etc/kubernetes/kubelet-conf.yml
```

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: systemd
cgroupsPerQOS: true
clusterDNS:
  - 10.96.0.10
clusterDomain: cluster.local
containerLogMaxFiles: 5
containerLogMaxSize: 10Mi
contentType: application/vnd.kubernetes.protobuf
cpuCFSQuota: true
cpuManagerPolicy: none
cpuManagerReconcilePeriod: 10s
enableControllerAttachDetach: true
enableDebuggingHandlers: true
enforceNodeAllocatable:
  - pods
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: true
fileCheckFrequency: 20s
hairpinMode: promiscuous-bridge
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 20s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
iptablesDropBit: 15
iptablesMasqueradeBit: 14
kubeAPIBurst: 10
kubeAPIQPS: 5
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
registryBurst: 10
registryPullQPS: 5
resolvConf: /etc/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
volumeStatsAggPeriod: 1m0s
```

所有节点启动kubelet

```bash
systemctl daemon-reload && systemctl enable --now kubelet
```

查看系统日志

```sh
tail -f /var/log/messages
```

此时系统日志/var/log/messages显示只有如下信息为正常，安装CNI网络插件Calico即可恢复正常

> Unable to update cni config: no networks found in /etc/cni/net.d
>
> "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"

![image-20221201145701274](二进制高可用安装k8s集群.assets/image-20221201145701274.png)

> 如果有很多报错日志，或者有大量看不懂的报错，说明kubelet的配置有误，需要检查kubelet配置！

master01查看集群状态（Ready或NotReady都正常）

```sh
[root@k8s-master01 bootstrap]# kubectl get node
```

 ![image-20221214144723481](二进制高可用安装k8s集群.assets/image-20221214144723481.png)

## 10.3  kube-proxy配置

注意：如果不是高可用集群，192.168.100.160:8443（VIP）需改为Master01的主机IP，8443改为apiserver的端口，默认是6443

以下操作在Master01执行

```sh
cd /opt/installation_package/k8s-ha-install
```

```sh
kubectl -n kube-system create serviceaccount kube-proxy

kubectl create clusterrolebinding system:kube-proxy --clusterrole system:node-proxier --serviceaccount kube-system:kube-proxy

SECRET=$(kubectl -n kube-system get sa/kube-proxy \
  --output=jsonpath='{.secrets[0].name}')

JWT_TOKEN=$(kubectl -n kube-system get secret/$SECRET \
  --output=jsonpath='{.data.token}' | base64 -d)

PKI_DIR=/etc/kubernetes/pki
K8S_DIR=/etc/kubernetes

kubectl config set-cluster kubernetes --certificate-authority=/etc/kubernetes/pki/ca.pem --embed-certs=true --server=https://192.168.100.160:8443 --kubeconfig=${K8S_DIR}/kube-proxy.kubeconfig

kubectl config set-credentials kubernetes --token=${JWT_TOKEN} --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config set-context kubernetes --cluster=kubernetes --user=kubernetes --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config use-context kubernetes --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig
```

在master01将kube-proxy的kube-proxy.kubeconfig配置文件发送到其他节点

```sh
for NODE in k8s-master02 k8s-master03; do
  scp /etc/kubernetes/kube-proxy.kubeconfig $NODE:/etc/kubernetes/kube-proxy.kubeconfig
done

for NODE in k8s-node01 k8s-node02; do
  scp /etc/kubernetes/kube-proxy.kubeconfig $NODE:/etc/kubernetes/kube-proxy.kubeconfig
done
```

所有节点配置kube-proxy的service启动文件：

```bash
vim /usr/lib/systemd/system/kube-proxy.service
```

```ini
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-proxy \
  --config=/etc/kubernetes/kube-proxy.yaml \
  --v=2

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

所有节点创建kube-proxy的配置文件

> 注意：如果更改了集群Pod的网段，需要更改kube-proxy.yaml的clusterCIDR: 172.16.0.0/12 参数为自己的Pod网段。

```bash
vim /etc/kubernetes/kube-proxy.yaml
```

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
  qps: 5
clusterCIDR: 172.16.0.0/12 
configSyncPeriod: 15m0s
conntrack:
  max: null
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
  masqueradeAll: true
  minSyncPeriod: 5s
  scheduler: "rr"
  syncPeriod: 30s
kind: KubeProxyConfiguration
metricsBindAddress: 127.0.0.1:10249
mode: "ipvs"
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
udpIdleTimeout: 250ms
```

所有节点启动kube-proxy

```sh
systemctl daemon-reload && systemctl enable --now kube-proxy
```

检查kube-proxy的启动日志是否正常打印：Adding new service port

```bash
systemctl status kube-proxy -l
```

```bash
"Adding new service port" portName="default/kubernetes:https" servicePort="10.96.0.1:443/TCP"
```

# 第十一章  安装Calico

> Calico的安装请必须听视频课程和最后一章升级Calico的视频

Calico官方K8s安装配置：https://github.com/projectcalico/calico/blob/release-v3.24/manifests/calico.yaml

## 11.1  安装官方推荐版本（推荐）

**以下步骤只在master01执行**

```sh
cd /opt/installation_package/k8s-ha-install/calico/
```

更改calico的网段，主要需要将红色部分的网段，改为自己的Pod网段

```bash
sed -i "s#POD_CIDR#172.16.0.0/12#g" calico.yaml
```

![image-20221201182549891](二进制高可用安装k8s集群.assets/image-20221201182549891.png)

检查网段是否修改为自己的Pod网段

```bash
[root@k8s-master01 calico]# grep "CALICO_IPV4POOL_CIDR" calico.yaml  -A 1
            - name: CALICO_IPV4POOL_CIDR
              value: "172.16.0.0/12"
```

部署Calico

```bash
[root@k8s-master01 calico]# kubectl apply -f calico.yaml
```

查看容器状态

```bash
[root@k8s-master01 calico]# kubectl get po -n kube-system
```

如果容器状态异常可以使用kubectl describe或者logs查看容器的日志

# 第十二章  安装CoreDNS

## 12.1  安装对应版本（推荐）

```sh
cd /opt/installation_package/k8s-ha-install/CoreDNS
```

> 注意：如果更改了k8s Service网段，需要更改coredns的serviceIP配置，改成k8s Service网段的第十个IP地址（如10.96.0.10）

```sh
[root@k8s-master01 k8s-ha-install]# COREDNS_SERVICE_IP=`kubectl get svc | grep kubernetes | awk '{print $3}'`0
[root@k8s-master01 k8s-ha-install]# sed -i "s#KUBEDNS_SERVICE_IP#${COREDNS_SERVICE_IP}#g" coredns.yaml
```

安装coredns

```sh
[root@k8s-master01 k8s-ha-install]# kubectl create -f coredns.yaml
```

```sh
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
```

## 12.2  安装最新版CoreDNS

```sh
git clone https://github.com/coredns/deployment.git && cd deployment/kubernetes
COREDNS_SERVICE_IP=`kubectl get svc | grep kubernetes | awk '{print $3}'`0
./deploy.sh -s -i ${COREDNS_SERVICE_IP} | kubectl apply -f -
```

查看状态

```sh
kubectl get po -n kube-system -l k8s-app=kube-dns
```

```sh
NAME                       READY   STATUS    RESTARTS   AGE
coredns-6fb54f4b75-swcsw   1/1     Running   0          10m
```

如果容器状态异常可以使用kubectl describe或者logs查看容器的日志

```bash
[root@k8s-master01 ~]# kubectl logs --namespace=kube-system -l k8s-app=kube-dns
.:53
[INFO] plugin/reload: Running configuration SHA512 = 165c2220dade77166d5aa3508a7dd59a865dbb9a90ca5af87e487b4217222b80d5b35bedd2640b4c4895a10c969c81b9d1ff90d2592ccccc0307eebe5469b6ab
CoreDNS-1.10.0
linux/amd64, go1.19.1, 596a9f9
```

# 第十三章  安装Metrics Server

在新版的Kubernetes中系统资源的采集均使用Metrics-server，可以通过Metrics采集节点和Pod的内存、磁盘、CPU和网络的使用率。

安装metrics server

```sh
[root@k8s-master01 metrics-server]# cd /opt/installation_package/k8s-ha-install/metrics-server && kubectl create -f .
```

```sh
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
```

查看状态

```bash
[root@k8s-master01 metrics-server]# kubectl get po -n kube-system -l k8s-app=metrics-server
```

```bash
NAME                             READY   STATUS    RESTARTS   AGE
metrics-server-d49c495f4-bfn6z   1/1     Running   0          60s
```

等待metrics server启动然后查看状态

```bash
[root@k8s-master01 metrics-server]# kubectl top node
```

```
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-master01   97m          2%     1521Mi          26%       
k8s-master02   81m          2%     1352Mi          23%       
k8s-master03   75m          1%     1359Mi          23%       
k8s-node01     37m          0%     655Mi           4%        
k8s-node02     36m          0%     705Mi           4% 
```

# [第十四章 k8s-master节点添加/移除污点taints](https://blog.csdn.net/cd_yourheart/article/details/108766912)

手动部署的k8s集群, 需要为master节点手动设置taints

## 14.1  污点设置常用命令

### 14.1.1 设置taint

语法：

```bash
kubectl taint node [node] key=value[effect]   
     其中[effect] 可取值: [ NoSchedule | PreferNoSchedule | NoExecute ]
      NoSchedule: 一定不能被调度
      PreferNoSchedule: 尽量不要调度
      NoExecute: 不仅不会调度, 还会驱逐Node上已有的Pod
```

示例：

```bash
kubectl taint node node1 key1=value1:NoSchedule
kubectl taint node node1 key1=value1:NoExecute
kubectl taint node node1 key2=value2:NoSchedule
```

### 14.1.2  查看taint

```bash
kubectl describe node node1
```

### 14.1.3  删除taint

```bash
kubectl taint node node1 key1:NoSchedule-  # 这里的key可以不用指定value
kubectl taint node node1 key1:NoExecute-
# kubectl taint node node1 key1-  删除指定key所有的effect
kubectl taint node node1 key2:NoSchedule-
```

## 14.2  Master节点设置taint（推荐）

```bash
kubectl taint nodes k8s-master01 k8s-master02 k8s-master03 node-role.kubernetes.io/master=:NoSchedule
```

> 注意⚠️ : 为master设置的这个taint中, `node-role.kubernetes.io/master`为`key`, `value`为空, `effect`为`NoSchedule`
>
> 如果输入命令时, 你丢掉了`=`符号, 写成了`node-role.kubernetes.io/master:NoSchedule`, 会报`error: at least one taint update is required`错误

## 14.3  容忍tolerations主节点的taints

> 以上面为 k8s-master01 设置的 taints 为例, 你需要为你的 yaml 文件中添加如下配置, 才能容忍 master 节点的污点
>
> 在 pod 的 spec 中设置 tolerations 字段

```yaml
tolerations:
- key: "node-role.kubernetes.io/master"
  operator: "Equal"
  value: ""
  effect: "NoSchedule"
```

## 14.4  Master节点移除taint

```bash
#查看污点策略，显示三个master节点都是NoSchedule
[root@master1 ~]# kubectl get no -o yaml | grep taint -A 5
    taints:
    - effect: NoSchedule
      key: node-role.kubernetes.io/master
  status:
    addresses:
    - address: 192.168.100.151
--
    taints:
    - effect: NoSchedule
      key: node-role.kubernetes.io/master
  status:
    addresses:
    - address: 192.168.100.152
--
    taints:
    - effect: NoSchedule
      key: node-role.kubernetes.io/master
  status:
    addresses:
    - address: 192.168.100.153

#去除污点，允许master节点部署pod
[root@master1 ~]# kubectl taint nodes --all node-role.kubernetes.io/master-
node/k8s-master01 untainted
node/k8s-master02 untainted
node/k8s-master03 untainted
taint "node-role.kubernetes.io/master" not found
taint "node-role.kubernetes.io/master" not found
 
#再次查看，无显示，说明污点去除成功
[root@master1 ~]# kubectl get no -o yaml | grep taint -A 5
```

# 第十五章  集群验证

> 集群验证请参考视频的集群验证，必须要做！！！

安装busybox

```sh
[root@k8s-master01 busybox]# cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - name: busybox
    image: registry.cn-beijing.aliyuncs.com/dotbalo/redis-trib:4.0.10
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
EOF
```

安装dnsutils

```bash
[root@k8s-master01 busybox]# cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dnsutils
  namespace: default
spec:
  containers:
  - name: dnsutils
    image: opsdockerimage/e2e-test-images-jessie-dnsutils:1.7
    command:
      - sleep
      - "infinity" # 终端睡眠无限长时间
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
EOF
```

1. Pod必须能解析Service
2. Pod必须能解析跨namespace的Service
3. 每个节点都必须要能访问Kubernetes的kubernetes svc 443和kube-dns的service 53
4. Pod和Pod之前要能通
   1. 同namespace能通信
   2. 跨namespace能通信
   3. 跨机器能通信



**验证解析（请参考视频集群验证）**

调试 DNS 问题：https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/dns-debugging-resolution/

```bash
[root@k8s-master01 ~]# kubectl exec -it dnsutils -- nslookup kubernetes.default
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	kubernetes.default.svc.cluster.local
Address: 10.96.0.1
```

> [nslookup reported "can't resolve '(null)': Name does not resolve" though it successfully resolved the DNS names](https://stackoverflow.com/questions/46457136/nslookup-reported-cant-resolve-null-name-does-not-resolve-though-it-succ)

```sh
# Pod必须能解析Service，default namespaces
[root@k8s-master01 CoreDNS]# kubectl exec busybox -n default -- nslookup kubernetes
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
nslookup: can't resolve '(null)': Name does not resolve

# Pod必须能解析跨namespace的Service，kusybox默认处于default namespaces下，而kube-dns处于kube-system namespaces下
[root@k8s-master01 CoreDNS]# kubectl exec busybox -n default -- nslookup kube-dns.kube-system
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kube-dns.kube-system
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
nslookup: can't resolve '(null)': Name does not resolve

# 每个节点都必须要能访问Kubernetes的kubernetes svc 443和kube-dns的service 53
[root@k8s-master01 CoreDNS] telnet 10.96.0.1 443
Trying 10.96.0.1...
Connected to 10.96.0.1.
Escape character is '^]'.

[root@k8s-master01 ~]# kubectl get svc -n kube-system
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
calico-typha     ClusterIP   10.108.155.65    <none>        5473/TCP                 20m
kube-dns         ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP   15m
metrics-server   ClusterIP   10.103.157.250   <none>        443/TCP                  13m

[root@k8s-master01 ~]# telnet 10.96.0.10 53
Trying 10.96.0.10...
Connected to 10.96.0.10.
Escape character is '^]'.
Connection closed by foreign host.

[root@k8s-master01 ~]# kubectl get po -n kube-system -owide
NAME                                       READY   STATUS    RESTARTS   AGE   IP                NODE           NOMINATED NODE   READINESS GATES
calico-kube-controllers-767f55c9d8-d6sxc   1/1     Running   0          16h   172.17.125.1      k8s-node01     <none>           <none>
calico-node-2b7fh                          1/1     Running   0          16h   192.168.100.152   k8s-master02   <none>           <none>
calico-node-6zz7f                          1/1     Running   0          16h   192.168.100.154   k8s-node01     <none>           <none>
calico-node-jt52q                          1/1     Running   0          16h   192.168.100.151   k8s-master01   <none>           <none>
calico-node-l4xtp                          1/1     Running   0          16h   192.168.100.153   k8s-master03   <none>           <none>
calico-node-m9h5n                          1/1     Running   0          16h   192.168.100.155   k8s-node02     <none>           <none>
calico-typha-68f6758b6-vjdpq               1/1     Running   0          16h   192.168.100.154   k8s-node01     <none>           <none>
coredns-7b49b88cb5-wjgcx                   1/1     Running   0          16h   172.25.92.65      k8s-master02   <none>           <none>
metrics-server-d49c495f4-mb8r2             1/1     Running   0          16h   172.27.14.193     k8s-node02     <none>           <none>

[root@k8s-master01 ~]# kubectl get po -owide
NAME       READY   STATUS    RESTARTS       AGE   IP              NODE           NOMINATED NODE   READINESS GATES
busybox    1/1     Running   16 (57s ago)   16h   172.27.14.194   k8s-node02     <none>           <none>
dnsutils   1/1     Running   0              32m   172.18.195.1    k8s-master03   <none>           <none>

[root@k8s-master01 ~]# kubectl exec -it busybox -- sh
/data # ping 192.168.100.151 -c 3
PING 192.168.100.151 (192.168.100.151): 56 data bytes
64 bytes from 192.168.100.151: seq=0 ttl=63 time=0.179 ms
64 bytes from 192.168.100.151: seq=1 ttl=63 time=0.205 ms
64 bytes from 192.168.100.151: seq=2 ttl=63 time=0.214 ms

# 快速部署一个Deployment
[root@k8s-master01 ~]# kubectl create deploy nginx --image=nginx:1.23.2 --replicas=3
deployment.apps/nginx created
[root@k8s-master01 ~]# kubectl get deploy
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           37s
[root@k8s-master01 ~]# kubectl get po -owide
NAME                     READY   STATUS              RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
busybox                  1/1     Running             0          45m   172.27.14.194   k8s-node02     <none>           <none>
nginx-6799fc88d8-fgr8w   0/1     ContainerCreating   0          45s   <none>          k8s-master03   <none>           <none>
nginx-6799fc88d8-ltvtm   0/1     ContainerCreating   0          45s   <none>          k8s-master02   <none>           <none>
nginx-6799fc88d8-srgln   0/1     ContainerCreating   0          45s   <none>          k8s-master01   <none>           <none>

# 删除集群验证相关资源
[root@k8s-master01 ~]# kubectl delete deploy nginx
[root@k8s-master01 ~]# kubectl delete po busybox dnsutils
[root@k8s-node01 ~]# ctr -n k8s.io i ls | grep 'nginx'
```

```bash
docker.io/library/nginx:1.23.2                                                                                                    application/vnd.docker.distribution.manifest.list.v2+json sha256:ab589a3c466e347b1c0573be23356676df90cd7ce2dbf6ec332a5f0a8b5e59db 54.2 MiB  linux/386,linux/amd64,linux/arm/v5,linux/arm/v7,linux/arm64/v8,linux/mips64le,linux/ppc64le,linux/s390x io.cri-containerd.image=managed 
docker.io/library/nginx@sha256:ab589a3c466e347b1c0573be23356676df90cd7ce2dbf6ec332a5f0a8b5e59db                                   application/vnd.docker.distribution.manifest.list.v2+json sha256:ab589a3c466e347b1c0573be23356676df90cd7ce2dbf6ec332a5f0a8b5e59db 54.2 MiB  linux/386,linux/amd64,linux/arm/v5,linux/arm/v7,linux/arm64/v8,linux/mips64le,linux/ppc64le,linux/s390x io.cri-containerd.image=managed 
```

```bash
ctr -n k8s.io i rm docker.io/library/nginx:1.23.2 docker.io/library/nginx@sha256:ab589a3c466e347b1c0573be23356676df90cd7ce2dbf6ec332a5f0a8b5e59db
```

# 第十六章  安装dashboard

## 16.1  Dashboard部署

Dashboard用于展示集群中的各类资源，同时也可以通过Dashboard实时查看Pod的日志和在容器中执行一些命令等。

### 16.1.1  安装指定版本dashboard（推荐）

```sh
[root@k8s-master01 dashboard]# cd /opt/installation_package/k8s-ha-install/dashboard/ && kubectl create -f .
```

```sh
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```

### 16.1.2  安装最新版

官方GitHub地址：https://github.com/kubernetes/dashboard

可以在官方dashboard查看到最新版dashboard

![img](二进制高可用安装k8s集群.assets/wps9.jpg) 

官方GitHub地址：https://github.com/kubernetes/dashboard

可以在官方dashboard查看到最新版dashboard

> 注意：需要手动设置静态端口（NodePort）暴露服务

```yaml
spec:
  type: NodePort # 静态端口（NodePort）暴露服务
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
```

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.1.0/aio/deploy/recommended.yaml
```

创建管理员用户

```bash
[root@k8s-master01 dashboard]# cd /opt/installation_package/k8s-ha-install/dashboard/
[root@k8s-master01 dashboard]# vim dashboard-user.yaml
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

```sh
kubectl apply -f dashboard-user.yaml -n kube-system
```

### 16.1.3  登录dashboard

在谷歌浏览器（Chrome）启动文件中加入启动参数，用于解决无法访问Dashboard的问题，参考图1-1：

```sh
--test-type --ignore-certificate-errors

# "C:\Program Files (x86)\Google\Chrome\Application\chrome.exe" --test-type --ignore-certificate-errors
```

![img](二进制高可用安装k8s集群.assets/wps10.jpg) 

图1-1  谷歌浏览器 Chrome的配置



更改dashboard的svc为NodePort：

```sh
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
```

![img](二进制高可用安装k8s集群.assets/wps11.jpg) 

将ClusterIP更改为NodePort（如果已经为NodePort忽略此步骤）：

查看端口号：

![image-20221205143329402](二进制高可用安装k8s集群.assets/image-20221205143329402.png)

```sh
[root@k8s-master01 dashboard]# kubectl get svc kubernetes-dashboard -n kubernetes-dashboard
NAME                   TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.96.174.10   <none>        443:32439/TCP   2m13s
```

根据自己的实例端口号，通过任意安装了kube-proxy的宿主机或者VIP的IP+端口即可访问到dashboard：

访问Dashboard：[https://192.168.100.160:32439（请更改32439为自己的端口）](https://192.168.100.160:32439)，选择登录方式为令牌（即token方式），参考图1-2

![image-20221205145016829](二进制高可用安装k8s集群.assets/image-20221205145016829.png)

图1-2 Dashboard登录方式

查看token值（**注意不要复制到空格**）：

```sh
[root@k8s-master01 dashboard]# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-wldm9
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: ece33cb7-7112-41c3-9a3d-bd1ae979ace8

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1411 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IkgxeEctbFp5S0NWdVE3Wnp6Z00zSEo3Sy1ZeFRRRlhJRnc2SlNJZ0xLMmsifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXdsZG05Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJlY2UzM2NiNy03MTEyLTQxYzMtOWEzZC1iZDFhZTk3OWFjZTgiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.E5lYrO24xpgbTvLY1oxEW267N3MwQg-XCxaT9ybgaBn15kup36sN4foIAFlQvtGmamp_29qs9pgkZWu_P5xHTUtibGHx568JpwSGYkUUAy6o615jCCspsj5BoUnrGBf9VoRVN2CCGRylbGE2EqueXTCPC-B3RGOaUVcjk7LO4nLD_O12jlF-qqRSbKfdnw3glmZNEAwrMBIJZ_681YxlQfb_qAEwjNkRofOOhTk72ozHR9eNtd8jdeed9KUUZO9nwumUVGJmkhOA-yTOt9UaGCO_cUNLdq3ih8H2f67K6mj8PVQe--smiJE2pFgvgBpamzFVQhU5qZGX2KLbf2m9UA
```

将token值输入到令牌后，单击登录即可访问Dashboard，参考图1-3：

![image-20221205145942780](二进制高可用安装k8s集群.assets/image-20221205145942780.png)

图1-3 Dashboard页面

# 第十七章  生产环境关键性配置（K8s≤ 1.24）

> 关键性配置请参考视频，不要直接配置！

所有节点

```
vim /etc/docker/daemon.json
```

```sh
{
	"registry-mirrors": [
	"https://registry.docker-cn.com",
	"http://hub-mirror.c.163.com",
	"https://docker.mirrors.ustc.edu.cn",
	"https://b9pmyelo.mirror.aliyuncs.com"
	],
	"exec-opts": ["native.cgroupdriver=systemd"],
	# 并发下载的线程数
	"max-concurrent-downloads": 10,
	# 并发上传的线程数
	"max-concurrent-uploads": 5,
	# docker容器控制台日志文件切割
	"log-opts": {
	"max-size": "300m",
	"max-file": "2"
	},
	# 重启docker不影响运行的容器
	"live-restore": true
}
```

仅master节点
kube-controller-manager.service签发证书有效期配置

```sh
vim /usr/lib/systemd/system/kube-controller-manager.service
```

```sh
# --feature-gates=RotateKubeletClientCertificate=true,RotateKubeletServerCertificate=true \
--cluster-signing-duration=876000h0m0s \
```

```sh
systemctl daemon-reload && systemctl restart kube-controller-manager && systemctl status kube-controller-manager
tail -f /var/log/messages
```

所有节点

10-kubelet.conf加密方式、镜像拉取时间配置

```sh
vim /etc/systemd/system/kubelet.service.d/10-kubelet.conf
```

```sh
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig"
Environment="KUBELET_SYSTEM_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_CONFIG_ARGS=--config=/etc/kubernetes/kubelet-conf.yml --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.2"
Environment="KUBELET_EXTRA_ARGS=--node-labels=node.kubernetes.io/node='' --tls-cipher-suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384    --image-pull-progress-deadline=30m "
ExecStart=
ExecStart=/usr/local/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_SYSTEM_ARGS $KUBELET_EXTRA_ARGS
```

> 关键性配置请参考视频，不要直接配置，可能会造成集群故障！！

所有节点

```
vim /etc/kubernetes/kubelet-conf.yml
```

添加如下配置

如格式丢失，粘贴前使用命令

```sh
:set paste
```

```sh
rotateServerCertificates: true
allowedUnsafeSysctls:
 - "net.core*"
 - "net.ipv4.*"
kubeReserved:
  cpu: "1"
  memory: 1Gi
  ephemeral-storage: 10Gi
systemReserved:
  cpu: "1"
  memory: 1Gi
  ephemeral-storage: 10Gi
```

```sh
systemctl daemon-reload && systemctl restart kubelet && systemctl status kubelet
tail -f /var/log/messages
```

安装总结：

1、 kubeadm

2、 二进制

3、 自动化安装

a) Ansible

i. Master节点安装不需要写自动化。

ii. 添加Node节点，playbook。

4、 安装需要注意的细节

a) 上面的细节配置

b) 生产环境中etcd一定要和系统盘分开，一定要用ssd硬盘。

c) Docker数据盘也要和系统盘分开，有条件的话可以使用ssd硬盘



# 第十八章  安装Ingress Controller

Ingress Examples：https://github.com/kubernetes/ingress-nginx/tree/main/docs/examples

## 18.1  安装对应版本（推荐）

官方安装文档：https://kubernetes.github.io/ingress-nginx/deploy/#bare-metal-clusters

> 基于[官方安装文件](https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/baremetal/deploy.yaml)，修改国内镜像制作

```bash
[root@k8s-master01 ingress]# cd /opt/installation_package/k8s-ha-install/ingress/ && kubectl create -f .
```

```bash
[root@k8s-master01 ingress]# kubectl get pod -n ingress-nginx 
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-6bdn8        0/1     Completed   0          10m
ingress-nginx-admission-patch-5htfh         0/1     Completed   0          10m
ingress-nginx-controller-64c6577bf5-b4xx2   1/1     Running     0          10m
```

## 18.2  Helm安装Ingress Controller（生产级高可用架构）

### 18.2.1  Helm管理工具安装

- Helm管理工具安装文档：https://helm.sh/docs/intro/install/
- Helm管理工具最新版本：https://github.com/helm/helm/releases

```bash
wget https://get.helm.sh/helm-v3.11.0-linux-amd64.tar.gz
tar -zxvf helm-v3.11.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm && helm version
```

### 18.2.2  Helm安装Ingress

Helm安装Ingress文档：[https://kubernetes.github.io/ingress-nginx/deploy/](https://kubernetes.github.io/ingress-nginx/deploy/)

1、添加ingress的helm仓库

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx && helm repo update
```

2、下载ingress的helm包至本地

```sh
helm search repo ingress-nginx/ingress-nginx
helm pull ingress-nginx/ingress-nginx
```

3、更改对应的配置

```sh
tar xf ingress-nginx-3.22.0.tgz
cd ingress-nginx
vim values.yaml
```

4、需要修改的位置

- Controller和admissionWebhook的镜像地址，需要将公网镜像同步至公司内网镜像仓库（和课程不一致的版本，需要自行同步gcr镜像的，可以百度查一下使用阿里云同步gcr的镜像，也可以参考这个连接https://blog.csdn.net/weixin_39961559/article/details/80739352，或者参考这个连接： https://blog.csdn.net/sinat_35543900/article/details/103290782）
  - 镜像的digest值注释！

- hostNetwork设置为true
  - 使用宿主机的网络，部署成功后可在node节点通过`netstat -lntp | grep 80`及`ps aux | grep nginx`，可以查看到nginx/master监听的端口号和进程

- dnsPolicy设置为 ClusterFirstWithHostNet
  - ClusterFirstWithHostNet专门为了hostNetwork而设计的，否则ingress-nginx-controller无法解析service

- nodeSelector添加ingress: "true"部署至指定节点

```yaml
  nodeSelector:
    kubernetes.io/os: linux
    ingress: "true"
```

- 类型更改为 kind: DaemonSet
  - 因为DaemonSet支持部署在指定节点，Deployment没有这种机制。

- 虚拟机类型更改为 type: ClusterIP，云服务器类型更改为 type: LoadBalancer
- ingress nginx 设置为默认的ingressClass

```yaml
  ## This section refers to the creation of the IngressClass resource
  ## IngressClass resources are supported since k8s >= 1.18 and required since k8s >= 1.19
  ingressClassResource:
    name: nginx
    enabled: true
    default: true # 设置为true
    controllerValue: "k8s.io/ingress-nginx"
```

5、部署ingress

给需要部署ingress的节点上打标签

```sh
kubectl label node k8s-node01 ingress=true
kubectl create ns ingress-nginx
helm install ingress-nginx -n ingress-nginx .
```

6、将ingress controller部署至Node节点

> ingress controller不能部署在Master节点，生产环境最少三个ingress controller，并且最好是独立的Node节点
>
> 注意：访问ingress域名时，需要访问ingress-controller所在节点的主机地址，而没有部署ingress-controller的节点，无法转发。

```sh
kubectl label node k8s-node02 ingress=true
kubectl label node k8s-master03 ingress-
```

# 第十九章  集群宕机恢复

检查所有Master节点的服务状态

```bash
systemctl status containerd kube-apiserver kube-controller-manager kube-scheduler kubelet kube-proxy etcd
```

检查所有Node节点的服务状态

```bash
systemctl status containerd kubelet kube-proxy
```

重新启动Master节点自启动失败的服务

```bash
systemctl restart containerd kube-apiserver kube-controller-manager kube-scheduler kubelet kube-proxy etcd
```

重新启动Node节点自启动失败的服务

```bash
systemctl restart containerd kubelet kube-proxy
```

查看系统日志信息

```bash
tail -f /var/log/messages
```

