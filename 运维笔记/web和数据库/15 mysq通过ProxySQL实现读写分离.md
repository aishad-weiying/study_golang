# mysq通过ProxySQL实现读写分离

 

## 常见的读写分离应用
		Oracle：mysql-proxy
		qihoo：Atlas
		美团：dbproxy
		网易：cetus
		amoeba
		阿里巴巴：cobar 基于amoeba研发
		Mycat：基于cobar实现
		ProxySQL



### ProxySQL：MySQL中间件
- 版本：  
	- 官方版  
	- percona版：percona公司基于官方版本用C++语言开发，性能更优  
- 特点：具有中间件所需的绝大多数功能，包括：  
	- 多种方式的读/写分离  
	- 定制基于用户、基于schema、基于语句的规则对SQL语句进行路由  
	- 缓存查询结果  
	- 后端节点监控  
> 官方站点：https://proxysql.com/


### ProxySQL安装：
- 准备：  
	- 实现读写分离前，先实现主从复制  
		
	> 注：slave服务器 配置文件中必须为 read_only=1，ProxySQL通过read_only=1参数，确定哪个是salve服务器  
	
- 基于YUM仓库安装：
```bash
    [ root@centos ~]# cat <<EOF | tee /etc/yum.repos.d/proxysql.repo
    [proxysql_repo]
    name= ProxySQL YUM repository
    baseurl=http://repo.proxysql.com/ProxySQL/proxysql-1.4.x/centos/\$releasever
    gpgcheck=1
    gpgkey=http://repo.proxysql.com/ProxySQL/repo_pub_key
    EOF
```

- 基于RPM下载安装：https://github.com/sysown/proxysql/releases

- ProxySQL的组成：
    - 服务脚本：/etc/init.d/proxysql
	- 配置文件：/etc/proxysql.cnf
	- 主程序：/usr/bin/proxysql
	- 基于SQLITE的数据库文件：/var/lib/proxysql/

- 启动ProxySQL：service proxysql start
	- 启动后会监听两个默认端口：
		- 6032：ProxySQL的管理端口
		- 6033：ProxySQL对外提供服务的端口

- 使用mysql客户端连接到ProxySQL的管理端口6032，默认管理员用户和密码都是admin
```bash
	[ root@weiying ~]# mysql -uadmin -padmin -p6032 -hhost
```

- ProxySQL实现读写分离:
	> 内置了SQLite小型数据库，里面存储了proxysql的设置
	- 内置的数据库说明：
		- main：是默认的数据库名，表里面存放后端db实例，用户验证，路由规则等信息，表名以runtime_开头表示ProxySQL当前运行的配置内容，不能通过dml语句修改，只能修改对应的不以runtime_开头的表，然后LOAD使其生效，save使其保存到硬盘一共下次重启加载
		- disk：是持久化到停盘的配置，sqlite数据文件
		- stats：是ProxySQL运行抓取到的统计信息，包括到后端各命令的执行次数、流量、processlist、查询种类汇总/执行时间、等等
		- monnitor：库存储monitor模块收集的信息，主要是对后端db的健康/延迟检查
			> 注：监控模块的指标存在log表中

        - 说明：
            1. 在main和monitor数据库中的表，runtime_开头的是运行时的配置，不能修改，只能修改非runtime_表
            2. 修改后必须执行LOAD … TO RUNTIME才能加载到RUNTIME生效
            3. 执行save … to disk 才将配置持久化保存到磁盘，即保存在proxysql.db文件中
            4. global_variables 有许多变量可以设置，其中就包括监听的端口、管理账号等   
            参考: https://github.com/sysown/proxysql/wiki/Global-variables

配置- ProxySQL：  
- 向ProxySQL中的main库中指定mysql节点(不需要使用use main也可以)：
```bash
    1. 查看指定的表结构：
        select * from sqlite_master where name='mysql_servers'\G 
    2. 添加所有参与主从复制的主机： 
        insert into mysql_servers(hostgroup_id,hostname,port) values(10,'172.22.45.131',3306);  
         insert into mysql_servers(hostgroup_id,hostname,port) values(10,'172.22.45.132',3306);  
    3. 加载到runtime中使其生效：
        load mysql servers to runtime;
    4. 保存到硬盘中：
        save mysql servers to disk;
    字段说明：  
        hostgroup_id：分组id，用来实现区分读组和写组，后续可通过ProxySQL程序自动判断  
        hostname：主从服务器的地址  
        port：主从服务器监听的端口号
```
- master和slave节点操作：
    - 添加监控后端节点的用户，ProxySQL通过每个节点的read_only值来自动调整它们是属于读组还是写组  
        - 主从节点创建用户：
```bash
            grant replication client on *.* to monitor@'172.22.45.%' identified by 'centos'; #用来实现proxysql连接主从节点
```
- ProxySQL上配置监控：
```bash
    set mysql-monitor_username='monitor';
    set mysql-monitor_password='centos';
    load mysql variables to runtime;
    save mysql variables to disk;	

    查看监控连接是否正常：
        select * from mysql_server_connect_log;
    查看监控心跳信息（对ping指标的监控）：
        select * from mysql_server_ping_log;
    查看read_only和replication_lag的监控日志
        select * from mysql_server_read_only_log;
        select * from mysql_server_replication_lag_log;
```
- 设置分组信息：
	- 需要修改的是main库中的mysql_replication_hostgroups表，该表有3个字段：writer_hostgroup，reader_hostgroup，comment, 指定写组的id为10，读组的id为20
```bash
        insert inot mysql_replication_hostgroups values(10,20,'test');
        load mysql servers to runtime;
        save mysql servers to disk;
       # Monitor模块监控后端的read_only值，按照read_only的值将节点自动移动到读/写组
       查看主从服务器的分组信息：
        select hostgroup_id,hostname,port,status,weight from mysql_servers; 
        +--------------+---------------+------+--------+--------+
|        hostgroup_id | hostname      | port | status | weight |
        +--------------+---------------+------+--------+--------+
        | 10           | 172.22.45.131 | 3306 | ONLINE | 1      |
        | 20           | 172.22.45.132 | 3306 | ONLINE | 1      |
        +--------------+---------------+------+--------+--------+
```
- 配置发送SQL语句的用户：
	- 在master节点上创建访问用户
```bash
	grant all on *.* to sqluser@'host' identified by 'centos'; #用来让用户连接proxy使用
```
    - 在ProxySQL配置，将用户sqluser添加到mysql_users表中， default_hostgroup默认组设置为写组10，当读写分离的路由规则不符合时，会访问默认组的数据库
```bash
    insert into mysql_users(username,password,default_hostgroup)values('sqluser','magedu',10);
    load mysql servers to runtime;
    save mysql servers to disk;
```
	- 测试：目前由于没有设置读写分离的路由规则，则所有的读写语句都到默认的分组10中实现
```bash
    mysql -usqluser -pcentos -P6033 -h127.0.0.1 -e 'select @@server_id'
        +-------------+
        | @@server_id |
        +-------------+
        |         131 |
        +-------------+
    mysql -usqluser -pcentos -P6033 -h127.0.0.1 -e 'create database testdb'
    mysql -usqluser -pcentos testdb -P6033 -h127.0.0.1 -e 'create table t(id int)'
```
- 配置读写分离的路由规则：
	> 与规则有关的表：mysql_query_rules和mysql_query_rules_fast_routing，后者是前者的扩展表，1.4.7之后支持
	> 插入路由规则：将select语句分离到20的读组，select语句中有一个特殊语句SELECT...FOR UPDATE它会申请写锁，应路由到10的写组：
```bash
    insert into mysql_query_rules(rule_id,active,match_digest,destination_hostgroup,apply) VALUES(1,1,'^SELECT.*FOR UPDATE$',10,1),(2,1,'^SELECT',20,1);

    load mysql query rules to runtime;
    save mysql query rules to disk;
#注意：因ProxySQL根据rule_id顺序进行规则匹配，select ... for update规则的 rule_id必须要小于普通的select规则的rule_id
```
- 测试读操作是否路由给20的读组
```bash
		mysql -usqluser -pmagedu -P6033 -h127.0.0.1 -e 'select @@server_id'
            +-------------+
            | @@server_id |
            +-------------+
            |         131 |
            +-------------+
```
- 测试写操作，以事务方式进行测试
```bash
    mysql -usqluser -pcentos -P6033 -h172.22.45.133 -e 'start transaction;select @@server_id;commit;select @@server_id'
        +-------------+
        | @@server_id |
        +-------------+
        |         131 |
        +-------------+
        +-------------+
        | @@server_id |
        +-------------+
        |         132 |
        +-------------+
    mysql -usqluser -pcentos -P6033 -h127.0.0.1 -e 'insert testdb.t values (1)'
    mysql -usqluser -pcentos -P6033 -h127.0.0.1 -e 'select id from testdb.t'
```
- 路由的信息：查询stats库中的stats_mysql_query_digest表
```bash
	SELECT hostgroup hg,sum_time, count_star, digest_text FROM stats_mysql_query_digest ORDER BY sum_time DESC;
```