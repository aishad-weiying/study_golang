# 系统安装
启动安装过程一般应位于引导设备；后续的anaconda及其安装用到的程序包等可来自下面几种方式：
1）本地光盘
2）本地硬盘
3）NFS
4）URL:
	ftp server: yum repository
	http server: yum repostory

如果想手动指定安装源：boot: linux askmethod

## 手动指定安装源

- CentOS6：
	DVD drive  		repo=cdrom :device
	Hard Drive  	repo=hd:device/path
	HTTP Server  	repo=http://host/path
	HTTPS Server 	repo=https://host/path
	FTP Server 		repo=ftp://username:password@host/path
	NFS Server 		repo=nfs:server:/path
	ISO images on an NFS Server repo=nfsiso:server:/path

- CentOS7：
	Any CD/DVD  	drive inst.repo=cdrom
	Hard Drive  	inst.repo=hd:device:/path
	HTTP Server  	inst.repo=http://host/path
	HTTPS Server  	inst.repo=https://host/path
	FTP Server  	inst.repo=ftp://username:password@host/path
	NFS Server  	inst.repo=nfs:[options:]server:/path

### anaconda的配置方式：
1. 交互式配置

2. 通过读取事先给定的配置文件自动完成配置
	按特定语法给出的配置选项
	kickstart文件

### kickstart文件格式
安装完成操作系统后，默认会在root的家目录下生产一个文件 anaconda-ks.cfg ，里面记录了本次安装过程中配置文件的选项

- 命令段
指明各种安装前配置，如键盘程序类型

- 程序包段：指明要安装的程序包或程序包组，不安装的程序包
	%packages：开始符
	@group_name：	要安装的包组
	package： 	要安装的程序包
	-package ：	不安装的程序包
	%end ：		结束符

- 脚本段：
	%pre：安装前脚本
	运行环境：运行于安装介质上微型Linux环境
	%end
	%post：安装后脚本(重启之前，安装完成之后运行的脚本)
	运行环境：安装完成后的系统
	%end

kickstart文件中必备的格式：
```bash
必须的：
        authconfig：认证方式
                authconfig --enableshadow(认证文件为/etc/shadow) --passalgo=sha512(密码加密格式)
        bootLoader：BootLoader的安装位置和相关选项
                bootloader --location=mbr --driveorder=sda --append="crashkernel=auto rhgb quiet"
        keyboard：设定键盘类型
                keyboard us
        lang：设定语言类型
                lang en_US.UTF-8
        part：创建分区(指明分区大小、挂载点等)
                #part /boot --fstype=ext2 --size=512
                #part / --fstype=ext4 --size=10240
                #part swap --grow --maxsize=2048 --size=2048
        rootpw：指明root用户的密码
                rootpw  --iscrypted $1$PNdoQ/HV$6NuRUT7ngRPC3potFlsb21
        timezone：指明时区
                timezone --utc Asia/Shanghai

可选的：
        install or upgrade
        test 文本安装界面
        network
                network --onboot no --device eth0 --bootproto static --ip 192.168.1.1 --netmask 255.255.255.0 --gateway 1.1.1.1 --hostname d01
                network --onboot no --device eth1 --bootproto dhcp --noipv6 --hostname d01
        firewall
        selinux
        halt
        poweroff
        reboot
        repo 额外用到的yum源的指明
        user 安装完成后直接创建新用户
        url 指明安装源
```

kickstart 文件的创建

1. 直接手动编辑
依据模板文件修改

2. 使用创建工具：system-config-kickstart
依据某模板修改并生成新配置/root/anaconda-ks.cfg

3. 检查ks文件的语法错误：ksvalidator
ksvalidator /PATH/TO/KICKSTART_FILE

安装系统时指定kickstart文件的位置
```bash
ks=
	DVD DRIVCE:ks=cdrom:/PATH/TO/KICKSTART_FILE 
	HARD DRIVCE:ks=hd:/device/drectory/KICKSTART_FILE 
	HTTP SERVER:ks=http://host:port/PATH/TO/KICKSTART_FILE 
	FTP SERVER:ks=ftp://host:port/PATH/TO/KICKSTART_FILE 
	HTTPS SERVER:ks=https://host:port/PATH/TO/KICKSTART_FILE 
```

## 光盘中isolinux目录列表

1. solinux.bin：光盘引导程序，在mkisofs的选项中需要明确给出文件路径，这个文件属于SYSLINUX项目

2. isolinux.cfg：isolinux.bin的配置文件，当光盘启动后（即运行isolinux.bin），会自动去找isolinux.cfg文件

3. vesamenu.c32：是光盘启动后的安装图形界面，也属于SYSLINUX项目，menu.c32版本是纯文本的菜单

4. Memtest：内存检测，这是一个独立的程序

5. splash.jgp：光盘启动界面的背景图

6. vmlinuz是内核映像

7. initrd.img是ramfs (先cpio，再gzip压缩)


### 制作引导光盘和U盘

```bash
  #创建引导光盘：
        mkdir –pv /app/myiso
        cp -r /misc/cd/isolinux/ /app/myiso/

        vim /app/myiso/isolinux/isolinux.cfg
                label linux
                        menu label ^Install CentOS 6.9 (^后面跟的字母位快捷键)
                        menu default （默认的选项）
                        kernel vmlinuz
                         append initrd=initrd.img text ks=cdrom:/myks.cfg

        cp /root/myks.cfg /app/myiso/

        mkisofs -R -J -T -v --no-emul-boot --boot-load-size 4 --boot-info-table -V "CentOS 6.9 x86_64 boot()" -b isolinux/ioslinux.bin(指明引导文件) -c ioslinux/boot.cat(指明第一阶段) -o /root/boot.iso(保存位置) /app/myiso/(对哪个目录创建)
         注意：以上相对路径都是相对于光盘的根，和工作目录无关

 mkisofs选项
        -o 指定映像文件的名称。
        -b 指定在制作可开机光盘时所需的开机映像文件。
        -c 制作可开机光盘时，会将开机映像文件中的 no-eltorito-catalog 全部内容作成一个文件。
        -no-emul-boot 非模拟模式启动。
        -boot-load-size 4 设置载入部分的数量
        -boot-info-table 在启动的图像中现实信息
        -R 或 -rock 使用 Rock RidgeExtensions
        -J 或 -joliet 使用 Joliet 格式的目录与文件名称
        -v 或 -verbose 执行时显示详细的信息
        -T 或 -translation-table 建立文件名的转换表，适用于不支持 Rock RidgeExtensions 的系统上

创建U盘启动盘
        dd if=/dev/sr0 of=/dev/sdb
```