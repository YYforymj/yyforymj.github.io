---

layout: post
title:  "使用mysql的general_log抓取sql"
date:   2025-03-14 10:05:00 +0800
categories: 日常工作

---

### 说明

测试环境服务运行中，如果 sql 执行报错了，但是又没有配置打印对应的日志，往往需要去配置中心开启对应的日志级别。这就需要找到mapper 的包，略微繁琐。但是利用 mysql 的 general_log 可以记录 sql 日志的特性，配合一定的查询条件，可以快速检索 mysql上执行的 sql 方便调试或排除处理问题。

### 一些准备工作

由于是利用 mysql 的 general_log ，所以首先要确认 general_log  的写入类型

```
show variables like '%log_output%';
```

如果返回如下，说明日志写入到了文件

| Variable_name | Value |
| ------------- | ----- |
| log_output    | FILE  |

需要执行如下 sql 设置为输出到数据表

```
set GLOBAL log_output='table'; -- 表格输出
```

然后再次检查  general_log  的写入类型，返回如下就可以了

| Variable_name | Value |
| ------------- | ----- |
| log_output    | TABLE |

### 抓取sql

开始抓取，执行sql

```
set GLOBAL general_log=on;-- 开启日志
```

表格输出后的查看方式，进入information_schema数据库执行如下脚本：

```
select general_log.*,convert(argument using utf8) from mysql.general_log order by event_time desc;
```

返回结果会列出开启日志之后记录的所有 sql，可以直接 `ctrl + f` 查找 sql，或者在上面的 sql 中增加筛选条件。

抓取完成后，需要关闭日志并清理数据，避免长时间开启日志影响数据库性能与磁盘空间。

```
set GLOBAL general_log=off;-- 关闭日志
truncate mysql.general_log;-- 清空日志
```

