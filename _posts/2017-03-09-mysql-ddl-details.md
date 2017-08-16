---
layout:     post
title:      "MySQL Data Definition Language 详细版"
subtitle:   "https://dev.mysql.com/doc/refman/5.7/en/sql-syntax-data-definition.html"
author:     "Chris"
header-img: "img/post-bg-4.jpg"
tags:
    - MySQL
---

## 创建数据库

```mysql
CREATE {DATABASE | SCHEMA} [IF NOT EXISTS] db_name
    [create_specification] ...

create_specification:
    [DEFAULT] CHARACTER SET [=] charset_name
  | [DEFAULT] COLLATE [=] collation_name
```

在MySQL 5.7中如果有`LOCK TABLES`不允许创建`CREATE DATABASE`

`DATABASE`与`SCHEMA`是同义词

新创建的数据库会在MySQL的`$DATADIR`下创建同名文件夹, 并在文件夹下生成`db.opt`

## 创建表

```mysql
CREATE [TEMPORARY] TABLE [IF NOT EXISTS] tbl_name
    (create_definition,...)
    [table_options]
    [partition_options]

CREATE [TEMPORARY] TABLE [IF NOT EXISTS] tbl_name
    [(create_definition,...)]
    [table_options]
    [partition_options]
    [IGNORE | REPLACE]
    [AS] query_expression

CREATE [TEMPORARY] TABLE [IF NOT EXISTS] tbl_name
    { LIKE old_tbl_name | (LIKE old_tbl_name) }

create_definition:
    col_name column_definition
  | [CONSTRAINT [symbol]] PRIMARY KEY [index_type] (index_col_name,...)
      [index_option] ...
  | {INDEX|KEY} [index_name] [index_type] (index_col_name,...)
      [index_option] ...
  | [CONSTRAINT [symbol]] UNIQUE [INDEX|KEY]
      [index_name] [index_type] (index_col_name,...)
      [index_option] ...
  | {FULLTEXT|SPATIAL} [INDEX|KEY] [index_name] (index_col_name,...)
      [index_option] ...
  | [CONSTRAINT [symbol]] FOREIGN KEY
      [index_name] (index_col_name,...) reference_definition
  | CHECK (expr)

column_definition:
    data_type [NOT NULL | NULL] [DEFAULT default_value]
      [AUTO_INCREMENT] [UNIQUE [KEY] | [PRIMARY] KEY]
      [COMMENT 'string']
      [COLUMN_FORMAT {FIXED|DYNAMIC|DEFAULT}]
      [STORAGE {DISK|MEMORY|DEFAULT}]
      [reference_definition]
  | data_type [GENERATED ALWAYS] AS (expression)
      [VIRTUAL | STORED] [UNIQUE [KEY]] [COMMENT comment]
      [NOT NULL | NULL] [[PRIMARY] KEY]

data_type:
    BIT[(length)]
  | TINYINT[(length)] [UNSIGNED] [ZEROFILL]
  | SMALLINT[(length)] [UNSIGNED] [ZEROFILL]
  | MEDIUMINT[(length)] [UNSIGNED] [ZEROFILL]
  | INT[(length)] [UNSIGNED] [ZEROFILL]
  | INTEGER[(length)] [UNSIGNED] [ZEROFILL]
  | BIGINT[(length)] [UNSIGNED] [ZEROFILL]
  | REAL[(length,decimals)] [UNSIGNED] [ZEROFILL]
  | DOUBLE[(length,decimals)] [UNSIGNED] [ZEROFILL]
  | FLOAT[(length,decimals)] [UNSIGNED] [ZEROFILL]
  | DECIMAL[(length[,decimals])] [UNSIGNED] [ZEROFILL]
  | NUMERIC[(length[,decimals])] [UNSIGNED] [ZEROFILL]
  | DATE
  | TIME[(fsp)]
  | TIMESTAMP[(fsp)]
  | DATETIME[(fsp)]
  | YEAR
  | CHAR[(length)] [BINARY]
      [CHARACTER SET charset_name] [COLLATE collation_name]
  | VARCHAR(length) [BINARY]
      [CHARACTER SET charset_name] [COLLATE collation_name]
  | BINARY[(length)]
  | VARBINARY(length)
  | TINYBLOB
  | BLOB
  | MEDIUMBLOB
  | LONGBLOB
  | TINYTEXT [BINARY]
      [CHARACTER SET charset_name] [COLLATE collation_name]
  | TEXT [BINARY]
      [CHARACTER SET charset_name] [COLLATE collation_name]
  | MEDIUMTEXT [BINARY]
      [CHARACTER SET charset_name] [COLLATE collation_name]
  | LONGTEXT [BINARY]
      [CHARACTER SET charset_name] [COLLATE collation_name]
  | ENUM(value1,value2,value3,...)
      [CHARACTER SET charset_name] [COLLATE collation_name]
  | SET(value1,value2,value3,...)
      [CHARACTER SET charset_name] [COLLATE collation_name]
  | JSON
  | spatial_type

index_col_name:
    col_name [(length)] [ASC | DESC]

index_type:
    USING {BTREE | HASH}

index_option:
    KEY_BLOCK_SIZE [=] value
  | index_type
  | WITH PARSER parser_name
  | COMMENT 'string'

reference_definition:
    REFERENCES tbl_name (index_col_name,...)
      [MATCH FULL | MATCH PARTIAL | MATCH SIMPLE]
      [ON DELETE reference_option]
      [ON UPDATE reference_option]

reference_option:
    RESTRICT | CASCADE | SET NULL | NO ACTION | SET DEFAULT

table_options:
    table_option [[,] table_option] ...

table_option:
    ENGINE [=] engine_name
  | AUTO_INCREMENT [=] value
  | AVG_ROW_LENGTH [=] value
  | [DEFAULT] CHARACTER SET [=] charset_name
  | CHECKSUM [=] {0 | 1}
  | [DEFAULT] COLLATE [=] collation_name
  | COMMENT [=] 'string'
  | COMPRESSION [=] {'ZLIB'|'LZ4'|'NONE'}
  | CONNECTION [=] 'connect_string'
  | DATA DIRECTORY [=] 'absolute path to directory'
  | DELAY_KEY_WRITE [=] {0 | 1}
  | ENCRYPTION [=] {'Y' | 'N'}
  | INDEX DIRECTORY [=] 'absolute path to directory'
  | INSERT_METHOD [=] { NO | FIRST | LAST }
  | KEY_BLOCK_SIZE [=] value
  | MAX_ROWS [=] value
  | MIN_ROWS [=] value
  | PACK_KEYS [=] {0 | 1 | DEFAULT}
  | PASSWORD [=] 'string'
  | ROW_FORMAT [=] {DEFAULT|DYNAMIC|FIXED|COMPRESSED|REDUNDANT|COMPACT}
  | STATS_AUTO_RECALC [=] {DEFAULT|0|1}
  | STATS_PERSISTENT [=] {DEFAULT|0|1}
  | STATS_SAMPLE_PAGES [=] value
  | TABLESPACE tablespace_name [STORAGE {DISK|MEMORY|DEFAULT}]
  | UNION [=] (tbl_name[,tbl_name]...)

partition_options:
    PARTITION BY
        { [LINEAR] HASH(expr)
        | [LINEAR] KEY [ALGORITHM={1|2}] (column_list)
        | RANGE{(expr) | COLUMNS(column_list)}
        | LIST{(expr) | COLUMNS(column_list)} }
    [PARTITIONS num]
    [SUBPARTITION BY
        { [LINEAR] HASH(expr)
        | [LINEAR] KEY [ALGORITHM={1|2}] (column_list) }
      [SUBPARTITIONS num]
    ]
    [(partition_definition [, partition_definition] ...)]

partition_definition:
    PARTITION partition_name
        [VALUES
            {LESS THAN {(expr | value_list) | MAXVALUE}
            |
            IN (value_list)}]
        [[STORAGE] ENGINE [=] engine_name]
        [COMMENT [=] 'comment_text' ]
        [DATA DIRECTORY [=] 'data_dir']
        [INDEX DIRECTORY [=] 'index_dir']
        [MAX_ROWS [=] max_number_of_rows]
        [MIN_ROWS [=] min_number_of_rows]
        [TABLESPACE [=] tablespace_name]
        [(subpartition_definition [, subpartition_definition] ...)]

subpartition_definition:
    SUBPARTITION logical_name
        [[STORAGE] ENGINE [=] engine_name]
        [COMMENT [=] 'comment_text' ]
        [DATA DIRECTORY [=] 'data_dir']
        [INDEX DIRECTORY [=] 'index_dir']
        [MAX_ROWS [=] max_number_of_rows]
        [MIN_ROWS [=] min_number_of_rows]
        [TABLESPACE [=] tablespace_name]

query_expression:
    SELECT ...   (Some valid select or union statement)
```

MySQL将表结构定义在`.frm`文件当中, 该文件位于每个数据库的磁盘目录下, 假如我们有`employees.titles`表, 那么`.frm`的位置就是`$DATADIR/employees/titles.frm`

以InnoDB为例如果使用独立表空间(`innodb_file_per_table=ON`)的话表中的**数据与相关的索引**就会储存在数据库目录下的`ibd`文件中, 如果使用的是系统表空间则会存放于`$DATADIR/ibdata*file`中.

### 临时表

使用`CREATE TEMPORARY TABLE`建临时表, 临时表仅在当前session可见, 并且会在session结束时自动删除. 这个特性就使得即使在两个session里建立相同的临时表也不会冲突. 其次, 已经存在的表会被隐藏, 直至临时表被删除, 也就是说就算有一个与现有的表重名也不会发生错误

### 表的克隆

- LIKE

  ```mysql
  CREATE TABLE new_tbl LIKE orig_tbl;
  ```

  创建一个`orig_tbl`的空表(no for views), 包括原来表中列的属性与索引

  如果`orig_tbl`是一个临时表, 那么要使用`CREATE TEMPORARY TABLE … LIKE`

  > NOTE: 在LOCK TABLES的情况下无法使用CREATE TABLE或CREATE TABLE … LIKE

- [AS] query_expression

  ```mysql
  mysql> CREATE TABLE test (a INT NOT NULL AUTO_INCREMENT,
      ->        PRIMARY KEY (a), KEY(b))
      ->        ENGINE=MyISAM SELECT b,c FROM test2;
  ```

  上面的Hello World我们使用了MyISAM引擎并定义了a, b和c三列.

  要注意的一点是, 来自`SELECT`的列是会**追加**到表上, 而不会覆盖, 看看下面的列子

  ```mysql
  mysql> SELECT * FROM foo;
  +---+
  | n |
  +---+
  | 1 |
  +---+

  mysql> CREATE TABLE bar (m INT) SELECT n FROM foo;
  Query OK, 1 row affected (0.02 sec)
  Records: 1  Duplicates: 0  Warnings: 0

  mysql> SELECT * FROM bar;
  +------+---+
  | m    | n |
  +------+---+
  | NULL | 1 |
  +------+---+
  1 row in set (0.00 sec)
  ```

  对于表`bar`来说, 每行的数据除了会来自foo表, 还会来自表上新的列的默认值

  > NOTE: 来自于 [`SELECT`](https://dev.mysql.com/doc/refman/5.7/en/select.html) 列的数据类型可被[`CREATE TABLE`](https://dev.mysql.com/doc/refman/5.7/en/create-table.html)中的定义所覆盖. 如果在克隆过程中有任何错误, 那么这张表将被不会被创建并会自动丢弃.

- IGNORE | REPLACE

  在`SELECT`语句前添加`IGNORE`或`REPLACE`

  参考 [Comparison of the IGNORE Keyword and Strict SQL Mode](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html#ignore-strict-comparison).

## 创建索引

```mysql
CREATE [UNIQUE|FULLTEXT|SPATIAL] INDEX index_name
    [index_type]
    ON tbl_name (index_col_name,...)
    [index_option]
    [algorithm_option | lock_option] ...

index_col_name:
    col_name [(length)] [ASC | DESC]

index_option:
    KEY_BLOCK_SIZE [=] value
  | index_type
  | WITH PARSER parser_name
  | COMMENT 'string'

index_type:
    USING {BTREE | HASH}

algorithm_option:
    ALGORITHM [=] {DEFAULT|INPLACE|COPY}

lock_option:
    LOCK [=] {DEFAULT|NONE|SHARED|EXCLUSIVE}
```

`CREATE INDEX`同样可以在`CREATE TABLE`的时候指定. 另外`CREATE INDEX`不可以用来创建`PRIMARY KEY`, 要使用`ALTER TABLE`来完成主键的添加

对于string类型的列, 可以建立前缀索引*col_name*(*length*)

- 前缀索引**可以**用于CHAR, VARCHAR, BINARY和VARBINARY
- 前缀索引**必须**用于BLOB和TEXT
- 使用`CREATE TABLE`, `ALTER TABLE`和`CREATE INDEX`创建前缀索引, 当应用于 nonbinary string types ([`CHAR`](https://dev.mysql.com/doc/refman/5.7/en/char.html), [`VARCHAR`](https://dev.mysql.com/doc/refman/5.7/en/char.html), [`TEXT`](https://dev.mysql.com/doc/refman/5.7/en/blob.html)) 会被解析成字符的数量, 应用于binary string types ([`BINARY`](https://dev.mysql.com/doc/refman/5.7/en/binary-varbinary.html), [`VARBINARY`](https://dev.mysql.com/doc/refman/5.7/en/binary-varbinary.html), [`BLOB`](https://dev.mysql.com/doc/refman/5.7/en/blob.html))就会被解析成字节的数量
- 对于空间类型的列, 前缀索引则无法支持(注意只是无法支持**前缀**索引)

下面给出创建前缀索引的例子

```mysql
CREATE INDEX part_of_name ON customer (name(10));
```

如果上面的`name`一列一般在前10个字符都不一样, 那么该前缀索引应该不会比完全所以慢太多. 而且, 使用前缀索引会节省大量的磁盘空间, 并间接优化了`INSERT`的插入速度

前缀索引所支持的最大长度是存储引擎所决定的, 如在InnoDB中一个前缀最长可到767字节, 如果开启了`innodb_large_prefix`则可以去到3072个字节

### 存储引擎索引特性

| Storage Engine | Index Type | Index Class | Stores NULL Values | Permits Multiple NULL Values | IS NULL Scan Type  | IS NOT NULL Scan Type |
| -------------- | ---------- | ----------- | ------------------ | ---------------------------- | ------------------ | --------------------- |
| `InnoDB`       | `BTREE`    | Primary key | No                 | No                           | N/A                | N/A                   |
| Unique         | Yes        | Yes         | Index              | Index                        |                    |                       |
| Key            | Yes        | Yes         | Index              | Index                        |                    |                       |
| Inapplicable   | `FULLTEXT` | Yes         | Yes                | Table                        | Table              |                       |
| Inapplicable   | `SPATIAL`  | No          | No                 | N/A                          | N/A                |                       |
| `MyISAM`       | `BTREE`    | Primary key | No                 | No                           | N/A                | N/A                   |
| Unique         | Yes        | Yes         | Index              | Index                        |                    |                       |
| Key            | Yes        | Yes         | Index              | Index                        |                    |                       |
| Inapplicable   | `FULLTEXT` | Yes         | Yes                | Table                        | Table              |                       |
| Inapplicable   | `SPATIAL`  | No          | No                 | N/A                          | N/A                |                       |
| `MEMORY`       | `HASH`     | Primary key | No                 | No                           | N/A                | N/A                   |
| Unique         | Yes        | Yes         | Index              | Index                        |                    |                       |
| Key            | Yes        | Yes         | Index              | Index                        |                    |                       |
| `BTREE`        | Primary    | No          | No                 | N/A                          | N/A                |                       |
| Unique         | Yes        | Yes         | Index              | Index                        |                    |                       |
| Key            | Yes        | Yes         | Index              | Index                        |                    |                       |
| `NDB`          | `BTREE`    | Primary key | No                 | No                           | Index              | Index                 |
| Unique         | Yes        | Yes         | Index              | Index                        |                    |                       |
| Key            | Yes        | Yes         | Index              | Index                        |                    |                       |
| `HASH`         | Primary    | No          | No                 | Table (see note 1)           | Table (see note 1) |                       |
| Unique         | Yes        | Yes         | Table (see note 1) | Table (see note 1)           |                    |                       |
| Key            | Yes        | Yes         | Table (see note 1) | Table (see note 1)           |                    |                       |

## 更新数据库

```mysql
ALTER {DATABASE | SCHEMA} [db_name]
    alter_specification ...
ALTER {DATABASE | SCHEMA} db_name
    UPGRADE DATA DIRECTORY NAME

alter_specification:
    [DEFAULT] CHARACTER SET [=] charset_name
  | [DEFAULT] COLLATE [=] collation_name
```

`ALTER DATABASE`能够改变数据库的属性, 这些属性存储在`$DATADIR/$YOUR_DB/db.opt`

DATABASE与SCHEMA是同义词

## 更新表

```mysql
ALTER [IGNORE] TABLE tbl_name
    [alter_specification [, alter_specification] ...]
    [partition_options]

alter_specification:
    table_options
  | ADD [COLUMN] col_name column_definition
        [FIRST | AFTER col_name ]
  | ADD [COLUMN] (col_name column_definition,...)
  | ADD {INDEX|KEY} [index_name]
        [index_type] (index_col_name,...) [index_option] ...
  | ADD [CONSTRAINT [symbol]] PRIMARY KEY
        [index_type] (index_col_name,...) [index_option] ...
  | ADD [CONSTRAINT [symbol]]
        UNIQUE [INDEX|KEY] [index_name]
        [index_type] (index_col_name,...) [index_option] ...
  | ADD FULLTEXT [INDEX|KEY] [index_name]
        (index_col_name,...) [index_option] ...
  | ADD SPATIAL [INDEX|KEY] [index_name]
        (index_col_name,...) [index_option] ...
  | ADD [CONSTRAINT [symbol]]
        FOREIGN KEY [index_name] (index_col_name,...)
        reference_definition
  | ALGORITHM [=] {DEFAULT|INPLACE|COPY}
  | ALTER [COLUMN] col_name {SET DEFAULT literal | DROP DEFAULT}
  | CHANGE [COLUMN] old_col_name new_col_name column_definition
        [FIRST|AFTER col_name]
  | LOCK [=] {DEFAULT|NONE|SHARED|EXCLUSIVE}
  | MODIFY [COLUMN] col_name column_definition
        [FIRST | AFTER col_name]
  | DROP [COLUMN] col_name
  | DROP PRIMARY KEY
  | DROP {INDEX|KEY} index_name
  | DROP FOREIGN KEY fk_symbol
  | DISABLE KEYS
  | ENABLE KEYS
  | RENAME [TO|AS] new_tbl_name
  | RENAME {INDEX|KEY} old_index_name TO new_index_name
  | ORDER BY col_name [, col_name] ...
  | CONVERT TO CHARACTER SET charset_name [COLLATE collation_name]
  | [DEFAULT] CHARACTER SET [=] charset_name [COLLATE [=] collation_name]
  | DISCARD TABLESPACE
  | IMPORT TABLESPACE
  | FORCE
  | {WITHOUT|WITH} VALIDATION
  | ADD PARTITION (partition_definition)
  | DROP PARTITION partition_names
  | DISCARD PARTITION {partition_names | ALL} TABLESPACE
  | IMPORT PARTITION {partition_names | ALL} TABLESPACE
  | TRUNCATE PARTITION {partition_names | ALL}
  | COALESCE PARTITION number
  | REORGANIZE PARTITION partition_names INTO (partition_definitions)
  | EXCHANGE PARTITION partition_name WITH TABLE tbl_name [{WITH|WITHOUT} VALIDATION]
  | ANALYZE PARTITION {partition_names | ALL}
  | CHECK PARTITION {partition_names | ALL}
  | OPTIMIZE PARTITION {partition_names | ALL}
  | REBUILD PARTITION {partition_names | ALL}
  | REPAIR PARTITION {partition_names | ALL}
  | REMOVE PARTITIONING
  | UPGRADE PARTITIONING

index_col_name:
    col_name [(length)] [ASC | DESC]

index_type:
    USING {BTREE | HASH}

index_option:
    KEY_BLOCK_SIZE [=] value
  | index_type
  | WITH PARSER parser_name
  | COMMENT 'string'

table_options:
    table_option [[,] table_option] ...  (see CREATE TABLE options)

partition_options:
    (see CREATE TABLE options)
```

`ALTER TABLE`可以改变表的结构. 如添加或者删除列, 创建或者删除索引, 更改已有列的类型, 重命名表名或列名, 更改表的存储引擎等.

- 使用`ALTER TABLE`需要`ALTER`, `CREATE`和`INSERT`权限.