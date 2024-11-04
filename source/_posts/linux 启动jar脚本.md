---
title: linux 启动jar脚本
date: 2021-05-07 14:04
tags: linux
categories: 
---

<!--more-->

```shell
#!/bin/bash
# 日志地址
LOG_NAME=/home/local/jar/log/admin.log
# app名称
APP_NAME=/home/local/jar/mall-admin.jar
#脚本菜单项
usage() {
 echo "Usage: sh 脚本名.sh [start|stop|restart|status]"
 exit 1
}

is_exist(){
 pid=`jps -l|grep $APP_NAME|awk '{print $1}' `
 #如果不存在返回1，存在返回0
 if [ -z "${pid}" ]; then
 return 1
 else
 return 0
 fi
}
#启动脚本
start(){
 is_exist
 if [ $? -eq "0" ]; then
 echo "${APP_NAME} is already running. pid=${pid} ."
 else
#此处注意修改jar和log文件文件位置：
source /etc/profile
 nohup java -jar ${APP_NAME} >> ${LOG_NAME}   2>&1 &
 fi
}
#停止脚本
stop(){
 is_exist
 if [ $? -eq "0" ]; then
 kill -9 $pid
 else
 echo "${APP_NAME} is not running"
 fi
}
#显示当前jar运行状态
status(){
 is_exist
 if [ $? -eq "0" ]; then
 echo "${APP_NAME} is running. Pid is ${pid}"
 else
 echo "${APP_NAME} is NOT running."
 fi
}
#重启脚本
restart(){
 stop
 start
 echo "${APP_NAME} restart"
return 0
}

case "$1" in
 "start")
 start
 ;;
 "stop")
 stop
 ;;
 "status")
 status
 ;;
 "restart")
 restart
 ;;
 *)
 usage
 ;;
esac

```