---
title: Mysql中的日志文件
date: 2019-11-17 08:52:53
tags:
    - mysql
    - innodb
photos:
    - ["http://sysummerblog.club/mysql_cover.jpg"]
---
mysql中有很多重要的日志文件，这些日志文件主要可以分为两大类：服务器的日志文件以及存储引擎的日志文件。平时我们用的存储引擎都是innodb，因此今天介绍的存储引擎的日志文件也是innodb的日志文件。
<!--more-->
## 服务器日志文件
* error log 错误日志
* slow query log 慢查询日志
* bin log 二进制日志
* relay log 中继日志


## innodb存储引擎的日志

* redo log 重做日志
* undo log 撤回日志

### 1.1 error log
error log存在于数据库的数据根目录下。用于记录mysql的启动、运行、关闭的过程中出现的错误和警告。在mysql启动异常的时候尤其有用。可以通过以下的命令来查看error log的具体的位置

```sql
 show variables like 'log_error'\G
```
### 1.2 slow query log
slow query log 记录了所有查询时间大于(不包括等于)查询阈值的sql语句。可以通过以下命令获取查询阈值

```sql
show variables like 'long_query_time'\G
```
另一个和慢查询日志有关的参数是`log_queries_not_using_indexes`,表示如果查询没有用到索引也会被记录到慢查询日志当中去，无论有没有超过查询阈值。

### 1.3 bin log
bin log记录的mysql所有数据库的所有数据表的更改的所有操作，无论使用的是什么存储引擎。他也是唯一一个使用二进制记录的日志。如果在更新一条记录的一个字段的时候是用原值更新的那么这条记录并没有改变但是还是会写入bin log。

bin log主要用来复制，在主库上源源不断的产生bin log并同步给各个从库来保证主从之间的数据一致。

bin log也可以以支持point-in-time的数据恢复。

bin log 的格式主要有三种
1. row 基于每一行
2. statement 基于sql语句
3. mixed 既有row形式的又有statement形式的

### 1.4 relay log
在从库中会有一个专门的IO线程不断的从主库中读取bin log并把读取的内容写入relay log中。然后会有另外的线程会将新产生的relay log重放到从库中。主从复制的流程大致为

![](http://sysummerblog.club/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_cde5d8b5-3035-4efa-bb0b-367394045f75.png)
### 2.1 redo log
存在于共享表空间中，是**物理日志**，记录的是一个事务中数据页的变化信息，不是某行的数据的变化信息。

用于保证事务的持久性。在使用innodb存储引擎的时候，当一页数据被更新时首先在innodb的缓冲池中进行更新，然后才会根据一定的策略异步刷新回磁盘。那么问题来了，如果缓冲池中的数据页还未来的急刷新回磁盘服务器就宕机了怎么办？innodb是采用的日志先于数据刷新到磁盘的策略。

每当一个事务写入数据的时候，innodb会向redo log缓存里写入**数据页**的物理变化信息，并以一定的策略更新到硬盘上。等到事务提交的时候强制redo log把数据页信息的更改更新到磁盘上。也就是说，当事务提交成功后相应的数据页可能还没更新到磁盘上，但是redo log一定更新到了磁盘上。如果此时宕机，那么innodb在重启时可以根据redo log来更新数据。

如果缓冲池中的脏页都已经刷新到磁盘了，那么redo log就相当于完成了他的使命。

redo log每次更新到磁盘的数据大小都是512kb，正好是一个扇区的大小，因此这个过程是原子性的。

redo log刷新回磁盘的策略

1. 主线程每秒刷新一次
2. 每次事务提交时
3. 当redo log的可用空间少于一半的时候

因此，redo log写入磁盘不是在事务提交的那一刹那完成的，是在提交之前就已经开始“运动”了。所以即使一个大的事务提交，redo log的写入磁盘都会很快，不会成为事务提交慢的原因。

### 2.2 undo log
存在于共享表空间里，5.7可以存放在单独的表空间里。是存放在一个叫做undo log segment里面。undo log不像redo log那样有缓冲池，是顺序的往物理页上面写的。

undo log记录的是一个事务中数据的逻辑改变。如果你删除一条数据那么他就增加一条数据，如果你更改一条数据那么他就反向更改一条数据。

undo log的作用主要有两点

1. 用于事务的回滚
2. 用于MVCC

每当一个事务开启的时候，当要更新一条数据的时候会先写undo log，只有undo log成功的写入了，整个事务才会有“后路”，才能回滚。之后才是写数据写redo log。特别需要注意的是，undo log 也是 redo log的保护对象，每一条undo log的写入都会有redo log的写入。

当一条数据被事务T1修改中，此时T2事务中想要读取此数据，那么T2读取的永远是T1还未修改的那个版本，这就能保证在T2中不会出现幻读的情况。

当一条数据更新时先写undo log，再写数据，再写redo log。

undo log是数据库并发(MVCC)的关关键。innodb为每一行的数据都分配了两个隐藏的列：**事务id和回滚指针**。事务id能代表当前哪个事务正在写这行数据。回滚指针指向undo log中的这条数据的“上一个版本”，一单回滚就使用undo log中的日志恢复数据。undo log与read view一起工作实现了innodb的MVCC。
