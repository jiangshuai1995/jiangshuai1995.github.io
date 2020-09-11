---
layout:    post
title:     远程调试postgresql
subtitle:   利用vscode调试linux下PG
date:       2020-07-30
author:     Jiangs
header-img: img/view.jpg
catalog:    true
tags:
     - Postgresql
---

## yum安装依赖库


yum install gcc gcc-c++ python python-devel tcl tcl-devel perl perl-devel zlib zlib-devel openssl openldap libxml2 libxml2-devel libxslt libxslt-devel python-kerberos readline-devel openssl-devel pam pam-devel perl-ExtUtils-Embed krb5-libs krb5-devel automake libtool autoconf gperf texinfo  gettext gettext-devel perl-libintl byacc  flex libcurl libcurl-devel dos2unix gdb gdb-gdbserver

## 编译安装cmake

由于centos默认的cmake版本过低，所以使用编译安装cmake

```
wget https://cmake.org/files/v3.9/cmake-3.9.6.tar.gz
tar xvf cmake-3.9.6.tar.gz
cd cmake-3.9.6
./bootstrap
gmake
make install
```

## 编译postgresql Debug版本


## VSCODE插件安装

- 首先需要有remote插件
- 其次调通远程后在linux上安装c/c++插件

## 启动数据库并开启连接

## 创建lanch.json文件

