## 环境搭建

**CentOS7 支持M1芯片下载地址**

下载地址：

链接: https://pan.baidu.com/s/1exFPj9ACw9ARr48o3fye2g 提取码: btpd

选择的是001下载，然后解压，使用PD安装

**Centos8 支持M1芯片下载地址**

链接: https://pan.baidu.com/s/17TF3Ah3fXqf8SZshVJMvFg 提取码: 0d8v

**安装步骤（我这边选择的是CentOS7）**

- 进入界面之后，选择第一个按回车
- 然后出现了各种安装选项，只需要按以下步骤输入
  - 第一次输入r，刷新配置，按回车
  - 磁盘选择，输入5，按回车，然后根据提示，c回车，c回车，c回车，一共3次
  - 设置root密码，输入8，按回车，然后开始输入密码，一共输入两次
  - 输入b，开始安装，等着就可以了
  - 然后会显示安装已完成，按Enter键，然后回车就可以了

**安装远程连接**

- ``yum install openssh-server``，然后一直y就可以了
- 重启ssh服务，``systemctl restart sshd``，安装之后自动就启动了，如果没启动就把restart换成start
- 设置为开机启动，``systemctl enable sshd``

**配置静态IP地址**

``vi /etc/sysconfig/network-scripts/ifcfg-enp0s5``

原版

```bash
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=dhcp
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
NAME=enp0s5
UUID=9bc85b69-6ef5-4046-8315-c972f63d30bd
DEVICE=enp0s5
ONBOOT=no
```

修改后

```bash
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
NAME=enp0s5
UUID=2d1561f4-898c-4fdd-bacb-62b9217d9344
#DEVICE=enp0s5
ONBOOT=yes

IPADDR=10.211.55.102 #自定义IP
NETMASK=255.255.255.0
GATEWAY=10.211.55.2
DNS1=8.8.8.8
```

最后重启网络，``systemctl restart network.service``，如果是远程连接就会卡住，因为IP换了，使用新IP进行登录，然后输入ip a查看当前网络信息

network服务启动失败

```bash
ip a查看mac地址
记录下来
00:1c:42:42:4c:bf
00:1c:42:e1:1a:e3
00:1c:42:23:e1:f4
00:1c:42:ad:29:de
00:1c:42:9d:3b:37
vi /etc/udev/rules.d/70-persistent-net.rules
HWARR=00:1c:42:42:4c:bf
```

关闭NetworkManager

```bash
systemctl stop NetworkManager
systemctl disable NetworkManager
```

然后修改网卡信息

```bash
cd /etc/sysconfig/network-scripts/
mv ifcfg-enp0s5 ifcfg-eth0
vi ifcfg-eth0,修改里面DEVICE=eth0, 把注释去掉
vi /etc/sysconfig/grub，在quiet后面添加net.ifnames=0 biosdevname=0
```

生成菜单

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg 
```

然后reboot重启，静态IP配置成功

**修改主机名**

- ``vi /etc/hostname``，输入主机名
- ``vi /etc/sysconfig/network`，添加IP与hostname的对应关系
  - 例如：10.211.55.11 k8s-master01
- 重启reboot

**安装一些必要工具**

``yum install wget -y``

**查找替换**

`` :%s/vivian/sky/g    #（等同于 :g/vivian/s//sky/g） 替换每一行中所有 vivian 为 sky``

**所有节点配置host**

首先输入``vi /etc/hosts/``

```bash
10.211.55.11 k8s-master01
10.211.55.12 k8s-master02
10.211.55.13 k8s-master03
10.211.55.236 k8s-master-lb # 如果不是高可用集群，该IP为Master01的IP
10.211.55.101 k8s-node01
10.211.55.102 k8s-node02
```

配置yum源

```bash
# 可以使用自带的repo, 不用替换源
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo

yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo
```

安装中科大的arm64架构的yum源

```bash
# CentOS-Base.repo
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for CentOS updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the
# remarked out baseurl= line instead.
#
#
 
[base]
name=CentOS-$releasever - Base
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os
baseurl=http://mirrors.ustc.edu.cn/centos-altarch/$releasever/os/$basearch/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
       file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-AltArch-Arm32
 
#released updates
[updates]
name=CentOS-$releasever - Updates
# mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates
baseurl=http://mirrors.ustc.edu.cn/centos-altarch/$releasever/updates/$basearch/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
       file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-AltArch-Arm32
 
#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras
# mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras
baseurl=http://mirrors.ustc.edu.cn/centos-altarch/$releasever/extras/$basearch/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
       file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-AltArch-Arm32
 
#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus
# mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=centosplus
baseurl=http://mirrors.ustc.edu.cn/centos-altarch/$releasever/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
       file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-AltArch-Arm32
```



安装必备工具

```bash
yum install wget jq psmisc vim net-tools telnet yum-utils device-mapper-persistent-data lvm2 git -y
```

所有节点关闭防火墙、selinux、dnsmasq、swap

```bash
systemctl disable --now firewalld 
# systemctl disable --now dnsmasq 没用
# 下面搭完就崩
systemctl disable --now NetworkManager

setenforce 0
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/sysconfig/selinux
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config
```

关闭swap分区

```bash
swapoff -a && sysctl -w vm.swappiness=0
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab
```

安装ntpdate

```bash
#rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm #我的加了repo就报错了
yum install ntpdate -y
```

所有节点同步时间

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo 'Asia/Shanghai' >/etc/timezone
ntpdate time2.aliyun.com
# 加入到crontab
输入crontab -e，然后复制下面的进去，并保存
*/5 * * * * /usr/sbin/ntpdate time2.aliyun.com
```

所有节点配置limit

```bash
ulimit -SHn 65535
```

```bash
vim /etc/security/limits.conf
# 末尾添加如下内容
* soft nofile 65536
* hard nofile 131072
* soft nproc 65535
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
```

在Master01上设置免密钥登录其他节点

```
ssh-keygen -t rsa

for i in k8s-master01 k8s-master02 k8s-master03 k8s-node01 k8s-node02;do ssh-copy-id -i /root/.ssh/id_rsa.pub $i;done
```

下载安装所有的源码文件

```bash
cd /root/ ; git clone https://github.com/dotbalo/k8s-ha-install.git
```

如果内核不是4.19+的话需要升级内核，我这边是5.11版本的，就不升级了，直接开始下一步操作

所有节点安装ipvsadm

```bash
yum install ipvsadm ipset sysstat conntrack libseccomp -y
```

所有节点配置ipvs模块，在内核4.19+版本nf_conntrack_ipv4已经改为nf_conntrack， 4.18以下使用nf_conntrack_ipv4即可

```
vim /etc/modules-load.d/ipvs.conf
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

然后执行``systemctl enable --now systemd-modules-load.service``

开启一些k8s集群中必须的内核参数，所有节点都配置k8s内核

```bash
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

sysctl --system
```

所有节点配置完内核后，重启，保证重启后内核依旧加载

```bash
reboot
lsmod | grep --color=auto -e ip_vs -e nf_conntrack
```

## kubeadm安装

### 基本组件安装

所有节点安装Docker-ce 19.03

```bash
yum install docker-ce-19.03.* docker-cli-19.03.* -y
```

由于新版kubelet建议使用systemd，所以可以把docker的CgroupDriver改成systemd

```bash
mkdir /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
```

设置所有节点开机自启动

```bash
systemctl daemon-reload && systemctl enable --now docker
```

设置k8s源,arm64

```bash
vim /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-aarch64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kube*
```

yum安装k8s

```bash
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

设置kubelet开机自启动

```bash
systemctl daemon-reload
systemctl enable --now kubelet
```

### 高可用组件安装

注意：如果不是高可用集群，haproxy和keepalived无需安装

- 公有云要用公有云自带的负载均衡，比如阿里云的SLB，腾讯云的ELB，用来替代haproxy和keepalived，因为公有云大部分都是不支持keepalived的，另外如果用阿里云的话，kubectl控制端不能放在master节点，推荐使用腾讯云，因为阿里云的slb有回环的问题，也就是slb代理的服务器不能反向访问SLB，但是腾讯云修复了这个问题。

所有Master节点通过yum安装HAProxy和KeepAlived：

```bash
yum install keepalived haproxy -y
```

所有Master节点配置HAProxy（详细配置参考HAProxy文档，所有Master节点的HAProxy配置相同）

先执行这个，防止报错

```
setsebool -P haproxy_connect_any=1 
```

**删除所有内容：``ggdG``**，gg是回到首行

```bash
mkdir /etc/haproxy
vim /etc/haproxy/haproxy.cfg 
替换下面内容
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

frontend monitor-in
  bind *:33305
  mode http
  option httplog
  monitor-uri /monitor

frontend k8s-master
  bind 0.0.0.0:16443
  bind 127.0.0.1:16443
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
  server k8s-master01	10.211.55.11:6443  check
  server k8s-master02	10.211.55.12:6443  check
  server k8s-master03	10.211.55.13:6443  check
```

所有Master节点配置KeepAlived，配置不一样

``vim /etc/keepalived/keepalived.conf ``，注意每个节点的IP和网卡（interface参数）

Master01

```bash
mkdir /etc/keepalived
vim /etc/keepalived/keepalived.conf 
替换下面内容
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
script_user root
    enable_script_security
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
    mcast_src_ip 10.211.55.11
    virtual_router_id 51
    priority 101
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        10.211.55.236
    }
    track_script {
       chk_apiserver
    }
}
```

Master02

```bash
mkdir /etc/keepalived
vim /etc/keepalived/keepalived.conf 
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
script_user root
    enable_script_security
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
    mcast_src_ip 10.211.55.12
    virtual_router_id 51
    priority 100
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        10.211.55.236
    }
    track_script {
       chk_apiserver
    }
}
```

Master03

```bash
mkdir /etc/keepalived
vim /etc/keepalived/keepalived.conf 
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
script_user root
    enable_script_security
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
    mcast_src_ip 10.211.55.13
    virtual_router_id 51
    priority 100
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        10.211.55.236
    }
    track_script {
       chk_apiserver
    }
}
```

所有master节点配置KeepAlived健康检查文件：

```bash
[root@k8s-master01 keepalived]# vim /etc/keepalived/check_apiserver.sh 
#!/bin/bash

err=0
for k in $(seq 1 3)
do
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

赋予权限

```bash
chmod +x /etc/keepalived/check_apiserver.sh
```

启动haproxy和keepalived

```bash
systemctl daemon-reload
systemctl enable --now haproxy
systemctl enable --now keepalived
```

查看端口起来没

```bash
netstat -lntp
```

### 集群初始化

官方初始化文档：

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/

Master01节点创建kubeadm-config.yaml配置文件如下：

**（# 如果不是高可用集群，10.211.55.236:16443改为master01的地址，16443改为apiserver的端口，默认是6443，注意更改v1.18.5自己服务器kubeadm的版本：kubeadm version）**

**宿主机网段、podSubnet网段、serviceSubnet网段不能重复**

Master01：

``vi kubeadm-config.yaml``

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: 7t2weq.bjbawausm0jaxury
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.211.55.11
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-master01
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  certSANs:
  - 10.211.55.236
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: 10.211.55.236:16443
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.23.1
networking:
  dnsDomain: cluster.local
  podSubnet: 172.168.0.0/12
  serviceSubnet: 192.168.0.0/12
scheduler: {}
```

更新kubeadm文件

```yaml
kubeadm config migrate --old-config kubeadm-config.yaml --new-config new.yaml
```

将new.yaml文件复制到其他master节点，之后所有Master节点提前下载镜像，可以节省初始化时间：

``kubeadm config images pull --config /root/new.yaml ``

Master01节点初始化，初始化以后会在/etc/kubernetes目录下生成对应的证书和配置文件，之后其他Master节点加入Master01即可：

``kubeadm init --config /root/new.yaml --upload-certs``

如果初始化失败，重置后再次初始化，命令如下：

``kubeadm reset -f ; ipvsadm --clear ; rm -rf ~/.kube``

## 二进制安装

所有节点安装基本工具

```bash
yum install wget jq psmisc vim net-tools yum-utils device-mapper-persistent-data lvm2 git -y
```

Master01下载安装文件

```bash
 cd /root/ ; git clone https://github.com/dotbalo/k8s-ha-install.git
```

配置内核

```bash
vim /etc/modules-load.d/ipvs.conf 
# 加入以下内容
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

然后执行``systemctl enable --now systemd-modules-load.service``即可

配置k8s内核

```bash
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
sysctl --system
```

#### 基本组件安装

所有节点安装Docker-ce 19.03

```bash
yum install docker-ce-19.03.* -y
```

由于新版kubelet建议使用systemd，所以可以把docker的CgroupDriver改成systemd

```bash
mkdir /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
```

所有节点设置开机自启动Docker：

```bash
systemctl daemon-reload && systemctl enable --now docker
```

Master01下载kubernetes安装包

官方下载地址

https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.24.md

```bash
wget https://dl.k8s.io/v1.24.0-alpha.1/kubernetes-server-linux-arm64.tar.gz
```

下载etcd

```bash
wget https://github.com/etcd-io/etcd/releases/download/v3.4.13/etcd-v3.4.13-linux-arm64.tar.gz
```

解压kubernetes安装文件

```bash
tar -xf kubernetes-server-linux-arm64.tar.gz  --strip-components=3 -C /usr/local/bin kubernetes/server/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy}
```

解压etcd安装文件

```bash
tar -zxvf etcd-v3.4.13-linux-arm64.tar.gz --strip-components=1 -C /usr/local/bin etcd-v3.4.13-linux-arm64/etcd{,ctl}
```

版本查看

```bash
kubelet --version
etcdctl version
```

将组件发送到其他节点

```bash
MasterNodes='k8s-master02 k8s-master03'
WorkNodes='k8s-node01 k8s-node02'
for NODE in $MasterNodes; do echo $NODE; scp /usr/local/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy} $NODE:/usr/local/bin/; scp /usr/local/bin/etcd* $NODE:/usr/local/bin/; done
for NODE in $WorkNodes; do     scp /usr/local/bin/kube{let,-proxy} $NODE:/usr/local/bin/ ; done
```

所有节点创建/opt/cni/bin目录

```bash
mkdir -p /opt/cni/bin
```

Master01切换到1.20.x分支（其他版本可以切换到其他分支）

查看分支``git branch -a ``

```bash
cd k8s-ha-install && git checkout manual-installation-v1.23.x
```

### 生成证书

Master01下载生成证书工具

```
wget "https://pkg.cfssl.org/R1.2/cfssl_linux-amd64" -O /usr/local/bin/cfssl
wget "https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64" -O /usr/local/bin/cfssljson
chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson
```

**etcd证书**

所有Master节点创建etcd证书目录

```bash
mkdir /etc/etcd/ssl -p
```

所有节点创建kubernetes相关目录

```bash
mkdir -p /etc/kubernetes/pki
```

Master01节点生成etcd证书

生成证书的CSR文件：证书签名请求文件，配置了一些域名、公司、单位

```bash
cd /root/k8s-ha-install/pki

# 生成etcd CA证书和CA证书的key
cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare /etc/etcd/ssl/etcd-ca
```

```bash
cfssl gencert \
   -ca=/etc/etcd/ssl/etcd-ca.pem \
   -ca-key=/etc/etcd/ssl/etcd-ca-key.pem \
   -config=ca-config.json \
   -hostname=127.0.0.1,k8s-master01,k8s-master02,k8s-master03,10.211.55.11,10.211.55.12,10.211.55.13 \
   -profile=kubernetes \
   etcd-csr.json | cfssljson -bare /etc/etcd/ssl/etcd
```

# ARM64狗都不用！

# 不多bb 云服务器启动！

### 开始搭建

服务器采用的是腾讯云，2核4G，Centos7.6

IP地址，因为经济跟不上，所以他们不在同一个网段，分三个号买的

```bash
# Master01
124.223.83.116  10.0.4.13
# Master02
124.223.119.35  10.0.12.5
# Master03
124.223.83.132  10.0.4.10
# Node01
8.140.155.94    172.28.38.65/20
```

话不多说，直接开装

#### 升级内核

查看当前内核版本三种方法

```bash
uname -r
uname -a
cat /etc/redhat-release
```

更新yum源仓库

```bash
yum -y update
```

启用ELRepo仓库

导入ELRepo仓库的公共密钥

```bash
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
```

安装ELRepo仓库的yum源

```bash
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
```

查看可用的系统内核包

```bash
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
```

删除所有旧内核相关包

```bash
yum remove kernel-devel kernel-tools-libs kernel-tools kernel-headers
```

安装最新版本内核

```bash
yum --enablerepo=elrepo-kernel install kernel-ml -y
```

安装新内核相关软件

```bash
yum --disablerepo=\* --enablerepo=elrepo-kernel install -y  kernel-ml-devel kernel-ml-tools kernel-ml-tools-libs kernel-ml-tools-libs-devel kernel-ml-headers
```

设置新内核为默认内核

```bash
grub2-set-default 0
```

生成grub配置文件

```bash
grub2-mkconfig -o /etc/grub2.cfg
```

查看系统上的所有可用内核

```bash
sudo awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
```

重启让配置生效

```bash
reboot
```

查看内核

```bash
uname -r
```

安装更新旧内核时被卸载的软件

```bash
yum install -y compat-glibc compat-glibc-headers gcc gcc-c++ gcc-gfortran glibc-devel glibc-headers libquadmath-devel libtool systemtap systemtap-devel
```

#### 初始化配

```bash
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld
 
# 关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
 
# 关闭swap
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久

# 根据规划设置主机名  hostname:名称，方便记忆
hostnamectl set-hostname hostname
 
# 在master添加hosts
cat >> /etc/hosts << EOF
124.223.83.116 k8s-master01 #k8s-master01->上文中的hostname
124.223.119.35 k8s-master02 #同理
124.223.83.132 k8s-master03 #同理
8.140.155.94 k8s-node01
EOF

modprobe br_netfilter
 
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack #内核版本小于4.19 nf_conntrack_ipv4
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack

#云服务器一般不需要
# 设置系统时区为 中国/上海
timedatectl set-timezone Asia/Shanghai
# 将当前的UTC时间写入硬件时钟
timedatectl set-local-rtc 0
# 重启依赖于系统时间的服务
systemctl restart rsyslog
systemctl restart crond
```

#### 添加阿里云yum软件源

```bash
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

#### 安装kubeadm，kubelet和kubectl

```bash
yum install -y kubelet-1.23.0 kubeadm-1.23.0 kubectl-1.23.0
systemctl enable kubelet
#查看版本信息
kubectl version
kubeadm version
kubelet --version
```

#### 安装docker

如果有旧的版本，移除

```bash
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
```

安装一些必要的系统工具

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
```

添加软件源信息

```bash
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo	    
```

更新yum缓存

```bash
yum makecache fast
```

安装Docker-ce

```bash
yum -y install docker-ce
```

启动Docker后台服务

```bash
systemctl start docker
```

测试hello-world

```bash
docker run hello-world
```

镜像加速配置

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://b58tfe63.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

#### 配置docker相应镜像

```bash
vi pullimages.sh
```

```bash
#!/bin/bash
#pull images

ver=v1.23.0
registry=registry.cn-hangzhou.aliyuncs.com/google_containers
images=`kubeadm config images list --kubernetes-version=$ver |awk -F '/' '{print $2}'`

for image in $images
do
if [ $image != coredns ];then
    docker pull ${registry}/$image
    if [ $? -eq 0 ];then
        docker tag ${registry}/$image k8s.gcr.io/$image
        docker rmi ${registry}/$image
    else
        echo "ERROR: 下载镜像报错，$image"
    fi
else
    docker pull coredns/coredns:1.8.6
    docker tag coredns/coredns:1.8.6  k8s.gcr.io/coredns/coredns:v1.8.6
    docker rmi coredns/coredns:1.8.6
fi
done
```

```bash
chmod +x pullimages.sh && ./pullimages.sh
```

#### 建立虚拟网卡（所有节点）

```bash
124.223.83.116 k8s-master01 #k8s-master01->上文中的hostname
124.223.119.35 k8s-master02 #同理
124.223.83.132 k8s-master03 #同理
8.140.155.94 k8s-node01
```

```bash
cat > /etc/sysconfig/network-scripts/ifcfg-eth0:1 <<EOF
BOOTPROTO=static
DEVICE=eth0:1
IPADDR=8.140.155.94 #你的公网IP
PREFIX=32
TYPE=Ethernet
USERCTL=no
ONBOOT=yes
EOF
```

#### 修改kubelet启动参数

```bash
vim /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf

[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS --node-ip=8.140.155.94
```

#### 使用kubeadm初始化主节点

```bash
执行上文中下载docker镜像的脚本

# step1 添加配置文件，注意替换下面的IP
cat > kubeadm-config.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.23.0
apiServer:
  certSANs:    
  - k8s-master01    #请替换为hostname
  - 124.223.83.116   #请替换为公网
  - 124.223.83.116  
  - 10.96.0.1   #不要替换，此IP是API的集群地址，部分服务会用到
controlPlaneEndpoint: 124.223.83.116:6443 #替换为公网IP
networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
--- 将默认调度方式改为ipvs
apiVersion: kubeproxy-config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
featureGates:
  SupportIPVSProxyMode: true
mode: ipvs
EOF

# step2 初始化 如果是1核心或者1G内存的请在末尾添加参数（--ignore-preflight-errors=all），否则会初始化失败
kubeadm init --config=kubeadm-config.yaml --ignore-preflight-errors=all
```

如果初始化失败，重置后再次初始化，命令如下：

``kubeadm reset -f ; ipvsadm --clear ; rm -rf ~/.kube``

**云服务器初始化失败，先去把所有端口打开，在云服务器官网上！！！**

**如果kubeadm init的时候kubelet不能正常初始化**

```bash
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp 127.0.0.1:10248: connect: connection refused.
```

kubelet起不来,根据[StackOverflow上这个帖子](https://stackoverflow.com/questions/52119985/kubeadm-init-shows-kubelet-isnt-running-or-healthy)的描述, 应该是安装kubeadm和docker后，这俩使用的cgroup驱动不一致导致。需要在指定docker的cgroup驱动为system. 具体做法为:

```bash
vim /etc/docker/daemon.json
```

```bash
{
        "exec-opts": ["native.cgroupdriver=systemd"],
        "log-driver": "json-file",
        "log-opts": {
                "max-size": "100m"
        }
}
```

保存之后执行

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

kubelet也要搞一下

```bash
sudo vi /var/lib/kubelet/config.yaml
# 在最后加入
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

然后重启一下kubelet

```bash
sudo systemctl restart kubelet 
```

#### 初始化完成之后

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
# 生成kubeconfig配置文件，用于请求api服务器
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:
# 这个是master节点使用
  kubeadm join 124.223.83.116:6443 --token xg18yw.vb83u52ycc03s46u \
        --discovery-token-ca-cert-hash sha256:c50e5c9ab05a24cffc1756bd47ec4e617279244840d69a46a287071a5dc61f92 \
        --control-plane 

Then you can join any number of worker nodes by running the following on each as root:
# 这个是node节点使用
kubeadm join 124.223.83.116:6443 --token xg18yw.vb83u52ycc03s46u \
        --discovery-token-ca-cert-hash sha256:c50e5c9ab05a24cffc1756bd47ec4e617279244840d69a46a287071a5dc61f92 
```

#### 修改kube-apiserver参数（主节点）

```bash
vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 124.223.83.116:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=124.223.83.116
    - --bind-address=0.0.0.0
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --insecure-port=0
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --secure-port=6443
    - --service-account-issuer=https://kubernetes.default.svc.cluster.local
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    image: k8s.gcr.io/kube-apiserver:v1.23.0
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 124.223.83.116
        path: /livez
        port: 6443
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    name: kube-apiserver
    readinessProbe:
      failureThreshold: 3
      httpGet:
        host: 124.223.83.116
        path: /readyz
        port: 6443
        scheme: HTTPS
      periodSeconds: 1
      timeoutSeconds: 15
    resources:
      requests:
        cpu: 250m
    startupProbe:
      failureThreshold: 24
      httpGet:
        host: 124.223.83.116
        path: /livez
        port: 6443
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: ca-certs
      readOnly: true
    - mountPath: /etc/pki
      name: etc-pki
      readOnly: true
    - mountPath: /etc/kubernetes/pki
      name: k8s-certs
      readOnly: true
  hostNetwork: true
  priorityClassName: system-node-critical
  volumes:
  - hostPath:
      path: /etc/ssl/certs
      type: DirectoryOrCreate
    name: ca-certs
  - hostPath:
      path: /etc/pki
      type: DirectoryOrCreate
    name: etc-pki
  - hostPath:
      path: /etc/kubernetes/pki
      type: DirectoryOrCreate
    name: k8s-certs
status: {}
```

Node01加入集群

```bash
[root@louis ~]# kubeadm join 124.223.83.116:6443 --token xg18yw.vb83u52ycc03s46u \
>         --discovery-token-ca-cert-hash sha256:c50e5c9ab05a24cffc1756bd47ec4e617279244840d69a46a287071a5dc61f92
[preflight] Running pre-flight checks
        [WARNING Service-Docker]: docker service is not enabled, please run 'systemctl enable docker.service'
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

**报错**

```bash
root@louis ~]# kubectl get nodes -o wide
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

解决方案

- 将主节点中的【/etc/kubernetes/admin.conf】文件拷贝到从节点相同目录下，然后配置环境变量

```bash
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile
```

