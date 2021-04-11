# 配置jenkins基于构建触发器自动拉取代码并部署
构建触发器(webhook)，有的人称为钩子，实际上是一个 HTTP 回调,其用于在开发人员向 gitlab 提交代码后能够触发 jenkins 自动执行代码构建操作。

以下为新建一个开发分支，只有在开发人员向开发(develop)分支提交代码的时候才会触发代码构建，而向主分支提交的代码不会自动构建，需要运维人员手动部署代码到生产环境。
![](images/11ab8f1da26176db1a34aef5b74ef7ca.png)

1. gitlab新建develop分支
![](images/60db2a215bb33755140e79fbd448b789.png)

2. jenkins安装需要的插件
系统管理-管理插件-可选插件-Gitlab Hook 和 Gitlab Authentication
安装插件后重启jenkins

对jenkins进行设置

在 jenkins 系统管理--全局安全设置，认证改为登录用户可以做任何事情并取消跨站请求伪造保护
![](images/b2b40e50f7e516d81c547516a21a52c6.png)

> 警告：Gitlab Hook Plugin 以纯文本形式存储和显示 GitLab API 令牌，不建议在生产环境使用

3. jenkins新建 develop job并设置为gitlab的develop分支
![](images/69eb8410c9b3f9050b2595934b11a700.png)

生产tocken认证
```bash
openssl rand -hex 12
3d744b7568daf3eef885bfce
```
![](images/531246d0207aff690bff9d6ede56ab4e.png)

jenkins 构建 shell 命令,构建命令为简单的测试命令，比如输出当前的账户信息：
![](images/674e70f1e6b0551618d5745c8a3cd09c.png)


jenkins 验证分支 job 配置文件
```bash
vim /var/lib/jenkins/jobs/develop/config.xml 
```
![](images/c456375fc5aedca838bd4b8de38a4a5d.png)

4. curl 命令测试触发并验证远程触发构建
使用浏览器直接访问 URL 地址
使用 curl 命令访问 URL
```bash
curl http://192.168.2.2:8080/job/develop/build?token=3d744b7568daf3eef885bfce
```

jenkins 验证 job 是否自动构建
![](images/7919db36c655baac6a46bd42bb581493.png)

5. gitlab 配置 webhook（系统勾子）
Admin area---groups/project—System Hooks
![](images/3ac6bc739588017af877699e791a35a0.png)

测试，返回值必须为201
![](images/9689e63c3aeb78cc2e272d3330a398bd.png)

6. jenkins将开发分支的执行 shell 命令更改为正式脚本
构建是jenkins下载的代码会下载到/var/lib/jenkins/workspace/ 目录下与job同名的目录
![](images/7baebf737a5f0dda2ba0fde2916cc105.png)
```bash
 cd /var/lib/jenkins/workspace/develop
tar czvf code.tar.gz index.html 
scp code.tar.gz www@192.168.2.7:/data/app_dir/

ssh www@192.168.2.7 "/etc/init.d/tomcat  stop && rm -rf /data/web_app/myapp/* && cd /data/app_dir/ &&  tar xvf code.tar.gz  -C /data/web_app/myapp/"

ssh www@192.168.2.7 "/etc/init.d/tomcat  start"
```

7. gitlab 开发分支 develop 测试提交代码
```bash
git clone http://192.168.2.1/linux_test/web1.git -b develop
vim index.html 
git add  ./*
git commit -m "develop test"
git push
```
jenkins 验证 develop job 自动构建
![](images/ff2cd3c760ab248cc07a1d9179e2441d.png)

## 构建后项目关联
用于多个 job 相互关联，需要穿行执行多个 job 的场景。

1. 配置构建后操作
![](images/e6659b8d2050b00fab9c9050e6420ce4.png)

2. 重新上传develop分支代码，查看master是否构建
![](images/ebaa145249f7509ac2082e129297b399.png)
master验证
![](images/435a91597d196daacaf5b2cf76ff1067.png)

# jenkins 分布式
在众多 Job 的场景下，单台 jenkins master 同时执行代码 clone、编译、打包及构建，其性能可能会出现瓶颈从而会影响代码部署效率，影响 jenkins 官方提供了 jenkins 分布式构建，将众多 job 分散运行到不同的 jenkins slave 节点，大幅提高并行 job 的处理能力。

1. 配置 slave 节点 java 环境
```bash
tar xvf jdk-8u212-linux-x64.tar.gz
mv jdk1.8.0_212 /usr/local/jdk
ln -sv /usr/local/jdk/bin/java /usr/bin/java

vim /etc/profile
export HISTTIMEFORMAT="%F %T `whoami` "
export export LANG="en_US.utf-8"
export JAVA_HOME=/usr/local/jdk
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
```

2. 创建 slave 节点的工作目录
Slave 服务器创建工作目录，如果 slave 需要执行编译 job，则也需要配置 java 环境并
且安装 git、svn、maven 等与 master 相同的基础运行环境，另外也要创建与 master 相
同的数据目录，因为脚本中调用的路径只有相对一 master 的一个路径，此路径在
master 与各 node 节点必须保持一致。
```bash
mkdir -pv /var/lib/jenkins/workspace

```

> slave 节点和 master 节点时间必须同步

配置 master 节点和 slave 节点的基于密钥登录
```bash

```

3. master 配置 slave 节点
Jenkins—系统管理—节点管理—新建节点
![](images/6946eacc1f88d117f105f4d79134b9a0.png)

![](images/cda3cdfac69617b6cebd2b7604e18ade.png)
验证方式基于用户名和密码
![](images/fa75468ed370c3387a7f60daabb5368e.png)
查看添加结果
![](images/a8d4c7415afc4d14f5a01ec41272a32e.png)
验证日志
![](images/2632367cf2b7ea1480d725f36d56b08c.png)

4. 创建指定目录并将java软连接到此处
```bash
mkdir /var/lib/jenkins/jdk/bin/ -pv
ln -sv /usr/local/jdk/bin/java /var/lib/jenkins/jdk/bin/
```