# 常见SQL

## 1、DDL

* 创建与删除数据库

  ~~~sql
  create database if not exists db1;
  DROP DATABASE db1;
  ~~~

* 创建表(当列为unique时，可以存在多条null值)

  ~~~sql
  create table test1(
  	id int primary key auto_increment,
      name varchar(32) unique not null,
  	age int 
  );
  ~~~

* 删除表

  ~~~sql
  drop table u1;
  ~~~

* 添加列

  ~~~sql
  alter table u1 add [column] address varchar(15) [after name]/[first]; 
  ~~~

* 删除列

  ~~~sql
  alter table u1 drop address;
  ~~~

* 修改列

  ~~~sql
  -- modify关键字只能修改列的类型或者长度，不能修改列的名称
  alter table u1 modify address varchar(32);
  -- change关键字既可以修改列名，又可以修改列的属性
  alter table u1 change address grade int;
  ~~~

* 查询表中所有列的属性

  ~~~sql
  desc u1；
  ~~~

## 2、DML

* 向表中插入数据

  ~~~sql
  insert into 表名 (id, name, age, ...) values (1, 'zhangsan', 10, ...);
  ~~~

* 删除数据

  ~~~sql
  delete from 表名 where id = 1;
  -- 全表清除使用truncate关键字
  truncate table u1;
  ~~~

* 更新数据

  ~~~sql
  update u1 set name = 'lisi' where id = 1;
  ~~~

## 3、DQL

* 普通查

  ~~~sql
  select * from user;
  ~~~

* 条件查询

  ~~~sql
  select * from user where id = 1;
  ~~~

* 范围查询

  ~~~sql
  select * from emp where entry_date between '2015-09-01' and '2022-09-01'
  ~~~

* 分组查询

  ~~~sql
  select count(*), dept_id from emp group by dept_id;
  ~~~

* 分页查询

  ~~~sql
  select * from user limit 0, 10;
  ~~~

* 分组后条件查询

  ~~~sql
  SELECT 
  	count(1) AS count,
  	(CASE
  			WHEN age > 50 THEN '老年'
  		WHEN age > 30 THEN '中年'
  		WHEN age > 18 THEN '青年'
  		ELSE '少年'
  	END) AS ag
  FROM
  	USER
  GROUP BY
  	ag
  HAVING 
  	count > 2
  ORDER BY
  	ag DESC;
  ~~~

  