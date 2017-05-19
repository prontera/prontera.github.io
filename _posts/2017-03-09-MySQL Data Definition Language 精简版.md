---
layout:     post
title:      "MySQL Data Definition Language 精简版"
subtitle:   "常用DDL的总结备忘"
author:     "Chris"
header-img: "img/post-bg-5.jpg"
tags:
    - MySQL
---

### 创建数据库

```mysql
CREATE DATABASE IF NOT EXISTS product DEFAULT CHARSET utf8 ;
```

### 删除数据库

```mysql
DROP DATABASE IF EXISTS product;
```

### 创建表

测试用的示例, 包含单值索引, 多值索引, 外键

```mysql
CREATE TABLE IF NOT EXISTS `account`.`t_user` (
  `id` BIGINT(19) UNSIGNED NOT NULL AUTO_INCREMENT,
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `update_time` DATETIME NOT NULL DEFAULT '1970-01-01 00:00:00',
  `delete_time` DATETIME NOT NULL DEFAULT '1970-01-01 00:00:00',
  `mobile` VARCHAR(20) NOT NULL COMMENT '手机号',
  `login_pwd` VARCHAR(255) NOT NULL COMMENT '登录密码',
  `pwd_salt` VARCHAR(255) NOT NULL COMMENT '密码盐',
  `balance` BIGINT(19) NOT NULL DEFAULT 100000000 COMMENT '余额',
  UNIQUE INDEX `uni_user_mobile` (`mobile` ASC),
  PRIMARY KEY (`id`),
  INDEX `idx_user_mobileCt` (`mobile` ASC, `create_time` ASC));

CREATE TABLE IF NOT EXISTS `account`.`t_order` (
  `id` BIGINT(19) NOT NULL AUTO_INCREMENT,
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `update_time` DATETIME NULL DEFAULT '1970-01-01 00:00:00',
  `delete_time` DATETIME NOT NULL DEFAULT '1970-01-01 00:00:00',
  `t_user_id` BIGINT(19) UNSIGNED NOT NULL,
  PRIMARY KEY (`id`),
  INDEX `fk_t_order_t_user_idx` (`t_user_id` ASC),
  CONSTRAINT `fk_t_order_t_user`
    FOREIGN KEY (`t_user_id`)
    REFERENCES `account`.`t_user` (`id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION);
```

### 更改表名

```mysql
ALTER TABLE `account`.`t_user` 
RENAME TO  `account`.`t_userrr` ;
```

### 删除表

```mysql
DROP TABLE IF EXISTS `t_user`;
```

### 查看表的数据字典

```mysql
DESC t_order;
```

### 查看表的创建语句

```mysql
SHOW CREATE TABLE t_order;
```

### 新增列

```mysql
ALTER TABLE `t_user` ADD COLUMN `new_clo` INT UNSIGNED NOT NULL DEFAULT 0 AFTER `balance`
```

### 删除列

```mysql
ALTER TABLE `account`.`t_user` 
DROP COLUMN `new_clo`;
```

### 修改列名

```mysql
ALTER TABLE `t_user` 
CHANGE COLUMN `new_clo` `new_cloumn` INT(10) UNSIGNED NOT NULL DEFAULT '0' ;
```

### 修改列的属性

```mysql
ALTER TABLE `t_user` 
CHANGE COLUMN `new_clo` `new_clo` VARCHAR(20) NOT NULL DEFAULT 'hey' ;
```

### 添加索引

```mysql
ALTER TABLE `t_order` ADD INDEX `idx_ct_ut`(`create_time` ASC, `update_time` ASC);
```

### 添加前缀索引

```mysql
ALTER TABLE `product`.`t_user` 
ADD INDEX `idx_user_login_pwd` (`login_pwd`(30) ASC);
```

### 添加唯一索引

```mysql
ALTER TABLE `product`.`t_user` 
ADD UNIQUE INDEX `uni_balance` (`balance` ASC);
```

### 删除索引

索引的更新只能先删除, 再创建

```mysql
ALTER TABLE `product`.`t_user` 
DROP INDEX `uni_user_mobile`;
```

### 添加外键约束

```mysql
ALTER TABLE `product`.`t_order` 
ADD INDEX `fk_t_order_t_user_idx` (`t_user_id` ASC);
ALTER TABLE `product`.`t_order` 
ADD CONSTRAINT `fk_t_order_t_user`
  FOREIGN KEY (`t_user_id`)
  REFERENCES `product`.`t_user` (`id`)
  ON DELETE NO ACTION
  ON UPDATE NO ACTION;
```

### 删除外键约束

```mysql
ALTER TABLE `product`.`t_order` 
DROP FOREIGN KEY `fk_t_order_t_user`;
ALTER TABLE `product`.`t_order` 
DROP INDEX `fk_t_order_t_user_idx` ;
```

