---
title: NVIDIA Docker 的安装和使用
tags:
  - Docker
  - NVIDIA
  - Deep Learning
  - Linux
abbrlink: bb9b6d25
date: 2021-01-20 22:14:01
categories:
- 技术经验总结
- Linux
---


## 前言
虽然在本地服务器安装 Anaconda 就可以跑深度学习运算，但是有时候我们的开发环境并不只有 Python。如果我们要复现论文中的模型或者 Github 上的代码时，我们可能需要按照它的要求安装指定版本的 GCC 及其它特定版本的开发包。久而久之，系统就会变得臃肿。<!-- more -->
多年的 Linux 操作系统使用经验告诉我，系统越臃肿越不稳定，系统越是滥用 sudo 权限越容易出问题。举个例子，在网上复制下来的命令，想都不想就用 sudo 权限安装一大堆连自己都不知道需不需要安装软件。又或者，在系统中某些配置文件可能为某次运算而特定配置的，然而在运算结束后忘记恢复了。这就很容易导致下次运算其他代码时，出现莫名奇妙的 BUG。还有，在安装大量的第三方 PPA 源后，在某次更新时出现的依赖问题。而依赖问题一般比较难解决，如果处理不好会让系统崩溃。
而 Docker 就可以很好的解决这些问题。你可以把它类比为虚拟机，但它比虚拟机更轻量，虚拟所带来的性能损失更低。在容器内（在虚拟机类比中就是一个虚拟系统）可以安装任意版本工具和依赖包，在配置开发环境过程中出现问题，直接删除容器即可。
注意在安装 NVIDIA Docker 之前需要安装 NVIDIA 驱动。由于本地服务器的 Anaconda 集成 CUDA 至 conda 中，而 NVIDIA Docker 使用容器内的 CUDA，所以可以不用在本地安装。至于如何安装驱动我就不说，方法有很多种，反正又不是很难。

## 安装
### docker-ce 安装
Ubuntu 官方的源里面也有包含 docker，不过版本比较低，所以我们还是添加 docker 官方的 PPA 源来安装 `docker-ce`。

首先安装支持 apt https 访问的依赖包。因为 docker 官方的 PPA 源是用 https 加密访问的。
```shell
$ sudo apt-get update
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

添加PPA源的 GPG 秘钥 
```shell
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

添加 `docker-ce` 的 PPA 源
```shell
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

安装 `docker-ce`
```shell
 $ sudo apt-get update
 $ sudo apt-get install docker-ce
```

**为了方便快速安装 docker-ce ，将命令都写到一个脚本里：**
```shell
#!/bin/bash

sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

sudo apt-get update
sudo apt-get install docker-ce
```

### nvidia-container-toolkit 安装
nvidia-container-toolkit 的安装方法和 docker-ce 的类似，也是需要添加源，然后使用 `apt-get` 来安装。但是由于 DNS 污染问题，`nvidia.github.io` 域名会指向 `127.0.0.1` 导致安装失败。所以我们需要在 `/etc/hosts` 文件里添加下面的内容。
```shell
# NVIDIA DOCKER Domain IP
185.199.108.153		nvidia.github.io
185.199.109.153		nvidia.github.io
185.199.110.153		nvidia.github.io
185.199.111.153		nvidia.github.io
```

接下来配置并添加 nvidia-container-toolkit 源
```shell
$ distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
   && curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add - \
   && curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
```

更新源并安装 nvidia-container-toolkit 
```shell
$ sudo apt-get update
$ sudo apt-get install nvidia-container-toolkit
```

重启 docker-ce
```shell
$ sudo systemctl restart docker
```

验证 nvidia-container-toolkit 是否成功安装。如果输出 `nvidia-smi` 相关信息，那说明安装成功。
```shell 
$ sudo docker run --rm --gpus all nvidia/cuda:11.0-base nvidia-smi
Unable to find image 'nvidia/cuda:11.0-base' locally
11.0-base: Pulling from nvidia/cuda
54ee1f796a1e: Pull complete 
f7bfea53ad12: Pull complete 
46d371e02073: Pull complete 
b66c17bbf772: Pull complete 
3642f1a6dfb3: Pull complete 
e5ce55b8b4b9: Pull complete 
155bc0332b0a: Pull complete 
Digest: sha256:774ca3d612de15213102c2dbbba55df44dc5cf9870ca2be6c6e9c627fa63d67a
Status: Downloaded newer image for nvidia/cuda:11.0-base
Wed Jan 20 20:36:14 2021       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 460.32.03    Driver Version: 460.32.03    CUDA Version: 11.2     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  TITAN RTX           Off  | 00000000:18:00.0 Off |                  N/A |
| 41%   55C    P2    64W / 280W |   5156MiB / 24220MiB |     28%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
+-----------------------------------------------------------------------------+
```

其中最后的参数 `nvidia/cuda:11.0-base` 是 NVIDIA 官方的镜像，需要根据工作站主机中实际安装的 CUDA 版本进行修改，版本可以用 `nvcc -V` 查看。

**为了方便快速安装 nvidia-container-toolkit ，将命令都写到一个脚本里：**
```shell
#!/bin/bash
sudo cat >> /etc/hosts <<EOF
# NVIDIA DOCKER Domain IP
185.199.108.153		nvidia.github.io
185.199.109.153		nvidia.github.io
185.199.110.153		nvidia.github.io
185.199.111.153		nvidia.github.io
EOF

distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
   && curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add - \
   && curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt-get update
sudo apt-get install nvidia-container-toolkit

sudo systemctl restart docker
sudo docker run --rm --gpus all nvidia/cuda:11.0-base nvidia-smi
```

## 使用命令
### 基本概念 
首先说一下 docker 的一些基本的概念。
* 镜像（ docker image ）：Docker 镜像是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。
* 容器（ docker container ）：容器的实质是进程，但与直接在宿主执行的进程不同，容器进程运行于属于自己的独立的命名空间。因此容器可以拥有自己的 root 文件系统、自己的网络配置、自己的进程空间，甚至自己的用户 ID 空间。容器内的进程是运行在一个隔离的环境里，使用起来，就好像是在一个独立于宿主的系统下操作一样。**容器与镜像的关系，就像是面向对象程序设计中的 `类` 和 `实例` 一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。**

下面来说一下基本的 docker 命令吧，学会这些命令可以涵盖很多使用场景了。

### 构建镜像
构建自己的 docker 镜像（也可以使用 Docker Hub 上面的镜像）
```shell
# <image-name> 自己设定的镜像名字
$ sudo docker build -t <image-name> .
```

查看和删除镜像
```shell
# 查看镜像。包括自己构建的镜像和在网上下载的镜像
$ sudo docker image list

# 删除镜像。在 rm 后面指定 <image-name> 或者 <image-id>
$ sudo docker image rm <image-id>
```

### 创建容器
以 `image-name` 为镜像创建 docker container，并且进入容器执行 `bash` 命令。执行 `bash` 命令的意思是进入容器后使用 `bash` 为 `shell` 的命令处理器，如果一些镜像没有 `bash`， 可以将 `bash` 改为 `sh` 
```shell
$ sudo docker run -it <image-name> bash
```

创建容器时指定参数。
```shell
# -v：挂载卷
# 参数格式：-v /local_path:/container_path，local_path 为本地服务器的绝对目录，container_path 为容器的绝对目录 
$ sudo docker run -it -v /path:/path <image-name> bash

# -p：暴露容器端口
# 参数格式：-p local_port:container_port，local_port 为本地服务器的端口，container_port 为容器的端口
$ sudo docker run -it -v /path:/path -p xxxx:xxxx <image-name> bash

# --name：指定容器的名字
# 参数格式：--name <container-name> 。如果多人共用一台服务器，最好指定容器的名字以便识别。
$ sudo ocker run -it -v /path:/path -p xxxx:xxxx --name <container-name> <image-name> bash
```

查看和删除容器
```shell
# 查看容器
$ sudo docker ps -a

# 删除容器。在 rm 后面指定 <container-name> 或者 <container-id>
$ sudo docker rm <container-id>
```

### 容器操作
```shell
# 启动容器
$ docker start <container-id>

# 停止容器
$ docker stop <container-id>

# 重启容器
$ docker restart <container-id>

# 进入容器
# exec 命令是在启动容器后才可以使用
$ docker exec -it <container-id> bash
```

### 创建容器并启用 GPU 
```shell
# --gpus：开启并指定GPU
# 参数格式：--gpus xxx

# 使用所有GPU
# sudo docker run -it -v /path:/path -p xxxx:xxxx --name <container-name> --gpus all <image-name> bash
$ sudo docker run -it --gpus all nvidia/cuda:11.0-base nvidia-smi

# 创建容器时分配的 GPU 数量
# sudo docker run -it -v /path:/path -p xxxx:xxxx --name <container-name> --gpus 2 <image-name> bash
$ docker run --gpus 2 nvidia/cuda:11.0-base nvidia-smi

# 创建容器时指定 GPU
# sudo docker run -it -v /path:/path -p xxxx:xxxx --name <container-name> --gpus '"device=x,x"' <image-name> bash
$ sudo docker run --gpus '"device=1,2"' nvidia/cuda:11.0-base nvidia-smi
$ sudo docker run --gpus '"device=UUID-ABCDEF,1"' nvidia/cuda:11.0-base nvidia-smi
```

**关于更多的 Docker 知识，可以查看教程地址：[Docker 从入门到实践](https://yeasy.gitbook.io/docker_practice/)**

## 关于安装和使用的技巧
* 更简单的使用 docker
为了方便使用 `docker` 命令，可以将自己的账号添加到 `docker` 用户组。如此就不需要在每次使用 `docker` 命令之前添加 `sudo`。
```shell
# user 为自己的用户名。执行命令后，需要注销当前用户才正式生效。
sudo usermod -aG docker user
```

* `install-nvidia-container-toolkit.sh`
为了方便安装 docker-ce 和 nvidia-container-toolkit，把所有命令综合起来（需要 `root` 权限执行）。
```shell
#!/bin/bash

#
# Add docker-ce to apt sources.list
#
apt-get update
apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

#
# add nvidia-container-toolkit to apt sources.list
#
cat >> /etc/hosts <<EOF
# NVIDIA DOCKER Domain IP
185.199.108.153		nvidia.github.io
185.199.109.153		nvidia.github.io
185.199.110.153		nvidia.github.io
185.199.111.153		nvidia.github.io
EOF

distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
   && curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | apt-key add - \
   && curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | tee /etc/apt/sources.list.d/nvidia-docker.list

#
# Install docker-ce and nvidia-container-toolkit
#
apt-get update
apt-get install docker-ce nvidia-container-toolkit

#
# Test docker-ce and nvidia-container-toolkit
#
systemctl restart docker
docker run --rm --gpus all nvidia/cuda:11.0-base nvidia-smi
```

* Miniconda Dockerfile 
 `conda` 镜像的 Dockerfile 文件。镜像使用的是 `Miniconda`。与 `Anaconda` 相比，`Miniconda` 不带有深度学习框架和科学计算包，体积更小，更精简。构建容器后，可根据需要自行安装相关的 Python 包，使用命令与 `Anaconda` 无异。
```Dockerfile
FROM nvidia/cuda:10.2-base

RUN  sed -i s@/archive.ubuntu.com/@/mirrors.aliyun.com/@g /etc/apt/sources.list && apt-get update; exit 0
RUN apt-get install -y vim wget curl cmake gcc && apt-get clean

RUN curl -o /root/miniconda.sh https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh && \
/bin/bash /root/miniconda.sh -b -p /miniconda && rm -rvf /root/miniconda.sh

RUN /miniconda/bin/conda init

WORKDIR /root
```

下面是构建 `Miniconda` 镜像的命令：
构建镜像时，最好新建一个文件夹，在文件夹内新建 Dockerfile 文件，并添加上面的内容，然后再执行下面的命令。注意确保文件夹内只有 Dockerfile 文件。否则在构建镜像时，docker 会自动缓存同文件夹内所有文件至容器里导致构建镜像时间比较长。
```shell
# 镜像名为 minicnoda-base-image
docker build -t minicnoda-base-image . 
```

## 参考资料
* [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/ "docker 安装")
* [NVIDIA Docker Installation Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
* [Docker 从入门到实践](https://yeasy.gitbook.io/docker_practice/)
