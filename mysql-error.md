# Linux - kernel: blk_update_request: I/O error, dev fd0, sector 0



**错误日志：**

centos 7 系统启动时，界面会显示如下错误：

```
blk_update_request: I/O error, dev fd0, sector 0
```

**问题原因：**

```sh
This is caused by AppAssure / Rapid Recovery, polling for supported devices to protect in the background and incidentally polls the floppy disk drive. The floppy drive is automatically added in many cases, to virtual machines weather you specify to add it or not.

Since the floppy device is not supported for protection via AppAssure / Rapid Recovery, the device does not properly handle the poll request and throws an error.

Also, it should be noted that this will not affect the integrity of the AppAssure or Rapid Recovery recovery points.

这是由 AppAssure / Rapid Recovery 引起的，它轮询支持的设备以在后台进行保护，并偶然轮询软盘驱动器。在许多情况下，软盘驱动器会自动添加到虚拟机中，无论您是否指定添加它。
由于不支持通过 AppAssure/Rapid Recovery 保护软盘设备，因此设备无法正确处理轮询请求并引发错误。
此外，还应注意，这不会影响 AppAssure 或 Rapid Recovery 恢复点的完整性。
```

**解决方法：**

CentOS/RHEL (Red Hat Enterprise Linux)

```
rmmod floppy
echo "blacklist floppy" > /etc/modprobe.d/blacklist-floppy.conf
cp /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.backup
dracut -f /boot/initramfs-$(uname -r).img
Reboot
```
Debian/Ubuntu
```
rmmod floppy
echo "blacklist floppy" > /etc/modprobe.d/blacklist-floppy.conf
cp /boot/initrd.img-$(uname -r) /boot/initrd.img-$(uname -r).backup
update-initramfs -u
Reboot (optional)
```
SUSE (SuSE Linux Enterprise Server)
```
rmmod floppy
echo "blacklist floppy" > /etc/modprobe.d/blacklist-floppy.conf
cp /boot/initrd-$(uname -r) /boot/initrd-$(uname -r).backup
mkinitrd
Reboot (optional)
```





MariaDB数据库管理系统是MySQL的一个分支，主要由开源社区在维护，采用GPL授权许可。开发这个分支的原因之一是：甲骨文公司收购了MySQL后，有将MySQL闭源的潜在风险，因此社区采用分支的方式来避开这个风险。

# signal 6 mysql_MySQL : mysqld got signal 6 ，数据库无法启动



**问题日志**

```sh
2017-08-31 14:18:05 4122 [Note] InnoDB: Database was not shutdown normally!
2017-08-31 14:18:05 4122 [Note] InnoDB: Starting crash recovery.
2017-08-31 14:18:05 4122 [Note] InnoDB: Reading tablespace information from the .ibd files...
2017-08-31 14:18:05 4122 [ERROR] InnoDB: Attempted to open a previously opened tablespace. Previous tablespace dev/tb_test uses spac
e ID: 1 at filepath: ./dev/tb_test.ibd. Cannot open tablespace mysql/innodb_table_stats which uses space ID: 1 at filepath: ./mysql/
innodb_table_stats.ibd
2017-08-31 14:18:05 2ad861898590  InnoDB: Operating system error number 2 in a file operation.
InnoDB: The error means the system cannot find the path specified.
InnoDB: If you are installing InnoDB, remember that you must create
InnoDB: directories yourself, InnoDB does not create them.
InnoDB: Error: could not open single-table tablespace file ./mysql/innodb_table_stats.ibd
InnoDB: We do not continue the crash recovery, because the table may becomeInnoDB: corrupt if we cannot apply the log records in the InnoDB log to it.
InnoDB: To fix the problem and start mysqld:
InnoDB: 1) If there is a permission problem in the file and mysqld cannot
InnoDB: open the file, you should modify the permissions.
InnoDB: 2) If the table is not needed, or you can restore it from a backup,
InnoDB: then you can remove the .ibd file, and InnoDB will do a normal
InnoDB: crash recovery and ignore that table.
InnoDB: 3) If the file system or the disk is broken, and you cannot remove
InnoDB: the .ibd file, you can set innodb_force_recovery > 0 in my.cnf
InnoDB: and force InnoDB to continue crash recovery here.
150126 14:18:06 mysqld_safe mysqld from pid file /home/mysql/mysql_app/dbdata/liuyazhuang136.pid ended
```

**解决方案**
在/etc/my.cnf中添加如下参数

```conf
[mysqld]
innodb_force_recovery=6
```

**innodb_force_recovery影响整个InnoDB存储引擎的恢复状况，默认值为0，表示当需要恢复时执行所有的恢复操作。当不能进行有效的恢复操作时，mysql有可能无法启动，并记录下错误日志。**

**innodb_force_recovery可以设置为1-6,大的数字包含前面所有数字的影响。当设置参数值大于0后，可以对表进行select,create,drop操作,但insert,update或者delete这类操作是不允许的。**

**innodb_force_recovery参数解释：**

* 1(SRV_FORCE_IGNORE_CORRUPT):忽略检查到的corrupt页
* 2(SRV_FORCE_NO_BACKGROUND):阻止主线程的运行，如主线程需要执行full purge操作，会导致crash
* 3(SRV_FORCE_NO_TRX_UNDO):不执行事务回滚操作。
* 4(SRV_FORCE_NO_IBUF_MERGE):不执行插入缓冲的合并操作。
* 5(SRV_FORCE_NO_UNDO_LOG_SCAN):不查看重做日志，InnoDB存储引擎会将未提交的事务视为已提交。
* 6(SRV_FORCE_NO_LOG_REDO):不执行前滚的操作。

备份数据库

```sh
mysqldump -h 192.168.209.136 -uroot -p dev > /home/mysql/dev.sql
```

删除数据库

```sql
mysql -h 192.168.209.136 -uroot -p
mysql> drop database dev;
ERROR 1051 (42S02): Unknown table 'dev.tb_test'
```

物理删除tb_test对应的frm和ibd文件

```sql
mysql> drop database dev;
Query OK, 0 rows affected (0.00 sec)
```

创建数据库

```sql
mysql> create database dev;
Query OK, 1 row affected (0.03 sec)
```

去掉参数innodb_force_recovery将之前设置的参数去掉后，重新启动数据库

```conf
#innodb_force_recovery=6
```

导入数据

```sh
mysql -h 192.168.209.136 -uroot -pmysql dev </home/mysql/dev.sql

```

提示表已经存在，这是因为将innodb_force_recovery参数去掉后，数据库会进行回滚操作，会生成相应的ibd文件，所有需要将该文件删除掉.
删除后重新导入

```sh
mysql -h 192.168.209.136 -uroot -pmysql dev </home/mysql/dev.sql
```







# Mariadb error 6_[ERROR] mysqld got signal 6 错误

启动mariadb的时候，在新的控制台查看报错日志tail -f /var/log/mariadb/mariadb.log 报错如下：

```
[ERROR] mysqld got signal 6 ;
```

**问题原因：和上面是一样，但是处理方式不太一样，因为mariadb是社区版本。**

**解决办法：**

在 配置文件my.cnf 里边添加配置

```sh
[mysqld]
innodb_force_recovery = 6
```

启动数据库，备份数据库。

```sh
systemctl start mariadb；systemctl status mariadb
mysqldump -uroot -p -A > /home/backup/mysql_all.sql
```

备份完之后，关闭数据库

强烈建议是重装数据库，亲测，如果不重装，导入数据库的时候，依然会报错。

```sh
yum remove mariadb mariadb-server -y
rm -rf /var/lib/mysql/*
rm -rf /etc/my.cnf
yum install mariadb mariadb-server -y
```

启动mariadb

```sh
systemctl start mariadb
```

把备份的数据进行备份恢复

```sql
mysql -uroot
source /home/backup/mysql_all.sql
```

要恢复之前的数据库用户和密码

```sql
mysqladmin -uroot password "密码"
grant all privileges on *.* to 'root'@'localhost' identified by '密码'；
flush privileges;
grant all privileges on *.* to 'root'@'%' identified by ''密码;
flush privileges;
```

这样问题就解决了.