**1.使用yum安装**  

默认安装路径：

```shell
#!/bin/bash
#Description:rpm离线安装mysql脚本

echo 获取安装的shell脚本路径
install_shell_path=$(cd $(dirname $(readlink -f "${BASH_SOURCE[0]}"))&&pwd)
echo shell脚本路径:${install_shell_path}
conf_path=/etc/erc-config.properties
source ${conf_path}
setup_log_home="${setup_home}setup.log"

echo 解压mysql-rpm安装包 2>&1|tee -a ${setup_log_home}
mysql_rpm_path=${install_shell_path}/MySQL 
mkdir ${mysql_rpm_path}
tar xvf $1 -C ${mysql_rpm_path} 2>&1|tee -a ${setup_log_home}

echo 先卸载使用rpm方式安装的mariadb和mysql 2>&1|tee -a ${setup_log_home}
service mysql stop
#pkill -f mysql
random_passwd_file=/root/.mysql_secret
if [[ -a $random_passwd_file ]];then
    echo 清空历史随机密码 2>&1|tee -a ${setup_log_home}
    echo '' > $random_passwd_file 2>&1|tee -a ${setup_log_home}
fi
rpm -qa|grep -ie mariadb -ie mysql | awk '{print "开始卸载:"$1;system("rpm -ev --nodeps " $1)}' 2>&1|tee -a ${setup_log_home}
find / -name mysql | awk '{print "开始清除mysql目录:"$1;system("rm -rf " $1)}' 2>&1|tee -a ${setup_log_home}
rm -rf /etc/my.cnf 2>&1|tee -a ${setup_log_home}

echo 循环安装mysql-rpm包 2>&1|tee -a ${setup_log_home}
for file in ${mysql_rpm_path}/*.rpm
do 
    if [ -f $file ]
    then 
    echo "------开始安装:${file}------" 2>&1|tee -a ${setup_log_home}
    rpm -ivh ${file}
    echo "------完成安装:${file}------" 2>&1|tee -a ${setup_log_home}
    fi
done
cp ${install_shell_path}/my.cnf /etc/my.cnf
echo 清除解压的mysql-rpm安装包 2>&1|tee -a ${setup_log_home}
rm -rf ${mysql_rpm_path} 2>&1|tee -a ${setup_log_home}
echo "------MySQL安装完毕------" 2>&1|tee -a ${setup_log_home}
echo "------启动MySQL服务------" 2>&1|tee -a ${setup_log_home}

chkconfig mysql on
awk 'BEGIN{system("service mysql start")}'
sleep 5
awk 'BEGIN{system("service mysql status")}'

echo 初始化MySQL配置 2>&1|tee -a ${setup_log_home}
mysql_passwd=$password
db_instance_name=$db_name

random_passwd=$(cat /root/.mysql_secret|sed '/^$/d'|awk '{split($0,a,"[:]");print a[4]}'|sed s/[[:space:]]//g)
mysqladmin -uroot -p$random_passwd password $mysql_passwd
echo "用新密码重新连接mysql服务器" 2>&1|tee -a ${setup_log_home}
hostname=$(hostname)
mysql -uroot -p$mysql_passwd <<EOF
    -- 开启远程连接
    GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '${mysql_passwd}' WITH GRANT OPTION;
    GRANT ALL PRIVILEGES ON *.* TO 'root'@'${hostname}' IDENTIFIED BY '${mysql_passwd}' WITH GRANT OPTION;
    FLUSH PRIVILEGES;
    -- 创建数据库实例
    create database ${db_instance_name};
    show databases;
EOF
echo 重启MySQL服务...... 2>&1|tee -a ${setup_log_home}
service mysql restart 
#echo 清空变量
unset random_passwd
unset mysql_passwd
unset db_instance_name
exit 0
```

**2.使用Docker容器安装**  

