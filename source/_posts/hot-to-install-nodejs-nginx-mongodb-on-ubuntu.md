---
title: 在ubuntu上从零搭建node.js + nginx + mongodb环境
tags:
  - ubuntu
  - node.js
  - nginx
  - mongodb
categories:
  - 后端
date: 2019-12-09 10:40:05
---


说到后端开发环境，最有名的莫过于LAMP和LNMP，最近由于node.js的强势崛起，越来越多的后端开发也开始试水node.js了。我最近也因为各种原因，前前后后总够构建了好几台node.js + nginx + mongodb的Linux服务器。

首先关于Linux服务器，比起CentOS来说，我更加喜欢ubuntu一点。所以无论是阿里云还是一些海外的vps服务器上，我也倾向选用ubuntu服务器，本贴也是基于ubuntu服务器里说明的。

## 1.开始前的一些准备

首先还是需要刷新一下ubuntu的包索引并安装build-essential和libssl-dev这2个包以及curl这个工具。

```sh
sudo apt-get update
sudo apt-get install build-essential libssl-dev
sudo apt-get isntall curl
```

## 2.安装node.js

关于安装node.js这一点，我不是很推荐使用apt-get 来安装node.js的环境。主要是因为node.js和io.js合并以后，版本迭代速度相当频繁(主要还是因为更多ES6的特性得到了支持）。今后很有可能会有在一台服务器上使用不同版本的node.js的需求。

这里推荐一个管理不同版本node.js的工具：nvm，官网: [https://github.com/creationix/nvm](https://github.com/creationix/nvm)  。安装nvm，如果前面你安装了curl的话可以

```sh
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.31.0/install.sh | bash
```
如果没有按照curl的话，也可以使用wget来进行安装

```sh
wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.31.0/install.sh | bash
```

然后nvm就会自动安装到home目录下面的.nvm目录里，并会在.bashrc里自动添加nvm的环境变量。为了让环境变量生效，最简单的方法就是通过ssh或是telnet重新连接你的服务器。

安装完nvm后，就可以通过nvm来安装指定版本的node.js了。

```sh
# 列出可以安装的node版本号
nvm ls-remote

# 安装指定版本的node (当前最新版本为v5.7.1, LTS版是v4.3.2)
nvm install v4.3.2
```

## 3.安装nginx

由于ubuntu源（尤其是阿里云的源）上的nginx经常不是最新的，如果需要安装最新版本nginx的时候需要手动添加nginx的源。

```sh
# 添加nginx的mainline仓库
cd /tmp/ && wget http://nginx.org/keys/nginx_signing.key
sudo apt-key add nginx_signing.key

# 编辑/etc/apt/sources.list.d/nginx.list 添加下面2行内容，井号不需要
# deb http://nginx.org/packages/mainline/ubuntu/ ubuntu代号 nginx
# deb-src http://nginx.org/packages/mainline/ubuntu/ ubuntu代号 nginx
sudo vi  /etc/apt/sources.list.d/nginx.list

# 更新源，并安装nginx
sudo apt-get update && sudo apt-get install nginx
```

在编辑/etc/apt/sources.list.d/nginx.list的时候需要注意，“ubuntu代号”需要根据ubuntu服务器的版本不同手动调整的，比如14.04是trusty。通过下面的命令可以获取ubuntu的代号。
```sh
lsb_release -cs
```

## 4.安装mongodb

同样和nginx有同样的问题，要安装最新3.2版本的mongodb也需要手动添加ubuntu的源。

```sh
# 导入mongodb的public key
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv EA312927

# 生成mongodb的源list
echo "deb http://repo.mongodb.org/apt/ubuntu trusty/mongodb-org/3.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.2.list

# 更新源
sudo apt-get update

# 安装最新版本的mongodb
sudo apt-get install -y mongodb-org
```

以上一台node.js + nginx + mongodb的ubuntu服务器就完成了。

