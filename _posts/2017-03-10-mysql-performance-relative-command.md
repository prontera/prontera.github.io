---
layout:     post
title:      "MySQL性能分析命令"
subtitle:   "慢查询、SQL执行计划分析与行锁仪表盘"
author:     "Chris"
header-img: "img/post-bg-6.jpg"
tags:
    - MySQL
---

# MySQL性能分析命令

本文主要记录MySQL中的慢查询日志和执行计划分析器的使用。

## 行锁的争夺情况

```mysql
mysql> show status like 'innodb_row_lock%';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| Innodb_row_lock_current_waits | 0     |
| Innodb_row_lock_time          | 0     |
| Innodb_row_lock_time_avg      | 0     |
| Innodb_row_lock_time_max      | 0     |
| Innodb_row_lock_waits         | 0     |
+-------------------------------+-------+
5 rows in set (0.00 sec)
```

`Innodb_row_lock_current_waits` 当前正在等待锁的数量

`Innodb_row_lock_time` 等待锁的总时间

`Innodb_row_lock_time_avg` 等待锁的平均时间

`Innodb_row_lock_time_max` 最长等待锁的时间

`Innodb_row_lock_waits` 等待锁的总数

## 慢查询日志

默认情况下数据库的慢查询日志是需要手工启动的，如果不是调优的需要一般不建议启动该参数，因为开启慢查询日志或多或少会带来一定的性能开销。

#### 查询是否开启慢查询

```mysql
mysql> show variables like 'slow_query_log';
+----------------+-------+
| Variable_name  | Value |
+----------------+-------+
| slow_query_log | OFF   |
+----------------+-------+
1 row in set (0.01 sec)
```

#### 启用慢查询日志

```java
mysql> set global slow_query_log=1;
```

#### 设置慢查询的时间阈值

```mysql
mysql> set global long_query_time=3;
```

#### 查询慢查询日志的位置

```mysql
mysql> show variables like 'slow_query_log_file';
+---------------------+--------------------------------------+
| Variable_name       | Value                                |
+---------------------+--------------------------------------+
| slow_query_log_file | /var/lib/mysql/551bc825aece-slow.log |
+---------------------+--------------------------------------+
1 row in set (0.01 sec)
```

#### 查询当前有多少条慢查询

```mysql
mysql> show status like 'Slow_queries';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Slow_queries  | 0     |
+---------------+-------+
1 row in set (0.00 sec)
```

#### 慢查询日志分析工具 - mysqldumpslow

`-s` 按照什么字段排序

​	`c`访问次数，`l`锁定时间，`r`返回记录，`t` 查询时间，`al` 平均锁定时间，`ar` 平均返回记录数，`at` 平均查询事件

`-t` 取出指定前几条数据

`-g` 正则表达式

##### 得到返回记录最多的15条SQL

```mysql
mysqldumpslow -s r -t 15 /var/lib/mysql/551bc825aece-slow.log
```

##### 获得访问次数最多的15条SQL

```mysql
mysqldumpslow -s c -t 15 /var/lib/mysql/551bc825aece-slow.log
```

按照时间排序，获取前10条有left join的慢查询语句

```mysql
mysqldumpslow -s t -t 10 -g "left join" /var/lib/mysql/551bc825aece-slow.log
```

## 查询SQL的执行细节

#### 查询是否有启用分析器

```mysql
mysql> show variables like 'profiling';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| profiling     | OFF   |
+---------------+-------+
1 row in set (0.00 sec)
```

#### 查看profile

```mysql
mysql> show profiles;
```

#### 诊断SQL

```mysql
mysql> show profile cpu,block io for query 2;
+----------------------+----------+----------+------------+--------------+---------------+
| Status               | Duration | CPU_user | CPU_system | Block_ops_in | Block_ops_out |
+----------------------+----------+----------+------------+--------------+---------------+
| starting             | 0.000138 | 0.000000 |   0.000000 |            0 |             0 |
| checking permissions | 0.000051 | 0.000000 |   0.000000 |            0 |             0 |
| Opening tables       | 0.000049 | 0.000000 |   0.000000 |            0 |             0 |
| init                 | 0.000276 | 0.000000 |   0.000000 |            0 |             0 |
| System lock          | 0.000072 | 0.000000 |   0.000000 |            0 |             0 |
| optimizing           | 0.000025 | 0.000000 |   0.000000 |            0 |             0 |
| optimizing           | 0.000018 | 0.000000 |   0.000000 |            0 |             0 |
| statistics           | 0.000563 | 0.000000 |   0.000000 |            0 |             0 |
| preparing            | 0.000053 | 0.000000 |   0.000000 |            0 |             0 |
| statistics           | 0.000106 | 0.000000 |   0.000000 |            0 |             0 |
| preparing            | 0.000039 | 0.000000 |   0.000000 |            0 |             0 |
| executing            | 0.000028 | 0.000000 |   0.000000 |            0 |             0 |
| Sending data         | 0.000019 | 0.000000 |   0.000000 |            0 |             0 |
| executing            | 0.000017 | 0.000000 |   0.000000 |            0 |             0 |
| Sending data         | 0.005607 | 0.010000 |   0.000000 |            0 |             0 |
| end                  | 0.000053 | 0.000000 |   0.000000 |            0 |             0 |
| query end            | 0.000028 | 0.000000 |   0.000000 |            0 |             0 |
| closing tables       | 0.000017 | 0.000000 |   0.000000 |            0 |             0 |
| removing tmp table   | 0.000885 | 0.000000 |   0.000000 |            0 |             0 |
| closing tables       | 0.000068 | 0.000000 |   0.000000 |            0 |             0 |
| freeing items        | 0.000105 | 0.000000 |   0.000000 |            0 |             0 |
| cleaning up          | 0.000047 | 0.000000 |   0.000000 |            0 |             0 |
+----------------------+----------+----------+------------+--------------+---------------+
22 rows in set, 1 warning (0.01 sec)
```

除了上述的cpu和block io以外，还有如下参数

`all` 显示所有的开销信息

`block io` 显示IO相关的开销

`cpu` CPU相关开销

`ipc` 发送和接收的开销

`memory` 内存相关开销

`page faults` 页面错误的相关开销



如果在Status中出现如下记录就要**必须特别注意**：

`converting HEAP to MyISAM`  查询结果太大，内存不够使用

`creating tmp table` 这个状态是必须要注意的，拷贝数据到临时表，完整需要的操作后将其删除

`copying to tmp table on disk` 把内存中临时表复制到磁盘

