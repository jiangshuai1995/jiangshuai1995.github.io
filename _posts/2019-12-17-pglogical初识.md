---
layout:     post
title:      pglogical初识
subtitle:   使用pglogcial进行数据的上游聚合
date:       2019-12-17
author:     JS
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - postgres
---

## 背景

某系统需要进行上游数据的聚合功能，可以使用pg原生的逻辑复制或者wal日志decode等方法。

pglogical插件是对逻辑复制的扩展和封装，搭建起来更容易，所以选择对pglogical进行调研。

## 测试过程

#### 准备工作

|IP|Port|Database|Role|
|--|--|--|--|
|192.168.5.111|5432|postgres|subscriber|
|192.168.5.112|5432|postgres|provider1|
|192.168.5.113|5432|postgres|provider2|

1. 安装PG10及相关依赖
    ```bash
    yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
    yum install postgresql10
    yum install postgresql10-server postgresql10-contrib
    ```

2. 安装pglogical插件
    ```bash
    curl https://access.2ndquadrant.com/api/repository/dl/default/release/10/rpm | bash
    yum install postgresql10-pglogical
    ```

3. 初始化数据库
    ```
    /usr/pgsql-10/bin/postgresql-10-setup initdb
    ```
    默认创建的数据库目录在/var/lib/pgsql/10/data目录下

4. 修改pg_hba.conf文件

    添加 host   all     all     192.168.0.0/16      trust

5. 修改postgresql.conf

    ```
    wal_level = 'logical'
    max_worker_processes = 10   # one per database needed on provider node
                                # one per node needed on subscriber node
    max_replication_slots = 10  # one per node needed on provider node
    max_wal_senders = 10        # one per node needed on provider node
    shared_preload_libraries = 'pglogical'
    track_commit_timestamp = on # needed for last/first update wins conflict resolution
                                # property available in PostgreSQL 9.5+
    ```
6. 启动数据库

    ```
    systemctl start postgresql-10
    ```

#### 测试过程

1. 创建表
    
    三个数据库均执行
    ```sql
    create table test_logical(id int  primary key,name text);
    ```
2. 创建扩展

    三个数据库均执行
    ```sql
    CREATE EXTENSION pglogical;
    ```
2. 创建节点


    __provider1__
    ```
    postgres=# SELECT pglogical.create_node(node_name:='provider1',dsn:='host=192.168.5.112 port=5432 dbname=postgres');

    create_node 
    -------------
    2976894835
    (1 行记录)

    ```

    __provider2__
    ```
    postgres=# SELECT pglogical.create_node(node_name:='provider2',dsn:='host=192.168.5.113 port=5432 dbname=postgres');

    create_node 
    -------------
    1828187473
    (1 行记录)
    ```

    __subscriber__
    ```
    postgres=# SELECT pglogical.create_node(node_name:='subscriber', dsn:='host=192.168.5.111 port=5432 dbname=postgres');

    create_node 
    -------------
    2941155235
    (1 行记录)

    ```

4. 创建复制策略

    本例中是对public模式下所有表都添加到默认复制集中，具体参见文档

    __provider1__
    ```
    postgres=# SELECT pglogical.replication_set_add_all_tables('default', ARRAY['public']);

    replication_set_add_all_tables 
    --------------------------------
    t
    (1 行记录)
    ```
    __provider2__
    ```
    postgres=# SELECT pglogical.replication_set_add_all_tables('default', ARRAY['public']);

    replication_set_add_all_tables 
    --------------------------------
    t
    (1 行记录)
    ```

5. 创建订阅

    __subscriber__
    ```
    postgres=# SELECT pglogical.create_subscription(subscription_name:='subscription1',provider_dsn:='host=192.168.5.112 port=5432 dbname=postgres');

    create_subscription 
    ---------------------
        1763399739
    (1 行记录)

    postgres=# SELECT pglogical.create_subscription(subscription_name:='subscription2',provider_dsn:='host=192.168.5.113 port=5432 dbname=postgres');

    create_subscription 
    ---------------------
        1871150101

    ```

6. 在发布者端插入数据

    __provider1__
    ```
    postgres=# insert into test_logical values(1,'shijiangzhuang');
    INSERT 0 1
    postgres=# insert into test_logical values(2,'tangshan');
    INSERT 0 1
    postgres=# insert into test_logical values(3,'baoding');
    INSERT 0 1
    ```

    __provider2__
    ```
    postgres=# insert into test_logical values(4,'zhangjiakou');
    INSERT 0 1
    postgres=# insert into test_logical values(5,'xingtai');
    INSERT 0 1
    postgres=# insert into test_logical values(6,'hengshui');
    INSERT 0 1
    ```

7. 在订阅者端查看数据

    ```
    postgres=# select * from test_logical;
    id |      name      
    ----+----------------
    1 | shijiangzhuang
    2 | tangshan
    3 | baoding
    4 | zhangjiakou
    5 | xingtai
    6 | hengshui
    (6 行记录)
    ```

## 结论

pglogical插件可以便捷的实现逻辑复制的过程，粒度为表级别，对于简单的上游数据集合可以满足该需求。

将发布者与订阅者用IP绑定也方便使用VIP时进行切换。

对于真实应用场景，例如出现主键冲突，自增列冲突，或者random函数返回值不同的问题，还需要确定场景后测试。

