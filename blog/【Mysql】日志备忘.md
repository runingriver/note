---
title: 【Mysql】日志备忘
date: 2016/11/12 11:12:22
toc: false
list_number: false
categories:
- mysql
tags:
- mysql
---

## 1. 介绍
mysql常用日志有** 错误日志**，**binlog日志(二进制日志)**，**慢查询日志**，查询日志等。
当日常开发中设计mysql问题的时候，我们可以通过日志来定位和解决问题，这里着重介绍binlog日志。

## 2. 相关命令
mysql 配置文件一般是：`/etc/my.cnf`（目录位置可能不同但都是`my.cnf`）
### 系统变量查看和设置

- 显示系统日志开启情况和位置：`show variables like 'log_%';` ，`show global variables like '%log%';`（显示全局所有类型日志情况）
### binlog日志相关命令
1. 查看所有binlog日志列表： `show master logs;`
2. 查看master状态，即最后(最新)一个binlog日志的编号名称，及其最后一个操作事件pos结束点(Position)值`show master status;`
3. 刷新log日志，自此刻开始产生一个新编号的binlog日志文件：` flush logs;`
 注：每当mysqld服务重启时，会自动执行此命令，刷新binlog日志；在mysqldump备份数据时加 -F 选项也会刷新binlog日志；
4. 重置(清空)所有binlog日志：`reset master;`
5. 显示binlog的模式：`show global variables like 'binlog_format';`
6. 将binlog转成能看懂的文件形式： `sudo ./mysqlbinlog /home/q/data/mysql/mysql-bin.000028 > ~/a.sql`

## 3. binlog二进制日志
### binlog的三种模式

1. `row`：日志中记录的是每一行的数据修改，所以日志会非常大，mysql后面版本做了优化，将改表的语句记录成statement形式（eg：alter），但是update，delete仍是拆分成多条记录形式。
eg：`update user set age=22 where sex=‘man’`，这样一条sql，影响的是多条记录，row模式会记录每行的影响，并作为一条记录和事件。
2. `statement`：只记录修改表的sql语句，每条sql语句作为一条事件记录，减少日志大小，减少主从复制的I/O。
3. `mixed`：是前两种模式的结合。在 Mixed 模式下，MySQL 会根据执行的每一条具体的 SQL 语句来区分对待记录的日志形式，也就是在 statement 和 row 之间选择一种。

### 查看binlog的方法
1. `mysqlbinlog` mysql自带工具查看
语法：`mysqlbinlog [OPTION] [-d db_name] binlog_file`，其中OPTION：`--start-datetime`，`--stop-datetime`，`--start-position`，`--stop-position`
eg：`sudo ./mysqlbinlog --start-datetime='2017-06-27 15:20:00' -d test /home/q/data/mysql/mysql-bin.000028`
mysqlbinlog 也可以查看远程mysql binlog，更多用法可以`./mysqlbinlog --help`
2. 在CLI中使用`show binlog`命令查看
语法：show binlog events [in 'log_name'] [from pos] [limit [offset,] row_count]; 
`log_name`: binlog日志文件名，`pos`：事件位置，`limit`和sql中的limit语义相同。
eg： `show binlog events in 'mysql-bin.000028' limit 2,10 \G;` 查询`mysql-bin.000028`日中第2到10条记录;
eg：`show binlog events in 'mysql-bin.000028' from 975 limit 10 \G;` 查询从pos为975开始的10条记录
3. 数据恢复
语法：`mysqlbinlog -d db_name mysql-bin.0000xx | mysql -u用户名 -p密码 数据库名`
其实就是用mysqlbinlog翻译成sql语句，然后管道传给mysql 客户端执行！
所以，可以先` ./mysqlbinlog /home/q/data/mysql/mysql-bin.000028 > ~/a.sql`到出sql，再给mysql执行,注意指定数据库名！。
恢复期间，先要` flush logs;`另启一个日志文件，且需要找到恢复的起始点（找到备份开始点，回滚结束点（记录时间和pos都可以））。

**Tip:上述数据恢复适合离线恢复，对于线上mysql数据还在实时写入的情况，需要闪回，可以使用binlog2sql，但是此时必须是row模式，而且也不能恢复DDL的操作！**

## 4. 错误日志，慢查询日志，查询日志
默认错误日志开启，慢查询日志不开启，查询日志不开启，这几个日志都是以普通文本形式呈现，一般建议不开启，因为有性能丢失，`show global variables like '%log%';`命令可以找到日志位置；
mysql为慢查询提供了一个命令行工具`mysqldumpslow`方便我们分析和查找。
### 其他日志文件说明
`ibdata1` ：InnoDB中保存数据的文件，用于存储InnoDB引擎下的数据库实际数据，索引等，库名的文件夹里面的那些表文件只是结构而已！可以配置将实际数据放到库名文件夹下，从而减小ibdata1文件的大小。
`ib_logfile0，ib_logfile1`：INNODB的REDO、UNDO日志（也称事务日志），事务执行中暂时保留一些事务信息(记录对数据文件数据修改的物理位置或叫做偏移量)，主要是在事务中起一个前滚或后滚的作用。
binlog中的事务日志是已经commit执行成功的事务，只有在事务执行成功才会刷入binlog的事务日志。

推荐博客：http://www.penglixun.com/archives 和 http://hedengcheng.com/?cat=4