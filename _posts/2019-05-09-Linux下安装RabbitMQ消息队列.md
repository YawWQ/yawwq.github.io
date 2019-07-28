---
layout: post
title:  "Linux下安装RabbitMQ消息队列"
date:   2019-05-09
excerpt: "如题"
tag:
- RabbitMQ
- Web后端
comments: true
---

## 安装依赖环境

1. 安装GCC等，装过的不用再装。

2. 安装ncurses

	yum -y install ncurses-devel

## 安装Erlang

1. 解压下载好的erlang文件：

	tar -zxvf otp_src_18.3.tar.gz

	cd /opt/otp_src_18.3/

2. 创建erlang目录

	mkdir /opt/erlang

3. 配置路径

	./configure --prefix=/opt/erlang

4. 执行编译结果

	make && make install

5. 配置erlang环境变量

	vi /etc/profile

	export PATH=$PATH:/opt/erlang/bin

6. 使得文件生效

	source  /etc/profile

## 安装RabbitMQ

1. 解压安装文件

	tar -xvJF rabbitmq-server-generic-unix-3.6.2.tar.xz

2. 配置环境变量

	export PATH=$PATH:/opt/rabbitmq/sbin

	source  /etc/profile

3. 进入sbin文件夹启动服务

	./rabbitmq-server -detached


**如果出现“permission denied”可以修改文件权限解决问题**