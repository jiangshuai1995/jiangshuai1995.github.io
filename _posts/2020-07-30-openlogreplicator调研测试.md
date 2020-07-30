---
layout:    post
title:     openlogreplicator调研测试
subtitle:   openlogreplicator
date:       2020-07-30
author:     Jiangs
header-img: img/view.jpg
catalog:    true
tags:
     - Postgresql
---

## 背景

在调研ORACLE的CDC工具时发现该项目，该项目由C++编写，近期版本更新较快，roadmap中有对接redis等内容。所以对该项目进行调研测试。 开源项目，协议为GPL v3。


 log-based replication (CDC, streaming) from Oracle Database to Kafka with minimal impact on the source database. 

 github地址如下

 ```
 https://github.com/bersler/OpenLogReplicator
 ```

## 基础环境

- CentOS 7.3
- kafka
- Oracle 11g

## 测试release

```
wget https://github.com/bersler/OpenLogReplicator/releases/tag/0.6.9
```

下载release版本后

根据参考文档编写了json文件，其中遇到了要求json版本为0.6.5的问题

但是下载的明明是0.6.9版本

```
{
"version": "0.6.5",
  "sources": [
    {
      "type": "ORACLE",
      "alias": "S1",
      "memory-min-mb": 64,
      "memory-max-mb": 1024,
      "name": "test1",
      "user": "abc",
      "password": "123456",
      "server": "//192.168.11.25:1521/ORCL",
      "tables": [
        {"table": "abc.jiang"}]
    }
  ],
  "targets": [
    {
      "type": "KAFKA",
      "alias": "T2",
      "format": {"stream": "JSON", "topic": "test1"},
      "brokers": "localhost:9092",
      "source": "S1"
    }
  ]
}
```

总的来说配置文件相对简洁，将tables中包含表的DML操作输出到topic test1中。

```
[root@master ~]# export ORACLE_SID=orcl
[root@master ~]# export ORACLE_BASE=/opt/oracle/base/
[root@master ~]# export ORACLE_HOME=/opt/oracle/base/db
[root@master ~]# export LD_LIBRARY_PATH=/opt/oracle/base/db/lib
[root@master ~]# ./OpenLogReplicator  OpenLogReplicator
Open Log Replicator v.0.6.5 (C) 2018-2020 by Adam Leszczynski (aleszczynski@bersler.com), see LICENSE file for licensing information
Adding source: test1
- loading character mapping for US7ASCII
- loading character mapping for AL16UTF16
- reading table schema for: abc.jiang
  * found: ABC.JIANG (OBJD: 13013, OBJN: 13013)
  * total: 1 tables
Adding target: T2
Starting thread: Oracle Analyser for: test1 in online mode
Starting thread: Kafka writer for: localhost:9092 topic: test1
Processing log: (1, 412642, 416764, 13, "/opt/oracle/base/flash_recovery_area/ORCL/onlinelog/o1_mf_1_hkj0qjsv_.log")
Processing log: (2, 416764, 0, 14, "/opt/oracle/base/flash_recovery_area/ORCL/onlinelog/o1_mf_2_hkj0ql00_.log")
```

配置环境变量后执行，无报错，但是kafka中无数据，进程一直停在这里,在执行DML语句后会出现KAFKA写入错误导致程序退出


随后排查C++ librdkafka++的问题发现，yum安装librdkafka-0.11.4-1.el7.x86_64时会安装librdkafka++.so

删除该so文件并安装github上的高版本

```
git clone https://github.com/edenhill/librdkafka
cd librdkafka
./configure
make
make install
```

随后启动程序后，无报错，kafka无数据，执行DML语句不会报错退出。无法使用。

## 编译安装


1.  yum安装依赖库

    ```
    yum -y install make gcc gcc-c++ git unzip libasan libaio-devel libnsl
    ```

    说明： 文档要求安装libnsl，但是本次未安装成功，并未发现影响。



2.  链接Oracle client libraries

    ```
    mkdir -p /opt/instantclient_11_2
    cd /opt/instantclient_11_2
    ln -s /opt/oracle/base/db/lib/libclntsh.so.11.1 libocci.so
    ln -s /opt/oracle/base/db/lib/libocci.so.11.1 libclntsh.so
    ln -s /opt/oracle/base/db/lib/libnnz11.so.11.1 libnnz11.so
    ```
3.  下载rapidjson
    ```
    cd /opt
    git clone https://github.com/Tencent/rapidjson/
    ```

4.  下载librdkafka
    ```
    cd /opt
    git clone https://github.com/edenhill/librdkafka
    cd /opt/librdkafka
    ./configure
    make
    make install
    ```

    说明： 如果yum源安装过librdkafka，可以移除或者手动删除libkafka++.so


5.  下载源码

    ```
    git clone https://github.com/bersler/OpenLogReplicator
    ```

6. 编译debug版本

    ```
    cd /opt/OpenLogReplicator/Debug
    make
    ```

    编译过后会有可执行程序OpenLogReplicator出现

7. 配置json文件

    ```
    vi /opt/OpenLogReplicator/Debug/OpenLogReplicator.json
    ```

    内容如下：
    ```
    {
    "version": "0.6.11",
        "sources": [
            {
            "type": "ORACLE",
            "alias": "S1",
            "memory-min-mb": 64,
            "memory-max-mb": 1024,
            "name": "test1",
            "user": "abc",
            "password": "123456",
            "server": "//192.168.11.25:1521/ORCL",
            "tables": [
                {"table": "abc.jiang"}]
            }
        ],
        "targets": [
            {
            "type": "KAFKA",
            "alias": "T2",
            "format": {"stream": "JSON", "topic": "test1"},
            "brokers": "localhost:9092",
            "source": "S1"
            }
        ]
    }
    ```
8. 配置环境变量
    ```
    export ORACLE_SID=orcl
    export ORACLE_BASE=/opt/oracle/base/
    export ORACLE_HOME=/opt/oracle/base/db
    export LD_LIBRARY_PATH=/opt/oracle/base/db/lib
    ```

9. 修改数据库配置

    在sqlpluns中执行

    本例中使用的abc用户，所有要对abc用户赋予权限
    ```
    SELECT SUPPLEMENTAL_LOG_DATA_MIN, LOG_MODE FROM V$DATABASE;
    SHUTDOWN IMMEDIATE;
    STARTUP MOUNT;
    ALTER DATABASE ARCHIVELOG;
    ALTER DATABASE OPEN;
    ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
    ALTER SYSTEM ARCHIVE LOG CURRENT;
    SELECT SUPPLEMENTAL_LOG_DATA_MIN, LOG_MODE FROM V$DATABASE;
    SELECT FORCE_LOGGING FROM V$DATABASE;
    ALTER DATABASE FORCE LOGGING;
    ALTER SYSTEM ARCHIVE LOG CURRENT;
    SELECT FORCE_LOGGING FROM V$DATABASE;


    GRANT SELECT ON SYS.CCOL$ TO abc;
    GRANT SELECT ON SYS.CDEF$ TO abc;
    GRANT SELECT ON SYS.COL$ TO abc;
    GRANT SELECT ON SYS.CON$ TO abc;
    GRANT SELECT ON SYS.OBJ$ TO abc;
    GRANT SELECT ON SYS.TAB$ TO abc;
    GRANT SELECT ON SYS.TABCOMPART$ TO abc;
    GRANT SELECT ON SYS.TABPART$ TO abc;
    GRANT SELECT ON SYS.TABSUBPART$ TO abc;
    GRANT SELECT ON SYS.USER$ TO abc;
    GRANT SELECT ON SYS.V_$ARCHIVED_LOG TO abc;
    GRANT SELECT ON SYS.V_$DATABASE TO abc;
    GRANT SELECT ON SYS.V_$DATABASE_INCARNATION TO abc;
    GRANT SELECT ON SYS.V_$LOG TO abc;
    GRANT SELECT ON SYS.V_$LOGFILE TO abc;
    GRANT SELECT ON SYS.V_$PARAMETER TO abc;
    GRANT SELECT ON SYS.V_$TRANSPORTABLE_PLATFORM TO abc;
    ```



10. 启动测试
    ```
    ./OpenLogReplicator
    ```

    根据实际安装路径启动

11. 启动consumer

    ```
    ./kafka-console-consumer.sh --bootstrap-server master:9092  --topic test1
    ```

    根据实际安装路径启动，topic为json文件中指定的

12. 进行DML操作

    对abc.jiang进行操作后，consumer中有数据出现
    ```
        输出json数据

        由于各种原因未能记录在本文档中
    ```


## 总结

OpenLogReplicator工具使用debug版本可以输出结果，发布的版本问题无法跟踪，总的来说该工具的优势应该是C++编写，可能在性能上较好（未进行测试），对于C++程序员方便修改或调试。不建议适用该工具作为ORACLE CDC工具。

## 参考资料

- [github repo](https://github.com/bersler/OpenLogReplicator)
- [OpenLogReplicator_doc](https://www.bersler.com/)