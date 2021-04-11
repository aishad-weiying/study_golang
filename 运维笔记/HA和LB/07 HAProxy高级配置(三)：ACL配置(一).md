HAProxy高级配置(三)：ACL
	acl：对接收到的报文进行匹配和过滤，基于请求报文头部中的源地址、源端口、目标地址、目标端口、请求方法、URL、文件后缀等信息内容进行匹配并执行进一步操作。

- 配置：
	acl <aclname\> <criterion\> [flags] [operator] [<value\>]
	acl 名称  条件   条件标记位 具体操作符 操作对象类型 

- 例：
	acl image_service hdr_dom(host) -i img.magedu.com

> ACL名称，可以使用大字母A-Z、小写字母a-z、数字0-9、冒号：、点.、中横线和下划线，并且严格区分大小写，必须Image_site和image_site完全是两个acl。

### Criterion-acl

1. hdr（[<name\> [，<occ\>]]）：完全匹配字符串
	hdr <string\>用于测试请求头部首部指定内容

2. hdr_beg（[<name\> [，<occ\>]]）：前缀匹配
	hdr_beg(host) 请求的host开头，如 www. img. video. download. ftp.

3. hdr_dir（[<name\> [，<occ\>]]）：路径匹配

4. hdr_dom（[<name\> [，<occ\>]]）：域匹配
	hdr_dom(host) 请求的host名称，如 www.aishad.top

5. hdr_end（[<name\> [，<occ\>]]）：后缀匹配
	hdr_end(host) 请求的host结尾，如 .com .net .cn

6. hdr_len（[<name\> [，<occ\>]]）：长度匹配

7. hdr_reg（[<name\> [，<occ\>]]）：正则表达式匹配

8. hdr_sub（[<name\> [，<occ\>]]）：子串匹配

9. path_beg 请求的URL开头，如/static、/images、/img、/css

10. path_end 请求的URL中资源的结尾，如 .gif .png .css .js .jpg .jpeg

#### 匹配条件
	dst 目标IP
	dst_port 目标PORT
	src 源IP
	src_port 源POR

### flags:条件标记
	-i 不区分大小写
	-m 使用指定的pattern匹配方法
	-n 不做DNS解析
	-u 禁止acl重名，否则多个同名ACL匹配或关系

### operator：操作符

- 整数比较：eq、ge、gt、le、lt
	一般用户匹配状态码

- 字符比较：

- exact match (-m str) :字符串必须完全匹配模式

- substring match (-m sub) :在提取的字符串中查找模式，如果其中任何一个被发现，ACL将匹配

- prefix match (-m beg) :在提取的字符串首部中查找模式，如果其中任何一个被发现，ACL将匹配

- suffix match (-m end) :将模式与提取字符串的尾部进行比较，如果其中任何一个匹配，则ACL进行匹配

- subdir match (-m dir) :查看提取出来的用斜线分隔（“/”）的字符串，如果其中任何一个匹配，则ACL进行匹配

- domain match (-m dom) :查找提取的用点（“.”）分隔字符串，如果其中任何一个匹配，则ACL进行匹配

### value

- Boolean #布尔值

- integer or integer range #整数或整数范围，比如用于匹配端口范围,1024~32768

- IP address/network #IP地址或IP范围, 192.168.0.1 ,192.168.0.1/24

- string
	exact –精确比较
	substring—子串 www.aishad.top中的www、aishad、top中的一个匹配成功即可
	suffix-后缀比较
	prefix-前缀比较
	subdir-路径， /wp-includes/js/jquery/jquery.js
	domain-域名，www.aishad.top

- regular expression #正则表达式

### block- hex block #16进制

## Acl定义与调用

- acl作为条件时的逻辑关系：
	与：隐式（默认）使用
	或：使用“or” 或 “||”表示
	否定：使用“!“ 表示

- 示例：
	if valid_src valid_port #与关系
	if invalid_src || invalid_port #或
	if ! invalid_src #非

### Acl 示例-域名匹配
```bash
	listen  web_port
	 bind 172.20.45.132:80
	 bind 192.168.45.132:80
	 mode http
	 log global
	 acl test_host hdr_dom(host) -i www.weiying1.com
	 use_backend test_host if test_host
	 default_backend default_web #以上都没有匹配到的时候使用默认backend

	backend test_host
	 mode http
	 server web1  192.168.45.134:80   check inter 3000 fall 2 rise 5

	backend default_web
	 mode http
	 server web1  192.168.45.133:80   check inter 3000 fall 2 rise 5
```

### Acl-源地址子网匹配
```bash
	listen  web_port
	 bind 172.20.45.132:80
	 bind 192.168.45.132:80
	 mode http
	 log global
	 acl ip_range_test src 172.20.45.1
	 use_backend web2 if ip_range_test
	 default_backend web1

	backend web2
	 mode http
	 server web1  192.168.45.134:80   check inter 3000 fall 2 rise 5

	backend web1
	 mode http
	 server web1  192.168.45.133:80   check inter 3000 fall 2 rise 5
```

### Acl示例-源地址访问控制
```bash
	listen  web_port
	 bind 172.20.45.132:80
	 bind 192.168.45.132:80
	 mode http
	 log global
	 acl block_test src 172.20.45.1
	 block if block_test # 只要是属于匹配到的地址范围就拒绝
	 default_backend web1

	backend web2
	 mode http
	 server web1  192.168.45.134:80   check inter 3000 fall 2 rise 5

	backend web1
	 mode http
	 server web1  192.168.45.133:80   check inter 3000 fall 2 rise 5
```

### Acl示例-匹配浏览器
```bash
	listen  web_port
	 bind 172.20.45.132:80
	 bind 192.168.45.132:80
	 mode http
	 log global
	 acl Chrome_web hdr(User-Agent) -m sub -i "Chrome"
	 #use_backend web2 if Chrome_web #用户请求报文的User-Agent字段带有Chrome的，访问web2定义的服务器
	 redirect prefix http://www.aishad.top if Chrome_web #用户请求报文的User-Agent字段带有Chrome的，重定向到指定页面
	 default_backend web2

	backend web2
	 mode http
	 server web1  192.168.45.134:80   check inter 3000 fall 2 rise 5

	backend web1
	 mode http
	 server web1  192.168.45.133:80   check inter 3000 fall 2 rise 5
```

## 自定义错误页面
	errorfile 500 /usr/local/haproxy/html/500.html #自定义错误页面跳转
	errorfile 502 /usr/local/haproxy/html/502.html
	errorfile 503 /usr/local/haproxy/html/503.html

### 自定义错误跳转
	errorloc 503 http://192.168.7.103/error_page/503.html