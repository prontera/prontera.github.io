---
layout:     post
title:      "MySQL主从复制实战 - 基于日志点的复制"
subtitle:   "动手尝试构建MySQL主从复制集群"
author:     "Chris"
header-img: "img/post-bg-2.jpg"
tags:
    - MySQL
---

## 基于日志点的复制

1. 在**主库与从库**上建立专用的复制账号

   ```mysql
   MariaDB [employees]> create user 'repl'@'172.%' identified by '123456';
   ```

   注意在生产上的密码必须依照相关规范以达到一定的密码强度, 并且规定在从库上的特定网段上才能访问主库

2. 在主库与从库上授予复制权限

   ```mysql
   MariaDB [employees]> grant replication slave on *.* to 'repl'@'172.%';
   ```

3. 配置主库

   注意启用二进制日志需要重启服务, 而server_id是一个动态参数, 可以结合命令行与配置文件以达到免重启的持久化配置. 注意server_id在集群中是唯一的.

   ```shell
   [mysqld]
   log_bin = /var/log/mysql/mariadb-bin
   log_bin_index = /var/log/mysql/mariadb-bin.index
   binlog_format = row
   server_id = 101
   ```
   > NOTE: 把日志与数据分开是个好习惯, 最好能放到不同的数据分区

4. 配置从库

   选项`log_slave_update`决定是否把中继日志relay_log存放到本机的binlog中, 如果是配置链路复制, 那么该选项必填. 注意server_id在集群中是唯一的.

   ```shell
   [mysqld]
   # replication
   log_bin = /var/log/mysql/mariadb-bin
   log_bin_index = /var/log/mysql/mariadb-bin.index
   server_id = 102
   # slaves
   relay_log		= /var/log/mysql/relay-bin
   relay_log_index	= /var/log/mysql/relay-bin.index
   relay_log_info_file	= /var/log/mysql/relay-bin.info
   log_slave_updates = ON
   read_only
   ```

5. 初始化从库的数据

   此处使用mysqldump在主库上进行备份, 在生产上建议大家用xtrabackup进行无锁的热备(基于innodb引擎).

   备份主库上的employees数据库的数据

   ```shell
   mysqldump --single-transaction --master-data=1 --triggers --routines --databases employees -u root -p >> backup.sql
   ```

   将备份文件`backup.sql`通过scp或者docker volume卷挂载到从服务器上, 并且导入至从库中

   ```shell
   mysql -u root -p < backup.sql
   ```

6. 启动复制链路

   现有`master@172.20.0.2`和`slave@172.20.0.3`, 并且已经通过mysqldump将数据同步至从库slave中. 现在在**从**服务器slave上配置复制链路

   ```mysql
   MariaDB [(none)]> CHANGE MASTER TO MASTER_HOST='master', MASTER_USER='repl', MASTER_PASSWORD='123456',  MASTER_LOG_FILE='mariadb-bin.000029', MASTER_LOG_POS=516;
   Query OK, 0 rows affected (0.02 sec)
   ```

   在**从**库上启动复制链路

   ```mysql
   MariaDB [(none)]> start slave;
   Query OK, 0 rows affected (0.01 sec)
   ```

7. 在**从**库上检查slave状态

   `Slave_IO_Running`与`Slave_SQL_Running`必须为YES, 如果出现错误须详细阅读`Last_IO_Error`或`Last_SQL_Error`的提示信息

   ```mysql
   MariaDB [(none)]> show slave status\G
   *************************** 1. row ***************************
                  Slave_IO_State: Waiting for master to send event
                     Master_Host: master
                     Master_User: repl
                     Master_Port: 3306
                   Connect_Retry: 60
                 Master_Log_File: mariadb-bin.000029
             Read_Master_Log_Pos: 516
                  Relay_Log_File: relay-bin.000002
                   Relay_Log_Pos: 539
           Relay_Master_Log_File: mariadb-bin.000029
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
             Exec_Master_Log_Pos: 516
                 Relay_Log_Space: 831
                 Until_Condition: None
                  Until_Log_File:
                   Until_Log_Pos: 0
              Master_SSL_Allowed: No
              Master_SSL_CA_File:
              Master_SSL_CA_Path:
                 Master_SSL_Cert:
               Master_SSL_Cipher:
                  Master_SSL_Key:
           Seconds_Behind_Master: 0
   Master_SSL_Verify_Server_Cert: No
                   Last_IO_Errno: 0
                   Last_IO_Error:
                  Last_SQL_Errno: 0
                  Last_SQL_Error:
     Replicate_Ignore_Server_Ids:
                Master_Server_Id: 101
                  Master_SSL_Crl:
              Master_SSL_Crlpath:
                      Using_Gtid: No
                     Gtid_IO_Pos:
         Replicate_Do_Domain_Ids:
     Replicate_Ignore_Domain_Ids:
                   Parallel_Mode: conservative
   1 row in set (0.00 sec)
   ```

8. 在**主**库检查dump线程

   检测是否已经正确启动binlog dump线程

   ```mysql
   MariaDB [(none)]> show processlist \G
   *************************** 1. row ***************************
         Id: 7
       User: root
       Host: 172.20.0.1:41868
         db: employees
    Command: Sleep
       Time: 56
      State:
       Info: NULL
   Progress: 0.000
   *************************** 2. row ***************************
         Id: 10
       User: repl
       Host: 172.20.0.3:45974
         db: NULL
    Command: Binlog Dump
       Time: 246
      State: Master has sent all binlog to slave; waiting for binlog to be updated
       Info: NULL
   Progress: 0.000
   ```

   可以看到row 2上有Command为Binlog Dump的命令被启动, 证明复制线程已经被成功启动

9. 总结

   - 优点

     1. 技术成熟, BUG相对较少
     2. 对SQL查询没有任何限制, 如基于GTID复制时不是所有SQL都可以使用

   - 缺点

     1. 故障转移时重新获取新主的日志偏移量较为困难

        在一主多从环境下, 若旧master宕机后在集群中选举出新master, 其他的从库要对这个新的master进行重新同步, 由于每个DB的binlog都是独立存在, 所以很难找出开始同步的日志点