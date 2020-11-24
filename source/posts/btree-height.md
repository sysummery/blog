---
title: Innodb查看B+tree的高度
date: 2019-11-21 10:46:47
tags:
    - mysql
    - innodb
photos:
    - ["http://sysummerblog.club/b-plus-tree_cover.png"]
---
在innodb中无论是主键索引还是非主键索引数据结构都是B+tree。而B+tree数的高度直接决定了在不考虑缓存的情况下读取一个数据页硬盘所需的IO次数，因此B+tree的高度相当重要。生产环境下高度一般为2~3.
<!--more-->
## 一颗B+tree能存放多少数据？
先回顾几个名词

* 磁盘的最小存储单元是扇区，大小为512字节
* 文件系统的最小存储单元是4kb
* innodb中数据的最小存储单元是页，大小为16kb（16384字节，不过也可以设置的更小）可以通过以下命令获取
```sql
show variables like 'innodb_page_size'
```
* innodb中聚簇索引（主键）的叶子节点存放的是数据，非叶子节点存放的是关键字和指向下一层的指针，如下图所示，不过实际上叶子结点之间是一个双向链表而不是单向链表
![](http://sysummerblog.club/%E6%88%AA%E5%B1%8F2019-11-21%E4%B8%8B%E5%8D%882.41.29.jpg)

我们假设主键是8字节的bigint型的，B+tree中的指针是6字节，那么一对数据+指针就是14字节，一个数据页的大小时16kb,但是一个数据页中不可能全部存放数据，还会有一些元信息，关于页结构可以参考这篇[文章](https://www.cnblogs.com/bdsir/p/8745553.html)。我们假设一个数据页存放数据部分最大16282字节。那么一个数据页可以存放`16282/14=1163`对“键值与指针”。

如果一个数据表的主键索引的高度是2，说明他的第一层是键值和指针，第二层就是数据页了。第一层有1163个指针，也就说明第二层有1163个数据页，总共能存放`1163*16282=18935966`比特的数据。如果一行数据的平均大小为1kb,那么高度为2的B+tree可以存放`18935966/1024=18492`条数据。下面的这个是sql是查看一张表的行数据的平均大小
```sql
SELECT
	* 
FROM
	information_schema.TABLES 
WHERE
	information_schema.TABLES.TABLE_SCHEMA = 'test' 
	AND information_schema.TABLES.TABLE_NAME = 'insert_asc'
```
其中test是库名insert_asc是表名。返回结果中的'AVG_ROW_LENGTH'就是每行数据的平均长度。

如果高度是3的B+tree，第一层有1163个关键字于指针对，第二层就有`1163*1163`个关键字与指针对，叶子层就可以存放`1163*1163*16282`比特的数据。还是假如平均一行的数据大小是1kb,那么高度是3的B+tree可以存放`1163*1163*16282/1024=21506375`条数据。

以上的推论过程有如下的局限性

1. 一个数据页可以用来存放数据的空间是多少，我们假设是16282比特，但具体是多少我查了很多资料都没找到。
2. 当一个数据页中的数据行的某一字段太大的时候，会把一部分数据放到额外的页中，为的是保证一个数据页至少有两条数据。所以啊，实际存储数据的数据页可能大于主键B+tree的叶子节点的数目。

尽管上面的论证有一些局限性，但是我们还是能够得到如下的结论

1. 如果单行数据不是太大的情况下，比如1kb,那么一课高度为3的主键索引可以存储2000多万行的数据，如果单行数据为0.5kb那么可以存储4000多万行的数据。
2. 如果数据行的字段很多，会导致行数据的平均大小变大，可能会使B+tree的高度变大
3. 一般来说B+tree的高度为2~3的时候性能是不错的，换言之如果单行记录不大的情况下，innodb的一张表能轻松地hold住2000万级别的数据

## 直接查看B+tree的高度
在innodb的表空间文件中,约定在根页偏移量64的地方存放了B+tree的高度。高度是从0开始算的。使用sql
```sql
SELECT
	b.NAME,
	a.NAME,
	index_id,
	type,
	a.space,
	a.PAGE_NO 
FROM
	information_schema.INNODB_SYS_INDEXES a,
	information_schema.INNODB_SYS_TABLES b 
WHERE
	a.table_id = b.table_id 
	AND a.space <> 0 
	AND b.NAME = 'test/insert_desc';
```
可看到test数据库insert_desc表的所有索引信息
![](http://sysummerblog.club/%E6%88%AA%E5%B1%8F2019-11-21%E4%B8%8B%E5%8D%884.44.31.jpg)

可以看到第一行是主键索引，他的根页在表空间的第三页。resblock_id是一个普通的索引，他的根页在表空间的第四页。使用
```sh
hexdump -s 49216 -n 10 insert_asc.ibd
```
可以看到表insert_asc.ibd主键索引的高度。
![](http://sysummerblog.club/%E6%88%AA%E5%B1%8F2019-11-21%E4%B8%8B%E5%8D%885.00.59.jpg)
可以看到前两个字节是2，因此高度是3因为高度是从0开始算的。

`-s 49216`表示从表空间文件的49216字节的偏移量开始读取，为什么是49216？因为主键索引的是在第三页,`49216=16kb*3+64`,其中16kb是数据页的大小。同理resblock_id的索引高度读取方法是
```
hexdump -s 65600 -n 10 insert_asc.ibd
```
得到的结果是
![](http://sysummerblog.club/%E6%88%AA%E5%B1%8F2019-11-21%E4%B8%8B%E5%8D%885.04.23.jpg)

也就是改索引的高度是1.
