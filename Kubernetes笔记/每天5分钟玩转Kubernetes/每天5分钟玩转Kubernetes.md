### 第一章：先把Kubernetes跑起来

https://kubernetes.io/docs/tutorials/kubernetes-basics/

**创建Kubernetes集群**

- ``minikube start``创建单节点的Kubernetes集群
- ``kubectl get nodes``查看当前节点数量
- ``kubectl cluster-info``查看集群信息

**部署应用**

- ``kubectl create deployment Kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1``创建应用
- ``kubectl get deployment``查看有哪些应用

使用kubectl run部署应用

- ``kubectl run Kubernetes-bootcamp --image=docker.io/jocatalin/kubernetes-bootcamp:v1 --port=8080``
- Deployment理解为应用

**Pod**

- Pod是容器的集合，通常会将紧密相关的一组容器放到一个Pod中，同一个Pod中的所有容器共享IP地址和Port空间，也就是说它们在一个network namespace中
- Pod是Kubernetes调度的最小单位，同一Pod中的容器始终被一起调度
- ``kubectl get pods``查看当前的Pod

**访问应用**

- 默认情况下，所有Pod只能在集群内部访问，对于刚刚部署的8080端口，为了能让外部访问，需要做一个映射
- ``kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080``
- ``kubectl get services``查看应用被映射到哪个端口
- Kubernetes是默认的service，端口号是随机分配的

**Scale应用**

- 默认情况下应用只会运行一个副本
- ``kubectl get deployments``查看副本数
- ``kubectl scale deployments/kubernetes-bootcamp --replicas=3``将副本数增加到3个
- ``kubectl get pods``查看当前Pod
- 使用curl进行访问，每次请求发送到不同的Pod，3个副本轮询处理，实现了负载均衡
- ``kubectl scale deployments/kubernetes-bootcamp --replicas=2``将副本数变成2个

**滚动更新**

- ``kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2``将image版本升级到v2
- 通过``kubectl get pods`观察滚动更新到过程，v1的Pod被逐个删除，同时启动了新的v2Pod
- 退回v1版本``kubectl rollout undo deployments/kubernetes-bootcamp``

### 第二章：重要概念

**1. Cluster**

- 是计算、存储和网络资源的集合

**2. Master**

- Master是Cluster的大脑，它的主要职责是调度，决定将应用放在哪里运行
- 为了实现高可用，可以运行多个Master

**3. Node**

- Node的职责是运行容器应用
- Node由Master管理
- Node负责监控并汇报容器的状态
- 根据Master的要求管理容器的生命周期
- 如果Cluster创建的只有一个主机，那么它既是Master也是Node

**4. Pod**

- Pod是Kubernetes的最小工作单元
- 每个Pod包含一个或者多个容器
- Pod中的容器会作为一个整体被Master调度到一个Node上运行

**Kubernetes引入Pod主要有两个目的**

- 可管理性
  - 有些容器天生就是需要紧密联系，一起工作
  - Pod提供了比容器更高层次的抽象，将它们封装到一个部署单元中
- 通信和资源共享
  - Pod中的所有容器使用同一个网络namespace，相同IP和Port
  - 它们可以直接用localhost进行通信，也可以共享存储
  - 当Kubernetes挂载volume到Pod，实际上是把volume挂载到Pod中的每一个容器

**Pod有两种使用方式**

- 运行单一容器
  - one-container-per-Pod是Kubernetes最常见的模型
  - 只是将单个的容器封装成Pod
  - 即便是只有一个容器，Kubernetes也是管理Pod而不是容器

- 运行多个容器
  - 需要关注哪些容器放到一个Pod
  - 这些容器联系必须非常紧密，而且需要直接共享资源

**5. Controller**

- Kubernetes通常不会直接创建Pod，而是通过Controller来管理Pod
- 定义了Pod的部署特性，比如几个副本、在什么样的Node上运行
- Deployment是最常用的Controller，可以管理Pod的多个副本
- ReplicaSet实现了Pod的多副本管理，**使用Deployment时会自动创建ReplicaSet**，Deployment通过ReplicaSet来管理Pod多个副本，通常不需要直接使用ReplicaSet
- DaemonSet用于每个Node最多只运行一个Pod副本的场景，通常用于运行daemon
- StatefuleSet能够保证Pod的每个副本在整个生命周期中名称是不变的，而其他Controller不提供这个功能。当某个Pod发生故障需要删除并重新启动时，Pod的名称会发生变化，同时StatefuleSet会保证副本按照固定的顺序启动、更新或者删除
- Job用于运行结束就删除的应用，其他Controller中的Pod通常时长期持续运行

**6. Service**

- Kubernetes Service定义了外界访问一组特定Pod的方式
- Service有自己的IP和端口
- Kubernetes运行容器（Pod）与访问容器（Pod）这两项任务分别由Controller和Service执行

**7. Namespace**

- Namespace可以将一个物理的Cluster逻辑上划分成多个虚拟Cluster
- 每个Cluster就是一个Namespace，不同Namespace里的资源是隔离的
- Kubernetes默认创建了两个Namespace
- ``kubectl get namespace``
  - default：创建资源时如果不指定，将被放到这个Namespace中
  - kube-system：Kubernetes自己创建的系统资源将被放到这个Namespace中

### 第三章：部署Kubernetes Cluster

**安装Docker**

https://mirrors.tuna.tsinghua.edu.cn/help/docker-ce/

```shell
sudo apt-get remove docker docker-engine docker.io

sudo apt-get install apt-transport-https ca-certificates curl gnupg2 software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

对于 amd64 架构的计算机，添加软件仓库:
sudo add-apt-repository \
   "deb [arch=amd64] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

sudo apt-get update
sudo apt-get install docker-ce
```

**安装Kubernetes**

```shell
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

apt-get update
apt-get install -y kubelet kubeadm kubectl
```

**用kubeadm创建Cluster**

- 在Master上执行``kubeadm init --apiserver-advertise-address 192.168.56.105 --pod-network-cidr=10.244.0.0/16``
- ``--apiserver-advertise-address``指明用Master哪个interface与Cluster的其他节点通信。如果不指定，会自动选择有默认网关的interface
- ``--pod-network--cidr``制定Pod网络的范围
- 初始化过程
  - kubeadm执行初始化前的检查
  - 生成token和证书
  - 生成KubeConfig文件，kubelet需要用这个文件与Master通信
  - 安装Master组件，会从Google的Registry下载组件的Docker镜像
  - 安装附加组件kube-proxy和kube-dns
  - Kubernetes Master初始化成功
  - 提示如何配置kubectl
  - 提示如何安装Pod网络
  - 提示如何注册其他节点到Cluster

**配置kubectl**

kubectl是管理Kubernetes Cluster的命令行工具

```
su - ubuntu
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

启用kubectl命令的自动补全功能：

```shell
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

**安装Pod网络**

要让Kubernetes Cluster能够工作，必须安装Pod网络，否则Pod之间无法通信

部署flannel

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

**添加k8s-node1和k8s-node2**

在node1和node2中分别执行

```shell
kubeadm join --token ....... 192.168.56.105:6443
```

来自--token中的提示，可以通过``kubeadm token list``查看

``kubectl get nodes``查看节点状态

目前所有节点都是NotReady，因为每个节点都需要启动若干组件，这些组件都是中Pod中运行，需要首先从Google下载镜像。

``kubectl get pod --all-namespaces``查看Pod的状态

Pending、ContainerCreating、ImagePullBackOff都表明Pod没有就绪，Running才是就绪状态

通过``kubectl describe pod <Pod Name>``查看Pod的具体状况

