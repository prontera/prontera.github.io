---
layout:     post
title:      "MySQL主从复制实战 - 基于GTID的复制"
subtitle:   "动手试一次MySQL的GTID主从复制模式"
author:     "Chris"
header-img: "img/post-bg-3.jpg"
tags:
    - MySQL
---

## 基于GTID的复制

### 简介

基于GTID的复制是**MySQL 5.6后**新增的复制方式.

GTID (global transaction identifier) 即全局事务ID, 保证了在每个在主库上提交的事务在集群中有一个唯一的ID.

在原来基于日志的复制中, 从库需要告知主库要从哪个偏移量进行增量同步, 如果指定错误会造成数据的遗漏, 从而造成数据的不一致.

而基于GTID的复制中, 从库会告知主库已经执行的事务的GTID的值, 然后主库会将所有未执行的事务的GTID的列表返回给从库. 并且可以保证同一个事务只在指定的从库执行一次. 

### 实战

1. 在**主库**上建立复制账户并授予权限

   基于GTID的复制会自动地将没有在从库执行的事务重放, 所以不要在其他从库上建立相同的账号. 如果建立了相同的账户, 有可能造成复制链路的错误.

   ```mysql
   mysql> create user 'repl'@'172.%' identified by '123456';
   ```

   注意在生产上的密码必须依照相关规范以达到一定的密码强度, 并且规定在从库上的特定网段上才能访问主库.

   ```mysql
   mysql> grant replication slave on *.* to 'repl'@'172.%';
   ```

   查看用户

   ```mysql
   mysql> select user, host from mysql.user;
   +-----------+-----------+
   | user      | host      |
   +-----------+-----------+
   | prontera  | %         |
   | root      | %         |
   | mysql.sys | localhost |
   | root      | localhost |
   +-----------+-----------+
   4 rows in set (0.00 sec)
   ```

   查看授权

   ```mysql
   mysql> show grants for repl@'172.%';
   +--------------------------------------------------+
   | Grants for repl@172.%                            |
   +--------------------------------------------------+
   | GRANT REPLICATION SLAVE ON *.* TO 'repl'@'172.%' |
   +--------------------------------------------------+
   1 row in set (0.00 sec)
   ```

2. 配置主库服务器

   ```sh
   [mysqld]
   log_bin = /var/log/mysql/mysql-bin
   log_bin_index = /var/log/mysql/mysql-bin.index
   binlog_format = row
   server_id = 101
   gtid_mode = ON
   enforce_gtid_consistency = ON
   #log_slave_updates = ON
   ```

   > NOTE: 把日志与数据分开是个好习惯, 最好能放到不同的数据分区

   `enforce_gtid_consistency` 强制GTID一致性, 启用后以下命令无法再使用

   - create table ... select ...

     ```mysql
     mysql> create table dept select * from departments;
     ERROR 1786 (HY000): Statement violates GTID consistency: CREATE TABLE ... SELECT.
     ```

     因为实际上是两个独立事件, 所以只能将其拆分先建立表, 然后再把数据插入到表中

   - create temporary table

     事务内部不能创建临时表

     ```mysql
     mysql> begin;
     Query OK, 0 rows affected (0.00 sec)

     mysql> create temporary table dept(id int);
     ERROR 1787 (HY000): Statement violates GTID consistency: CREATE TEMPORARY TABLE and DROP TEMPORARY TABLE can only be executed outside transactional context.  These statements are also not allowed in a function or trigger because functions and triggers are also considered to be multi-statement transactions.
     ```

   - 同一事务中更新事务表与非事务表(MyISAM)

     ```mysql
     mysql> CREATE TABLE `dept_innodb` (id INT(11) UNSIGNED NOT NULL PRIMARY KEY AUTO_INCREMENT);
     Query OK, 0 rows affected (0.04 sec)

     mysql> CREATE TABLE `dept_myisam` (id INT(11) UNSIGNED NOT NULL PRIMARY KEY AUTO_INCREMENT)   ENGINE = `MyISAM`;
     Query OK, 0 rows affected (0.03 sec)

     mysql> begin;
     Query OK, 0 rows affected (0.00 sec)

     mysql> insert into dept_innodb(id) value(1);
     Query OK, 1 row affected (0.00 sec)

     mysql> insert into dept_myisam(id) value(1);
     ERROR 1785 (HY000): Statement violates GTID consistency: Updates to non-transactional tables can only be done in either autocommitted statements or single-statement transactions, and never in the same statement as updates to transactional tables.
     ```

     所以建议选择Innodb作为默认的数据库引擎.

   `log_slave_updates` 该选项在MySQL 5.6版本时基于GTID的复制是必须的, 但是其增大了从服务器的IO负载, 而在MySQL 5.7中该选项已经不是必须项

3. 配置从库服务器

   `master_info_repository` 与`relay_log_info_repository`

   在MySQL 5.6.2之前, slave记录的master信息以及slave应用binlog的信息存放在文件中, 即master.info与relay-log.info. 在5.6.2版本之后, 允许记录到table中. 对应的表分别为mysql.slave_master_info与mysql.slave_relay_log_info, 且这两个表均为innodb引擎表.

   ```mysql
   [mysqld]
   log_bin = /var/log/mysql/mysql-bin
   log_bin_index = /var/log/mysql/mysql-bin.index
   server_id = 102
   # slaves
   relay_log		= /var/log/mysql/relay-bin
   relay_log_index	= /var/log/mysql/relay-bin.index
   relay_log_info_file	= /var/log/mysql/relay-bin.info
   enforce_gtid_consistency = ON
   log_slave_updates = ON
   read_only = ON
   master_info_repository = TABLE
   relay_log_info_repository = TABLE
   ```

4. 从库数据初始化 - [optional]

   先在**主库**上备份数据

   ```shell
   mysqldump --single-transaction --master-data=2 --triggers --routines --all-databases --events -u root -p > backup.sql
   ```

   `—master-data=2` 该选项将当前服务器的binlog的位置和文件名追加到输出文件中(show master status). 如果为1, 将偏移量拼接到CHANGE MASTER 命令. 如果为2, 输出的偏移量信息将会被注释。

   `--all-databases` 因为基于GTID的复制会记录全部的事务, 所以要构建一个完整的dump这个选项是推荐的

   ##### 常见错误

   当从库导入SQL的时候出现

   ```shell
   ERROR 1840 (HY000) at line 24: @@GLOBAL.GTID_PURGED can only be set when @@GLOBAL.GTID_EXECUTED is empty.
   ```

   此时进入从库的MySQL Command Line, 使用`reset master`即可

5. 启动基于GTID的复制

   现有`master@172.20.0.2`和`slave@172.20.0.3`, 并且已经通过mysqldump将数据同步至从库slave中. 现在在**从**服务器slave上配置复制链路

   ```mysql
   mysql> change master to master_host='master', master_user='repl', master_password='123456', master_auto_position=1;
   Query OK, 0 rows affected, 2 warnings (0.06 sec)
   ```

   启动复制

   ```mysql
   mysql> start slave;
   ```

   启动成功后查看slave的状态

   ```mysql
   mysql> show slave status\G
   *************************** 1. row ***************************
                  Slave_IO_State: Queueing master event to the relay log
                     Master_Host: master
                     Master_User: repl
                     Master_Port: 3306
                   Connect_Retry: 60
                 Master_Log_File: mysql-bin.000002
             Read_Master_Log_Pos: 12793692
                  Relay_Log_File: relay-bin.000002
                   Relay_Log_Pos: 1027
           Relay_Master_Log_File: mysql-bin.000002
                Slave_IO_Running: Yes
               Slave_SQL_Running: Yes
                 Replicate_Do_DB:
             Replicate_Ignore_DB:
              Replicate_Do_Table:
          Replicate_Ignore_Table:
         Replicate_Wild_Do_Table:
     Replicate_Wild_Ignore_Table:
                      Last_Errno: 0
                      Last_Error:
                    Skip_Counter: 0
             Exec_Master_Log_Pos: 814
                 Relay_Log_Space: 12794106
                 Until_Condition: None
                  Until_Log_File:
                   Until_Log_Pos: 0
              Master_SSL_Allowed: No
              Master_SSL_CA_File:
              Master_SSL_CA_Path:
                 Master_SSL_Cert:
               Master_SSL_Cipher:
                  Master_SSL_Key:
           Seconds_Behind_Master: 5096
   Master_SSL_Verify_Server_Cert: No
                   Last_IO_Errno: 0
                   Last_IO_Error:
                  Last_SQL_Errno: 0
                  Last_SQL_Error:
     Replicate_Ignore_Server_Ids:
                Master_Server_Id: 101
                     Master_UUID: a9fd4765-ec70-11e6-b543-0242ac140002
                Master_Info_File: mysql.slave_master_info
                       SQL_Delay: 0
             SQL_Remaining_Delay: NULL
         Slave_SQL_Running_State: Reading event from the relay log
              Master_Retry_Count: 86400
                     Master_Bind:
         Last_IO_Error_Timestamp:
        Last_SQL_Error_Timestamp:
                  Master_SSL_Crl:
              Master_SSL_Crlpath:
              Retrieved_Gtid_Set: a9fd4765-ec70-11e6-b543-0242ac140002:1-39
               Executed_Gtid_Set: a9fd4765-ec70-11e6-b543-0242ac140002:1-4
                   Auto_Position: 1
            Replicate_Rewrite_DB:
                    Channel_Name:
              Master_TLS_Version:
   1 row in set (0.00 sec)
   ```

   当`Slave_IO_Running`, `Slave_SQL_Running`为YES, 

   且`Slave_SQL_Running_State` 为Slave has read all relay log; waiting for more updates时表示成功构建复制链路

6. 总结

   - 优点
     1. 因为不用手工设置日志偏移量, 可以很方便地进行故障转移
     2. 如果启用`log_slave_updates`那么从库不会丢失主库上的任何修改
   - 缺点
     1. 对执行的SQL有一定限制
     2. 仅支持MySQL 5.6之后的版本, 而且不建议使用早期5.6版本