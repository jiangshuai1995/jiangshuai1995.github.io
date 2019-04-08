---
layout:    post
title:     Pgbouncer测试
subtitle:   "Pgbouncer的连接测试"
date:       2019-04-08
author:     Jiangs
header-img: img/view.jpg
catalog:    true
tags:
     - pgbouncer
     - Greenplum
---

## 背景

在上次安装好Pgbouncer后需要测试是否会有性能提升，本次采用pgbench进行测试

由于Greenplum中未打包pgbench工具，故在pgbouncer服务器上安装postgresql10(rpm安装)来使用pgbench工具

数据库服务器 192.168.251.230 CentOS7.3 (Greenplum Master节点)

Pgbouncer服务器 192.168.250.33 CentOS7.3 (包含pgbench工具)

## 测试过程

#### Initialization

数据库服务器上执行

`pgbench -i -F 100 -s 500 testdb `

如果没有该数据库请先创建

#### Test1

```
[gpadmin@master ~]$ pgbench -c 10 -C -T 120 -S -h 192.168.251.230  -p 1328 testdb
Password: 
starting vacuum...end.
transaction type: <builtin: select only>
scaling factor: 500
query mode: simple
number of clients: 10
number of threads: 1
duration: 120 s
number of transactions actually processed: 18585
latency average = 64.590 ms
tps = 154.822784 (including connections establishing)
tps = 171.803839 (excluding connections establishing)
```

```
[gpadmin@master ~]$ pgbench -c 10 -C -T 120 -S -h 192.168.250.33  -p 6432 testdb
starting vacuum...end.
transaction type: <builtin: select only>
scaling factor: 500
query mode: simple
number of clients: 10
number of threads: 1
duration: 120 s
number of transactions actually processed: 88021
latency average = 13.637 ms
tps = 733.275173 (including connections establishing)
tps = 742.852783 (excluding connections establishing)
```

#### Test2

```
[gpadmin@master ~]$ pgbench -c 10 -C -T 120 -N  -h 192.168.251.230  -p 1328 testdb
Password: 
starting vacuum...end.
transaction type: <builtin: simple update>
scaling factor: 500
query mode: simple
number of clients: 10
number of threads: 1
duration: 120 s
number of transactions actually processed: 7306
latency average = 164.677 ms
tps = 60.724806 (including connections establishing)
tps = 62.876675 (excluding connections establishing)
```

```
[gpadmin@master ~]$ pgbench -c 10 -C -T 120 -N  -h 192.168.250.33  -p 6432 testdb
starting vacuum...end.
transaction type: <builtin: simple update>
scaling factor: 500
query mode: simple
number of clients: 10
number of threads: 1
duration: 120 s
number of transactions actually processed: 9455
latency average = 126.982 ms
tps = 78.751475 (including connections establishing)
tps = 78.877217 (excluding connections establishing)
```

#### Test3

```
[gpadmin@master ~]$ pgbench -c 50 -C -T 120 -N  -h 192.168.251.230  -p 1328 testdb
Password: 
starting vacuum...end.
transaction type: <builtin: simple update>
scaling factor: 500
query mode: simple
number of clients: 50
number of threads: 1
duration: 120 s
number of transactions actually processed: 9723
latency average = 618.331 ms
tps = 80.862891 (including connections establishing)
tps = 82.169761 (excluding connections establishing)
```

```
[gpadmin@master ~]$ pgbench -c 50 -C -T 120 -N  -h 192.168.250.33  -p 6432 testdb
starting vacuum...end.
transaction type: <builtin: simple update>
scaling factor: 500
query mode: simple
number of clients: 50
number of threads: 1
duration: 120 s
number of transactions actually processed: 14051
latency average = 435.113 ms
tps = 114.912709 (including connections establishing)
tps = 114.966602 (excluding connections establishing)
```



## 结论

1. 使用pgbouncer后连接时间明显缩减
2. 仅有select查询时性能提升最明显，推测是简单查询
3. 连接数越多性能提升越明显
4. 本次试用了-C参数，即对每个事务都创建了新的连接，强化了pgbouncer的性能优势，之后还需要进行多方面测试

## 出现问题

#### 问题1 

由于本地psql连接使用Unix domain socket，出现如下问题

```
psql: could not connect to server: 没有那个文件或目录
	Is the server running locally and accepting
	connections on Unix domain socket "/tmp/.s.PGSQL.6432"?
```
**解决方法**

将pgbouncer.ini中listen_addr改为实际IP，不使用127.0.0.1并重新启动

#### 问题2

使用pgbouncer进行压力测试时出现如下问题

```
client 0 aborted while establishing connection
connection to database "testdb" failed:
could not connect to server: Cannot assign requested address
	Is the server running on host "192.168.250.33" and accepting
	TCP/IP connections on port 6432?
```

**解决方法**

修改/etc/sysctl.conf文件

```
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
```

执行`sysctl -p`使其生效

#### 问题3

```
client 9 aborted in command 8 of script 0; ERROR:  duplicate key value violates unique constraint "pgbench_branches_pkey"  (seg1 192.168.251.232:50000 pid=29293)
DETAIL:  Key (bid)=(137) already exists.
```

由于该脚本中违反了GP中的约束，暂时没办法解决，先使用-N或-S模式测试，后面可能会重新写脚本不使用默认脚本

## 附录

以下为pgbench默认脚本
```
static char *tpc_b = {
 "\\set nbranches :scale\n"
 "\\set ntellers 10 * :scale\n"
 "\\set naccounts 100000 * :scale\n"
 "\\setrandom aid 1 :naccounts\n"
 "\\setrandom bid 1 :nbranches\n"
 "\\setrandom tid 1 :ntellers\n"
 "\\setrandom delta -5000 5000\n"
 "BEGIN;\n"
 "UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;\n"
 "SELECT abalance FROM pgbench_accounts WHERE aid = :aid;\n"
 "UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;\n"
 "UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;\n"
 "INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);\n"
 "END;\n"
};
/* -N case */
static char *simple_update = {
 "\\set nbranches :scale\n"
 "\\set ntellers 10 * :scale\n"
 "\\set naccounts 100000 * :scale\n"
 "\\setrandom aid 1 :naccounts\n"
 "\\setrandom bid 1 :nbranches\n"
 "\\setrandom tid 1 :ntellers\n"
 "\\setrandom delta -5000 5000\n"
 "BEGIN;\n"
 "UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;\n"
 "SELECT abalance FROM pgbench_accounts WHERE aid = :aid;\n"
 "INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);\n"
 "END;\n"
};
/* -S case */
static char *select_only = {
 "\\set naccounts 100000 * :scale\n"
 "\\setrandom aid 1 :naccounts\n"
 "SELECT abalance FROM pgbench_accounts WHERE aid = :aid;\n"
};
```
