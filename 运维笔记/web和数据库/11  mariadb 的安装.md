# MariaDB 数据库的安装
MariaDB 的安装方式

1. 源码安装:编译安装

2. 二进制格式的程序包安装:展开至特定的路径，并经过简单的配置后即可使用

3. 使用程序包管理器安装,根据一些程序包的镜像仓库进行安装
项目官方：https://downloads.mariadb.org/mariadb/repositories/
国内镜像：https://mirrors.tuna.tsinghua.edu.cn/mariadb/yum/ 或者 https://mirrors.tuna.tsinghua.edu.cn/mysql/yum/

## RPM 包安装
服务器包:mariadb-server
客户端工具包:mariadb

## 二进制格式的程序包安装

1. 下载官方提供的二进制程序包
www.mariadb.org

2. 创建 mysql 用户和组,使用该用户启动服务
```bash
[root@localhost ~]# groupadd -r -g 306 mysql
[root@localhost ~]# useradd -r -g 306 -u 306 mysql
```

3. 展开二进制程序包到指定的位置
```bash
[root@localhost ~]# tar -xvf mariadb-10.2.23-linux-x86_64.tar.gz -C /usr/local/
```

4. 创建软连接并更改目录的属主和属组
```bash
# 创建软连接
[root@localhost ~]# ln -sn /usr/local/mariadb-10.2.23-linux-x86_64 /usr/local/mysql

[root@localhost ~]# chown -R mysql:mysql /usr/local/mysql
[root@localhost ~]# chown -R mysql:mysql /usr/local/mariadb-10.2.23-linux-x86_64
```

5. 初始化数据库:(最好是放在独立的分区上)
```bash
[root@localhost ~]# /usr/local/mysql/scripts/mysql_install_db --user=mysql --datadir=/usr/local/mysql/data
Installing MariaDB/MySQL system tables in '/usr/local/mysql/data' ...
OK

[root@localhost ~]# ls /usr/local/mysql/data/
aria_log.00000001  ib_buffer_pool  ib_logfile0  mysql               test
aria_log_control   ibdata1         ib_logfile1  performance_schema
```

6. 复制数据库自带的启动脚本放到指定的位置
```bash
[root@localhost ~]# cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
# 添加开机自启动
[root@localhost ~]# chkconfig --level 2345 --add mysqld
```

7. 准备配置文件
```bash
[root@localhost ~]# cp /usr/local/mysql/support-files/my-huge.cnf /etc/my.cnf
# mysqld 字段添加下面配置
[root@localhost ~]# vim /etc/my.cnf
[mysqld]
datadir         = /usr/local/mysql/data   // 指定数据存储位置
innodb_file_per_table = on					// 每个表单独保存,而不是都村粗在 innodb 的表空间总
skip_name_resolve = on						// 跳过地址反解
thread_concurrency = 8 						// CPU 核心数的两倍
```

8. 启动服务
```bash
[root@localhost ~]# /etc/init.d/mysqld start
Starting mysqld (via systemctl):                           [  确定  ]
# 验证服务已经监听指定的端口
[root@localhost ~]# ss -tnl | grep 3306
LISTEN     0      80        [::]:3306                  [::]:*  
```

9. 安全初始化
```bash
[root@localhost ~]# /usr/local/mysql/bin/mysql_secure_installation 

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none): 
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] Y
New password: 
Re-enter new password: 
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] Y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] n
 ... skipping.

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] Y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] Y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

10. 访问测试
```bash
[root@localhost ~]# /usr/local/mysql/bin/mysql -uroot -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 18
Server version: 10.2.23-MariaDB-log MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
3 rows in set (0.00 sec)
```

11. 将 mysql 命令行工具添加到环境变量
```bash
[root@localhost ~]# vim /etc/profile
# 最下面添加
export PATH=$PATH:/usr/local/mysql/bin

#立即生效
[root@localhost ~]# source /etc/profile
```

## 编译安装 MariaDB

1. 安装依赖包
```bash
[root@localhost ~]# yum -y install bison bison-devel zlib-devel libcurl-devel libarchive-devel boost-devel gcc gcc-c++ cmake ncurses-devel gnutls-devel libxml2-devel openssl-devel libevent-devel libaio-devel
```

2. 准备用户和数据目录
```bash
[root@localhost ~]# useradd -r -s /sbin/nologin -d /data/mysql mysql
[root@localhost ~]# mkdir -p /data/mysql
[root@localhost ~]# chown mysql.mysql /data/mysql
```
3. 解压程序包并编译安装
```bash
#解压
[root@localhost ~]# tar -xvf mariadb-10.2.23.tar.gz
[root@localhost ~]# cd mariadb-10.2.23/
# 检查依赖环境
[root@localhost mariadb-10.2.23]# cmake . -DCMAKE_INSTALL_PREFIX=/app/mysql -DMYSQL_DATADIR=/data/mysql/ -DSYSCONFDIR=/etc/mysql -DMYSQL_USER=mysql -DWITH_INNOBASE_STORAGE_ENGINE=1  -DWITH_ARCHIVE_STORAGE_ENGINE=1  -DWITH_BLACKHOLE_STORAGE_ENGINE=1  -DWITH_PARTITION_STORAGE_ENGINE=1  -DWITHOUT_MROONGA_STORAGE_ENGINE=1  -DWITH_DEBUG=0  -DWITH_READLINE=1  -DWITH_SSL=system  -DWITH_ZLIB=system  -DWITH_LIBWRAP=0  -DENABLED_LOCAL_INFILE=1  -DMYSQL_UNIX_ADDR=/data/mysql/mysql.sock  -DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci -DPLUGIN_TOKUDB=NO

# -DCMAKE_INSTALL_PREFIX=/app/mysql 			安装路径
# -DMYSQL_DATADIR=/data/mysql/ \ 数据库文件存放位置
# -DSYSCONFDIR=/etc/mysql \ 配置文件存放位置
# -DMYSQL_USER=mysql \	用户
#-DMYSQL_UNIX_ADDR=/mysql/mysql.sock \ socket文件存放位置

# 编译安装,时间较长
[root@localhost mariadb-10.2.23]# make && make install

```
> 上面的编译安装的过程中,如果出错执行:rm -f CMakeCache.txt

4. 配置环境变量
```bash
[root@localhost ~]# echo 'PATH=/app/mysql/bin:$PATH' > /etc/profile.d/mysql.sh
```

5. 生成数据文件
```bash
[root@localhost ~]# /app/mysql/scripts/mysql_install_db --user=mysql --datadir=/data/mysql
Installing MariaDB/MySQL system tables in '/mysql' ...
OK
```
6. 准备配置文件
```bash
[root@localhost ~]# cp /app/mysql/support-files/my-huge.cnf /etc/my.cnf
[root@localhost ~]# vim /etc/my.cnf
# 在 mysqld 字段添加一下配置
datadir         =/data/mysql
innodb_file_per_table = on
skip_name_resolve = on
```

7. 准备启动脚本文件
```bash
[root@localhost ~]# cp /app/mysql/support-files/mysql.server /etc/init.d/mysqld
```

8. 启动服务并设置开始自启动
```bash
[root@localhost ~]# chkconfig --level 2345 --add mysqld
[root@localhost ~]# /etc/init.d/mysqld start
```