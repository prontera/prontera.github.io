---
layout:     post
title:      "MySQL二进制日志格式对复制的影响"
subtitle:   "make differents between SBR and RBR"
author:     "Chris"
header-img: "img/post-bg-4.jpg"
tags:
    - MySQL
---

## 复制的分类

### 基于SQL语句的复制 - SBR

  主库二进制日志格式使用STATEMENT

  在MySQL 5.1之前仅存在SBR模式, 又称之为逻辑复制.

  主库记录CUD操作的SQL语句, 从库会读取并重放.

- 优点

  1. 生成的日志量少, 节约网络传输IO

  2. 当主从的列的顺序不一致时, SBR依然可以正常工作.

     如对大表进行结构修改时, 可以先修改从库, 然后再进行主从切换.

- 缺点

  1. 对不确定性函数无法保证主从数据的一致
  2. 对于procedure, trigger, function有可能在主从上表现不一致(SBR BUG)
  3. 主库上要锁定多少行, 从库上也需要所以多少行, 所以相对于ROW复制时从库上需要更多的行锁

### 基于行的复制 - RBR

 主库二进制日志格式使用ROW

- 优点

  1. 对不确定性函数友好, 如UUID()

  2. 减少从库上数据库锁的使用

     ```mysql
     insert into t_order_cnt(timestr, total, amount)
     	select date(order_date), count(1), sum(amout)
     	from t_order group by date(order_date);
     ```

     上面的SQL在主库执行时会对`t_order`进行锁表操作, 对于STATEMENT的复制从库上也会对同样的表进行锁定, 但是基于ROW的复制仅需增加`t_order`对应的行的数据即可.

- 缺点

  1. 要求主从数据库的表的结构一致, 否则可能会中断复制
  2. 无法在从库上激活trigger