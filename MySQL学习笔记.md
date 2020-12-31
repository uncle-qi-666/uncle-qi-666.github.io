##### 一、MariaDB基本教程

1.1 打开数据库表

```mysql
-- root登录
mysql -u root -p qige;
-- 展示数据库
show databases;
-- 打开classicmodels库
use classicmodels;
-- 展示当前库的所有表
show tables;
```

1.2 order by排序问题

```mysql
-- 多列排序时，orderby后以逗号分隔
select contactLastname, contactFirstname from customers order by contactLastname desc, contactFirstname asc limit 10;

-- field函数自定义列表，并按列表顺序排序
select orderNumber,status from orders order by FIELD(status, 'In Process', 'On Hold', 'Cancelled', 'Resolved', 'Disputed', 'Shipped') limit 10;
```

1.3 数据过滤问题

```mysql
-- 传统的where子句，select先检查所有行，再选择查询条件中的行
select * from table_name where search_condition; 

-- where与between结合，a <= x <= b
select firstName from employees where officeCode between 2 and 6;

-- where与like运算符结合, %表示任意个字符，_表示任意单个字符
select lastName, firstName from table_name where firstName like '%son%';
select lastName, firstName from table_name where firstName like '%Tsen_%';

-- where与regexp运算符结合， ^表示字符以某字符开头，$表示以某字符结尾
select lastName, firstName from table_name where firstName regexp '^P';
select lastName, firstName from table_name where firstName regexp 'son$';

-- where与in运算符结合， 
select lastName, firstName from table_name where id in (value1,value2);

-- where与 IS NULL结合，数据库中，NULL是个标记，指一条信息丢失或未知，不等于0或空字符
select lastName, firstName, reportsTo from employees where reportsTo IS NULL;

select lastName from employees where jobTitle <> 'Sales Rep';
```

1.4 数据去重问题

```mysql
-- 单行去重
select distinct lastName from employees order by lastName limit 10;

-- 如果该列中包含了NULL值，也会被去重，因为distinct将NULL值视为相同的值
-- 多列去重，distinct 列A，列B    将会筛选出AB的唯一组合；
select distinct state, city from customers where state IS NOT NULL order by state, city;

-- distinct与group by，当select一个列时，没啥区别，筛选多个列时，结果不一样；
-- distinct与聚合函数（sum、avg、count）
select count(distinct state) from customers where country='usa';
```



##### 二、实践中细节

2.1 关于insert into ... select的锁表问题：在锁期间不允许任何操作（保证一致性）

```mysql
-- case1:不带查询、不带排序     ans1:MySql逐行加锁
insert into table1 select * from table2;
-- case2:带排序，使用主键排序   ans2:MySql逐行加锁
insert into table1 select * from table2 order by id;
-- case3:带排序，使用非主键排序  ans3:MySql直接锁整个表
insert into table1 select * from table2 order by other_id;
-- case4:带查询，使用非主键查询  ans4:MySql逐行加锁
insert into table1 select * from table2 where other_id >= '666';
```

2.2 清空数据表

```mysql
https://segmentfault.com/a/1190000019087348
-- delete清空整个表数据,但是自增ID不会还原,而是从删除前最后的ID继续自增。


-- truncate则会将自增ID一同还原为初始值，效率更快，保留表结构，所有状态相当于新表
```









##### 三、数据库基本原理概念

https://www.cnblogs.com/souyoulang/p/11113652.html

3.1 InnoDB缓存设计

​      MySQL的数据存储在物理磁盘上，数据处理在内存中，为了减少I/O，提高性能，InnoDB将数据划分为若干页，`以页作为磁盘和内存交互的基本单位`， 页的大小为16k，其设计认为一条被使用的数据，其附近的数据也大概率会被使用。

3.2 事务（使用Innodb引擎的数据库或表支持事务）

​	   将多条SQL语句作为一个整体进行操作的功能，被称为数据库**事务**，具备ACID四个特性：

- Atomic，原子性，将所有SQL作为原子工作单元执行，要么全部执行，要么全部不执行；
- Consistent，一致性，指系统从一个正确的状态迁移到另一个正确的状态。正确的状态指满足预定约束的状态，无论事务前后；
- Isolation，隔离性，如果有多个事务并发执行，每个事务作出的修改必须与其他事务隔离；
- Duration，持久性，即事务完成后，对数据库的修改被持久化存储；

```mysql
-- 使用begin或start transaction开始事务，commit提交事务，rollback回滚事务，savepoint savepoint_name允许在事务中创建保存点，而release savepoint savepoint_name可删除保存点，rollback to savepoint_name回滚到标记点，set transaction设置事务隔离级别；

-- test表中会被插入一条记录
begin；
insert into test value(1);
commit;

-- test表中不会被插入数据，因为事务回滚，操作取消
begin；
insert into test value(2);
rollback;
```

隔离级别：可通过`set transaction = 级别` 来设置，Innodb引擎下支持四种类别：默认级别`Repeatable Read`

- read uncommitted：A事务会读取B事务更新后但未提交的数据，如果B事务回滚，则A事务读取到的就是脏数据，俗称脏读；
- read committed：A事务会多次读取同一数据，在事务没结束时，若B事务恰好修改了该数据，修改后，A事务读取的结果会发生变化，不可重复读；
- Repeatable  read：在A事务查询某条记录时发现没有，此时B事务插入了该条记录，当A事务更新该条记录并再次查询竟能查到。俗称幻读；（此处A事务必须先更新才能查询到，因为在该级别下，事务对同一数据的读取都是一致的，所以即使B事务插入记录，A直接读取的结果与之前仍然一致，只有更新后才能查询到B事务commit的数据）
- serializable：最严格隔离级别，所有事务顺序串行执行，因此脏读、不可重复读、幻读均不会出现，但效率降低，失去了事务的隔离特性，一般不用；

不可重复读的重点是修改数据，幻读的重点是增删数据；