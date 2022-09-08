# 锁、InnoDB引擎高级

## 1. 锁

### 1.1 全局锁

**定义**

* 全局锁会锁所有库中的所有表，<font color=red>**被锁的表只能进行查询操作，所有写操作都会阻塞。**</font>

**使用**

~~~sql
-- 创建全局锁
flush tables WITH READ LOCK;

-- 用途
-- 执行数据库备份操作，如果不是备份本地的数据库，就需要使用-h命令配置MySQL的主机地址
mysqldump [-h 192.168.130.128] -uroot -proot test > D:/xiongyangyang/documents/SQL转储/test.sql

-- 关闭全局锁
unlock tables;
~~~

### 1.2 表锁

**<font color=red>定义</font>**

* 表锁会指定锁住哪一个表，不会影响到其他的表

**<font color=red>读锁</font>**

* 使用读锁会导致当前连接和其他连接都只能进行数据的读取操作，不能进行写入操作

  ~~~sql
  -- 创建读锁
  lock tables user read;
  -- 释放读锁
  unlock tables;
  ~~~

**<font color=red>写锁</font>**

* 使用写锁会导致其他连接既不能进行读操作，也不能进行写操作，但是当前连接可以进行读写操作

  ~~~sql
  -- 创建写锁
  lock tables user write;
  -- 释放写锁
  unlock tables;
  ~~~

**<font color=red>元数据锁（MDL）</font>**

* meta data lock，保护表结构的锁，当一个会话对表进行增删改查时，会自动添加元数据的读锁，这个读锁指的就是不能对表结构进行修改，其他的操作可以进行，所以其他的会话可以同时进行增删改的操作，但是与此同时，**如果一个未提交事务的会话开启了元数据的读锁，另一个会话准备进行对表结构进行修改的情况，修改表结构的会话就会阻塞直到另一个会话提交事务。因为MDL的读锁和写锁是互斥的。**

  ~~~sql
  -- 查看元数据表数据sql
  SELECT
  	object_type,
  	object_schema,
  	object_name,
  	lock_type,
  	lock_duration
  FROM
  	performance_schema.metadata_locks ;
  ~~~


**<font color=red>意向锁</font>**

* 用来解决某个事务想要给某一个表添加表锁的时候需要去判断这个表中每一行是否添加了行锁的性能消耗

* 意向共享锁（Intention Share，简称IS），表示某个会话执行的是<font color=red>select类型的SQL且后面添加了lock in share mode，如果没有添加就不会创建意向锁</font>，意向共享锁和表锁的读锁是兼容的，和写锁是互斥的。

* 意向排他锁（Intention  eXclusive，简称IX），表示某个会话执行的是写操作的类型的SQL，它和表锁中的读锁和写锁都是互斥的。

  ~~~sql
  -- 查看意向锁表数据sql
  SELECT
  	object_schema,
  	object_name,
  	lock_type,
  	lock_mode,
  	lock_data
  FROM
  	performance_schema.data_locks ;
  ~~~

  

### 1.3 行锁

* 定义
  加锁之后只会导致某一行数据加锁，其他行的数据不会受到影响，它是粒度最小的锁，并发度最高，应用于InnoDB引擎中
  
* 分类
  * 记录锁（Record Lock）
    锁定单行记录。在RC（读已提交）、RR（可重复读）事务隔离级别中使用，主要针对update和delete的写操作。查询操作后加lock in share mode会添加共享行锁，加for update会添加排他行锁，什么都不加不会添加行锁。
  * 间隙锁（Gap Lock）
    锁定索引记录之间的间隙，从给定值到上一个索引值之间，不会锁索引值的数据。在RR事务隔离级别下使用，主要针对insert操作的操作，防止产生幻读。
  * 临键锁（Next-Key Lock）
    行锁加上间隙锁组成临键锁，RR隔离级别使用。
  
* 具体情景分析

  * 唯一索引列
    当条件包含索引列时，命中条件后添加记录锁，只会锁住单条记录，并且是读锁，其他数据可以访问但是不能修改，没有命中条件时添加的是间隙锁，锁住当前条件附近的两个索引之间的间隙，此时可以对间隙之外的数据进行写操作，也能对两个间隙之间的数据进行操作（不改变当前索引值和主键的情况下）。如果修改当前索引值只能往间隙之外修改，不能将索引值修改为间隙之内。

  * 非唯一索引
    当条件命中索引值时，不仅会给命中的记录加记录锁，还会给左边添加临键锁，右边加上间隙锁。

    如果没有命中索引值，添加的也是一个间隙锁，情况和唯一索引列未命中是一致。

  * 无索引
    加锁的话直接转换成共享表锁，只能进行数据读取，不能进行写操作

* 记录锁分类
  * 共享锁
    当某个共享锁锁住了一行数据，其他的共享锁可以同时获得该行的锁，但是排他锁不能获取该行的锁
  * 排他锁
    当一个事务使用排他锁锁住一行数据，其他事务在该行既不能添加共享锁，也不能添加排他锁
  * 行锁对应的具体情形
    ![1662469951947](C:\Users\19816\AppData\Roaming\Typora\typora-user-images\1662469951947.png)

## 2. InnoDB引擎高级

