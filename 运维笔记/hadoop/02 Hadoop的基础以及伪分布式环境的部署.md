# Hadoop
官方网站:hadoop.apache.org
Hadoop=HDFS(分布式文件系统,负责底层的存储)+MapReduce(批处理系统,负责处理数据)

Hadoop可用于一般的商用服务器上,具有高容错.高可靠性.高扩展性等特点,特别适合一次写多次读的场景,不适用于低延时的数据访问.大量的小文件和需要频繁的修改文件

## Hadoop2的架构
![](http://aishad.top/wordpress/wp-content/uploads/2019/10/061826464f93ad5e8deef45d5d0a17f4.png)

HDFS:分布式文件存储系统

YARN:分布式资源管理系统

MapReduce:分布式计算

Others:利用YARN的资源管理功能实现的其它的数据处理方式

> 各个节点基本上都采用的Master-Worker架构


### Hadoop的部署方式

1. 单机模型:测试使用

2. 伪分布式模型:运行与单机

3. 分布式模型:集群模型

### 单节点的Hadoop的部署

1. 配置hadoop需要的java环境

```bash
tar xvf jdk-8u212-linux-x64.tar.gz -C /usr/local/src/
ln -sv /usr/local/src/jdk1.8.0_212 /usr/local/src/jdk

#设置环境变量
vim /etc/profile
export HISTTIMEFORMAT="%F %T `whoami` "
export export LANG="en_US.utf-8"
export JAVA_HOME=/usr/local/src/jdk
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
```

2. 创建hadoop的程序用户hadoop
```bash
useradd hadoop -m -s /bin/bash
```
3. 下载hadoop(当前使用的版本为2.7.7)
软件下载地址:https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/
```bash
wget https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-2.7.7/hadoop-2.7.7.tar.gz

# 解压程序包

tar -xf hadoop-2.7.7.tar.gz -C /home/hadoop

# 更改文件的属主和属组
ln -sv /home/hadoop/hadoop-2.7.7 /home/hadoop/hadoop

chown -R hadoop:hadoop /home/hadoop/
# hadoop目录下文件的说明
root@hbase-master:~# tree /home/hadoop/hadoop -L 1
/home/hadoop/hadoop
├── bin					# 存放hadoop的二进制程序文件
├── etc					# .sh结尾的文件为配置hadoop某个程序运行环境的脚本,.xml结尾的文件为Hadoop程序的配置文件
├── include				# 存放hadoop程序的头文件目录
├── lib					# 存放hadoop的库文件
├── libexec				# 存放hadoop库文件
├── LICENSE.txt
├── NOTICE.txt
├── README.txt
├── sbin				# 存放hadoop的启动或停止的脚本
└── share				# 存放hadoop
```

4. 修改hadoop的环境配置文件
```bash
vim /etc/profile.d/hadoop.sh

export HADOOP_PREFIX=/home/hadoop/hadoop
export PATH=$PATH:${HADOOP_PREFIX}/bin:${HADOOP_PREFIX}/sbin
export HADOOP_YARN_HOME=${HADOOP_PREFIX}
export HADOOP_MAPPRED_HOME=${HADOOP_PREFIX}
export HADOOP_COMMON_HOME=${HADOOP_PREFIX}
export HADOOP_HDFS_HOME=${HADOOP_PREFIX}

# 载入配置文件
source /etc/profile.d/hadoop.sh
```

5. 创建hadoop的数据目录和日志目录
```bash
# 创建hdfs的数据存放目录
mkdir -pv /home/hadoop/hadoop/data/hdfs/{nn,snn,dn}
chown hadoop:hadoop /home/hadoop/hadoop/data -R
# 创建日志目录
mkdir /home/hadoop/hadoop/logs
chown hadoop:hadoop /home/hadoop/hadoop/logs
```

#### 配置hadoop

1. core-site.xml 配置文件
core-site.xml 配置文件包含了NameNode主机地址以及监听的RPC端口等信息,对于伪分布式模型来说,其主机地址为localhost,NameNode默认使用的RPC端口为8020
```bash
vim /home/hadoop/hadoop/etc/hadoop/core-site.xml

<configuration>
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://localhost:8020</value>
                <final>true</final>
        </property>
</configuration>
```


2. hdfs-site.xml配置文件
主要用于配置HDFS相关的属性,例如副本数.NN和DN用于存储的数据目录等,数据块的副本数对于伪分布式的hadoop来说应该为1,而NN和DN用于存储的数据的目录为前面的步骤中创建的路径,另外,前面的步骤也为SNN创建了相关的目录,这里也一并配置其为启动状态
```bash
vim /home/hadoop/hadoop/etc/hadoop/hdfs-site.xml

<configuration>
		# 定义副本数量
        <property>
                <name>dfs.replication</name>
                <value>1</value>
        </property>
		# 定义namenode节点名称数据的存放位置
		 <property>
                <name>dfs.namenode.name.dir</name>
                <value>file:///home/hadoop/hadoop/data/hdfs/nn</value>
        </property>
		# 定义数据节点的数据目录位置
        <property>
                <name>dfs.datanode.data.dir</name>
                <value>file:///home/hadoop/hadoop/data/hdfs/dn</value>
        </property>
		# 定义检查点文件的存放路径
        <property>
                <name>fs.checkpoint.dir</name>
                <value>file:///home/hadoop/hadoop/data/hdfs/snn</value>
        </property>
		# 定义检查点的编辑目录
		<property>
                <name>fs.checkpoint.edits.dir</name>
                <value>file:///home/hadoop/hadoop/data/hdfs/snn</value>
        </property>
		# 如果需要其它用户对hdfs有写入权限,还需定义下面的属性
		<property>
                <name>dfs.permissions</name>
                <value>false</value>
        </property>
</configuration>

```

3. mapred-site.xml 配置文件
主要用于集群的的MapReduce framework(制定MapReduce是运行在yarn上还是自己直接运行),此处应该制定使用yarn,另外的可用值还有local(使用本地的而不是集群)和classic(使用经典的MapReduce运行机制),该配置文件默认不存在,但是在配置文件目录下有模板文件
```bash
cp  /home/hadoop/hadoop/etc/hadoop/mapred-site.xml.template /home/hadoop/hadoop/etc/hadoop/mapred-site.xml

vim /home/hadoop/hadoop/etc/hadoop/mapred-site.xml
# 应以MapReduce使用的机制是yarn
<configuration>
        <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
        </property>
</configuration>

```

4. yarn-site.xml 配置文件
主要用于配置yarn进程以及yarn的相关属性,首先需要制定ResourceManager守护进程的主机和监听的端口,对于伪分布式模型来讲,其主机为localhost,默认的端口为8032;其次需要制定ResourceManager使用的scheduler(调度器),以及NodeManager的辅助服务
```bash
vim /home/hadoop/hadoop/etc/hadoop/yarn-site.xml

<configuration>

<!-- Site specific YARN configuration properties -->
		# 配置ResourceManager监听的地址和端口
        <property>
                <name>yarn.resourcemanager.address</name>
                <value>localhost:8032</value>
        </property>
		# 配置调度器监听的地址和端口
        <property>
                <name>yarn.resourcemanager.scheduler.address</name>
                <value>localhost:8030</value>
        </property>
		# 配置资源追踪器监听的地址和端口
        <property>
                <name>yarn.resourcemanager.resource-tracker.address</name>
                <value>localhost:8031</value>
        </property>
		# 配置管理地址
        <property>
                <name>yarn.resourcemanager.admin.address</name>
                <value>localhost:8033</value>
        </property>
		# 配置web应用程序的地址
		<property>
                <name>yarn.resourcemanager.webapp.address</name>
                <value>localhost:8088</value>
        </property>
		# 辅助程序
		<property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>
		# 辅助服务使用的类
        <property>
                <name>yarn.nodemanager.auxservices.mapreduce_shuffle.class</name>
                <value>org.apache.hadoop.mapred.ShuffleHandler</value>
        </property>
		# 调度器使用的类
		<property>
                <name>yarn.resourcemanager.scheduler.class</name>                <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
        </property>



</configuration>

```

5. 配置Hadoop的守护进程环境(暂时使用默认)
管理员可使用hadoop/etc/hadoop/hadoop-env.sh 脚本定制hadoop守护进程的站点特有的环境变量;另外可选用的脚本还有hadoop/etc/hadoop/mapred-env.sh和hadoop/etc/hadoop/yarn-env.sh两个

6. slave配置文件
储存当前集群所有slave节点的列表,对于伪分布式模式来说,其文件内容仅有localhost

#### 格式化hdfs
在hdfs的NN启动之前,需要先初始化其用于存储数据的目录,如果hdfs-site.xml中dfs.namenode.name.dir指定目录不存在,格式化命令会自动将其创建,如果存在,请确保其权限设置正确即可,此格式化操作会清除其内部的所有数据并建立一个新的文件系统,需要在hadoop的身份下执行
```bash
su - hadoop
hdfs namenode -format

# 显示一下信息初始化成功
INFO common.Storage: Storage directory /home/hadoop/hadoop/data/hdfs/nn has been successfully formatted
# 验证信息
ls hadoop/data/hdfs/nn/current/
fsimage_0000000000000000000  fsimage_0000000000000000000.md5  seen_txid  VERSION
```
> 作为NameNode节点来说,会将内存中的元数据周期性的持久化到hadoop/data/hdfs/nn/current/fsimage文件中,但是由于不能实时的去做持久化的操作,元数据会先存储在checkpoint文件当中,checkpoint文件的维护由snn进行,存储在hadoop/data/hdfs/snn/目录中

7. 启动hadoop服务
hdfs监听的端口为:50010.50020.50070.50090.50075

hadoop2 的启动等操作可通过其位于sbin路径下的专用脚本进行
NameNode:hadoop-daemon.sh (start|stop) namenode
DataNode:hadoop-daemon.sh (start|stop) datanode
Secondary NameNode:hadoop-daemon.sh (start|stop) secondarynamenode
ResourceManager:yarn-daemon.sh (start|stop) resourcemanager
NodeManager:yarn-daemon.sh (start|stop) nodemanager
```bash
# 使用hadoop用户启动程序
# 先启动hdfs的三个守护进程 namenode,datanode和secondarynamenode
hadoop@hbase-master:~$ hadoop-daemon.sh start namenode
starting namenode, logging to /home/hadoop/hadoop/logs/hadoop-hadoop-namenode-hbase-master.out

hadoop@hbase-master:~$ hadoop-daemon.sh start datanode
starting datanode, logging to /home/hadoop/hadoop/logs/hadoop-hadoop-datanode-hbase-master.out

hadoop@hbase-master:~$ hadoop-daemon.sh start secondarynamenode
starting secondarynamenode, logging to /home/hadoop/hadoop/logs/hadoop-hadoop-secondarynamenode-hbase-master.out


# 上述命令执行后给出了一个日志信息的保存指向,但是实际用于保存日志的文件是以.log结尾的文件,而非.out结尾,可通过日志文件中的信息来判断进程是否启动成功,如果进程全部启动正常,可以通过jps来查看相关的java进程状态,-v选项可以查看详细信息
hadoop@hbase-master:~$ jps
2273 DataNode
2392 SecondaryNameNode
2172 NameNode
2445 Jps
```

8. 启动yarn服务
yarn监听的端口为:8020.8030.8031.8032.8033.8040.8042.8088
```bash
# yarn有两个守护进程resourcemanager和nodemanager
hadoop@hbase-master:~$ yarn-daemon.sh start resourcemanager
starting resourcemanager, logging to /home/hadoop/hadoop/logs/yarn-hadoop-resourcemanager-hbase-master.out

hadoop@hbase-master:~$ yarn-daemon.sh start nodemanager
starting nodemanager, logging to /home/hadoop/hadoop/logs/yarn-hadoop-nodemanager-hbase-master.out

hadoop@hbase-master:~$ jps
2273 DataNode
2392 SecondaryNameNode
2952 ResourceManager
3195 NodeManager
2172 NameNode
3230 Jps

# 完全分布式模式下NameNode运行在主节点,DataNode运行在各个从节点
```
9. 上传文件测试
文件的属主为hadoop,属组为supergroup,上传的文件保存在本地的 hadoop/data/hdfs/dn/current/BP-2022395624-192.168.1.100-1571745813664/current/finalized/subdir0/subdir0/ 目录下对应的块编号的文件中
```bash
# 使用hdfs命令的dfs子命令
# 查看存在的文件
hadoop@hbase-master:~$ hdfs dfs -ls /
# 上传文件测试
hadoop@hbase-master:~$ hdfs dfs -put /etc/fstab /fstab
hadoop@hbase-master:~$ hdfs dfs -ls /
Found 1 items
-rw-r--r--   1 hadoop supergroup        483 2019-10-22 20:30 /fstab
# 创建测试目录
hadoop@hbase-master:~$ hdfs dfs -mkdir /test
hadoop@hbase-master:~$ hdfs dfs -ls /
Found 2 items
-rw-r--r--   1 hadoop supergroup        483 2019-10-22 20:30 /fstab
drwxr-xr-x   - hadoop supergroup          0 2019-10-22 20:31 /test
# 再次上传文件到测试目录
hadoop@hbase-master:~$ hdfs dfs -put /etc/fstab /test
# 递归显示目录
hadoop@hbase-master:~$ hdfs dfs -lsr /
lsr: DEPRECATED: Please use 'ls -R' instead.
-rw-r--r--   1 hadoop supergroup        483 2019-10-22 20:30 /fstab
drwxr-xr-x   - hadoop supergroup          0 2019-10-22 20:34 /test
-rw-r--r--   1 hadoop supergroup        483 2019-10-22 20:34 /test/fstab
```

#### hadoop的web UI接口
hdfs和yarn ResourceManager各自提供了一个web端口,通过这些接口可以检查hdfs集群以及yarn集群的相关信息,他们的访问接口分别如下:
```bash
# 监听的地址和端口可以根据实际情况更改
HDFS-NameHost:http://<NameNnodeHost>:50070/
YARN-ResourceManager:http://<ResourceManagerHost>:8088/
注意:yarn-site.xml的文件中yarn.resourcemanager.webapp.address 定义为localhost:8088那么直监听本机的8088端口
```

hdfs状态信息查看web界面
![](http://aishad.top/wordpress/wp-content/uploads/2019/10/a6ca42b665b045c800d22fee845e9cb4.png)

hdfs的NameNode日志信息
![](http://aishad.top/wordpress/wp-content/uploads/2019/10/bb70c90e638f75dfdaea25d621806ba5.png)

> 各个节点的fsimage文件通过edits_log(相当于关系数据库的二进制日志)进行数据的合并,用于持久所有节点上的元数据信息

yarn状态信息查看web界面
![](http://aishad.top/wordpress/wp-content/uploads/2019/10/93e5a9dd187faedb7bc5d372224a58c1.png)


## 通过hadoop自带的测试程序运行MapReduce作业
yarn自带了许多的测试样例
自带的测试程序:/home/hadoop/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.7.jar
```bash
# 查看测试样例中的测试程序
hadoop@hbase-master:~$ yarn jar /home/hadoop/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.7.jar
An example program must be given as the first argument.
Valid program names are:
  aggregatewordcount: An Aggregate based map/reduce program that counts the words in the input files.
  aggregatewordhist: An Aggregate based map/reduce program that computes the histogram of the words in the input files.
  bbp: A map/reduce program that uses Bailey-Borwein-Plouffe to compute exact digits of Pi.
  dbcount: An example job that count the pageview counts from a database.
  distbbp: A map/reduce program that uses a BBP-type formula to compute exact bits of Pi.
  grep: A map/reduce program that counts the matches of a regex in the input.
  join: A job that effects a join over sorted, equally partitioned datasets
  multifilewc: A job that counts words from several files.
  pentomino: A map/reduce tile laying program to find solutions to pentomino problems.
  pi: A map/reduce program that estimates Pi using a quasi-Monte Carlo method.
  randomtextwriter: A map/reduce program that writes 10GB of random textual data per node.
  randomwriter: A map/reduce program that writes 10GB of random data per node.
  secondarysort: An example defining a secondary sort to the reduce.
  sort: A map/reduce program that sorts the data written by the random writer.
  sudoku: A sudoku solver.
  teragen: Generate data for the terasort
  terasort: Run the terasort
  teravalidate: Checking results of terasort
  wordcount: A map/reduce program that counts the words in the input files.
  wordmean: A map/reduce program that counts the average length of the words in the input files.
  wordmedian: A map/reduce program that counts the median length of the words in the input files.
  wordstandarddeviation: A map/reduce program that counts the standard deviation of the length of the words in the input files.
```
1. 使用wordcount测试程序
统计指定文件中的各个单词出现的次数
```bash
hadoop@hbase-master:~$ yarn jar /home/hadoop/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.7.jar wordcount /fstab /fstab.out
19/10/22 21:21:47 INFO client.RMProxy: Connecting to ResourceManager at localhost/127.0.0.1:8032
19/10/22 21:21:49 INFO input.FileInputFormat: Total input paths to process : 1
19/10/22 21:21:49 INFO mapreduce.JobSubmitter: number of splits:1
19/10/22 21:21:50 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1571749219761_0001
19/10/22 21:21:51 INFO impl.YarnClientImpl: Submitted application application_1571749219761_0001
19/10/22 21:21:51 INFO mapreduce.Job: The url to track the job: http://hbase-master:8088/proxy/application_1571749219761_0001/
19/10/22 21:21:51 INFO mapreduce.Job: Running job: job_1571749219761_0001
19/10/22 21:22:06 INFO mapreduce.Job: Job job_1571749219761_0001 running in uber mode : false
19/10/22 21:22:06 INFO mapreduce.Job:  map 0% reduce 0%
19/10/22 21:22:20 INFO mapreduce.Job:  map 100% reduce 0%
19/10/22 21:22:32 INFO mapreduce.Job:  map 100% reduce 100%
19/10/22 21:22:33 INFO mapreduce.Job: Job job_1571749219761_0001 completed successfully
...

# 查看输出的内容
hadoop@hbase-master:~$ hdfs dfs -ls /fstab.out
Found 2 items
-rw-r--r--   1 hadoop supergroup          0 2019-10-22 21:22 /fstab.out/_SUCCESS
-rw-r--r--   1 hadoop supergroup        512 2019-10-22 21:22 /fstab.out/part-r-00000
hadoop@hbase-master:~$ hdfs dfs -cat /fstab.out/part-r-00000
#	7
'blkid'	1
/	1
/dev/mapper/ubuntu--vg-root	1
/dev/mapper/ubuntu--vg-swap_1	1
/etc/fstab:	1
0	3
1	1
<dump>	1
<file	1
<mount	1
<options>	1
<pass>	1
<type>	1
...
```