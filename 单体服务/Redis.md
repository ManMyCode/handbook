**1.基于yum安装**  

依赖yum源
- jemalloc-3.6.0-1.el7.x86_64.rpm
- redis40u-4.0.10-1.ius.centos7.x86_64.rpm
默认安装路径：

```shell
#!/bin/bash
#Description:redis安装脚本
Base_Path=$(cd `dirname $0`; pwd)
#Redis_Home=/usr/ywsoft/redis
Redis_Config=/etc/redis.conf
#mkdir $Redis_Home
#Ip=127.0.0.1
redis_password=yuanwang@123

echo 先卸载已安装的redis及依赖
rpm -qa|grep -ie redis -ie jemalloc | awk '{print "开始卸载:"$1;system("rpm -ev --nodeps " $1)}'

echo 再安装redis及依赖
rpm -ivh $Base_Path/jemalloc-3.6.0-1.el7.x86_64.rpm
rpm -ivh $Base_Path/redis40u-4.0.10-1.ius.centos7.x86_64.rpm

echo 初始化redis配置

#sed -i "s/127.0.0.1/${Ip}/g" $Redis_Config

sed -i "s/protected-mode yes/protected-mode no/g" $Redis_Config
#设置redis密码
echo "requirepass ${redis_password}" >> $Redis_Config

systemctl enable redis

#将8080端口加入防火墙
firewall-cmd --zone=public --add-port=6379/tcp --permanent
firewall-cmd --reload 
echo "查看开启端口："
firewall-cmd --zone=public --list-port

#启动redis
systemctl start redis

unset Base_PATH
unset Redis_Config
exit 0
```
**基于Docker容器安装**