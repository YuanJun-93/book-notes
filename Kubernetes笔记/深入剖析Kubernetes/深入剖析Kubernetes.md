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

#### 第四章 Kubernetes集群搭建与配置

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

