# MySQL高可用
- MMM：MySQL主主复制管理器是一套灵活的脚本程序，基于perl实现，用来对mysqk replication进行监控和故障迁移，并能管理mysql master-master复制的配置（同一时间只有一个节点是可写的）

- MHA：对主节点进行监控，可实现自动故障转移至其他从节点，通过提升某一节点为新的主节点，基于主从复制实现，还需要客户端配合时间，目前MHA主要支持一主多从的架构，要搭建MHA，要求一个复制集群汇总最少有三台数据库服务器，一主二从，即一太充当master，一台充当备用master，另外一台充当从库，由于机器成本的考虑，阿里巴巴对其进行了改造，目前TMHA已支持一主一从

- Galera Cluster：wsrep(MySQL extended with the Write Set Replication)，通过wsrep协议在全局实现复制，在任何一节点可读写，不需要主从复制，实现多主写

- GR：MySQL官方提供的组复制技术（MySQL 5.7.17引入的技术），基于原生复制技术Paxos算法

## MHA的工作原理：
1. 从宕机崩溃的master保存二进制日志事件（binlog event）
2. 识别含有最新的更新的slave
3. 应用差异的中继日志（relay log）到其他的slave
4. 应用从master保存的二进制日志事件
5. 提升一个slave为新的master
6. 使其他的slave连接新的master进行复制

### MHA软件包的两个组成部分：
	- Manager工具包：
	    masterha_check_ssh		检查MHA的ssh配置情况
	    masterha_check_repl		检查MySQL复制状况
	    masterha_manager		启动MHA
	    masterha_check_status	检测当前MHA运行状态
	    masterha_master_monitor	检测master是否宕机
	    masterha_master_switch 	故障转移（自动或手动）
	    masterha_conf_host		添加或删除配置的server信息
	- Node工具包：
	    save_biary_logs		保存和复制master的二进制日志
	    apply_diff_relay_logs	去除不必要的rollback时间（MHA已不再使用）
	    purge_relay_logs		清除中继日志（不会阻塞SQL线程）
> 注：
>   1. 这些工具通常是由MHA Manager的脚本触发，无需人为操作
>   2. 为了尽可能的减少主库硬件损坏宕机造成数据丢失，因此在此配置的MHA的同时建议配置成MySQL5.5的半同步复制

    - 自定义扩展：
        secondary_check_script		通过多条网络路由检测master的可用性
        mater_ip_ailover_script		更新application使用的master ip
        shutdown_script			强制关闭master节点
        report_script			发送报告
        init_conf_load_script		加载初始化配置参数
        master_ip_online_change_script 	更新master节点ip地址

### 配置文件：
        global配置：为各个application提供默认配置
        application配置：为每个主从复制集群提供默认配置
### 软件包：
- 管理节点的软件包：
    mha4mysql-manager
    mha4mysql-node
- 在被管理节点安装：
        mha4mysql-node
> 注：  
> 1.管理节点与主从节点要实现ssh基于key验证  
> 2.所有节点的时间要同步  

## 实现MHA：
1. 在所有节点实现相互之间ssh key验证

2. 在管理节点建立配置文件：
```bash
    vim /etc/mastermha/app1.cnf
        [server default]  	#MAH节点配置
        user=mhauser		#管理主从节点的账号
        password=admin123							
        manager_workdir=/data/mastermha/app1/
        manager_log=/data/mastermha/app1/manager.log
        remote_workdir=/data/mastermha/app1/
        ssh_user=root 		#通过ssh协议连接的用户
        repl_user=repluser 	#主从同步的用户
        repl_password=admin123
        ping_interval=1 	#检测master节点正常与否的频率
        [server1]
        hostname=172.22.45.131 #master节点
        candidate_master=1
        [server2]
        hostname=172.22.45.132 #master节点故障后，默认将此从节点提升为master
        candidate_master=1
        [server3]
        hostname=172.22.45.133
```
3. 主节点配置：
```bash
    [mysqld]
        log-bin
        server_id=1
        skip_name_resolve=131

    mysql>show master logs
    
    mysql>grant replication slave on *.* to repluser@'172.22.45.131.%' identified by 'admin123';
    
    mysql>grant all on *.* to mhauser@'172.22.45.131.%' identified by 'admin123';
```
4. 从节点1：
```bash
    vim /etc/my.cnf
        [mysqld]
        server_id=132 
        log-bin
        read_only
        relay_log_purge=0
        skip_name_resolve=1

    mysql>CHANGE MASTER TO MASTER_HOST='172.22.45.131',MASTER_USER='repluser', MASTER_PASSWORD='admin123',MASTER_LOG_FILE='mariadb-bin.000001', MASTER_LOG_POS=245;
```
5. 从节点2：
```bash
        vim /etc/my.cnf
        [mysqld]
        server_id=133 
        log-bin
        read_only
        relay_log_purge=0
        skip_name_resolve=1

        mysql>CHANGE MASTER TO MASTER_HOST='172.22.45.131',MASTER_USER='repluser', MASTER_PASSWORD='admin123',MASTER_LOG_FILE='mariadb-bin.000001', MASTER_LOG_POS=245;
```
6. 验证启动：
- Mha节点验证和启动
```bash
    masterha_check_ssh --conf=/etc/mastermha/app1.cnf
       # 验证ssh基于key验证是否正确 
    masterha_check_repl --conf=/etc/mastermha/app1.cnf
        # 验证主从服务器同步是否正确
    masterha_manager --conf=/etc/mastermha/app1.cnf
```
- 排错日志：  /data/mastermha/app1/manager.log
7. 测试：  
    杀掉mysql master主进程后看到MHA管理端退出并报告：  
        MySQL Master failover 172.22.45.131(172.22.45.131:3306) to 172.22.45.132(172.22.45.132:3306)
        
    - 将master从172.22.45.131迁移为172.22.45.132
    - 同时将172.22.45.132的read_only的值改为0： 
```bash 
            select @@read_only;  
                +-------------+  
                | @@read_only |  
                +-------------+  
                |           0 |  
                +-------------+  
```
    - 将172.22.45.133的主从复制的主服务器指向改为172.22.45.132

## Galera Cluster:
		继承了Galera插件的MySQL集群，是一种新型的，数据不共享的，高度冗余的高可用方案，目前目前Galera Cluster有两个版本，分别是Percona Xtradb Cluster及MariaDB Cluster，Galera本身是具有多主特性的，即采用multi-master的集群架构，是一个既稳健，又在数据一致性、完整性及高性能方面有出色表现的高可用解决方案
	
		如：三个节点组成了一个集群，与普通的主从架构不同，它们都可以作为主节点，三个节点是对等的，称为multi-master架构，当有客户端要写入或者读取数据时，连接哪个实例都是一样的，读到的数据是相同的，写入某一个节点之后，集群自己会将新数据同步到其它节点上面，这种架构不共享任何数据，是一种高冗余架构

- 特点：
    - 多主架构：真正的多点读写的集群，在任何时候读写数据，都是罪行的
	-  同步复制：集群不同节点之间的数据同步，没有延迟在数据库挂掉之后，数据不会丢失
    - 并发复制：从节点APPLY数据时，支持并行执行，更好的性能
    - 故障切换：在出现数据库故障时，因支持多点写入，切换容易
    - 热插拔：在服务期间，如果数据库挂了，只要监控程序发现的够快，不可服务时间就会非常少。在节点故障期间，节点本身对集群的影响非常小
    - 自动节点克隆：在新增节点，或者停机维护时，增量数据或者基础数据不需要人工手动备份提供，Galera Cluster会自动拉取在线节点数据，最终集群会变为一致
    - 对应用透明：集群的维护，对应用程序是透明的

- Galera Cluster包括两个组件：
    - Galera replication library (galera-3)
    - WSREP：MySQL extended with the Write Set Replication 
- WSREP复制实现：
    - PXC：Percona XtraDB Cluster，是Percona对Galera的实现
    - MariaDB Galera Cluster
        - 参考仓库：https://mirrors.tuna.tsinghua.edu.cn/mariadb/mariadb-5.5.64/yum/centos7-amd64/
> 注意：都至少需要三个节点，不能安装mariadb-server

### 安装部署：
1. 创建yum源并安装：
```bash
    vim /etc/yum.repos.d/ganlera.repo
        [galera]
        baseurl=https://mirrors.tuna.tsinghua.edu.cn/mariadb/mariadb-5.5.64/yum/centos7-amd64/
        gpgcheck=0
    yum install MariaDB-Galera-server
```
2. 更改配置文件： 
```bash
    vim /etc/my.cnf.d/server.cnf
        [galera]
        wsrep_provider = /usr/lib64/galera/libgalera_smm.so #模块文件路径
        wsrep_cluster_address="gcomm://172.22.45.131,172.22.45.132,172.22.45.133" #集群节点
        binlog_format=row #二进制日志记录格式
        default_storage_engine=InnoDB #存储引擎
        innodb_autoinc_lock_mode=2
        bind-address=0.0.0.0
        
        #下面配置可选项
            wsrep_cluster_name = 'mycluster'#集群名默认my_wsrep_cluster
            wsrep_node_name = 'node1' #每个节点的名字
            wsrep_node_address ='172.22.45.134'
```
3. 启动
    > 首次启动时，需要初始化集群，在其中一个节点上执行命令  
    - service mysql start --wsrep-new-cluster：开启一个新的集群节点
    -  而后正常启动其它节点  
            service mysql start
    - 查看集群中相关系统变量和状态变量  
            SHOW VARIABLES LIKE 'wsrep_%';  
            SHOW STATUS LIKE 'wsrep_%';  
            SHOW STATUS LIKE 'wsrep_cluster_size'; 查看集群节点个数  