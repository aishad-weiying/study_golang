# MariaDB数据库的备份与恢复  
### 前提条件：  
> 开启二进制日志：二进制日志可用于数据库恢复时使用，建议二进制日志与数据库数据分开存放。  
> > 开启二进制日志的方法：
> > ```bash  
> > [ root@Centos~]# vim /etc/my.cnf
> > [mysqld]
> > log_bin=/data/binary-log/mariadb-bin
> > ```
```
> 注：日志会存放在/data/binary-log目录下，文件名为mariadb-bin.000001下  

### mysqldump
> 逻辑备份工具，适用所有存储引擎，温备；支持完全或部分备份；对InnoDB存储引擎支持热备，结合binlog的增量备份     

####  mysqldump工具：客户端命令，通过mysql协议连接至mysql服务器进行备份
     
- 用法1： 
     > mysqldump [OPTIONS] database [tables]:默认只将查询结果打印到屏幕  
    - 备份  
    mysqldump -uroot -ppasswd database > database.sql  
    - 还原  
mysql database [table] < database.sql
        - 注：还原的时候，直接重定向执行，但是前提是database数据库存在，如果不存在择不能还原

- 用法2：   
    > mysqldump [OPTIONS] –B DB1 [DB2 DB3...]  (会备份数据库的定义)     
    - 备份：
            mysqldump -B database > database.sql  
    - 还原：  
                mysql [table] < database.sql  
- 用法3：  
    > mysqldump [OPTIONS] –A [OPTIONS]（备份所有数据库不包括performance_schema和information_schema库）  
    - 备份  
    mysqldump -A > alltabase.sql  
    - 还原：  
                mysql> set sql_log_bin=OFF; 临时关闭二进制日志  
                mysql> reset master; 清除所有二进制日志  
                mysql> source ~/alltabase.sql  
                mysql> set sql_log_bin=ON; 开启二进制日志  
>注意：在还原数据库时，都应该临时关闭二进制日志，因为还原数据库的操作，没有必要记录到二进制日志

## mysqldump常见选项：
- -A， --all-databases 备份所有数据库，含create database  
- -B , --databases db_name… 指定备份的数据库，包括create database语句  
- -E, --events：备份相关的所有event scheduler 计划任务  
- -R, --routines：备份所有存储过程和自定义函数  
- --triggers：备份表相关触发器，默认启用,用--skip-triggers，不备份触发器
- --default-character-set=utf8 指定字符集
- --master-data[=#]： 此选项须启用二进制日志  
        1：所备份的数据之前加一条记录为CHANGE MASTER TO语句，非注释，不指定#，默认为1  
        2：记录为注释的CHANGE MASTER TO语句此选项会自动关闭--lock-tables功能，自动打开-x | --lock-all-tables功能（除非开启--single-transaction）  
- -F, --flush-logs ：备份前滚动日志，锁定表完成后，执行flush logs命令,生成新的二进制日志文件，配合-A 或 -B 选项时，会导致刷新多次数据库。建议在同一时刻执行转储和日志刷新，可通过和--single-transaction或-x，--master-data 一起使用实现，此时只刷新一次日志
- --compact 去掉注释，适合调试，生产不使用
- -d, --no-data 只备份表结构
- -t, --no-create-info 只备份数据,不备份create table
- -n,--no-create-db 不备份create database，可被-A或-B覆盖
- --flush-privileges 刷新权限，备份mysql或相关时需要使用
- -f, --force 忽略SQL错误，继续执行
- --hex-blob 使用十六进制符号转储二进制列，当有包括BINARY，VARBINARY，BLOB，BIT的数据类型的列时使用，避免乱码  
- -q, --quick 不缓存查询，直接输出，加快备份速度

## 通过备份文件和二进制文件恢复数据库到当前装填  
> 通常数据库的备份都发生在凌晨用户访问量较少的时候进行，例如我们在凌晨3点对数据库进行了完全备份，但是在上午11点的时候，因为管理员的误操作，到时候数据库某张表被删除，如果我们用数据库的完整备份的话，就只能恢复到凌晨3点，那么要恢复3点到11点这段时间的数据，就需要用到二进制日志  

>前提:在进行数据库备份的时候加了--master-data=[1|2]选项，记录了数据库备份时，日志的位置

### 具体步骤
1. 对数据库进行完整备份
​```bash
    [ root@Centos~]# mysqldump -A --master-data=2 > database.sql
    [ root@Centos~]# less database.sql
        -- CHANGE MASTER TO MASTER_LOG_FILE='mariadb-bin.000007', MASTER_LOG_POS=8134;
        这条记录了当前的备份所在的日志位置，为mariadb-bin.000007的8134
```
2. 对数据库的数据进行修改
```bash
    MariaDB [hellodb]> select * from teachers;
        +-----+---------------+-----+--------+
        | TID | Name          | Age | Gender |
        +-----+---------------+-----+--------+
        |   1 | Song Jiang    |  45 | M      |
        |   2 | Zhang Sanfeng |  94 | M      |
        |   3 | Miejue Shitai |  77 | F      |
        |   4 | Lin Chaoying  |  93 | F      |
        |   5 | weiying       |  25 | NULL   |
        +-----+---------------+-----+--------+
    MariaDB [hellodb]> insert teachers(name,age) value("xiao ming",20);
    MariaDB [hellodb]> select * from teachers;
        +-----+---------------+-----+--------+
        | TID | Name          | Age | Gender |
        +-----+---------------+-----+--------+
        |   1 | Song Jiang    |  45 | M      |
        |   2 | Zhang Sanfeng |  94 | M      |
        |   3 | Miejue Shitai |  77 | F      |
        |   4 | Lin Chaoying  |  93 | F      |
        |   5 | weiying       |  25 | NULL   |
        |   6 | xiao ming     |  20 | NULL   |
        +-----+---------------+-----+--------+
```
3. 查看当前二进制日志记录的位置
```bash
    MariaDB [hellodb]> show master logs;
        +--------------------+-----------+
        | Log_name           | File_size |
        +--------------------+-----------+
        | mariadb-bin.000001 |     15763 |
        | mariadb-bin.000002 |     30376 |
        | mariadb-bin.000003 |   1038814 |
        | mariadb-bin.000004 |      8153 |
        | mariadb-bin.000005 |     30376 |
        | mariadb-bin.000006 |   1038814 |
        | mariadb-bin.000007 |      8373 |
        +--------------------+-----------+
    之前数据库备份时二进制的位置为8134
```
4. 删除数据库并恢复  
- 停止服务并删除数据库：
```bash
   [ root@Centos~]# systemctl stop mariadb
   [ root@Centos~]# rm -rf /var/lib/mysql/*  
   [ root@Centos~]# systemctl start mariadb
```
- 恢复：  
    - 先获取备份之后二进制日志文件的内容： 
```bash 
      [ root@Centos~]# mysqlbinlog --start-position=8134 /data/binary-log/mariadb-bin.000007  > binary.sql
```
  	  导入备份和二进制日志恢复： 
```bash 
        MariaDB [(none)] set sql_log_bin=OFF; 临时关闭二进制日志  
        MariaDB [(none)] reset master; 清除所有二进制日志        
        MariaDB [(none)] source database.sql
        MariaDB [(none)] source binary.sql
        MariaDB [(none)] mysql> set sql_log_bin=ON; 开启二进制日志
```
- 查询数据表：数据库恢复到最新状态
    - 注：如果在备份完成之后，是因为删除了某张表导致的数据库崩溃，那么在提取出来的二进制文件中找到该条drop语句，并将其注释。