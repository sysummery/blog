---
title: Innodb中的锁
date: 2019-12-07 11:47:11
tags:
    - mysql
    - innodb
photos:
    - ["http://sysummery.top/lock_cover.jpg"]
---
从粒度上来分可以分为**行锁**、**页锁**、**表锁**; 从“性格”上分可以分为**乐观锁**与**悲观锁**; 从粒度上的分类应该很容易理解，不再敖述。所以是说说锁在性格上的分类。
<!--more-->

# 1. 锁的分类

## 1.1 乐观锁
用数据版本记录机制实现，是实现乐观锁的最常用的方式。每次取出一条数据的时候同时也会读取出这行数据的版本version；每次更新数据的时候也会把相应的版本version加一。如果更新的时候发现数据的版本号与第一次取出来的不一样，那么肯定是有别的事务更新了数据，就不更新；如果前后版本号一样，则更新。

## 1.2 悲观锁

### 1.2.1 共享锁
简称s锁，可以允许用户并发的读取数据。共享锁之间可以兼容，共享锁与排它锁不兼容

```sql
# 行粒度的共享锁
select * from students where id =1 in share mode;

# 表粒度的共享锁
lock table students read;
```

### 1.2.2 排它锁
简称x锁，排它锁之间不兼容，排它锁与共享锁也不兼容

```sql
# 行粒度的排它锁
select * from students where id=1 for update;

# 表粒度的排他锁
lock table students write;
```
**需要注意的是，如果在一个事务里面锁住了一张表，即使事务提交了表锁仍然存在必须手动调用**`unlock tables`**命令释放表锁。**

# 2. innodb中的锁
根据上面锁的分类，我们可以立马知道innodb里面有**共享行锁**、**共享表锁**、**排他行锁**、**排他表锁**。

## 2.1 innodb中的行锁
行锁有时候锁住的可能不止是一行而是一个**间隙**。按行锁锁住的范围来说可以分为三种

* Record Lock 单个行记录上的锁，只锁住一行
* Gap Lock 锁住两个边界值之间的间隙，但是不包括两个边界值
* Next-Key Lock 锁住一个间隙和间隙的右边界


**行锁是通过锁住索引的方式来锁住记录的。**如果在一个事务中使用`select ... for update`或者`select ... in share mode`显示的获取行锁的时候，如果查询没有命中索引，那么会锁住整张表！因此下面我们从索引的角度并且事务的隔离级别为Repeatable Read下的情况来研究行锁是怎么工作的。

### 2.1.1 主键（唯一）索引
一般在查询的条件使用了主键索引或者唯一索引的值去查询的时候只会为查找到的记录上锁。比如直接用主键去获取一条数据
```sql
select * from students where id=10 for update;
```
如果是in查询比如
```sql
select * students where id in (10,20,30) for update;
```
也会锁住匹配的行数据不会锁住任何的间隙
但是如果是主键或唯一索引的范围查询那么不仅会锁住一行数据还会锁住一个间隙，比如我有一张score表
![](http://sysummery.top/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_c712bf1b-119b-4309-ab36-0251fa0538ae.png)
里面只有两条数据id分别为35和37

| 事务1 | 事务2 |
| --- | --- |
|select * from score where id>=35 and id<37 for update; |  |
|  | select * from score where id=35 for update;(这条sql语句被阻塞，说明id=35的数据被Record Lock锁住了) |
| commit；(提交事务释放锁) |  |
|  | 由于事务1提交了，相应的锁也释放了，事务2阻塞结束 |
| begin； |  |
| select * from score where id>=35 and id<37 for update;（与之前的sql是相同的） |  |
|  | select * from score where id=37 for update;(这条sql语句没有被阻塞，说明id=37的数据没有被Record Lock锁住) |
|  | insert into score(id,student_id,subject_id,score)value(36,1,1,90);(这条sql语句被阻塞了，因为id=36位于id=35和id=37之间) |
| commit； |  |
|  | 阻塞结束，成功插入了语句 |
|  | commit；|

结论

* 如果只是使用一个或者几个主键（或者唯一索引）获取锁的时候，只会锁住记录不会锁住间隙
* 如果是使用主键或者唯一索引范围查询，会锁住这两个值之间的间隙（Gap Lock）。锁不锁住边界值取决于查询里面有没有等于边界值

### 2.1.2 普通索引
现在score表变成了这样

![](http://sysummery.top/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_96c4d877-3f2e-4bd2-a3cd-486e2233b1ea.png)



| 事务1 | 事务2 |
| --- | --- |
| begin; | begin; |
|select * from score where score=87 for update;||
|  | insert into score(id,student_id,subject_id,score)value(7,1,7,86);（sql被阻塞） |
| commit; |  |
|  | 阻塞结束，成功插入了数据； |
|  | rollback;（不保存插入的结果） |
| begin; |  |
| select * from score where score=87 for update;（与上一条sql一样） |  |
|  | insert into score(id,student_id,subject_id,score)value(8,1,7,88);（sql被阻塞） |
| commit; |  |
|  | 阻塞结束，成功插入了数据； |
|  | rollback；(不保存插入的结果) |
| begin; |  |
| select * from score where score=87 for update;（与上一条sql一样） |  |
|  | insert into score(id,student_id,subject_id,score)value(10,2,2,89);（没有被阻塞插入成功）|
|  | insert into score(id,student_id,subject_id,score)value(11,2,2,84);（没有被阻塞插入成功） |
|  | select * from score where id=2 for update;(sql被阻塞) |
| commit; |  |
|  | 阻塞结束 |
|  | commit; |

结论

* 当使用普通索引（score=87）获取锁的时候首先会在索引值的上区间加上一个Next-Key Lock锁，即锁住(84,87]这个区间，同时会为下区间加上一个Gap Lock，即锁住(84,89)这个区间
* 使用普通索引，同时会使用Record Lock锁住主键索引。这可能也是因为所有的非主键索引的查询最后都会转化成使用主键索引查询，因为innodb的数据是在主键索引的叶子节点上。

## 2.2 innodb中的表锁
innodb除了拥有mysql数据库级别的表锁外，还有另外两个表锁

* 意向共享锁
* 意向排它锁

这两个表锁有啥作用？设想一下当我想给一张表加一个排他锁的时候，那么我得检查以下三项

1. 该表有没有共享锁
2. 该表有没有排它锁
3. 该表的某（几）条记录有没有共享锁
4. 该表的某（几）条记录有没有排它锁

这其中1和2是比较好确定的，但是3、4怎么确定？锁是属于一个事务的不是属于一张表的，难道一条条数据的检查？显然是不行的，意向锁就应运而生。看一下意向锁与其他锁的兼容情况

|  | 意向共享锁 | 意向排它锁 | 行级共享锁 | 行级排它锁 | 表级共享锁 | 表级排它锁 |
| --- | --- | --- | --- | --- | --- | --- |
| 意向共享锁 | 兼容 | 兼容 | 兼容 | 兼容 | 兼容 | 互斥 |
| 意向排它锁 | 兼容 | 兼容 | 兼容 | 兼容 | 互斥 | 互斥 |

意向锁与意向锁、行级锁都是兼容的。表级锁中只与共享锁兼容。好像意向锁就是为了"抵抗"表级锁而生。

意向锁的添加是innodb来完成的，我们没有办法干预。举个例子，事务1想为数据a加了一个排它锁，那么在为这行数据加锁前会为整张表加一张**意向排它锁**，这时事务2想为数据b加一个共享锁，那么会先为表加一个意向共享锁，因为意向锁之间兼容，所以事务2不会阻塞，共享锁也会加到数据b上。这时事务3想为整个数据表加一个表级的排他锁，因为表级排它锁与意向锁相排斥所以事务3会被阻塞，直至事务1和事务2提交或者回滚。

# 3. innodb中的事务与锁
之前在2.1.1和2.2.2中讨论的内容在事务中同样适用。正式因为Gap Lock和Next-Key Lock的存在才会使innodb在Repeatable Read下能够避免幻读。

## 3.1 一致性非锁定度
也叫快照读。在一个事务中同一个查询得到的结果总是相同的。这个是因为数据读取的是数据的一个版本。这个版本可能是表中的数据（最新的版本），也可能是undo log中的该行的一个历史版本。读取的时候是不会加任何的锁。比如
```sql
select * from score;
```
那么什么时候直接读取表的记录？又什么时候去undo log找历史版本呢？这与最后更改数据的事务id与当前活跃的事务id数组的关系决定的

1. 如果最后修改数据行的事务id比所有的活跃的事务id都小，那么这行数据对当前事务是可见的
2. 如果最后修改数据行的事务id比所有活跃的事务id大，那么是不可见的要去undo log中找合适的版本
3. 如果等于当前事务的id，当然是可见的
4. 如果即不大于也不小于，那么要遍历活跃事务id数组如果最后修改数据行的事务id在这里面就对当前事务不可见，如果不在说明已经commit了就可见

## 3.2 一致性锁定读
也叫当前读。当事务中执行 select...for update, insert, update, delete操作的时候读取的就是数据表当前的数据（最新的版本）并同时加上行级排它锁。

# 4. 自增锁
自增锁不是innodb独有的，很多存储引擎都可以为一张表的某一列设置成auto increment，innodb也不例外。因为平时工作中所有表的id都是auto increment的，因此想学习一下自增锁。

一个参数innodb_autoinc_lock_mode控制着在向auto_increment列的表插入数据时，相关的锁行为。
```sql
show  variables like 'innodb_autoinc_lock_mode'
```

这个参数有三个值

* 0，这个模式下在插入语句开始的时候会得到一个表级别的锁，语句结束则释放。**注意：是语句开始的时候加锁结束的时候释放锁**
* 1，这个是mysql默认的值，可以理解成插入的时候先锁住这个“自增值”，然后拿到这个自增值作为插入记录的id，然后再把这个自增值的值加一，最后释放锁。
* 2，没有锁
