# mysql运维之二进制日志。（east_sun参考文档centos 7）
## 1.二进制日志开启
　　服务器的二进制日志（binary log简称binlog）是备份的最重要因素之一，它们对于基于时间点的恢复操作是必要的，并且通常比数据要小，所以更容易进行频繁的备份。MySQL 二进制日志是非常重要的，所以DBA们应该尽可能将二进制日志和数据库文件分开存储。

　　二进制日志主要作用有三个：1.基于备份恢复数据 2.数据库主从复制3.挖掘分析SQL语句。

　　首先我们需要知道如何开启二进制日志。在centos 7 系统下面找全局配置文件my.cnf(而非直接在MySQl里面直接通过set global配置)，首先每个系统my.cnf文件存储的存放位置不同，在我的centos 7， 文件存储在 /etc下面。
　　
　　mycnf文件配置前MySQL返回查询结果：

```
mysql> SHOW GLOBAL VARIABLES LIKE '%log_bin%';#查询二进制日志
+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| log_bin                         | OFF   |
| log_bin_basename                |       |
| log_bin_index                   |       |
| log_bin_trust_function_creators | OFF   |
| log_bin_use_v1_row_events       | OFF   |
+---------------------------------+-------+
5 rows in set (0.00 sec)

mysql> SET GLOBAL log_bin =1  ;
ERROR 1238 (HY000): Variable 'log_bin' is a read only variable
```
　　主要二进制配置参数设置后显示结果：
  
```
log-bin=mysql-bin                ＃开起命令
log-bin = /var/lib/mysql/mysql-bin　＃二进制日志存放地点
expire_logs_days        = 7         ＃二进制日志更新日期
binlog_cache_size       = 4m        
max_binlog_cache_size   = 512m      
max_binlog_size         = 100m  
binlog_format                  = statement 
```


```
mysql> SHOW GLOBAL VARIABLES LIKE '%log_bin%';#再一次查看二进制日志是否开启
+---------------------------------+--------------------------------+
| Variable_name                   | Value                          |
+---------------------------------+--------------------------------+
| log_bin                         | ON                             |
| log_bin_basename                | /var/lib/mysql/mysql-bin       |
| log_bin_index                   | /var/lib/mysql/mysql-bin.index |
| log_bin_trust_function_creators | OFF                            |
| log_bin_use_v1_row_events       | OFF                            |
| sql_log_bin                     | ON                             |
+---------------------------------+--------------------------------+
6 rows in set (0.00 sec)

```
  需要决定日志的过期车略以防止被二进制日志写满。日志增长取决于负载和日志格式。个人建议DBA如果在允许的条件下，尽量不要清楚二进制日志。
  最常见的设置是使用expire_log_days变量来告诉MySQL定期清除日志。
  
## 2.二进制日志运用

  二进制的实例运用主要通过以下常见的操作来实现：
  
```
show binary logs;（查看当前所有的二进制日志）
flush logs;（生成一个新的日志）
show binlog events in 'XXX';（查看具体一个二进制文档）
show binlog events in 'XXX';
show binlog events in 'XXX' from XXX limit 2;
```

flush log 具体运用：



```
mysql> SHOW binary logs ;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       177 |
| mysql-bin.000002 |       177 |
| mysql-bin.000003 |     38729 |
| mysql-bin.000004 |       177 |
| mysql-bin.000005 | 123805025 |
| mysql-bin.000006 |       177 |
| mysql-bin.000007 |       154 |
+------------------+-----------+
7 rows in set (0.00 sec)

mysql> FLUSH logs ;#(生成一个新的二进制日志);
Query OK, 0 rows affected (0.07 sec)

mysql> SHOW binary logs ;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       177 |
| mysql-bin.000002 |       177 |
| mysql-bin.000003 |     38729 |
| mysql-bin.000004 |       177 |
| mysql-bin.000005 | 123805025 |
| mysql-bin.000006 |       177 |
| mysql-bin.000007 |       201 |
| mysql-bin.000008 |       154 |
+------------------+-----------+
8 rows in set (0.00 sec)
```
SHOW binlog EVENTS IN 'XXX';（查看具体一个二进制文档）


```
mysql> SHOW binlog EVENTS IN 'mysql-bin.000001';
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                                  |
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
| mysql-bin.000001 |   4 | Format_desc    |         1 |         123 | Server ver: 5.7.17-log, Binlog ver: 4 |
| mysql-bin.000001 | 123 | Previous_gtids |         1 |         154 |                                       |
| mysql-bin.000001 | 154 | Stop           |         1 |         177 |                                       |
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
3 rows in set (0.01 sec)

mysql> SHOW binlog EVENTS IN 'mysql-bin.000001' FROM 4 LIMIT 2;  注意：4是position
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                                  |
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
| mysql-bin.000001 |   4 | Format_desc    |         1 |         123 | Server ver: 5.7.17-log, Binlog ver: 4 |
| mysql-bin.000001 | 123 | Previous_gtids |         1 |         154 |                                       |
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
2 rows in set (0.00 sec)

```

##mysqlbinlog工具运用

mysqlbinlog用于处理二进制日志文件的自带实用工具详解mysqlbinlog从二进制日志读取语句的工具。

```
＃＃＃mysqlbinlog /usr/local/mysql/data/mysql-bin.000001
mysqlbinlog: [ERROR] unknown variable 'default-character-set=utf8'
小插曲，刚开始用binlog遇到了这个问题，网上找到两种解决办法：

　　原因是mysqlbinlog这个工具无法识别binlog中的配置中的default-character-set=utf8这个指令。

      两个方法可以解决这个问题

一是在MySQL的配置/etc/my.cnf中将default-character-set=utf8 修改为 
character-set-server = utf8，但是这需要重启MySQL服务，如果你的MySQL
服务正在忙，那这样的代价会比较大。

二是用mysqlbinlog --no-defaults mysql-bin.000001 命令打开


$ sudo mysqlbinlog --no-defaults  /usr/local/mysql/data/mysql-bin.000001
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#170304 19:41:14 server id 1  end_log_pos 123 CRC32 0x6248ba69 	Start: binlog v 4, server v 5.7.17-log created 170304 19:41:14 at startup
ROLLBACK/*!*/;
BINLOG '
2qe6WA8BAAAAdwAAAHsAAAAAAAQANS43LjE3LWxvZwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAADap7pYEzgNAAgAEgAEBAQEEgAAXwAEGggAAAAICAgCAAAACgoKKioAEjQA
AWm6SGI=
'/*!*/;
# at 123
#170304 19:41:14 server id 1  end_log_pos 154 CRC32 0x7a111cdc 	Previous-GTIDs
# [empty]
# at 154
#170305  0:05:55 server id 1  end_log_pos 177 CRC32 0xb7177bcf 	Stop
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
louhaomiaodeMacBook-Pro:etc louhaomiao$ 
```



