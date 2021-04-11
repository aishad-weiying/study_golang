## NSQ的介绍

[NSQ](https://nsq.io/)是Go语言编写的一个开源的实时分布式内存消息队列,其性能十分优异,优势如下:

-  NSQ 提倡分布式和分散的拓扑,没有单点故障,支持容错和高可用性,并提供可靠的消息交付保障
-  NSQ 支持横向扩展,没有任何集中式代理
-  NSQ易于配置和部署,并且内置了管理界面

### NSQ 的使用场景

1. 异步处理
把业务流程中的非关键流程异步化,从而显著降低业务请求的响应的时间

[![](http://aishad.top/wordpress/wp-content/uploads/2020/08/%E5%BC%82%E6%AD%A5.png)](http://aishad.top/wordpress/wp-content/uploads/2020/08/%E5%BC%82%E6%AD%A5.png)

 2. 应用解耦
通过使用消息队列将不同的业务逻辑解耦,降低系统间的耦合,提高系统的健壮性,后续的其他业务要使用订单信息科直接订阅消息队列,提高系统的灵活性

[![](http://aishad.top/wordpress/wp-content/uploads/2020/08/%E8%A7%A3%E8%80%A6.png)](http://aishad.top/wordpress/wp-content/uploads/2020/08/%E8%A7%A3%E8%80%A6.png)

3. 流量削峰
类似秒杀等场景下,某一时间可能会产生大量的请求,使用消息队列能够为后端处理请求提供一定的缓冲区,保证后端服务的稳定性

[![](http://aishad.top/wordpress/wp-content/uploads/2020/08/%E6%B5%81%E9%87%8F%E5%89%8A%E5%B3%B0.png)](http://aishad.top/wordpress/wp-content/uploads/2020/08/%E6%B5%81%E9%87%8F%E5%89%8A%E5%B3%B0.png)

### NSQ的组件

1. nsqd
nsqd 是守护进程,它负责接收消息,将消息排队并向客户端发送消息

2. nsqlookupd
nsqlookupd 是维护所有nsqd 状态,提供服务发现的守护进程,它能为小分着查找特定`topic`下的 nsqd 提供了运行时的自动发现服务,它不维持持久状态,也不需要与其他的 nsqlookupd 示例协调以满足查询,因此根据熊的冗余要求尽可能多的部署`nslookupd`节点,它们消耗的资源很少,可以与其他服务共存

3. nsqadmin
一个试试监控集群状态,执行各种管理任务的 web 管理平台,启动 nsqadmin,需要指定 nsqlookupd 的地址

### NSQ 架构
[![](http://aishad.top/wordpress/wp-content/uploads/2020/08/nsql.png)](http://aishad.top/wordpress/wp-content/uploads/2020/08/nsql.png)

### Topic和Channel

每个 nsqd 示例皆在一次处理多个数据流,这些数据流称为`tocics`,一个`tocic`具有一个或多个`channels`,每个`channel`都会收到`tocic`所有消息的副本,实际上,上下游服务是通过 channel 来消费`tocic`消息的

`tocic` 和`channel`不是预先配置的,`tocic`在首次使用时创建,方法是将其发布到指定`tocic`,或者订阅自定`tocic`上的`channel`,`channel`是通过订阅指定的 channel 在第一次 使用时的创建的

`tocic`和`channel`都相互独立的缓冲数据,防止缓慢的消费者导致其他 channel 积压(同样适用于`tocic`级别)

`channel`可以并且通常会连接多个客户端,假如所有连接的客户端都处于准备接收消息的状态,则每条消息将被传递到随机客户端

[![](http://aishad.top/wordpress/wp-content/uploads/2020/08/nsq5.gif)](http://aishad.top/wordpress/wp-content/uploads/2020/08/nsq5.gif)

总而言之，消息是从topic -> channel（每个channel接收该topic的所有消息的副本）多播的，但是从channel -> consumers均匀分布（每个消费者接收该channel的一部分消息）。

### NSQ 的特性


1. 消息默认不会持久化,可以配置成持久化模式,nsq 采用的方式是内存+硬盘的持久化方式,当内存达到一定的程度就会将数据持久化到硬盘

- 如果将--mem-queue-size设置为0，所有的消息将会存储到磁盘。
- 服务器重启时也会将当时在内存中的消息持久化。

2. 每条笑死至少被传递一次
3. 消息不保证有序

## NSQ的部署(单机)

[官方下载页面](https://nsq.io/deployment/installing.html)根据自己的平台下载并解压即可。

1. 启动 nsqlookupd
```bash
$ nsqlookupd 
[nsqlookupd] 2020/08/06 10:43:29.772044 INFO: nsqlookupd v1.2.0 (built w/go1.13.5)
[nsqlookupd] 2020/08/06 10:43:29.775350 INFO: TCP: listening on [::]:4160
[nsqlookupd] 2020/08/06 10:43:29.775352 INFO: HTTP: listening on [::]:4161
```

2. 启动 nsqd 守护进程
```bash
$ nsqd -broadcast-address=127.0.0.1 -lookupd-tcp-address=127.0.0.1:4160
[nsqd] 2020/08/06 10:44:57.698832 INFO: nsqd v1.2.0 (built w/go1.13.5)
[nsqd] 2020/08/06 10:44:57.700478 INFO: ID: 938
[nsqd] 2020/08/06 10:44:57.701271 INFO: NSQ: persisting topic/channel metadata to nsqd.dat
[nsqd] 2020/08/06 10:44:57.719304 INFO: TCP: listening on [::]:4150
[nsqd] 2020/08/06 10:44:57.719308 INFO: HTTP: listening on [::]:4151
[nsqd] 2020/08/06 10:44:57.719735 INFO: LOOKUP(127.0.0.1:4160): adding peer
[nsqd] 2020/08/06 10:44:57.719763 INFO: LOOKUP connecting to 127.0.0.1:4160
[nsqd] 2020/08/06 10:44:57.733131 INFO: LOOKUPD(127.0.0.1:4160): peer info {TCPPort:4160 HTTPPort:4161 Version:1.2.0 BroadcastAddress:weiyingdeMacBook-Air.local}


# 如果使用的不是 nsqlookupd 模式的话.不需要指定选项 -lookupd-tcp-address=127.0.0.1:4160
```

> 如果是部署了多个`nsqlookupd`节点的集群，那还可以指定多个`-lookupd-tcp-address`
>
3. 启动 nsqadmin
```bash
$ nsqadmin -lookupd-http-address=127.0.0.1:4161
[nsqadmin] 2020/08/06 10:55:12.482467 INFO: nsqadmin v1.2.0 (built w/go1.13.5)
[nsqadmin] 2020/08/06 10:55:12.486876 INFO: HTTP: listening on [::]:4171
```

#### nsqlookupd 的使用
```bash
$ nsqlookupd -h
Usage of nsqlookupd:
  -broadcast-address string
    	address of this lookupd node, (default to the OS hostname) (default "weiyingdeMacBook-Air.local")
  -config string
    	path to config file
  -http-address string
    	<addr>:<port> to listen on for HTTP clients (default "0.0.0.0:4161")
  -inactive-producer-timeout duration
    	duration of time a producer will remain in the active list since its last ping (default 5m0s)
  -log-level value
    	set log verbosity: debug, info, warn, error, or fatal (default INFO)
  -log-prefix string
    	log message prefix (default "[nsqlookupd] ")
  -tcp-address string
    	<addr>:<port> to listen on for TCP clients (default "0.0.0.0:4160")
  -tombstone-lifetime duration
    	duration of time a producer will remain tombstoned if registration remains (default 45s)
  -verbose
    	[deprecated] has no effect, use --log-level
  -version
    	print version string
```

#### nsqd 的使用
```bash
$ nsqd -h
Usage of nsqd:
  -auth-http-address value
    	<addr>:<port> to query auth server (may be given multiple times)
  -broadcast-address string
    	address that will be registered with lookupd (defaults to the OS hostname) (default "weiyingdeMacBook-Air.local")
  -config string
    	path to config file
  -data-path string
    	path to store disk-backed messages
  -deflate
    	enable deflate feature negotiation (client compression) (default true)
  -e2e-processing-latency-percentile value
    	message processing time percentiles (as float (0, 1.0]) to track (can be specified multiple times or comma separated '1.0,0.99,0.95', default none)
  -e2e-processing-latency-window-time duration
    	calculate end to end latency quantiles for this duration of time (ie: 60s would only show quantile calculations from the past 60 seconds) (default 10m0s)
  -http-address string
    	<addr>:<port> to listen on for HTTP clients (default "0.0.0.0:4151")
  -http-client-connect-timeout duration
    	timeout for HTTP connect (default 2s)
  -http-client-request-timeout duration
    	timeout for HTTP request (default 5s)
  -https-address string
    	<addr>:<port> to listen on for HTTPS clients (default "0.0.0.0:4152")
  -log-level value
    	set log verbosity: debug, info, warn, error, or fatal (default INFO)
  -log-prefix string
    	log message prefix (default "[nsqd] ")
  -lookupd-tcp-address value
    	lookupd TCP address (may be given multiple times)
  -max-body-size int
    	maximum size of a single command body (default 5242880)
  -max-bytes-per-file int
    	number of bytes per diskqueue file before rolling (default 104857600)
  -max-channel-consumers int
    	maximum channel consumer connection count per nsqd instance (default 0, i.e., unlimited)
  -max-deflate-level int
    	max deflate compression level a client can negotiate (> values == > nsqd CPU usage) (default 6)
  -max-heartbeat-interval duration
    	maximum client configurable duration of time between client heartbeats (default 1m0s)
  -max-msg-size int
    	maximum size of a single message in bytes (default 1048576)
  -max-msg-timeout duration
    	maximum duration before a message will timeout (default 15m0s)
  -max-output-buffer-size int
    	maximum client configurable size (in bytes) for a client output buffer (default 65536)
  -max-output-buffer-timeout duration
    	maximum client configurable duration of time between flushing to a client (default 30s)
  -max-rdy-count int
    	maximum RDY count for a client (default 2500)
  -max-req-timeout duration
    	maximum requeuing timeout for a message (default 1h0m0s)
  -mem-queue-size int
    	number of messages to keep in memory (per topic/channel) (default 10000)
  -min-output-buffer-timeout duration
    	minimum client configurable duration of time between flushing to a client (default 25ms)
  -msg-timeout duration
    	default duration to wait before auto-requeing a message (default 1m0s)
  -node-id int
    	unique part for message IDs, (int) in range [0,1024) (default is hash of hostname) (default 938)
  -output-buffer-timeout duration
    	default duration of time between flushing data to clients (default 250ms)
  -snappy
    	enable snappy feature negotiation (client compression) (default true)
  -statsd-address string
    	UDP <addr>:<port> of a statsd daemon for pushing stats
  -statsd-interval duration
    	duration between pushing to statsd (default 1m0s)
  -statsd-mem-stats
    	toggle sending memory and GC stats to statsd (default true)
  -statsd-prefix string
    	prefix used for keys sent to statsd (%s for host replacement) (default "nsq.%s")
  -statsd-udp-packet-size int
    	the size in bytes of statsd UDP packets (default 508)
  -sync-every int
    	number of messages per diskqueue fsync (default 2500)
  -sync-timeout duration
    	duration of time per diskqueue fsync (default 2s)
  -tcp-address string
    	<addr>:<port> to listen on for TCP clients (default "0.0.0.0:4150")
  -tls-cert string
    	path to certificate file
  -tls-client-auth-policy string
    	client certificate auth policy ('require' or 'require-verify')
  -tls-key string
    	path to key file
  -tls-min-version value
    	minimum SSL/TLS version acceptable ('ssl3.0', 'tls1.0', 'tls1.1', or 'tls1.2') (default 769)
  -tls-required
    	require TLS for client connections (true, false, tcp-https)
  -tls-root-ca-file string
    	path to certificate authority file
  -verbose
    	[deprecated] has no effect, use --log-level
  -version
    	print version string
  -worker-id
    	[deprecated] use --node-id
```

#### nsqadmin 的使用
```go
$ nsqadmin -h
Usage of nsqadmin:
  -acl-http-header string
    	HTTP header to check for authenticated admin users (default "X-Forwarded-User")
  -admin-user value
    	admin user (may be given multiple times; if specified, only these users will be able to perform privileged actions; acl-http-header is used to determine the authenticated user)
  -allow-config-from-cidr string
    	A CIDR from which to allow HTTP requests to the /config endpoint (default "127.0.0.1/8")
  -base-path string
    	URL base path (default "/")
  -config string
    	path to config file
  -graphite-url string
    	graphite HTTP address
  -http-address string
    	<addr>:<port> to listen on for HTTP clients (default "0.0.0.0:4171")
  -http-client-connect-timeout duration
    	timeout for HTTP connect (default 2s)
  -http-client-request-timeout duration
    	timeout for HTTP request (default 5s)
  -http-client-tls-cert string
    	path to certificate file for the HTTP client
  -http-client-tls-insecure-skip-verify
    	configure the HTTP client to skip verification of TLS certificates
  -http-client-tls-key string
    	path to key file for the HTTP client
  -http-client-tls-root-ca-file string
    	path to CA file for the HTTP client
  -log-level value
    	set log verbosity: debug, info, warn, error, or fatal (default INFO)
  -log-prefix string
    	log message prefix (default "[nsqadmin] ")
  -lookupd-http-address value
    	lookupd HTTP address (may be given multiple times)
  -notification-http-endpoint string
    	HTTP endpoint (fully qualified) to which POST notifications of admin actions will be sent
  -nsqd-http-address value
    	nsqd HTTP address (may be given multiple times)
  -proxy-graphite
    	proxy HTTP requests to graphite
  -statsd-counter-format string
    	The counter stats key formatting applied by the implementation of statsd. If no formatting is desired, set this to an empty string. (default "stats.counters.%s.count")
  -statsd-gauge-format string
    	The gauge stats key formatting applied by the implementation of statsd. If no formatting is desired, set this to an empty string. (default "stats.gauges.%s")
  -statsd-interval duration
    	time interval nsqd is configured to push to statsd (must match nsqd) (default 1m0s)
  -statsd-prefix string
    	prefix used for keys sent to statsd (%s for host replacement, must match nsqd) (default "nsq.%s")
  -verbose
    	[deprecated] has no effect, use --log-level
  -version
    	print version string
```