Nginx第三篇：Nginx基础配置(二)

## location的详细配置使用
	在没有使用正则表达式的时候，nginx会现在server中的多个location选取匹配度最高的一个uri，uri是用户请求的字符串，即域名侯爱民的web文件路径，然后使用该location模块中的正则url和字符串，如果匹配成功就结束搜索，并使用此location处理请求

| 语法规则： |                                                              |
| ---------- | ------------------------------------------------------------ |
| =          | 用于标准uri前，表示包含正则表达式并且区分大小写              |
| ~          | 用于标准uri前，表示包含正则表达式并且区分大小写              |
| ~*         | 用于标准uri前，表示包含正则表达式并且不区分大写              |
| !~         | 用于标准uri前，表示包含正则表达式并且区分大小写不匹配        |
| !~*        | 用于标准uri前，表示包含正则表达式并且不区分大小写不匹配      |
| ^~         | 用于标准uri前，表示包含正则表达式并且匹配以什么结尾          |
| \          | 用于标准uri前，表示包含正则表达式并且转义字符。可以转. * ?等 |
| $          | 用于标准uri前，表示包含正则表达式并且匹配以什么结尾          |
| *          | 用于标准uri前，表示包含正则表达式并且代表任意长度的任意字符  |



1. 精确匹配：= ：用于标准uri前，需要请求字串与uri精确匹配，如果匹配成功就停止向下匹配并立即处理请求
	访问www.weiyign.com/1.jpg的时候，会去查找location指定的/data目录下面的1.jpg文件
```bash
	server{
        listen 80;
        server_name www.weiying.com;
        location / {
                root /data/html/pc;
                index index.html;
        }
        location = /1.jpg {
                root /data;
        }
	}
```

2. 区分大小写匹配：~ ：用于标准uri前，表示包含正则表达式并且区分大小写
	访问www.weiyign.com/Aa.jpg的时候，会去查找location指定的/data目录下面的Aa.jpg文件，严格区分大小写
```bash
	server{
        listen 80;
        server_name www.weiying.com;
        location / {
                root /data/html/pc;
                index index.html;
        }
        location ~ /A.*\.jpg {
                root /data;
                index index.html;
        }
	}
```

3.  区分大小写匹配：~* ：用于标准uri前，表示包含正则表达式并且不区分大写
	对于不区分大小写的location，则可以访问任意大小写结尾的图片文件,如区分大小写则只能访问aa.jpg，不区分大小写则可以访问aa.jpg以外的资源比如Aa.JPG、aA.jPG这样的混合名称文件

```bash
	server{
        listen 80;
        server_name www.weiying.com;
        location / {
                root /data/html/pc;
                index index.html;
        }
        location ~* /A.*\.jpg {
                root /data;
                index index.html;
        }
	}
```

4. 匹配URI开始：^~ ：用于标准uri前，表示包含正则表达式并且匹配以什么开头
```bash
	location ^~ /images {
		root /data/nginx;
		index index.html;
	}
	location /images1 {
		alias /data/nginx/html/pc;
		index index.html;
	}
	重启Nginx并访问测试，实现效果是访问images和images1返回不同的结果
	[root@s2 images]# curl http://www.magedu.net/images/
	images
	[root@s2 images]# curl http://www.magedu.net/images1/
	pc web
```

5. 匹配文件名后缀
```bash
	[root@s2 ~]# mkdir /data/nginx/images1
	#上传一个和images目录不一样内容的的图片1.j到/data/nginx/images1
		location ~* \.(gif|jpg|jpeg|bmp|png|tiff|tif|ico|wmf|js)$ {
			root /data/nginx/images1;
			index index.html;
		}
	重启Nginx并访问测试
```
6. 优先级匹配
```bash
	location /1.jpg {
		root /var/www/nginx/images;
		index index.html;
	}
	[root@s2 ~]# mkdir /var/www/nginx/images -p
	上传图片到/var/www/nginx/images并重启nginx访问测试
	匹配优先级：=, ^~, ～/～*，/
	location优先级：(location =) > (location 完整路径) > (location ^~ 路径) > (location ~,~* 正则顺序) > (location 部分起始路径) > (/)
```

### nginx的访问控制
	访问控制基于模块ngx_http_access_module实现，可以通过匹配客户端源IP地址进行限制。

```bash
	location = /login/ {
		root /data/nginx/html/pc;
	}
	location /about {
		alias /data/nginx/html/pc;
		index index.html;
		deny 192.168.1.1;
		allow 192.168.1.0/24;
		allow 10.1.1.0/16;
		allow 2001:0db8::/32;
		deny all; #先允许小部分，再拒绝大部分
	}
```
## Nginx账户认证功能
```bash
	[root@s2 ~]# yum install httpd-tools -y
	[root@s2 ~]# htpasswd -cbm /apps/nginx/conf/.htpasswd user1 123456
	Adding password for user user1
	[root@s2 ~]# htpasswd -bm /apps/nginx/conf/.htpasswd user2 123456
	Adding password for user user2
	[root@s2 ~]# tail /apps/nginx/conf/.htpasswd
		user1:$apr1$Rjm0u2Kr$VHvkAIc5OYg.3ZoaGwaGq/
		user2:$apr1$nIqnxoJB$LR9W1DTJT.viDJhXa6wHv.
	[root@s2 ~]# vim /apps/nginx/conf/conf.d/pc.conf
		location = /login/ {
			root /data/nginx/html/pc;
			index index.html;
			auth_basic "login password";
			auth_basic_user_file /apps/nginx/conf/.htpasswd;
		}
	重启Nginx并访问测试
```
### 自定义错误页面
	error_page 错误代码 错误页面;
	location = /错误页面 { root 错误页面位置； }
```bash
	error_page 500 502 503 504 404 /error.html;
	location = /error.html{
		root /data/error;
	}
```
### 自定义访问日志
	access_log 访问日志文件路径；
	error_log 错误日志文件路径；
```bash
	mkdir /data/logs
	vim /apps/nginx/conf/vhosts/pc.conf
		server{
		listen 80;
		server_name www.weiying.com;
		
		access_log /data/logs/pc_access.log;
		error_log /data/logs/pc_error.log;
		location / {
			root /data/html/pc;
			index index.html;
		}
```

## 检测文件是否存在
	try_files会按顺序检查文件是否存在，返回第一个找到的文件或文件夹（结尾加斜线表示为文件夹），如果所有文件或文件夹都找不到，会进行一个内部重定向到最后一个参数。只有最后一个参数可以引起一个
	
	try_files 要匹配的文件 $uri.html =自定义错误码;
```bash
	location /about {
		root /data/nginx/html/pc;
		index index.html;
		#try_files $uri /about/default.html;
		#try_files $uri $uri/index.html $uri.html /about/default.html;
		try_files $uri $uri/index.html $uri.html =489;
	}
	[root@s2 ~]# echo "default" >> /data/nginx/html/pc/about/default.html
	重启nginx并测试，当访问到http://www.magedu.net/about/xx.html等不存在的uri会显示default，
	如果是自定义的状态码则会显示在返回数据的状态码中，如：
	[root@s2 about]# curl --head http://www.magedu.net/about/xx.html
		HTTP/1.1 489 #489就是自定义的状态返回码
		Server: nginx
		Date: Thu, 21 Feb 2019 00:11:40 GMT
		Content-Length: 0
		Connection: keep-alive
		Keep-Alive: timeout=65

```
## nginx长连接设置

- keepalive_timeout 65 60; 
	设定保持连接超时时长，0表示禁止长连接，默认为65s，通常配置在http字段作为站点全局配置 

- keepalive_requests 100; 
	在一次长连接上所允许请求的资源的最大数量，默认为100次

	开启长连接后，返回客户端的会话保持时间为60s，单次长连接累计请求达到指定次数请求或65秒就会被断开，后面的60为发送给客户端应答报文头部中显示的超时时间设置为60s：如不设置客户端将不显示超时时间。

- Keep-Alive:timeout=60 ：浏览器收到的服务器返回的报文

- 如果设置为0表示关闭会话保持功能，将如下显示：
	Connection:close #浏览器收到的服务器返回的报文

## 当nginx作为下载服务器时的设置

- autoindex on; 自动索引功能

- autoindex_exact_size on; 
	计算文件确切大小（单位bytes），off只显示大概大小（单位kb、mb、gb）

- autoindex_localtime on;  显示本机时间而非GMT(格林威治)时间

- limit_rate rate; 限制响应给客户端的传输速率，单位是bytes/second，默认值0表示无限制

## 当nginx作为上传服务器时的设置

- client_max_body_size 1m； 设置允许客户端上传单个文件的最大值，默认值为1m

- client_body_buffer_size size; 
	用于接收每个客户端请求报文的body部分的缓冲区大小；默认16k；超出此大小时，其将被暂存到磁盘上的由下面client_body_temp_path指令所定义的位置

- client_body_temp_path path [level1 [level2 [level3]]];
	设定存储客户端请求报文的body部分的临时存储路径及子目录结构和数量，目录名为16进制的数字，使用hash之后的值从后往前截取1位、2位、2位作为文件名：

## 其他设置

- keepalive_disable none | browser ...;对哪种浏览器禁用长连接

- limit_except method ... { ... }；仅用于location，限制客户端使用除了指定的请求方法之外的其它方法
	method:GET, HEAD, POST, PUT, DELETE，MKCOL, COPY, MOVE, OPTIONS, PROPFIND,PROPPATCH, LOCK, UNLOCK, PATCH
	limit_except GET {
		allow 192.168.0.0/24;
		allow 192.168.7.101;
		deny all;
	}

```bash
	例如：仅允许指定的客户端使用GET之外的方法
		location /upload {
			root /data/magedu/pc;
			index index.html;
			limit_except GET {
			allow 172.22.45.131;
			deny all;
			}
		}
	测试：
	131客户端测试：curl -XPUT /etc/issue http://www.magedu.net/about
		返回405，nginx已运行上传，但是程序为支持上传功能
	其他客户端测试
		返回403，nginx拒绝上传
```

- aio on|of；#是否启用asynchronous file I/O(AIO)(异步传输文件)功能，需要编译开启
	linux 2.6以上内核提供以下几个系统调用来支持aio：主要用户在磁盘读文件
	步骤如下：
1. SYS_io_setup：建立aio 的context
2. SYS_io_submit: 提交I/O操作请求
3. SYS_io_getevents：获取已完成的I/O事件
4. SYS_io_cancel：取消I/O操作请求
5. SYS_io_destroy：毁销aio的context

- directio size | off; #操作完全和aio相反，aio是读取文件而directio是写文件到磁盘，启用直接I/O，默认为关闭，当文件大于等于给定大小时，例如directio 4m，同步（直接）写磁盘，而非写缓存。

### 缓存相关的配置
- open_file_cache off; #是否缓存打开过的文件信息
	open_file_cache max=N [inactive=time];
	nginx可以缓存以下三种信息：
	(1) 文件元数据：文件的描述符、文件大小和最近一次的修改时间
	(2) 打开的目录结构
	(3) 没有找到的或者没有权限访问的文件的相关信息
	max=N：可缓存的缓存项上限数量；达到上限后会使用LRU(Least recently used，最近最少使用)算法实现管理
	inactive=time：缓存项的非活动时长，在此处指定的时长内未被命中的或命中的次数少于open_file_cache_min_uses指令所指定的次数的缓存项即为非活动项，将被删除

- open_file_cache_errors on | off;
	是否缓存查找时发生错误的文件一类的信息，默认值为off

- open_file_cache_min_uses number;
	open_file_cache指令的inactive参数指定的时长内，至少被命中此处指定的次数方可被归类为活动项默认值为1

- open_file_cache_valid time;
	缓存项有效性的检查验证频率，默认值为60s

- 具体配置
	open_file_cache max=10000 inactive=60s; #最大缓存10000个文件，非活动数据超时时长60s
	open_file_cache_valid 60s; #每间隔60s检查一下缓存数据有效性
	open_file_cache_min_uses 5; #60秒内至少被命中访问5次才被标记为活动数据
	open_file_cache_errors on; #缓存错误信息