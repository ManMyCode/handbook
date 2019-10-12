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

my.cnf
```ini
# Example MySQL config file for medium systems.
#
# This is for a system with little memory (32M - 64M) where MySQL plays
# an important part, or systems up to 128M where MySQL is used together with
# other programs (such as a web server)
#
# MySQL programs look for option files in a set of
# locations which depend on the deployment platform.
# You can copy this option file to one of those
# locations. For information about these locations, see:
# http://dev.mysql.com/doc/mysql/en/option-files.html
#
# In this file, you can use all long options that a program supports.
# If you want to know which options a program supports, run the program
# with the "--help" option.

# The following options will be passed to all MySQL clients
[client]
#password	= your_password
port		= 3306
socket		= /var/lib/mysql/mysql.sock
default-character-set=utf8

# Here follows entries for some specific programs

# The MySQL server
[mysqld]
port		= 3306
socket		= /var/lib/mysql/mysql.sock
skip-external-locking
default-storage-engine=MyISAM
default-tmp-storage-engine=MyISAM
open_files_limit=102400
group_concat_max_len=4294967295
max_connections=1000
key_buffer_size = 16M
max_allowed_packet = 10M
table_open_cache = 64
sort_buffer_size = 512K
net_buffer_length = 8K
read_buffer_size = 256K
read_rnd_buffer_size = 512K
myisam_sort_buffer_size = 8M
character-set-server=utf8
log_bin_trust_function_creators=1
group_concat_max_len=102400
transaction-isolation=READ-UNCOMMITTED
lower_case_table_names=1

# Don't listen on a TCP/IP port at all. This can be a security enhancement,
# if all processes that need to connect to mysqld run on the same host.
# All interaction with mysqld must be made via Unix sockets or named pipes.
# Note that using this option without enabling named pipes on Windows
# (via the "enable-named-pipe" option) will render mysqld useless!
# 
#skip-networking

# Replication Master Server (default)
# binary logging is required for replication
#log-bin=mysql-bin

# binary logging format - mixed recommended
#binlog_format=mixed

# required unique id between 1 and 2^32 - 1
# defaults to 1 if master-host is not set
# but will not function as a master if omitted
server-id	= 1

# Replication Slave (comment out master section to use this)
#
# To configure this host as a replication slave, you can choose between
# two methods :
#
# 1) Use the CHANGE MASTER TO command (fully described in our manual) -
#    the syntax is:
#
#    CHANGE MASTER TO MASTER_HOST=<host>, MASTER_PORT=<port>,
#    MASTER_USER=<user>, MASTER_PASSWORD=<password> ;
#
#    where you replace <host>, <user>, <password> by quoted strings and
#    <port> by the master's port number (3306 by default).
#
#    Example:
#
#    CHANGE MASTER TO MASTER_HOST='125.564.12.1', MASTER_PORT=3306,
#    MASTER_USER='joe', MASTER_PASSWORD='secret';
#
# OR
#
# 2) Set the variables below. However, in case you choose this method, then
#    start replication for the first time (even unsuccessfully, for example
#    if you mistyped the password in master-password and the slave fails to
#    connect), the slave will create a master.info file, and any later
#    change in this file to the variables' values below will be ignored and
#    overridden by the content of the master.info file, unless you shutdown
#    the slave server, delete master.info and restart the slaver server.
#    For that reason, you may want to leave the lines below untouched
#    (commented) and instead use CHANGE MASTER TO (see above)
#
# required unique id between 2 and 2^32 - 1
# (and different from the master)
# defaults to 2 if master-host is set
# but will not function as a slave if omitted
#server-id       = 2
#
# The replication master for this slave - required
#master-host     =   <hostname>
#
# The username the slave will use for authentication when connecting
# to the master - required
#master-user     =   <username>
#
# The password the slave will authenticate with when connecting to
# the master - required
#master-password =   <password>
#
# The port the master is listening on.
# optional - defaults to 3306
#master-port     =  <port>
#
# binary logging - not required for slaves, but recommended
#log-bin=mysql-bin

# Uncomment the following if you are using InnoDB tables
#innodb_data_home_dir = /var/lib/mysql
#innodb_data_file_path = ibdata1:10M:autoextend
#innodb_log_group_home_dir = /var/lib/mysql
# You can set .._buffer_pool_size up to 50 - 80 %
# of RAM but beware of setting memory usage too high
#innodb_buffer_pool_size = 16M
#innodb_additional_mem_pool_size = 2M
# Set .._log_file_size to 25 % of buffer pool size
#innodb_log_file_size = 5M
#innodb_log_buffer_size = 8M
#innodb_flush_log_at_trx_commit = 1
#innodb_lock_wait_timeout = 50

[mysqldump]
quick
max_allowed_packet = 16M

[mysql]
no-auto-rehash
default-character-set=utf8

# Remove the next comment character if you are not familiar with SQL
#safe-updates

[myisamchk]
key_buffer_size = 20M
sort_buffer_size = 20M
read_buffer = 2M
write_buffer = 2M

[mysqlhotcopy]
interactive-timeout

```

**2.使用Docker容器安装**  

