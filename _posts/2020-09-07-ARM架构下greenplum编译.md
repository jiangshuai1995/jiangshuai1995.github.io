---
layout:    post
title:     ARM架构下greenplum编译
subtitle:   greenplum编译
date:       2020-07-30
author:     Jiangs
header-img: img/view.jpg
catalog:    true
tags:
     - Greenplum
---

## 环境

ARM服务器\
CentOS7.5

## yum安装依赖包

```
yum install -y curl-devel bzip2-devel python-devel openssl-devel wget perl-ExtUtils-Embed libxml2-devel openldap-devel pam pam-devel gcc-c++ libtool libaio bison flex apr-devel readline-devel libevent-devel libffi-devel openssh* lockdev-devel
```

##  安装setuptools

由于安装pip未默认安装，使用pip需要先安装setuptools

```
tar xvf setuptools-20.3.tar.gz
cd setuptools-20.3
python setup.py  install
```

## 安装pip

```
tar xvf pip-20.2.2.tar.gz
cd pip-20.2.2
python setup.py  install
```

## pip安装依赖库

```
pip install --upgrade setuptools
```
本此编译中setuptools升级后版本为44

```
pip install conan
pip install cryptography==2.3
pip install paramiko=1.18.4
pip install ipaddress
pip install asn1crypto 
pip install psutil 
pip install lockfile
pip install astroid 
pip install bottle
pip install pycrypto
pip install ecdsa=0.13
```

不指定cryptography版本会报glob的错误

## 安装cmake
由于centos7安装cmake默认版本为2.8.4，无法直接使用yum install cmake安装

```
wget https://cmake.org/files/v3.3/cmake-3.3.2.tar.gz
tar -xzvf cmake-3.3.2.tar.gz
cd cmake-3.3.2
./bootstrap
gmake
make install
```

## 安装gp-xerces

注意此处xerces是用于gp的特定版本需要从github上下载其源码

```
tar -xzvf gp-xerces-3.1.2-p1.tar.gz
cd gp-xerces-3.1.2-p1/
./configure –prefix=/usr/local
make
make install
```

## 安装zstd
```
tar xvf zstd-1.3.8.tar.gz
cd zstd-1.3.8
make
make install
```


## 安装orca

本例中使用的是gpdb6.0.0.-beta1版本，指定orca为3.29

此处代码readme中使用ninja安装，但是由于ninjia对cmake版本有要求，本例中未使用ninja工具

```
tar xvf gporca-3.29.0.tar.gz
cd gporca-3.29.0
vi libgpos/src/common/CStackDescriptor.cpp
```

打开文件后注释167行

```cpp
void
CStackDescriptor::BackTrace
        (
        ULONG
        )
{
//      GPOS_CPL_ASSERT(!"Backtrace is not supported for this platform");
}

#endif
```

```
cmake -H. -Bbuild
cd build
make
make install

```

配置一下环境，不然会出现编译gpdb时找不到orca或者orca版本不对的问题
```
vi /etc/ld.so.conf
```
添加如下两行:
```
/usr/local/lib
/usr/local/include
```

执行ldconfig使其生效
```
ldconfig
```

## 安装gpdb

```
tar xvf gpdb-6.0.0-beta.1.tar.gz
cd gpdb-6.0.0-beta.1
./configure --with-perl --with-python --with-libxml --with-gssapi --prefix=/usr/local/gpdb
make
make install
```

## 安装postgis

以下为安装postgis的相关步骤

#### 安装geos

```
tar xvf geos-3.7.1.tar
cd geos-3.7.1
./configure  --prefix=/root/postgis/geos
make
make install
```

#### 安装proj

```
tar xvf proj-4.9.3.tar.gz
cd proj-4.9.3
./configure  --prefix=/root/postgis/proj
make
make install
```

#### 安装libxml2
```
tar xvf libxml2-2.9.4.tar.gz
cd  libxml2-2.9.4
./autogen.sh
./configure  --prefix=/root/postgis/libxml2
make
make install
```

#### 安装gdal
```
tar xvf gdal-2.2.3.tar.gz
cd gdal-2.2.3
./autogen.sh
./configure  --prefix=/root/postgis/gdal
make
make install
```

#### 安装jsonc
```
tar xvf json-c-0.12.1.tar.gz
cd json-c-0.12.1
./autogen.sh
./configure  --prefix=/root/postgis/json-c
make
make install
```


#### 安装protobuf
```
tar xvf protobuf-3.6.1.3.tar.gz
cd protobuf-3.6.1.3
./autogen.sh
./configure  --prefix=/root/postgis/protobuf
make
make install
```

#### 安装protobuf-c
```
tar xvf protobuf-c-1.3.1.tar.gz
cd protobuf-c-1.3.1
./autogen.sh
./configure  --prefix=/root/postgis/protobuf-c
make
make install
```

#### 安装postgis

此处用的是修改过的适配gp的postgis版本
```
tar xvf PostGIS2.5-master.tar.gz
cd postgis2.5

```

注意configure后的输出情况


## 安装madlib