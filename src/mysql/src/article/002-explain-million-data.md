# explain 分析百万级别数据表

本文首先新建用户表和订单表，使用存储过程向表中插入百万级数据。然后学习 explain 的使用。

## 建表

- 用户表

    ```sql

    CREATE TABLE `user_info` (
      `id`   BIGINT(20)  NOT NULL AUTO_INCREMENT,
      `name` VARCHAR(50) NOT NULL DEFAULT '',
      `age`  INT(11)              DEFAULT NULL,
      PRIMARY KEY (`id`),
      KEY `name_index` (`name`)
    )ENGINE = InnoDB DEFAULT CHARSET = utf8;

    ```

- 向用户表插入百万数据
    ```sql
    DELIMITER //
    CREATE PROCEDURE AddUsers()
    BEGIN
      DECLARE v1 INT DEFAULT 1;
      SET autocommit = 0;
      WHILE v1 < 1000000 DO
        insert into `user_info` (`name`,`age`)
        values(concat("liwei",v1), v1 % 99 + 1);
        SET v1 = v1 + 1;
      END WHILE;
    END;//
    DELIMITER ;
    call AddUsers();

    ```


- 订单表
    ```sql 
    CREATE TABLE `order_info` (
      `id`           BIGINT(20)  NOT NULL AUTO_INCREMENT,
      `user_id`      BIGINT(20)           DEFAULT NULL,
      `product_name` VARCHAR(50) NOT NULL DEFAULT '',
      `product_id`   BIGINT(20)  NOT NULL  DEFAULT 0,
      PRIMARY KEY (`id`),
      KEY `idx_userid_productid_productname` (`user_id`, `product_id`, `product_name`)
    ) ENGINE = InnoDB DEFAULT CHARSET = utf8;
    ```

- 向订单表插入百万数据

    ```sql
    DELIMITER //
    CREATE PROCEDURE AddOrder(IN `userCount` INT)
    BEGIN
        DECLARE productBegin INT DEFAULT 1;
        DECLARE productEnd INT DEFAULT 8;
        DECLARE v1 INT DEFAULT 1;
        DECLARE v2 INT DEFAULT 1;
        DECLARE randomVal INT DEFAULT 1;
        SET autocommit = 0;
        WHILE v1 <= userCount DO
            SET v2 = 1;
            SELECT FLOOR(productBegin + RAND()*(productEnd - productBegin)) into randomVal;
            WHILE v2 <= randomVal DO
                INSERT INTO order_info (user_id, product_id, product_name) VALUES (v1, v2, concat("p",v2));
                SET v2 = v2 +1;
            END WHILE;
            SET v1 = v1 + 1;
        END WHILE;
    END;//
    DELIMITER ;
    call AddOrder(1000000);
    ```

## EXPLAIN

要使用 explain, 只需在查询的`SELECT` 关键词词前加 `explain` 这个词。

```sql
mysql> explain select * from user_info where id = 1 \G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user_info
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 8
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

| 列            | 用途                                                          | 备注 |
|---------------|---------------------------------------------------------------|------|
| id            | 查询语句的标识符。 每个 SELECT 都会自动分配一个唯一的标识符。 |      |
| select_type   | 查询语句的类型。                                              |      |
| table         | 表                                                            |      |
| type          | 索引访问类型                                                  |      |
| possible_keys | 可能选用的索引                                                |      |
| key           | 使用到的索引                                                  |      |
| key_len       | 使用到索引的长度                                              |      |
| ref           | 具体使用的索引列                                              |      |
| rows          | 估计扫描行数                                                  |      |
| Extra         | 备注信息                                                      |      |

下文主要分析type和key_len。type 涉及索引的性能。key_len涉及索引列的选择。

### type  
通常来说, 不同的 type 类型的性能关系如下: ALL < index < range ~ index_merge < ref < eq_ref < const < system。下面分析下常见的类型。

- const  
  针对主键或唯一索引的等值查询扫描, 最多只返回一行数据. const 查询速度非常快, 因为它仅仅读取一次即可.
  ```sql
    mysql> explain select * from user_info where id = 2\G
    *************************** 1. row ***************************
               id: 1
      select_type: SIMPLE
            table: user_info
       partitions: NULL
             type: const
    possible_keys: PRIMARY
              key: PRIMARY
          key_len: 8
              ref: const
             rows: 1
         filtered: 100.00
            Extra: NULL
    1 row in set, 1 warning (0.02 sec)

  ```
- ref  
  此类型通常出现在多表的 join 查询, 针对于非唯一或非主键索引, 或者是使用了 最左前缀 规则索引的查询.
  ```sql 
    mysql> EXPLAIN SELECT * FROM user_info, order_info WHERE user_info.id = order_info.user_id AND order_info.user_id = 5\G
    *************************** 1. row ***************************
               id: 1
      select_type: SIMPLE
            table: user_info
       partitions: NULL
             type: const
    possible_keys: PRIMARY
              key: PRIMARY
          key_len: 8
              ref: const
             rows: 1
         filtered: 100.00
            Extra: NULL
    *************************** 2. row ***************************
               id: 1
      select_type: SIMPLE
            table: order_info
       partitions: NULL
             type: ref
    possible_keys: idx_userid_productid_productname,idx_user_id
              key: idx_userid_productid_productname
          key_len: 9
              ref: const
             rows: 9
         filtered: 100.00
            Extra: Using index
    2 rows in set, 1 warning (0.00 sec)
  ```
- range  
    表示使用索引范围查询, 通过索引字段范围获取表中部分数据记录. 这个类型通常出现在 =, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, IN() 操作中.
    ```sql 
    mysql> explain select * from user_info where id between 2 and 8 \G;
    *************************** 1. row ***************************
               id: 1
      select_type: SIMPLE
            table: user_info
       partitions: NULL
             type: range
    possible_keys: PRIMARY
              key: PRIMARY
          key_len: 8
              ref: NULL
             rows: 7
         filtered: 100.00
            Extra: Using where
    1 row in set, 1 warning (0.00 sec)
    ```

- index
    表示全索引扫描(full index scan), 和 ALL 类型类似, 只不过 ALL 类型是全表扫描, 而 index 类型则仅仅扫描所有的索引, 而不扫描数据.Extra 字段 会显示 Using index.
    ```sql 
    mysql> EXPLAIN SELECT name FROM  user_info \G
    *************************** 1. row ***************************
               id: 1
      select_type: SIMPLE
            table: user_info
       partitions: NULL
             type: index
    possible_keys: NULL
              key: name_index
          key_len: 152
              ref: NULL
             rows: 998420
         filtered: 100.00
            Extra: Using index
    1 row in set, 1 warning (0.00 sec)
    ```
- all 
  表示全表扫描, 不使用索引。

  ```sql 
    mysql>  EXPLAIN SELECT age FROM  user_info WHERE age = 20 \G
    *************************** 1. row ***************************
               id: 1
      select_type: SIMPLE
            table: user_info
       partitions: NULL
             type: ALL
    possible_keys: NULL
              key: NULL
          key_len: NULL
              ref: NULL
             rows: 998420
         filtered: 10.00
            Extra: Using where
    1 row in set, 1 warning (0.00 sec)
  ```

  百万用户表全表访问，需要200ms。是不可接受的。

  ```sql 
    mysql> SELECT age FROM  user_info WHERE age = 101;
    Empty set (0.23 sec)
  ```


### key_len
表示查询优化器使用了索引的字节数. 这个字段可以评估组合索引是否完全被使用, 或只有最左部分字段被使用到.
key_len 的计算规则如下:

- 字符串
  - char(n): n 字节长度
  - varchar(n): 如果是 utf8 编码, 则是 3 n + 2字节; 如果是 utf8mb4 编码, 则是 4 n + 2 字节.
- 数值类型:
  - TINYINT: 1字节
  - SMALLINT: 2字节
  - MEDIUMINT: 3字节
  - INT: 4字节
  - BIGINT: 8字节
- 时间类型
  - DATE: 3字节
  - TIMESTAMP: 4字节
  - DATETIME: 8字节
- 字段属性: 
  - NULL 属性 占用一个字节
  - NOT NULL 的, 不占用字节


举个列子  
例子1 
```sql 
mysql>  EXPLAIN SELECT * FROM order_info WHERE user_id < 3 AND product_name = 'p1' AND product_id = 1 \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: order_info
   partitions: NULL
         type: range
possible_keys: idx_userid_productid_productname,idx_user_id,idx_product_id
          key: idx_userid_productid_productname
      key_len: 9
          ref: NULL
         rows: 11
     filtered: 0.45
        Extra: Using where; Using index
1 row in set, 1 warning (0.01 sec)
```

 因为先进行 user_id 的范围查询, 而根据 最左前缀匹配 原则, 当遇到范围查询时, 就停止索引的匹配, 因此实际上我们使用到的索引的字段只有 user_id, 因此在 EXPLAIN 中, 显示的 key_len 为 9. 因为 user_id 字段是 BIGINT, 占用 8 字节, 而 NULL 属性占用一个字节, 因此总共是 9 个字节. 若我们将user_id 字段改为 BIGINT(20) NOT NULL DEFAULT '0', 则 key_length 应该是8.


列子2  
```sql 
mysql> EXPLAIN SELECT * FROM order_info WHERE user_id = 1 AND product_id = 1 \G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: order_info
   partitions: NULL
         type: ref
possible_keys: idx_userid_productid_productname,idx_user_id,idx_product_id
          key: idx_userid_productid_productname
      key_len: 17
          ref: const,const
         rows: 1
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.01 sec)
```
命中`idx_userid_productid_productname`, 根据最左前缀匹配原则，命中user_id 和 product_id。user_id和product_id都为bigint, 占用16个字节。由于user_id有个NULL属性，增加一个字节，key_len为17个字节。






## 总结
本文首先使用存储过程向用户表和订单表插入百万级别数据。之后主要分析 explain 中的 type和key_len, 通过type，可以知道本次使用索引的性能。通过分析key_len和key, 可以分析联合索引中具体使用了哪些前缀索引。其中还给出了百万用户下，全表查询的性能，超过200ms, 全表扫描是不可接受的。对于explain的其它属性，需后续继续探索。


## 参考

- [1][MySQL 性能优化神器 Explain 使用分析](https://segmentfault.com/a/1190000008131735)
- [2] 高性能 Mysql 附录D
