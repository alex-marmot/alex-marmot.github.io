---
layout: post
title: "These Are a Few of My Favorite (psql) Things"
date: 2016-03-26
categories: 翻译
---

##### 感谢作者 Thijs Cadier [原文链接](http://blog.thefourthparty.com/these-are-a-few-of-my-favorite-psql-things/?utm_source=postgresweekly&utm_medium=email)

## 我最喜欢的 psql 技巧

我使用 psql 做日常的快速查询，提醒我某张表特定的表结构，或者分析性能以及其事项。这里跟大家分享一些使我的体验更高大上的 psql 命令。

    \? help with psql commands
    myawesomedb=# \?

首先是 psql 帮助命令的快捷方式。这会列出所有的可用命令。我会挑选一两个有用的命令并专注于将其应用一两周。这样有助于我记住它们并最终将它们加入
到我的工作流中。

    \x [on|off|auto] toggle expanded output (currently off)
    myawesomedb=# \x

这大概是我最常用的命令，它会将输出结果的列按行排序来代替默认的按列输出。对拥有大量列的表格，这能使结果更清晰。

    \timing [on|off] toggle timing of commands (currently off)
    myawesomedb=# \timing

在测试查询性能时特别有用。开启 timing 后，就会在 psql 终端中输出每个命令的运行时间。我在调整目录或者实验时用得最多。

    \c[onnect] [DBNAME|- USER|- HOST|- PORT|-]
    connect to new database (currently "myawesomedb")
    myawesomedb=# \c laurens-other-db

连接到另一个数据库。举例来说，在测试数据库和开发数据库间切换时特别有用。同样也节省了退出 psql 终端并以新的数据库名启动新的会话的额外步骤。

    \l[+] [PATTERN] list databases
    \dn[S+] [PATTERN] list schemas
    \dt[S+] [PATTERN] list tables
    myawesomedb=# \l+
    myawesomedb=# \dn+
    myawesomedb=# \dt+

列出所有的信息同时包括它们的大小！注意这表示这张表或者数据库占了多少空间。这也意味着 你的数据库是否需要。

有的查询会用到 postgres 内部的表用以报告关联数据的大小。略。总的来说，‘+’ 会添加 ‘额外的细节’。对数据库来说，会添加访问优先级和描述信息。
对于表来说，会添加大小和描述信息。对于 schema 来说会添加访问优先级和描述信息。在'+'的结果，通常是
 谷歌上搜索一个特定的 Postgres内部查询 之前有益的起点。

在'+'通常还包括描述列。 Postgres 的很多东西可以注释其中（详见完整列表注释文档）。这些注释就是在数据库中记录的文档。然而，我的经验是因为这些列不容易看到，所以它们很少被检查。
因此，他们通常与真实的数据不同步。


    \q quit psql
    myawesomedb=# \q

就是以上这些！好好享用吧！
