# tomcat 启动脚本
```bash
#!/bin/bash
# #########################################################
# Tomcat init script ####
###########################################################
# chkconfig: 2345 96 14 ###################################
# description: 2018/08/06. 魏颖##########################
# #########################################################

JDK_HOME=/apps/jdk1.7.0_79
CATALINA_HOME=/apps/tomcat
export JDK_HOME CATALINA_HOME
source /etc/profile
#PID=`ps -ef  | grep  -v grep  | grep java | awk  '{print $2}'`
#NUM=`ps -ef  | grep  -v grep  | grep java | awk  '{print $2}' | wc -l`

#case $1 in
start() {
    	echo "正在判断服务状态，请稍等！"	
    	echo "请稍等3秒钟"
    	echo "3";sleep 1;echo "2";sleep 1;echo "1";sleep 1
   	if	netstat -an | grep 8080 | grep LISTEN >/dev/null
     	then
   		echo "Tomcat已经正在运行了！"  
  	else 
   		echo "Tomcat没有运行，1秒后启动！"
		echo 1;sleep 1  
  		$CATALINA_HOME/bin/catalina.sh start 
  		echo  "Tomcat 已经成功启动完成,5秒后判断是否启动成功"
  		echo "5";sleep 1;echo "4";sleep 1
        echo "3";sleep 1;echo "2";sleep 1;echo "1";sleep 1
	if  netstat -an | grep 8080 | grep LISTEN >/dev/null
	    then
		PID=`ps -ef | grep  tomcat | grep jdk | awk '{print $2}'`
		NUM=`ps -ef | grep  tomcat | grep jdk | awk '{print $2}' | wc -l`
		echo "Tomcat 已经成功启动${NUM} 个Tomcat进程!,PID为${PID}"
	    else
		echo "Tomcat启动失败，请重新启动！"
        	echo 1
	fi
 	fi
	}
stop() {
		PID=`ps -ef  | grep  -v grep  | grep java | awk  '{print $2}'`
		NUM=`ps -ef | grep  -v "color"  | grep tomcat | awk '{print $2}' | wc -l`
		echo "正在判断服务状态，请稍等3秒钟！"	
		echo "3";sleep 1;echo "2";sleep 1;echo "1";sleep 1
	if  netstat -an | grep 8080 | grep LISTEN >/dev/null 
	   then	
		echo "Tomcat运行中，1秒后关闭！"
		echo  1;sleep 1 
		echo "即将关闭Tomcat服务，请稍等！" 
        $CATALINA_HOME/bin/catalina.sh stop ;echo "已经执行关闭命令,正在检查关闭了多少Tomcat进程，请稍等30秒钟！"
		sleep 27
        echo "3";sleep 1;echo "2";sleep 1;echo "1";sleep 1
		pkill java && pkill tomcat
		if  netstat -an | grep 8080 | grep LISTEN >/dev/null;then
			PID=`ps -ef  | grep  -v grep  | grep java | awk  '{print $2}'`
			NUM=`ps -ef | grep  -v "color"  | grep tomcat | awk '{print $2}' | wc -l`
			kill -9 $PID ;echo "已成功关闭${NUM} 个tomcat进程"
		else
  			echo  "Tomcat 已经关闭完成！" 
        	echo "3";sleep 1;echo "2";sleep 1;echo "1";sleep 1 
		fi
	else
		echo "Tomcat 没有运行"
		echo 1
	fi
	if  netstat -an | grep 8080 | grep LISTEN >/dev/null;then
            PID=`ps -ef  | grep  -v grep  | grep java | awk  '{print $2}'`
            #NUM=`ps -ef | grep  -v "color"  | grep tomcat | awk '{print $2}' | wc -l`
            echo "关闭失败，即将强制删除tomcat进程!"
            sleep 2
            pkill tomcat ;sleep 2 
            if  netstat -an | grep 8080 | grep LISTEN >/dev/null;then
                echo "强制关闭失败，即将再次强制删除tomcat进程!"
                pkill java; sleep 2
            fi
	fi
	}
restart() {
	stop 
	start 
 }

case "$1" in 
start) 
start 
;; 

stop) 
stop 
;; 

restart) 
restart 
;; 

*) 
echo $"Usage: $0 {start|stop|restart|status}" 
esac
```