---
layout: post
title: "通过Shell 脚本简化日常Java web开发中繁琐的打包、部署等重复劳动！"
date: 2012-10-28 18:20
keywords: maven,java,tomcat,shell
comments: true
tags: linux,shell
categories: 
- linux
- 技术
---

在平时的开发工作当中，经常会遇到如下一种场景：  

本地需要启动多个tomcat实例，多个tomcat实例通过统一的一个apache web server来管理，并且项目是maven管理的，每次修改了文件要部署的时候，一般会重复做如下事情：

1. maven 打包工程
2. 删除日志文件
3. 启动tomcat实例

这样的工作做多了，真的既不高效，也不符合优秀的程序猿的习惯，所以我一般会将其写一个脚本，每次运行下脚本，让脚本帮我们来完成这件事情，岂不快哉！

Ok,I will show the code !
<!-- more -->
```bash tomcatctl.sh 
   
#!/bin/bash
PROJECTS_NAME="eboss,boss,buyersystem,open"
DEV_PROJECT_HOME="/Users/tiger/develop/project/kariqu"
APPSERVER_HOME="/Users/tiger/develop/appserver"
APP_LOG_HOME="/Users/tiger/develop/appserver/apache/logs"
##1.kill tomcat pid 
appName=$1
#input var is null
if [ -n "$appName" ]; then
   echo "stop appServer $appName"
   #get app pid
   pid=`ps aux  |grep ${appName} |awk '{print $2 " " $11}' |grep java |awk '{print $1}'`
   if [ "$pid" != "" ]; then
      kill -9 $pid
   else
      echo -e "appServer $appName is not yet start !"
   fi
else
  echo "please input appServer name !"
  exit
fi 

###2.complie and package war
echo "comple project and packaging war ..."
current_project_home=$DEV_PROJECT_HOME/`echo $PROJECTS_NAME |awk -F"," '{for(i=1;i<=NF;i++) print  $i}' |grep $appName`
if [ ! -d $current_project_home ]; then
	echo "error,$current_project_home don't exist"
	exit
fi

echo "current project home : $current_project_home"
cd $current_project_home
mvn clean package -Pdev -DskipTests 


###3.clean logs
echo "cleaning logs ..."
cd $APP_LOG_HOME/$appName
rm -rf  *
cd $APPSERVER_HOME/${appName}-tomcat-7.0.26/logs
rm -rf *.log *.txt *.out
echo "clean $logPath success !"

###4.start tomcat
echo "starting tomcat ..."
$APPSERVER_HOME/${appName}-tomcat-7.0.26/bin/startup.sh
sleep 3

###5. check tomcat service 
newpid=`ps aux |grep ${appName} |grep -v "grep" | awk '{print $2}'`
if [ -n "$newpid" ]; then
    echo -e "${appName} tomcat service  success !"
else
    echo "start ${appName} tomcat service failed !" 
fi


```
通过如上的脚本后，启动某个系统的命令如下:`  
./tomcatctl.sh systemname`
