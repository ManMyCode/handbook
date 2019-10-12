**1.使用二进制包安装(单节点)**
```shell
#!/bin/bash
#Description: es安装脚本
BASE_PATH=$(cd `dirname $0`; pwd)
#conf_path=/etc/erc-config.properties
#source ${conf_path}
#需先配置Jdk环境变量
JAVA_HOME=/usr/local/java/jdk1.8.0_171
ElASTIC_HOME=/usr/local/elasticsearch

#指定安装包名称
SRC=elasticsearch-2.4.1.tar.gz
#安装包解压后的文件夹名
Extract=elasticsearch-2.4.1

#解压安装包到指定目录下
tar -zxvf $BASE_PATH/$SRC -C $BASE_PATH
mv -f $BASE_PATH/$Extract/* $ElASTIC_HOME
rm -rf $BASE_PATH/$Extract

#配置文件位置
conf=$ElASTIC_HOME/config/elasticsearch.yml

#修改es配置
echo "cluster.name: es" >> ${conf}
echo "node.name: node1" >> ${conf}
echo "network.host: 127.0.0.1" >> ${conf}
echo "http.port: 9200" >> ${conf}

user=elastic
#create user if not exists  
id $user >& /dev/null  
if [ $? -ne 0 ]  
then  
   useradd $user  
fi 

chown -R elastic:elastic  $ElASTIC_HOME

#解压安装head插件
HEAD_PATH="$ElASTIC_HOME/plugins/head"
mkdir -p $HEAD_PATH
rm -rf $HEAD_PATH/*
unzip $BASE_PATH/elasticsearch-head-master.zip -d $HEAD_PATH
cp -rf $HEAD_PATH/elasticsearch-head-master/*  $HEAD_PATH

#将elasticsearch注册成服务:elasticd文件第4行插入elasticsearch启动脚本路径
sed -i '5a 'ELASTIC_HOME=$ElASTIC_HOME/bin'' $BASE_PATH/elasticd
cp $BASE_PATH/elasticd /etc/init.d/elasticd
chmod 755 /etc/init.d/elasticd
chkconfig --add elasticd
chkconfig elasticd on
service elasticd start 
```

服务启停脚本 `elasticd`
```shell
#!/bin/bash  
# chkconfig: 2345 50 95
# description: elasticd Start Stop Restart  
# processname: elasticd  

case $1 in  
	start)  
		su - elastic -c ''${ELASTIC_HOME}'/elasticsearch -d'
		echo "Starting elasticd server..."
		;;   
	stop)     
		pkill -f elasticsearch
		echo "Stoping elasticd server..."
		;;   
	restart)  
		pkill -f elasticsearch
		echo "Stoping elasticd server..."	
		su - elastic -c ''${ELASTIC_HOME}'/elasticsearch -d' 
		echo "Starting elasticd server..."		
		;;  
	status)
		ps -ef | grep elastic |grep elasticsearch-2.4.1.jar | grep -v grep >>null
                if [ $? -ne 0 ]
                then
                        echo "elasticd service Stoped..."
                else
                        ps -ef | grep elastic | grep elasticsearch-2.4.1.jar | grep -v grep | awk '{print "ElasticSearch pid: "$2}'
                        echo "elasticd service is Running..."
                fi
                ;;

	*)
        echo "Please use start|stop|restart|status as first argument"
        ;;		
esac      
exit 0
```
