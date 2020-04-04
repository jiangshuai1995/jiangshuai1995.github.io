## ETCD + Patroni

### 服务器设置 

CentOS7 四台服务器      
192.168.11.28 node1  master     
192.168.11.30 node2  standby            
192.168.11.22 node3  standby        
192.168.11.29 node4  etcd       

postgresql 12

### 架构说明

### 基础环境

1. 关闭防火墙
2. 安装openssl[yum安装]
3. 时间同步

----------------------

### 安装PG12

master上可以initdb，standby上安装即可，etcd节点无需安装PG

### 安装etcd

node4上执行如下操作安装etcd
```
yum install etcd
cp /etc/etcd/etcd.conf /etc/etcd/etcd.conf.bak
vi /etc/etcd/etcd.conf 
```

复制如下内容，注意IP修改地址即可
```
#[Member]
#ETCD_CORS=""
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#ETCD_WAL_DIR=""
ETCD_LISTEN_PEER_URLS="http://192.168.11.29:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.11.29:2379,http://127.0.0.1:2379"
#ETCD_MAX_SNAPSHOTS="5"
#ETCD_MAX_WALS="5"
ETCD_NAME="default"
#ETCD_SNAPSHOT_COUNT="100000"
#ETCD_HEARTBEAT_INTERVAL="100"
#ETCD_ELECTION_TIMEOUT="1000"
#ETCD_QUOTA_BACKEND_BYTES="0"
#ETCD_MAX_REQUEST_BYTES="1572864"
#ETCD_GRPC_KEEPALIVE_MIN_TIME="5s"
#ETCD_GRPC_KEEPALIVE_INTERVAL="2h0m0s"
#ETCD_GRPC_KEEPALIVE_TIMEOUT="20s"
#
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.11.29:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.11.29:2379"
#ETCD_DISCOVERY=""
#ETCD_DISCOVERY_FALLBACK="proxy"
#ETCD_DISCOVERY_PROXY=""
#ETCD_DISCOVERY_SRV=""
ETCD_INITIAL_CLUSTER="default=http://192.168.11.29:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_STRICT_RECONFIG_CHECK="true"
#ETCD_ENABLE_V2="true"
#
#[Proxy]
#ETCD_PROXY="off"
#ETCD_PROXY_FAILURE_WAIT="5000"
#ETCD_PROXY_REFRESH_INTERVAL="30000"
#ETCD_PROXY_DIAL_TIMEOUT="1000"
#ETCD_PROXY_WRITE_TIMEOUT="5000"
#ETCD_PROXY_READ_TIMEOUT="0"
#
#[Security]
#ETCD_CERT_FILE=""
#ETCD_KEY_FILE=""
#ETCD_CLIENT_CERT_AUTH="false"
#ETCD_TRUSTED_CA_FILE=""
#ETCD_AUTO_TLS="false"
#ETCD_PEER_CERT_FILE=""
#ETCD_PEER_KEY_FILE=""
#ETCD_PEER_CLIENT_CERT_AUTH="false"
#ETCD_PEER_TRUSTED_CA_FILE=""
#ETCD_PEER_AUTO_TLS="false"
#
#[Logging]
#ETCD_DEBUG="false"
#ETCD_LOG_PACKAGE_LEVELS=""
#ETCD_LOG_OUTPUT="default"
#
#[Unsafe]
#ETCD_FORCE_NEW_CLUSTER="false"
#
#[Version]
#ETCD_VERSION="false"
#ETCD_AUTO_COMPACTION_RETENTION="0"
#
#[Profiling]
#ETCD_ENABLE_PPROF="false"
#ETCD_METRICS="basic"
#
#[Auth]
#ETCD_AUTH_TOKEN="simple"
```

查看ETCD服务状态
```
# systemctl status etcd.service 
● etcd.service - Etcd Server
   Loaded: loaded (/usr/lib/systemd/system/etcd.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
```

修改service文件，本例中是按照ETCD集群进行启动 ------ [_可以调研有没有单点启动方式_]
```
# vi /usr/lib/systemd/system/etcd.service
[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd \
--name=\"${ETCD_NAME}\" \
--data-dir=\"${ETCD_DATA_DIR}\" \
--listen-peer-urls=\"${ETCD_LISTEN_PEER_URLS}\" \
--listen-client-urls=\"${ETCD_LISTEN_CLIENT_URLS}\" \
--initial-advertise-peer-urls=\"${ETCD_INITIAL_ADVERTISE_PEER_URLS}\" \
--advertise-client-urls=\"${ETCD_ADVERTISE_CLIENT_URLS}\" \
--initial-cluster=\"${ETCD_INITIAL_CLUSTER}\"  \
--initial-cluster-token=\"${ETCD_INITIAL_CLUSTER_TOKEN}\" \
--initial-cluster-state=\"${ETCD_INITIAL_CLUSTER_STATE}\""
Restart=on-failure
LimitNOFILE=65536
```

启动ETCD
```
systemctl start etcd.service
```


检查etcd状态
```
[root@node4 ~]# etcdctl cluster-health
member 8e9e05c52164694d is healthy: got healthy result from http://192.168.11.29:2379
cluster is healthy
```

### 安装Patroni

node1,node2,node3 上需要安装[由于从多个节点截取的输出和错误，所以命令前的主机名有node1或者node2，3，不必在意]

patroni依赖python环境，本例中使用的python是CentOS7自带的2.7.5

安装PIP
```
[root@node2 ~]# curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 1764k  100 1764k    0     0   301k      0  0:00:05  0:00:05 --:--:--  370k
[root@node2 ~]# python get-pip.py 
DEPRECATION: Python 2.7 reached the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 is no longer maintained. A future version of pip will drop support for Python 2.7. More details about Python 2 support in pip, can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support
Collecting pip
  Downloading pip-20.0.2-py2.py3-none-any.whl (1.4 MB)
     |████████████████████████████████| 1.4 MB 764 kB/s 
Collecting wheel
  Downloading wheel-0.34.2-py2.py3-none-any.whl (26 kB)
Installing collected packages: pip, wheel
  Attempting uninstall: pip
    Found existing installation: pip 1.5.4
    Uninstalling pip-1.5.4:
      Successfully uninstalled pip-1.5.4
Successfully installed pip-20.0.2 wheel-0.34.2
```

更新setuptools版本
```
[root@node3 ~]# pip uninstall setuptools
DEPRECATION: Python 2.7 reached the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 is no longer maintained. A future version of pip will drop support for Python 2.7. More details about Python 2 support in pip, can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support
Found existing installation: setuptools 0.9.8
Uninstalling setuptools-0.9.8:
  Would remove:
    /usr/bin/easy_install
    /usr/bin/easy_install-2.7
    /usr/lib/python2.7/site-packages/_markerlib
    /usr/lib/python2.7/site-packages/easy_install.py
    /usr/lib/python2.7/site-packages/easy_install.pyo
    /usr/lib/python2.7/site-packages/pkg_resources.py
    /usr/lib/python2.7/site-packages/pkg_resources.pyo
    /usr/lib/python2.7/site-packages/setuptools
    /usr/lib/python2.7/site-packages/setuptools-0.9.8-py2.7.egg-info
Proceed (y/n)? y
  Successfully uninstalled setuptools-0.9.8
[root@node3 ~]# pip install setuptools
DEPRECATION: Python 2.7 reached the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 is no longer maintained. A future version of pip will drop support for Python 2.7. More details about Python 2 support in pip, can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support
Collecting setuptools
  Downloading setuptools-44.1.0-py2.py3-none-any.whl (583 kB)
     |████████████████████████████████| 583 kB 523 kB/s 
Installing collected packages: setuptools
Successfully installed setuptools-44.1.0
```

安装configparser
```
[root@node1 ~]# pip install configparser
DEPRECATION: Python 2.7 reached the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 is no longer maintained. A future version of pip will drop support for Python 2.7. More details about Python 2 support in pip, can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support
Collecting configparser
  Downloading configparser-4.0.2-py2.py3-none-any.whl (22 kB)
Installing collected packages: configparser
Successfully installed configparser-4.0.2
```

安装psycopg2-binary
```
[root@node2 ~]# pip install psycopg2-binary
DEPRECATION: Python 2.7 reached the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 is no longer maintained. A future version of pip will drop support for Python 2.7. More details about Python 2 support in pip, can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support
Collecting psycopg2-binary
  Downloading psycopg2_binary-2.8.4-cp27-cp27mu-manylinux1_x86_64.whl (2.9 MB)
     |████████████████████████████████| 2.9 MB 146 kB/s 
Installing collected packages: psycopg2-binary
Successfully installed psycopg2-binary-2.8.4
```

安装pyopenssl
```
[root@node1 ~]# pip install pyopenssl
DEPRECATION: Python 2.7 reached the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 is no longer maintained. A future version of pip will drop support for Python 2.7. More details about Python 2 support in pip, can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support
Requirement already satisfied: pyopenssl in /usr/lib64/python2.7/site-packages (0.13.1)
```

安装patroni
```
[root@node2 patroni]# pip install patroni[etcd]
DEPRECATION: Python 2.7 reached the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 is no longer maintained. A future version of pip will drop support for Python 2.7. More details about Python 2 support in pip, can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support
Requirement already satisfied: patroni[etcd] in /usr/lib/python2.7/site-packages (1.5.6)
Requirement already satisfied: requests in /usr/lib/python2.7/site-packages (from patroni[etcd]) (2.6.0)
Requirement already satisfied: six>=1.7 in /usr/lib/python2.7/site-packages (from patroni[etcd]) (1.9.0)
Requirement already satisfied: cdiff in /usr/lib/python2.7/site-packages (from patroni[etcd]) (1.0)
Requirement already satisfied: PyYAML in /usr/lib64/python2.7/site-packages (from patroni[etcd]) (5.3.1)
Requirement already satisfied: click>=4.1 in /usr/lib/python2.7/site-packages (from patroni[etcd]) (7.1.1)
Requirement already satisfied: psutil>=2.0.0 in /usr/lib64/python2.7/site-packages/psutil-4.1.0-py2.7-linux-x86_64.egg (from patroni[etcd]) (4.1.0)
Requirement already satisfied: urllib3!=1.21,>=1.19.1 in /usr/lib/python2.7/site-packages (from patroni[etcd]) (1.25.8)
Requirement already satisfied: prettytable>=0.7 in /usr/lib/python2.7/site-packages (from patroni[etcd]) (0.7.2)
Requirement already satisfied: python-dateutil in /usr/lib/python2.7/site-packages (from patroni[etcd]) (1.5)
Requirement already satisfied: tzlocal in /usr/lib/python2.7/site-packages (from patroni[etcd]) (2.0.0)
Collecting python-etcd<0.5,>=0.4.3; extra == "etcd"
  Downloading python-etcd-0.4.5.tar.gz (37 kB)
Requirement already satisfied: pytz in /usr/lib/python2.7/site-packages (from tzlocal->patroni[etcd]) (2012d)
Collecting dnspython>=1.13.0
  Downloading dnspython-1.16.0-py2.py3-none-any.whl (188 kB)
     |████████████████████████████████| 188 kB 636 kB/s 
Building wheels for collected packages: python-etcd
  Building wheel for python-etcd (setup.py) ... done
  Created wheel for python-etcd: filename=python_etcd-0.4.5-py2-none-any.whl size=38498 sha256=5f352808daa04754b8da7c52fff5056138bcd52993e83a3fcc4d50beb967e00e
  Stored in directory: /root/.cache/pip/wheels/48/40/4e/128dadf25274df79bd9e65365e375f77d7c1a9d0900c598c2a
Successfully built python-etcd
Installing collected packages: dnspython, python-etcd
  Attempting uninstall: dnspython
    Found existing installation: dnspython 1.12.0
    Uninstalling dnspython-1.12.0:
      Successfully uninstalled dnspython-1.12.0
Successfully installed dnspython-1.16.0 python-etcd-0.4.5
```

配置patroni文件
```
vi /etc/patroni/patroni_postgresql.yml
```
内容如下,不同节点请注意IP地址，name，以及认证部分的用户名密码，注意postgres此时还没有设置密码，replicator用户没有创建也没有设置密码【这两个用户指的都是数据库用户】
```
scope: pg12
namespace: /test/
name: pgsql12_node2

restapi:
  listen: 192.168.11.30:8008
  connect_address: 192.168.11.30:8008

etcd:
  host: 192.168.11.29:2379

bootstrap:
  # this section will be written into Etcd:/<namespace>/<scope>/config after initializing new cluster
  # and all other cluster members will use it as a `global configuration`
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    master_start_timeout: 300
    synchronous_mode: false
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        listen_addresses: "0.0.0.0"
        port: 5432
        wal_level: logical
        hot_standby: "on"
        wal_keep_segments: 1000
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
#        archive_mode: "on"
#        archive_timeout: 1800s
#        archive_command: gzip < %p > /data/backup/pgwalarchive/%f.gz
#      recovery_conf:
#        restore_command: gunzip < /data/backup/pgwalarchive/%f.gz > %p

postgresql:
  listen: 0.0.0.0
  connect_address: 192.168.11.30:5432
  data_dir: /var/lib/pgsql/12/data
  bin_dir: /usr/pgsql-12/bin
#  config_dir: /etc/postgresql/12/main
  authentication:
    replication:
      username: replicator
      password: '123456'
    superuser:
      username: postgres
      password: '123456'
#watchdog:
#  mode: automatic # Allowed values: off, automatic, required
#  device: /dev/watchdog
#  safety_margin: 5

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```


### 修改master数据库配置

node1 上执行

修改postgresql.conf文件
```
# vi /var/lib/pgsql/12/data/postgresql.conf
```
```
listen_addresses = '0.0.0.0'
hot_standby = 'on'
synchronous_standby_names = ''
```
修改pg_hba.conf文件
```
# vi /var/lib/pgsql/12/data/pg_hba.conf
```
注意一些ident的认证方式需要改为md5，不然会出现连接问题，12之后复制权限好像不能用all代表了，所以我们稍后会创建replicator复制用户，先加入认证。
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
host    all             all             192.168.0.0/16          md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication         replicator      192.168.0.0/16               md5
host    replication     all             127.0.0.1/32            md5
host    replication     all             ::1/128                 md5
```

### 启动数据库并创建角色

node1上执行

创建replicator角色和密码，修改postgres密码，因为默认是没有密码的
```
systemctl start postgresql-12
sudo -u postgres psql

postgres=# create user replicator replication login encrypted password '123456';


postgres=# \password postgres;
输入新的密码：
再次输入：
```
注意这个地方密码要和patroni配置文件中的密码一致


### 流复制

node2上执行
```
sudo -u postgres pg_basebackup -R -D /var/lib/pgsql/12/data -F p -X s -v -P -C -S pgsql12_node2 -h 192.168.11.28 -p 5432 -U postgres
```

node3上执行
```
sudo -u postgres pg_basebackup -R -D /var/lib/pgsql/12/data -F p -X s -v -P -C -S pgsql12_node3 -h 192.168.11.28 -p 5432 -U postgres
```

区别是-S参数创建的复制槽名称不同，这里要说明一下，复制槽名称需要和patroni文档中节点名称相同


### 启动patroni

node1，node2，node3 依次启动[请确保前一个启动成功再启动下一个]

```
sudo -u postgres patroni /etc/patroni/patroni_postgresql.yml
```

node1 输出如下
```
2020-04-03 17:14:14,036 INFO: no action.  i am the leader with the lock
2020-04-03 17:14:24,023 INFO: Lock owner: pgsql12_node1; I am pgsql12_node1
2020-04-03 17:14:24,030 INFO: no action.  i am the leader with the lock
2020-04-03 17:14:34,023 INFO: Lock owner: pgsql12_node1; I am pgsql12_node1
2020-04-03 17:14:34,030 INFO: no action.  i am the leader with the lock
2020-04-03 17:14:44,023 INFO: Lock owner: pgsql12_node1; I am pgsql12_node1
2020-04-03 17:14:44,031 INFO: no action.  i am the leader with the lock
2020-04-03 17:14:54,023 INFO: Lock owner: pgsql12_node1; I am pgsql12_node1
2020-04-03 17:14:54,032 INFO: no action.  i am the leader with the lock
```

node2 输出如下
```
2020-04-03 17:15:04,028 INFO: Lock owner: pgsql12_node1; I am pgsql12_node2
2020-04-03 17:15:04,028 INFO: does not have lock
2020-04-03 17:15:04,040 INFO: no action.  i am a secondary and i am following a leader
2020-04-03 17:15:14,028 INFO: Lock owner: pgsql12_node1; I am pgsql12_node2
2020-04-03 17:15:14,028 INFO: does not have lock
2020-04-03 17:15:14,045 INFO: no action.  i am a secondary and i am following a leader
2020-04-03 17:15:24,027 INFO: Lock owner: pgsql12_node1; I am pgsql12_node2
```

node3 输出如下
```
2020-04-03 17:14:54,014 INFO: Lock owner: pgsql12_node1; I am pgsql12_node3
2020-04-03 17:14:54,014 INFO: does not have lock
2020-04-03 17:14:54,031 INFO: no action.  i am a secondary and i am following a leader
2020-04-03 17:15:04,012 INFO: Lock owner: pgsql12_node1; I am pgsql12_node3
2020-04-03 17:15:04,012 INFO: does not have lock
2020-04-03 17:15:04,032 INFO: no action.  i am a secondary and i am following a 
```

###  查看集群状态

任何一个节点执行均可
```
[root@node2 ~]# patronictl -c /etc/patroni/patroni_postgresql.yml list
+---------+---------------+---------------+--------+---------+----+-----------+
| Cluster |     Member    |      Host     |  Role  |  State  | TL | Lag in MB |
+---------+---------------+---------------+--------+---------+----+-----------+
|   pg12  | pgsql12_node1 | 192.168.11.28 | Leader | running |  3 |           |
|   pg12  | pgsql12_node2 | 192.168.11.30 |        | running |  3 |       0.0 |
|   pg12  | pgsql12_node3 | 192.168.11.22 |        | running |  3 |       0.0 |
+---------+---------------+---------------+--------+---------+----+-----------+

```

至此 etcd+patroni集群部署完毕，主库可以读写，备库只读，可以连接主库查看复制详细信息
```
postgres=# \x
Expanded display is on.
postgres=# select * from  pg_replication_slots;
-[ RECORD 1 ]-------+--------------
slot_name           | pgsql12_node2
plugin              | 
slot_type           | physical
datoid              | 
database            | 
temporary           | f
active              | t
active_pid          | 2809
xmin                | 
catalog_xmin        | 
restart_lsn         | 0/9000638
confirmed_flush_lsn | 
-[ RECORD 2 ]-------+--------------
slot_name           | pgsql12_node3
plugin              | 
slot_type           | physical
datoid              | 
database            | 
temporary           | f
active              | t
active_pid          | 2810
xmin                | 
catalog_xmin        | 
restart_lsn         | 0/9000638
confirmed_flush_lsn | 

postgres=# select * from  pg_stat_replication ;
-[ RECORD 1 ]----+------------------------------
pid              | 2809
usesysid         | 16384
usename          | replicator
application_name | pgsql12_node2
client_addr      | 192.168.11.30
client_hostname  | 
client_port      | 40672
backend_start    | 2020-04-03 17:12:35.118859+08
backend_xmin     | 
state            | streaming
sent_lsn         | 0/9000638
write_lsn        | 0/9000638
flush_lsn        | 0/9000638
replay_lsn       | 0/9000638
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2020-04-03 17:21:25.949291+08
-[ RECORD 2 ]----+------------------------------
pid              | 2810
usesysid         | 16384
usename          | replicator
application_name | pgsql12_node3
client_addr      | 192.168.11.22
client_hostname  | 
client_port      | 38202
backend_start    | 2020-04-03 17:12:38.463031+08
backend_xmin     | 
state            | streaming
sent_lsn         | 0/9000638
write_lsn        | 0/9000638
flush_lsn        | 0/9000638
replay_lsn       | 0/9000638
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2020-04-03 17:21:26.104331+08

```






### 问题记录与处理

1. lock文件路径权限问题

    /var/run/postgresql/或/tmp无权限
    ```
    chown postgre：postgres /var/run/postgresql

    chown postgre：postgres /tmp
    ```

2. 安装了patroni1.5.4导致生成了recovery.conf文件，与PG12不兼容

3. 出现patroni无法安装，重新执行一遍后成功
    ```
    [root@node2 ~]# pip install patroni[etcd]
    DEPRECATION: Python 2.7 reached the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 is no longer maintained. A future version of pip will drop support for Python 2.7. More details about Python 2 support in pip, can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support
    Collecting patroni[etcd]
    Using cached patroni-1.6.4.tar.gz (128 kB)
        ERROR: Command errored out with exit status 1:
        command: /usr/bin/python -c 'import sys, setuptools, tokenize; sys.argv[0] = '"'"'/tmp/pip-install-djviZf/patroni/setup.py'"'"'; __file__='"'"'/tmp/pip-install-djviZf/patroni/setup.py'"'"';f=getattr(tokenize, '"'"'open'"'"', open)(__file__);code=f.read().replace('"'"'\r\n'"'"', '"'"'\n'"'"');f.close();exec(compile(code, __file__, '"'"'exec'"'"'))' egg_info --egg-base /tmp/pip-install-djviZf/patroni/pip-egg-info
            cwd: /tmp/pip-install-djviZf/patroni/
        Complete output (54 lines):
        /usr/lib64/python2.7/distutils/dist.py:267: UserWarning: Unknown distribution option: 'python_requires'
        warnings.warn(msg)
        warning: no previously-included files matching '*.pyc' found anywhere in distribution
        no previously-included directories found matching 'docs/build/'
        zip_safe flag not set; analyzing archive contents...
        
        Installed /tmp/pip-install-djviZf/patroni/.eggs/flake8-3.7.9-py2.7.egg
        Searching for typing
        Reading https://pypi.python.org/simple/typing/
        Best match: typing 3.7.4.1
        Downloading https://files.pythonhosted.org/packages/67/b0/b2ea2bd67bfb80ea5d12a5baa1d12bda002cab3b6c9b48f7708cd40c34bf/typing-3.7.4.1.tar.gz#sha256=91dfe6f3f706ee8cc32d38edbbf304e9b7583fb37108fef38229617f8b3eba23
        Processing typing-3.7.4.1.tar.gz
        Writing /tmp/easy_install-toLSfz/typing-3.7.4.1/setup.cfg
        Running typing-3.7.4.1/setup.py -q bdist_egg --dist-dir /tmp/easy_install-toLSfz/typing-3.7.4.1/egg-dist-tmp-87R1fJ
        zip_safe flag not set; analyzing archive contents...
        Moving typing-3.7.4.1-py2.7.egg to /tmp/pip-install-djviZf/patroni/.eggs
        
        Installed /tmp/pip-install-djviZf/patroni/.eggs/typing-3.7.4.1-py2.7.egg
        Searching for functools32
        Reading https://pypi.python.org/simple/functools32/
        Best match: functools32 3.2.3.post2
        Downloading https://files.pythonhosted.org/packages/c5/60/6ac26ad05857c601308d8fb9e87fa36d0ebf889423f47c3502ef034365db/functools32-3.2.3-2.tar.gz#sha256=f6253dfbe0538ad2e387bd8fdfd9293c925d63553f5813c4e587745416501e6d
        Traceback (most recent call last):
        File "<string>", line 1, in <module>
        File "/tmp/pip-install-djviZf/patroni/setup.py", line 166, in <module>
            setup_package(__version__)
        File "/tmp/pip-install-djviZf/patroni/setup.py", line 150, in setup_package
            entry_points={'console_scripts': CONSOLE_SCRIPTS},
        File "/usr/lib64/python2.7/distutils/core.py", line 112, in setup
            _setup_distribution = dist = klass(attrs)
        File "build/bdist.linux-x86_64/egg/setuptools/dist.py", line 269, in __init__
        File "build/bdist.linux-x86_64/egg/setuptools/dist.py", line 313, in fetch_build_eggs
        File "build/bdist.linux-x86_64/egg/pkg_resources/__init__.py", line 826, in resolve
        File "build/bdist.linux-x86_64/egg/pkg_resources/__init__.py", line 1092, in best_match
        File "build/bdist.linux-x86_64/egg/pkg_resources/__init__.py", line 1104, in obtain
        File "build/bdist.linux-x86_64/egg/setuptools/dist.py", line 380, in fetch_build_egg
        File "build/bdist.linux-x86_64/egg/setuptools/command/easy_install.py", line 628, in easy_install
        
        File "build/bdist.linux-x86_64/egg/setuptools/package_index.py", line 615, in fetch_distribution
        File "build/bdist.linux-x86_64/egg/setuptools/package_index.py", line 531, in download
        File "build/bdist.linux-x86_64/egg/setuptools/package_index.py", line 772, in _download_url
        File "build/bdist.linux-x86_64/egg/setuptools/package_index.py", line 778, in _attempt_download
        File "build/bdist.linux-x86_64/egg/setuptools/package_index.py", line 694, in _download_to
        File "/usr/lib64/python2.7/socket.py", line 380, in read
            data = self._sock.recv(left)
        File "/usr/lib64/python2.7/httplib.py", line 602, in read
            s = self.fp.read(amt)
        File "/usr/lib64/python2.7/socket.py", line 380, in read
            data = self._sock.recv(left)
        File "/usr/lib64/python2.7/ssl.py", line 759, in recv
            return self.read(buflen)
        File "/usr/lib64/python2.7/ssl.py", line 653, in read
            v = self._sslobj.read(len or 1024)
        socket.error: [Errno 104] Connection reset by peer
        ----------------------------------------
    ERROR: Command errored out with exit status 1: python setup.py egg_info Check the logs for full command output.
    ```

4. 安装patroni未使用[etcd]
    ```
    [root@node1 patroni]# patroni /etc/patroni/patroni_postgresql.yml 
    2020-04-02 09:41:43,438 INFO: Failed to import patroni.dcs.consul
    2020-04-02 09:41:43,439 INFO: Failed to import patroni.dcs.etcd
    2020-04-02 09:41:43,440 INFO: Failed to import patroni.dcs.exhibitor
    2020-04-02 09:41:43,441 INFO: Failed to import patroni.dcs.kubernetes
    2020-04-02 09:41:43,442 INFO: Failed to import patroni.dcs.zookeeper
    Traceback (most recent call last):
    File "/usr/bin/patroni", line 8, in <module>
        sys.exit(main())
    File "/usr/lib/python2.7/site-packages/patroni/__init__.py", line 196, in main
        return patroni_main()
    File "/usr/lib/python2.7/site-packages/patroni/__init__.py", line 157, in patroni_main
        patroni = Patroni()
    File "/usr/lib/python2.7/site-packages/patroni/__init__.py", line 28, in __init__
        self.dcs = get_dcs(self.config)
    File "/usr/lib/python2.7/site-packages/patroni/dcs/__init__.py", line 95, in get_dcs
        Available implementations: """ + ', '.join(available_implementations))
    patroni.exceptions.PatroniException: 'Can not find suitable configuration of distributed configuration store\nAvailable implementations: '
    [root@node1 patroni]#  vi patroni_postgresql.yml
    [root@node1 patroni]#  vi patroni_postgresql.yml
    [root@node1 patroni]# patroni /etc/patroni/patroni_postgresql.yml 
    2020-04-02 09:44:40,457 INFO: Failed to import patroni.dcs.consul
    2020-04-02 09:44:40,458 INFO: Failed to import patroni.dcs.etcd
    2020-04-02 09:44:40,459 INFO: Failed to import patroni.dcs.exhibitor
    2020-04-02 09:44:40,460 INFO: Failed to import patroni.dcs.kubernetes
    2020-04-02 09:44:40,461 INFO: Failed to import patroni.dcs.zookeeper
    Traceback (most recent call last):
    File "/usr/bin/patroni", line 8, in <module>
        sys.exit(main())
    File "/usr/lib/python2.7/site-packages/patroni/__init__.py", line 196, in main
        return patroni_main()
    File "/usr/lib/python2.7/site-packages/patroni/__init__.py", line 157, in patroni_main
        patroni = Patroni()
    File "/usr/lib/python2.7/site-packages/patroni/__init__.py", line 28, in __init__
        self.dcs = get_dcs(self.config)
    File "/usr/lib/python2.7/site-packages/patroni/dcs/__init__.py", line 95, in get_dcs
        Available implementations: """ + ', '.join(available_implementations))
    patroni.exceptions.PatroniException: 'Can not find suitable configuration of distributed configuration store\nAvailable implementations: '
    ```

5. data目录权限问题
    使用了自己创建的data目录，权限并不是postgres
    ```
    [beyondb@node2 patroni]$ patroni /etc/patroni/patroni_postgresql.yml 
    2020-04-02 14:37:20,829 INFO: Selected new etcd server http://192.168.11.29:2379
    2020-04-02 14:37:20,839 INFO: No PostgreSQL configuration items changed, nothing to reload.
    2020-04-02 14:37:20,873 INFO: establishing a new patroni connection to the postgres cluster
    Traceback (most recent call last):
    File "/usr/bin/patroni", line 8, in <module>
        sys.exit(main())
    File "/usr/lib/python2.7/site-packages/patroni/__init__.py", line 224, in main
        return patroni_main()
    File "/usr/lib/python2.7/site-packages/patroni/__init__.py", line 186, in patroni_main
        patroni = Patroni(conf)
    File "/usr/lib/python2.7/site-packages/patroni/__init__.py", line 35, in __init__
        self.postgresql = Postgresql(self.config['postgresql'])
    File "/usr/lib/python2.7/site-packages/patroni/postgresql/__init__.py", line 99, in __init__
        self.config.write_postgresql_conf()  # we are "joining" already running postgres
    File "/usr/lib/python2.7/site-packages/patroni/postgresql/config.py", line 418, in write_postgresql_conf
        with ConfigWriter(self._postgresql_conf) as f:
    File "/usr/lib/python2.7/site-packages/patroni/postgresql/config.py", line 231, in __enter__
        self._fd = open(self._filename, 'w')
    IOError: [Errno 13] Permission denied: '/opt/data/data/postgresql.conf'
    ```


6. 未安装psycopg2-binary
    ```
    [root@node4 ~]# pip install patroni[etcd]
    DEPRECATION: Python 2.7 reached the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 is no longer maintained. A future version of pip will drop support for Python 2.7. More details about Python 2 support in pip, can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support
    Collecting patroni[etcd]
    Downloading patroni-1.6.4.tar.gz (128 kB)
        |████████████████████████████████| 128 kB 509 kB/s 
        ERROR: Command errored out with exit status 1:
        command: /usr/bin/python -c 'import sys, setuptools, tokenize; sys.argv[0] = '"'"'/tmp/pip-install-MfUOQt/patroni/setup.py'"'"'; __file__='"'"'/tmp/pip-install-MfUOQt/patroni/setup.py'"'"';f=getattr(tokenize, '"'"'open'"'"', open)(__file__);code=f.read().replace('"'"'\r\n'"'"', '"'"'\n'"'"');f.close();exec(compile(code, __file__, '"'"'exec'"'"'))' egg_info --egg-base /tmp/pip-install-MfUOQt/patroni/pip-egg-info
            cwd: /tmp/pip-install-MfUOQt/patroni/
        Complete output (7 lines):
        Traceback (most recent call last):
        File "<string>", line 1, in <module>
        File "/tmp/pip-install-MfUOQt/patroni/setup.py", line 164, in <module>
            check_psycopg2()
        File "patroni/__init__.py", line 218, in check_psycopg2
            fatal('Patroni requires psycopg2>={0} or psycopg2-binary', min_psycopg2_str)
        TypeError: 'NoneType' object is not callable
        ----------------------------------------
    ERROR: Command errored out with exit status 1: python setup.py egg_info Check the logs for full command output.
    ```

7. 未安装configparser
    ```
    [root@node4 ~]# pip install patroni[etcd]
    DEPRECATION: Python 2.7 reached the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 is no longer maintained. A future version of pip will drop support for Python 2.7. More details about Python 2 support in pip, can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support
    Collecting patroni[etcd]
    Using cached patroni-1.6.4.tar.gz (128 kB)
        ERROR: Command errored out with exit status 1:
        command: /usr/bin/python -c 'import sys, setuptools, tokenize; sys.argv[0] = '"'"'/tmp/pip-install-0nRb0x/patroni/setup.py'"'"'; __file__='"'"'/tmp/pip-install-0nRb0x/patroni/setup.py'"'"';f=getattr(tokenize, '"'"'open'"'"', open)(__file__);code=f.read().replace('"'"'\r\n'"'"', '"'"'\n'"'"');f.close();exec(compile(code, __file__, '"'"'exec'"'"'))' egg_info --egg-base /tmp/pip-install-0nRb0x/patroni/pip-egg-info
            cwd: /tmp/pip-install-0nRb0x/patroni/
        Complete output (34 lines):
        /usr/lib64/python2.7/distutils/dist.py:267: UserWarning: Unknown distribution option: 'python_requires'
        warnings.warn(msg)
        warning: no previously-included files matching '*.pyc' found anywhere in distribution
        no previously-included directories found matching 'docs/build/'
        zip_safe flag not set; analyzing archive contents...
        
        Installed /tmp/pip-install-0nRb0x/patroni/flake8-3.7.9-py2.7.egg
        Searching for configparser
        Reading https://pypi.python.org/simple/configparser/
        Best match: configparser 5.0.0
        Downloading https://files.pythonhosted.org/packages/e5/7c/d4ccbcde76b4eea8cbd73b67b88c72578e8b4944d1270021596e80b13deb/configparser-5.0.0.tar.gz#sha256=2ca44140ee259b5e3d8aaf47c79c36a7ab0d5e94d70bd4105c03ede7a20ea5a1
        Processing configparser-5.0.0.tar.gz
        Writing /tmp/easy_install-1iMlqT/configparser-5.0.0/setup.cfg
        Running configparser-5.0.0/setup.py -q bdist_egg --dist-dir /tmp/easy_install-1iMlqT/configparser-5.0.0/egg-dist-tmp-TXESOl
        warning: install_lib: 'build/lib' does not exist -- no Python modules to install
        
        zip_safe flag not set; analyzing archive contents...
        
        Installed /tmp/pip-install-0nRb0x/patroni/UNKNOWN-0.0.0-py2.7.egg
        Traceback (most recent call last):
        File "<string>", line 1, in <module>
        File "/tmp/pip-install-0nRb0x/patroni/setup.py", line 166, in <module>
            setup_package(__version__)
        File "/tmp/pip-install-0nRb0x/patroni/setup.py", line 150, in setup_package
            entry_points={'console_scripts': CONSOLE_SCRIPTS},
        File "/usr/lib64/python2.7/distutils/core.py", line 112, in setup
            _setup_distribution = dist = klass(attrs)
        File "/usr/lib/python2.7/site-packages/setuptools/dist.py", line 265, in __init__
            self.fetch_build_eggs(attrs.pop('setup_requires'))
        File "/usr/lib/python2.7/site-packages/setuptools/dist.py", line 289, in fetch_build_eggs
            parse_requirements(requires), installer=self.fetch_build_egg
        File "/usr/lib/python2.7/site-packages/pkg_resources.py", line 626, in resolve
            raise DistributionNotFound(req)
        pkg_resources.DistributionNotFound: configparser
        ----------------------------------------
    ERROR: Command errored out with exit status 1: python setup.py egg_info Check the logs for full command output.
    ```

8. setoptool版本过低
    ```
    [root@node4 ~]# pip install patroni[etcd]
    DEPRECATION: Python 2.7 reached the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 is no longer maintained. A future version of pip will drop support for Python 2.7. More details about Python 2 support in pip, can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support
    Collecting patroni[etcd]
    Using cached patroni-1.6.4.tar.gz (128 kB)
        ERROR: Command errored out with exit status 1:
        command: /usr/bin/python -c 'import sys, setuptools, tokenize; sys.argv[0] = '"'"'/tmp/pip-install-63hnKu/patroni/setup.py'"'"'; __file__='"'"'/tmp/pip-install-63hnKu/patroni/setup.py'"'"';f=getattr(tokenize, '"'"'open'"'"', open)(__file__);code=f.read().replace('"'"'\r\n'"'"', '"'"'\n'"'"');f.close();exec(compile(code, __file__, '"'"'exec'"'"'))' egg_info --egg-base /tmp/pip-install-63hnKu/patroni/pip-egg-info
            cwd: /tmp/pip-install-63hnKu/patroni/
        Complete output (110 lines):
        /usr/lib64/python2.7/distutils/dist.py:267: UserWarning: Unknown distribution option: 'python_requires'
        warnings.warn(msg)
        warning: no previously-included files matching '*.pyc' found anywhere in distribution
        no previously-included directories found matching 'docs/build/'
        zip_safe flag not set; analyzing archive contents...
        
        Installed /tmp/pip-install-63hnKu/patroni/flake8-3.7.9-py2.7.egg
        Searching for mccabe>=0.6.0,<0.7.0
        Reading https://pypi.python.org/simple/mccabe/
        Best match: mccabe 0.6.1
        Downloading https://files.pythonhosted.org/packages/06/18/fa675aa501e11d6d6ca0ae73a101b2f3571a565e0f7d38e062eec18a91ee/mccabe-0.6.1.tar.gz#sha256=dd8d182285a0fe56bace7f45b5e7d1a6ebcbf524e8f3bd87eb0f125271b8831f
        Processing mccabe-0.6.1.tar.gz
        Writing /tmp/easy_install-nS4DIC/mccabe-0.6.1/setup.cfg
        Running mccabe-0.6.1/setup.py -q bdist_egg --dist-dir /tmp/easy_install-nS4DIC/mccabe-0.6.1/egg-dist-tmp-Igu1D9
        Searching for pytest-runner
        Reading https://pypi.python.org/simple/pytest-runner/
        Best match: pytest-runner 5.2
        Downloading https://files.pythonhosted.org/packages/5b/82/1462f86e6c3600f2471d5f552fcc31e39f17717023df4bab712b4a9db1b3/pytest-runner-5.2.tar.gz#sha256=96c7e73ead7b93e388c5d614770d2bae6526efd997757d3543fe17b557a0942b
        Processing pytest-runner-5.2.tar.gz
        Writing /tmp/easy_install-nS4DIC/mccabe-0.6.1/temp/easy_install-3UgYAL/pytest-runner-5.2/setup.cfg
        Running pytest-runner-5.2/setup.py -q bdist_egg --dist-dir /tmp/easy_install-nS4DIC/mccabe-0.6.1/temp/easy_install-3UgYAL/pytest-runner-5.2/egg-dist-tmp-HN1LRJ
        Searching for setuptools-scm>=1.15.0
        Reading https://pypi.python.org/simple/setuptools_scm/
        Best match: setuptools-scm 3.5.0
        Downloading https://files.pythonhosted.org/packages/0a/6c/06de5bb5c6a745a651b97d3bcc8cd1cc0f926688ff5c0489e3ff8400c22f/setuptools_scm-3.5.0-py2.7.egg#sha256=fa6511072840d7eaad3ea36cc8849f437fefd85b32499a7f6026cba16ebbf63a
        Processing setuptools_scm-3.5.0-py2.7.egg
        Moving setuptools_scm-3.5.0-py2.7.egg to /tmp/easy_install-nS4DIC/mccabe-0.6.1/temp/easy_install-3UgYAL/pytest-runner-5.2
        
        Installed /tmp/easy_install-nS4DIC/mccabe-0.6.1/temp/easy_install-3UgYAL/pytest-runner-5.2/setuptools_scm-3.5.0-py2.7.egg
        Traceback (most recent call last):
        File "<string>", line 1, in <module>
        File "/tmp/pip-install-63hnKu/patroni/setup.py", line 166, in <module>
            setup_package(__version__)
        File "/tmp/pip-install-63hnKu/patroni/setup.py", line 150, in setup_package
            entry_points={'console_scripts': CONSOLE_SCRIPTS},
        File "/usr/lib64/python2.7/distutils/core.py", line 112, in setup
            _setup_distribution = dist = klass(attrs)
        File "/usr/lib/python2.7/site-packages/setuptools/dist.py", line 265, in __init__
            self.fetch_build_eggs(attrs.pop('setup_requires'))
        File "/usr/lib/python2.7/site-packages/setuptools/dist.py", line 289, in fetch_build_eggs
            parse_requirements(requires), installer=self.fetch_build_egg
        File "/usr/lib/python2.7/site-packages/pkg_resources.py", line 618, in resolve
            dist = best[req.key] = env.best_match(req, self, installer)
        File "/usr/lib/python2.7/site-packages/pkg_resources.py", line 862, in best_match
            return self.obtain(req, installer) # try and download/install
        File "/usr/lib/python2.7/site-packages/pkg_resources.py", line 874, in obtain
            return installer(requirement)
        File "/usr/lib/python2.7/site-packages/setuptools/dist.py", line 339, in fetch_build_egg
            return cmd.easy_install(req)
        File "/usr/lib/python2.7/site-packages/setuptools/command/easy_install.py", line 623, in easy_install
            return self.install_item(spec, dist.location, tmpdir, deps)
        File "/usr/lib/python2.7/site-packages/setuptools/command/easy_install.py", line 653, in install_item
            dists = self.install_eggs(spec, download, tmpdir)
        File "/usr/lib/python2.7/site-packages/setuptools/command/easy_install.py", line 849, in install_eggs
            return self.build_and_install(setup_script, setup_base)
        File "/usr/lib/python2.7/site-packages/setuptools/command/easy_install.py", line 1130, in build_and_install
            self.run_setup(setup_script, setup_base, args)
        File "/usr/lib/python2.7/site-packages/setuptools/command/easy_install.py", line 1115, in run_setup
            run_setup(setup_script, args)
        File "/usr/lib/python2.7/site-packages/setuptools/sandbox.py", line 69, in run_setup
            lambda: execfile(
        File "/usr/lib/python2.7/site-packages/setuptools/sandbox.py", line 120, in run
            return func()
        File "/usr/lib/python2.7/site-packages/setuptools/sandbox.py", line 71, in <lambda>
            {'__file__':setup_script, '__name__':'__main__'}
        File "setup.py", line 58, in <module>
            class PyTest(Command):
        File "/usr/lib64/python2.7/distutils/core.py", line 112, in setup
            _setup_distribution = dist = klass(attrs)
        File "/usr/lib/python2.7/site-packages/setuptools/dist.py", line 265, in __init__
            self.fetch_build_eggs(attrs.pop('setup_requires'))
        File "/usr/lib/python2.7/site-packages/setuptools/dist.py", line 289, in fetch_build_eggs
            parse_requirements(requires), installer=self.fetch_build_egg
        File "/usr/lib/python2.7/site-packages/pkg_resources.py", line 618, in resolve
            dist = best[req.key] = env.best_match(req, self, installer)
        File "/usr/lib/python2.7/site-packages/pkg_resources.py", line 862, in best_match
            return self.obtain(req, installer) # try and download/install
        File "/usr/lib/python2.7/site-packages/pkg_resources.py", line 874, in obtain
            return installer(requirement)
        File "/usr/lib/python2.7/site-packages/setuptools/dist.py", line 339, in fetch_build_egg
            return cmd.easy_install(req)
        File "/usr/lib/python2.7/site-packages/setuptools/command/easy_install.py", line 623, in easy_install
            return self.install_item(spec, dist.location, tmpdir, deps)
        File "/usr/lib/python2.7/site-packages/setuptools/command/easy_install.py", line 653, in install_item
            dists = self.install_eggs(spec, download, tmpdir)
        File "/usr/lib/python2.7/site-packages/setuptools/command/easy_install.py", line 849, in install_eggs
            return self.build_and_install(setup_script, setup_base)
        File "/usr/lib/python2.7/site-packages/setuptools/command/easy_install.py", line 1130, in build_and_install
            self.run_setup(setup_script, setup_base, args)
        File "/usr/lib/python2.7/site-packages/setuptools/command/easy_install.py", line 1115, in run_setup
            run_setup(setup_script, args)
        File "/usr/lib/python2.7/site-packages/setuptools/sandbox.py", line 69, in run_setup
            lambda: execfile(
        File "/usr/lib/python2.7/site-packages/setuptools/sandbox.py", line 120, in run
            return func()
        File "/usr/lib/python2.7/site-packages/setuptools/sandbox.py", line 71, in <lambda>
            {'__file__':setup_script, '__name__':'__main__'}
        File "setup.py", line 21, in <module>
            AUTHOR_EMAIL = 'alexander.kukushkin@zalando.de, dmitrii.dolgov@zalando.de, alexk@hintbits.com'
        File "/usr/lib64/python2.7/distutils/core.py", line 112, in setup
            _setup_distribution = dist = klass(attrs)
        File "/usr/lib/python2.7/site-packages/setuptools/dist.py", line 269, in __init__
            _Distribution.__init__(self,attrs)
        File "/usr/lib64/python2.7/distutils/dist.py", line 287, in __init__
            self.finalize_options()
        File "/usr/lib/python2.7/site-packages/setuptools/dist.py", line 302, in finalize_options
            ep.load()(self, ep.name, value)
        File "/tmp/easy_install-nS4DIC/mccabe-0.6.1/temp/easy_install-3UgYAL/pytest-runner-5.2/setuptools_scm-3.5.0-py2.7.egg/setuptools_scm/integration.py", line 9, in version_keyword
        File "/tmp/easy_install-nS4DIC/mccabe-0.6.1/temp/easy_install-3UgYAL/pytest-runner-5.2/setuptools_scm-3.5.0-py2.7.egg/setuptools_scm/version.py", line 66, in _warn_if_setuptools_outdated
        setuptools_scm.version.SetuptoolsOutdatedWarning: your setuptools is too old (<12)
        ----------------------------------------
    ERROR: Command errored out with exit status 1: python setup.py egg_info Check the logs for full command output.
    ```

9. putil出问题
    ```
    [root@node4 ~]# pip install psutil
    DEPRECATION: Python 2.7 reached the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 is no longer maintained. A future version of pip will drop support for Python 2.7. More details about Python 2 support in pip, can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support
    Collecting psutil
    Using cached psutil-5.7.0.tar.gz (449 kB)
    Building wheels for collected packages: psutil
    Building wheel for psutil (setup.py) ... error
    ERROR: Command errored out with exit status 1:
    command: /usr/bin/python -u -c 'import sys, setuptools, tokenize; sys.argv[0] = '"'"'/tmp/pip-install-Id3J0K/psutil/setup.py'"'"'; __file__='"'"'/tmp/pip-install-Id3J0K/psutil/setup.py'"'"';f=getattr(tokenize, '"'"'open'"'"', open)(__file__);code=f.read().replace('"'"'\r\n'"'"', '"'"'\n'"'"');f.close();exec(compile(code, __file__, '"'"'exec'"'"'))' bdist_wheel -d /tmp/pip-wheel-_63oKZ
        cwd: /tmp/pip-install-Id3J0K/psutil/
    Complete output (44 lines):
    running bdist_wheel
    running build
    running build_py
    creating build
    creating build/lib.linux-x86_64-2.7
    creating build/lib.linux-x86_64-2.7/psutil
    copying psutil/_pswindows.py -> build/lib.linux-x86_64-2.7/psutil
    copying psutil/_psbsd.py -> build/lib.linux-x86_64-2.7/psutil
    copying psutil/_compat.py -> build/lib.linux-x86_64-2.7/psutil
    copying psutil/_common.py -> build/lib.linux-x86_64-2.7/psutil
    copying psutil/_psposix.py -> build/lib.linux-x86_64-2.7/psutil
    copying psutil/__init__.py -> build/lib.linux-x86_64-2.7/psutil
    copying psutil/_psaix.py -> build/lib.linux-x86_64-2.7/psutil
    copying psutil/_pslinux.py -> build/lib.linux-x86_64-2.7/psutil
    copying psutil/_pssunos.py -> build/lib.linux-x86_64-2.7/psutil
    copying psutil/_psosx.py -> build/lib.linux-x86_64-2.7/psutil
    creating build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/test_aix.py -> build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/test_system.py -> build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/test_bsd.py -> build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/test_connections.py -> build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/test_unicode.py -> build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/test_posix.py -> build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/test_windows.py -> build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/test_misc.py -> build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/test_sunos.py -> build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/__init__.py -> build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/test_memory_leaks.py -> build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/test_osx.py -> build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/test_process.py -> build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/test_linux.py -> build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/__main__.py -> build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/test_contracts.py -> build/lib.linux-x86_64-2.7/psutil/tests
    copying psutil/tests/runner.py -> build/lib.linux-x86_64-2.7/psutil/tests
    running build_ext
    building 'psutil._psutil_linux' extension
    creating build/temp.linux-x86_64-2.7
    creating build/temp.linux-x86_64-2.7/psutil
    gcc -pthread -fno-strict-aliasing -O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -D_GNU_SOURCE -fPIC -fwrapv -DNDEBUG -O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -D_GNU_SOURCE -fPIC -fwrapv -fPIC -DPSUTIL_POSIX=1 -DPSUTIL_SIZEOF_PID_T=4 -DPSUTIL_VERSION=570 -DPSUTIL_LINUX=1 -I/usr/include/python2.7 -c psutil/_psutil_common.c -o build/temp.linux-x86_64-2.7/psutil/_psutil_common.o
    psutil/_psutil_common.c:9:20: 致命错误：Python.h：没有那个文件或目录
    #include <Python.h>
                        ^
    编译中断。
    error: command 'gcc' failed with exit status 1
    ----------------------------------------
    ERROR: Failed building wheel for psutil

    ```
    需要安装yum install python-devel


