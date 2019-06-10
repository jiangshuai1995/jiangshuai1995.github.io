---
layout:     post
title:      VSCode-remote
subtitle:    "VSCode-remote功能试用"
date:       2019-06-10
author:     JS
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - VSCode
---

## 背景

前段时间微软发布了VSCode1.35版本,之前测试的remote功能正式成为稳定版，所以我决定试用一下

## 安装步骤

#### VSCODE安装扩展Remote Development

安装完成后在左侧菜单栏会出现一个小电脑图标

####  WINDOW安装OPENSSH

首先以管理员身份启动 PowerShell。 然后执行如下代码

```
Get-WindowsCapability -Online | ? Name -like 'OpenSSH*'
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```
此步骤可以参考该网址[window10下OpenSSH安装](https://docs.microsoft.com/zh-cn/windows-server/administration/openssh/openssh_install_firstuse)

操作系统版本为1809以上可以省去此步骤

#### 配置密钥

cmd执行 `ssh-keygen -t` 会在用户目录下生成密钥文件

例如我的目录是 C:\Users\Jiangs\ .ssh

将生成的密钥id_rsa.pub的内容追加到远程linux主机的authorized_keys文件中

配置完成后用cmd ssh连接先进行测试

此步骤有疑问可自行百度配置ssh互信

#### 创建remote配置文件

```
Host 192.168.250.50
    HostName 192.168.250.50
    User gpadmin
    IdentityFile  C:\Users\Jiangs\.ssh\id_rsa
```

然后选择配置文件就可以连接主机了，开启linux的代码之旅了

## 试用结论

这次试用只测试了linux，官方说还支持容器和WSL我并没有进行测试。总体来说部署比较方便，代码调试等内容都正常，可以减少很多交叉编译的工作。不用重复安装IDE也确实减少了很大的工作量。

## 文档及参考

[vscode-extension-remote](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)

    
