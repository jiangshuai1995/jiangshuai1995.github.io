---
layout:    post
title:     Keepalived初识
subtitle:   使用keepalived搭建PG主备HA   
date:       2020-04-09
author:     Jiangs
header-img: img/view.jpg
catalog:    true
tags:
     - Postgresql
---

## 背景 

Keepalived是一个心跳检测工具，应用比较广泛，使用它来处理HA中的虚拟IP（VIP）非常合适。

## 基础环境
CentOS7     
Potgresql-12        

192.168.11.30 master        
192.168.11.22 backup        
192.168.11.28 vip       

## 流复制环境

流复制环境搭建此处不再叙述，具体请查阅《PG12同步流复制》

## 安装keepalived

```
yum install keepalived
```

本例中使用yum源安装，安装版本如下`keepalived x86_64 1.3.5-16.el7`

## 配置keepalived.conf
```
cd /etc/keepalived
mv keepalived.conf  keepalived.conf.bak
vi keepalived.conf
```

master上配置文件如下
```
! Configuration File for keepalived
vrrp_script check_pg_alived {
        script "/etc/keepalived/scripts/check_pg.sh"
        interval 2
}

vrrp_instance VI_1 {
    state MASTER
    interface p4p2
    virtual_router_id 22
    priority 100
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass 123456
    }


    virtual_ipaddress {
        192.168.11.28/24
    }

   track_script {
       check_pg_alived
   }
}

```
        

BACKUP上配置文件如下
```
! Configuration File for keepalived
vrrp_script check_pg_alived {
        script "/etc/keepalived/scripts/check_pg.sh"
        interval 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface p4p2
    virtual_router_id 22
    priority 80
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass 123456
    }


    virtual_ipaddress {
        192.168.11.28/24
    }

   track_script {
       check_pg_alived
   }
}

```
参数简要说明
vrrp_script check_pg_alived 定义了检查脚本，track_script中调用            
vrrp_instance VI_1 定义了名为VI_1的vrrp实例     
state  角色MASTER或BACKUP       
interface 网卡名称
virtual_router_id 保证主备同一路由ID即可        
virtual_ipaddress 虚拟IP地址        
priority 权重高的为主

## 配置PG监测脚本

```
mkdir scripts
cd scripts
vi check_pg.sh
chmod +x check_pg.sh
```
脚本内容如下，简单的检查pg进程是否存在
```
#!/bin/bash
num=`ps -ef |grep postgres|grep -v grep |wc -l`
if [ $num -ge 1 ];then
   status=0
else
   status=1
fi
exit $status
```

## 启动keepalived

```
[root@node3 scripts]# systemctl status  keepalived.service
● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; disabled; vendor preset: disabled)
   Active: active (running) since 四 2020-04-09 15:29:21 CST; 1min 10s ago
  Process: 32731 ExecStart=/usr/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 32732 (keepalived)
   CGroup: /system.slice/keepalived.service
           ├─32732 /usr/sbin/keepalived -D
           ├─32733 /usr/sbin/keepalived -D
           └─32734 /usr/sbin/keepalived -D
```

主节点部分日志如下
```
4月 09 15:29:51 node2 Keepalived_vrrp[25474]: Sending gratuitous ARP on p4p2 for 192.168.11.28
4月 09 15:29:51 node2 Keepalived_vrrp[25474]: Sending gratuitous ARP on p4p2 for 192.168.11.28
4月 09 15:29:51 node2 Keepalived_vrrp[25474]: Sending gratuitous ARP on p4p2 for 192.168.11.28
4月 09 15:29:51 node2 Keepalived_vrrp[25474]: Sending gratuitous ARP on p4p2 for 192.168.11.28
4月 09 15:29:56 node2 Keepalived_vrrp[25474]: Sending gratuitous ARP on p4p2 for 192.168.11.28
4月 09 15:29:56 node2 Keepalived_vrrp[25474]: VRRP_Instance(VI_1) Sending/queueing gratuitous ARPs on p4p2 for 192.168.11.28
4月 09 15:29:56 node2 Keepalived_vrrp[25474]: Sending gratuitous ARP on p4p2 for 192.168.11.28
4月 09 15:29:56 node2 Keepalived_vrrp[25474]: Sending gratuitous ARP on p4p2 for 192.168.11.28
4月 09 15:29:56 node2 Keepalived_vrrp[25474]: Sending gratuitous ARP on p4p2 for 192.168.11.28
4月 09 15:29:56 node2 Keepalived_vrrp[25474]: Sending gratuitous ARP on p4p2 for 192.168.11.28
```
可以看到VIP已经分配给了主节点

执行ip a也可以看到p4p2（网卡名）下有了IP28
```
[root@node2 scripts]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: p4p2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether a4:1f:72:73:f1:ec brd ff:ff:ff:ff:ff:ff
    inet 192.168.11.30/24 brd 192.168.11.255 scope global p4p2
       valid_lft forever preferred_lft forever
    inet 192.168.11.28/24 scope global secondary p4p2
       valid_lft forever preferred_lft forever
    inet6 fe80::a61f:72ff:fe73:f1ec/64 scope link 
       valid_lft forever preferred_lft forever
3: wlp3s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN qlen 1000
    link/ether e2:a6:c5:3e:00:63 brd ff:ff:ff:ff:ff:ff
```

## 测试VIP切换

__测试1__ 主节点上停止PG服务，主keepalived日志如下
```
4月 09 16:48:58 node2 Keepalived_vrrp[25474]: /etc/keepalived/scripts/check_pg.sh exited with status 1
4月 09 16:48:58 node2 Keepalived_vrrp[25474]: VRRP_Script(check_pg_alived) failed
4月 09 16:49:00 node2 Keepalived_vrrp[25474]: VRRP_Instance(VI_1) Entering FAULT STATE
4月 09 16:49:00 node2 Keepalived_vrrp[25474]: VRRP_Instance(VI_1) removing protocol VIPs.
4月 09 16:49:00 node2 Keepalived_vrrp[25474]: VRRP_Instance(VI_1) Now in FAULT state
4月 09 16:49:00 node2 Keepalived_vrrp[25474]: /etc/keepalived/scripts/check_pg.sh exited with status 1
```
备节点成功获取到VIP


__测试2__ 主节点上停止keepalived服务，备keepalived日志如下
```
4月 09 16:55:07 node2 Keepalived_vrrp[25474]: VRRP_Instance(VI_1) Transition to MASTER STATE
4月 09 16:55:09 node2 Keepalived_vrrp[25474]: VRRP_Instance(VI_1) Entering MASTER STATE
4月 09 16:55:09 node2 Keepalived_vrrp[25474]: VRRP_Instance(VI_1) setting protocol VIPs.
4月 09 16:55:09 node2 Keepalived_vrrp[25474]: Sending gratuitous ARP on p4p2 for 192.168.11.28
4月 09 16:55:09 node2 Keepalived_vrrp[25474]: VRRP_Instance(VI_1) Sending/queueing gratuitous ARPs on p4p2 for 192.168.11.28
4月 09 16:55:09 node2 Keepalived_vrrp[25474]: Sending gratuitous ARP on p4p2 for 192.168.11.28
4月 09 16:55:09 node2 Keepalived_vrrp[25474]: Sending gratuitous ARP on p4p2 for 192.168.11.28
4月 09 16:55:09 node2 Keepalived_vrrp[25474]: Sending gratuitous ARP on p4p2 for 192.168.11.28
4月 09 16:55:09 node2 Keepalived_vrrp[25474]: Sending gratuitous ARP on p4p2 for 192.168.11.28
```
此时会出现问题，所以需要进一步研究。