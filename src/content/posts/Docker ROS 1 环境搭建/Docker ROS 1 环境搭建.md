---
title: Docker ROS 1 环境搭建全攻略：实现 RViz 在 Linux 宿主机及 Windows SSH 双平台的显示
published: 2025-08-23
description: ''
image: ''
tags: [X11, docker, ROS, RViz, ROS1, Windows, SSH, VcXsrv]
category: ''
draft: false 
lang: ''
---

  

## 零、背景介绍

工作环境：Windows + VS Code 通过 SSH 连接 Linux 主机（Ubuntu 24），在 Linux 上的 Docker 容器内运行 ROS 仿真。


目标：将容器内的 RViz 跨平台显示到 Windows。

  

思路：先在 Linux 宿主机验证 X11 转发（xeyes/gedit），再扩展到 Windows 并运行 RViz。

  

## 一、Linux宿主机直接打开 `xeyes`、`gedit`

  

为了方便后续扩展，本次实践采用“Dockerfile + Docker Compose + start.sh”的组合方式：

1. **Dockerfile** 用于定义镜像内容与构建步骤

2. **Docker Compose** 负责编排服务与运行时配置

3. **start.sh** 则封装常用启动命令，提供统一入口。

  

### 1、Docker 文件

**Dockerfile**：基于轻量镜像，安装 `x11-apps`、`gedit`

```dockerfile

# 基础镜像

FROM ubuntu

# 非交互式安装, 避免apt-get安装时的交互提示

ENV DEBIAN_FRONTEND=noninteractive

# 安装gedit, xeyes

RUN apt-get update && apt-get install -y \

    gedit \

    x11-apps

# 默认启动命令：进入 bash 交互环境

CMD ["bash"]

```

**docker-compose.yml**：挂载 `/tmp/.X11-unix`，透传 `$DISPLAY`

  

注：Docker 资源名称必须全部使用小写字母，包括服务名(service)、镜像名(image)、容器名(container)

  

```yaml

services:

# service name

learn-docker:

  build: .

  image: learn-docker:latest

  container_name: learn-docker

  

  # 保持容器运行

  stdin_open: true # 保持STDIN打开

  tty: true # 分配tty

  

  environment:

  # X11相关，透传 DISPLAY 环境变量

    - DISPLAY=${DISPLAY}

  volumes:

  # X11相关，挂载X11 socket

    - /tmp/.X11-unix:/tmp/.X11-unix

```

**start.sh**：统一封装 `docker compose up`

```bash

#!/bin/bash

set -e

# 构建并启动

docker compose up -d --build learn-docker

# 进入容器

docker exec -it learn-docker bash

```

### 2、启动前设置

  

在宿主机执行：

  

```bash

chmod +x start.sh # 给 start.sh 增加执行权限

./start.sh

```

  

进入容器后运行：

  

```bash

# '&'的作用: 将命令放到后台运行，终端可以继续输入命令

xeyes &

gedit &

```

  

此时应用窗口出现在 Linux 桌面上。

  

## 二、Windows SSH 到 Linux 宿主机显示 `xeyes`、`gedit`

  

### 1、转发设置

- Windows 上安装 **VcXsrv**，打开 **XLaunch**，设置一路默认 → 最后勾选 _Disable access control_

  

[XLaunch 设置](./Pasted image 20250823113002.png)

### 2、Docker 文件

  

**Dockerfile**保持不变

  

**docker-compose.yml**

environment 中的 DISPLAY 改为 Windows 的 IP 地址

```yaml

# 其他保持不变

    environment:

    # 旧：DISPLAY=${DISPLAY}

    # 新：DISPLAY=windows_ip:0.0，windows_ip为Windows的IP地址

    # 例：DISPLAY=192.168.1.101:0.0

      - DISPLAY=192.168.1.101:0.0

    volumes:

    # X11相关，挂载X11 socket

      - /tmp/.X11-unix:/tmp/.X11-unix

```

  

**start.sh**：根据服务名修改

```bash

#!/bin/bash

set -e

# 构建并启动

docker compose up -d --build your-service-name

# 进入容器

docker exec -it your-service-name bash

```

### 3、启动步骤

  

在 Linux 终端（terminal）：

  

```bash

./start.sh

xeyes &

gedit &

```

  

此时窗口显示在 Windows 桌面上。

  

## 三、在 Windows 上显示 RViz

ROS 1 里 RViz 等 GUI 程序由 Qt 实现，Qt 默认使用 X11，因此需要设置环境变量 `QT_X11_NO_MITSHM=1`。

  

### 1、Docker 文件

  

**Dockerfile**：基础镜像换成 `osrf/ros:noetic-desktop-full`

```dockerfile

# 基础镜像：带 rviz/rviz_plugin 的 ROS1 Noetic 全家桶

FROM osrf/ros:noetic-desktop-full

  

ENV DEBIAN_FRONTEND=noninteractive

  

RUN apt-get update && apt-get install -y \

    gedit \

    x11-apps \

    mesa-utils \

    && echo "source /opt/ros/noetic/setup.bash" >> /etc/bash.bashrc \

    && rm -rf /var/lib/apt/lists/*

  

# 对远端X服务器更稳的环境变量

ENV QT_X11_NO_MITSHM=1

# 如需禁用直连GL，可启用：LIBGL_ALWAYS_INDIRECT=1

  

CMD ["bash"]

```

  

**docker-compose.yml**：继承二的配置

  

**start.sh**：继承二的配置

  

### 2、启动

  

```bash

./start.sh

roscore &

rviz &

```

  

此时 RViz 应该出现在 Windows 桌面上。

  

## 四、在 Windows Terminal 或 VS Code 内直接使用 `./start.sh`

  

### 1、Windows Terminal

  

SSH 连接 Linux 时添加 `-Y` 参数：

  

  ```bash

  ssh -Y user@linux-host

  ```

  

### 2、VS Code

  

在 `~/.ssh/config` 增加：

  

  ```

  ForwardX11 yes

  ForwardX11Trusted yes

  ```

  

之后就可以在 VS Code 里 SSH 到 Linux 端，并在终端（terminal）运行 `./start.sh`