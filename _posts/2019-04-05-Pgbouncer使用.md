---
layout:    post
title:     Pgbouncer初识
subtitle:   "Pgbouncer工具的安装与使用"
date:       2019-04-05
author:     Jiangs
header-img: img/view.jpg
catalog:    true
tags:
     - pgbouncer
     - postgresql
---

## 背景

为了减少连接数据库的开销，准备使用pgbouncer工具，先将pgbonucer工具安装完成在进行使用及性能测试

此次未将pgbonucer安装在数据库服务器上，而是使用另一台安装了psql客户端的服务器

数据库服务器 192.168.251.230 CentOS7.3

Pgbouncer服务器 192.168.250.33 CentOS7.3

注意修改数据库服务的pg_hba文件使pgbouncer服务器能够访问数据库

## 安装过程

#### 安装依赖

>由于该服务器已经预装了libevent及libevent-devel，故该步骤跳过

`yum install -y libevent libevent-devel`

#### 下载与安装

```
wget https://github.com/pgbouncer/pgbouncer/releases/download/pgbouncer_1_9_0/pgbouncer-1.9.0.tar.gz
tar xvf pgbouncer-1.9.0.tar.gz
./configure --prefix=/usr/local --with-libevent=libevent-prefix
make
make install
```

#### 配置cfg文件

将安装目录下etc/pgbouncer.ini拷贝至/home/gpadmin/pgbouncer.ini

属主修改为gpadmin

修改或添加如下项

```
[databases]

demo = dbname=postgres host=192.168.251.230 port=1328 user=gpadmin password=123456

[pgbouncer]

logfile = /home/gpadmin/pgbouncer.log
pidfile = /home/gpadmin/pgbouncer.pid

listen_addr = 127.0.0.1
listen_port = 6432
auth_type = trust
auth_file = /home/gpadmin/userlist.txt
admin_users = gpadmin
max_client_conn = 300
default_pool_size = 20
```

#### 配置userlist.txt文件

将安装目录下etc/userlist.txt拷贝至/home/gpadmin/userlist.txt

属主修改为gpadmin

添加如下内容

```
"gpadmin" "123456"
```

#### 启动pgbouncer

注意不能使用root用户，本例默认使用gpadmin用户

`pgbouncer -d /home/gpadmin/pgbouncer.ini`

出现如下提示
```
2019-04-05 15:44:06.088 49558 LOG file descriptor limit: 1024 (H:4096), max_client_conn: 300, max fds possible: 330
```
并且/home/gpadmin目录下出现log及pid文件可以认为启动成功

## 连接测试

使用如下命令
`psql -p 6432 pgbouncer` 测试内置pgbouncer能否连接
`psql -p 6432 demo`      测试配置的demo数据库能否连接

## 出现问题

#### 问题1

```
$ psql -p 6432 pgbouncer
psql: ERROR:  not allowed
```
查看ini文件中admin_user是否为当前用户

#### 问题2

```
psql: could not connect to server: 没有那个文件或目录
	Is the server running locally and accepting
	connections on Unix domain socket "/tmp/.s.PGSQL.6432"?
```
查看pid文件是否存在，6432端口是否被占用



