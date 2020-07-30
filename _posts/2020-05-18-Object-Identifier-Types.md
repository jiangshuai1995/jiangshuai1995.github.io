---
layout:    post
title:     Object Identifier Types
subtitle:   Oid的别名类型
date:       2020-05-18
author:     Jiangs
header-img: img/view.jpg
catalog:    true
tags:
     - Postgresql
---

## 背景

```
select relnamespace::regnamespace, indrelid::regclass, indexrelid::regclass   
from pg_index t1,pg_class t2   
where t1.indexrelid =t2.oid   
and not t1.indisvalid; 
```
本文的记录源于这条SQL,由于知道::是类型转换操作符，所以对于::后的例如regnamespace的类型并不了解，所以查阅了资料在本文进行记录

## 关于OID

1.  OID是一个四字节无符号整型，对于一些系统表均用oid作为主键，对于系统对象也常用oid进行标识。
2.  OID查询需要使用select oid，* from table1; 默认是一个隐藏字段



我们对照pg_index观察一下
```
postgres=# select * from pg_index limit 1;
-[ RECORD 1 ]--+----------
indexrelid     | 2831
indrelid       | 2830
indnatts       | 2
indnkeyatts    | 2
indisunique    | t
indisprimary   | t
indisexclusion | f
indimmediate   | t
indisclustered | f
indisvalid     | t
indcheckxmin   | f
indisready     | t
indislive      | t
indisreplident | f
indkey         | 1 2
indcollation   | 0 0
indclass       | 1981 1978
indoption      | 0 0
indexprs       | 
indpred        | 
```

```
indexrelid	oid	pg_class.oid	The OID of the pg_class entry for this index
indrelid	oid	pg_class.oid	The OID of the pg_class entry for the table this index is for
```



Name|	References	|Description	|Value Example
-|-|-|-
|oid|any numeric| object identifier	|564182|
|regproc	|pg_proc	|function name	|sum
regprocedure	|pg_proc	|function with argument types|	sum(int4)
regoper	|pg_operator	|operator name	|+
regoperator	|pg_operator	|operator with argument types	|*(integer,integer) or -(NONE,integer)
regclass	|pg_class	|relation name	|pg_type
regtype	|pg_type	|data type name	|integer
regrole	|pg_authid	|role name	|smithee
regnamespace	|pg_namespace	|namespace name	|pg_catalog
regconfig	|pg_ts_config	|text search configuration	|english
regdictionary	|pg_ts_dict	|text search dictionary	|simple


```
SELECT * FROM pg_attribute WHERE attrelid = 'mytable'::regclass;

SELECT * FROM pg_attribute
  WHERE attrelid = (SELECT oid FROM pg_class WHERE relname = 'mytable');
```

## 结论 

总的来看oid的类型别名更像是做了一次映射，将可读的名称与OID进行映射，本质上还是OID类型，在使用过程中如果未映射正确类型，例如class的OID映射了regproc，则OID还会以数字的形式显示，并且不会报错。