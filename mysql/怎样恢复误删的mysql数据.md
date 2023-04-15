# 怎样恢复误删的mysql数据

> 由于工作失误，把控不严，一时迷糊将正式数据表的某个字段删除了。还好数据库有开启日志功能，并且每天有定时全盘备份。可以先将数据恢复到上一天的，再使用日志恢复当天的数据。

### 利用备份恢复为上一天的数据

全盘备份的数据库`sql`文件有差不多`2G`大，使用`Navicat for MySQL`导入到新建的测试库。时间过了好久，进度还是 2%，完全导入的话，起码得大半天时间，不等我导完就给公司的同事催死了。

由于我只是想导入误删表结构的两个表，所以无需整体导入，最终在网上搜索到方法，可以使用`sed`只切割这两个表的数据内容。

```
sed -n '/^-- Table structure for table `ar`/,/^-- Table structure for table `/p' oa_2023041023.sql > ar_2023041023.sql
sed -n '/^-- Table structure for table `ar_detail`/,/^-- Table structure for table `/p' oa_2023041023.sql > ar_detail_2023041023.sql
```

这里不得不说`sed`真是神器，差不多`2G`大的文件，这么简单就给我切割成只有几十M的小文件了。

再使用这两个小文件导入，那就是十来分钟的事情了。



### 利用日志恢复当天数据

接下来，利用日志恢复当天的数据。首先我们导出当天的所有数据，目的是为了准确定位误删字段时的时间点或者偏移量。

```shell
1. 导出当天的所有数据
mysqlbinlog --base64-output=decode-rows --verbose --start-datetime='2023-04-11 00:00:00' --stop-datetime='2023-04-12 23:59:59' -d oa /var/lib/mysql/mysql-bin.000030 > /var/lib/mysql/oa.20230411.sql
# --base64-output=decode-rows --verbose 如果不加这个参数，导出的数据为加密的数据

2. 查询时间点和结束偏移量
grep -i 'drop' -C 8 /var/lib/mysql/oa.20230411.sql
###   @35=0
# at 300581227
#230411 10:14:52 server id 107  end_log_pos 300581254   Xid = 1419710277
COMMIT/*!*/;
# at 300581254
#230411 10:15:29 server id 107  end_log_pos 300581460   Query   thread_id=88187924      exec_time=6     error_code=0
SET TIMESTAMP=1681179329/*!*/;
ALTER TABLE `ar`
DROP COLUMN `sum`
/*!*/;
# at 300581460
#230411 10:15:50 server id 107  end_log_pos 300581534   Query   thread_id=88200561      exec_time=0     error_code=0
SET TIMESTAMP=1681179350/*!*/;

3. 查询起始偏移量
grep '230411 ' -C 8 /var/lib/mysql/oa.20230411.sql | less
```

误删字段时的时间点是`“230411 10:14:52”`，偏移量是`300581227`。找到这两个关键参数就可以利用它来将时间恢复到未删除之前。

```shell
mysqlbinlog --start-datetime='2023-04-11 00:00:00' --stop-datetime='2023-04-11 10:14:52' -d oa /var/lib/mysql/mysql-bin.000030 > /var/lib/mysql/oa.20230411.sql

mysqlbinlog --start-position=299785648 --stop-position=300581227 -d oa /var/lib/mysql/mysql-bin.000030 > /var/lib/mysql/oa.20230411.sql
```



最后导入这两份数据即可。



