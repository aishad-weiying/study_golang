# Nginx第四篇：Nginx高级配置(一)

## Nginx状态页
	基于nginx模块ngx_http_auth_basic_module实现，在编译安装nginx的时候需要添加编译参数--with-http_stub_status_module，否则配置完成之后监测会是提示语法错误。

- 配置实例：
	location /nginx_status {
	stub_status;
	allow 192.168.0.0/16;
	allow 127.0.0.1;
	deny all;
	}

- 状态页输出信息：

1. Active connections: 291：当前处于活动状态的客户端连接数，包括连接等待空闲的连接数
	
2. accepts：Nginx自从启动后已经接受客户端请求的总数，reload后会重新计算

3. handled：Nginx自从启动后已经处理完成的客户端请求总数，通常等于accepts，除非有应为worker_connections限制等被拒绝的连接
	
4. requests：Nginx自从启动后客户端发来的总的请求总数

5. Reading：当前状态，正在读取客户端请求报文首部连接的连接数

6. writing：正在向客户端发送响应报文过程中的连接数

7. waiting：正在等待客户端发出连接请求的空闲连接数，开启keep-ailve的情况下，这个值登录active-(reading+writing)

## 第三方模块
	第三模块是对nginx 的功能扩展，第三方模块需要在编译安装Nginx 的时候使用参数--add-module=PATH指定路径添加，有的模块是由公司的开发人员针对业务需求定制开发的，有的模块是开源爱好者开发好之后上传到github进行开源的模块，nginx支持第三方模块需要从源码重新编译支持，比如开源的echo模块 https://github.com/openresty/echo-nginx-module

### echo模块的使用

1. 编译安装echo模块

- yum install git -y

- git clone https://github.com/openresty/echo-nginx-module.git

- nginx -s stop

- cd ~/nginx-1.14.2/

- --prefix=/apps/nginx \
	--user=nginx --group=nginx \
	--with-http_ssl_module \
	--with-http_v2_module \
	--with-http_realip_module \
	--with-http_stub_status_module \
	--with-http_gzip_static_module \
	--with-pcre \
	--with-stream \
	--with-stream_ssl_module \
	--with-stream_realip_module \
	--with-http_perl_module \
	--add-module=/root/echo-nginx-module
	
- make && make install

2. echo模块的使用
```bash
	vim /apps/nginx/conf/vhosts/mobile.conf
		location /main {
			index index.html;
			default_type text/html;
			echo "hello world,main-->";
			echo_reset_timer;
			echo_location /sub1;
			echo_location /sub2;
			echo "took $echo_timer_elapsed sec for total.";
		}
		location /sub1 {
			echo_sleep 1;
			echo sub1;
		}
		location /sub2 {
			echo_sleep 1;
			echo sub2;
		}
	测试：
		curl http://www.magedu.net/main
			hello world,main-->
			sub1
			sub2
			took 2.010 sec for total
```
## nginx的内置变量
	nginx的变量可以在配置文件中引用，作为功能判断或者日志等场景使用，变量可以分为内置变量和自定义变量，内置变量是由nginx模块自带，通过变量可以获取到众多的与客户端访问相关的值。

1. $remote_addr;存放了客户端的地址，注意是客户端的公网IP，也就是一家人访问一个网站，则会显示为路由器的公网IP。

2. $args；
	变量中存放了URL中的指令，例如http://www.magedu.net/main/index.do?id=20190221&partner=search 中的id=20190221&partner=search

3. $document_root；
	保存了针对当前资源的请求的系统根目录，如/apps/nginx/html。

4. $document_uri;
	保存了当前请求中不包含指令的URI，注意是不包含请求的指令
	例如：http://www.magedu.net/main/index.do?id=20190221&partner=search会被定义为/main/index.do

5. $host;
	存放了请求的host名称

6. $http_user_agent;
	客户端浏览器的详细信息

7. $http_cookie；
	客户端cookie信息

8. $limit_rate 10240;
	echo $limit_rate;如果nginx服务器使用limit_rate配置了显示网络速率，则会显示，如果没有设置， 则显示0。

9. $remote_port;
	客户端请求nginx服务器时随机打开的端口，这是每个客户端自己的端口

10. remote_user；
	已经经过auth basic module验证的用户名

11. request_body_file；
	做反向代理时发送给后端服务器的本地资源名称

12. $request_method;
	请求资源的方式 GET/PUT/DELETE等

13. $request_filename;
	当前请求的资源文件的路径名称，由root或alias指令与URI请求生成的文件绝对路径
	如/apps/nginx/html/main/index.html

14. $request_uri;
	包含请求参数的原始URI，不包含主机名，，如：/main/index.do?id=20190221&partner=search 。

15. $scheme；
	请求的协议，如ftp，http，https等

16. server_protocol;
	保存了客户端请求资源使用的协议版本，如HTTP/1.0，HTTP/1.1，HTTP/2.0等

17. $server_addr:
	保存了服务器端的IP地址

18. $server_name；
	请求的服务器主机名

19. $server_port;
	请求的服务器端口号

### 自定义变量
	使用指令 set $variable value;
```bash
	set $name magedu;
	echo $name;
	set $my_port $server_port;
	echo $my_port;
	echo "$server_name:$server_port";
```