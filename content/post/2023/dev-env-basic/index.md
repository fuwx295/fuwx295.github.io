+++
author = "GreyWind"
title = "Development Environment (Basic)"
date = "2023-11-03"
description = "Development Environment (Basic)"
tags = [
    "development",
]
image = "dev.webp"
+++

**Support Platform: Ubuntu 22.04 & Mac Sonoma 14**

* Give the system description only on that platform, otherwise both are possible
    

## Add Account (Ubuntu)

尽量不要使用 root 账号来开发

```bash
$ useradd fuwx
$ passwd fuwx
```

添加用户 `sudoers`

```bash
$ sed -i '/^root.*ALL=(ALL).*ALL/a\fuwx\tALL=(ALL) \tALL' /etc/sudoers
```

## Bash

配置 `$HOME/.bashrc` 文件

```plaintext
# .bashrc
 
# User specific aliases and functions
 
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'
 
# Source global definitions
if [ -f /etc/bashrc ]; then
        . /etc/bashrc
fi
 
# User specific environment
# Basic envs
export LANG="en_US.UTF-8" # 设置系统语言为 en_US.UTF-8，避免终端出现中文乱码
export PS1='[\u@dev \W]\$ ' # 默认的 PS1 设置会展示全部的路径，为了防止过长，这里只展示："用户名@dev 最后的目录名"
export WORKSPACE="$HOME/workspace" # 设置工作目录
export PATH=$HOME/bin:$PATH # 将 $HOME/bin 目录加入到 PATH 变量中
 
# Default entry folder
cd $WORKSPACE # 登录系统，默认进入 workspace 目录
```

## GitHub

```bash
$ git config --global user.name "fuwx"    # 用户名改成自己的
$ git config --global user.email "f2928560492@163.com"    # 邮箱改成自己的
$ git config --global credential.helper store    # 设置 Git，保存用户名和密码
$ git config --global core.longpaths true # 解决 Git 中 'Filename too long' 的错误

$ git config --global core.quotepath off
$ git lfs install --skip-repo
```

## APT (Ubuntu)

配置 apt 镜像源

先备份 apt 配置文件

```bash
$ cp /etc/apt/sources.list /etc/apt/sources.list.bak
```

编辑 apt 配置文件

```bash
$ vim /etc/apt/sources.list
```

镜像源选择

* 中科大
    
    ```plaintext
    deb https://mirrors.ustc.edu.cn/ubuntu/ bionic main restricted universe multiverse
    deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic main restricted universe multiverse
    deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
    deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
    deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
    deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
    deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
    deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
    deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
    deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
    ```
    
* 阿里云
    
    ```plaintext
    deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
    deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
    deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
    deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
    deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
    deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
    deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
    deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
    deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
    deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
    ```
    
* 清华
    
    ```plaintext
    deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
    deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
    deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
    deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
    deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
    deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
    deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
    deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
    deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
    deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-proposed main restricted universe
    ```
    

## [HomeBrew](https://brew.sh/zh-cn/) (Mac)

Install

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## Go

配置 golang 环境

```bash
$ go env -w GO111MODULE=on
$ go env -w GOPROXY=https://goproxy.io,direct
```

## Cargo

配置 `$HOME/.cargo/config.toml` 文件

```plaintext
[registries]
ustc = { index = "https://mirrors.ustc.edu.cn/crates.io-index/" }
```

在项目中 `Cargo.toml` 中配置即可

```plaintext
[dependencies]
time = {  registry = "ustc" }
```

[字节跳动](https://rsproxy.cn/)

## Docker

Docker 镜像服务器设置

1. 编辑 `/etc/docker/daemon.json` 配置文件
    
    ```bash
    $ sudo nano /etc/docker/daemon.json
    ```
    
    ```json
    {
      "registry-mirrors": [
        "https://hub-mirror.c.163.com",
        "https://docker.mirrors.ustc.edu.cn",
        "https://mirror.baidubce.com"
      ]
    }
    ```
    
2. 重启 Docker 服务
    
    ```bash
    $ sudo systemctl daemon-reload 
    $ sudo systemctl restart docker
    ```