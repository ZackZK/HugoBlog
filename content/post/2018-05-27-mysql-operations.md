---
date: 2018-05-27T00:00:00Z
description: mysql数据库操作
modified: 2018-05-27
tags:
- mysql
title: mysql数据库笔记
---

# １. 数据库备份和恢复

## 数据备份

1. 备份整个数据库  

   ``` bash
     $ mysqldump -u [uname] -p[pass] db_name > db_backup.sql
   ```

2. 备份指定表 

   ```bash
    $ mysqldump -u [uname] -p[pass] db_name table1 table2 > table_backup.sql
   ```

3. 备份压缩

   ```bash
   $ mysqldump -u [uname] -p[pass] db_name | gzip > db_backup.sql.gz
   ```

## 数据恢复

1. 恢复数据库

   ```bash
   $ mysql -p -u[user] [database] < db_backup.dump
   ```

2. 从恢复表

   ```bash
   $ mysql -uroot -p DatabaseName < path\TableName.sql
   ```

3. 从整个数据库备份中恢复某个表

   ```bash
   $ grep -n "Table structure" mydump.sql
   # identify the first and last line numbers (n1 and n2) of desired table
   $ sed -n n1,n2p mydump.sql > mytable.sql # (e.g. sed -n 48,112p)
   $ mysql -uroot -p DatabaseName <mytable.sql
   ```
## dump数据库表创建sql
```bash
mysqldump -d --compact --compatible=mysql323 ${dbname}|egrep -v "(^SET|^/\*\!)"
```



[参考１]: https://dba.stackexchange.com/questions/14716/can-mysql-restore-a-single-table-from-a-large-mysqldump
[参考２]: https://stackoverflow.com/questions/9696249/restoring-a-mysql-table-back-to-the-database
[参考３]: https://stackoverflow.com/questions/13484667/downloading-mysql-dump-from-command-line
[参考4]: https://stackoverflow.com/questions/1842076/how-do-i-use-mysqldump-to-export-only-the-create-table-commands

