# 手动镜像制作
Docker 制作类似于虚拟机的镜像制作，即按照公司的实际业务务求将需要安装的软件、相关配置等基础环境配置完成，然后将其做成镜像，最后再批量从镜像批量生产实例，这样可以极大的简化相同环境的部署工作，Docker 的镜像制作分为手动制作和自动制作(基于 DockerFile)，其中手动制作镜像步骤具体如下：


## 手动制作 yum 版 nginx 镜像

1. 下载镜像并初始化系统
基于某个基础镜像之上重新制作，因此需要先有一个基础镜像，本次使用官方提供的 centos 镜像为基础
```bash
[ root@node1 ~]# docker pull centos
```

2. 进入容器中安装并配置nginx
```bash
[ root@node1 ~]# docker run -it centos:latest  bash

[root@1b280639a458 /]# yum -y install epel-release 

[root@1b280639a458 /]# yum -y install nginx

# 安装常用命令
[root@1b280639a458 /]# yum install  vim iotop bc gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl  openssl-devel zip unzip zlib-devel  net-tools lrzsz tree ntpdate telnet lsof tcpdump wget libevent libevent-devel bc  systemd-devel bash-completion traceroute  bridge-utils -y
```

3. 安装nginx后台运行
容器要想长期运行，必须有个前台进程作为守护进程一直运行
```bash
[root@1b280639a458 /]# vim /etc/nginx/nginx.conf
daemon off;
```

4. 自定义web界面
```bash
[root@1b280639a458 /]# vim /usr/share/nginx/html/index.html
Docker Yum Nginx
```

5. 另一个窗口使用宿主机提交为镜像
```bash
[ root@node1 ~]# docker commit -m "nginx base image" 1b280639a458 weiying_nginx:v1
sha256:98aa0696d3030bdfb4ecbf19d59e7bc8753a79edb14449c885963e86d2eab102
[ root@node1 ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
weiying_nginx       v1                  98aa0696d303        9 seconds ago       565MB
centos              latest              9f38484d220f        3 months ago        202MB
```

6. 从自己的镜像启动容器
```bash
[ root@node1 ~]# docker run -d -p 80:80 --name my_centos_nginx weiying_nginx:v1 /usr/sbin/nginx
2354a29417acd1c7510b196bd2155c2403ba0133ac776f292d9ae9e35f956a45
[ root@node1 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                NAMES
2354a29417ac        weiying_nginx:v1    "/usr/sbin/nginx"   5 seconds ago       Up 3 seconds        0.0.0.0:80->80/tcp   my_centos_nginx
```

7. 访问宿主机的80端口测试


## Dockerfile 制作yum版nginx镜像
DockerFile 可以说是一种可以被 Docker 程序解释的脚本，DockerFile 是由一条条的命令组成的，每条命令对应 linux 下面的一条命令，Docker 程序将这些DockerFile 指令再翻译成真正的 linux 命令，其有自己的书写方式和支持的命令，Docker 程序读取 DockerFile 并根据指令生成 Docker 镜像，相比手动制作镜像的方式，DockerFile 更能直观的展示镜像是怎么产生的，有了 DockerFile，当后期有额外的需求时，只要在之前的 DockerFile 添加或者修改响应的命令即可重新生成新的 Docke 镜像，避免了重复手动制作镜像的麻烦

1. 下载镜像并初始化
```bash
#目录结构按照业务类型或系统类型等方式划分，方便后期镜像比较多的时候进行分类
[ root@node1 ~]# mkdir /opt/nginx

[ root@node1 nginx]# docker pull centos
```

2. 编写Dockerfile文件
```bash
[ root@node1 ~]# cd /opt/nginx
[ root@node1 nginx]# vim Dockerfile #生成的镜像的时候会在执行命令的当前目录查找 Dockerfile 文件，所以名称不可写错，而且 D 必须大写
# my Dockerfile
#
#
from centos
maintainer weiying 2286416563@qq.com

run yum -y install epel-release
run yum install -y nginx
run rm -rf /usr/share/nginx/html/index.html
add index.tar.gz /usr/share/nginx/html
add conf.tar.gz /etc/nginx

expose 80 443
cmd ["nginx"]
```

3. 准备dockerfile需要的文件
```bash
[ root@node1 nginx]# tree
.
├── conf.tar.gz
├── Dockerfile
└── index.tar.gz

0 directories, 3 files
```

4. 执行镜像构建
```bash
[ root@node1 nginx]# docker build -t nginx:v2 .
# 查看镜像是否构建成功
[ root@node1 nginx]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
nginx               v2                  aa6220a5bd41        About a minute ago   509MB
nginx               v1                  307d9707c76f        29 minutes ago       564MB
centos              latest              9f38484d220f        3 months ago         202MB
```

5. 从镜像启动容器
```bash
[ root@node1 nginx]# docker run -d -d -p 80:80 --name dockerfile_nginx nginx:v2 nginx
73014066d2c1a500641e5a1fdaee74ebf965e4246f17682db3a22166f84cc002
```

6. 访问测试

### Dockerfile文件参数详解

"#"字符开头的为注释信息

from xxx：除了#字符开头的第一行必须是from，用来指明基础镜像，如果本地没有指定的基础镜像，那么会自动到镜像仓库中获取

maintainer weiying 2286416563@qq.com：用来指明镜像的维护者信息

user：指定该容器运行时的用户名和UID，后续的run命令也会使用这里指定的用户执行

workdir /a
workdir b ：指定工作目录，最终的工作目录为/a/b

volume ["/dir_1", "/dir_2" ..]：设置容器挂载宿主机的目录

env name jack：用来设置容器变量，常用于向容器中传递用户名密码等

run：用来指明要在容器中执行的命令，必须是非交互式

add：用来将宿主机当前目录下的文件拷贝到容器的指定位置，会自动解压.tat.gz 的文件

expose 80 443：用来指明向外开放的端口，多个端口之间使用空格隔开，启动容器是-p需要使用此处指定的端口向外映射，如-p 8080:80 此处的80为使用expose指定的容器呢的端口

cmd ["命令","参数",...]：运行的命令，每个Dockerfile只能有一条，如果有多条则之后最后一个会被执行

> 如果在从该镜像启动容器的时候也指定了命令，那么指定的命令会覆盖Dockerfile 构建的镜像里面的 CMD 命令，即指定的命令优先级更高，Dockerfile 的优先级较低一些

### 使用Dockerfile源码编译安装nginx

1. 编写Dockerfile文件
```bash
[ root@node1 nginx_binary]# vim Dockerfile
#binary nginx
#
#
from centos
maintainer weiying 2286416563@qq.com

run yum -y install vim lrzsz tree screen psmisc lsof tcpdump wget ntpdate gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl openssl-devel systemd-devel net-tools iotop bc zip unzip zlib-devel bash-completion nfs-utils automake libxml2 libxml2-devel libxslt libxslt-devel perl perl-ExtUtils-Embed

run yum -y install epel-release
run mkdir /data
add nginx-1.12.2.tar.gz /data
run cd /data/nginx-1.12.2 && ./configure --prefix=/apps/nginx --user=nginx --group=nginx --with-http_ssl_module --with-http_v2_module --with-http_realip_module --with-http_stub_status_module --with-http_gzip_static_module --with-pcre --with-stream --with-stream_ssl_module --with-stream_realip_module && make && make install

add conf.tar.gz /apps/nginx/conf
run echo "binary nginx" > /apps/nginx/html/index.html
run useradd nginx -s /sbin/nologin
run ln -sv /apps/nginx/sbin/nginx /usr/sbin/
run chown nginx.nginx -R /apps/nginx

expose 80 443

cmd ["nginx"]
```

2. 准备nginx的配置文件
```bash
# 更改nginx 的配置文件
[ root@node1 nginx_binary]# vim nginx.conf
user  nginx;
worker_processes  auto;
daemon off;
master_process on;

[ root@node1 nginx_binary]# tar czvf conf.tar.gz nginx.conf 
```

3. 准备nginx的二进制源码程序包
```bash
[ root@node1 nginx_binary]# tree /opt/nginx_binary/
/opt/nginx_binary/
├── conf.tar.gz
├── Dockerfile
├── nginx-1.12.2.tar.gz
└── nginx.conf

0 directories, 4 files
```

4. 构建镜像
```bash
[ root@node1 nginx_binary]# docker build -t nginx:v3 .


[ root@node1 nginx_binary]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               v3                  cd6086a0f7a4        7 minutes ago       673MB
nginx               v2                  aa6220a5bd41        2 hours ago         509MB
nginx               v1                  307d9707c76f        2 hours ago         564MB
centos              latest              9f38484d220f        3 months ago        202MB
```

5. 从镜像启动容器
```bash
[ root@node1 nginx_binary]# docker run  -d -p 80:80 --name binary_nginx nginx:v3 nginx
809df68dda9dd28979e90cfa600ca16a059d9cf1cf8ba047acca0b388de8ffeb
[ root@node1 nginx_binary]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                         NAMES
809df68dda9d        nginx:v3            "nginx"             3 seconds ago       Up 3 seconds        0.0.0.0:80->80/tcp, 443/tcp   binary_nginx
```

6. 访问测试


### 自定义tomcat镜像

#### 构建基础jdk镜像

1. 准备Dockerfile文件
```bash
[ root@node1 nginx_binary]# mkdir /opt/jdk
[ root@node1 nginx_binary]# cd /opt/jdk

[ root@node1 jdk]# vim Dockerfile 
from centos_base:v1

MAINTAINER weiying "2286416563@qq.com"

add jdk-8u192-linux-x64.tar.gz /usr/local/src
run ln -sv /usr/local/src/jdk1.8.0_192 /usr/local/src/jdk

ENV JAVA_HOME /usr/local/src/jdk
ENV JRE_HOME $JAVA_HOME/jre
ENV CLASSPATH $JAVA_HOME/lib/:$JRE_HOME/lib/
ENV PATH $PATH:$JAVA_HOME/bin
ADD profile /etc/profile

run rm -rf rm -rf /etc/localtime && ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo  "Asia/Shanghai" > /etc/timezone
```

2. 准备build脚本
```bash
[ root@node1 jdk]# vim build-command.sh
#!/bin/bash

docker build -t jdk-base:1.8.0.192 .
```

3. 准备jdk压缩包和profile文件
```bash
[ root@node1 jdk]# tree /opt/jdk/
/opt/jdk/
├── build-command.sh
├── Dockerfile
├── jdk-8u192-linux-x64.tar.gz
└── profile

0 directories, 4 files

[ root@node1 jdk]# vim profile
# /etc/profile

# System wide environment and startup programs, for login setup
# Functions and aliases go in /etc/bashrc

# It's NOT a good idea to change this file unless you know what you
# are doing. It's much better to create a custom.sh shell script in
# /etc/profile.d/ to make custom changes to your environment, as this
# will prevent the need for merging in future updates.

pathmunge () {
    case ":${PATH}:" in
        *:"$1":*)
            ;;
        *)
            if [ "$2" = "after" ] ; then
                PATH=$PATH:$1
            else
                PATH=$1:$PATH
            fi
    esac
}


if [ -x /usr/bin/id ]; then
    if [ -z "$EUID" ]; then
        # ksh workaround
        EUID=`/usr/bin/id -u`
        UID=`/usr/bin/id -ru`
    fi
    USER="`/usr/bin/id -un`"
    LOGNAME=$USER
    MAIL="/var/spool/mail/$USER"
fi

# Path manipulation
if [ "$EUID" = "0" ]; then
    pathmunge /usr/sbin
    pathmunge /usr/local/sbin
else
    pathmunge /usr/local/sbin after
    pathmunge /usr/sbin after
fi

HOSTNAME=`/usr/bin/hostname 2>/dev/null`
HISTSIZE=1000
if [ "$HISTCONTROL" = "ignorespace" ] ; then
    export HISTCONTROL=ignoreboth
else
    export HISTCONTROL=ignoredups
fi

export PATH USER LOGNAME MAIL HOSTNAME HISTSIZE HISTCONTROL

# By default, we want umask to get set. This sets it for login shell
# Current threshold for system reserved uid/gids is 200
# You could check uidgid reservation validity in
# /usr/share/doc/setup-*/uidgid file
if [ $UID -gt 199 ] && [ "`/usr/bin/id -gn`" = "`/usr/bin/id -un`" ]; then
    umask 002
else
    umask 022
fi

for i in /etc/profile.d/*.sh /etc/profile.d/sh.local ; do
    if [ -r "$i" ]; then
        if [ "${-#*i}" != "$-" ]; then 
            . "$i"
        else
            . "$i" >/dev/null
        fi
    fi
done

unset i
unset -f pathmunge

export JAVA_HOME=/usr/local/src/jdk
export TOMCAT_HOME=/apps/tomcat
export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$TOMCAT_HOME/bin:$PATH
export CLASSPATH=.$CLASSPATH:$JAVA_HOME/lib:$JAVA_HOME/jre/lib:$JAVA_HOME/lib/tools.jar

```

4. 执行脚本，构建镜像
```bash
[ root@node1 jdk]# bash build-command.sh 
```

5. 从容器启动镜像并验证
```bash
[ root@node1 jdk]# docker run -it jdk-base:1.8.0.192 bash
[root@7328ae9e21d4 /]# java -version
java version "1.8.0_192"
Java(TM) SE Runtime Environment (build 1.8.0_192-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.192-b12, mixed mode)
[root@7328ae9e21d4 /]# date
Tue Jul  9 12:21:27 CST 2019
```

#### 从jdk镜像制作tomcat镜像

1. 创建tomcat基础镜像Dockerfile
```bash
[ root@node1 tomcat]# mkdir /opt/tomcat
[ root@node1 tomcat]# cd /opt/tomcat

[ root@node1 tomcat]# vim Dockerfile
#tomcat base
#
#
from jdk-base:1.8.0.192
MAINTAINER weiying "2286416563@qq.com"

run mkdir -pv /apps

add apache-tomcat-8.5.37.tar.gz /apps
run useradd tomcat
run chown tomcat:tomcat -R /apps/apache-tomcat-8.5.37
run ln -sv /apps/apache-tomcat-8.5.37 /apps/tomcat && ln -sv /apps/tomcat/bin/* /usr/bin/
```

2. 创建build脚本
```bash
[ root@node1 tomcat]# cat build-command.sh 
#!/bin/bash

docker build -t tomcat-base:8.5.37 .
```

3. 上传tomcat程序包
```bash
[ root@node1 tomcat]# tree /opt/tomcat/
/opt/tomcat/
├── apache-tomcat-8.5.37.tar.gz
├── build-command.sh
└── Dockerfile

```
4. 创建tomcat基础镜像
```bash
[ root@node1 tomcat]# bash build-command.sh 

```

5. 创建tomcat应用镜像
```bash
[ root@node1 opt]# mkdir /opt/tomcat-app1
[ root@node1 opt]# cd /opt/tomcat-app1/

[ root@node1 tomcat-app1]# vim Dockerfile 
#tomcat base
#
#
from tomcat-base:8.5.37
MAINTAINER weiying "2286416563@qq.com"

run mkdir -pv /data/webapps/app1
add code.tar.gz /data/webapps/app1
add server.xml /apps/tomcat/conf
add run_tomcat.sh /apps/tomcat/bin
run chmod +x /apps/tomcat/bin/run_tomcat.sh
run chown -R tomcat:tomcat /data/webapps

expose 8080 8443

cmd ["/apps/tomcat/bin/run_tomcat.sh"]
```

6. 准备首页文件
```bash
[ root@node1 tomcat-app1]# cat index.html 
tomcat
[ root@node1 tomcat-app1]# tar -czvf code.tar.gz index.html
```

7. 准备tomcat的配置文件
```bash
[ root@node1 tomcat-app1]# vim server.xml 
# 更改配置文件中的appBase指向放代码的目录
<Host name="localhost"  appBase="/data/webapps"
            unpackWARs="true" autoDeploy="true">
```

8. 准备tomcat启动脚本文件
```bash
[ root@node1 tomcat-app1]# vim run_tomcat.sh 
#!/bin/bash
source /etc/profile
#echo "1.1.1.1 www.weiying.net" >> /etc/hosts
#su - tomcat -c "/apps/tomcat/bin/catalina.sh start"
su - tomcat -c "/apps/tomcat/bin/catalina.sh  run"
#tail -f /etc/hosts
```

9. 根据build脚本构建镜像
```bash
[ root@node1 tomcat-app1]# cat build-command.sh 
#!/bin/bash

docker build -t tomcat-app1:v1 .

[ root@node1 tomcat-app1]# bash build-command.sh
```

10. 从镜像创建容器
```bash
[ root@node1 tomcat-app1]# docker run -it -d --name tomcat_apps tomcat-app1:v1 
72067964033f8eb5f6f52fbfc89b7158392f9c7125a7a7c81086101e6f19a992
```

## Dockerfile制作HAProxy
1. 准备HAProxy的Dockerfile文件
```bash
[ root@node1 haproxy]# vim Dockerfile

# my Dockerfile
#
#
from centos_base:v1
maintainer weiying 2286416563@qq.com

add haproxy-1.8.17.tar.gz /usr/local/src

run yum -y install gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl openssl-devel systemd-devel net-tools vim iotop bc zip unzip zlib-devel lrzsz tree screen lsof tcpdump wget ntpdate
run cd /usr/local/src/haproxy-1.8.17 && make ARCH=x86_64 TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1 USE_SYSTEMD=1 USE_CPU_AFFINITY=1 PREFIX=/usr/local/haproxy && make install PREFIX=/usr/local/haproxy && cp haproxy /usr/bin/

add haproxy.cfg /etc/haproxy/haproxy.cfg

add run_haproxy.sh /usr/bin
run chmod +x /usr/bin/run_haproxy.sh

expose 80 9999

cmd ["/usr/bin/run_haproxy.sh"]
```

2. 准备haproxy的配置文件
```bash
global
maxconn 100000
chroot /usr/local/haproxy
#stats socket /var/lib/haproxy/haproxy.sock mode 600 level admin
uid 99
gid 99
daemon
#nbproc 2
#cpu-map 1 0
#cpu-map 2 1
pidfile /usr/local/haproxy/run/haproxy.pid
log 127.0.0.1 local3 info

defaults
option http-keep-alive
option  forwardfor
maxconn 100000
mode http
timeout connect 300000ms
timeout client  300000ms
timeout server  300000ms

listen stats
 mode http
 bind 0.0.0.0:9999
 stats enable
 log global
 stats uri     /haproxy-status
 stats auth    haadmin:123456

listen  web_port
 bind 0.0.0.0:80
 mode http
 log global
 stats uri     /haproxy-status
 stats auth    haadmin:123456

listen  web_port
 bind 0.0.0.0:80
 mode http
 log global
 server web1  192.168.6.100:8080  check inter 3000 fall 2 rise 5
```

3. 准备haproxy的启动脚本文件
```bash
[ root@node1 haproxy]# vim run_haproxy.sh 

#!/bin/bash

haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
tail -f /etc/hosts
```

4. 使用build脚本创建镜像
```bash
[ root@node1 haproxy]# vim build-command.sh 

#!/bin/bash
docker build -t haproxy-base:7.5-1.8.12 .

[ root@node1 haproxy]# bash build-command.sh
```

5. 从镜像启动容器并测试
```bash
[ root@node1 haproxy]# docker run -p -d 80:80 -p 9999:9999 haproxy-base:7.5-1.8.12
```