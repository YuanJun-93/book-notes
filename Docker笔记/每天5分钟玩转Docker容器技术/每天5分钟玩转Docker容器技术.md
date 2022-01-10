### 第一章：鸟瞰容器生态系统

#### 1.1 容器生态系统

**容器核心技术**

- 指的是能让Container在host上运行起来的技术
- 容器规范
  - runtime spec
  - image format spec
- 容器runtime
  - 容器真正运行的地方
  - runtime需要跟操作系统kernel紧密协作，为容器提供运行环境
  - lxc、runc和rkt是主流的容器runtime
  - lxc是Linux上老牌的容器runtime。Docker最初也是用lxc作为runtime
  - runc是Docker自己开发的runtime，符合oci标准，也是现在Docker默认的runtime
  - rkt是CoreOS开发的runtime
- 容器管理工具
  - 对内与runtime交互
  - 对外为用户提供interface
  - lxd（lxd）、docker engine（常用的）、rkt cli
- 容器定义工具
  - docker image。runtime依据docker image创建容器
  - dockerfile。包含若干命令的文本文件，通过这些命令创建出docker iamge
  - ACI（App Container Image）
- Registries
  - 容器是通过image创建的，需要有一个仓库来统一存放image
  - Docker Registry
  - Docker Hub。Docker为公众提供的托管Registry
  - Quay.io
- 容器OS

**容器平台技术**

- 容器编排引擎
  - 编排（orchestration），通常包括容器管理、调度、集群定义和服务发现等
  - 通过容器编排引擎，容器被有机地组合成微服务应用，实现业务需求
- 容器管理平台
  - Rancher
  - ContainerShip
- 基于容器的PaaS
  - Deis
  - Flynn
  - Dokku

**容器支持技术**

- 容器网络
  - docker network。Docker原生的网络解决方案
  - flannel
  - weave
  - calico
- 服务发现
  - 保存容器集群中所有微服务最新的信息，比如IP和端口，并对外提供API，提供服务查询功能
  - etcd、consul和zookeeper是典型解决方案
- 监控
  - docker ps/top/stats。是Docker原生的命令行监控工具
  - sysdig、cAdvisor/Heapster和Weave Scope是其他开源解决方案
- 数据管理
  - Rex-Ray
- 日志管理
  - docker logs。Docker原生的日志工具
  - logspout。对日志提供了路由功能。它可以收集不同容器的日志并转发给其他工具进行后处理
- 安全性
  - OpenSCAP。对容器镜像进行扫描，发现潜在的漏洞

#### 1.3 准备实验环境

##### 1.3.2 安装Docker

- ```shell
  sudo apt-get install \ apt-transport-https \ ca-certificates \ curl \ software-properties-common
  ```

- ```shell
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
  ```

- ```shell
  sudo add-apt-repository \
     "deb [arch=amd64] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu \
     $(lsb_release -cs) \
     stable"
  ```

#### 1.4 运行第一个容器

- ``docker run -d -p 80:80 httpd``
- 从Docker Hub下载httpd镜像
- 启动httpd容器，并将容器的80端口映射到host的80端口
- 在daocloud.io注册账户，然后使用加速器
- ``curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io``
- ``systemctl restart docker.service``重启

### 第二章：容器核心知识概述

#### 2.1 What——什么是容器

**容器与虚拟机**

- 两者都是为应用提供封装和隔离
- 容器由两部分组成：应用程序本身和依赖
- 传统的虚拟化技术除了部署应用本身及其依赖，还得安装整个操作系统
- 所有的容器共享同一个Host OS，容器在体积上要比虚拟机小很多

#### 2.2 Why——为什么需要容器

##### 2.2.3 容器的优势

- 开发人员：Build Once、Run Anywhere
- 运维人员：Configure Once、Run Anything

#### 2.3 How——容器是如何工作的

**Docker的核心组件**

客户端和服务器可以运行在同一个Host上，客户端也可以通过Socket或REST API与远程的服务器通信

- Docker客户端：Client
- Docker服务器：Docker daemon
  - 以Linux后台服务的方式运行
  - ``systemctl status docker.service``
  - 负责创建、运行、监控容器、构建、存储镜像
  - 默认情况下，只能响应本地Host的客户端请求，如果要允许远程的话，需要在配置文件中打开TCP监听
    - ``/etc/systemd/system/multi-user.target.wants/docker.service``
    - 在环境变量ExecStart后面添加``-H tcp://0.0.0.0``
    - 重启Docker daemon
    - ``systemctl daemon-reload``、``systemctl restart docker.service``
    - 通信，``docker -H 192.168.56.102 info``
- Docker镜像：Image
- Registry

- Docker容器：Container

``docker images``查看本地的images

``docker ps``或者``docker container ls``显示正在运行的容器

### 第三章：Docker镜像

#### 3.1 镜像的内部结构

##### 3.1.2 base镜像

- 不依赖其他镜像，从scratch创建
- 其他镜像可以以之为基础进行扩展
- 一般base镜像通常都是各种Linux发行版的Docker镜像，Ubuntu、Debian等
- 下载镜像：``docker pull centos``

**Linux操作系统是由内核空间和用户空间组成**

- rootfs

  - 内核空间是kernel，Linux刚启动时会加载bootfs文件系统，之后bootfs会被卸载掉
  - 用户空间的文件系统时rootfs，包括常见的/dev、/proc、/bin等目录
  - 对于base镜像来说，底层直接用host的kennel，只需要自己提供rootfs就可以了

- base镜像提供的是最小安装的Linux发行版

  - ```shell
    FROM scratch
    ADD centos-7-docker.tar.xz /
    CMD ["/bin/bash"]
    ```

  - 第二行ADD指令添加到镜像的var包就是centos7的rootfs

  - 这个tar包会自动解压到/目录下，生成/dev、/proc、/bin等目录

  - 注：可在Docker Hub的镜像描述页面中查看Dockerfile

- 支持运行多种Linux OS

  - 不同Linux发行版的区别主要就是rootfs
  - 例如ubuntu使用upstart管理服务，apt管理软件包，而centos使用systemd和yum。都是用户空间上的区别，Linux kennel差别不大
  - 所以Docker可以同时支持多种Linux镜像

- Base镜像只是在用户空间与发行版一致，kernel版本与发行版是不同的

- 容器只能使用Host的kernel，并且不能修改，如果需要修改，那么虚拟机更合适

##### 3.1.2 镜像的分层结构

- Docker支持通过扩展现有镜像，创建新的镜像

- ```shell
  FROM debian
  RUN apt-get install emacs
  RUN apt-get install apache2
  CMD ["/bin/bash"]
  ```

- 新镜像直接从Debian base镜像上构建

- 最大的好处：共享资源

- 如果多个容器共享一份基础镜像，当某个容器修改了基础镜像的内容，比如/etc下的文件，这个时候，其他的容器的/etc是否也会一起修改？

- 答案是不会，修改会被限制在单个容器内。Copy-on-Write特性

**可写的容器层**

- 当容器启动时，一个新的可写层被加载到镜像的顶部
- 这一层通常被称作“容器层”，容器层之下的都叫“镜像层”
- **所有对容器的改动，都只会发生在容器层，只有容器层是可写的，其他都是只读**

**镜像层数量可能会很多，所有镜像层会联合在一起组成一个统一的文件系统**

- 如果不同层中，有一个相同路径的文件，比如/a，上层的/a会覆盖下层的/a
- 用户只能访问到上层中的文件/a，用户看到的是一个叠加之后的文件系统
  - 添加文件。在容器中创建文件时，新文件被添加到容器层中
  - 读取文件。在容器中读取某个文件时，Docker会从上往下依次在各镜像层中查找此文件。一旦找到，打开并读入内存
  - 修改文件。同上，一旦找到，立即将其复制到容器层，然后修改之
  - 删除文件。同上，找到后，会在容器层中记录下此删除操作
- 只有当修改时才复制一份数据，容器层保存的是镜像变化的部分，不会对镜像本身进行任何修改

#### 3.2 构建镜像

- docker commit命令
- Dockerfile构建文件

##### 3.2.1 docker commit

- ``docker run -it ubuntu``  -it是以交互模式进入容器，并打开终端
- ``apt-get install -y vim`` 在容器中安装vim
- ``docker ps``查看容器名字
- ``docker commit practical_hertz ubuntu-with-vi`` 前面是容器名字，后面是新镜像名
- ``docker images``查看新镜像属性

##### 3.2.2 Dockerfile

Dockerfile是一个文本文件，记录了镜像构建的所有步骤

```shell
FROM ubuntu
RUN apt-get update && apt-get install -y vim
```

```shell
docker build -t ubuntu-with-vi-dockerfile .
# -t 将新镜像命名为ubuntu-with-vi-dockerfile，在当前目录找Dockerfile
# Docker将 build context中的所有文件发送给Docker daemon，build context为镜像构建提供所需要的文件或者目录
# Dockerfile中的ADD、COPY等命令可以将build context中的文件添加到镜像
# 使用build context的时候要注意，不要把多余的文件放到build context
# 第一步：执行FROM，将Ubuntu作为base镜像
# 第二步：执行RUN安装vim
# 启动临时容器安装vim，安装成功后将容器保存为镜像，再把临时容器删掉
# 镜像构建成功
```

``docker history 镜像名``可以查看构建历史，使用dockerfile只是在原镜像的顶部多了一层

**missing表示无法获取IMAGE ID，通常从Docker Hub下载的镜像都有这个问题**

**镜像的缓存特性**

```shell
在上面的基础上加一行
COPY testfile /
```

- 执行``docker build -t ubuntu-with-vi-dockerfile-2 .``会使用镜像，直接在镜像上加一层
- 如果不希望使用缓存，在``docker build``命令加上``--no-cache``
- 镜像是上层依赖下层的，无论什么时候，只要某一层发生变化，其上面所有层的缓存都会失效
- 改变了Dockerfile指令的执行顺序，都会使缓存失效

**调试Dockerfile**

构建过程

1. 从base镜像运行一个容器
2. 执行一条命令，对容器做修改
3. 执行类似docker commit的操作，生成一个新的镜像层
4. Docker再基于刚刚提交的镜像运行一个新容器
5. 重复2-4步，直到Dockerfile中的所有指令执行完毕

**如果执行到某个指令失败了，我们也能得到前一个指令成功执行构建出的镜像，这对调试比较有用**

**Dockerfile常用指令**

**FROM**

- 指定base镜像

**MAINTAINER**

- 设置镜像的作者，可以是任意字符串

**COPY**

- 将文件从build context复制到镜像
- 支持两种形式：COPY src dest与COPY["src","dest"]
- src只能指定build context中的文件或目录

**ADD**

- 与COPY类似，从build context复制文件到镜像
- 如果src是归档文件（tar、zip、tgz、xz等），文件会被自动解压到dest

**ENV**

- 设置环境变量，环境变量可被后面的指令使用
- ``EVN MY_VERSION 1.3 RUN apt-get install -y mypackage=$MY_VERSION``

**EXPOSE**

- 指定容器中的进程会监听某个端口，Docker可以将该端口暴露出来

**VOLUME**

- 将文件或目录声明为volume

**WORKDIR**

- 为后面的RUN、CMD、ENTRYPOINT、ADD或COPY指令设置镜像中的当前工作目录

**RUN**

- 在容器中运行指定的命令

**CMD**

- 容器启动时运行指定的命令
- Dockerfile中可以有多个CMD指令，但只有最后一个生效
- CMD可以被docker run之后的参数替换

**ENTRYPOINT**

- 设置容器启动时运行的命令
- Dockerfile可以有多个ENTRYPOINT指令，但只有最后一个生效
- CMD或docker run之后的参数会被当作参数传递给ENTRYPOINT

完整的Dockerfile

```shell
FROM busybox
MAINTAINER louis
WORKDIR /testdir
RUN touch tmpfile1
COPY ["tmpfile2","."]
ADD ["bunch.tar.gz","."]
ENV WELCOME "You are in my container, welcome!"
```

#### 3.3 RUN vs CMD vs ENTRYPOINT

- RUN：执行命令并创建新的镜像层，RUN经常用于安装软件包
- CMD：设置容器启动后默认执行的命令及其参数，但CMD能够被docker run后面跟的命令行参数替换
- ENTRYPOINT：配置容器启动时运行的命令

##### 3.3.1 Shell和Exec格式

**Shell**

- <instruction> <command>

- ```shell
  RUN apt-get install python3
  CMD echo "Hello world"
  ENTRYPOINT echo "Hello world"
  ```

- shell格式底层会调用/bin/sh -c [command]

- ``ENV name Louis ENTRYPOINT echo "Hello, $name"``

- 输出``Hello, Louis``

**Exec**

- <instruction> ["executable", "param1", "param2", ...]

- ```shell
  RUN ["apt-get","install","python3"]
  CMD ["/bin/echo", "Hello world"]
  ENTRYPOINT ["/bin/echo", "Hello world"]
  ```

- 当指令执行时，会直接调用[commnd]，不会被shell解析

- ``ENV name Louis ENTRYPOINT ["/bin/echo", "Hello, $name"]``

- 输出``Hello, $name``，环境变量name没有替换

- 如果希望使用环境变量``ENV name Louis ENTRYPOINT ["/bin/sh","-c","echo Hello, $name"]``

**CMD和ENTRYPOINT推荐使用Exec格式，因为指令可读性更强，RUN两种都可以**

##### 3.3.2 RUN

- ``RUN apt-get update && apt-get install -y \bzr\cvx\git\mercurial\subversion``
- ``apt-get update``和``apt-get install``被放在一个RUN指令中执行，这样保证每次都是最新的包

##### 3.3.3 CMD

- 如果docker run指定了其他命令，CMD指定的默认命令将被忽略
- 如果Dockerfile中有多个CMD命令，只有最后一个CMD有效
- 三种格式
  1. Exec格式：``CMD ["executable","param1","param2"]`` 推荐格式
  2. ``CMD ["param1","param2"]``为ENTRYPOINT提供额外的参数，此时ENTRYPOINT必须使用Exec格式
  3. Shell格式：CMD command param1 param2

##### 3.3.4 ENTRYPOINT

- 两种格式
  1. 推荐格式，Exec格式：``ENTRYPOINT ["executable", "param1", "param2"]``
  2. Shell格式：``ENTRYPOINT command param1 param2``，会忽略CMD或docker run的参数
  3. 格式选择的时候要小心，因为效果差别很大

- Exec格式用于设置要执行的命令及其参数，同时可通过CMD提供额外的参数
- ENTRYPOINT中的参数始终都会被使用，而CMD的额外参数可以在容器启动时动态替换掉
  - ``ENTRYPOINT ["/bin/echo", "Hello"] CMD ["world"]``
  - ``docker run -it [image]``，输出为：Hello world
  - ``docker run -it [image] Louis``，输出：Hello Louis

##### 3.3.5 最佳实践

1. 使用RUN指令安装应用和软件包，构建镜像
2. 如果Docker镜像的用途是运行程序或服务，比如运行一个MySQL，应该优先使用Exec格式的ENTRYPOINT指令。CMD可为ENTRYPOINT提供额外的默认参数，同时可利用docker run命令行来替换默认参数
3. 如果想为容器设置默认的启动命令，可使用CMD指令。用户可在docker run命令行中替换此默认命令

#### 3.4 分发镜像

##### 3.4.1 为镜像命名

一个特定镜像的名字由两部分组成：repository和tag

- ``[image name] = [repository]:[tag]``

如果执行docker build时，没有指定tag，会使用默认值latest，效果相当于

- ``docker build -t ubuntu-with-vi:latest``

tag常用于描述镜像的版本信息

**latest tag**

- latest并没有什么特殊的含义。当没有指明镜像tag时，Docker会使用默认值latest
- 虽然很多repository将latest作为稳定最新版本的别名，这只是一个普通约定
- 使用镜像时避免使用latest，最好还是指定某个tag，例如：httpd:2.3

通过docker tag命令给镜像打tag

```shell
docker tag myimage-v1.9.1 myimage:1 docker tag myimage-v1.9.1 myimage:1.9 docker tag myimage-v1.9.1 myimage:1.9.1 docker tag myimage-v1.9.1 myimage:latest
```

- myimage：1 始终指向1这个分支中最新的镜像
- myimage：1.9  始终指向1.9x中最新的镜像
- myimage：latest始终指向所有版本中最新的镜像

##### 3.4.2 使用公共Registry

在Docker Host上登陆 ``docker login -i louisyuan``

修改镜像的repository，使之与Docker Hub账户匹配

``docker tag httpd louisyuan/httpd:v1``

通过docker push，将镜像上传到Docker Hub

``docker push louisyuan/httpd:v1``

如果要删除，就只能在Docker Hub界面操作了

这个镜像就可以被其他人下载使用了，``docker pull louisyuan/httpd:v1``

##### 3.4.3 搭建本地Registry

**启动Registry容器**

``docker run -d -p 5000:5000 -v /myregistry:/var/lib/registry registry:2``

- -d后台启动容器
- -p将容器的5000端口映射到Host的5000端口
- -v将容器/var/lib/registry目录映射到Host的/myregistry

重命名镜像

``docker tag louisyuan/httpd:v1 registry.example.net:5000/louisyuan/httpd:v1``

- repository完整格式``[registry-host]:[port]/[username]/xxx``

#### 3.5 小结

- images：显示镜像列表
- history：显示镜像构建历史
- commit：从容器创建新镜像
- build：从Dockerfile构建镜像
- tag：给镜像打tag
- pull：从registry下载镜像
- push：将镜像上传到registry
- rmi：删除Docker host中的镜像
- search：搜索Docker Hub中的镜像

**rm1**

- 只能删除host上的镜像，不会删除registry的镜像
- 如果一个镜像对应了多个tag，只有当最后一个tag被删除，才是真的删除

**search**

- 不需要打开浏览器，在命令行中就可以搜索Docker Hub中的镜像，如果想知道有哪些tag，还是得访问Docker Hub
- 例如``docker search httpd``

### 第四章：Docker容器

#### 4.1 运行容器

- CMD指令
- ENTRYPOINT指令
- 在docker run命令行中指定

例如执行``docker run ubuntu pwd``，返回的/是容器中的当前目录

使用``docker ps -a``或``docker container ls -a``查看

##### 4.1.1 让容器长期运行

因为容器的生命周期依赖于启动时执行的命令，只要该命令不结束，容器也就不会退出

以后台的方式启动容器，加上参数-d，``docker run -d ubuntu``

```shell
root@louis:~# docker run -d ubuntu /bin/bash -c "while true ; do sleep 1; done"
ca2b25c60e5a9a5fdaf7266c0c17df63c9cc60e883ef8e8ce30a62f7a7ee9e39
```

```shell
docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS     NAMES
ca2b25c60e5a   ubuntu    "/bin/bash -c 'while…"   3 seconds ago   Up 3 seconds             compassionate_dubinsky
```

- 这个CONTAINER ID是容器的短ID，启动容器的时候返回的是长ID，短ID是长ID前12个字符
- NAMES字段显示容器的名字，在启动容器的时候可以通过--name参数显式地为容器命名，如果不指定，docker就会自动为容器分配名字
- 停止容器``docker stop 容器ID或者名称``

##### 4.1.2 两种进入容器的方法

**docker attach**

- 可以attach到容器启动命令的终端

- ```shell
  root@louis:~# docker run -d ubuntu /bin/bash -c "while true ; do sleep 1; echo i_am_in_container;  done"
  18307622be57f3eae8efde01c57afdb0480582f81baf45ee3d3e24f442e30bef
  root@louis:~# docker attach 18307622be57f3eae8efde01c57afdb0480582f81baf45ee3d3e24f442e30bef
  i_am_in_container
  i_am_in_container
  i_am_in_container
  ```

- 可以通过Ctrl+p，然后Ctrl+q退出attach终端（我直接软件都退出去了），容器并没有停止

**docker exec**

- ```shell
  root@louis:~# docker run -d ubuntu /bin/bash -c "while true; do sleep 1; echo i am in container; done"
  4057234aba2232ea7ffeb6ab441342a54494e54f467831c9788ac514fc5bffe0
  root@louis:~# docker ps
  CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS     NAMES
  4057234aba22   ubuntu    "/bin/bash -c 'while…"   3 seconds ago   Up 2 seconds             nifty_cori
  root@louis:~# docker exec -it 4057234aba22 bash
  root@4057234aba22:/# ps -elf
  F S UID          PID    PPID  C PRI  NI ADDR SZ WCHAN  STIME TTY          TIME CMD
  4 S root           1       0  0  80   0 -   994 do_wai 00:57 ?        00:00:00 /bin/bash -c while true; do sleep 1; echo i
  4 S root          29       0  0  80   0 -  1027 do_wai 00:57 pts/0    00:00:00 bash
  0 S root          40       1  0  80   0 -   627 hrtime 00:57 ?        00:00:00 sleep 1
  4 R root          41      29  0  80   0 -  1474 -      00:57 pts/0    00:00:00 ps -elf
  ```

- -it以交互模式打开pseudo-TTY，执行bash，就是打开一个bash终端

- 可以像在普通Linux中一样执行命令

- exit退出容器

- 常用方式：``docker exec -it <container> bash | sh``

**attach与exec的区别**

- attach直接进入容器启动命令的终端，不会启动新的进程
- exec是在容器中打开新的终端，并且可以启动新的进程
- 如果新直接在终端查看启动命令的输出，用attach，其他情况用exec
- 如果只是为了看输出可以用``docker logs -f 容器id``  -f是持续打印

##### 4.1.3 运行容器的最佳实践

**服务类容器以daemon的形式运行，对外提供服务**

- 如果要排查问题，通过exec -it进入容器

**工具类容器通常给我们提供一个临时的工作环境**

- 通常以run -it运行

#### 4.2 stop/start/restart容器

容器在docker host中实际上是一个进程，docker stop命令本质上是向该进程发送一个``SIGTERM``信号

如果想快速停止容器，可以使用docker kill命令，其作用是向容器进程发送``SIGKILL``信号

对于处于停止状态的容器，可以通过docker start重新启动，**docker start会保留容器的第一次启动时的所有参数**

docker restart的作用：依次执行docker stop和docker start

- 当容器因为某种错误停止运行，对于这类服务类容器，希望能够自动重启
- ``docker run -d --restart=always httpd``
- ``--restart=always``无论怎么退出，都会立即重启，也可以设置重启次数
- ``--restart=on-failure:3``

#### 4.3 pause/unpause容器

有的时候希望让容器暂停运行一段时间，比如要对容器的文件系统打个快照，或者docker host需要使用CPU，这个时候就可以执行docker pause

处于暂停状态的容器不会占用CPU资源，直到通过docker unpause恢复运行

#### 4.4 删除容器

docker使用一段时间后，host上会有大量已经退出了的容器

这些容器依然会占用host的文件系统资源，如果确认已经不会再启动此类容器，可以通过docker rm删除

批量删除所有已经退出的容器

``docker rm -v $(docker ps -aq -f status=exited)``

``docker rm``是删除容器，``docker rmi`是删除镜像

#### 4.5 State Machine

- 可以先创建容器，稍后再启动
  - ``docker create httpd``
  - docker create创建的容器处于Create状态
  - docker start将以后台方式启动容器
  - **docker run实际上是docker create和docker start的组合**
- 只有当容器的启动进程退出时，-- restart才会生效
  - 正常退出或者OOM掉会根据策略判断是否重启
  - docker stop和docker kill是不会重启的

#### 4.6 资源限制

##### 4.6.1内存限额

**容器可使用的内存包括两部分：物理内存和swap**

- -m或--memory：设置内存的使用限额，例如100MB，2GB
- --memory-swap：设置内存+swap的使用限额
- ``docker run -m 200M 00memory-swap=300M ubuntu``
  - 允许该容器最多使用200MB内存和100MB的swap
  - 默认情况下上面两组参数为-1，对容器内层和swap没有限制

使用progrium/stress镜像来学习如何为容器分配内存，该镜像可以做压力测试

```shell
docker run -it -m 200M --memory-swap=300M progrium/stress --vm 1 --vm-bytes 280M
```

- --vm  1 ：启动1个内存工作线程
- --vm-bytes  280M ：每个线程分配280MB内存

**ubuntu遇到问题：WARNING: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.**

- ubuntu默认没开启swap。(开启后会使系统内存占用多1%,性能下降约10%,即使没有运行docker)
- 解决方案
  - ``sudo -i``获取系统sudo权限
  - 修改系统的``/etc/default/grub``文件，在末尾差插入``GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"``
  - 更新系统的GRUB，``sudo update-grub``

```Shell
root@louis:~# docker run -it -m 200M --memory-swap=300M progrium/stress --vm 1 --vm-bytes 280M
stress: info: [1] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd
stress: dbug: [1] using backoff sleep of 3000us
stress: dbug: [1] --> hogvm worker 1 [7] forked
stress: dbug: [7] allocating 293601280 bytes ...
stress: dbug: [7] touching bytes in strides of 4096 bytes ...
stress: FAIL: [1] (416) <-- worker 7 got signal 9
stress: WARN: [1] (418) now reaping child worker processes
stress: FAIL: [1] (422) kill error: No such process
stress: FAIL: [1] (452) failed run completed in 0s
```

- 分配的内存超过限额，stress线程报错，容器退出
- 如果在启动容器时只指定-m而不指定-- memory-swap，那么--memory-swap默认为-m的两倍
  - ``docker run -it -m 200M ubuntu``
- 容器最多使用200MB物理内存和200MBswap

##### 4.6.2 CPU限额

默认设置下，所有容器可以平等的使用hostCPU资源并没有限制

Docker通过-c或--cpu-shares设置容器使用CPU的权重，**如果不指定，默认值为1024**

**与内存限额不同，通过-c设置的cpu share不是CPU的绝对数量，而是一个权重值**

通过cpu share可以设置容器使用CPU的优先级

```shell
docker run --name "containerA" -c 1024 ubuntu docker run --name "containerB" -c 512 ubuntu
```

- 当两个container都需要CPU的时候，A可以得到的是B的两倍
- 这种按权重分配只会发生在CPU资源紧张的情况下
- 如果A处于空闲状态，这个时候为了充分利用CPU资源，B也可以分配到所有CPU

```shell
root@louis:~# docker run --name container_A -it -c 1024 progrium/stress --cpu 1
stress: info: [1] dispatching hogs: 1 cpu, 0 io, 0 vm, 0 hdd
stress: dbug: [1] using backoff sleep of 3000us
stress: dbug: [1] --> hogcpu worker 1 [7] forked
```

```shell
root@louis:~# docker run --name containerB -it -c 512 progrium/stress --cpu 1
stress: info: [1] dispatching hogs: 1 cpu, 0 io, 0 vm, 0 hdd
stress: dbug: [1] using backoff sleep of 3000us
stress: dbug: [1] --> hogcpu worker 1 [7] forked
```

```shell
top - 11:12:43 up 31 min,  3 users,  load average: 1.65, 0.66, 0.25
Tasks:  95 total,   3 running,  92 sleeping,   0 stopped,   0 zombie
%Cpu(s): 99.0 us,  1.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   1906.2 total,   1266.6 free,    194.4 used,    445.2 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   1559.2 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                     
   1242 root      20   0    7312     96      0 R  65.7   0.0   1:55.22 stress                      
   1359 root      20   0    7312    100      0 R  33.0   0.0   0:22.37 stress         
```

**containerA消耗的CPU是containerB的两倍**

##### 4.6.3 Block IO带宽限额

Block IO指的是磁盘的读写，**目前Block IO限额只对direct IO（不使用文件缓存）有效**

**block IO权重**

- 通过设置--blkio-weight参数来改变容器block IO的优先级

- --blkio-weight与--cpu-shares类似，设置的是权重值默认为500

- ```shell
  docker run -it --name container_A --blkio-weight 600 ubuntu docker run -it --name container_B --blkio-weight 300 ubuntu
  ```

**限制bps和iops**

- bps是byte per second，每秒读写的数据量
- iops是io per second，每秒IO的次数

- ``--device-read-bps``：限制读某个设备的bps
- ``--device-write-bps``：限制写某个设备的bps
- ``--device-read-iops``：限制读某个设备的iops
- ``--device-write-iops``：限制写某个设备的iops

```shell
# 限制容器写/dev/sda的速率为30MB/s
docker run -it --device-write-bps /dev/sad:30MB ubuntu
```

```shell
root@louis:~# time dd if=/dev/zero of=test.out bs=1M count=800 oflag=direct
800+0 records in
800+0 records out
838860800 bytes (839 MB, 800 MiB) copied, 7.511 s, 112 MB/s

real    0m7.513s
user    0m0.003s
sys     0m0.197s
```

#### 4.7 实现容器的底层技术

##### 4.7.1 cgroup（实现资源限额）

全称Control Group，Linux操作系统通过cgroup可以设置进程使用CPU、内存和IO资源的限额

cgroup在``/sys/fs/cgroup``中，在``/sys/fs/cgroup/cpu/docker``目录中，Linux会为每个容器创建一个cgroup目录，以容器长ID命名

```shell
# 起一个容器，设置cpu为512
root@louis:~# docker run -it --cpu-shares 512 progrium/stress -c 1
stress: info: [1] dispatching hogs: 1 cpu, 0 io, 0 vm, 0 hdd
stress: dbug: [1] using backoff sleep of 3000us
stress: dbug: [1] --> hogcpu worker 1 [7] forked

# 找到ID
root@louis:~# docker ps
CONTAINER ID   IMAGE             COMMAND                  CREATED         STATUS         PORTS     NAMES
8705dfc31384   progrium/stress   "/usr/bin/stress --v…"   9 seconds ago   Up 8 seconds             peaceful_sinoussi

# 查看容器下面的配置目录
root@louis:~# ls /sys/fs/cgroup/cpu/docker/8705dfc313844fccb524be92a72e9a31d11c4d7042a195af6392c5eea562234f/
cgroup.clone_children  cpuacct.usage_percpu       cpu.cfs_period_us  cpu.uclamp.min
cgroup.procs           cpuacct.usage_percpu_sys   cpu.cfs_quota_us   notify_on_release
cpuacct.stat           cpuacct.usage_percpu_user  cpu.shares         tasks
cpuacct.usage          cpuacct.usage_sys          cpu.stat
cpuacct.usage_all      cpuacct.usage_user         cpu.uclamp.max

# 查看配置值
root@louis:~# cat /sys/fs/cgroup/cpu/docker/8705dfc313844fccb524be92a72e9a31d11c4d7042a195af6392c5eea562234f/cpu.shares 
512
```

**``/sys/fs/cgroup/memory/docker``和``/sys/fs/cgroup/blkio/docker``中保存的是内存以及block IO的cgroup配置**

##### 4.7.2 namespace（资源隔离）

Linux有6种namespace

**1. Mount namespace**

- 让容器看上去拥有整个文件系统
- 容器有自己的/目录，可以执行mount和umount。只在当前容器中生效

**2. UTS namespace**

- 让容器有自己的hostname
- 默认情况下容器的hostname是短ID
- 可以通过-h或--hostname参数设置
- ``docker run -h myhost -it ubuntu``

**3. IPC namespace**

- 让容器拥有自己的共享内存和信号量（semaphore）来实现进程间通信
- 不会与host和其他容器的IPC混在一起

**4. PID namespace**

使用``ps axf``查看容器进程，所有容器进程挂到dockerd进程下，同时也能看到容器自己的子进程

```shell
   2154 ?        Sl     0:00 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 186569c5331978228
   2183 ?        Ss     0:00  \_ httpd -DFOREGROUND
   2215 ?        Sl     0:00      \_ httpd -DFOREGROUND
   2216 ?        Sl     0:00      \_ httpd -DFOREGROUND
   2217 ?        Sl     0:00      \_ httpd -DFOREGROUND
```

容器中PID=1的进程不是host的init进程，容器拥有自己独立的一套PID

##### 5. Network namespace

- 容器拥有自己独立的网卡、IP、路由等资源

##### 6. User namespace

- 让容器能够管理自己的用户，host不能看到容器中创建的用户

### 第五章：Docker网络

Docker在安装的时候，会自动在host上创建三个网络

```shell
root@louis:~# docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
5773b127df3a   bridge    bridge    local
70e1bb1365ea   host      host      local
5d6febbdceaf   none      null      local
```

#### 5.1 none网络

挂在这个网络下的容器除了lo，没有其他任何网卡，创建容器时，通过--network=none指定使用none网络

```shell
root@louis:~# docker run -it --network=none busybox
/ #
/ # ifconfig
lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

应用场景：安全性较高的并且不需要联网的应用，比如某个容器的唯一用途是生成随机密码，就可以放到none网络中避免密码被窃取

#### 5.2 host网络

在容器中可以看到host的所有网卡，并且连hostname也是host的

应用场景：

- 当容器对网络传输效率有较高要求，可以选择hsot网络
- 会牺牲一些灵活性，比如要考虑端口冲突问题，docker host上已经使用的端口就不能再使用了

#### 5.3 bridge网络

- Docker安装时会创建一个命名为docker0的Linux bridge
- 如果不指定-- network，创建容器默认都会挂到docker0

安装``apt install bridge-utils``

```shell
root@louis:~# brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.02423e66802e       no
root@louis:~# docker run -d httpd
57197adde85b0acfd61411a317b64dc98eb2423b1586e12d6bfba1eb7e4448eb
root@louis:~# brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.02423e66802e       no              veth2beb276
```

当创建了一个容器后，就会有一个新的网络接口挂到docker0上，这个就是新容器的网卡

查看容器的网络配置``docker exec -it 容器id bash``，容器有一个网卡eth0@if34，并不是刚刚看到的虚拟网卡。

实际上，这两个是一对veth pair，一头在容器中，另一头挂在网桥docker0上，其效果就是将eth0@if34也挂在docker0上

**网桥就是网关**

- IP：172.17.0.1
- 子网：172.17.0.0/16
- 容器创建时，docker会自动从172.17.0.0/16中分配一个IP，这里16位的掩码保证有足够多的IP可以供容器使用

#### 5.4 user-defined网络

Docker提供三种user-defined网络驱动，用于创建跨主机的网络

- bridge、overlay
- macvlan、overlay
- macvlan

通过bridge驱动创建类似前面默认的bridge网络

```Shell
root@louis:~# docker network create --driver bridge my_net
d57f6e1d3d80a2b827d0174f8878e754cad21f242494afc11c97d22c1a8d89d3
root@louis:~# brctl show
bridge name     bridge id               STP enabled     interfaces
br-d57f6e1d3d80         8000.02426448c8f2       no
docker0         8000.02423e66802e       no              veth2beb276
root@louis:~# docker network inspect my_net
[
    {
        "Name": "my_net",
        "Id": "d57f6e1d3d80a2b827d0174f8878e754cad21f242494afc11c97d22c1a8d89d3",
        "Created": "2021-12-22T18:29:27.866272047+08:00",
        "Scope": "local",
        "Driver": "bridge",
```

新增了一个网桥br-d57f6e1d3d80，这里的d57f6e1d3d80正好是新建bridge网络my_net的短id。查看my_net的配置信息，Driver刚好是刚刚设置的bridge

**可以自己指定IP网段**

``docker network create --driver bridge --subnet 172.22.16.0/24 --gateway 172.22.16.1 my_net2``

**当容器需要使用新网络的时候，需要用--network指定**

``docker run -it --network=my_net2 busybox``

可以指定静态IP，通过--ip指定，**只有使用--subnet创建的网络才能指定静态IP**

``docker run -it --network=my_net2 --ip 172.22.16.8 busybox``

查看路由表 ``ip r``

给容器添加网卡``docker network connect my_net2 容器ID``，现在可以相互通信

#### 5.5 容器间通信

容器之间可通过IP、Docker DNS Server或joined容器三种方式通信

##### 5.5.2 Docker DNS Server

从Docker 1.10版本开始，docker daemon实现了一个内嵌的DNS server，使容器可以通过容器名通信。方法很简单，只需要在启动的时候用--name为容器命名就可以了

```shell
docker run -it --network=my_net2 --name=bbox1 busybox docker run -it --network=my_net2 --name=bbox2 busybox


root@louis:~# docker run -it --network=my_net2 --name=box1 busybox
/ # ping -c 3 box2
PING box2 (172.22.16.3): 56 data bytes
64 bytes from 172.22.16.3: seq=0 ttl=64 time=0.092 ms
64 bytes from 172.22.16.3: seq=1 ttl=64 time=0.077 ms
64 bytes from 172.22.16.3: seq=2 ttl=64 time=0.093 ms
```

bbox2可以直接ping到bbox1，**不过，docker DNS只能在user-defined网络中使用**

**默认的bridge网络是无法使用DNS的**

##### 5.5.3 joined容器

它可以使两个或多个容器共享一个网络栈，共享网卡和配置信息，joined容器之间可以通过127.0.0.1直接通信

先创建一个httpd容器，名字为web1，``docker run -d -it --name=web1 httpd``

然后创建busybox容器并通过--network=container:web1指定joined容器为web1

``docker run -it --network=container:web1 busybox``

适用场景：

- 不同容器中的程序希望通过loopback高效快速的通信，比如Web Server与App Server
- 希望监控其他容器的网络流量，比如运行在独立容器中的网络监控程序

#### 5.6 将容器与外部世界连接

##### 5.6.1 容器访问外部世界

查看路由表 ``iptables -t nat -S``

容器访问外网的流程

- busybox发送ping包：172.17.0.2 -> www.bing.com
- docker0收到包，发现是发送到外网的，交给NAT处理
- NAT将源地址换成enp0s3的IP：10.0.2.15 -> www.bing.com
- ping包从enp0s3发送出去，到达www.bing.com

通过NAT，docker实现了容器对外网的访问

##### 5.6.2 外部世界访问容器

外网如何访问到容器？ ——端口映射

docker可将容器对外提供服务的端口映射到host的某个端口，外网通过该端口访问容器

容器启动时通过-p参数映射端口 ``docker run -d -p 80 httpd``

```shell
root@louis:~# docker run -d -p 80 httpd
7e8d4663e45594dcfc6a5211289735d88af8a05789a20d3fbaf7fc67e80d6557
root@louis:~# docker ps
CONTAINER ID   IMAGE     COMMAND              CREATED         STATUS         PORTS                                     NAMES
7e8d4663e455   httpd     "httpd-foreground"   3 seconds ago   Up 2 seconds   0.0.0.0:49153->80/tcp, :::49153->80/tcp   mystifying_khayyam
```

可以通过``<host ip>:<49153>``访问客户端的web服务

除了映射动态端口，也可以在-p中指定映射到host某个特定端口

``docker run -d -p 8080:80 httpd`` 将80端口映射到host的8080端口

每一个映射的端口，host都会启动一个docker-proxy进程来处理访问容器的流量

``ps -ef | grep docker-proxy``

- docker-proxy监听host的49153端口
- 当curl访问127.0.0.1:49153时，docker-proxy转发给容器172.17.0.2:80
- httpd容器响应请求并返回结果

### 第六章：Docker存储

#### 6.1 storage driver

分层结构使镜像和容器的创建、共享以及分发变得非常高效，而这些都要归功于Docker storage driver，实现了多层数据的堆叠，为用户提供一个单一的合并之后的统一视图

- 没有哪个driver能够适应所有的场景
- driver本身在快速发展和迭代

Docker安装时会根据当前系统的配置选择默认的driver。

查看ubuntu默认driver，``docker info``

- 比如无状态的应用，放在storage driver维护的层中是很好的选择
- 无状态意味着容器没有需要持久化的数据，随时可以从镜像直接创建

#### 6.2 Data Volume

Data Volume本质上是Docker Host文件系统中的目录或文件，能够直接被mount到容器的文件系统中

- Data Volume是目录或文件，而非没有格式化的磁盘（块设备）
- 容器可以读写volume中的shuj
- volume数据可以被永久地保存，即使使用它的容器已经销毁

如何设置volume的容量？

- volume的容量取决于文件系统当前未使用的空间，目前没有方法设置volume的容量

##### 6.2.1 bind mount

将host上已存在的目录或文件mount到容器

通过-v将其mount到httpd容器

``docker run -d -p 80:80 -v ~/htdocs:/usr/local/apache2/htdocs httpd``

- -v的格式：<host path>:<container path>
- /usr/local/apache2/htdocs就是Apache Server存放静态文件的地方
- 由于/usr/local/apache2/htdocs已经存在，原有数据会被隐藏起来，取而代之的是host $HOME/htdocs/中的数据，跟Linux mount命令的行为是一样的

bind mount可以让host与容器共享数据，把容器删掉，bind mount也还在，因为是host文件系统中的数据

- bind mount可以指定数据的读写权限，默认是可读可写
- ``-v ~/htdocs:/usr/local/apache2/htdocs:ro``指定为只读
- 也可以单独指定文件

bind mount缺点

- 需要指定host文件系统的特定路径，限制了容器的可移植性
- 当需要将容器迁移到其他host，而该host没有要mount的数据或者数据不在相同的路径时，操作会失败
- 移植性更好的是docker managed volume

##### 6.2.2 docker managed volume

**docker managed volume与bind mount在使用上最大的区别是不用指定mount源**

指明mount point就可以了

```shell
root@louis:~# docker run -d -p 80:80 -v /usr/local/apache2/htdocs httpd
83ec66fb0a8e8067c0ee67be33c162d4f3ab02a69dd3372d5a56b356d31f062b
```

**通过-v告诉docker需要一个data volume，将其mount到/usr/local/apache2/htdocs**

通过docker inspect找到data volume在哪

```shell
root@louis:~# docker inspect 83ec66fb0a8e8067c0ee67be33c162d4f3ab02a69dd3372d5a56b356d31f062b
[
    {
        ......
        "Mounts": [
            {
                "Type": "volume",
                "Name": "027a2d7b84fffd9a672eacf03669edc2c9b70d068daf24abf0d79d4d6075cf08",
                "Source": "/var/lib/docker/volumes/027a2d7b84fffd9a672eacf03669edc2c9b70d068daf24abf0d79d4d6075cf08/_data",
                "Destination": "/usr/local/apache2/htdocs",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ],
        ......
]
```

- Source就是volume在host上的目录
- 每当容器申请mount docker managed volume时，docker都会在/var/lib/docker/volumes下生成一个目录

查看volume有什么东西

```shell
root@louis:~# ls -l /var/lib/docker/volumes/027a2d7b84fffd9a672eacf03669edc2c9b70d068daf24abf0d79d4d6075cf08/_data
total 4
-rw-r--r-- 1 504 staff 45 Jun 12  2007 index.html
```

**volume的内容跟容器原有/usr/local/apache2/htdocs完全一样**

- 如果mount point指向的是已有目录，原有数据会被复制到volume中
- **此时的``/usr/local/apache2/htdocs``已不再是由storage driver管理的层数据了**
- 它已经是一个data volume，可以像bind mount一样对数据进行操作

**docker managed volume的创建过程**

- 容器启动时，简单的告诉docker，我需要一个volume存放数据，帮我mount到目录/abc
- docker在/var/lib/docker/volumes中生成一个随机目录作为mount源
- 如果/abc已经存在，则将数据复制到mount源
- 将volume mount到/abc

**查看volume**

```shell
root@louis:~# docker volume ls
DRIVER    VOLUME NAME
local     027a2d7b84fffd9a672eacf03669edc2c9b70d068daf24abf0d79d4d6075cf08
root@louis:~# docker volume inspect 027a2d7b84fffd9a672eacf03669edc2c9b70d068daf24abf0d79d4d6075cf08
[
    {
        "CreatedAt": "2021-12-23T09:20:37+08:00",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/027a2d7b84fffd9a672eacf03669edc2c9b70d068daf24abf0d79d4d6075cf08/_data",
        "Name": "027a2d7b84fffd9a672eacf03669edc2c9b70d068daf24abf0d79d4d6075cf08",
        "Options": null,
        "Scope": "local"
    }
]
```

**docker volume只能查看docker managed volume，看不到bind mount，也不知道volume对应的容器，这些信息还是得靠docker inspect**

#### 6.3 数据共享

##### 6.3.1 容器与host共享数据

bind mount很明确的将要共享的目录mount到容器

docker managed volume比较麻烦一点，因为volume位于host中目录，是在容器启动时才生成，所以需要将共享数据复制到volume中

``docker cp ~/htdocs/index.html 容器ID:/usr/local/apache2/htdocs``

**docker cp可以在容器与host之间复制数据**

##### 6.3.2 容器之间共享数据

将共享数据放到bind mount中，然后将其mount到多个容器

- 将$HOME/htdocs mount到三个httpd容器

- ```shell
  root@louis:~# docker run --name web1 -d -p 80 -v ~/htdocs:/usr/local/apache2/htdocs httpd
  d868875c1aa5f79373569cb47fc80af574653e8352e1cae608d00912adaacca3
  root@louis:~# docker run --name web2 -d -p 80 -v ~/htdocs:/usr/local/apache2/htdocs httpd
  57444b5e866e16a0dbeacdcf2f7d3e7e8d88dfd9e3a71686928dd8ac378a8157
  root@louis:~# docker run --name web3 -d -p 80 -v ~/htdocs:/usr/local/apache2/htdocs httpd
  4a472edf596c042266a0fea9a8236f99219142fc2ee1eeba45ed61e4846ebe80
  root@louis:~# echo "This is a new index page for web cluster" > ~/htdocs/index.html
  root@louis:~# docker ps
  CONTAINER ID   IMAGE     COMMAND              CREATED              STATUS              PORTS                                     NAMES
  4a472edf596c   httpd     "httpd-foreground"   45 seconds ago       Up 44 seconds       0.0.0.0:49156->80/tcp, :::49156->80/tcp   web3
  57444b5e866e   httpd     "httpd-foreground"   54 seconds ago       Up 53 seconds       0.0.0.0:49155->80/tcp, :::49155->80/tcp   web2
  d868875c1aa5   httpd     "httpd-foreground"   About a minute ago   Up About a minute   0.0.0.0:49154->80/tcp, :::49154->80/tcp   web1
  root@louis:~# curl 127.0.0.1:49156
  This is a new index page for web cluster
  root@louis:~# curl 127.0.0.1:49155
  This is a new index page for web cluster
  root@louis:~# curl 127.0.0.1:49154
  This is a new index page for web cluster
  ```

另一种在容器之间共享数据的方式是使用volume container

#### 6.4 volume container

Volume container是专门为其他容器提供volume的容器。**提供的卷可以是bind mount，也可以是docker managed volume**

创建volume container

```shell
root@louis:~# docker create --name vc_data -v ~/htdocs:/usr/local/apache2/htdocs -v /other/useful/tools busybox
5d2f5c02f121f81e1a4e5b2a4607239d341ef5fb7a14843f1d0a6d673ccc5a1a
```

- 容器命名为vc_data（vc是volume container的缩写）
- 因为volume container的作用只是提供数据，所以不需要运行状态，使用的是craete命令
- 容器mount了两个volume
  - bind mount，用来存放Web Server的静态文件
  - docker managed volume，存放一些实用工具

```shell
root@louis:~# docker inspect vc_data
[
	......
	"Mounts": [
            {
                "Type": "bind",
                "Source": "/root/htdocs",
                "Destination": "/usr/local/apache2/htdocs",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            },
            {
                "Type": "volume",
                "Name": "230bf10875bf87f212aab3f3477801bbcc4f0dee75e6b5740ae01287b1af4783",
                "Source": "/var/lib/docker/volumes/230bf10875bf87f212aab3f3477801bbcc4f0dee75e6b5740ae01287b1af4783/_data",
                "Destination": "/other/useful/tools",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ],
  .......
]
```

**其他容器可以使用--volumes-fron使用vc_data这个volume container**

```shell
root@louis:~# docker run --name web1 -d -p 80 --volumes-from vc_data httpd
174449b5cc4deb56ac32a81cdc357a9d5208a26cfbddd2792c6a65d108b15132
root@louis:~# docker run --name web2 -d -p 80 --volumes-from vc_data httpd
9ae6209cb5a482876d3027496c46b32614a8c5617537d1d42193b56315f8d710
root@louis:~# docker run --name web3 -d -p 80 --volumes-from vc_data httpd
aed7783d0dbbc2b3691f85e4e259e5de733179b31a752449bbda420134e5dd65

root@louis:~# docker ps
CONTAINER ID   IMAGE     COMMAND              CREATED         STATUS         PORTS                                     NAMES
aed7783d0dbb   httpd     "httpd-foreground"   8 minutes ago   Up 8 minutes   0.0.0.0:49159->80/tcp, :::49159->80/tcp   web3
9ae6209cb5a4   httpd     "httpd-foreground"   8 minutes ago   Up 8 minutes   0.0.0.0:49158->80/tcp, :::49158->80/tcp   web2
174449b5cc4d   httpd     "httpd-foreground"   9 minutes ago   Up 9 minutes   0.0.0.0:49157->80/tcp, :::49157->80/tcp   web1
root@louis:~# echo "This content is from a volume container" > ~/htdocs/index.html
root@louis:~# curl 127.0.0.1:49159
This content is from a volume container
root@louis:~# curl 127.0.0.1:49158
This content is from a volume container
root@louis:~# curl 127.0.0.1:49157
This content is from a volume container
```

volume container的特点

- 与bind mount相比，不必为每一个容器指定host path，所有path都在volume container中定义好了，容器只需与volume container关联，实现了容器与host的解藕
- 使用volume container的容器，其mount point是一致的，有利于配置的规范和标准化，但是也有一定的局限，使用时还是得综合考虑

#### 6.5 data-packed volume container

上面的volume container的数据终究还是在host里，需要找个办法，把数据完全放到volume container中，同时又能与其他容器共享

称这种容器叫data-packed volume container，原理是将数据打包到镜像中，然后通过docker managed volume共享

```shell
FROM busybox:latest
ADD htdocs /usr/local/apache2/htdocs
VOLUME /usr/local/apache2/htdocs
```

- ADD将静态文件添加到容器目录 /usr/local/apache2/htdocs
- VOLUME的作用与-v等效，用来创建docker managed volume，mount point为 /usr/local/apache2/htdocs。因为这个目录是ADD添加的目录，所以会把已有数据复制到volume中

```shell
docker create --name vc_data datapacked
```

- 因为在Dockerfile中已经使用了VOLUME指令，这里不要指定volume的mount point
- 启动httpd容器，并使用data-packed volume container
- ``docker run -d -p 80:80 --volumes-from vc_data httpd``

 **data-packed volume container 是自包含的，不依赖host提供数据，具有很强的移植性，非常适合只使用静态数据的场景，比如应用的配置信息、Web server的静态文件等**

#### 6.6 Data Volume生命周期管理

**Data Volume中存放的是重要的应用数据**

##### 6.6.1 备份

volume实际上是host文件系统中的目录和文件，所以volume的备份实际上是对文件系统的备份

```Shell
root@louis:~# docker run -d -p 5000:5000 -v /myregistry:/var/lib/registry registry:2
Unable to find image 'registry:2' locally
2: Pulling from library/registry
79e9f2f55bf5: Pull complete 
0d96da54f60b: Pull complete 
5b27040df4a2: Pull complete 
e2ead8259a04: Pull complete 
3790aef225b9: Pull complete 
Digest: sha256:169211e20e2f2d5d115674681eb79d21a217b296b43374b8e39f97fcf866b375
Status: Downloaded newer image for registry:2
1e3e02f717a8a884471961b8daf47041f2e14d40870b750fc036099007b291af
```

所有的本地镜像都保存在host的/myregistry目录中，定期备份这个目录

##### 6.6.2 恢复

如果数据损坏了，直接用之前备份的数据复制到/myregistry就可以了

##### 6.6.3 迁移

如果想更新版本的Registry

- docker stop当前Registry容器
- 启动新版本容器并mount原有volume

```shell
docker run -d -p 5000:5000 -v /myregistry:/var/lib/registry registry:latest
```

**启用新容器前，要确保新版本的默认数据路径是否发生变化**

##### 6.6.4 销毁

**volume删除之后数据是找不回来的**

docker不会销毁bind mount，删除数据的工作只能由host负责

- 对于docker managed volume，在执行docker rm删除容器时，带上-v参数
- docker会将容器使用到的volume一并删除
- 如果没有带-v，**会产生孤儿volume**
- 查看volume，``docker volume ls``

对于孤儿volmue可以使用docker volume rm删除

批量删除孤儿volume， ``docker volume rm $(docker volume ls -q)``

### 第七章：多主机管理

Docker Machine可以批量安装和配置docker host，这个host可以是本地的虚拟机、物理机，也可以是公有云中的云主机

Docker Machine为这些环境起了一个统一的名字：provider

- 对于某个特定的provider，Docker Machine使用相应的driver安装和配置docker host

#### 7.1 实验环境描述

三个运行的Ubuntu的host，在其中一台上安装Docker Machine，通过docker-machine命令在其他两个上部署docker

#### 7.2 安装Docker Machine

**Docker Machine**

> **Deprecated**
>
> Docker Machine has been deprecated. Please use Docker Desktop instead. See [Docker Desktop for Mac](https://docs.docker.com/desktop/mac/) and [Docker Desktop for Windows](https://docs.docker.com/desktop/windows/). You can also use other cloud provisioning tools.

The source code for Docker Machine has been archived. You can find the source code on [GitHub](https://github.com/docker/machine).

[docker](https://docs.docker.com/search/?q=docker), [machine](https://docs.docker.com/search/?q=machine)

官网上看了一下，好像被弃用了，这部分跳过。。。

### 第八章：容器网络

跨主机网络方案包括

- docker原生的overlay和macvlan
- 第三方方案
  - 常用的包括flannel、weave和calico

#### 8.1 libnetwork & CNM

libnetwork是docker容器网络库，最核心的内容是其定义的Container Network Model（CNM），这个模型对容器网络进行了抽象

- Sandbox
  - Sandbox是容器的网络栈，包含容器的interface、路由表和DNS设置
  - Linux Network Namespace是Sandbox的标准实现
  - Sandbox可以包含来自不同Network的Endpoint
- Endpoint
  - 作用是将Sandbox接入Network
  - 典型实现是veth pair
  - 一个Endpoint只能属于一个网络，也只能属于一个sandbox
- Network
  - Network包含一组Endpoint，同一Network的Endpoint可以直接通信
  - Network的实现可以是Linux Bridge

#### 8.2 overlay

为支持容器跨主机通信，Docker提供了overlay driver，使用户可以创建基于VxLAN的overlay网络。

VxLAN可将二层数据封装到UDP进行传输，VxLAN提供与VLAN相同的以太网二层服务，拥有更强的扩展性和灵活性

**Docker overlay网络需要一个key-value数据库用于保存网络状态信息，包括Network、Endpoint、IP等**

##### 8.2.1 实验环境描述

docker主机1（192.168.56.104），主机2（192.168.56.105），Consul在（192.168.56.101）

```shell
docker run -d -p 8500:8500 -h consul --name consul progrium/consul -server -bootstrap
```

通过访问``http://192.168.56.101:8500``

修改host的docker daemon配置文件 ``/etc/systemd/system/docker.service``

``--cluster-store=consul://192.168.56.101:8500 --cluster-advertise=enp0s8:2376``

- --cluster-store：指定consul的地址
- --cluster-advertise：告知consul自己的连接地址
- 重启docker daemon ``systemctl daemon-reload systemctl restart docker.service``

##### 8.2.2 创建overlay网络

在host1中创建overlay网络ov_net1 ``docker network create -d overlay ov_net1``

- -d overlay 指定 driver 为 overlay
- ov_net1的SCOPE为global
- 创建ovnet1时，host1将overlay网络信息存入了consul，host2也能从consul读取到新网络的数据
- docker network inspect查看 ov_net1的详细信息
- IPAM指 IP Address Management

**docker会创建一个bridge网络“docker_gwbridge"，为所有连接到overlay网络的容器提供访问外网的能力**

外网访问容器，可以通过主机端口映射 ``docker run -p 80:80 -d --net ov_net1 --name web1 httpd``

overlay网络的具体实现

- docker会为每个overlay网络创建一个独立的network namespace
- 其中会有一个linux bridge br0，endpoint由veth pair实现，一端连容器eh0，另一端连namespace的br0
- br0除了连接所有的endpoint，还会连接一个vxlan设备，用于与其他host建立vxlan tunnel
- 容器之间通过tunnel通信

##### 8.2.5 overlay网络隔离

不同的overlay网络是相互隔离的，跟vlan一样

##### 8.2.6 overlay IPAM

docker默认为overlay网络分配24位掩码的子网（10.0.X.0/24）

所有主机共享这个subnet，容器启动时会顺序从此空间分配IP，也可以--subnet指定IP空间

``docker network create -d overlay --subnet 10.22.1.0/24 ov_net3``

#### 8.3 macvlan

支持跨主机容器网络的driver：macvlan

macvlan本身是linux kernel模块，其功能是允许同一个物理网卡配置多个MAC地址，即多个interface，每个interface可以配置自己的IP

**macvlan本质上是一种网卡虚拟化技术**

macvlan最大优点是性能极好，相比其他实现，macvlan不需要创建Linux bridge，而是直接通过以太interface连接到物理网络

打开网卡模式``ip link set enp0s9 promisc on``

##### 8.3.2 创建macvlan网络

```shell
docker network create -d macvlan \
	--subnet=172.16.86.0/24 \
	--gateway=172.16.86.1 \
	-o parent=enp0s9 mac-net1
```

- -d macvlan指定dirver为macvlan
- macvlan网络是local网络，为了保证跨主机能够通信，用户一般需要自己管理IP subnet
- 与其他网络不同，docker不会位macvlan创建网关，这里的网关应该是真实存在的，否则容器无法路由
- -o parent指定使用的网络interface

在host1中运行容器bbox1，并连接到mac_net1

``docker run -itd --name bbox1 --ip=172.16.86.10 --network mac_net1 busybox``

**docker没有为macvlan提供DNS服务，这点是与overlay网络不同的**

##### 8.3.3 macvlan网络结构分析

macvlan不依赖Linux bridge，brctl show可以确认没有创建新的bridge

查一下容器bbox1的网络设备，``docker exec bbox1 ip link``

**容器的eth0就是enp0s9通过macvlan虚拟出来的interface**

容器的interface直接与主机的网卡连接，这种方案使得容器无需通过NAT和端口映射就能与外网直接通信（只要有网关），在网络上与其他独立主机没有区别

##### 8.3.4 用sub- interface实现多macvlan网络

VLAN是现代网络常用的网络虚拟化技术，它可以将物理的二层网络划分成最多4094个逻辑网络，这些逻辑网络在二层上是隔离的，每个逻辑网络（即VLAN）由VLAN ID区分，VLAN ID的取值为1～4094

Linux的网卡也能支持VLAN（apt-get install vlan），同一个interface可以收发多个VLAN的数据包，不过前提是要创建VLAN的sub- interface

**macvlan网络的连通和隔离完全依赖VLAN、IP subnet和路由，docker本身不做任何限制，用户可以像管理传统VLAN网络那样管理macvlan**

#### 8.4 flannel

flannel是CoreOS开发的容器网络解决方案

flannel为每个host分配一个subnet，容器从subnet中分配IP，这些IP可以在host间路由，容器间无需NAT和port mapping就可以跨主机通信

每个subnet都是从一个更大的IP池中划分的，flannel会在每个主机上运行一个叫flanneld的agent，其职责就是从池子中分配subnet。

数据包在主机间转发是由backend实现的，最常用的有vxlan和host-gw

##### 8.4.1 实验环境描述

etcd部署在192.168.56.101，host1和host2上运行flanneld

#### 8.5 weave

weave是Weaveworks开发的容器网络解决方案。

weave创建的虚拟网络可以将部署在多个主机上的容器连接起来。对容器来说，weave就像一个巨大的以太网交换机，所有容器都被接入会这个交换机，容器可以直接通信，无需NAT和端口映射

weave的DNS模块使容器可以通过hostname访问。

##### 8.5.1 实验环境描述

weave不依赖分布式数据库交换网络信息，每个主机上只需运行weave组件就能建立起跨主机容器网络

#### 8.6 calico

Calico是一个纯三层的虚拟网络方案，Calico为每个容器分配一个IP，每个host都router，把不同的host的容器连接起来

Calico不对数据包做额外封装，不需要NAT和端口映射，扩展性和性能都很好

Calico还有一大优势：network policy

- 用户可以动态定义ACL规则，控制进出容器的数据包，实现业务需求

##### 8.6.1 实验环境描述

Calico依赖etcd在不同主机间共享和交换信息，存储Calico网络状态

Calico网络中的每个主机都需要运行Calico组件，实现容器interface管理、动态路由、动态ACL、报告状态等

#### 8.7 比较各种网络模型

##### 8.7.1 网络模型

Docker overlay

- overlay网络，建立主机间VxLAN隧道，原始数据包在发送端被封装成VxLAN数据包，到达目的后在接收端解包

Macvlan网络

- 在二层上通过VLAN连接容器，在三层上依赖外部网关连接不同macvlan，
- 数据包直接发送，不需要封装，属于underlay网络

Flannel两种backend

- vxlan与Docker overlay类似，属于overlay网络
- host-gw将主机作为网关，依赖三层IP转发， 不需要像vxlan那样对包进行封装，属于underlay网络

Weave

- 是VxLAN实现，属于overlay网络

##### 8.7.3 IPAM

Docker Overlay网络

- 所有主机共享同一个subnet，容器启动时会顺序分配IP，可以通过--subnet定制此IP空间

Macvlan

- 需要用户自己管理subnet，为容器分配IP，不同subnet通信依赖外部网关

Flannel

- 为每个主机自动分配独立的subnet，用户只需要指定一个大的IP池
- 不同subnet之间的路由信息也由Flannel自动生成和配置

Weave

- 默认配置下所有容器使用10.32.0.0/12 subnet，如果此地址空间与现有IP冲突，可以通过--ipalloc-range分配特定的subnet

Calico

- 从IP Pool（可定制）中为每个主机分配自己的subnet

##### 8.7.4 连通与隔离

Docker Overlay网络中的容器可以通信，但不同网络之间无法通信，要实现跨网络访问，只有将容器加入多个网络。与外网通信可以通过docker_gwbridge网络。

Macvlan网络的连通或隔离完全取决于二层VLAN和三层路由。

不同Flannel网络中的容器直接就可以通信，没有提供隔离。与外网通信可以通过bridge网络

Weave网络默认配置下所有容器在一个大的subnet中，可以自由通信，如果要实现隔离，需要为容器指定不同的subnet和IP。与外网通信的方案是将主机加入到weave网络，并把主机当作网关。

Calico默认配置下只允许位于同一网络中的容器之间通信，但通过其强大的Policy能够实现几乎任意场景的访问控制

##### 8.7.5 性能

最朴素的判断：**Underlay网络性能优于Overlay网络**

Overlay网络利用隧道技术，将数据包封装到UDP中进行传输。因为涉及数据包的封装和解封，存在额外的CPU和网络开销。虽然几乎所有Overlay网络方案底层都采用Linux kernel的vxlan模块，这样可以尽量减少开销，但这个开销与Underlay网络相比还是存在的。

**所以Macvlan、Flannel host-gw、Calico的性能会优于Docker overlay、Flannel vxlan和Weave**

Overlay较Underlay可以支持更多的二层网段，能更好地利用已由网络，以及有避免物理交换机MAC表耗尽等优势，所以在方案选型的时候需要综合考虑

### 第九章：容器监控

#### 9.1 Docker自带的监控子命令

新版的Docker提供了一个新命令``docker container ls``， 其作用和用法与``docker container ps``一样，命令含义表达的更准确，推荐使用

##### 9.1.2 top

查看某个容器中运行了哪些进程，执行``docker container top [container]``命令

也可以查特定的信息 ``docker container top [container] -au``

##### 9.1.3 stats

用于显示每个容器各种资源的使用情况，默认会显示一个实时变化的列表

``docker container stats``

#### 9.2 sysdig

轻量级的系统监控工具

```shell
docker container run -it --rm --name=sysdig --privileged=true --volume=/var/run/docker.sock:/host/var/run/docker.sock --volume=/dev:/host/dev --volume=/proc:/host/proc:ro --volume=/boot:/host/boot:ro --volume=/lib/modules:/host/lib/modules:ro --volume=/usr:/host/usr:ro sysdig/sysdig
```

通过``docker container exec -it sysdig bash``进入容器，执行csysdig命令，以交互方式启动sysdig，类似Linux top命令的界面

可以点击底部views菜单，或者按F2键，显示View选择列表

特点

- 监控信息全，包括Linux操作系统和容器
- 界面交互性强

#### 9.3 Weave Scope

自动生成一张Docker Host地图

##### 9.3.1 安装

``curl -L git.io/scope -o /usr/local/bin/scope chmod a+x /usr/local/bin/scope scope launch``

Weave Scope的访问地址：``http://[Host IP]:4040/``

##### 9.3.4 多主机监控

在多个host上执行 ``scope launch ip1 ip2...``

#### 9.4 cAdvisor

google开发的容器监控工具，略

#### 9.5 Prometheus

非常优秀的监控方案，提供了监控数据搜集、存储、处理、可视化和告警一套完整的解决方案

##### 9.5.1 架构

1. Prometheus Server
   - 负责从Exporter拉取和存储监控数据，并提供一套灵活的查询语言PromQL
2. Exporter
   - 负责收集目标对象（host、container等）的性能数据，并通过HTTP接口供Prometheus Server获取
3. 可视化组件
   - Grafana与Prometheus集成，提供完美的数据展示能力
4. Alertmanager
   - 用户可以定义基于监控数据的告警规则，规则会触发告警
   - 支持的方式包括Email、PagerDuty、Webhook等

Prometheus最大的亮点和先进性来自它的多维数据模型

##### 9.5.2 多维数据模型

Prometheus定义一个全局的指标 container_memory_usage_bytes，通过添加不同的维度数据来满足不同的业务需求

##### 9.5.3 实践

需要运行的组件

- Prometheus Server，Prometheus Server本身也将以容器的方式运行在host 192.168.56.103上

- Exporter，Prometheus有很多现场的Exporter，https://prometheus.io/docs/instrumenting/exporters/

  - Node Exporter：负责收集host硬件和操作系统数据，它将以容器方式运行在所有host

  - cAdvisor：负责收集容器数据，它将以容器方式运行在所有host上

- Grafana，显示多维数据，Grafana本身也将以容器方式运行在host 192.168.56.103上

**运行Node Exporter**

在两个host上执行命令

```shell
docker run -d -p 9100:9100 \ -v "/proc:/host/proc" \ -v "/sys:/host/sys" \ -v "/:/rootfs" \ --net=host \ prom/node-exporter \ -collector.procfs /host/proc \ -collector.sysfs /host/sys \ -collector.filesystem.ignored-mount-points "^/(sys|proc|dev|host|etc) ($|/)"
```

- 使用了--net=host，这样Prometheus Server可以直接与Node Exporter通信
- 在浏览器中通过：``http://192.168.56.102:9100/metrics``测试

**运行cAdvisor**

在两个host上执行命令

```shell
docker run \ --volume=/:/rootfs:ro \ --volume=/var/run:/var/run:rw \ --volume=/sys:/sys:ro \ --volume=/var/lib/docker/:/var/lib/docker:ro \ --publish=8080:8080 \ --detach=true \ --name=cadvisor \ --net=host \ google/cadvisor:latest
```

- 使用了--net=host，这样Prometheus Server可以直接与cAdvisor通信
- 在浏览器中通过：``http://192.168.56,102:8080/metrics``测试

**运行Prometheus Server**

在host 192.168.56.103上执行

```shell
docker run -d -p 9090:9090 \ -v /root/prometheus.yml:/etc/prometheus/prometheus.yml \ --name prometheus \ --net=host \ prom/prometheus
```

- 使用了--net=host，这样Prometheus Server可以直接与Exporter和Grafana通信
- prometheus.yml是Prometheus Server的配置文件

重要的配置是static_configs中的，指定从哪些exporter抓取数据。这里指定了两台host上的Node Exporter和cAdvisor。

localhost:9090是Prometheus Server自己，Prometheus本身也会收集自己的监控数据

- 浏览器打开``http://192.168.56.103:9090``

**运行Grafana**

在host 192.168.56.103上执行

```shell
docker run -d -i -p 3000:3000 \ -e "GF_SERVER_ROOT_URL=http://grafana.server.name" \ -e "GF_SECURITY_ADMIN_PASSWORD=secret" \ --net=host \ grafana/grafana
```

#### 9.6 比较不同的监控工具

Sysdig、Weave Scope和cAdvisor可以监控到Host操作系统的状态，而Prometheus则可以通过Exporter支持应用级的监控，比如监控ceph、haproxy等

#### 9.7 几点建议

- Docker ps/top/stats最适合快速了解容器运行状态，从而判断是否需要进一步分析和排查
- Sysdig提供了丰富的分析和挖掘功能，是Troubleshotting的神器
- cAdvisor一般不会单独使用，通常作为其他监控工具的数据收集器，比如Prometheus
- Weave Scope流畅简洁的操控界面是最大亮点，而且支持直接在Web界面上执行命令
- Prometheus的数据模型和架构决定了它几乎具有无限的可能性。Prometheus和Weave Scope都是优秀的容器监控方案。Prometheus可以监控其他应用和系统，更加综合全面
- 监控系统的选择，并不是一道单选题，应该根据需求和实际情况搭配组合，优势互补

### 第十章：日志管理

#### 10.1 Docker logs

对于一个运行的容器，Docker会将日志发送到容器的标准输出设备（STDOUT）和标准错误设备（STDERR），实际上是容器的控制台终端

如果容器是后台运行的话，查看容器的日志有两种方法

- attach到该容器
  - 只能看到attach之后的日志，之前日志不可见
  - 退出attach状态比较麻烦
- docker logs命令查看日志
  - docker logs -f 容器ID

#### 10.2 Docker logging driver

Docker提供了多种日志机制帮助用户从运行的容器中提取日志信息，这些机制被称为logging driver

Docker的默认logging driver是json-file

```shell
root@louis:~# docker info | grep 'Logging Driver'
 Logging Driver: json-file
WARNING: API is accessible on http://0.0.0.0:2375 without encryption.
         Access to the remote API is equivalent to root access on the host. Refer
         to the 'Docker daemon attack surface' section in the documentation for
         more information: https://docs.docker.com/go/attack-surface/
```

#### 10.3 ELK

三个软件的合称

**Elasticsearch**

- 近乎实时查询的全文搜索引擎。Elasticsearch的设计目标就是要能够处理和搜索巨量的日志数据

**Logstash**

- 读取原始日志，并对其进行分析和过滤，然后将其转发给其他组件（比如Elasticsearch）进行索引或存储。
- Logstash支持丰富的Input和Output类型，能够处理各种应用的日志

**Kibana**

- 基于JavaScript的Web图形界面程序，专门用于可视化Elasticsearch的数据。
- Kibana能够查询Elasticsearch并通过丰富的图表展示结果
- 用户可以创建Dashboard来监控系统的日志

##### 10.3.1 日志处理流程

Logstash负责从各个Docker容器中提取日志，Logstash将日志转发到Elasticsearch进行索引和保存，Kibana分析和可视化数据

##### 10.3.2 安装ELK套件

```shell
docker run -p 5601:5601 -p 9200:9200 -p 5044:5044 -it --name elk sebp/elk
```

- 使用sebp/elk这个现成的image，里面包含了整个ELK stack
- 5601:Kibana Web接口
- 9200:Elasticsearch JSON接口
- 5044:Logstash日志接收接口

##### 10.3.3 Filebeat

Docker会将容器日志记录到 /var/lib/docker/containers/<container ID>/<container ID>-json.log ，只要将这文件发送给ELK就可以实现日志管理

Filebeat能将指定路径下的日志文件转发给ELK，同时它还会监控日志文件，当日志更新时，Filebeat会将新的内容发送给ELK

#### 10.4 Fluentd

ELK中使用Filebeat收集Docker容器的日志，利用的是Docker默认的logging driver json-file

Flunted是一个开源的数据收集器，可以连接各种数据源和数据输出组件

#### 10.5 Graylog

Graylog是与ELK可以相提并论的一款集中式日志管理方案，支持数据收集、检索、可视化Dashboard

##### 10.5.1 Graylog架构

- Graylog负责接收来自各种设备和应用的日志，并为最终用户提供Web访问捷克
- Elasticsearch用于索引和保存Graylog接收到的日志
- MongoDB负责保存Graylog自身的配置信息

### 第十一章：数据管理

#### 11.1 从一个例子开始

假设有两个Docker主机，Host1运行了一个MySQL容器，为了保护数据，data volume由storage provider提供

- 用volume driver实现这个跨主机管理data volume方案
- 任何一个data volume都是由driver管理的，创建volume时如果不特别指定，将使用local类型的driver，即从Docker Host的本地目录中分配存储空间。
- **如果要支持跨主机的volume，则需要使用第三方driver**

**Rex-Ray driver**

- 开源，社区活跃
- 支持多种backend，如VirtualBox的Virtual Media、Amazon EBS、Ceph RBD等
- 支持多种操作系统，如Ubuntu、CentOS、RHEL和CoreOS
- 支持多种容器编排引擎，如Docker Swarm、Kubernetes和Mesos
- 安装使用比较简单
