# Redis的配置
- 配置文件：
	yum安装：/etc/redis.conf
	编译安装：/apps/redis/etc/redis.conf

## Redis的主要配置项：

1. bind 0.0.0.0 
	监听的地址，可以使用空格隔开多个

2. protected-mode yes 
	redis3.2 之后加入的新特性，在没有设置 bind IP 和密码的时候只允许访问127.0.0.1:6379，无法远程访问

3. port 6379 
	监听的端口

4. tcp-backlog 511
	三次握手的时候 server 端收到 client ack 确认号之后的队列值。

5. timeout 0 
	客户端和 Redis 服务端的连接超时时间，默认是 0，表示永不超时。

6. tcp-keepalive 300
	tcp 会话保持时间

7. daemonize no
	默认情况下redis 不是作为守护进程运行的，如果你想让它在后台运行，你就把它改成yes,当 redis 作为守护进程运行的时候，它会写一个 pid 到 /var/run/redis.pid 文件里面

8. supervised no
	和操作系统相关参数，可以设置通过 upstart 和 systemd 管理 Redis 守护进程，centos 7以后都使用 systemd

9. pidfile /var/run/redis_6379.pid 
	pid 文件路径

10. loglevel notice ：日志级别
	debug：保存了所有信息，仅用于开发和测试
	verbose：许多很少有用的信息，但不像调试级别那么混乱
	notice：默认，保存了一些常规的信息
	warning：只记录非常重要/关键的消息

11. logfile "/apps/redis/logs/redis_6379.log" ：日志路径

12. databases 16 :设置 db 库数量，默认 16 个库 0-15
	在redis中使用"select #"，切换到指定的数据库，redis中再相同的库中，key的值不能相同，但是在不同的的库中，可以存在相同的key值

13. always-show-logo yes 
	在启动 redis 时是否显示 log

### Redis RDB模式配置

1. 持久化相关的参数：默认是3个，可以写多个，多个参数之间是"或"的关系，只要其中有一个满足即可
	save 900 1  ：在 900 秒内有一个key内容发生更改就出就快照机制
	save 300 10
	save 60 10000

2. stop-writes-on-bgsave-error no 
	快照出错时是否禁止 redis 写入操作,一定要关闭，防止内存不足时，快照不成功导致redis不能写入数据

16. rdbcompression yes
	持久化到 RDB 文件时，是否压缩，"yes"为压缩，"no"则反之

17. rdbchecksum yes 
	是否开启 RC64 校验，默认是开启

18. dbfilename dump_6379.rdb 
	快照文件名

19. dir /apps/redis/data
	快照文件保存路径

### Redis主从节点配置

1. replica-serve-stale-data yes
	当从库同主库失去连接或者复制正在进行，从机库有两种运行方式：1) 如果 replica-serve-stale-data 设置为 yes(默认设置)，从库会继续响应客户端的读请求。2) 如果 replicaserve-stale-data 设置为 no，除去指定的命令之外的任何请求都会返回一个错误"SYNC with master inprogress"。

2. replica-read-only yes 
	是否设置从库只读

22. repl-diskless-sync no
	是否使用 socket 方式复制数据，目前 redis 复制提供两种方式，disk 和 socket，如果新的 slave 连上来或者重连的 slave 无法部分同步，就会执行全量同步，master 会生成 rdb 文件，有2 种方式：disk 方式是 master 创建一个新的进程把 rdb 文件保存到磁盘，再把磁盘上的 rdb 文件传递给 slave，socket 是 master 创建一个新的进程，直接把 rdb 文件以 socket 的方式发给 slave，disk 方式的时候，当一个 rdb 保存的过程中，多个 slave 都能共享这个 rdb 文件，socket 的方式就是一个个 slave顺序复制，只有在磁盘速度缓慢但是网络相对较快的情况下才使用 socket 方式，否则使用默认的 disk方式

23. repl-diskless-sync-delay 30
	diskless 复制的延迟时间，一旦复制开始还没有结束之前，master 节点不会再接收新 slave 的复制请求，直到下一次开始，设置 0 为关闭

24. repl-ping-slave-period 10 
	slave 根据 master 指定的时间进行周期性的 PING 监测

25. repl-timeout 60 
	复制链接超时时间，需要大于 repl-ping-slave-period，否则会经常报超时

26. repl-disable-tcp-nodelay no 
	在 socket 模式下是否在 slave 套接字发送 SYNC 之后禁用 TCP_NODELAY，如果你选择“yes”Redis 将使用更少的 TCP 包和带宽来向 slaves 发送数据。但是这将使数据传输到 slave上有延迟，Linux 内核的默认配置会达到 40 毫秒，如果你选择了 "no" 数据传输到 salve 的延迟将会减少但要使用更多的带宽

27. repl-backlog-size 1mb 
	复制缓冲区大小，只有在 slave 连接之后才分配内存。

28. repl-backlog-ttl 3600 
	多长时间 master 没有 slave 连接，就清空 backlog 缓冲区。

29. replica-priority 100 
	当 master 不可用，Sentinel 会根据 slave 的优先级选举一个 master。最低的优先级的 slave，当选 master。而配置成 0，永远不会被选举。

### Redis安全和连接配置

1. requirepass foobared 
	设置 redis 连接密码，连接的时候用-a指定密码，或者在redis中使用auth指定密码

2. rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52 
	重命名一些高危命令

### Redis内存配置

1. maxclients 10000
	最大连接客户端

2. maxmemory <bytes\>
	最大内存，最好不要超过物理内存的一半，单位为 bytes 字节，8G 内存的计算方式 8(G)*1024(MB)*1024(KB)*1024(Kbyte)，需要注意的是 slave 的输出缓冲区是不计算在 maxmemory 内。

### AOF配置

1. appendonly no
	是否开启 AOF 日志记录，默认 redis 使用的是 rdb 方式持久化，这种方式在许多应用中已经足够用了。但是 redis 如果中途宕机，会导致可能有几分钟的数据丢失，根据 save 来策略进行持久化，Append Only File 是另一种持久化方式，可以提供更好的持久化特性。Redis 会把每次写入的数据在接收后都写入 appendonly.aof 文件，每次启动时 Redis 都会先把这个文件的数据读入内存里，先忽略 RDB 文件。

2. appendfilename "appendonly.aof"
	AOF 文件名，保存位置为上面设置的RDB文件的位置

> RDB文件的优先级没有AOF文件的优先级高，系统在重启后，默认先读取AOF文件恢复数据

3. appendfsync everysec
	aof 持久化策略的配置,no 表示不执行 fsync,由操作系统保证数据同步到磁盘,always 表示每次写入都执行 fsync，以保证数据同步到磁盘,everysec 表示每秒执行一次 fsync，可能会导致丢失这 1s 数据

4. no-appendfsync-on-rewrite no
	在 aof rewrite 期间,是否对 aof 新记录的 append 暂缓使用文件同步策略,主要考虑磁盘 IO 开支和请求阻塞时间。默认为 no,表示"不暂缓",新的 aof 记录仍然会被立即同步，Linux 的默认 fsync 策略是 30 秒，如果为 yes 可能丢失 30 秒数据，但由于 yes 性能较好而且会避免出现阻塞因此比较推荐。

5. auto-aof-rewrite-percentage 100 
	当 Aof log 增长超过指定百分比例时，重写 log file， 设置为 0 表示不自动重写 Aof 日志，重写是为了使 aof 体积保持最小，而确保保存最完整的数据。

6. auto-aof-rewrite-min-size 64mb
	触发 aof rewrite 的最小文件大小，当文件达到128M的时候，会触发重写，假如重写后的文件为100M，当文件再次达到200M的时候回再次触发重写，以此类推

7. aof-load-truncated yes
	是否加载由于其他原因导致的末尾异常的 AOF 文件(主进程被 kill/断电等)
	redis-check-aof --fix AOF文件 或 redis-check-rdb RDB文件：修复异常的文件

8. aof-use-rdb-preamble yes
	redis4.0 新增 RDB-AOF 混合持久化格式，在开启了这个功能之后，AOF 重写产生的文件将同时包含 RDB 格式的内容和 AOF 格式的内容，其中 RDB 格式的内容用于记录已有的数据，而 AOF 格式的内存则用于记录最近发生了变化的数据，这样 Redis 就可以同时兼有 RDB 持久化和AOF 持久化的优点（既能够快速地生成重写文件，也能够在出现问题时，快速地载入数据）。

9. lua-time-limit 5000 
	lua 脚本的最大执行时间，单位为毫秒

### 集群模式下的配置：

> 以下配置会由redis在集群模式下自动生成

1. cluster-enabled yes
	是否开启集群模式，默认是单机模式

2. cluster-config-file nodes-6379.conf
	由 node 节点自动生成的集群配置文件

3. cluster-node-timeout 15000
	集群中 node 节点连接超时时间

4. cluster-replica-validity-factor 10
	在执行故障转移的时候可能有些节点和 master 断开一段时间数据比较旧，这些节点就不适用于选举为 master，超过这个时间的就不会被进行故障转移

5. cluster-migration-barrier 1
	一个主节点拥有的至少正常工作的从节点，即如果主节点的 slave 节点故障后会将多余的从节点分配到当前主节点成为其新的从节点。(类似RAID中的热备盘)

6. cluster-require-full-coverage no
	集群槽位覆盖，如果一个主库宕机且没有备库就会出现集群槽位不全，那么 yes 情况下 redis 集群槽位验证不全就不再对外提供服务，而 no 则可以继续使用但是会出现查询数据查不到的情况(因为有数据丢失)。

### 慢日志配置
	#Slow log 是 Redis 用来记录查询执行时间的日志系统，slow log 保存在内存里面，读写速度非常快，因此你可以放心地使用它，不必担心因为开启 slow log 而损害 Redis 的速度。

1. slowlog-log-slower-than 10000
	以微秒为单位的慢日志记录，为负数会禁用慢日志，为 0 会记录每个命令操作。

2. slowlog-max-len 128
	记录多少条慢查询日志保存在队列，超出后会删除最早的，以此滚动删除

-  slowlog len：查看当前有多少条慢查询日志

-  slowlog get：获取当前的慢查询日志








### redis数据持久化
	redis 虽然是一个内存级别的缓存程序，即 redis 是使用内存进行数据的缓存的，但是其可以将内存的数据按照一定的策略保存到硬盘上，从而实现数据持久保存的目的，redis 支持两种不同方式的数据持久化保存机制，分别是 RDB 和 AOF

1. RDB模式
	基于时间的快照，只保留当前最新的一次快照，特点是执行速度比较快，缺点是可能会丢失从上次快照到当前快照未完成之间的数据。

- RDB的实现过程：
	RDB 实现的具体过程 Redis 从主进程先 fork 出一个子进程，使用写时复制机制，子进程将内存的数据保存为一个临时文件，比如 dump.rdb.temp，当数据保存完成之后再将上一次保存的 RDB 文件替换掉，然后关闭子进程，这样可以保存每一次做 RDB 快照的时候保存的数据都是完整的，因为直接替换 RDB文件的时候可能会出现突然断电等问题而导致 RDB 文件还没有保存完整就突然关机停止保存而导致数据丢失的情况，可以手动将每次生成的 RDB 文件进程备份，这样可以最大化保存历史数据。

- 优点：
	1）RDB 快照保存了某个时间点的数据，可以通过脚本执行 bgsave(非阻塞)或者 save(阻塞)命令自定义时间点备份，可以保留多个备份，当出现问题可以恢复到不同时间点的版本。
	2）可以最大化 o 的性能，因为父进程在保存 RDB 文件的时候唯一要做的是 fork 出一个子进程，然后的
	3）操作都会有这个子进程操作，父进程无需任何的 IO 操作
	4）RDB 在大量数据比如几个 G 的数据，恢复的速度比 AOF 的快

- 缺点：
	1）不能实时的保存数据，会丢失自上一次执行 RDB 备份到当前的内存数据
	2）数据量非常大的时候，从父进程 fork 的时候需要一点时间，可能是毫秒或者秒

2. AOF模式
	按照操作顺序依次将操作添加到指定的日志文件当中，特点是数据安全性相对较高，缺点是即使有些操作是重复的也会全部记录。(相当于MySQL的binlog)
	AOF 和 RDB 一样使用了写时复制机制，AOF 默认为每秒钟 fsync 一次，即将执行的命令保存到 AOF 文件当中，这样即使 redis 服务器发生故障的话顶多也就丢失 1 秒钟之内的数据，也可以设置不同的 fsync策略，或者设置每次执行命令的时候执行 fsync，fsync 会在后台执行线程，所以主线程可以继续处理用户的正常请求而不受到写入 AOF 文件的 IO 影响

- 优缺点：
	1）AOF 的文件大小要大于 RDB 格式的文件
	2）根据所使用的 fsync 策略(fsync 是同步内存中 redis 所有已经修改的文件到存储设备)，默认是appendfsync everysec 即每秒执行一次 fsync