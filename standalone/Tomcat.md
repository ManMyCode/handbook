**1.使用tar.gz包安装**

```shell
#!/bin/bash
#Description: tomcat安装脚本
Base_PATH=$(cd `dirname $0`; pwd)

##安装目标位置
CATALINA_HOME=/usr/local/tomcat
JAVA_HOME=/usr/local/java/jdk1.8.0_171
catalina=$CATALINA_HOME/bin/catalina.sh

##指定安装包名称
SRC=apache-tomcat-8.5.24.tar.gz
##安装包解压后的文件夹名
Extract=apache-tomcat-8.5.24
##服务文件
Service=/usr/lib/systemd/system/tomcat.service


##获取tomcat安装路径
if [ -d $CATALINA_HOME ]; then
        rm -rf $CATALINA_HOME
fi
mkdir -p $CATALINA_HOME

echo "正在安装tomcat"
tar -zxvf $Base_PATH/$SRC -C $Base_PATH
mv -f $Base_PATH/$Extract/* $CATALINA_HOME

cp -rf $Base_PATH/tomcat.service $CATALINA_HOME/bin
rm -rf $Base_PATH/$Extract

##删除tomcat里面多余的WEB资源
chmod +x $CATALINA_HOME/bin/*.sh


##环境变量写到catalina.sh中
sed -i 2'i\export JAVA_HOME='$JAVA_HOME'' $catalina
sed -i 3'i\export CATALINA_HOME='$CATALINA_HOME'' $catalina
sed -i 4'i\export CATALINA_BASE='$CATALINA_HOME'' $catalina
sed -i 5'i\export CATALINA_PID='$CATALINA_HOME'/bin/tomcat.pid' $catalina
meminfo=`cat /proc/meminfo |grep MemTotal| awk '{split($0,a," ");print a[2]}'`
if [ $meminfo -gt 5000000 ]; then
	sed -i 6'i\export JAVA_OPTS="-Dfile.encoding=UTF-8 -server -Xms2g -Xmx2g -Xmn600m -XX:SurvivorRatio=3 -XX:PermSize=256m -XX:MaxPermSize=256m -Xss256k -XX:+DisableExplicitGC -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:+UseParNewGC -XX:+UseCMSCompactAtFullCollection -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=50 -XX:LargePageSizeInBytes=128m -XX:+UseFastAccessorMethods"' $catalina
else
	sed -i 6'i\export JAVA_OPTS="-Dfile.encoding=UTF-8 -server -Xms512m -Xmx512m -XX:+UseAdaptiveSizePolicy -XX:PermSize=256m -XX:MaxPermSize=256m -Xss256k -XX:+DisableExplicitGC"' $catalina
fi

echo "创建tomcat服务"
if [ -f $Service ]; then
	rm -rf $Service
fi
ln -s $CATALINA_HOME/bin/tomcat.service $Service
systemctl daemon-reload 
systemctl enable tomcat

#将8080端口加入防火墙(开放tomcat端口操作移到安装包的壳里去处理)
#firewall-cmd --zone=public --add-port=8080/tcp --permanent
#firewall-cmd --reload 
#echo "查看开启端口："
#firewall-cmd --zone=public --list-port

#启动tomcat
systemctl start tomcat


unset catalina
unset CATALINA_HOME
unset Extract
unset Service
unset SRC
unset Base_PATH

exit 0
```  

tomcat.service
```ini
[Unit]
Description=ywvsms
After=network.target

[Service]
Type=forking
WorkingDirectory=/usr/local/tomcat/

ExecStart=/usr/local/tomcat/bin/startup.sh
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/usr/local/tomcat/bin/shutdown.sh
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target

```

**2.使用Docker容器安装**
```shell
```