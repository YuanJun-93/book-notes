## 第一部分 Kubernetes基础

### 第二章 容器技术基础

#### 2.1 从进程开始说起

Namespace的使用方式

- 其实是Linux创建新进程的一个可选参数
- 创建进程的系统调用是clone
  - ``int pid = clone(main_function, stack_size, SIGCHLD, NULL);``
  - 这个系统调用会创建一个新的进程，返回PID
- 在用clone()系统调用创建一个新进程时，可以在参数重指定CLONE_NEWPID
  - ``int pid = clone(main_function, stack_size, CLONE_NEWPID | SIGCHLD, NULL)``
  - 在这个新创建的进程中会看到一个全新的进程空间。它的PID是1
  - 但是在宿主机真实的进程空间中，这个进程的PID还是真实的数值，比如100
- 可以多次执行clone()调用，这样就会创建多个PID Namespace，每个Namespace中的应用进程都会认为自己是当前容器的1号进程，看不到其他PID Namespace的情况

**Docker实际上是在创建进程的时候，为它们加上了各种各样的Namespace参数**

#### 2.2 隔离与限制

Namespace技术实际上修改了应用进程看待整个计算机“视图”的视野

**对宿主机来说，这些被隔离的进程跟其他进程并没有太大区别**

在Linux内核中，有很多资源和对象是不能被Namespace化的，最典型的例子：时间

**Linux Cgroups（Linux control groups）是Linux内核中用来为进程设置资源限制的一个重要功能**

- 限制一个进程组能够使用的资源上限，包括CPU、内存、磁盘、网络带宽等
- 还能对进程进行优先级设置、审计，以及将进程挂起和恢复等操作

Cgroups向用户暴露出来的操作接口是文件系统，它以文件和目录的方式组织在操作系统的/sys/fs/cgroup路径下，用mount指令显示，``mount -t cgroup``

**在/sys/fs/cgroup下面有很多cpuset、cpu、memory这样的子目录，也叫子系统**

```shell
root@louis:/sys/fs/cgroup/cpu# mkdir container
root@louis:/sys/fs/cgroup/cpu# ls container/
cgroup.clone_children  cpuacct.usage_all          cpuacct.usage_sys   cpu.shares      notify_on_release
cgroup.procs           cpuacct.usage_percpu       cpuacct.usage_user  cpu.stat        tasks
cpuacct.stat           cpuacct.usage_percpu_sys   cpu.cfs_period_us   cpu.uclamp.max
cpuacct.usage          cpuacct.usage_percpu_user  cpu.cfs_quota_us    cpu.uclamp.min
# 上面这些目录称为控制组
# 后台执行一个死循环礁本，把CPU占满
root@louis:/sys/fs/cgroup/cpu# while : ; do : ; done &
[1] 12071

#当前container控制组的CPU quota还没有限制，CPU period也是默认的
root@louis:/sys/fs/cgroup/cpu/container# cat /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us 
-1
root@louis:/sys/fs/cgroup/cpu/container# cat /sys/fs/cgroup/cpu/container/cpu.cfs_period_us 
100000

# 修改内容，来做限制，向container组里的cfs_quota文件写入20ms（20000us）
# 意味着，100ms内，该控制组的进程只能使用20ms的CPU时间，即只能使用20%CPU带宽
echo 20000 > /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us 

# 把被限制的进程PID写入
echo 12071 > /sys/fs/cgroup/cpu/container/tasks

top
   PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                               
  12071 root      20   0   10628   1476      0 R  13.3   0.1  14:05.29 bash     
```

除了CPU子系统外，Cgroups的每一项子系统都有其独有的资源限制能力

- blkio：为块设备设定I/O限制，一般用于磁盘等设备
- cpuset：为进程分配单独的CPU核和对应的内存节点
- memory：为进程设定内存使用限制

Cgroup简单的说，就是一个子系统目录加上一组资源限制文件的组合

- 对于Docker等Linux容器来说，只需要在每个子系统下面为每个容器创建一个控制组（新目录）
- 然后在启动容器进程之后，把这个进程的PID填写到对应控制组的tasks文件中即可

```shell
docker run -it --cpu-period=100000 --cpu-quota=20000 ubuntu /bin/bash

root@louis:/sys/fs/cgroup/cpu/docker# cat 757b1d3a5999b83aad03c5f6aea2a072b9ead894536aeb30e9fbb84128761e56/cpu.cfs_period_us 
100000
```

**容器是一个“单进程”的模型**

Linux下的/proc目录存储的是记录当前内核运行状态的一系列特殊文件，用户可以通过访问这些文件查看系统以及当前正在运行的进程的信息，比如CPU使用情况、内存占用率等，**这也是top指令查看系统信息的主要数据来源**

如果在容器中执行top，会返回宿主机的CPU和内存数据，因为/proc文件系统并不知道用户通过Cgroups给这个容器做了怎样的资源限制

#### 2.3 深入理解容器镜像

```shell
mount("none", "/tmp", "tmpfs", 0, "")
```

- 容器以tmpfs（内存盘）格式重新挂载/tmp目录

- 检查挂载 ``mount -l | grep tmpfs``

在Linux操作系统中，有一个chroot的命令可以帮你在shell中方便地完成这项工作，即改变进程的根目录到指定位置

假设有一个$HOME/test目录，把他作为一个/bin/bash进程的根目录

首先创建test目录和几个lib文件夹

```shell
mkdir -p $HOME/test
mkdir -p $HOME/test/{bin,lib64,lib}
cd $T
```

把bash命令复制到test目录对应的bin路径下

```shell
cp -v /bin/{bash,ls} $HOME/test/bin
```

把bash命令需要的所有so文件也复制到test目录对应的lib路径下，so文件用ldd命令找到

```shell
T=$HOME/test
list="$(ldd /bin/ls | egrep -o '/lib.*\.[0-9]')"
for i in $list; do cp -v "$i" "${T}${i}"; done
```

最后执行chroot命令，告诉操作系统将使用$HOME/test目录作为/bin/bash进程的根目录

``chroot $HOME/test /bin/bash``

**Mount Namespace基于对chroot的不断改良才被发现出来，这是Linux操作系统里第一个Namespace**

Docker在镜像的设计中引入了层的概念，用户制作镜像的每一步操作都会生成一个层，也就是一个增量rootfs

- 用到 一种UnionFS（union file system，联合文件系统）的能力

- 例如通过联合挂载，将两个目录挂到C上

  - ```shell
    mkdir C
    mount -t aufs -o dirs=./A:./B none ./C
    ```

  - 合并后的目录C中，有a,b,x 这三个文件，并且x文件只有一份，本来A和B都有一个x的，现在如果在C中修改了x，A和B都会生效

- docker使用AuFS这个UnionFS的实现

**所谓的镜像，实际上是一个Ubuntu操作系统的rootfs，它的内容是Ubuntu操作系统的所有文件和目录**

```shell
"RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:9f54eef412758095c8079ac465d494a2872e02e90bf1fb5f12a1641c0d1bb78b"
            ]
        },
```

在使用镜像的时候，Docker会把这些增量联合挂载在一个统一的挂载点上

- ``var/lib/docker/aufs/mnt/<ID>``

查看挂载信息可以找到这个目录对应的AuFS的内部ID

- ``cat /proc/mounts | grep aufs``
- 然后使用这个ID，在/sys/fs/aufs查看被联合挂载在一起的各个层的信息

1. 只读层
   - 最下面的几层
   - 以增量的方式分别包含了Ubuntu操作系统的一部分
2. 可读写层
   - 最上面的一层，挂载方式rw，在写入文件之前，目录是空的
   - 一旦出现了写操作，修改产生的内容会以增量的方式出现在该层中
   - 如果要删除一个名为foo的文件，实际上是在可读写层创建了一个名为.wh.foo的文件，然后这两个层联合挂载后，foo文件就会被.wh.foo文件覆盖，从而消失。**这个功能是ro+wh的挂载方式，只读+whiteout，whiteout称为白障**

3. Init层
   - Init层是一个以-init结尾的层，夹在只读层和可读写层之间
   - 是Docker项目单独生成的一个内部层，专门存放/etc/hosts、/etc/resolv.conf等信息
   - 需要这一层的原因是：这些文件本来属于自读的Ubuntu镜像的一部分，但是用户需要在启动容器时写一些指定的值（比如hostname），所以需要在可读写层修改它们
     - 重要的是这些修改，只对当前容器有效，就很麻烦
     - 所以在修改了这些文件之后以一个单独的层挂载出来
     - 用户提交docker commit的时候，只会提交可读写层

**最终这些层都会被联合挂载到/var/lib/docker/aufs/mnt目录下，表现为一个完整的Ubuntu系统**

#### 2.4 重新认识Linux容器

docker exec

- 一个进程的每种Linux Namespace都在它对应的/rpoc/[进程号]
- 一个进程可以选择加入进程已有的某个Namespace当中，从而“进入”该容器所在容器
- 该操作依赖的是setns()的Linux系统调用
  - ``sets(fd, 0)`` 把文件描述符fd交给setns使用
  - 执行之后，当前进程就加入这个文件对应的Linux Namespace中了

**Volume（数据卷）**

- Volume机制允许你将宿主机上指定的目录或者文件挂载到容器中进行读取和修改
- 两种声明方式
  - ``docker run -v /test ...``
  - ``docker run -v /home:/test ...``

**只需要在rootfs准备好后，在执行chroot之前，把Volume指定的宿主机目录（比如/home目录）挂载到指定的容器目录（比如/test目录）在宿主机上对应的目录（/var/lib/docker/aufs/mnt/[可读写层ID]/test）上，这个Volume的挂载工作就完成了**

用到的挂载技术是Linux的绑定挂载（bind mount）机制。

- 允许你将一个目录或文件，而不是整个设备挂载到指定目录上
- 在该挂载点上进行的任何操作，只发生在被挂载的目录或文件上
- 原挂载点的内容会被隐藏起来，且不受影响

**绑定挂载实际上是一个inode替换的过程**

- inode可以理解为存放文件内容的“对象”
- dentry（目录项）就是访问这个inode所使用的“指针”

宿主机的临时工作目录 ``ls /var/lib/docker/volumes/volume ID/_data/``

``_data``这个文件夹是这个容器的Volume在宿主机上对应的临时目录

在容器的Volume中添加一个文件text.txt

```bash
docker exec -it cf53b766fa6f /bin/sh
cd test/
touch text.txt
```

宿主机上可以看到对应的text.txt

如果在宿主机上查看该容器的可读写层，虽然可以看到/test目录，但是内容是空的

```bash
ls /var/lib/docker/aufs/mnt/6780d0778b8a/test
```

**所以，容器Volume的信息并不会被docker commit提交，但是这个挂载点目录/test会出现在新的镜像当中**

### 第三章 Kubernetes设计与架构

#### 3.1 Kubernetes核心设计与架构

一个正在运行的Linux容器，可以被一分为二的看待

- 一组联合挂载在``/var/lib/docker/aufs/mnt``上的rootfs，称为容器镜像(container image)，是容器的静态视图
- 一个由Namespace+Cgroups构成的隔离环境，这一部分称为容器运行时(container runtime)，是容器的动态视图

Kubernetes项目仅仅把Docker作为最底层的一种容器运行时的实现

Kubernetes项目最主要的设计思想，以统一的方式抽象底层基础设施能力（比如计算、存储、网络），定义任务编排的各种关系（比如亲密关系、访问关系、代理关系），将这些抽象以声明式API的方式对外暴露，从而允许平台构建者基于这些抽象进一步构建自己的PaaS仍至任何上层平台。

- 即一个用来帮助用户构建上层平台的基础平台

**Pod是Kubernetes项目中最基础的一个对象，源自于谷歌Borg论文中Alloc的设计**

如果两个不同的Pod之间不仅有访问关系，还需要发起加上授权信息

- Kubernetes项目提供了Secret对象，保存在etcd里的键值对数据
- 这样Kubernetes就会在指定的Pod启动时，自动把Secret里的数据以Volume的方式挂载到容器

在Kubernetes项目中，推崇的使用方法

- 首先，通过一个任务编排对象，比如Pod、Job、CronJob等，描述试图管理等应用
- 然后，为它定义一些运维能力对象，比如Service、Ingress、Horizontal Pod Autoscaler（自动水平扩展器）等，这些对象会负责具体的运维能力侧功能

**使用这种方法就是所谓的声明式API**，这个事Kubernetes最核心的设计理念

### 第四章 Kubernetes集群搭建与配置

#### 4.1 Kubernetes部署利器：kubeadm

##### 4.1.1 kubeadm的工作原理

**为什么不用docker run去启动这些组件容器？**

- 早期版本确实有一个脚本是用Docker部署的，但是有一个很麻烦的问题，如何容器化kubelet
- kubelet是用来操作容器运行时的核心组件，但是kubelet在操作的时候都需要直接操作宿主机
- 如果kubelet直接在一个容器里面运行的话，那么就会变得很麻烦

**不推荐用容器部署kubelet，直接在宿主机上运行kubelet，然后使用容器部署其他Kubernetes组件**

需要替换源，然后再update

```bash
deb http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ bionic-updates main restricted universe multiverse
#deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ bionic-updates main restricted universe multiverse
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ bionic-security main restricted universe multiverse
#deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ bionic-security main restricted universe multiverse
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ bionic-backports main restricted universe multiverse
#deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ bionic-backports main restricted universe multiverse
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ bionic main universe restricted
#deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ bionic main universe restricted
```

**如果没有安装docker，提前安装**

``sudo apt install docker.io ``

**再安装Kubectl**

``sudo apt-get install kubectl``

**如果需要安装kubelet，得提前安装**

```bash
# 顺着安装

# 再安装iptables
sudo apt-get install iptables

sudo apt-get install libxtables12=1.6.1-2ubuntu2

sudo apt autoremove

# 先得安装ebtables
sudo apt-get install ebtables

# 最后安装kubelet
sudo apt-get install kubelet
```

安装第一步：

``apt-get install kubeadm``

如果安装不成功，就替换成阿里的源，然后再安装

```bash
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-xenial main
EOF
```

添加源之后，使用 apt-get update 命令会出现错误，原因是缺少相应的key，可以通过下面命令添加(E084DAB9 为上面报错的key后8位)：

```bash
gpg --keyserver keyserver.ubuntu.com --recv-keys E084DAB9
gpg --export --armor E084DAB9 | sudo apt-key add -
```

**关闭swap**

```bash
sudo swapoff -a #暂时关闭
echo "vm.swappiness = 0">> /etc/sysctl.conf #永久关闭
```

**获取镜像列表**

```bash
kubeadm config images list
```

```bash
images=(  # 下面的镜像应该去除"k8s.gcr.io/"的前缀，版本换成上面获取到的版本
    kube-apiserver:v1.23.1
    kube-controller-manager:v1.23.1
    kube-scheduler:v1.23.1
    kube-proxy:v1.23.1
    pause:3.6
    etcd:3.5.1-0
    coredns:1.8.6
)

for imageName in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
```

执行的两种方法，bash shell脚本或者给权限然后./脚本执行

```bash
vi imagelist.sh
# 然后复制进来
sudo chmod a+x imagelist.sh
sudo bash ./imagelist.sh
```

**kubeadm init**

初始化之前会做预检，判断机器是否符合部署Kubernetes的要求

```bash
kubeadm init --image-repository=registry.aliyuncs.com/google_containers
```

Kubernetes对外提供服务的时候，除非专门开启"非安全模式"，否则都要通过HTTPS才能访问kube-apiserver

- 为Kubernetes集群配置证书文件
- kubeadm为Kubernetes项目生成的证书文件都在``Master``节点的``/etc/kubernetes/pki``目录下，最主要的是ca.crt和对应的私钥ca.key

用户使用kubectl获取容器日志等streaming操作时，需要通过kube-apiserver向kubelet发起请求，这个连接也必须是安全的

- kubeadm为此生成的是apiserver-kubelet-client.crt文件
- 对应的私钥是apiserver-kubelet-client.key

**可以不让kebeadm生成这些证书**

- 只需要将证书拷贝到``/etc/kubernetes/pki/ca.{crt,key}``

kubeadm为其他组件生成访问kebe-apiserver所需的配置文件

- 路径``etc/kubernetes/xxx.conf``
- 记录了当前这个Master节点的服务器地址、监听端口、证书目录等信息，对应的客户端（比如scheduler、kubelet等）可以直接加载对应的文件，使用其中的信息与kube-apiserver建立安全连接

- 接下来kubeadm会为Master组件生成Pod配置文件

**此时Kubernetes集群不存在，难道kubeadm会直接执行docker run来启动这些容器吗**

- 不会，Kubernetes中有一种特殊的容器启动方法，叫做“Static Pod”
- 允许你把要部署的Pod的YAML文件放在一个指定的目录中
- 这样当这台机器上kubelet启动时，它会自动检查该目录，加载所有Pod YAML文件并在这台机器上启动它们

Pod的YAML文件，Master组件的YAML文件都在``/etc/kubernetes/manifests``路径里面

- Pod里只定义了一个容器，使用的镜像时：k8s.gcr.io/kube-apiserver-amd64:v1.18.8，这是Kubernetes官方维护的一个组件镜像

- 这个容器的启动命令是kube-apiserver --authorization-mode=Node,....很长的命令，其实，它就是容器里kube-apiserver这个二进制文件再加上指定的配置参数
- 如果要修改一个已有集群的kube-apiserver的配置，需要修改这个YAML文件
- 这些组件的参数也可以在部署时指定

完成以上之后，kubeadm还会生成一个etcd的Pod YAML文件，同样用Static Pod的方式启动etcd

```bash
louis1@louis1:~$ ls /etc/kubernetes/manifests/
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

一旦这些YAML文件出现在被kubelet监视的``/etc/kubernetes/manifests``下，kubelet就会自动创建这些YAML文件中定义的Pod，即Master组件的容器

**Master容器启动后，kubeadm会通过检查localhost:6443/healthz这个Master组件的健康来检查URL**

等Master组件完全运行起来之后，kubeadm会为集群生成一个bootstrap token，之后只要持有这个token，任何安装了kubelet和kubeadm的节点都可以通过**kubeadm join**加入这个集群

当token生成之后，kubeadm会将ca.crt等Master节点的重要信息，**通过ConfigMap的方式保存在etcd当中，供后续部署Node节点使用**

- ConfigMap名字是cluster-info

**kubeadm init的最后一步是安装默认插件**

- 必须安装kube-proxy和DNS这俩插件
- 分别用来提供整个集群的服务发现和DNS功能
- 其实也就是两个容器镜像，用Kubernetes客户端创建两个Pod就可以了

##### 4.1.3 kubeadm join的工作流程

为什么执行kubeadm join需要token？

- 因为任何一台机器想要成为Kubernetes集群中的一个节点，就必须在集群的kube-apiserver上注册

可是想要跟apiserver打交道，这个机器就必须获取相应的证书文件，**为了能够一键安装**，kubeadm至少需要发起一次**非安全模式**的访问到kube-apiserver，从而拿到保存在ConfigMap中的cluster-info（保存了API Server的授权信息）

**在此过程中bootstrap token扮演了安全验证的角色**

##### 4.1.4 配置kubeadm的部署参数

当使用kubeadm init部署Master节点时，推荐使用下面的命令

```bash
kubeadm init --config kubeadm.yaml
```

#### 4.2 从0到1：搭建一个完整的Kubernetes集群

**第一步：安装kubeadm和Docker**

**第二步：部署Kubernetes的Master节点**

如果之前开启了kubeadm，重置一下``sudo kubeadm reset``

执行Kubernetes的部署``kubeadm init --config kubeadm.yaml``

**Kubeadm.yaml**

```bash
apiVersion: kubeadm.k8s.io/v1beta3 #版本信息参考kubeadm config print init-defaults命令结果
kind: ClusterConfiguration
kubernetesVersion: 1.23.1 #根据自己安装的k8s版本来写,版本信息参考kubeadm config print init-defaults>命令结果
imageRepository: registry.aliyuncs.com/google_containers #配置国内镜像

apiServer:
  extraArgs:
    runtime-config: "api/all=true"

#controllerManager:
# extraArgs:
# horizontal-pod-autoscaler-use-rest-clients: "true" #开启kube-controller-manager能够使用自定义资源>（Custom Metrics）进行自动水平扩展,但是高版本不支持该参数需要去掉。
# horizontal-pod-autoscaler-sync-period: "10s"
# node-monitor-grace-period: "10s"

etcd:
  local:
    dataDir: /data/k8s/etcd
```

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

init成功之后生成的token

```bash
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.211.55.8:6443 --token udthxc.9ujgk5o3qmcv8nfg \
        --discovery-token-ca-cert-hash sha256:944335dedd3bca9bddce78a2464436e47096953f92d90dd11be83e7e9a5d232a 
```

第一次使用Kubernetes集群所需要的配置命令

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**需要这些配置的原因是，Kubernetes集群默认需要以加密方式访问**

- 这几条命令是将刚刚部署生成的Kubernetes集群的安全配置文件，保存到当前用户的.kube目录下
- kubectl默认会使用这个目录下的授权信息访问Kubernetes集群

使用kubectl get命令来查看当前唯一节点的状态

```bash
louis1@louis1:~$ kubectl get nodes
NAME     STATUS     ROLES                  AGE     VERSION
louis1   NotReady   control-plane,master   4m55s   v1.23.1
```

在调试Kubernetes集群的时候，最重要的手段就是用kubectl describe来查看该节点的详细信息、状态和事件

```bash
louis1@louis1:~$ kubectl describe node louis1
。。。
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Tue, 11 Jan 2022 14:32:05 +0000   Tue, 11 Jan 2022 14:11:23 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Tue, 11 Jan 2022 14:32:05 +0000   Tue, 11 Jan 2022 14:11:23 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Tue, 11 Jan 2022 14:32:05 +0000   Tue, 11 Jan 2022 14:11:23 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            False   Tue, 11 Jan 2022 14:32:05 +0000   Tue, 11 Jan 2022 14:11:23 +0000   KubeletNotReady              container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
  。。。。
```

**出现NodeNotReady的原因是尚未部署任何网络插件**

可以通过kubectl检查该节点上各个系统Pod的状态，其中kube-system是Kubernetes项目预留的系统Pod的工作空间（Namespace，这个不是Linux Namespace，而是Kubernetes划分不同工作空间的单位）

```bash
louis1@louis1:~$ kubectl get pods -n kube-system
NAME                             READY   STATUS    RESTARTS   AGE
coredns-6d8c4cb4d-sxfh9          0/1     Pending   0          11h
coredns-6d8c4cb4d-x959b          0/1     Pending   0          11h
etcd-louis1                      1/1     Running   0          11h
kube-apiserver-louis1            1/1     Running   0          11h
kube-controller-manager-louis1   1/1     Running   1          11h
kube-proxy-2s9vv                 1/1     Running   0          11h
kube-scheduler-louis1            1/1     Running   1          11h
```

可以看到依赖网络的Pod都处于Pending状态，即调度失败，符合预期，因为这个Master节点的网络尚未部署

**第三步：部署网络插件**

在Kubernetes项目“一切皆容器”设计理念的指导下，部署网络插件非常简单，只需要执行一条kubectl apply指令，以Weave为例

```bash
louis1@louis1:~$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
serviceaccount/weave-net created
clusterrole.rbac.authorization.k8s.io/weave-net created
clusterrolebinding.rbac.authorization.k8s.io/weave-net created
role.rbac.authorization.k8s.io/weave-net created
rolebinding.rbac.authorization.k8s.io/weave-net created
daemonset.apps/weave-net created
```

```bash
louis1@louis1:~$ kubectl get pods -n kube-system
NAME                             READY   STATUS    RESTARTS        AGE
coredns-6d8c4cb4d-sxfh9          1/1     Running   0               12h
coredns-6d8c4cb4d-x959b          1/1     Running   0               12h
etcd-louis1                      1/1     Running   0               12h
kube-apiserver-louis1            1/1     Running   0               12h
kube-controller-manager-louis1   1/1     Running   1               12h
kube-proxy-2s9vv                 1/1     Running   0               12h
kube-scheduler-louis1            1/1     Running   1               12h
weave-net-8r6db                  2/2     Running   1 (2m11s ago)   2m33s
```

所有的系统Pod都成功启动了，刚刚部署的Weave网络插件在kube-system下面新建了一个名叫weave-net-8r6db的Pod。**这些Pod就是容器网络插件在每个节点上的控制组件**

Kubernetes支持容器网络插件，使用的是一个名叫CNI的通用接口，它也是当前容器网络的事实标准，市面上所有的容器网络开源项目都可以通过CNI接入Kubernetes，比如Flannel、Calico、Cannal、Romana等，部署方式都是类似一键部署

**第四步：部署Kubernetes的Worker节点**

Worker节点跟Master节点几乎相同，它们都运行一个kubelet组件

唯一的区别

- 在kubeadm init的过程中，当kubelet启动后，Master节点上还会自动运行kube-apiserver、kube-scheduler、kube-controller-manger这3个系统Pod

部署Worker节点仅需两步

- 在所有Worker节点上执行“安装kubeadm和Docker”的所有步骤

- 需要在指定docker的cgroup驱动为system，跟上面master操作一样，然后再join

- 执行部署Master节点时生成的kubeadm join指令

  - ```bash
    kubeadm join 10.211.55.8:6443 --token udthxc.9ujgk5o3qmcv8nfg \
            --discovery-token-ca-cert-hash sha256:944335dedd3bca9bddce78a2464436e47096953f92d90dd11be83e7e9a5d232a 
    ```

```bash
louis2@louis2:~$ sudo kubeadm join 10.211.55.8:6443 --token udthxc.9ujgk5o3qmcv8nfg         --discovery-token-ca-cert-hash sha256:944335dedd3bca9bddce78a2464436e47096953f92d90dd11be83e7e9a5d232a
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W0111 14:27:27.884473   97762 utils.go:69] The recommended value for "resolvConf" in "KubeletConfiguration" is: /run/systemd/resolve/resolv.conf; the provided value is: /run/systemd/resolve/resolv.conf
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

**第五步：通过Taint/Toleration调整Master执行Pod的策略**

默认情况下Master节点是不允许运行用户Pod的，而Kubernetes做到了这一点，依靠的是它的Taint/Toleration机制

原理很简单

- 一旦某个节点被加上了一个Taint，即“染上污点”，那么所有Pod都不能在该节点上运行

- 除非有个别Pod声明自己能“容忍”这个”污点“，即声明了Toleration，它才可以在该节点上运行

- 加污点的命令``kubectl taint nodes node1 foo=bar:NoSchedule``

  - ```bash
    louis1@louis1:~$ kubectl taint nodes louis2 foo=bar:NoSchedule
    node/louis2 tainted
    ```

  - 此时，node1节点上就会增加一个键值对格式的Taint，即foo=bar:NoSchedule

  - 其中，NoSchedule意味着这个Taint只会在调度新Pod时产生作用，不会影响node1上已经在运行的Pod，哪怕它们没有声明Toleration

Pod声明Toleration，**在Pod的.yaml文件中的spec部分加入tolerations字段**

```bash
spec:
  tolerations:
  - key: "foo"
    operator: "Equal"
    value: "bar"
    effect: "NoSchedule"
```

如果只有键，没有值，则需要用Exists操作符（operator：“Exists”）来说明，该Pod能够容忍所有以foo为键的Taint，才能在该Master节点上运行这个Pod

```bash
spec:
  tolerations:
  - key: "foo"
    operator: "Exists"
    effect: "NoSchedule"
```

**如果只是想要一个单节点的Kubernetes，删除这个Taint才是正确的选择**

foo是key，后面的-表示移除所有以foo为键的Taint

```bash
louis1@louis1:~$ kubectl taint nodes --all foo-
node/louis2 untainted
```

**第六步：部署Dashboard可视化插件**

官方GitHub地址:https://github.com/kubernetes/dashboard

给用户提供一个可视化的Web界面来查看当前集群的各种信息

```bash
louis1@louis1:~$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
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
Warning: spec.template.metadata.annotations[seccomp.security.alpha.kubernetes.io/pod]: deprecated since v1.19, non-functional in v1.25+; use the "seccompProfile" field instead
deployment.apps/dashboard-metrics-scraper created
```

1.7版本之后的Dashboard项目部署后，只能通过Proxy方式在本地访问，如果需要从集群外访问的话，需要用到Ingress

查看Dashboard对应Pod的状态

```bash
louis1@louis1:~$ kubectl get pods -n kubernetes-dashboard
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-577dc49767-l5spt   1/1     Running   0          5m2s
kubernetes-dashboard-78f9d9744f-2bg45        1/1     Running   0          5m2s
```

**创建管理员用户**

``vi admin.yaml``

```bash
# 添加以下内容 
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

然后执行``kubectlapply-fadmin.yaml-nkube-system``

```bash
louis1@louis1:~$ kubectl apply -f admin.yaml -n kube-system
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
```

配置dashboard的访问端口，将type: ClusterIP改为type: NodePort

```bash
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
```

找到dashboard使用的端口

```bash
louis1@louis1:~$ kubectl get svc -A | grep kubernetes-dashboard
kubernetes-dashboard   dashboard-metrics-scraper   ClusterIP   10.108.254.120   <none>        8000/TCP                 17m
kubernetes-dashboard   kubernetes-dashboard        NodePort    10.96.164.216    <none>        443:31268/TCP            17m
```

生成登录使用的token

```bash
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')

...
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6Ijc4Z1dSeWQ5aEIxSXdyR1h2LXBIU0RocHNXQmp6MVprOEFHdXYzQm93QVkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC10b2tlbi1xNm56MiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjY5OTZjY2JiLWNjODEtNDFjMS1iNDVkLTNiMDZhZDY3NTMzMiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDprdWJlcm5ldGVzLWRhc2hib2FyZCJ9.5uRJbVYC2pcJJuVWNLLzCzGnzXvrjoUdFUFnqtNwUVl5204xp8f7BgbN7GJ0dAxBqYks7v1S08FZCQ6UHe1iyFw9T-HcQHuh3OZohM7TsgO7rPFEa04nkWQfn39QejTDEGYcDAMxc1V3WSH2mfeGdxu_ZhsbVKgoCK9ndXurQts32XN-IOqzq7Lp9OmvuMqZm5Y5XjB-jUgJ4Obm7yILx_yOj6SrIkginewXPPvNMfy4KYW8-JZAi1UgrcCuTWlMNYeKRZEZrnrrbrVOxaMvhVTFsTxWK5PitRPaiZ-938Ga0lJDOp10jnyuYaa0-dXWk90O61p-xuacpfAqlPNspA
```





**换一种方式部署，使用阿里云镜像（部署失败）**

https://cr.console.aliyun.com/cn-hangzhou/instances/images

先创建yaml文件，之后用create创建``vim kubernetes-dashboard.yaml``

yaml文件内容

```bash
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
      - name: kubernetes-dashboard
        # 更改为国内的镜像        
        image: registry.cn-hangzhou.aliyuncs.com/lynchj/kubernetes-dashboard-amd64:v1.10.0
        ports:
        - containerPort: 8443
          protocol: TCP
```

然后再加个配置，还是那个yaml文件

```bash
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
```

**报错：The connection to the server raw.githubusercontent.com was refused - did you specify the right host or port?**  

原因：外网不可访问

```bash
# 在https://www.ipaddress.com/查询raw.githubusercontent.com的真实IP。
sudo vim /etc/hosts
185.199.108.133 raw.githubusercontent.com
```

开启IPVS，修改ConfigMap中的kube-system/kube-proxy中的模式为ipvs

```bash
kubectl edit cm kube-proxy -n kube-system  # mode: "ipvs"
```

重启kube-proxy

```bash
kubectl get pod -n kube-system | grep kube-proxy | awk '{system("kubectl delete pod "$1" -n kube-system")}'
```

**第七步：部署容器存储插件（部署失败）**

因为容器是**无状态**的，所以容器的持久化存储，就是保存容器存储状态的重要手段

存储插件会在容器里挂载一个基于网络或者其他机制的远程Volume，这使得在容器里创建的文件，实际上保存在远处存储服务器上，或者以分布式的方式保存在多个节点上，**与当前宿主机没有任何绑定关系**

此次选择部署一个很重要的Kubernetes存储插件项目：Rook

Rook项目是一个基于Ceph的Kubernetes存储插件（后期增加了对更多存储实现的支持），不同于对Ceph的简单封装，Rook在实现中加入了水平扩展、迁移、灾难备份、监控等大量的企业级功能，使得该项目变成了一个完整的、生产级别可用的容器存储插件

三条命令，Rook即可完成复杂的Ceph存储后端部署（没外网，搞不来）

```bash
kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.1/cluster/examples/kubernetes/ceph/common.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.1/cluster/examples/kubernetes/ceph/operator.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.1/cluster/examples/kubernetes/ceph/cluster.yaml
```

在部署后，可以看到Rook项目会将自己的Pod放置在由它自己管理的Namespace中

```bash
kubectl get pods -n rook-ceph
```

#### 4.3 第一个Kubernetes应用

一个YAML文件，对应到Kubernetes中就是一个对象

用指令运行起来，``kubectl create -f _我的配置文件_``

yaml示例

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        #image: nginx:1.7.9 如果不知道，就直接拉最新的
        image: nginx:latest
        ports:
        - containerPort: 80
```

kind字段指定了这个API对象的类型

Deployment，是一个定义多副本应用（多个副本Pod）的对象

- Deployment还负责在Pod定义发生变化时，对每个副本进行滚动更新
- spec.replicas是Pod副本个数

**Pod就是Kubernetes世界里面的“应用运行单元”，而一个应用运行单元可以由多个容器组成**

使用一种API对象（Deployment）管理另一种API对象（Pod）的方法，在Kubernetes中叫做“控制器模式”（controller pattern）

每个API对象都有一个叫作Metadata的字段，这个字段就是API对象的“标识”，即元数据，它也是我们从Kubernetes里找到这个对象的主要依据。其中最主要的字段是Labels

Labels是一组键值对格式的标签，Deployment这样的控制器对象，可以通过这个Labels字段从Kubernetes中过滤出它所关心的被控制对象

- 上面的yaml，Deployment会把所有正在运行的、携带app：nginx标签的Pod识别为被管理的对象，并确保这些Pod的总数严格等于2
- 过滤规则的定义，在Deployment的spec.selector.matchLabels字段，一般称为Label Selector

在Metadata中，还有一个与Labels格式、层级完全相同的字段，叫做Annotations，**专门用来携带键值对格式的内部信息**

- 内部信息，指的是对这些信息感兴趣的是Kubernetes组件，而不是用户
- 大多数Annotations是在Kubernetes运行过程中被自动加在这个API对象上

一个Kubernetes的API对象的定义，大多可以分成Metadata和Spec两个部分

- 前者存放这个对象的元数据，对所有API对象来说，这部分的字段和格式基本相同
- 后者存放的是属于这个对象独有的定义，用来描述它所要表达的功能

将这个yaml文件运行起来

```bash
louis1@louis1:~$ kubectl create -f nginx-deployment.yaml 
deployment.apps/nginx-deployment create
```

如果运行坏了，使用``kubectl delete po NAME``来删除Pod

```bash
louis1@louis1:~$ kubectl get pods
NAME                               READY   STATUS             RESTARTS      AGE
nginx-deployment-f7ccf9478-hf5wk   0/1     CrashLoopBackOff   23 (9h ago)   12h
nginx-deployment-f7ccf9478-rh8gp   0/1     CrashLoopBackOff   27 (9h ago)   12h
```

查看所有pod，``kubectl get pod --all-namespaces ``，然后使用命令``kubectl delete -f nginx-deployment.yaml  ``删除status不是running的pod，并不会删除原yaml文件

查看一个API对象的细节

```bash
louis1@louis1:~$ kubectl describe pod nginx-deployment-8d545c96d-74qz9
Name:         nginx-deployment-8d545c96d-74qz9
Namespace:    default
Priority:     0
Node:         louis3/10.211.55.9
Start Time:   Thu, 13 Jan 2022 02:15:45 +0000
Labels:       app=nginx
              pod-template-hash=8d545c96d
Annotations:  <none>
Status:       Running
IP:           10.36.0.2
IPs:
  IP:           10.36.0.2
Controlled By:  ReplicaSet/nginx-deployment-8d545c96d
Containers:
  nginx:
    Container ID:   docker://01e41d740143987a88aea6728fc3b69b518489e4df9bcefb37fbfd3e37ef1162
    Image:          nginx:latest
    Image ID:       docker-pullable://nginx@sha256:0d17b565c37bcbd895e9d92315a05c1c3c9a29f762b011a10c54a66cd53c9b31
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thu, 13 Jan 2022 02:15:58 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-hqltp (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-hqltp:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  31m   default-scheduler  Successfully assigned default/nginx-deployment-8d545c96d-74qz9 to louis3
  Normal  Pulling    31m   kubelet            Pulling image "nginx:latest"
  Normal  Pulled     31m   kubelet            Successfully pulled image "nginx:latest" in 11.908289403s
  Normal  Created    31m   kubelet            Created container nginx
  Normal  Started    31m   kubelet            Started container nginx
```

其中**Events**值得特别关注，对API对象的所有重要操作都会被记录在这个对象的Evnets里，并且显示在kubectl describe指令返回的结果中

如果后续要升级Nginx服务，把镜像从1.7.9升到1.8，只需要修改yaml里面的值，但是修改之后只会发生在本地，如果想要在Kubernetes里面也生效，需要使用kubectl replace

- ``kubectl replace -f nginx-deployment.yaml``

**推荐使用``kubectl apply``命令来统一进行Kubernetes对象的创建和更新操作**

- 不论是创建还是更新，都用这个，这种面向终态的分布式系统设计原则，**被称为是“声明式API”**

- ```bash
  
  $ kubectl apply -f nginx-deployment.yaml
  
  # 修改nginx-deployment.yaml的内容
  
  $ kubectl apply -f nginx-deployment.yaml
  ```

如果开发人员修改了应用，发布了新的发布内容，那么这个YAML文件也需要修改，并且成为这次变更的一部分，接下来，**运维人员可以使用git diff命令查看这个yaml文件本身的变化，然后使用kubectl apply命令更新这个应用**

在Deployment中尝试声明一个Volume

- 在Kubernetes中，Volume属于Pod对象的一部分，需要修改这个YAML文件里template.spec字段

```bash

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: nginx-vol
      volumes:
      - name: nginx-vol
        emptyDir: {}
```

在Deployment的Pod模版部分添加了一个volumes字段，定义了这个Pod声明的所有Volume

- 名字叫作nginx-vol
- 类型是emptyDir
  - emptyDir等同于Docker的隐式Volume参数，即不显示声明宿主机目录的Volume
  - 所以Kubernetes也会在宿主机上创建一个临时目录，这个目录将来会被绑定挂载到容器所声明的Volume目录上

**Kubernetes的emptyDIr类型只是把Kubernetes创建的临时目录作为Volume的宿主机目录，交给了Docker**

- 这么做的原因，Kubernetes不想依赖Docker自己创建的那个_data目录

Pod中的容器使用volumeMounts字段来声明自己要挂载到哪个Volume，并通过mountPath字段来定义容器内的Volume目录，比如``/usr/share/nginx/html``

Kubernetes也提供了显示的Volume定义，叫作hostPath，这样，容器Volume挂载的宿主机目录就变成了/var/data

```bash
 ...   
    volumes:
      - name: nginx-vol
        hostPath: 
          path:  " /var/data"
```

**完成修改之后，使用kubectl apply指令来更新这个Deployment**

可以使用kubectl exec指令进入这个Pod当中（容器的Namespace）中，查看这个Volume目录

```bash
louis1@louis1:~$ kubectl exec -it nginx-deployment-57b48455b4-b76c4 -- /bin/bash
root@nginx-deployment-57b48455b4-b76c4:/# ls /usr/share/nginx/html/
```

## 第二部分 Kubernetes核心原理

### 第五章 Kubernetes编排原理

#### 5.1 为什么我们需要Pod

**Pod是Kubernetes项目的原子调度单位**

Namespace做隔离，Cgroups做限制，rootfs做文件系统

容器的本质是什么？

- 容器的本质是进程，容器就是这个系统里的.exe安装包

- 则Kubernetes就是操作系统

```bash
louis1@louis1:~$ pstree -g
systemd(1)─┬─accounts-daemon(873)─┬─{accounts-daemon}(873)
           │                      └─{accounts-daemon}(873)
           ├─atd(900)
           ├─containerd(25994)─┬─{containerd}(25994)
           │                   ├─{containerd}(25994)
           │                   ├─{containerd}(25994)
           │                   ├─{containerd}(25994)
 ...
           ├─networkd-dispat(886)
           ├─polkitd(961)─┬─{polkitd}(961)
           │              └─{polkitd}(961)
           ├─rsyslogd(889)─┬─{rsyslogd}(889)
           │               ├─{rsyslogd}(889)
           │               └─{rsyslogd}(889)
           ├─snapd(891)─┬─{snapd}(891)
           │            ├─{snapd}(891)
           │            ├─{snapd}(891)
           │            ├─{snapd}(891)
           │            ├─{snapd}(891)
           │            ├─{snapd}(891)
           │            ├─{snapd}(891)
           │            ├─{snapd}(891)
           │            ├─{snapd}(891)
           │            ├─{snapd}(891)
           │            ├─{snapd}(891)
           │            ├─{snapd}(891)
           │            └─{snapd}(891)   				├─sshd(2474)───sshd(578341)───sshd(578341)───bash(578497)───pstree(683838)
           ├─systemd(1585)─┬─(sd-pam)(1585)
           │               ├─dirmngr(18102)
           │               └─gpg-agent(18106)
           ├─systemd-journal(413)
           ├─systemd-logind(893)
           ├─systemd-network(835)
           ├─systemd-resolve(837)
           ├─systemd-timesyn(632)───{systemd-timesyn}(632)
           ├─systemd-udevd(440)
           ├─udisksd(895)─┬─{udisksd}(895)
           │              ├─{udisksd}(895)
           │              ├─{udisksd}(895)
           │              └─{udisksd}(895)
           └─unattended-upgr(971)───{unattended-upgr}(971)
```

在一个真正的操作系统中，进程是以进程组的方式组织在一起

例如rsyslogd这个程序，它负责的是Linux操作系统中的日志处理，它是由imklog、imuxsock和main这几个线程组成的一个进程，这些线程或者说轻量级进程之间是可以共享文件、信号、数据内存甚至部分代码，从而紧密协作共同履行一个程序的职责

现在假如要把这个reyslogd容器化，**由于受限于“单进程模型”**，这3个模块必须分别做成3个容器

- 单进程模型并不是指容器里只能运行一个进程，而是指容器无法管理多个进程
- 因为容器里PID=1的进程就是应用本身，它无法对里面的其他应用进行管理，假如其他进程退出之后，那么垃圾收集工作也没人做

**Pod最重要的事实**

- 它只是一个逻辑概念
- Kubernetes真正处理的，还是宿主机操作系统上Linux容器的Namespace和Cgroups
- 并不存在所谓的Pod的边界或者隔离环境

**Pod其实是一组共享了某些资源的容器**

- Pod里的所有容器都共享了一个Network Namespace，并且可以声明共享同一个Volume

在Kubernetes项目里，Pod的实现需要使用一个中间容器，这个容器叫作Infra容器

- 在这个Pod中，Infra容器永远是第一个被创建的容器
- Pod的生命周期只跟Infra容器一致，与其他容器无关
- 用户定义的其他容器则通过Join Network Namespace的方式与Infra容器关联在一起

Infra容器占用极少的资源，使用的是特殊的镜像，叫作k8s.gcr.io/pause，这个镜像是用汇编写的、永远处于暂停状态的容器，解压后的大小只有100～200KB

同一个Pod里的所有用户容器，它们的进出流量可以认为都是通过Infra容器完成的

- 意味着，将来为Kubernetes开发一个网络插件，**应该重点考虑如何配置这个Pod的Network Namespace**，而不是每一个用户容器如何使用你的网络配置，这是没有意义的
- 如果网络插件需要在容器里安装某些包或者配置才能完成，是不可取的，因为Infra容器镜像里的rootfs里几乎什么都没有
- 也意味着网络插件完全不必关心用户容器的启动与否，只需要关注如何配置Pod，也就是Infra容器的Network Namespace即可

```bash

apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
    hostPath:      
      path: /data
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```

Debian-container和nginx-container都声明挂载了shared-data这个Volume

- 而shared-data是hostPath类型，所以宿主机对应的目录就是/data
- 实际上被绑到两个容器中

Pod这种“超亲密关系”容器的设计思想，实际上是希望当用户想在一个容器里运行多个功能无关的应用时，应该优先考虑它们是否更应该被描述成一个Pod里的多个容器

**1.WAR包与Web服务器**

```bash

apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: geektime/sample:v2
    name: war
    command: ["cp", "/sample.war", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: geektime/tomcat:7.0
    name: tomcat
    command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
    volumeMounts:
    - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001 
  volumes:
  - name: app-volume
    emptyDir: {}
```

在Pod中，所有Init Container定义的容器都会比spec.containers定义的容器先启动，**Init Container容器会按顺序逐一启动，直到它们都启动并且退出了，用户容器才会启动**

**这是容器设计模式中最常用的一种模式，称为sidecar**

- sidecar指的是我们可以在一个Pod中启动一个辅助容器，来完成一些独立于主进程（主容器）的工作

**2.容器的日志收集**

现在有一个应用，需要不断地把日志文件输出到容器的/var/log目录中，这时就可以把一个Pod里的Volume挂载到应用容器的/var/log目录上。

接下来sidecar容器就需要不断地从自己的/var/log的目录里读取日志文件，转发到MongoDB或者ElasticSearch中存储起来

Pod的另一个重要特性是，它的所有容器都共享同一个Network Namespace

- 很多与Pod网络相关的配置和管理都可以交给sidecar完成，完全无需干涉用户容器

**可以把整台虚拟机想象成一个Pod，把这些进程分别做成容器镜像，把有顺序关系的容器定义为Init Container。这才是更加合理的、松耦合的容器编排诀窍，也是从传统应用架构到微服务架构最自然的过渡方式**

#### 5.2 深入解析Pod对象

Pod中几个重要字段的含义和用法

NodeSelector。一个供用户将Pod与Node进行绑定的字段

```bash
apiVersion: v1
kind: Pod
...
spec:
 nodeSelector:
   disktype: ssd
```

这样的配置意味着这个Pod永远只能在携带了disktype:ssd标签的节点上运行，否则将调度失败

NodeName。一旦Pod的这个字段被赋值，Kubernetes项目就会认为这个Pod已调度，调度的结果就是赋值的节点名称，**这个字段一般由调度器负责设置，但用户也可以设置它来“骗过”调度器**，一般测试或者调试的时候才会用到

HostAliases。定义了Pod的hosts文件（比如/etc/hosts）里的内容

```bash

apiVersion: v1
kind: Pod
...
spec:
  hostAliases:
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
...
```

如果要设置hosts文件里的内容，一定要通过这种方法，如果直接修改了hosts文件在Pod被删除重建之后，kubelet会自动覆盖被修改的内容

Pod的设计就是要让其中的容器尽可能多地共享Linux Namespace，仅保留必要的隔离和限制能力

例如，在这个yaml中，定义了shareProcessNamespace=true，意味着这个Pod的容器要共享PID Namespace

```bash
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```

在这个yaml文件里，定义了两个容器，nginx容器和开启了tty和stdin的shell容器

- 开启等同于设置docker run -it（-i即stdin，-t即tty）参数
- tty的作用是接受用户的标准输入，返回操作系统的标准输出
- 为了能够在tty中输入信息，需要开启stdin（标准输入流）

```bash
louis1@louis1:~$ kubectl create -f nginx.yaml 
pod/nginx created
louis1@louis1:~$ kubectl attach -it nginx -c shell
If you don't see a command prompt, try pressing enter.
/ # ps ax
PID   USER     TIME  COMMAND
    1 65535     0:00 /pause
    6 root      0:00 nginx: master process nginx -g daemon off;
   36 101       0:00 nginx: worker process
   37 101       0:00 nginx: worker process
   38 root      0:00 sh
   43 root      0:00 ps ax
```

凡是Pod中的容器要共享宿主机的Namespace，也一定是Pod级别的定义

```bash
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  hostNetwork: true
  hostIPC: true
  hostPID: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```

在这个Pod中，定义了共享宿主机的Network、IPC和PID Namespace，意味着这个Pod里的所有容器会直接使用宿主机的网络，直接与宿主机进行IPC通信，“看到”宿主机里正在运行的所有进程

**Containers的重要属性**

ImagePullPolicy字段。定义了镜像拉取的策略

- 默认值是Always，即每次创建Pod都重新拉取一次镜像
- 当容器的镜像是类似于nginx或者nginx：lastest的时候，也会认为是always
- 当它的值被定义为Never字段或者IfNotPresent，则意味着Pod永远不会主动拉取这个镜像，或者只在宿主机上不存在这个镜像时才拉取

 Lifecycle字段。定义的是Container Lifecycle Hooks，作用是在容器状态发生变化的时候触发一系列“钩子”

```bash

apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
```

- postStart，指的是在容器启动后立刻执行一个指定操作
  - postStart定义的操作虽然是在Docker容器ENTERYPOINT执行之后，但是不保证顺序，在postStart启动时，ENTERYPOINT有可能尚未结束
  - 如果postStart执行超时或者出错，Kubernetes会在该Pod的Events中报出该容器启动失败的错误信息，导致Pod也处于失败状态
- preStop发生的时机则是容器被结束之前（比如收到了SIGKILL信号）
- preStop操作的执行时同步的，**所以它会阻塞当前的容器结束流程，直到这个Hook定义操作完成之后，才允许容器被结束**，实现优雅退出

**Pod生命周期的变化主要体现在Pod API对象的Status部分**，这时它除Metadata和Spec外的第三个重要字段

- pod.status.phase就是Pod当前状态
  - Pending。Pod的YAML文件已经提交给了Kubernetes，API对象已经被创建并保存到etcd当中，但是这个Pod里面有些容器不能被顺序创建，例如调度不成功
  - Running。Pod调度成功，跟一个具体的节点绑定。包含的容器都创建成功，至少有一个正在运行
  - Succeeded。意味着Pod里所有容器都正常运行完毕，并且成功退出了，这种情况在运行与创新任务时最为常见
  - Failed。Pod里至少有一个容器以不正常的状态（非0的返回码）退出，出现这个状况要想办法调试这个容器的应用，比如查看Pod的Events和日志
  - Unknown。异常状态，意味着Pod的状态不能持续地被kubelet汇报给kube-apiserver，很可能是主从节点（Master和kubelet）间的通信出现了问题

更进一步，Pod对象的Status字段还可以细分一组Conditions，主要用于描述造成当前Status的具体原因是什么

- PodScheduled
- Ready
- Initialized
- Unshcedulable

#### 5.3 Pod对象使用进阶

在Kubernetes中有几种特殊的Volume，它们存在的意义不是为了存放容器里的数据，也不是用于容器和宿主机之间的数据交换，而是为容器提供预先定义好的数据。

**从容器来看，这些Volume里的信息就仿佛是被Kubernetes“投射”进入容器中的，这正是Projected Volume的含义**

**1.Secret**

- 作用是把Pod想要访问的加密数据存放到etcd中
- 后续就可以通过在Pod的容器里挂载Volume的方式访问这些Secret里保存的信息了

Secret最典型的场景，存放数据库的Credential信息

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-projected-volume
spec:
  containers:
  - name: test-secret-volume
    image: busybox
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: mysql-cred
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: mysql-cred
    projected:
      sources:
      - secret:
          name: mysecret
```

这个Volume的数据来源。则是名为user和pass的Secret对象，分别对应数据库的用户名和密码

```bash
$ cat ./username.txt
admin
$ cat ./password.txt
c1oudc0w!

$ kubectl create secret generic user --from-file=./username.txt
$ kubectl create secret generic pass --from-file=./password.txt
```

如果想查看Secret对象，只需要执行``kubectl get secrets``

除了使用kubectl create secret指令，也可以直接编写yaml文件来创建这个secret对象

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  user: YWRtaW4=
  pass: MWYyZDFlMmU2N2Rm
```

里面的数据必须是base64转码后的，避免明文

```bash
$ echo -n 'admin' | base64
YWRtaW4=
$ echo -n '1f2d1e2e67df' | base64
MWYyZDFlMmU2N2Rm
```

当Pod变成Running之后，验证这些Secret对象是否已经在容器里

```bash
louis1@louis1:~$ kubectl exec -it test-projected-volume -- /bin/sh
/ # ls /projected-volume/
pass  user
/ # cat  /projected-volume/user
/ # cat  /projected-volume/pass
1f2d1e2e67df/
```

保存在etcd里的用户名和密码信息已经以文件的形式出现在容器的Volume目录里了。这个文件名称就是kubectl create secret指定的key，**或者说是Secret对象的data字段指定的key**

- 像这样通过挂载方式进入容器的Secret，一旦其对应的etcd数据更新，这些Volume里的文件内容也会更新，**kubelet组件在定时维护这些Volume**
- 这个更新可能会有一定的延时，**所以在编写应用程序时，在发起数据库连接的代码处写好重试和超时的逻辑，绝对是好习惯！**

**2.ConfigMap**

与Secret类似，区别在于ConfigMap保存的是无须加密的、应用所需的配置信息，用法与Secret完全相同

- 可以使用kubectl create configmap从文件或者目录创建ConfigMap
- 也可以直接编写ConfigMap对象的YAML文件

例如Java应用所需的配置文件，就可以保存在ConfigMap中

```bash

# .properties文件的内容
$ cat example/ui.properties
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice

# 从.properties文件创建ConfigMap
$ kubectl create configmap ui-config --from-file=example/ui.properties

# 查看这个ConfigMap里保存的信息(data)
$ kubectl get configmaps ui-config -o yaml
apiVersion: v1
data:
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
kind: ConfigMap
metadata:
  name: ui-config
  ...
```

注意，kubectl get -o yaml这样的参数会将指定的Pod API对象以YAML的方式展示出来

**3.Downward API**

作用是让Pod里的容器能够直接获取这个Pod API对象本身的信息

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-downwardapi-volume
  labels:
    zone: us-est-coast
    cluster: test-cluster1
    rack: rack-22
spec:
  containers:
    - name: client-container
      image: k8s.gcr.io/busybox
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
          readOnly: false
  volumes:
    - name: podinfo
      projected:
        sources:
        - downwardAPI:
            items:
              - path: "labels"
                fieldRef:
                  fieldPath: metadata.labels
```

Volume的数据来源变成了Downward API，这个Downward API Volume声明了要暴露Pod的metadata.labels信息给容器

当前Pod的Labels字段的值被Kubernetes自动挂载成容器里的/etc/podinfo/labels文件

```bash
$ kubectl create -f dapi-volume.yaml
$ kubectl logs test-downwardapi-volume
cluster="test-cluster1"
rack="rack-22"
zone="us-est-coast"
```

**在使用Downward API时，要查阅官方文档**

Downward API能够获取的信息一定是Pod里的容器进程启动之前就能确定下来的信息，如果想获取Pod容器运行后的信息，比如容器进程的PID，**那么就应该考虑在Pod里定义一个sidecar**

**4.ServiceAccountToken**

Service account：从Pod里面调用k8s API来控制集群

Service Account的授权信息和文件，实际上保存在它所绑定的一个特殊的Secret对象里。这个特殊的对象叫作ServiceAccountToken

**任何运行在 Kubernetes 集群上的应用，都必须使用这个 ServiceAccountToken 里保存的授权信息，也就是 Token，才可以合法地访问 API Server。**

Kubernetes 已经提供了一个默认“服务账户”（default Service Account）

- 任何一个运行在 Kubernetes 里的 Pod，都可以直接使用这个默认的 Service Account，而无需显示地声明挂载它

Kubernetes 其实在每个 Pod 创建的时候，自动在它的 spec.volumes 部分添加上了默认 ServiceAccountToken 的定义，然后自动给每个容器加上了对应的 volumeMounts 字段。**这个过程对于用户来说是完全透明的**

**这种把 Kubernetes 客户端以容器的方式运行在集群里，然后使用 default Service Account 自动授权的方式，被称作“InClusterConfig”，也是最推荐的进行 Kubernetes API 编程的授权方式。**

 Pod 的另一个重要的配置：容器健康检查和恢复机制

- Pod 里的容器定义一个健康检查“探针”（Probe）
- kubelet 就会根据这个 Probe 的返回值决定这个容器的状态，而不是直接以容器镜像是否运行（来自 Docker 返回的信息）作为依据

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: test-liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

```bash
louis1@louis1:~$ kubectl create -f test-liveness-exec.yaml 
pod/test-liveness-exec created

louis1@louis1:~$ kubectl get pod
NAME                                READY   STATUS        RESTARTS      AGE
nginx                               2/2     Terminating   1 (23h ago)   26h
nginx-deployment-57b48455b4-2xgtr   0/1     Pending       0             4m26s
nginx-deployment-57b48455b4-86xq7   0/1     Pending       0             4m16s
nginx-deployment-57b48455b4-b76c4   1/1     Terminating   0             31h
nginx-deployment-57b48455b4-s4vkc   1/1     Terminating   0             31h
test-liveness-exec                  0/1     Pending       0             5s
test-projected-volume               1/1     Terminating   0             22h

louis1@louis1:~$ kubectl describe pod test-liveness-exec
...
Events:
  Type     Reason            Age                    From               Message
  ----     ------            ----                   ----               -------
  ...
  Warning  Unhealthy         20h (x6 over 20h)      kubelet            Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
  Normal   Killing           20h (x2 over 20h)      kubelet            Container liveness failed liveness probe, will be restarted
```

Pod 并没有进入 Failed 状态，而是保持了 Running 状态

RESTARTS字段从0变成了1，这个异常的容器已经被 Kubernetes 重启了。在这个过程中，Pod 保持 Running 状态不变。

这个功能就是 Kubernetes 里的 Pod 恢复机制，也叫 restartPolicy。

- 它是 Pod 的 Spec 部分的一个标准字段（pod.spec.restartPolicy），默认值是 Always，
- 即：任何时候这个容器发生了异常，它一定会被重新创建。

**一旦一个 Pod 与一个节点（Node）绑定，除非这个绑定发生了变化（pod.spec.node 字段被修改），否则它永远都不会离开这个节点**

- 这也就意味着，如果这个宿主机宕机了，这个 Pod 也不会主动迁移到其他节点上去。
- 而如果想让 Pod 出现在其他的可用节点上，就必须使用 Deployment 这样的“控制器”来管理 Pod，哪怕只需要一个 Pod 副本

作为用户，还可以通过设置restartPolicy，改变Pod的恢复策略

- Always：在任何情况下，只要容器不在运行状态，就自动重启容器
- OnFailure: 只在容器 异常时才自动重启容器
- Never: 从来不重启容器

restartPolicy 和 Pod 里容器的状态，以及 Pod 状态的对应关系，**只需要记住两个基本的设计原理**

- **只要 Pod 的 restartPolicy 指定的策略允许重启异常的容器（比如：Always），那么这个 Pod 就会保持 Running 状态，并进行容器重启。**否则，Pod 就会进入 Failed 状态
- **对于包含多个容器的 Pod，只有它里面所有的容器都进入异常状态后，Pod 才会进入 Failed 状态。**在此之前，Pod都是Running 状态。

所以，假如一个 Pod 里只有一个容器，然后这个容器异常退出了。那么，只有当 restartPolicy=Never 时，这个 Pod 才会进入 Failed 状态。而其他情况下，由于 Kubernetes 都可以重启这个容器，所以 Pod 的状态保持 Running 不变。

**除了在容器中执行命令外，livenessProbe 也可以定义为发起 HTTP 或者 TCP 请求的方式**

```yaml
...
livenessProbe:
     httpGet:
       path: /healthz
       port: 8080
       httpHeaders:
       - name: X-Custom-Header
         value: Awesome
       initialDelaySeconds: 3
       periodSeconds: 3
```

```yaml
    ...
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

Pod 其实可以暴露一个健康检查 URL（比如 /healthz），或者直接让健康检查去检测应用的监听端口。这两种配置方法，在 Web 服务类的应用中非常常用。

**Kubernetes 能不能自动给 Pod 填充某些字段呢？**

- 可以

运维人员可以定义一个 PodPreset 对象。在这个对象中，凡是他想在开发人员编写的 Pod 里追加的字段，都可以预先定义好

```yaml
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: allow-database
spec:
  selector:
    matchLabels:
      role: frontend
  env:
    - name: DB_PORT
      value: "6379"
  volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir: {}
```

在这个 PodPreset 的定义中，首先是一个 selector。这就意味着后面这些追加的定义，只会作用于 selector 所定义的、带有“role: frontend”标签的 Pod 对象，这就可以防止“误伤”。

开发人员的yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: website
  labels:
    app: website
    role: frontend
spec:
  containers:
    - name: website
      image: nginx
      ports:
        - containerPort: 80
```

运维人员先创建这个PodPreset，然后开发人员才创建Pod

```bash
kubectl create -f preset.yaml
kubectl create -f pod.yaml

然后查看Pod的API对象
kubectl get pod website -o yaml
```

如果create preset.yaml出错，尝试下面这个，然后重启systemctl restart kubelet（试过还是无效，以后再来弄）

**破案了**

**PodPreset API 从k8s v1.20后被remove了： The v1alpha1 PodPreset API and admission plugin has been removed with no built-in replacement. Admission webhooks can be used to modify pods on creation.**

```bash
# 别改，改了之后kubectl直接G了
vi /etc/kubernetes/manifests/kube-apiserver.yaml
#增加这一行
- --runtime-config=settings.k8s.io/v1alpha1=true
#在下一行后面增加",PodPreset"
- --enable-admission-plugins=NodeRestrictio
```

**PodPreset 里定义的内容，只会在 Pod API 对象被创建之前追加在这个对象本身上，而不会影响任何 Pod 的控制器的定义。**

如果定义了同时作用于一个 Pod 对象的多个 PodPreset，会发生什么呢？

- Kubernetes 项目会帮你合并（Merge）这两个 PodPreset 要做的修改。而如果它们要做的修改有冲突的话，这些冲突字段就不会被修改。

#### 5.4 编排确实很简单：谈谈“控制器”思想

Pod 这个看似复杂的 API 对象，实际上就是对容器的进一步抽象和封装而已。

controller-manager 组件负责管理 Pod 的调度和控制

```bash
$ cd kubernetes/pkg/controller/
$ ls -d */              
deployment/             job/                    podautoscaler/          
cloud/                  disruption/             namespace/              
replicaset/             serviceaccount/         volume/
cronjob/                garbagecollector/       nodelifecycle/          replication/            statefulset/            daemon/
...
```

这个目录下面的每一个控制器，都以独有的方式负责某种编排功能。而我们的 Deployment，正是这些控制器中的一种。

**它们都遵循 Kubernetes 项目中的一个通用编排模式，即：控制循环（control loop）**

```bash
for {
  实际状态 := 获取集群中对象X的实际状态（Actual State）
  期望状态 := 获取集群中对象X的期望状态（Desired State）
  if 实际状态 == 期望状态{
    什么都不做
  } else {
    执行编排动作，将实际状态调整为期望状态
  }
}
```

实际状态数据来源： 

- kubelet通过心跳汇报 
- 监控系统中保存的应用监控数据 
- 控制器自己收集

期望状态，一般来自于用户提交的 YAML 文件。

以 Deployment 为例，描述它对控制器模型的实现：

- Deployment 控制器从 Etcd 中获取到所有携带了“app: nginx”标签的 Pod，然后统计它们的数量，这就是实际状态；
- Deployment 对象的 Replicas 字段的值就是期望状态；
- Deployment 控制器将两个状态做比较，然后根据比较结果，确定是创建 Pod，还是删除已有的 Pod

**这个操作，通常被叫作调谐（Reconcile）。这个调谐的过程，则被称作“Reconcile Loop”（调谐循环）或者“Sync Loop”（同步循环）。**

Deployment 定义的 template 字段，在 Kubernetes 项目中有一个专有的名字，**叫作 PodTemplate（Pod 模板）**

类似 Deployment 这样的一个控制器，实际上都是由上半部分的控制器定义（包括期望状态），加上下半部分的被控制对象的模板组成的。

#### 5.5 经典PaaS的记忆：作业副本与水平扩展

Deployment实现了Kubernetes项目中Pod的“水平扩展 / 收缩”（horizontal scaling out/in），“滚动更新”（rolling update）的方式，来升级现有的容器。

这个能力的实现，依赖的是 Kubernetes 项目中的一个非常重要的概念（API 对象）：ReplicaSet。

————————————————————————

休息一段时间，暂时不看这本书了
