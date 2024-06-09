# 一、基础环境配置
| 主机名       | IP地址            | 服务           | 系统       |
| --------- | :-------------- | ------------ | -------- |
| php       | 192.168.235.140 | php-8.1.11   | CentOS 7 |
| nginx     | 192.168.235.141 | nginx-1.20.2 | CentOS 7 |
| mysql one | 192.168.235.142 | mysql-5.7.38 | CentOS 7 |
| mysql tow | 192.168.235.143 | mysql-5.7.38 | CentOS 7 |
|           |                 |              |          |
## 相关服务目录说明
nginx根目录：/usr/local/nginx/html/

**使用vm，先对服务器快照**
# 二、安装前配置项
```bash
1、 hostnamectl set-hostname #服务器名字
su
2、 systemctl stop firewalld
3、 systemctl disable firewalld
4、 setenforce 0
第4步可以修改为永久关闭：
vim /etc/selinux/config
SELINUX=enforcing修改为disabled  
```
# 三、nginx部署
## 1、nginx的安装
### 1.1、安装前的工作
```bash
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
```
### 1.2、nginx安装两种方式

####  1.2.1、yum安装

```bash
yum install -y epel-release   （下载epel源）
yum install -y nginx
```

####  1.2.2、编译安装 Nginx 服务
##### 1、安装依赖包
```bash
yum -y install pcre-devel zlib-devel gcc gcc-c++ make wget
yum -y groups mark install 'Development Tools'
 ```
##### 2、创建运行用户
```
useradd -r -M -s /sbin/nologin nginx
 ```
 ##### 3、创建日志存放目录
```bash
mkdir -p /var/log/nginx
chown -R nginx.nginx /var/log/nginx
```
##### 4、下载nginx
```bash
cd /usr/src/
wget http://nginx.org/download/nginx-1.20.2.tar.gz
tar xf nginx-1.20.2.tar.gz 
cd nginx-1.20.2
```
##### 5、编译安装
```bash
1、第一种
 ./configure --prefix=/usr/local/nginx --user=nginx --group=nginx --with-debug --with-http_ssl_module --with-http_realip_module --with-http_image_filter_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_stub_status_module --http-log-path=/var/log/nginx/access.log --error-log-path=/var/log/nginx/error.log
make -j $(grep 'processor' /proc/cpuinfo | wc -l) && make install
2、第二种
 cd nginx-1.12.0/
 ./configure \
--prefix=/usr/local/nginx \
--user=nginx \
 --group=nginx \
--with-http_stub_status_module
make && make install
```
##### 6、优化路径
```bash
ln -s /usr/local/nginx/sbin/nginx /usr/local/sbin/
```
##### 7、添加 Nginx 系统服务
```bash
vim /lib/systemd/system/nginx.service
[Unit]
Description=nginx
After=network.target
[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
[Install]
WantedBy=multi-user.target
 
chmod 754 /lib/systemd/system/nginx.service
systemctl start nginx.service
systemctl enable nginx.service
systemctl  daemon-reload
```
## 2.2、nginx安装后配置
### 1、配置环境变量
```bash
[root@master ~]# echo 'export PATH=/usr/local/nginx/sbin:$PATH' > /etc/profile.d/nginx.sh
[root@master ~]# . /etc/profile.d/nginx.sh
[root@master ~]# nginx
[root@master ~]# ss -anlt
State    Recv-Q   Send-Q     Local Address:Port     Peer Address:Port  Process   
LISTEN   0        128              0.0.0.0:80            0.0.0.0:*               
LISTEN   0        128              0.0.0.0:22            0.0.0.0:*               
LISTEN   0        128                 [::]:22               [::]:*    
```

**服务控制方式，使用nginx命令**
    **-t  //检查配置文件语法**
    **-v  //输出nginx的版本**
    **-c  //指定配置文件的路径**
    **-s  //发送服务控制信号，可选值有{stop|quit|reopen|reload}****

### 2、测试web页面访问
在游览器输入http：//IP地址，测试nginx是否可以访问
nginx部署完成，修改信息

### 3、创建存放网站名称,写入php网页信息
```bash
[root@nginx ~]# rm -rf /usr/local/nginx/html/*
[root@nginx ~]# vim /usr/local/nginx/html/index.php
<?php
        phpinfo();
?>

```
### 4、修改nginx服务的配置
[root@nginx ~]# vi /usr/local/nginx/conf/nginx.conf
在`server`大括号内，修改或添加下列配置信息。
        
除下面提及的需要添加或修改的配置信息外，其他配置保持默认值即可。
        
- 添加或修改`location /`配置信息。
            
```shell
location / {
	index index.php index.html index.htm;#增加index.php
}
```
            
- 添加或修改`location ~ .php$`配置信息。
            
```shell
#添加下列信息，配置Nginx通过fastcgi方式处理您的PHP请求。
   location ~ .php$ {
	    root /usr/local/nginx/html/;   #写入你自己定义的网页根目录。
        fastcgi_pass 127.0.0.1:9000;   #php环境主机的IP。
        fastcgi_index index.php;
        fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;   #Nginx调用fastcgi接口处理PHP请求。
                    }
```

## 2.3、重新启动nginx
```bash
systemctl restart nginx
```

# 四、msyql部署

## 3.1MySQL的安装

### 3.1.1MySQL两种安装方式

#### 1. 二进制格式mysql安装

##### 1、下载二进制格式的mysql软件包
```bash
[root@yxt ~]# cd /usr/src/
[root@yxt src]# wget https://downloads.mysql.com/archives/get/p/23/file/mysql-5.7.38-linux-glibc2.12-x86_64.tar.gz
```
##### 2、创建用户和组
```bash
groupadd -r mysql
useradd -M -s /sbin/nologin -g mysql mysql
```
##### 3、解压软件至/usr/local/
```bash
 tar xf mysql-5.7.38-linux-glibc2.12-x86_64.tar.gz -C /usr/local/
[root@yxt src]# ls /usr/local/
apr                    httpd-2.4.54.tar.gz
apr-1.6.5              include
apr-1.6.5.tar.gz       lib
apr-util               lib64
apr-util-1.6.1         libexec
apr-util-1.6.1.tar.gz  mysql-5.7.38-linux-glibc2.12-x86_64
bin                    sbin
etc                    share
games                  src
httpd-2.4.54
[root@yxt src]# cd /usr/local/
[root@yxt local]# ln -sv mysql-5.7.38-linux-glibc2.12-x86_64/ mysql 
'mysql' -> 'mysql-5.7.38-linux-glibc2.12-x86_64/'
[root@yxt local]# ls
apr                    httpd-2.4.54.tar.gz
apr-1.6.5              include
apr-1.6.5.tar.gz       lib
apr-util               lib64
apr-util-1.6.1         libexec
apr-util-1.6.1.tar.gz  mysql
bin                    mysql-5.7.38-linux-glibc2.12-x86_64
etc                    sbin
games                  share
httpd-2.4.54           src
```

##### 4、修改目录/usr/local/mysql的属主属组
```bash
[root@yxt local]# chown -R mysql:mysql /usr/local/mysql
[root@yxt local]# ll /usr/local/mysql -d
lrwxrwxrwx. 1 mysql mysql 36 Jul 27 12:40 /usr/local/mysql -> mysql-5.7.38-linux-glibc2.12-x86_64/
```

##### 5、添加环境变量
```bash
[root@yxt ~]# echo 'export PATH=/usr/local/mysql/bin:$PATH' > /etc/profile.d/mysql.sh
[root@yxt ~]# source /etc/profile.d/mysql.sh 
[root@yxt ~]# echo $PATH
/usr/local/mysql/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/usr/local/apache/bin/:/root/bin
```

##### 6、头文件
[root@yxt ~]# ln -sv /usr/local/mysql/include/  /usr/include/mysql
'/usr/include/mysql' -> '/usr/local/mysql/include/'

##### 7、库文件
[root@yxt ~]# vim /etc/ld.so.conf.d/mysql.conf
/usr/local/mysql/lib/
[root@yxt ~]# ldconfig      

##### 8、man文档
[root@yxt ~]# vim /etc/man_db.conf    //添加
MANDATORY_MANPATH                       /usr/local/mysql/man

##### 9、建立数据存放目录
[root@yxt ~]# mkdir /opt/data
[root@yxt ~]# chown -R mysql:mysql /opt/data/
##### 10、初始化数据库 (**得到初始密码**)
```bash
[root@yxt ~]# /usr/local/mysql/bin/mysqld --initialize --user=mysql --datadir=/opt/data/
2022-07-27T07:37:05.550738Z 1 [Note] A temporary password is generated for root@localhost: ph3w7r,1))2L
```
K6u&dy2rkj,Z

##### 11、生成配置文件
```bash
[root@yxt ~]# vim /etc/my.cnf 
[mysqld]
basedir = /usr/local/mysql
datadir = /opt/data
socket = /tmp/mysql.sock
port = 3306
pid-file = /opt/data/mysql.pid
user = mysql
skip-name-resolve
sql-mode = STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

##### 12、配置服务启动脚本
```bash
[root@yxt support-files]# cp -a /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
[root@yxt ~]# vim /etc/init.d/mysqld     //查找安装路径和数据存放路径并进行修改
basedir=/usr/local/mysql       //安装路径
datadir=/opt/data              /数据存放路劲
```

##### 13、修改selinux为disabled   （/etc/selinux/conf）
##### 14、启动mysql
```bash
 /etc/init.d/mysqld start
Starting MySQL.Logging to '/opt/data/yxt.err'.
 SUCCESS! 
```
 - **Starting MySQL. ERROR! The server quit without updating PID file (/opt/data/mysql.pid).**
 - 解决办法： [[常见问题#2.Starting MySQL. ERROR! The server quit without updating PID file (/opt/data/mysql.pid).]]
```bash
[root@yxt ~]# ps -ef | grep mysql
root       53498       1  0 15:44 pts/0    00:00:00 /bin/sh /usr/local/mysql/bin/mysqld_safe --datadir=/opt/data --pid-file=/opt/data/mysql.pid
mysql      53698   53498  1 15:44 pts/0    00:00:00 /usr/local/mysql/bin/mysqld --basedir=/usr/local/mysql --datadir=/opt/data --plugin-dir=/usr/local/mysql/lib/plugin --user=mysql --log-error=yxt.err --pid-file=/opt/data/mysql.pid --socket=/tmp/mysql.sock --port=3306
root       53728   53116  0 15:44 pts/0    00:00:00 grep --color=auto mysql
[root@yxt ~]# ss -anlt|grep 3306
LISTEN 0      80                 *:3306             *:*       

```
##### 15、修改密码
##### 16使用临时密码登录
[root@yxt ~]# yum install libncurses*
[root@yxt ~]# /usr/local/mysql/bin/mysql -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.38

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
	owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> set password = password('123456');
Query OK, 0 rows affected, 1 warning (0.01 sec)


##### 17、将mysql加入到systemctl服务控制
[root@yxt ~]# cp /usr/lib/systemd/system/sshd.service /usr/lib/systemd/system/mysqld.service 
[root@yxt ~]# vim /usr/lib/systemd/system/mysqld.service
### 修改为以下内容
```bash
[Unit]
Description=mysql    
After=network.target 
[Service]
Type=forking          //后台运行模式
ExecStart=/usr/local/mysql/support-files/mysql.server start
ExecStop=/usr/local/mysql/support-files/mysql.server stop
ExecReload=/bin/kill -HUP $MAINPID
[Install]
WantedBy=multi-user.target

```
### 重新加载MySQL
```bash
[root@yxt ~]# systemctl  daemon-reload
[root@yxt ~]# systemctl  stop firewalld
[root@yxt ~]# systemctl  disable firewalld
Removed /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
[root@yxt ~]# vim /etc/selinux/config 
SELINUX=disabled
[root@yxt ~]# setenforce 0
[root@yxt ~]# systemctl  start mysqld
[root@yxt ~]# ss -anlt|grep 3306
LISTEN 0      80                 *:3306             *:*      

```
2. mysql配置文件
mysql的配置文件为**/etc/my.cnf**
配置文件查找次序：若在多个配置文件中均有设定，则最后找到的最终生效

/etc/my.cnf --> /etc/mysql/my.cnf --> --default-extra-file=/PATH/TO/CONF_FILE --> ~/.my.cnf

- mysql常用配置文件参数：

```bash
参数	说明
port = 3306	设置监听端口
socket = /tmp/mysql.sock	指定套接字文件位置
basedir = /usr/local/mysql	指定MySQL的安装路径
datadir = /data/mysql	指定MySQL的数据存放路径
pid-file = /data/mysql/mysql.pid	指定进程ID文件存放路径
user = mysql	指定MySQL以什么用户的身份提供服务
skip-name-resolve	禁止MySQL对外部连接进行DNS解析
使用这一选项可以消除MySQL进行DNS解析的时间。
若开启该选项，则所有远程主机连接授权都要使用IP地址方式否则MySQL将无法正常处理连接请求
```

#### 2、编译安装mysqld 服务
```bash
--------编译安装mysqld 服务--------
1.将安装mysql 所需软件包传到/opt目录下
mysql-boost-5.7.44.tar.gz  #其中包含支持c++的运行库
 
2.安装环境依赖包
yum -y install \
gcc \
gcc-c++ \
ncurses \			#字符终端下图形互动功能的动态库
ncurses-devel \			#ncurses开发包
bison \				#语法分析器
cmake				#mysql需要用cmake编译安装
----------------------------------------------------------------------------------------------------------
yum -y install gcc gcc-c++ ncurses ncurses-devel bison cmake openssl-devel
 
3.配置软件模块
tar zxvf mysql-boost-5.7.44.tar.gz
 
cd /opt
cd mysql-5.7.44/
cmake \
-DCMAKE_INSTALL_PREFIX=/usr/local/mysql \		    #指定mysql的安装路径
-DMYSQL_UNIX_ADDR=/usr/local/mysql/mysql.sock \ #指定mysql进程监听套接字文件（数据库连接文件）的存储路径
-DSYSCONFDIR=/etc \                             #指定配置文件的存储路径
-DSYSTEMD_PID_DIR=/usr/local/mysql \            #指定进程文件的存储路径
-DDEFAULT_CHARSET=utf8  \                       #指定默认使用的字符集编码，如 utf8
-DDEFAULT_COLLATION=utf8_general_ci \			      #指定默认使用的字符集校对规则
-DWITH_EXTRA_CHARSETS=all \						          #指定支持其他字符集编码
-DWITH_INNOBASE_STORAGE_ENGINE=1 \              #安装INNOBASE存储引擎
-DWITH_ARCHIVE_STORAGE_ENGINE=1 \               #安装ARCHIVE存储引擎 
-DWITH_BLACKHOLE_STORAGE_ENGINE=1 \             #安装BLACKHOLE存储引擎 
-DWITH_PERFSCHEMA_STORAGE_ENGINE=1 \            #安装FEDERATED存储引擎 
-DMYSQL_DATADIR=/usr/local/mysql/data \         #指定数据库文件的存储路径
-DWITH_BOOST=/usr/local/boost \          #指定boost的路径，若使用mysql-boost集成包安装则-DWITH_BOOST=boost
-DWITH_SYSTEMD=1				#生成便于systemctl管理的文件，生成在/usr/local/mysql/usr/lib/的子目录中
 
存储引擎选项：
MYISAM，MERGE，MEMORY，和CSV引擎是默认编译到服务器中，并不需要明确地安装。
静态编译一个存储引擎到服务器，使用-DWITH_engine_STORAGE_ENGINE= 1
可用的存储引擎值有：ARCHIVE, BLACKHOLE, EXAMPLE, FEDERATED, INNOBASE (InnoDB), PARTITION (partitioning support), 和PERFSCHEMA (Performance Schema)
----------------------------------------------------------------------------------------------------------
cmake \
-DCMAKE_INSTALL_PREFIX=/usr/local/mysql \
-DMYSQL_UNIX_ADDR=/usr/local/mysql/mysql.sock \
-DSYSCONFDIR=/etc \
-DSYSTEMD_PID_DIR=/usr/local/mysql \
-DDEFAULT_CHARSET=utf8  \
-DDEFAULT_COLLATION=utf8_general_ci \
-DWITH_EXTRA_CHARSETS=all \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DMYSQL_DATADIR=/usr/local/mysql/data \
-DWITH_BOOST=boost \
-DWITH_SYSTEMD=1  
 
注意：如果在CMAKE的过程中有报错，当报错解决后，需要把源码目录中的CMakeCache.txt文件删除，然后再重新CMAKE，否则错误依旧
 
4.编译及安装
make && make install
 
5.创建mysql用户
useradd -M -s /sbin/nologin  mysql
 
6.修改mysql 配置文件
vim /etc/my.cnf						#删除原配置项，再重新添加下面内容
[client]									#客户端设置
port = 3306
socket = /usr/local/mysql/mysql.sock			
 
[mysql]										#服务端设置
port = 3306
socket = /usr/local/mysql/mysql.sock
auto-rehash								#开启自动补全功能
 
[mysqld]									#服务全局设置
user = mysql       							#设置管理用户
basedir=/usr/local/mysql					#指定数据库的安装目录
datadir=/usr/local/mysql/data				#指定数据库文件的存储路径
port = 3306									#指定端口
character-set-server=utf8					#设置服务器字符集编码格式为utf8
pid-file = /usr/local/mysql/mysqld.pid		#指定pid 进程文件路径
socket=/usr/local/mysql/mysql.sock			#指定数据库连接文件
bind-address = 0.0.0.0				#设置监听地址，0.0.0.0代表允许所有，如允许多个IP需空格隔开
skip-name-resolve							#禁止域名解析，包括主机名，所以授权的时候要使用 IP 地址
max_connections=4096					#设置mysql的最大连接数
default-storage-engine=INNODB				#指定默认存储引擎是INNODB
max_allowed_packet=32M#设置在网络传输中一次消息传输量的最大值。系统默认值为 1MB，最大值是 1GB，必须设置 1024 的倍数。
server-id = 1				#指定服务ID号##做后端SQL集群的时候，每个server都有自己的编号
 
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES,NO_AUTO_CREATE_USER,NO_AUTO_VALUE_ON_ZERO,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,PIPES_AS_CONCAT,ANSI_QUOTES
 
----------------------------------------------------------------------------------------------------------
sql_mode常用值如下:
NO_ENGINE_SUBSTITUTION
如果需要的存储引擎被禁用或未编译,那么抛出错误。不设置此值时,用默认的存储引擎替代,并抛出一个异常
 
STRICT_TRANS_TABLES
在该模式下,如果一个值不能插入到一个事务表中,则中断当前的操作,对非事务表不做限制
 
NO_AUTO_CREATE_USER
禁止GRANT创建密码为空的用户
 
NO_AUTO_VALUE_ON_ZERO
mysql中的自增长列可以从0开始。默认情况下自增长列是从1开始的，如果你插入值为0的数据会报错
 
NO_ZERO_IN_DATE
不允许日期和月份为零
 
NO_ZERO_DATE
mysql数据库不允许插入零日期,插入零日期会抛出错误而不是警告
 
ERROR_FOR_DIVISION_BY_ZERO
在INSERT或UPDATE过程中，如果数据被零除，则产生错误而非警告。默认情况下数据被零除时MySQL返回NULL
 
PIPES_AS_CONCAT
将"||"视为字符串的连接操作符而非或运算符，这和Oracle数据库是一样的，也和字符串的拼接函数Concat相类似
 
ANSI_QUOTES
启用ANSI_QUOTES后，不能用双引号来引用字符串，因为它被解释为识别符
----------------------------------------------------------------------------------------------------------
[client]
port = 3306
socket=/usr/local/mysql/mysql.sock
 
[mysql]
port = 3306
socket = /usr/local/mysql/mysql.sock
auto-rehash
 
[mysqld]
user = mysql
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data
port = 3306
character-set-server=utf8
pid-file = /usr/local/mysql/mysqld.pid
socket=/usr/local/mysql/mysql.sock
bind-address = 0.0.0.0
skip-name-resolve
max_connections=4096
default-storage-engine=INNODB
max_allowed_packet=32M
server-id = 1
 
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES,NO_AUTO_CREATE_USER,NO_AUTO_VALUE_ON_ZERO,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,PIPES_AS_CONCAT,ANSI_QUOTES
 
7.更改mysql安装目录和配置文件的属主属组
chown -R mysql:mysql /usr/local/mysql/
chown mysql:mysql /etc/my.cnf
 
8.设置路径环境变量
echo 'export PATH=/usr/local/mysql/bin:/usr/local/mysql/lib:$PATH' >> /etc/profile	
source /etc/profile
 
9.初始化数据库
cd /usr/local/mysql/bin/
./mysqld \
--initialize-insecure \				#生成初始化密码为空
--user=mysql \                #指定管理用户
--basedir=/usr/local/mysql \  #指定数据库的安装目录
--datadir=/usr/local/mysql/data		#指定数据库文件的存储路径
----------------------------------------------------------------------------------------------------------
./mysqld \
--initialize-insecure \
--user=mysql \
--basedir=/usr/local/mysql \
--datadir=/usr/local/mysql/data
 
10.添加mysqld系统服务
cp /usr/local/mysql/usr/lib/systemd/system/mysqld.service /usr/lib/systemd/system/		#用于systemctl服务管理
systemctl daemon-reload         #刷新识别     
systemctl start mysqld.service  #开启服务
systemctl enable mysqld         #开机自启动
netstat -anpt | grep 3306       #查看端口
 
11.修改mysql 的登录密码
mysqladmin -u root password "abc123" 	#给root账号设置密码为abc123，原始密码为空
 
12.授权远程登录
mysql -u root -p
mysql -uroot -p密码  ##可以免交互登录 只能本地登录
 
grant all privileges on *.* to 'root'@'%' identified by 'abc123';
##grant 就是授权 all privileges可以享受所有特权  on *.*作用于所有库所有表  
##'root'@'%'中表示%表示作用所有，身份认证即密码是abc123
##表示授予root用户可以在所有终端远程登录，使用的密码是abc123，并对所有数据库和所有表有操作权限
 
##数据库的命令学习
show databases;			#查看当前已有的数据库
create database 库名; #创建新的库名
grant all on 库名.* to 'root'@'主机(网段，ip，localhost，%所有)' identified by 'root在mysql的密码'
flush privileges;   #刷新库的权限
quit; #退出
```
### 设置数据库密码，进入数据库，授权mysql
```bash
[root@21bbe49c32fa mysql-5.6.29]# mysqladmin -uroot -p password
Enter password: 
New password: 
Confirm new password: 

##创建wordpress表
mysql> create database wordpress default charset utf8 COLLATE utf8_general_ci;
Query OK, 1 row affected (0.00 sec)

mysql> grant all privileges on wordpress.* to 'wordpress'@'%' identified by '123456' with grant option;
Query OK, 0 rows affected (0.00 sec)
mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> exit
Bye
```

[root@21bbe49c32fa mysql-5.6.29]# exit
exit

# 五、PHP部署
## 4.1 PHP安装

### 4.1.1安装依赖包
```bash
yum -y install gcc gcc-c++ vim make wget libxml2 libxml2-devel openssl openssl-devel bzip2 bzip2-devel libcurl libcurl-devel libicu-devel libjpeg libjpeg-devel libpng libpng-devel openldap-devel  pcre-devel freetype freetype-devel gmp gmp-devel libmcrypt libmcrypt-devel readline readline-devel libxslt libxslt-devel mhash mhash-devel php-mysqlnd wget
```
### 4.1.2下载PHP
```bash
wget https://www.php.net/distributions/php-8.1.11.tar.gz
```

###  4.1.3解压
```bash
tar -zxvf php-8.1.11.tar.gz -C /usr/src/
```
###  4.1.4编译安装php
```bash
[root@php ~]# yum -y install libsqlite3x-devel libzip-devel http://mirror.centos.org/centos/8-stream/PowerTools/x86_64/os/Packages/oniguruma-devel-6.8.2-2.el8.x86_64.rpm

[root@php ~]# cd /usr/src/php-8.1.11/

[root@php php-8.1.11]# ./configure --prefix=/usr/local/php8   --with-config-file-path=/etc  --enable-fpm  --enable-inline-optimization  --disable-debug  --disable-rpath  --enable-shared  --enable-soap  --with-openssl  --enable-bcmath  --with-iconv  --with-bz2  --enable-calendar  --with-curl  --enable-exif   --enable-ftp  --enable-gd  --with-jpeg  --with-zlib-dir  --with-freetype  --with-gettext  --enable-json  --enable-mbstring  --enable-pdo  --with-mysqli=mysqlnd  --with-pdo-mysql=mysqlnd  --with-readline  --enable-shmop  --enable-simplexml  --enable-sockets  --with-zip  --enable-mysqlnd-compression-support  --with-pear  --enable-pcntl  --enable-posix

[root@php php-8.1.11]# make && make install

```
## 4.2配置PHP
### 1、配置环境变量
```bash
[root@php ~]# echo 'export PATH=/usr/local/php8/bin/:$PATH' > /etc/profile.d/php8.sh
[root@php ~]# source /etc/profile.d/php8.sh 
[root@php ~]# php -v
PHP 8.1.11 (cli) (built: Oct 11 2022 04:03:30) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.1.11, Copyright (c) Zend Technologies

```
### 2、配置php-fpm
#### 1、进入的时php的解压目录
[root@php ~]# cd /usr/src/php-8.1.11/
[root@php php-8.1.11]# cp php.ini-production /etc/php.ini 
cp: overwrite '/etc/php.ini'? y
[root@php php-8.1.11]# cd sapi/
[root@php sapi]# ls
apache2handler  cgi  cli  embed  fpm  fuzzer  litespeed  phpdbg
[root@php sapi]# ls
apache2handler  cgi  cli  embed  fpm  fuzzer  litespeed  phpdbg
[root@php sapi]# cd fpm/
[root@php fpm]# cp init.d.php-fpm /etc/init.d/php-fpm
#### 2、开放文件夹权限
[root@php fpm]# chmod +x /etc/init.d/php-fpm 
[root@php fpm]# service php-fpm status
php-fpm is stopped
[root@php fpm]# cp /usr/local/php8/etc/php-fpm.conf.default  /usr/local/php8/etc/php-fpm.conf
[root@php fpm]# cp /usr/local/php8/etc/php-fpm.d/www.conf.default /usr/local/php8/etc/php-fpm.d/www.conf

### 3、启动php-fpm
[root@php fpm]# service php-fpm start
Starting php-fpm  done

[root@php fpm]# ss -anlt
State    Recv-Q   Send-Q     Local Address:Port     Peer Address:Port  Process   
LISTEN   0        128            127.0.0.1:9000          0.0.0.0:*               
LISTEN   0        128              0.0.0.0:22            0.0.0.0:*               
LISTEN   0        128                 [::]:22               [::]:*    

### 连接nginx和php
#### 创建网站目录
```bash
[root@php ~]# mkdir -p /usr/local/nginx/html
[root@php ~]# cat > /usr/local/nginx/html/index.php << EOF
 <?php
     phpinfo();
 ?>
 EOF

[root@php ~]# vim /usr/local/php8/etc/php-fpm.d/www.conf
//修改
listen = 192.168.160.139:9000
listen.allowed_clients = 192.168.160.132
[root@php ~]# service php-fpm restart

```
# 六、部署wordpress
## nginx节点
### 预先安装工具
```bash
yum install unzip -y
```
### 压缩包上传到/root目录
```bash
[root@nginx ~]# ls
anaconda-ks.cfg  wordpress-4.7.3-zh_CN.zip
 
```
### 解压压缩包
unzip wordpress-4.7.3-zh_CN.zip
[root@nginx ~]# ls
anaconda-ks.cfg  wordpress  wordpress-4.7.3-zh_CN.zip
 
### 将解压后的文件复制到网页根目录
[root@nginx ~]# mv wordpress/* /usr/local/nginx/html
## php节点
### 预先安装工具
```bash
yum install unzip -y
```
压缩包上传到/root目录，并解压
[root@php ~]# unzip wordpress-4.7.3-zh_CN.zip
 
[root@php ~]# ls
anaconda-ks.cfg  wordpress  wordpress-4.7.3-zh_CN.zip
### 将解压后的文件复制到网页根目录
```bash
[root@php ~]# mv wordpress/* /www/
WordPress应用提供了wp-config-sample.php模板文件
将模板文件复制为wp-config.php并修改
[root@nginx ~]# cp /www/wp-config-sample.php /www/wp-config.php
[root@nginx ~]# vi /www/wp-config.php
 
// ** MySQL 设置 - 具体信息来自您正在使用的主机 ** //
/** WordPress数据库的名称 */
define('DB_NAME', 'wordpress');
 
/** MySQL数据库用户名 */
define('DB_USER', 'root');
 
/** MySQL数据库密码 */
define('DB_PASSWORD', '000000');
 
/** MySQL主机 */
define('DB_HOST', '192.168.100.30');      
 
/** 创建数据表时默认的文字编码 */
define('DB_CHARSET', 'utf8');
 
/** 数据库整理类型。如不确定请勿更改 */
define('DB_COLLATE', '');
 
注意：/** MySQL主机 */下面一行配置的是主数据库mysql1的IP地址
按照上述文件修改配置文件，保存并退出
```
 
 
使用scp命令将配置文件传送至php节点的网站根目录下
```bash
scp /www/wp-config.php root@192.168.100.60:/www/
```
php节点查看验证一下，可以看到/www目录下面有了这个文件
[root@php ~]# ls /www | grep wp-conf
wp-config.php
#### 创建WordPress数据库
```bash
[root@mysql1 ~]# mysql -uroot -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 6
Server version: 5.5.68-MariaDB MariaDB Server
 
Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.
 
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
 
MariaDB [(none)]> create database wordpress;
Query OK, 1 row affected (0.02 sec)
 
MariaDB [(none)]> exit
Bye
```

[[部署主从数据库]]
[[前端资源优化]]
