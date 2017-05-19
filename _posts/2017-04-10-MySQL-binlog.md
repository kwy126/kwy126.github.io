## 概述

* MySQL的binlog以event的形式，记录了MySQL中所有的变更情况，必用binlog我们就能重现所有记录的所有操作。

* MySQL引入binlog主要有两个用途/目的：一是为了主从复制；二是用于备份恢复后需要重新应用部分binlog，从而达到全备+增备的效果。



#### binlog的三种可选格式（binlog_format）

###### statement

* binlog_format=statement
* 基于SQL语句的模式，该模式不对数据进行加密，所有体积较小。但是由于对某些不确定的SQL语句，比如uuid(),user(),sysdate(),在重放的过程导致数据不一致情况。因此不建议使用。
```
SET TIMESTAMP=1472352439/*!*/;

insert into t1 (frist_name) values (user())

/*!*/;

SET TIMESTAMP=1472352456/*!*/;

insert into t1 (frist_name) values (sysdate())

/*!*/;

```

######  row

* binlog_format=row
* 基于数据行的模式，记录的是数据行的完整变化。
* 对于event数据是加密的，所有更安全，但是文件体积会更大。
* 推介使用
```
### INSERT INTO `circle_pay`.`t1`
### SET
###   @1=5 /* INT meta=0 nullable=0 is_null=0 */
###   @2='bbbb' /* STRING(12) meta=65036 nullable=0 is_null=0 */
# at 297
#161129 13:32:12 server id 71493308  end_log_pos 328 CRC32 0xaa145a6a   Xid = 12912
COMMIT/*!*/;
```
我们可以发现，binlog中并没有记录列名，只记录了顺序，这样会有什么风险呢？
单机的时候问题不大，但是在双Master的结构下，这问题就大了，假设Master A<- -> Master B 正在相互复制，我在Master做一个DDL改变了字段顺序，那么从B DDL完成开始，到A接受执行完成DDL为止，之间A产生的这张表的数据复制到B都是错误的。因为MySQL只按字段顺序应用binlog
###### mixed

* binlog_format=mixed
Mixed格式会混合使用Statement和row，其中DDL会使用Statement格式，而表操作则使用row格式。可是如果使用Innodb数据库，且事务级别使用Read Commited或Read Uncommitted。则会被强制使用Row格式
当DML语句更新一个NDB表时
当函数包含uuid()时
2个及以上包含auto_increment字段的表被更新时
任何insert delayed语句时
视图中必须要求使用RBR时，例如创建视图使用uuid（）函数时

## mysqlbinlog

mysqldump进行逻辑备份时，如果使用--single-transaction，则隔离级别会改为repeatable read，获得快照



## mysqlbinlog（通过这个命令可以查看二进制日志）

可以通过man mysqlbinlog查看详细的参数，下面罗列几个比较重要的参数

|参数|说明|

|-------|-------|

|--base64-output|基于innodb数据不锁|

|--verbose/-v|详细格式显示|

|--start-position|指定日志的起始位置|

|--start-datetime|指定日志的起始读取时间|

|--stop-position|指定日志停止的位置|

|--stop-datatime|指定日志的结束读取时间|

|--read-from-remote-server/-R|从远程active server上读取日志|

|--stop-never|结合着-R使用|

|--database|指定数据库|







```
mysqlbinlog --base64-output=decode-rows -v 3306-mysql-bin.000002 > /tmp/2.txt
mysqlbinlog --base64-output=decode-rows -v  --start-position=222 --database=student_db 3306-mysql-bin.000002 > /tmp/2.txt

```

###### 实验

当我们一不小心删除了某个库时，一定不要紧张，及时你没有备份也没有关系。只要做到以下几步就可以还原数据库。

背景 
有一次在我安装sonarqube时，不小心删除了sonarqube数据库，更糟糕的是当时还没来得及备份，幸好binlog是没损坏的，于是乎我进行如下操作： 
1、flush logs（保存现场） 
2、做好binlog文件的备份工作 
3、通过mysqlbinlog工具，找到drop database的position 
4、删除原先的binlog文件，将drop database之前的binlog重新执行一边

## 查看binlog日志
```
mysql> show binary logs;
ERROR 2006 (HY000): MySQL server has gone away
No connection. Trying to reconnect...
Connection id:    9
Current database: *** NONE ***
+-----------------------+-----------+
| Log_name              | File_size |
+-----------------------+-----------+
| 3306-mysql-bin.000001 |       174 |
| 3306-mysql-bin.000002 |       174 |
| 3306-mysql-bin.000003 |       174 |
| 3306-mysql-bin.000004 |       174 |
| 3306-mysql-bin.000005 |       174 |
| 3306-mysql-bin.000006 |     63834 |
| 3306-mysql-bin.000007 |   1180803 |
| 3306-mysql-bin.000008 |       203 |
| 3306-mysql-bin.000009 |      7852 |
| 3306-mysql-bin.000010 |       243 |
| 3306-mysql-bin.000011 |       191 |
+-----------------------+-----------+
11 rows in set (0.00 sec)
```

```
mysql> show master status\G;
*************************** 1. row ***************************
             File: 3306-mysql-bin.000009
         Position: 4083
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 187588dc-4fb3-11e6-9a91-00163e0131f0:1-26
1 row in set (0.00 sec)
ERROR: 
No query specified
```
接着查看binlog的格式，此处binlog_format的格式是row 
```
mysql>  show global variables like '%format%';
+--------------------------+-------------------+
| Variable_name            | Value             |
+--------------------------+-------------------+
| binlog_format            | ROW               |
| date_format              | %Y-%m-%d          |
| datetime_format          | %Y-%m-%d %H:%i:%s |
| default_week_format      | 0                 |
| innodb_file_format       | Barracuda         |
| innodb_file_format_check | ON                |
| innodb_file_format_max   | Antelope          |
| time_format              | %H:%i:%s          |
+--------------------------+-------------------+
8 rows in set (0.00 sec)
```


#### 复制binlog方案一：此方法复制的DML语句都是加密过的
```
[root@nbview db3306]# mysqlbinlog -v 3306-mysql-bin.000009 > /tmp/mysql09.txt
```
**结果**
```
[root@nbview db3306]# vim /tmp/mysql09.txt 
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!40019 SET @@session.max_insert_delayed_threads=0*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#700101  8:00:00 server id 71493306  end_log_pos 120 CRC32 0x81e22bd2   Start: binlog v 4, server v 5.6.30-log created 700101  8:00:00
# Warning: this binlog is either in use or was not closed properly.
BINLOG '
AAAAAA+65kIEdAAAAHgAAAABAAQANS42LjMwLWxvZwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAEzgNAAgAEgAEBAQEEgAAXAAEGggAAAAICAgCAAAACgoKGRkAAdIr
4oE=
'/*!*/;
# at 120
#700101  8:00:00 server id 71493306  end_log_pos 151 CRC32 0x7ec5bbd1   Previous-GTIDs
# [empty]
```
#### 复制binlog方案二
```
[root@nbview db3306]# mysqlbinlog -v 3306-mysql-bin.000009 --base64-output=decode-rows > /tmp/mysql20.txt
```
**结果**
```
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!40019 SET @@session.max_insert_delayed_threads=0*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#700101  8:00:00 server id 71493306  end_log_pos 120 CRC32 0x81e22bd2   Start: binlog v 4, server v 5.6.30-log created 700101  8:00:00
# Warning: this binlog is either in use or was not closed properly.
# at 120
#700101  8:00:00 server id 71493306  end_log_pos 151 CRC32 0x7ec5bbd1   Previous-GTIDs
# [empty]
# at 151
#160725 13:15:52 server id 71493306  end_log_pos 199 CRC32 0x908b8e18   GTID [commit=yes]
SET @@SESSION.GTID_NEXT= '187588dc-4fb3-11e6-9a91-00163e0131f0:1'/*!*/;
# at 199
#160725 13:15:52 server id 71493306  end_log_pos 314 CRC32 0x0016590a   Query   thread_id=31    exec_time=0     error_code=0
SET TIMESTAMP=1469423752/*!*/;
SET @@session.pseudo_thread_id=31/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1073741824/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=33/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
create database student_db1
/*!*/;
# at 314（314表示下一个事务的position）
#160725 13:18:44 server id 71493306  end_log_pos 362 CRC32 0x586d3310   GTID [commit=yes]
SET @@SESSION.GTID_NEXT= '187588dc-4fb3-11e6-9a91-00163e0131f0:2'/*!*/;
# at 362
#160725 13:18:44 server id 71493306  end_log_pos 534 CRC32 0xdf055b3e   Query   thread_id=31    exec_time=0     error_code=0
use `student_db1`/*!*/;
SET TIMESTAMP=1469423924/*!*/;
create table tab1(id int not null auto_increment,t_name varchar(32),primary key(id))
/*!*/;
# at 534
#160725 13:24:30 server id 71493306  end_log_pos 582 CRC32 0xc86faa9a   GTID [commit=yes]
```

