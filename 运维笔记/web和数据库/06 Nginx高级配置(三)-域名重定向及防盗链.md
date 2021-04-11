# Nginx高级配置(三)
	Nginx服务器利用ngx_http_rewrite_module 模块解析和处理rewrite请求，此功能依靠 PCRE(perl compatible regularexpression)，因此编译之前要安装PCRE库，rewrite是nginx服务器的重要功能之一，用于实现URL的重写，URL的重写是非常有用的功能，比如它可以在我们改变网站结构之后，不需要客户端修改原来的书签，也无需其他网站修改我们的链接，就可以设置为访问，另外还可以在一定程度上提高网站的安全性。

1. if指令
	用于条件匹配判断，并根据条件判断结果选择不同的Nginx配置，可以配置在server或location块中进行配置，Nginx的if语法仅能使用if做单次判断，不支持使用if else或者if elif这样的多重判断，用法如下：
	if （条件匹配） {
	action
	}

```bash
	location /main {
	index index.html;
	default_type text/html;
	if ( $scheme = http ){
		echo "if-----> $scheme";
	}
	if ( $scheme = https ){
		echo "if ----> $scheme";
	}
	}

```

- 使用正则表达式对变量进行匹配，匹配成功时if指令认为条件为true，否则认为false，变量与表达式之间使用以下符号链接：
	=： 比较变量和字符串是否相等，相等时if指令认为该条件为true，反之为false。
	!=: 比较变量和字符串是否不相等，不相等时if指令认为条件为true，反之为false。
	~： 表示在匹配过程中区分大小写字符，（可以通过正则表达式匹配），满足匹配条件为真，不满足为假。
	~*: 表示在匹配过程中不区分大小写字符，（可以通过正则表达式匹配），满足匹配条件为真，不满足问假。
	!~：区分大小写不匹配，不满足为真，满足为假，不满足为真。
	!~*:为不区分大小写不匹配，满足为假，不满足为真。
	-f 和 ! -f:判断请求的文件是否存在和是否不存在
	-d 和 ! -d: 判断请求的目录是否存在和是否不存在。
	-x 和 ! -x: 判断文件是否可执行和是否不可执行。
	-e 和 ! -e: 判断请求的文件或目录是否存在和是否不存在(包括文件，目录，软链接)

> 如果$变量的值为空字符串或是以0开头的任意字符串，则if指令认为该条件为false，其他条件为true。

2. set指令
	指定key并给其定义一个变量，变量可以调用Nginx内置变量赋值给key，另外set定义格式为set $key $value，及无论是key还是value都要加$符号。
```bash
	location /main {
	root /data/nginx/html/pc;
	index index.html;
	default_type text/html;
	set $name magedu;
	echo $name;
	set $my_port $server_port;
	echo $my_port;
}

```

3. break指令
	用于中断当前相同作用域(location)中的其他Nginx配置，与该指令处于同一作用域的Nginx配置中，位于它前面的配置生效，位于后面的指令配置就不再生效了，Nginx服务器在根据配置处理请求的过程中遇到该指令的时候，回到上一层作用域继续向下读取配置，该指令可以在server块和location块以及if块中使用，使用语法如下：
```bash
	location /main {
	root /data/nginx/html/pc;
	index index.html;
	default_type text/html;
	set $name magedu;
	echo $name;
	break;
	set $my_port $server_port;
	echo $my_port;
	}
```

4. return指令
	从nginx版本0.8.2开始支持，return用于完成对请求的处理，并直接向客户端返回响应状态码，比如其可以指定重定向URL(对于特殊重定向状态码，301/302等) 或者是指定提示文本内容(对于特殊状态码403/500等)，处于此指令后的所有配置都将不被执行，return可以在server、if和location块进行配置，用法如下：
```bash
	return code; #返回给客户端指定的HTTP状态码
	return code (text); #返回给客户端的状态码及响应体内容，可以调用变量
	return code URL； #返回给客户端的URL地址
	例如：
		location /main {
			root /data/nginx/html/pc;
			default_type text/html;
			index index.html;
			if ( $scheme = http ){
				#return 666;
				#return 666 "not allow http";
				#return 301 http://www.baidu.com;
				return 500 "service error";
				echo "if-----> $scheme"; #return后面的将不再执行
			}
			if ( $scheme = https ){
			echo "if ----> $scheme";
			}
		}

```

5. rewrite_log指令
	设置是否开启记录ngx_http_rewrite_module模块日志记录到error_log日志文件当中，可以配置在http、server、location或if当中，需要日志级别为notice 。
```bash
	location /main {
	index index.html;
	default_type text/html;
	set $name magedu;
	echo $name;
	rewrite_log on;
	break;
	set $my_port $server_port;
	echo $my_port;
	}

```

6. rewrite指令
	通过正则表达式的匹配来改变URI，可以同时存在一个或多个指令，按照顺序依次对URI进行匹配，rewrite主要是针对用户请求的URL或者是URI做具体处理，以下是URL和URI的具体介绍：

- URI(universal resource identifier)：通用资源标识符，标识一个资源的路径，可以不带协议。

- URL(uniform resource location):统一资源定位符，是用于在Internet中描述资源的字符串，是URI的子集，主要包括传输协议(scheme)、主机(IP、端口号或者域名)和资源具体地址(目录和文件名)等三部分，一般格式为 scheme://主机名[:端口号][/资源路径],如：http://www.a.com:8080/path/file/index.html 就是一个URL路径，URL必须带访问协议。

- 每个URL都是一个URI，但是URI不都是URL
	例如：
	http://example.org/path/to/resource.txt #URI/URL
	ftp://example.org/resource.txt #URI/URL
	/absolute/path/to/resource.txt #URI

[![](images/uri.png)](http://aishad.top/wordpress/wp-content/uploads/2019/06/uri.png)

>  rewrite可以配置在server、location、if，其具体使用方式为：
>  rewrite regex replacement [flag];

	rewrite将用户请求的URI基于regex所描述的模式进行检查，匹配到时将其替换为表达式指定的新的URI。 注意：如果在同一级配置块中存在多个rewrite规则，那么会自下而下逐个检查；被某条件规则替换完成后，会重新一轮的替换检查，隐含有循环机制,但不超过10次；如果超过，提示500响应码，[flag]所表示的标志位用于控制此循环机制，如果替换后的URL是以http://或https://开头，则替换结果会直接以重向返回给客户端, 即永久重定向301

### rewrite flag使用
	利用nginx的rewrite的指令，可以实现url的重新跳转，rewrtie有四种不同的flag，分别是redirect(临时重定向)、permanent(永久重定向)、break和last。其中前两种是跳转型的flag，后两种是代理型，跳转型是指有客户端浏览器重新对新地址进行请求，代理型是在WEB服务器内部实现跳转的。

> Syntax: rewrite regex replacement [flag]; #通过正则表达式处理用户请求并返回替换后的数据包。 Default: —Context: server, location, if

- redirect
	临时重定向，重写完成后以临时重定向方式直接返回重写后生成的新URL给客户端，由客户端重新发起请求；使用相对路径,或者http://或https://开头，状态码：302

- permanent
	重写完成后以永久重定向方式直接返回重写后生成的新URL给客户端，由客户端重新发起请求，状态码：301

- last
	重写完成后停止对当前URI在当前location中后续的其它重写操作，而后对新的URL启动新一轮重写检查，不建议在location中使用

- break
	重写完成后停止对当前URL在当前location中后续的其它重写操作，而后直接跳转至重写规则配置块之后的其它配置；结束循环，建议在location中使用

1. ：rewrite案例--域名永久与临时重定向
	临时重定向不会缓存域名解析记录(A记录)，但是永久重定向会缓存。
```bash
	location / {
		root /data/nginx/html/pc;
		index index.html;
		rewrite / http://www.aishad.top permanent;
		rewrite / http://www.aishad.top redirect;
		}

```

	域名临时重定向，告诉浏览器域名不是固定重定向到当前目标域名，后期可能随时会更改，因此浏览器不会缓存当前域名的解析记录，而浏览器会缓存永久重定向的DNS解析记录，这也是临时重定向与永久重定向最大的本质区别。

2. rewrite中的last和break

```bash
	location /break {
		rewrite ^/break/(.*) /test$1 break; #break不会跳转到其他的location
		return 666 "break";
	}
	location /last {
		rewrite ^/last/(.*) /test$1 last; #last会跳转到其他的location继续匹配新的URI
		return 888 "last";
	}
	location /test {
		return 999 "test";
	}

测试：
	curl -L -i http://www.magedu.net/break/ #brak不会跳转至location /test中
	 curl -L -i http://www.magedu.net/last/ #last会跳转到location /test继续执行匹配操作

```

3. rewrite案例-自动跳转https

```bash
	server {
		listen 443 ssl;
		listen 80;
		ssl_certificate /apps/nginx/certs/www.magedu.net.crt;
		ssl_certificate_key /apps/nginx/certs/www.magedu.net.key;
		ssl_session_cache shared:sslcache:20m;
		ssl_session_timeout 10m;
		server_name www.magedu.net;
		location / {
			root /data/nginx/html/pc;
			index index.html;
			if ($scheme = http ){ #未加条件判断，会导致死循环
				rewrite / https://www.magedu.net permanent;
			}
		}
	}

测试：
	curl -L -k -i https://www.magedu.net/
```

4. rewrite案例-判断文件是否存在
	要求：当用户访问到公司网站的时输入了一个错误的URL，可以将用户重定向至官网首页

```bash
	location / {
		root /data/nginx/html/pc;
		index index.html;
		if (!-f $request_filename) {
			#return 404 "linux35";
			rewrite (.*) http://www.magedu.net/index.html;
		}
	}
#重启Nginx并访问测试
```

## Nginx防盗链
	防盗链基于客户端携带的referer实现，referer是记录打开一个页面之前记录是从哪个页面跳转过来的标记信息，如果别人只链接了自己网站图片或某个单独的资源，而不是打开了网站的整个页面，这就是盗链，referer就是之前的那个网站域名，正常的referer信息有以下几种：

- none：请求报文首部没有referer首部，比如用户直接在浏览器输入域名访问web网站，就没有referer信息。

- blocked：请求报文有referer首部，但无有效值，比如为空。

- server_names：referer首部中包含本主机名及即nginx 监听的server_name。

- arbitrary_string：自定义指定字符串，但可使用*作通配符。

- regular expression：被指定的正则表达式模式匹配到的字符串,要使用~开头，例如：~.*\.magedu\.com。


- 正常访问的referer信息：
	"referer":"https://www.baidu.com/s?ie=utf8&f=8&rsv_bp=1&rsv_idx=1&tn=baidu&wd=www.magedu.net&oq=www.mageedu.net&rsv_pq=d63060680002eb69&rsv_t=de01TWnmyTdcJqph7SfI1hXgXLJxSSfUPcQ3QkWdJk%2FLNrN95ih3XOhbRs4&rqlang=cn&rsv_enter=1&inputT=321&rsv_sug3=41&rsv_sug2=0&rsv_sug4=1626","tcp_xff":"","http_user_agent":"Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/72.0.3626.119 Safari/537.36","status":"304"}

- 被盗链的referer信息
	referer":"http://www.mageedu.net/","tcp_xff":"","http_user_agent":"Mozilla/5.0(Windows NT 6.1; Win64; x64; rv:65.0) Gecko/20100101 Firefox/65.0","status":"304"}

### 防盗链的实现
	在一个web 站点盗链另一个站点的资源信息，比如图片、视频等。
	基于访问安全考虑，nginx支持通过ungx_http_referer_module模块，检查访问请求的referer信息是否有效实现防盗链功能，定义方式如下：

```bash
	location ^~ /images {
	root /data/nginx;
	index index.html;
	valid_referers none blocked server_names *.aishad.top www.aishad.* api.online.test/v1/hostlist ~\.google\. ~\.baidu\.; #定义有效的referer
	if ($invalid_referer) { #假如是使用其他的无效的referer访问：
	return 403; #返回状态码403
	}
	}
#重启Nginx并访问测试
```