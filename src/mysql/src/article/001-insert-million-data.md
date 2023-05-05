# 使用存储过程向 Mysql 中插入百万数据

移动互联网的蓬勃发展，用户规模轻松超百万,在程序设计之初需要考虑后续程序架构的演进方向。本文通过存储过程生成百万数据，为后续性能的分析打好基础。


## 1 新建用户表
建立一个简单的用户表，`id`为自增逐渐, `account_id` 用来模仿外键，`user_name`为用户名。

```sql
create table `users`(
    `id` int auto_increment,
    `account_id` int,
    `user_name` varchar(23) default null comment "用户名",
     primary key (`id`),
     key `idx_account_id` (`account_id`)
);
```

## 2 存储过程

 Sql 添加了编程能力，具有变量、条件、循环语句，并可以封装成函数，函数可以有参数。可以把存储过程简单等价于函数。

### 2.1 新建存储过程
`DELIMITER` 用来设定分割符号，默认为; 。在这里设置为 // ，因为存储过程中包含;。`SET autocommit = 0;` 禁用自动提交，加快插入速度。注意每个语句要加;。否则会报语法问题。

```sql

DELIMITER //
CREATE PROCEDURE AddUsers()
BEGIN
  DECLARE v1 INT DEFAULT 1;
  SET autocommit = 0;
  WHILE v1 < 1000000 DO
    insert into `users` (`user_name`,`account_id`)
    values(concat("liwei",v1), v1);
    SET v1 = v1 + 1;
  END WHILE;
END;//
DELIMITER ;

```

### 2.2 执行存储过程

```sql
CALL AddUsers();
```

### 2.3 删除存储过程

```sql
DROP PROCEDURE IF EXISTS AddUsers

```


### 2.4 存储过程列表
`information_schema.routines`表里可查询定义的存储过程。

```sql
SELECT routine_name FROM     information_schema.routines WHERE     routine_type = 'PROCEDURE' and routine_schema='test';
```

### 2.5 存储过程详情

```sql
SHOW CREATE PROCEDURE AddUsers \G;
```

### 2.6 查看插入数据

```sql
select count(*) from users;
```

## 3 总结

本文建立简单表，新建存储过程，并执行。向表中插入了百万数据。
