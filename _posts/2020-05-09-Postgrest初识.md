---
layout:    post
title:     Postgrest初识
subtitle:   "PG RESTful API"
date:       2020-05-09
author:     Jiangs
header-img: img/view.jpg
catalog:    true
tags:
     - Postgresql
     - RESTful
---

## 背景

之前已经介绍过prest,本文介绍另一个更常用的PG rest服务 : **Postgrest**


## 使用

1. 下载

   https://github.com/PostgREST/postgrest/releases

   本文中用的版本是V7.0.0


2. 参数配置

    首先看模板配置文件

    ```
        Example Config File:
        db-uri = "postgres://user:pass@localhost:5432/dbname"
        db-schema = "public" # this schema gets added to the search_path of every request
        db-anon-role = "postgres"
        db-pool = 10
        db-pool-timeout = 10

        server-host = "!4"
        server-port = 3000

        ## unix socket location
        ## if specified it takes precedence over server-port
        # server-unix-socket = "/tmp/pgrst.sock"
        ## unix socket file mode
        ## when none is provided, 660 is applied by default
        # server-unix-socket-mode = "660"

        ## base url for swagger output
        # openapi-server-proxy-uri = ""

        ## choose a secret, JSON Web Key (or set) to enable JWT auth
        ## (use "@filename" to load from separate file)
        # jwt-secret = "secret_with_at_least_32_characters"
        # secret-is-base64 = false
        # jwt-aud = "your_audience_claim"

        ## limit rows in response
        # max-rows = 1000

        ## stored proc to exec immediately after auth
        # pre-request = "stored_proc_name"

        ## jspath to the role claim key
        # role-claim-key = ".role"

        ## extra schemas to add to the search_path of every request
        # db-extra-search-path = "extensions, util"

        ## stored proc that overrides the root "/" spec
        ## it must be inside the db-schema
        # root-spec = "stored_proc_name"

        ## content types to produce raw output
        # raw-media-types="image/png, image/jpg"

    ```

    主要配置参数

    ```
        db-uri = "postgres://beyondb:123456@localhost:1328/postgres"  # 数据库连接信息
        db-schema = "public"                                          # 数据库模式
        db-anon-role = "beyondb"                                      # 数据库角色
    ```

2. 运行

   ```
        [root@localhost ~]# ./unisdbrest unisdb.conf 

        Attempting to connect to the database...
        Listening on port 3000
        Connection successful
   ```

3. 测试

    使用curl工具进行测试

    ```BASH
    [root@localhost ~]# curl http://127.0.0.1:3000/todos

        {"hint":null,"details":null,"code":"42P01","message":"关系 \"public.todos\" 不存在"}

    [root@localhost ~]# curl http://127.0.0.1:3000/city;

        [{"id":1,"name":"beijing"}, 
        {"id":2,"name":"shanghai"}, 
        {"id":3,"name":"tianjin"}, 
        {"id":4,"name":"chongqing"}]
    
    [root@localhost ~]# curl http://127.0.0.1:3000/city?select=name;

        [{"name":"beijing"}, 
        {"name":"shanghai"}, 
        {"name":"tianjin"}, 
        {"name":"chongqing"}]

    [root@localhost ~]# curl http://127.0.0.1:3000/city?select=cityname:name;

        [{"cityname":"beijing"}, 
        {"cityname":"shanghai"}, 
        {"cityname":"tianjin"}, 
        {"cityname":"chongqing"}]

    [root@localhost ~]# curl http://127.0.0.1:3000/city?select=cityid:id::text;

        [{"cityid":"1"}, 
        {"cityid":"2"}, 
        {"cityid":"3"}, 
        {"cityid":"4"}]

    [root@localhost ~]# curl  -H "Range-Unit: items" -H "Range: 0-1"  http://127.0.0.1:3000/city
        
        [{"id":1,"name":"beijing"}, 
         {"id":2,"name":"shanghai"}]

    [root@localhost ~]# curl  -X HEAD -v "http://192.168.146.134:3000/city"   -H "Prefer: count=exact"

        * About to connect() to 192.168.146.134 port 3000 (#0)
        *   Trying 192.168.146.134...
        * Connected to 192.168.146.134 (192.168.146.134) port 3000 (#0)
        > HEAD /city HTTP/1.1
        > User-Agent: curl/7.29.0
        > Host: 192.168.146.134:3000
        > Accept: */*
        > Prefer: count=exact
        > 
        < HTTP/1.1 200 OK
        < Date: Mon, 11 May 2020 05:32:47 GMT
        < Server: postgrest/7.0.0 (2b61a63)
        < Content-Type: application/json; charset=utf-8
        < Content-Range: 0-3/4
        < Content-Location: /city
        * no chunk, no close, no size. Assume close to signal end
        < 
        * Closing connection 0



    [root@localhost ~]# curl  "http://192.168.146.134:3000/rpc/add_them?a=2&b=2"

        [{"add_them":4}]

    ## insert
    [root@localhost ~]# curl -d "id=6&name=guangzhou"  -X POST "http://192.168.146.134:3000/city" 


    ## delete
    [root@localhost ~]# curl  -X DELETE "http://192.168.146.134:3000/city?id=eq.6"  -H "Prefer: return=representation"

        [{"id":6,"name":"guangzhou"}, 
        {"id":6,"name":"guangzhou"}]

    ## UPDATE 必须有主键,根据主键插入，没有主键无法出现冲突就无法完成更新操作

    [root@localhost ~]# curl  -d "id=6&name=wuhan" -X POST "http://192.168.146.134:3000/city"  -H "Prefer: resolution=merge-duplicates"

    ```

## 其他问题


After creating a table or changing its primary key, you must refresh PostgREST schema cache for UPSERT to work properly. To learn how to refresh the cache see Schema Reloading.

要刷新缓存而不重新启动PostgREST服务器，请向服务器进程发送SIGUSR1信号：

killall -SIGUSR1 postgrest



## 文档及手册

```
Abbreviation	In PostgreSQL	Meaning
eq 	=	equals
gt	>	greater than
gte	>=	greater than or equal
lt	<	less than
lte	<=	less than or equal
neq	<> or !=	not equal
like	LIKE	LIKE operator (use * in place of %)
ilike	ILIKE	ILIKE operator (use * in place of %)
in	IN	one of a list of values, e.g. ?a=in.(1,2,3) – also supports commas in quoted strings like ?a=in.("hi,there","yes,you")
is	IS	checking for exact equality (null,true,false)
fts	@@	Full-Text Search using to_tsquery
plfts	@@	Full-Text Search using plainto_tsquery
phfts	@@	Full-Text Search using phraseto_tsquery
wfts	@@	Full-Text Search using websearch_to_tsquery
cs	@>	contains e.g. ?tags=cs.{example, new}
cd	<@	contained in e.g. ?values=cd.{1,2,3}
ov	&&	overlap (have points in common), e.g. ?period=ov.[2017-01-01,2017-06-30] – also supports array types, use curly braces instead of square brackets e.g. :code: ?arr=ov.{1,3}
sl	<<	strictly left of, e.g. ?range=sl.(1,10)
sr	>>	strictly right of
nxr	&<	does not extend to the right of, e.g. ?range=nxr.(1,10)
nxl	&>	does not extend to the left of
adj	-|-	is adjacent to, e.g. ?range=adj.(1,10)
not	NOT	negates another operator, see below
```