# 环境说明

系统：CentOS 7.7.1908
ip：192.168.15.188
部署目录：/opt
数据库: 5.5.64-MariaDB
代理:nginx/1.16.1

# 一站式部署

1. 配置防火墙和selinux
```bash
# 关闭selinux
[ root@jumpserver ~]# setenforce 0
[ root@jumpserver ~]# sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

# 配置防火墙nginx的端口和coco的ssh端口
[ root@jumpserver ~]# systemctl start firewalld
[ root@jumpserver ~]# systemctl enable firewalld
[ root@jumpserver ~]# firewall-cmd --zone=public --add-port=80/tcp --permanent
[ root@jumpserver ~]# firewall-cmd --zone=public --add-port=2222/tcp --permanent

# 安装基础软件包
yum install  vim iotop bc gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl  openssl-devel zip unzip zlib-devel  net-tools lrzsz tree ntpdate telnet lsof tcpdump wget libevent libevent-devel bc  systemd-devel bash-completion traceroute  bridge-utils  epel-release git -y

```



2. 安装redis
jumpserver 使用redis做cache
```bash
[ root@jumpserver ~]# yum -y install redis
[ root@jumpserver ~]# systemctl enable redis
[ root@jumpserver ~]# systemctl start redis
# 如果需要redis的其它配置,参考之前写的redis的相关文档
```

3. 安装mysql
```bash
yum -y install mariadb mariadb-devel mariadb-server MariaDB-shared
[ root@jumpserver ~]# systemctl enable mariadb
[ root@jumpserver ~]# systemctl start mariadb
# 数据库授权
[ root@jumpserver ~]# DB_PASSWORD="admin123" # 数据库的密码
[ root@jumpserver ~]# mysql -uroot -e "create database jumpserver default charset 'utf8'; grant all on jumpserver.* to 'jumpserver'@'127.0.0.1' identified by '$DB_PASSWORD'; flush privileges;"
```

4. 安装python3.6的环境
```bash
[ root@jumpserver ~]# yum -y install python36 python36-devel
# 创建python3.6的虚拟环境 py3可以自定义命名
[ root@jumpserver ~]# cd /opt
[ root@jumpserver ~]# python3.6 -m venv py3
[ root@jumpserver ~]# source /opt/py3/bin/activate
# 看到下面的提示符代表成功, 以后运行 Jumpserver 都要先运行以上 source 命令, 载入环境后默认以下所有命令均在该虚拟环境中运行
(py3) [ root@jumpserver opt]#
```

5. 下载jumpserver
```bash
(py3) [ root@jumpserver opt]# cd /opt/
(py3) [ root@jumpserver opt]# git clone https://github.com/jumpserver/jumpserver.git
(py3) [ root@jumpserver opt]# cd /opt/jumpserver
(py3) [ root@jumpserver opt]# git checkout 1.5.2

# 安装依赖 RPM 包
(py3) [ root@jumpserver opt]# yum -y install $(cat /opt/jumpserver/requirements/rpm_requirements.txt)

# 安装 Python 库依赖
(py3) [ root@jumpserver opt]# pip install --upgrade pip setuptools
(py3) [ root@jumpserver opt]# pip install -r /opt/jumpserver/requirements/requirements.txt
```

6. 修改jumpserver的配置文件
```bash
(py3) [ root@jumpserver opt]# cd /opt/jumpserver
(py3) [ root@jumpserver opt]# cp config_example.yml config.yml

# 生成随机SECRET_KEY,加密秘钥,不能是纯数字不可以
(py3) [ root@jumpserver opt]# SECRET_KEY=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 50`
(py3) [ root@jumpserver opt]# echo "SECRET_KEY=$SECRET_KEY" >> ~/.bashrc
# 生成随机BOOTSTRAP_TOKEN,预共享Token coco和guacamole用来注册服务账号
(py3) [ root@jumpserver opt]# BOOTSTRAP_TOKEN=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 16`
(py3) [ root@jumpserver opt]# echo "BOOTSTRAP_TOKEN=$BOOTSTRAP_TOKEN" >> ~/.bashrc
# 设置SECRET_KEY和BOOTSTRAP_TOKEN
(py3) [ root@jumpserver opt]# sed -i "s/SECRET_KEY:/SECRET_KEY: $SECRET_KEY/g" /opt/jumpserver/config.yml
(py3) [ root@jumpserver opt]# sed -i "s/BOOTSTRAP_TOKEN:/BOOTSTRAP_TOKEN: $BOOTSTRAP_TOKEN/g" /opt/jumpserver/config.yml
# 设置日志级别
(py3) [ root@jumpserver opt]# sed -i "s/# DEBUG: true/DEBUG: false/g" /opt/jumpserver/config.yml
(py3) [ root@jumpserver opt]# sed -i "s/# LOG_LEVEL: DEBUG/LOG_LEVEL: ERROR/g" /opt/jumpserver/config.yml
# 开启浏览器session过期时长,默认24小时
# SESSION_COOKIE_AGE: 86400
(py3) [ root@jumpserver opt]# sed -i "s/# SESSION_EXPIRE_AT_BROWSER_CLOSE: false/SESSION_EXPIRE_AT_BROWSER_CLOSE: true/g" /opt/jumpserver/config.yml

# 设置数据库的密码
(py3) [ root@jumpserver opt]# sed -i "s/DB_PASSWORD: /DB_PASSWORD: $DB_PASSWORD/g" /opt/jumpserver/config.yml

(py3) [ root@jumpserver opt]# echo -e "\033[31m 你的SECRET_KEY是 $SECRET_KEY \033[0m"
(py3) [ root@jumpserver opt]# echo -e "\033[31m 你的BOOTSTRAP_TOKEN是 $BOOTSTRAP_TOKEN \033[0m"

# 查看配置文件保证配置文件没有问题,如果redis设置了密码,需要在配置文件中指定redis的密码
```

7. 运行jumpserver
```bash
(py3) [ root@jumpserver opt]# cd /opt/jumpserver
(py3) [ root@jumpserver opt]# ./jms start all -d
```

## 部署coco和guacamole插件
coco和guacamole插件的安装要使用docker的方式安装,需要先安装docker

1. 安装docker
```bash
# 安装必要的工具包
(py3) [ root@jumpserver opt]# yum install -y yum-utils device-mapper-persistent-data lvm2
# 添加软件源信息
(py3) [ root@jumpserver opt]# yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
(py3) [ root@jumpserver opt]# yum makecache fast
# 安装证书
(py3) [ root@jumpserver opt]# rpm --import https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
# 安装docker
(py3) [ root@jumpserver opt]# yum -y install docker-ce
(py3) [ root@jumpserver opt]# systemctl enable docker
# 配置镜像加速地址
(py3) [ root@jumpserver opt]# curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io
(py3) [ root@jumpserver opt]# systemctl restart docker
```

2. 防火墙配置
允许 容器ip 访问宿主 8080 端口(jumpserver启动的时候默认监听的是本地的8080端口,如果更改请在此处也更改)
```bash
(py3) [ root@jumpserver opt]# firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="172.17.0.0/16" port protocol="tcp" port="8080" accept"
(py3) [ root@jumpserver opt]# firewall-cmd --reload
```

3. 部署coco和guacamole插件
```bash
# 获取当前服务器 IP
(py3) [ root@jumpserver opt]# Server_IP=`ip addr | grep inet | egrep -v '(127.0.0.1|inet6|docker)' | awk '{print $2}' | tr -d "addr:" | head -n 1 | cut -d / -f1`

# http://<Jumpserver_url> 指向 jumpserver 的服务端口, 如 http://192.168.244.144:8080
# BOOTSTRAP_TOKEN 为 Jumpserver/config.yml 里面的 BOOTSTRAP_TOKEN
(py3) [ root@jumpserver opt]# docker run --name jms_koko -d -p 2222:2222 -p 5000:5000 -e CORE_HOST=http://$Server_IP:8080 -e BOOTSTRAP_TOKEN=$BOOTSTRAP_TOKEN jumpserver/jms_koko:1.5.2
(py3) [ root@jumpserver opt]# docker run --name jms_guacamole -d -p 8081:8081 -e JUMPSERVER_SERVER=http://$Server_IP:8080 -e BOOTSTRAP_TOKEN=$BOOTSTRAP_TOKEN jumpserver/jms_guacamole:1.5.2
```

4. 安装luna
安装 Web Terminal 前端: Luna  需要 Nginx 来运行访问 访问(https://github.com/jumpserver/luna/releases)下载对应版本的 release 包, 直接解压, 不需要编译
```bash
(py3) [ root@jumpserver opt]# cd /opt
(py3) [ root@jumpserver opt]# wget https://github.com/jumpserver/luna/releases/download/1.5.2/luna.tar.gz

# 如果网络有问题导致下载无法完成可以使用下面地址
(py3) [ root@jumpserver opt]# wget https://demo.jumpserver.org/download/luna/1.5.2/luna.tar.gz

(py3) [ root@jumpserver opt]# tar xf luna.tar.gz
(py3) [ root@jumpserver opt]# chown -R root:root luna
```

## 部署nginx

1. 创建nginx的软件源并安装
```bash
(py3) [ root@jumpserver opt]# vim /etc/yum.repos.d/nginx.repo

[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1

(py3) [ root@jumpserver opt]# yum -y install nginx
(py3) [ root@jumpserver opt]# systemctl enable ngin
```

2. 配置 Nginx 整合各组件
```bash
(py3) [ root@jumpserver opt]# /etc/nginx/conf.d/jumpserver.conf
server {
    listen 80;

    client_max_body_size 100m;  # 录像及文件上传大小限制

    location /luna/ {
        try_files $uri / /index.html;
        alias /opt/luna/;  # luna 路径, 如果修改安装目录, 此处需要修改
    }

    location /media/ {
        add_header Content-Encoding gzip;
        root /opt/jumpserver/data/;  # 录像位置, 如果修改安装目录, 此处需要修改
    }

    location /static/ {
        root /opt/jumpserver/data/;  # 静态资源, 如果修改安装目录, 此处需要修改
    }

    location /socket.io/ {
        proxy_pass       http://localhost:5000/socket.io/;
        proxy_buffering off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        access_log off;
    }

    location /coco/ {
        proxy_pass       http://localhost:5000/coco/;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        access_log off;
    }

    location /guacamole/ {
        proxy_pass       http://localhost:8081/;
        proxy_buffering off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $http_connection;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        access_log off;
    }

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

3. 启动nginx
```bash
(py3) [ root@jumpserver opt]# nginx -t   # 确保配置没有问题, 有问题请先解决
(py3) [ root@jumpserver opt]# systemctl start nginx
```

4. 测试jumpserver
```bash
# 访问 http://192.168.244.144 (注意 没有 :8080 通过 nginx 代理端口进行访问)
# 默认账号: admin 密码: admin  到会话管理-终端管理 接受 coco Guacamole 等应用的注册
# 测试连接
$ ssh -p2222 admin@192.168.244.144
$ sftp -P2222 admin@192.168.244.144
  密码: admin

# 如果是用在 Windows 下, Xshell Terminal 登录语法如下
$ ssh admin@192.168.244.144 2222
$ sftp admin@192.168.244.144 2222
  密码: admin
  如果能登陆代表部署成功

# sftp默认上传的位置在资产的 /tmp 目录下
# windows拖拽上传的位置在资产的 Guacamole RDP上的 G 目录下
```

5. 登录网页访问测试
登录后在会话管理中查看插件注册成功,如果注册失败,删除插件重新部署
![](http://aishad.top/wordpress/wp-content/uploads/2019/11/ac8edee2e99a08fc85a931777988b911.png)

## 多组件负载说明
```bash
# coco 服务默认运行在单核心下面, 当负载过高时会导致用户访问变慢, 这时可运行多个 docker 容器缓解
$ docker run --name jms_koko01 -d -p 2223:2222 -p 5001:5000 -e CORE_HOST=http://<Jumpserver_url> -e BOOTSTRAP_TOKEN=****** jumpserver/jms_koko:1.5.2
$ docker run --name jms_koko02 -d -p 2224:2222 -p 5002:5000 -e CORE_HOST=http://<Jumpserver_url> -e BOOTSTRAP_TOKEN=****** jumpserver/jms_koko:1.5.2
...

# guacamole 也是一样
$ docker run --name jms_guacamole01 -d -p 8082:8081 -e JUMPSERVER_SERVER=http://<Jumpserver_url> -e BOOTSTRAP_TOKEN=****** jumpserver/jms_guacamole:1.5.2
$ docker run --name jms_guacamole02 -d -p 8083:8081 -e JUMPSERVER_SERVER=http://<Jumpserver_url> -e BOOTSTRAP_TOKEN=****** jumpserver/jms_guacamole:1.5.2
...

# nginx 代理设置
$ vi /etc/nginx/nginx.conf
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}

# 加入 tcp 代理
stream {
    log_format  proxy  '$remote_addr [$time_local] '
                       '$protocol $status $bytes_sent $bytes_received '
                       '$session_time "$upstream_addr" '
                       '"$upstream_bytes_sent" "$upstream_bytes_received" "$upstream_connect_time"';

    access_log /var/log/nginx/tcp-access.log  proxy;
    open_log_file_cache off;

    upstream cocossh {
        server localhost:2222 weight=1;
        server localhost:2223 weight=1;  # 多节点
        server localhost:2224 weight=1;  # 多节点
        # 这里是 coco ssh 的后端ip
        hash $remote_addr;
    }
    server {
        listen 2220;  # 不能使用已经使用的端口, 自行修改, 用户ssh登录时的端口
        proxy_pass cocossh;
        proxy_connect_timeout 10s;
    }
}
# 到此结束

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    # tcp_nopush     on;

    keepalive_timeout  65;

    # 关闭版本显示
    server_tokens off;

    include /etc/nginx/conf.d/*.conf;
}

$ firewall-cmd --zone=public --add-port=2220/tcp --permanent
$ firewall-cmd --reload

$ vi /etc/nginx/conf.d/jumpserver.conf
upstream jumpserver {
    server localhost:8080;
    # 这里是 jumpserver 的后端ip
}

upstream kokows {
    server localhost:5000 weight=1;
    server localhost:5001 weight=1;  # 多节点
    server localhost:5002 weight=1;  # 多节点
    # 这里是 koko ws 的后端ip
    ip_hash;
}

upstream guacamole {
    server localhost:8081 weight=1;
    server localhost:8082 weight=1;  # 多节点
    server localhost:8083 weight=1;  # 多节点
    # 这里是 guacamole 的后端ip
    ip_hash;
}

server {
    listen 80;
    server_name demo.jumpserver.org;  # 自行修改成你的域名

    client_max_body_size 100m;  # 录像上传大小限制

    location / {
        proxy_pass http://jumpserver;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        access_log off;
    }

    location /luna/ {
        try_files $uri / /index.html;
        alias /opt/luna/;
    }

    location /media/ {
        add_header Content-Encoding gzip;
        root /opt/jumpserver/data/;  # 录像位置, 如果修改安装目录, 此处需要修改
    }

    location /static/ {
        root /opt/jumpserver/data/;  # 静态资源, 如果修改安装目录, 此处需要修改
    }

    location /socket.io/ {
        proxy_pass       http://kokows/socket.io/;  # coco
        proxy_buffering off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        access_log off;
    }

    location /coco/ {
        proxy_pass       http://kokows/coco/;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        access_log off;
    }

    location /guacamole/ {
        proxy_pass       http://guacamole/;  #  guacamole
        proxy_buffering off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $http_connection;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        access_log off;
    }
}

$ nginx -t
$ nginx -s reload
```