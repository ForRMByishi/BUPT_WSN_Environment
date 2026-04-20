# 环境配置

> 🟢方案使用Docker，省去额外安装维护一套旧版Ubuntu的工作  
> 🟡旧软件源疑似已经失效，改用源码构建nesC 和 TinyOS tools

使用新版本Ubuntu作为宿主机，以下为整体流程：

* 安装 Docker Engine  
* 构建一个 Ubuntu 22.04 容器镜像  
* 在容器中源码编译 nesc
* 在容器中源码编译 tinyos-main/tools

## 在宿主机上安装Docker Engine

先配置 apt 仓库，再安装 docker-ce、docker-buildx-plugin、docker-compose-plugin

``` bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

当前用户加入 docker 组

```bash
sudo usermod -aG docker $USER
newgrp docker
docker version
```

## 准备TinyOS工作目录

在宿主机上新建一个目录

```bash
mkdir -p ~/tinyos-sim
cd ~/tinyos-sim
```

## 创建Dockerfile

在 `~/tinyos-sim` 目录里新建 `Dockerfile`，内容如下：

``` bash
FROM ubuntu:16.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    stow \
    automake \
    autoconf \
    libtool \
    libc6-dev \
    git \
    graphviz \
    python \
    python-serial \
    python2.7-dev \
    python3 \
    python3-serial \
    bison \
    flex \
    gperf \
    emacs-nox \
    default-jdk \
    ca-certificates \
    make \
    gcc \
    g++ \
    wget \
    autoconf \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /opt

# 1) build nesC from source
RUN git clone https://github.com/tinyos/nesc.git && \
    cd nesc && \
    ./Bootstrap && \
    ./configure && \
    make -j1 && \
    make install

# 2) clone TinyOS main development tree
RUN git clone https://github.com/tinyos/tinyos-main.git /opt/tinyos-main

# 3) build tinyos tools from source
RUN cd /opt/tinyos-main/tools && \
    ./Bootstrap && \
    cd /opt/tinyos-main/tools/tinyos/c/coap && \
    autoreconf -fi && \
    cd /opt/tinyos-main/tools && \
    ./configure && \
    make -j1 && \
    make install

WORKDIR /work
CMD ["/bin/bash"]
```

## 构建镜像

``` bash
cd ~/tinyos-sim
docker build --no-cache -t tinyos-sim-src16 . # 关闭cache避免之前失败构建干扰
```

## 运行容器

测试时可以加`--rm`退出时容器自动删除

``` bash
mkdir -p ~/tinyos-sim/work

docker run -it --rm \
  --name tinyos-sim \
  -v ~/tinyos-sim/work:/work \
  tinyos-sim-src16
```

如果没有问题就去掉`--rm`

``` bash
mkdir -p ~/tinyos-sim/work

docker run -it \
  --name tinyos-sim \
  -v ~/tinyos-sim/work:/work \
  tinyos-sim-src16
```

之后想要重新进入容器就使用

``` bash
docker start -ai tinyos-sim
```

看所有容器：

``` bash
docker ps -a
```

停止容器：

``` bash
docker stop tinyos-sim
```

删除容器：

``` bash
docker rm tinyos-sim
```

## 容器内配置环境变量

``` bash
cat >> ~/.bashrc <<'EOF'
export TINYOS_ROOT_DIR=/opt/tinyos-main
export TR=$TINYOS_ROOT_DIR
export CLASSPATH=.:$TR/support/sdk/java/tinyos.jar
export PYTHONPATH=$TR/support/sdk/python:$PYTHONPATH
EOF

source ~/.bashrc
```

## 仿真编译

``` bash
cd /opt/tinyos-main/apps/Blink
make micaz sim
```