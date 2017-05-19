---
layout:     post
title:      "MySQL 二进制日志格式"
subtitle:   "statement与row格式的优缺点"
author:     "Chris"
header-img: "img/post-bg-5.jpg"
tags:
    - MySQL
---

## 日志分类

- MySQL存储引擎层日志
  - innodb
    - 重做日志
    - 回滚日志
- MySQL服务层日志
  - **二进制日志**
  - 慢查日志
  - 通用日志

## 二进制日志介绍

记录了所有对MySQL数据库的**<u>修改</u>**事件, 包括DDL和DML操作. 其中binlog仅记录成功执行的日志, 对于回滚或者Syntax Error而未执行的事件并不记录.

## 启用二进制日志

```mysql
MariaDB [(none)]> show variables like 'log_bin';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | OFF   |
+---------------+-------+
1 row in set (0.00 sec)
```

log_bin为OFF说为未启用二进制日志, 我们尝试通过以下指令打开

```mysql
MariaDB [(none)]> set global log_bin = 'mariadb-bin';
ERROR 1238 (HY000): Variable 'log_bin' is a read only variable
```

好吧, 其实仍然是要修改my.cnf配置文件, 在my.cnf或自定义配置中添加[mysqld]组, 设置log_bin属性的值为`mariadb-bin`以开启binlog.

```mysql
[mysqld]
log_bin = /var/log/mysql/mariadb-bin
```

以后我们查看目录/var/log/mysql/的时候就会发现有如`mariadb-bin.000001`的一系列日志.待重启MySQL后执行

```mysql
MariaDB [(none)]> show variables like 'log_bin';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
1 row in set (0.00 sec)
```

现在也可以正常显示binary的信息

```mysql
MariaDB [(none)]> show binary logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       313 |
+------------------+-----------+
1 row in set (0.00 sec)
```

> NOTE: mysql> flush logs; 用于产生新的二进制日志, 方便调试.

## 二进制日志的格式

- 基于段的日志格式 - statement

  这是MySQL 5.7之前默认的日志格式, 记录CUD的**SQL语句**.

  ```mysql
  MariaDB [(none)]> show variables like 'binlog_format';
  +---------------+-----------+
  | Variable_name | Value     |
  +---------------+-----------+
  | binlog_format | STATEMENT |
  +---------------+-----------+
  1 row in set (0.00 sec)
  ```

  - 查看日志

  mysqlbinlog工具可直接用于查看日志

  ```shell
  mysqlbinlog mysql-bin.000001
  ```

  - 优点:

    记录每个事件所执行的SQL语句, 故不需要记录每一行的具体变化, 所以日志记录量相对较少, 节约磁盘IO与网络IO(如果只对一条记录修改或者插入, ROW格式的日志量有可能少于STATEMENT格式).

  - 缺点:

    为了确保这些SQL语句能在从库中正确地执行, 所以要记录上下文信息, 以保证重放时的行为一致. 但如果使用UUID()这类非确定性函数, 可能会造成主从的数据不一致.

- 基于行的日志格式 - row

  在MySQL 5.7后的默认格式, 每当进行CUD操作修改行记录时都会将**数据**写入binlog.

  如果我们有一个修改了10k条数据的情况下, 基于段的日志格式仅会记录该条SQL, 而基于行的日志则会有10k条记录

  * 查看日志

    ```shell
    mysqlbinlog -vv mysql-bin.000001
    ```


* 优点
    1. 使主从复制更加安全, 由于ROW格式记录了每一行的更改, 当日志被从库重放时仅需应用该条更改即可, 使得不确定性函数也能安全地使用. 更大程度上减少由于主从数据不一致而造成复制链路中断的情况.
    2. 基于行的复制比基于段的复制要高效
* 缺点
    1. 记录的日志量较大

- 混合日志格式 - mixed

  混合使用STATEMENT和ROW, 根据SQL语句由系统决定使用基于段还是基于行的日志格式, 如非确定性函数则会以ROW格式记录, 其他的以STATEMENT记录.

## 修改binlog格式

```mysql
MariaDB [(none)]> set binlog_format = 'statement';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> show variables like 'binlog_format';
+---------------+-----------+
| Variable_name | Value     |
+---------------+-----------+
| binlog_format | STATEMENT |
+---------------+-----------+
1 row in set (0.00 sec)
```

## ROW日志格式的相关优化

由于ROW格式记录的日志量巨大, 在MySQL 5.6以后, 官方增加了`binlog_row_image`参数改善其记录方式.

```mysql
MariaDB [(none)]> show variables like 'binlog_row_image';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| binlog_row_image | FULL  |
+------------------+-------+
1 row in set (0.00 sec)
```

`FULL`为默认值, 意思是记录一行纪录里面的所有内容, 无论该列是否被修改.

`MINIMAL`仅记录被修改的列, 这样就可以大大减少记录量.

`NOBLOB`与FULL类型, 但是如果没有修改BLOB或TEXT类型的列, 就不会记录该大数据类型的列.

### 实验

假如有表结构如下

```mysql
CREATE TABLE `t_test` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `title` varchar(32) NOT NULL DEFAULT '',
  `content` text,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

表中有记录

`id->1, title->'prontera', content->'hey guy'`

* FULL (全记录)

  我们修改id为1的记录, 然后查看/var/log/mysql中的binlog

  ```mysql
  MariaDB [employees]> update t_test set title = 'solar' where id = 1;
  Query OK, 1 row affected (0.01 sec)
  Rows matched: 1  Changed: 1  Warnings: 0
  ```

  可以看出, 我们仅仅set title, 但是binlog_row_image为FULL的时候会记录所有的列

  ```shell
  BINLOG '
  HPyWWBMBAAAANwAAAM8BAAAAACEAAAAAAAEACWVtcGxveWVlcwAGdF90ZXN0AAMDD/wDYAACBA==
  HPyWWBgBAAAASQAAABgCAAAAACEAAAAAAAEAA///+AEAAAAIcHJvbnRlcmEHAGhleSBndXn4AQAA
  AAVzb2xhcgcAaGV5IGd1eQ==
  '/*!*/;
  ### UPDATE `employees`.`t_test`
  ### WHERE
  ###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
  ###   @2='prontera' /* VARSTRING(96) meta=96 nullable=0 is_null=0 */
  ###   @3='hey guy' /* BLOB/TEXT meta=2 nullable=1 is_null=0 */
  ### SET
  ###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
  ###   @2='solar' /* VARSTRING(96) meta=96 nullable=0 is_null=0 */
  ###   @3='hey guy' /* BLOB/TEXT meta=2 nullable=1 is_null=0 */
  ```

* MINIMAL (按需记录)

  同样地, 将id为1的记录的title更新为原先的'prontera', 对比binlog的确是有效地减少了日志量

  ```shell
  BINLOG '
  rv2WWBMBAAAANwAAAM8BAAAAACEAAAAAAAEACWVtcGxveWVlcwAGdF90ZXN0AAMDD/wDYAACBA==
  rv2WWBgBAAAALQAAAPwBAAAAACEAAAAAAAEAAwEC/gEAAAD+CHByb250ZXJh
  '/*!*/;
  ### UPDATE `employees`.`t_test`
  ### WHERE
  ###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
  ### SET
  ###   @2='prontera' /* VARSTRING(96) meta=96 nullable=0 is_null=0 */
  ```

* NOBLOB (除去BLOB&TEXT的全纪录)
  更新时不包含text类型的列

  ```mysql
  MariaDB [employees]> update t_test set title = 'juno' where id = 1;
  Query OK, 1 row affected (0.01 sec)
  Rows matched: 1  Changed: 1  Warnings: 0
  ```

  类似于FULL模式, 但是binlog不会记录没有被更新的大数据类型(BLOB或TEXT类型)

  ```shell
  BINLOG '
  hP+WWBMBAAAANwAAAM8BAAAAACEAAAAAAAEACWVtcGxveWVlcwAGdF90ZXN0AAMDD/wDYAACBA==
  hP+WWBgBAAAAMwAAAAICAAAAACEAAAAAAAEAAwMD/AEAAAAFcGF5b278AQAAAARqdW5v
  '/*!*/;
  ### UPDATE `employees`.`t_test`
  ### WHERE
  ###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
  ###   @2='payon' /* VARSTRING(96) meta=96 nullable=0 is_null=0 */
  ### SET
  ###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
  ###   @2='juno' /* VARSTRING(96) meta=96 nullable=0 is_null=0 */
  ```

  更新时包含text列

  ```mysql
  MariaDB [employees]> update t_test set title = 'prontera', content = 'google'  where id = 1;
  Query OK, 1 row affected (0.00 sec)
  Rows matched: 1  Changed: 1  Warnings: 0
  ```

  当有大数据类型的列被更新时, 该列才会被记录, 一定程度上减少了日志量.

  ```shell
  BINLOG '
  PQCXWBMBAAAANwAAAM8BAAAAACEAAAAAAAEACWVtcGxveWVlcwAGdF90ZXN0AAMDD/wDYAACBA==
  PQCXWBgBAAAAPgAAAA0CAAAAACEAAAAAAAEAAwMH/AEAAAAEanVub/gBAAAACHByb250ZXJhBgBn
  b29nbGU=
  '/*!*/;
  ### UPDATE `employees`.`t_test`
  ### WHERE
  ###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
  ###   @2='juno' /* VARSTRING(96) meta=96 nullable=0 is_null=0 */
  ### SET
  ###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
  ###   @2='prontera' /* VARSTRING(96) meta=96 nullable=0 is_null=0 */
  ###   @3='google' /* BLOB/TEXT meta=2 nullable=1 is_null=0 */
  ```


  通过实验, 在基于ROW格式情况下, 即使处于同一IDC, 也建议使用minimal模式以大量减少日志量.