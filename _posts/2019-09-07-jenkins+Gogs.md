---
layout:     post
title:      jenkins+Goge
subtitle:   jenkins和Gogs实现自动化编译
date:       2019-09-05
author:     JS
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - Jenkins 
    - Gogs
---

## 背景

之前虽然用jenkins实现过自动化编译，但并未和git进行绑定。
使用的是定时编译，脚本中git pull拉取代码，这次试一下提交代码实时编译(触发脚本)

已有环境： jenkins gogs

## 搭建步骤

#### 安装gogs-webhooks

在jenkins插件管理界面 安装gogs plugin

#### 创建工程

创建工程后在配置界面会出现 gogs Webhook ,勾选Use Gogs secret

![](/img/postimg/jenkins1.png)

Source Code Management部分需要配置Credentials,点击Add 输出用户名密码，仓库地址也需要填写

![](/img/postimg/jenkins2.png)

Build Triggers部分需要勾选 Build when a change is pushed to Gogs

![](/img/postimg/jenkins3.png)

最后Build部分填写编译脚本就可以了

#### Gogs设置webhook

在仓库设置里添加WebHook

地址为如下格式

    http(s)://<< jenkins-server >>/gogs-webhook/?job=<< jobname >>

![](/img/postimg/jenkins4.png)

此时，环境已经搭建完成了，当有push推送到仓库时就会触发构建脚本。

## 使用说明

1.构建时会记录commit log与版本号
2.jenkins服务器本地会更新一份代码,默认工作空间在/var/lib/jenkins/workspace

在构建这个环境之前，计划时构建一个可以实时更新的Blog，通过触发构建脚本拉取代码，最后发现jenkins会自动拉取代码，所以直接使用jenkins下的仓库就可以了。


[参考文档](https://wiki.jenkins.io/display/JENKINS/Gogs+Webhook+Plugin)