---
layout: post
title: "PostgreSQL 即学即用 读书笔记"
date: 2016-03-23
categories: Note
---
## PostgreSQL 即学即用读书笔记

### 第一章 基础知识
#### PostgreSQL 数据库对象

- 服务：PostgreSQL作为一种服务(守护进程)来安装
- database：每个 PostgreSQL 服务可以有多个独立的 database
- schema：database 下一层的逻辑结构就是 schema。PostgreSQL 默认会为新建的 database 创建
一个名为 public 的 schema。
- catalog：这是系统级的 schema，用于存储系统函数和系统元数据。database 默认会有 pg_catalog
 和 information_schema 这两个 catalog。
- 变量：PostgreSQL 统一配置机制的一部分，可以在多个级别进行设置的各种选项，包括服务级、database
级以及其他级别。
- 扩展包：PostgreSQL 9.1 起引入了扩展包支持。
- 表：在 PostgreSQL 中，表首先属于某个 schema，而 schema 又属于某个 database，构成一种三
级存储结构。

  PostgreSQL 的表还支持两种功能:
  - 表继承：一张表可以有父表和子表。
  - 创建一张表的同时，系统会自动为此表创建一种对应的自定义数据结构
- 外部表和外部数据封装器：通过它们可以直接在本地数据库中访问来自外部数据源的数据。
- 表空间：用于存储数据的物理空间。表空间和 schema 无耦合关系
- 视图
- 函数
- 内置编程语言：PostgreSQL 默认支持 SQL、PL/pgSQL 和 C 语言。
- 运算符：本质是符号化的已命名函数
- 数据类型（或仅仅类型）
- 数据类型转换器：PostgreSQL 支持用户自定义转换器
- 序列：序列控制 serial 数据类型的自动递增。
- 行或记录：在 PostgreSQL 中 “行” 或 “记录”可以脱离表而独立存在。
- 触发器
- 规则：能够将一种动作替换为另一种动作的机制。PostgreSQL 靠此实现视图。

### 第二章 数据库管理
#### 配置文件
PostgreSQL 主要有三个配置文件：
- postgresql.conf:

    包含一些通用设置如内存分配，新建 database 的默认存储位置，ip 地址，日志位置等。
    9.4 引入了 postgresql.auto.conf，任何时候执行 ALTER SYSTEM SQL 命令时，都会创建或重写
  该文件。该文件中的设置会代替 postgresql.conf 中的设置。
- pg_hba.conf:

  用于控制访问安全，管理客户端对 PostgreSQL 服务器的访问权限。

- pg_ident.conf：

  pg_hba.conf 的权限控制信息中的身份验证模式如果指定为 ident 方式，则用户连接时系统会尝试访问 pg_ident 文件并基于文件内容将当前执行登录操作的操作系统用户映射为一个 PostgreSQL 数据库内部用户的身份来登录。

重新加载配置文件:
- `pg_ctl reload -D your_data_directory_here`
- `service postgresql-9.x reload`
- 以超级用户权限登录到任一一个 database 并执行 `SELECT pg_reload_conf()`

#### 连接管理
终止连接三部曲：
- 查出活动链接列表及其进程 ID

  `SELECT * FROM pg_stat_activity;`
- 取消连接上的活动查询
  `SELECT pg_cancel_backend(pid)` 该操作本身不会终止连接本身。
- 终止该连接
  `SELECT pg_terminate_backend(pid)`

#### 角色
##### 创建可登录角色
`CREATE ROLE leo LOGIN PASSWORD 'king' CREATEDB VALID UNTIL 'infinity'`

VALID 行可选，用于给所创建的角色的权限设定有效期，默认 'infinity'，即永不过期。

`CREATE ROLE regina LOGIN PASSWORD 'queen' SUPERUSER VALID UNTIL '2020-1-1 00:00';`

##### 创建组角色
组角色一般用于将一组权限汇聚成一个集合以便于将这组权限批量授予别的普通角色。

`CREATE ROLE royalty INHERIT;` INHERIT 表示组角色 royalty 的任何一个成员角色将自动继承
其除 “超级用户权限” 外的所有权限。

`GRANT royalty TO leo;` 将组角色的权限授予其成员角色。

可以通过 `SET ROLE role;` 来临时获取组角色权限（会话期间有效）。拥有 'SUPERUSER' 权限的用户还能执行 `SET SESSION AUTHORIZATION some_role;` `SET ROLE role;` 只更改 'current_user' 而 'SET SESSION AUTHORIZATION some_role;' 还会更改 'session_user'。

##### 创建 database
`CREATE DATABASE db_name;` 该命令会以 'template1' 库为模板生成一份副本并将此副本作为新 database。每个 database 的属主就是执行此 SQL 命令的角色。只有用 CREATEDB 权限的人才能创建新 database。

###### 模板数据库
PostgreSQL 默认自带 'template0' 和 'template1' 两个模板数据库。

基本语法：

`CREATE DATABASE my_db TEMPLATE my_template_db;`

将某个数据库标记为模板数据库

`UPDATE pg_database SET datistemplate = True WHERE datname = 'mydb';`

###### schema 的使用
`CREATE SCHEMA my_extensions;`

将新创建的 schema 加入 search_path：

`ALTER DATABASE mydb SET search_path='"$user", public, my_extensions';`

#### 权限管理

PostgreSQL 在安装阶段会默认创建一个超级用户角色 'postgres' 以及一个同名的 database。(Mac 下用 `brew` 安装的话，超级用户与你的登录账户同名。)

database 创建三部曲：

1. `CREATE ROLE mydb_admin LOGIN PASSWORD 'password';`

  创建此 database 的所有者。

2. `CREATE DATABASE my_db WITH owner = mydb_admin;`

  创建 database 并设定所有者

3. 用 'mydb_admin' 身份登录并做相关操作。


`GRANT` 基本用法：

`GRANT some_privilege TO some_role;`

**注意要点：**

- 可以通过 `WITH GRANT` 使被授权者可以再次授予别人。
- `PUBLIC` 关键字能用来指代所有角色。
- `REVOKE` 命令能取消权限。

   具体语法：

   `REVOKE EXECUTE ON ALL FUNCTIONS IN SCHEMA my_schema FROM PUBLIC;`

- 不要忘记对 schema 对象进行授权。

#### 扩展包机制

查找服务器上已安装的扩展

`SELECT name, default_version, installed_version, left(comment,30) AS comment FROM pg_available_extensions WHERE installed_version IS NOT NULL ORDER BY name;`

`\dx+ extension_name` 用来查看某个已安装的扩展包的详细信息。

`SELECT pg_catalog.pg_describe_object(d.classid, d.objid, 0) AS description FROM pg_catalog.pg_depend AS D INNER JOIN pg_catalog.pg_extension AS E ON D.refobjib = E.oid WHERE D.refclassid = 'pg_catalog.pg_extension'::pg_catalog.regclass AND deptype
= 'e' AND E.extname = 'extension_name'`显示了该扩展包包含了哪些内容。

#### 备份与恢复

PostgreSQL 自带两个备份工具：pg_dump 和 pg_dumpall（需要 superuser 权限）。

`pg_dump -h localhost -p 5432 -U someuser -F c -b -v -f mydb.backup mydb` 这是备份某个数据库，备份的结果以自定义压缩格式输出

`pg_dump -h localhost -p 5432 -U someuser -C -F p -b -v -f mydb.backup mydb` 备份某个 database,备份结果以 SQL 文本方式输出,输出结果中需包括 CREATE DATABASE 语句:


`pg_dump -h localhost -p5432 -U someuser -F c -b -v -t *.pay* -f pay.backup mydb` 备份某个 database 中所有名称以“pay”开头的表,备份结果以自定义压缩格式输出

备份某个 database 中 hr 和 payroll 这两个 schema 中的所有数据,备份结果以自定义压缩
格式输出:
pg_dump -h localhost -p 5432 -U someuser -F c -b -v -n hr -n payroll -f hr.back- up mydb

备份某个 database 中除了 public schema 中的数据以外的所有数据,备份结果以自定义压缩格式输出:
pg_dump -h localhost -p 5432 -U someuser -F c -b -v -N public -f all_sch_ex cept_pub.backup mydb

将数据备份为 SQL 文本文件,且生成的 INSERT 语句是带有字段名列表的标准格式,该文 件可用于将数据导入到低于当前版本的 PostgreSQL 或者其他支持 SQL 的非 PostgreSQL 数 据库中(之所以能够实现这种数据移植过程,是因为标准的 SQL 文本可在任何支持 SQL 标准的数据库中执行):
pg_dump -h localhost -p 5432 -U someuser -F p --column-inserts -f se lect_tables.backup mydb

目录格式备份
pg_dump -h localhost -p 5432 -U someuser -F d -f /somepath/a_directory mydb

目录格式并行备份
pg_dump -h localhost -p 5432 -U someuser -j 3 -Fd -f /somepath/a_directory mydb

以下命令可实现仅备份角色和表空间定义:
pg_dumpall -h localhost -U postgres --port=5432 -f myglobals.sql --globals-only

如果仅需备份角色定义而无需备份表空间,那么可以加上 --roles-only 选项:
pg_dumpall -h localhost -U postgres --port=5432 -f myroles.sql --roles-only

##### 基于表空间机制进行存储管理

CREATE TABLESPACE secondary LOCATION '/usr/data/pgdata94_secondary'; 创建表空间

如果希望将一个 database 的所有对象都 移动到另一个表空间中,可以执行以下命令:
ALTER DATABASE mydb SET TABLESPACE secondary; 如果只希望移动一张表,命令如下:
ALTER TABLE mytable SET TABLESPACE secondary;

将 pg_default 默认表空间中的所有对象迁移到 secondary 表空间,所需的命令行如下: ALTER TABLESPACE pg_default MOVE ALL TO secondary;
在迁移过程中所涉及的 database 和表会被锁定。

#### 禁止的行为

1. 切记不要删除PostgreSQL系统文件
2. 不要把操作系统管理员权限授予PostgreSQL的系统 账号(postgres)
3. 不要把shared_buffers缓存区设置得过大
4. 不要将PostgreSQL服务器的侦听端口设为一个已被 其他程序占用的端口
