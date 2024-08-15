
- [1. 为什么使用Docker —— 对比传统虚拟机](#1-为什么使用docker--对比传统虚拟机)
- [2. Image 镜像](#2-image-镜像)
  - [2.1. 分层存储](#21-分层存储)
  - [2.2. Dockerfile](#22-dockerfile)
    - [2.2.1. 镜像构建上下文（Context）](#221-镜像构建上下文context)
    - [2.2.2. **FROM** 指定基础镜像](#222-from-指定基础镜像)
    - [2.2.3. **RUN** 运行命令](#223-run-运行命令)
    - [2.2.4. **COPY / ADD** 复制文件](#224-copy--add-复制文件)
    - [2.2.5. **CMD** 容器主进程的启动命令](#225-cmd-容器主进程的启动命令)
    - [2.2.6. **ENTRYPOINT** CMD接收器](#226-entrypoint-cmd接收器)
    - [2.2.7. **ENV** 环境变量](#227-env-环境变量)
    - [2.2.8. **ARG** 构建变量](#228-arg-构建变量)
    - [2.2.9. **VOLUME** 定义匿名卷](#229-volume-定义匿名卷)
    - [2.2.10. **EXPOSE** 声明端口](#2210-expose-声明端口)
    - [2.2.11. **WORKDIR** 切换构建工作目录](#2211-workdir-切换构建工作目录)
    - [2.2.12. **USER** 切换用户](#2212-user-切换用户)
    - [2.2.13. **HEALTHCHECK** 健康检查](#2213-healthcheck-健康检查)
    - [2.2.14. **ONBUILD** 二次构建的命令](#2214-onbuild-二次构建的命令)
    - [2.2.15. **LABEL** 添加镜像元数据](#2215-label-添加镜像元数据)
    - [2.2.16. **SHELL** 切换指令SHELL](#2216-shell-切换指令shell)
- [3. Container 容器](#3-container-容器)
  - [3.1. 新建并启动](#31-新建并启动)
  - [3.2. 启动或停止已有容器](#32-启动或停止已有容器)
  - [3.3. 查看守护态容器内主进程的输出](#33-查看守护态容器内主进程的输出)
  - [3.4. 进入容器](#34-进入容器)
  - [3.5. 导入/导出容器](#35-导入导出容器)
  - [3.6. 删除容器](#36-删除容器)
- [4. 数据管理](#4-数据管理)
  - [4.1. 数据卷（Volumes）](#41-数据卷volumes)
  - [4.2. 挂载 Host 目录 (Bind mounts)](#42-挂载-host-目录-bind-mounts)
- [5. Docker 网络](#5-docker-网络)
  - [5.1. 外部访问](#51-外部访问)
  - [5.2. 容器互联](#52-容器互联)
  - [5.3. 容器hostname, DNS](#53-容器hostname-dns)
    - [5.3.1. 修改方式](#531-修改方式)


# 1. 为什么使用Docker —— 对比传统虚拟机
以下是Docker容器与传统虚拟机的主要特性对比：
| 特性       | 容器               | 虚拟机     |
| ---------- | ------------------ | ---------- |
| 启动       | 秒级               | 分钟级     |
| 硬盘使用   | 一般为 MB          | 一般为 GB  |
| 性能       | 接近原生           | 弱于       |
| 系统支持量 | 单机支持上千个容器 | 一般几十个 |

使用Docker容器的优势在于快速启动、硬盘占用小、性能接近原生，以及能够在单机上支持更多的系统实例。

# 2. Image 镜像
$\underline{镜像}$，基于宿主的内核，包含一套特殊的root文件系统，以及一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。
镜像不包含任何动态数据，其内容在构建之后也不会被改变。


## 2.1. 分层存储
镜像区别于ISO（等同于打包文件），是基于Union FS的多层文件系统联合构成。每层初始基于上一层，当前层的修改不会影响上一层。



---
## 2.2. Dockerfile

### 2.2.1. 镜像构建上下文（Context）   
```
docker build [选项] <上下文路径/URL/->
eg: 
docker build -t nginx:v3 .
```
将上下文路径下的文件打包发送到服务端，进行docker build。当不使用 `-f dockerfile` 指定 dockerfile时，默认选择Context下名为`Dockerfile`的文件。

后续 COPY, ADD 类的src都是**基于Context的相对路径**。

 <br>

### 2.2.2. **FROM** 指定基础镜像
```docker
FROM <Image>

eg:
FROM nginx:latest
FROM scratch // 指定空镜像
```

 <br>

### 2.2.3. **RUN** 运行命令
```docker
# 格式
RUN <命令>
RUN ["可执行文件", "参数1", "参数2"]

eg:
RUN set -x; buildDeps='gcc libc6-dev make wget' \
    && apt-get update \
    && apt-get install -y $buildDeps \
    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz" \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \
    && rm -rf /var/lib/apt/lists/* \
    && rm redis.tar.gz \
    && rm -r /usr/src/redis \
    && apt-get purge -y --auto-remove $buildDeps
```
基于 UNION FS，每一个 RUN 会生成一层Docker Layer，上限127层。

为避免**镜像臃肿**，dockerfile 乃至每一个 RUN 结尾都应该清楚构建残余。

 <br>

### 2.2.4. **COPY / ADD** 复制文件
```docker
# 格式
COPY [--chown=<user>:<group>] <源路径>... <目标路径>
COPY [--chown=<user>:<group>] ["<源路径1>",... "<目标路径>"]

eg:
COPY ./package.json /usr/src/app/
COPY a* d?.txt  /mydir/
```
- **目标路径不存在**会创建。
- 源路径是基于Context的相对路径，目标路径是Docker server的绝对路径。
- 源文件的各种元数据都会保留。比如读、写、执行权限、文件变更时间等。

```docker
# 格式
ADD [--chown=<user>:<group>] <源路径>... <目标路径>
ADD [--chown=<user>:<group>] ["<源路径1>",... "<目标路径>"]

eg:
ADD package.tar /usr/src/app/ # gzip, bzip2, xz
```
- 源路径支持URL、压缩包等。
- 当源路径为tar包时，会自动解压。

**应尽量使用COPY，除非需要解压文件。**

 <br>

### 2.2.5. **CMD** 容器主进程的启动命令
```docker
# 格式
CMD <命令>
CMD ["可执行文件", "参数1", "参数2"...]

eg:
CMD echo $HOME 
# 等价于
CMD [ "sh", "-c", "echo $HOME" ]

# 运行时覆盖默认 CMD
docker run -it ubuntu cat /etc/os-release
```
- 应当指定 Frontend 形式的指令。
- 可以使用 shell 环境变量。

 <br>

### 2.2.6. **ENTRYPOINT** CMD接收器
```docker
# 格式
ENTRYPOINT <命令>
ENTRYPOINT ["可执行文件", "参数1", "参数2"...]

eg:
ENTRYPOINT ["echo"]
CMD ["-n", "Default print out"]

# 运行时动态指定
docker run -it ubuntu --entrypoint echo ""
```
- 当指定了 ENTRYPOINT 后，CMD 的含义就变成将 CMD 的内容作为参数传给 ENTRYPOINT 指令。<br>
  `<ENTRYPOINT> "<CMD>"`

 <br>

### 2.2.7. **ENV** 环境变量
```docker
# 格式
ENV <key> <value>
ENV <key1>=<value1> <key2>=<value2>...

eg:
ENV NODE_VERSION 7.2.0

RUN curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION-linux-x64.tar.xz" \
  && curl -SLO "https://nodejs.org/dist/v$NODE_VERSION/SHASUMS256.txt.asc" \
  && gpg --batch --decrypt --output SHASUMS256.txt SHASUMS256.txt.asc \
  && grep " node-v$NODE_VERSION-linux-x64.tar.xz\$" SHASUMS256.txt | sha256sum -c - \
  && tar -xJf "node-v$NODE_VERSION-linux-x64.tar.xz" -C /usr/local --strip-components=1 \
  && rm "node-v$NODE_VERSION-linux-x64.tar.xz" SHASUMS256.txt.asc SHASUMS256.txt \
  && ln -s /usr/local/bin/node /usr/local/bin/nodejs
```
- 以下变量支持变量展开：`ADD`, `COPY`, `ENV`, `EXPOSE`, `FROM`, `LABEL`, `USER`, `WORKDIR`, `VOLUME`, `STOPSIGNAL`, `ONBUILD`, `RUN`

 <br>

### 2.2.8. **ARG** 构建变量
```docker
# 格式
ARG <key1>=<value1> <key2>=<value2>...

eg:
# build 时动态指定
docker build --build-arg <参数名>=<值>

eg:
# FROM 前的 ARG 只作用于 FROM
ARG DOCKER_USERNAME=library
FROM ${DOCKER_USERNAME}/alpine

RUN set -x ; echo ${DOCKER_USERNAME}    # NUL

# 要想在 FROM 之后使用，必须再次指定
ARG DOCKER_USERNAME=library
RUN set -x ; echo ${DOCKER_USERNAME}
```
- ARG 所设置的构建环境的环境变量**不会**存在于容器运行时。

 <br>

### 2.2.9. **VOLUME** 定义匿名卷
```docker
# 格式
VOLUME ["<路径1>", "<路径2>"...]
VOLUME <路径>

eg:
VOLUME /data
```
- 卷内数据变化不会保存在容器运行的存储层，保证了容器存储层的无状态化。
- `docker run -d -v myLocalData:/data xxxx` 挂载卷会覆盖匿名卷，此时数据可以持久化到 host 环境（local）。

 <br>

### 2.2.10. **EXPOSE** 声明端口
```docker
# 格式
EXPOSE <端口1> [<端口2>...]

eg:
    EXPOSE 3333 3334
```
- EXPOSE 是一种文档化的做法，用于声明容器意图打开哪些端口，但不会导致端口映射。
- docker run -p 手动指定容器端口到宿主机端口的映射。
- docker run -P 自动将所有 EXPOSE 的端口映射到宿主机的随机端口。

 <br>

### 2.2.11. **WORKDIR** 切换构建工作目录
```docker
# 格式
WORKDIR <工作目录绝对/相对路径>

eg:
    # 如操作 /app/config
    RUN cd /app
    RUN echo "Somthing" > ./config      # 错误，每层独立，cd 将无效

    WORKDIR /app
    RUN echo "Somthing" > ./config      # 正确
```
- 如不存在，则创建。
- 此后的各层当前目录就改为指定目录。

 <br>

### 2.2.12. **USER** 切换用户
```docker
# 格式
USER <用户名>[:<用户组>]

eg:
RUN groupadd -r redis && useradd -r -g redis redis
USER redis
RUN [ "redis-server" ]
```
- 只能切换存在的用户。

 <br>

### 2.2.13. **HEALTHCHECK** 健康检查
```docker
# 格式
HEALTHCHECK NONE：屏蔽默认健康检查指令
HEALTHCHECK [选项] CMD <命令>：设置检查容器健康状况的命令

# 参数
--interval=<间隔>：两次健康检查的间隔，默认为 30 秒；
--timeout=<时长>：健康检查命令运行超时时间，如果超过这个时间，本次健康检查就被视为失败，默认 30 秒；
--retries=<次数>：当连续失败指定次数后，则将容器状态视为 unhealthy，默认 3 次

eg:
HEALTHCHECK --interval=5s --timeout=3s \
  CMD curl -fs http://localhost/ || exit 1
```
- 解决主进程假死，但容器状态显示正常的场景。
- 命令的返回值决定了该次健康检查的成功与否：<br>0：成功<br>1：失败<br>2：保留，不要使用这个值。

 <br>

### 2.2.14. **ONBUILD** 二次构建的命令
```docker
# 格式
ONBUILD <其它指令>

eg:
# dockerfile -- node1
    FROM node1

    RUN mkdir /app
    WORKDIR /app

    # 构建 node1 时不会执行
    ONBUILD COPY ./package.json /app
    ONBUILD RUN [ "npm", "install" ]
    ONBUILD COPY . /app/

    CMD [ "npm", "start" ]

# dockerfile -- node2
    FROM node1
    # 执行 node1 的 ONBUILD 命令
```
- 用于制作基础镜像时，指定后续镜像的标准步骤。

 <br>

### 2.2.15. **LABEL** 添加镜像元数据
```docker
# 格式
LABEL <key>=<value> <key>=<value> <key>=<value> ...

eg:
LABEL org.opencontainers.image.authors="yeasy"
LABEL org.opencontainers.image.documentation="https://yeasy.gitbooks.io"
```
- 申明镜像的作者、文档地址等。

 <br>

### 2.2.16. **SHELL** 切换指令SHELL
```docker
# 格式
SHELL ["executable", "parameters"]

eg:
SHELL ["/bin/sh", "-c"]
RUN lll ; ls

# 第二个 RUN 运行的命令会打印出每条命令并当遇到错误时退出
SHELL ["/bin/sh", "-cex"]
RUN lll ; ls

ENTRYPOINT nginx    # /bin/sh -cex "nginx"
CMD nginx           # /bin/sh -cex "nginx"
```
- 指定 RUN ENTRYPOINT CMD 指令的 shell。
- 不指定时，默认等价于 `SHELL ["/bin/sh", "-c"]`。


# 3. Container 容器
$\underline{容器}$，是独立运行的**一个或一组应用**，以及它们的**运行态环境**。

## 3.1. 新建并启动  
```docker
docker run <Image> <CMD>
eg: 
    docker run ubuntu:18.04 /bin/echo 'Hello world'
    docker run -it ubuntu:18.04 /bin/bash

# -d: 守护态运行
# -i: 保持打开 stdin
# -t: 接入伪终端（pseudo-tty）并绑定到容器的标准输入
```
当利用 docker run 来创建容器时，Docker 在后台运行的标准操作包括：

- 检查本地是否存在指定的镜像，不存在就从 registry 下载
- 利用镜像创建并启动一个容器
- 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
- 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
- 从地址池配置一个 ip 地址给容器
- 执行用户指定的应用程序
- 执行完毕后容器被终止

 <br>

## 3.2. 启动或停止已有容器
```docker
docker container start [OPTIONS] <CONTAINER> [CONTAINER...]
docker container stop [OPTIONS] <CONTAINER> [CONTAINER...]
```

 <br>

## 3.3. 查看守护态容器内主进程的输出
```docker
docker container logs [OPTIONS] <CONTAINER>
```

 <br>

## 3.4. 进入容器
```docker
docker attach <CONTAINER>
docker exec [OPTIONS] <CONTAINER> <CMD>

eg:
    docker attach 69d1          # exit 会终止容器
    docker exec -it 69d1        # exit 不会终止容器
```

 <br>

## 3.5. 导入/导出容器
```docker
# 导入/导出容器快照
docker export <CONTAINER> > .tar
cat .tar | docker import - <NEW IMAGE>

# 导入镜像存储
docker load

eg:
    docker export 7691a814370e > ubuntu.tar
    cat ubuntu.tar | docker import - test/ubuntu:v1.0

    docker import http://example.com/exampleimage.tgz example/imagerepo
```
- **容器快照文件**将丢弃所有的历史记录和元数据信息（即仅保存容器当时的快照状态），导入时可以重新指定标签等元数据信息。
- **镜像存储文件**将保存完整记录，体积也要更大。

 <br>

## 3.6. 删除容器
```docker
docker container rm <CONTAINER>

# 删除所有终止的容器
docker container prune
```

 <br>

# 4. 数据管理
```docker
# 查看容器挂载信息 ("Mounts" Key)
docker inspect <CONTAINER>
```

## 4.1. 数据卷（Volumes）
```docker
# 创建 Volume
docker volume create <Volume Name>
# 删除 Volume
docker volume rm <Volume Name>
# 删除所有无主 Volumes
docker volume prune


# 挂载 Volume 启动容器
docker run -d -P \
    --name web \
    # -v my-vol:/usr/share/nginx/html \
    --mount source=my-vol,target=/usr/share/nginx/html \
    nginx:alpine

# -v        <dst> [<Volume>:<dst>]
# --mount   source=<Volume>,target=<dst>[, readonly]

```
$\underline{数据卷}$是一个可供 **一个或多个** 容器使用的特殊目录：
- 可以在容器之间共享和重用
- 对 **数据卷** 的修改会立马生效，且不会影响镜像
- 默认会一直存在，即使容器被删除
- -v 不指定宿主机 source 时，会创建临时目录 `/var/lib/docker/volumes/[VOLUME_ID]/_data` 进行挂载
 
类似于 Linux 下对目录或文件进行 mount，**数据卷为空** 时，镜像中指定挂载数据卷会 **复制** 目录下的文件到数据卷中。

 <br>

## 4.2. 挂载 Host 目录 (Bind mounts)
可以使用 --mount，直接挂载一个 host 目录作为数据卷。
```docker
# -v        <Source>:<dst>
# --mount   source=<Source>,target=<dst>[, readonly]
```
 - Host 目录必须为 **绝对路径**。
 - -v 指定 Docker dst 如不存在会创建；--mount 会 **报错**。

# 5. Docker 网络
```docker
# 查看访问记录
docker logs <CONTAINER>
```


## 5.1. 外部访问
```docker
# 随机映射端口到容器（-P）
docker run -P <CONTAINER>

# 指定端口（-p）
docker run -p [Host IP:][Host Port]:<Container Port>
eg:
    # 映射到指定地址的指定端口
    docker run -d -p localhost:80:80 nginx:alpine
    # 映射所有 ip 地址
    docker run -d -p 80:80 nginx:alpine
    # 映射到指定地址的随机端口
    docker run -d -p localhost::80 nginx:alpine
    # udp 映射
    docker run -d -p localhost:80:80/udp nginx:alpine

# 查看映射情况
docker port <CONTAINER> [PORT]
```

 <br>

## 5.2. 容器互联
```docker
# 创建容器网络
docker network create -d <bridge|overlay> <Network Name>

eg:
    docker network create -d bridge my_net
    docker run -it --name container1 --network my_net ubuntu bash
    docker run -it --name container2 --network my_net ubuntu bash
    # Inside container1
    container1$: ping container2    # connected
    # Inside container2
    container2$: ping container1    # connected
```

<br>

## 5.3. 容器hostname, DNS
容器通过挂载宿主机的 resolv.conf，来与宿主机的 DNS 同步。

### 5.3.1. 修改方式
```docker
# 修改配置文件，全局改变所有容器的DNS
vim /etc/docker/daemon.json
{
  "dns" : [
    "114.114.114.114",
    "8.8.8.8"
  ]
}

# 启动容器时设置各容器的hostname、DNS
docker run ... \
    -h HOSTNAME[|--hostname=HOSTNAME] \ # 设置 hostname
    --dns=IP_ADDRESS                    # 设置 DNS
```
如果在容器启动时没有指定最后两个参数，Docker 会默认用主机上的 `/etc/resolv.conf` 来配置容器。

# 容器实现原理 —— 本质是一种特殊的进程
 1. 启用 Linux Namespace 配置；
 2. 设置指定的 Cgroups 参数；
 3. 切换进程的根目录（Change Root）。

## 隔离 —— namespace
容器的创建，等价于 Linux 系统调用
```C
# CLONE_NEWPID
int pid = clone(main_function, stack_size, CLONE_NEWPID | SIGCHLD, NULL); 
```
创建容器进程时，指定了 Mount、UTS、IPC、Network 和 User 等进程 namespace 空间，每个空间内的进程 PID 从 1 开始计数（就是容器内部看到的进程列表）。

**容器只是一种特殊的进程**，本质上并不是真正创建了一个“容器”。

<br>

## 特殊的namespace —— Mount Namespace（rootfs）
对容器进程视图的改变，一定是伴随着挂载操作（mount）才能生效。

因此容器进程启动进行 Mount Namespace 之后，需要重新挂载整个根目录“/”，文件系统才会隔离出来。

就是容器镜像包含的 **rootfs**。

<br>

## 限制 —— Cgroups（Linux Control Group）

### 作用
 - 限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。
 - 对进程进行优先级设置、审计，以及将进程挂起和恢复等操作。

### 使用实例
```sh
# 查看 /sys/fs/cgroup 路径下的 cgroup 文件
mount -t cgroup 
# cpuset on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
# cpu on /sys/fs/cgroup/cpu type cgroup (rw,nosuid,nodev,noexec,relatime,cpu)
# cpuacct on /sys/fs/cgroup/cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpuacct)
# blkio on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
# memory on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)

# === 给某个进程限制资源使用（如限制 pid 226的 CPU）===
cd /sys/fs/cgroup/cpu

mkdir container			# 创建子系统【container】，并会自动创建其中对应的资源文件
ls container/
# cgroup.clone_children cpu.cfs_period_us cpu.rt_period_us  cpu.shares notify_on_release
# cgroup.procs      cpu.cfs_quota_us  cpu.rt_runtime_us cpu.stat  tasks

# 限制 CPU 时间片为 20000 us/100000 us (20%)
echo 20000 > /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us
cat /sys/fs/cgroup/cpu/container/cpu.cfs_period_us 
# 100000

# 限制进程号 226
echo 226 > /sys/fs/cgroup/cpu/container/tasks 

top
# %Cpu0 : 20.3 us, 0.0 sy, 0.0 ni, 79.7 id, 0.0 wa, 0.0 hi, 0.0 si, 0.0 st
```
此外还支持如：
 - blkio，为​​​块​​​设​​​备​​​设​​​定​​​I/O 限​​​制，一般用于磁盘等设备；
 - cpuset，为进程分配单独的 CPU 核和对应的内存节点；
 - cpu，为进程设定 CPU 的限制。
 - memory，为进程设定内存使用的限制。
 - ... 
 
 创建容器时直接指定Cgroups：
```docker
docker run -it --cpu-period=100000 --cpu-quota=20000 ubuntu /bin/bash
```

## namespace 和 Cgroups 的不足
 - 容器是一个“单进程”模型，一个容器内无法同时运行两个不同的应用。
 - 容器查询 /proc 的系统资源时，会返回**宿主机**的 /proc。
   因为 /proc 文件系统不了解 Cgroups 限制的存在。
   解决办法：把宿主机的 `/var/lib/lxcfs/proc/*` 文件挂载到容器的 `/proc/*`
 
