# 环境版本：
本文的示例步骤中，使用的软件版本信息如下所述。当您使用不同的软件版本时，需要根据实际情况自行调整命令和参数配置。
- Nginx版本：Nginx 1.20.1
    
- MySQL版本：MySQL 8.0.36
    
- PHP版本：PHP 8.0.30
    
# 具体部署安装步骤：
## 步骤一：关闭防火墙和SELinux

**重要提示！！！！！！**

为避免因使用管理员权限不当造成不可预期的风险，建议您使用普通用户操作。如果普通用户没有sudo权限，具体操作，请参见[如何为普通用户添加sudo权限](https://help.aliyun.com/zh/ecs/use-cases/manually-build-an-lnmp-environment-on-a-centos-instance#section-0ho-omf-0ct)。

#### 1. 远程连接需要部署LNMP环境的ECS实例。

#### 2. 关闭防火墙。
	
 1. 运行以下命令，查看当前防火墙的状态。
	
```shell    
systemctl status firewalld
```
![[Pasted image 20240411155256.png]]
- 如果防火墙的状态参数是`inactive`，则防火墙为关闭状态，请执行步骤。
            
- 如果防火墙的状态参数是`active`，则防火墙为开启状态，请执行步骤[[#2. 关闭防火墙。]]。
    
2. 关闭防火墙。
				
	- 临时关闭防火墙：
	```shell
	sudo systemctl stop firewalld
	```
	**说明**    临时关闭防火墙后，如果Linux实例重启，则防火墙将会自动开启。
		
	- 永久关闭防火墙：
	
		1. 关闭防火墙。
			```shell
			sudo systemctl stop firewalld
			```
		2. 实例开机时，禁止启动防火墙服务。
			```shell
			sudo systemctl disable firewalld
			```        
		**说明** 如果您想重新开启防火墙，请参见[firewalld官网信息](https://firewalld.org/)。
		
		
3. 关闭SELinux。

    1. 运行以下命令，查看SELinux的当前状态。
        
        ```shell
        getenforce
        ```
        
        - 如果SELinux状态参数是`Disabled`，则SELinux为关闭状态，请执行[[#**步骤二：安装Nginx**]]。
            
        - 如果SELinux状态参数是`Enforcing`，则SELinux为开启状态，请执行步骤3.2。
            
        
    2. 关闭SELinux。
        
        SELinux关闭的方式分为临时关闭和永久关闭，请您根据自身业务需求进行选择。具体操作，请参见[[#开启或关闭SELinux]]。
        

## **步骤二：安装Nginx**

**说明**

本文只提供一个版本的Nginx作为示例，如果您需要安装其他版本的Nginx，请参见[[#常见问题]]。

### 安装依赖包：
```bash
[root@localhost ~]#yum -y install gcc gcc-c++ pcre-devel zlib-devel make
```
[[Linux/服务部署文档/linux服务部署搭建/高可用架构/nginx_apache负载均衡|nginx_apache负载均衡]]
### 配置yum源：
```bash
tee /etc/yum.repo.d/Centos-Nginx.repo << EOF
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
EOF
```
1. 运行以下命令，安装Nginx。
    
    ```shell
    sudo yum -y install nginx
    ```
    
2. 运行以下命令，查看Nginx版本。
    
    ```shell
    nginx -v
    ```
    
    返回结果类似如下所示，表示Nginx安装成功。
    
    ```shell
    nginx version: nginx/1.20.1
    ```
    

## **步骤三：安装并配置MySQL**

#### **安装MySQL**

##### 1. 当ECS实例操作系统为Alibaba Cloud Linux 3，需安装MySQL依赖包。(不是Alibaba Cloud Linux 3直接第二部)
    
```shell
sudo yum install -y compat-openssl10
```
    
2. 运行以下命令，更新YUM源。
    
    ```shell
    sudo rpm -Uvh https://dev.mysql.com/get/mysql80-community-release-el7-11.noarch.rpm
    ```
    
3. 运行以下命令，安装MySQL。
    
    ```shell
    sudo yum -y install mysql-community-server
    ```
    
4. 运行以下命令，查看MySQL版本号。
    
    ```shell
    mysql -V
    ```
    
    返回结果如下所示，表示MySQL安装成功。
    
    ```shell
    mysql  Ver 8.0.36 for Linux on x86_64 (MySQL Community Server - GPL)
    ```
    
5. 运行以下命令，启动MySQL。
    
    ```shell
    sudo systemctl start mysqld
    ```
    
6. 依次运行以下命令，设置开机启动MySQL。
    
    ```shell
    sudo systemctl enable mysqld
    sudo systemctl daemon-reload
    ```
    

#### **配置MySQL**

##### 1. 运行以下命令，查看`/var/log/mysqld.log`文件，获取并记录root用户的初始密码。
    
```shell
sudo grep 'temporary password' /var/log/mysqld.log
```
    
    命令行返回结果如下，其中`ARQTRy3+****`为MySQL的初始密码。在下一步重置root用户密码时，会使用该初始密码。
    
    ```shell
    2021-11-10T07:01:26.595215Z 1 [Note] A temporary password is generated for root@localhost: ARQTRy3+****
    ```
    
##### 2. 运行以下命令，配置MySQL的安全性。
```shell
sudo mysql_secure_installation
```
###### 1. 输入MySQL的初始密码。

**说明**
在输入密码时，系统为了最大限度地保证数据安全，命令行将不做任何回显。您只需要输入正确的密码信息，然后按Enter键即可。
```shell
Securing the MySQL server deployment.
Enter password for user root: #输入上一步获取的root用户初始密码
```
###### 2. 设置MySQL的新密码。
        
```shell
The existing password for the user account root has expired. Please set a new password.
        
New password: #输入新密码。长度为8至30个字符，必须同时包含大小写英文字母、数字和特殊符号。特殊符号包含()` ~!@#$%^&*-+=|{}[]:;‘<>,.?/
        
Re-enter new password: #确认新密码。
The 'validate_password' plugin is installed on the server.
The subsequent steps will run with the existing configuration of the plugin.
Using existing password for root.
        
Estimated strength of the password: 100 #返回结果包含您设置的密码强度。
Change the password for root ? (Press y|Y for Yes, any other key for No) :Y #您需要输入Y以确认使用新密码。
        
#新密码设置完成后，需要再次验证新密码。
New password:#再次输入新密码。
        
Re-enter new password:#再次确认新密码。
        
Estimated strength of the password: 100
Do you wish to continue with the password provided?(Press y|Y for Yes, any other key for No) :Y #您需要输入Y，再次确认使用新密码。
```
###### 3. 输入Y删除匿名用户。
        
```shell
Remove anonymous users? (Press y|Y for Yes, any other key for No) :Y
Success.
```
###### 4. 输入Y禁止使用root用户远程登录MySQL。
        
```shell
Disallow root login remotely? (Press y|Y for Yes, any other key for No) :Y
Success.
```
        
###### 5. 输入Y删除test库以及用户对test库的访问权限。
        
```shell
Remove test database and access to it? (Press y|Y for Yes, any other key for No) :Y
- Dropping test database...
Success.
        
- Removing privileges on test database...
Success.
```
        
###### 6. 输入Y重新加载授权表。
        
```shell
Reload privilege tables now? (Press y|Y for Yes, any other key for No) :Y
Success.
        
All done!
```
密码：    
pXHYAyrxf1&1
更多信息，请参见[MySQL文档](https://dev.mysql.com/doc/refman/5.7/en/mysql-secure-installation.html)。

## **步骤四：安装并配置PHP**

### **安装PHP**

#### 1. 安装PHP。
    
Alibaba Cloud Linux 3/2
    
CentOS 7.x
    
##### 1. 当ECS实例操作系统为Alibaba Cloud Linux 3，需安装MySQL依赖包。
        
```shell
  sudo yum install -y compat-openssl10
```
        
##### 2. 运行以下命令，更新YUM源。    
 ```shell
 yum install epel-release //新增扩展源
 
sudo rpm -Uvh https://mirrors.aliyun.com/remi/enterprise/remi-release-7.rpm
```     
        
##### 3. 运行以下命令，启用PHP 8.0仓库。
        
```shell
# 安装yum-utils
yum install yum-utils createrepo -y

## yum-utils：reposync同步工具
## createrepo：编辑yum库工具

sudo yum-config-manager --enable remi-php80
```    
        
##### 4. 运行以下命令，安装PHP。
        
```shell
sudo yum install -y php php-cli php-fpm php-common php-mysqlnd php-gd php-mbstring
```
        
    
2. 运行以下命令，查看PHP版本。
    
    ```shell
    php -v
    ```
    
    返回结果如下所示，表示安装成功。
    
    ```shell
    PHP 8.0.30 (cli) (built: Aug  3 2023 17:13:08) ( NTS gcc x86_64 )
    Copyright (c) The PHP Group
    Zend Engine v4.0.30, Copyright (c) Zend Technologies           
    ```
    

#### **修改Nginx配置文件以支持PHP**

1. 运行以下命令，备份Nginx配置文件。
    
    ```shell
    sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
    ```
    
2. 修改Nginx配置文件，添加Nginx对PHP的支持。
    
    **重要**
    
    若不添加此配置信息，后续您使用浏览器访问PHP页面时，页面将无法显示。
    
    1. 运行以下命令，打开Nginx配置文件。
        
        ```shell
        sudo vim /etc/nginx/conf.d/default.conf
        ```
        
    2. 按`i`进入编辑模式。
        
    3. 在`server`大括号内，修改或添加下列配置信息。
        
        除下面提及的需要添加或修改的配置信息外，其他配置保持默认值即可。
        
        - 添加或修改`location /`配置信息。
            
            ```shell
                    location / {
                        index index.php index.html index.htm;
                    }
            ```
            
        - 添加或修改`location ~ .php$`配置信息。
            
            ```shell
                    #添加下列信息，配置Nginx通过fastcgi方式处理您的PHP请求。
                    location ~ .php$ {
                        root /usr/share/nginx/html;    #将/usr/share/nginx/html替换为您的网站根目录，本文使用/usr/share/nginx/html作为网站根目录。
                        fastcgi_pass 127.0.0.1:9000;   #Nginx通过本机的9000端口将PHP请求转发给PHP-FPM进行处理。
                        fastcgi_index index.php;
                        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
                        include fastcgi_params;   #Nginx调用fastcgi接口处理PHP请求。
                    }
            ```
            
        
        添加或修改配置信息后，文件内容如下图所示：![nginx配置文件](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/1100356361/p350073.png)
        
    4. 按`Esc`键，输入`:wq`，按`Enter`键关闭并保存配置文件。
        
3. 运行以下命令，启动Nginx服务。
    
    ```shell
    sudo systemctl start nginx 
    ```
    
4. 运行以下命令，设置Nginx服务开机自启动。
    
    ```shell
    sudo systemctl enable nginx
    ```
    

#### **配置PHP**

1. 新建并编辑`phpinfo.php`文件，用于展示PHP信息。
    
    1. 运行以下命令，新建`phpinfo.php`文件。
        
        ```shell
        sudo vim <网站根目录>/phpinfo.php
        ```
        
        _<网站根目录>_是您在`nginx.conf`配置文件中`location ~ .php$`大括号内，配置的`root`参数值，如下图所示。![网站根目录](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/2100356361/p350096.png)本文配置的网站根目录为`/usr/share/nginx/html`，因此需要运行以下命令新建`phpinfo.php`文件：
        
        ```shell
        sudo vim /usr/share/nginx/html/phpinfo.php
        ```
        
    2. 按`i`进入编辑模式。
        
    3. 输入下列内容，函数`phpinfo()`​会展示PHP的所有配置信息。
        
        ```shell
        <?php echo phpinfo(); ?>
        ```
        
    4. 按`Esc`键后，输入`:wq`并回车，保存关闭配置文件。
        
2. 运行以下命令，启动PHP-FPM。
    
    ```shell
    sudo systemctl start php-fpm
    ```
    
3. 运行以下命令，设置PHP-FPM开机自启动。
    
    ```shell
    sudo systemctl enable php-fpm
    ```
    

## **步骤五：测试访问LNMP配置信息页面**

1. 在本地Windows主机或其他具有公网访问能力的Windows主机中，打开浏览器。
    
2. 在浏览器的地址栏输入`http://<ECS实例公网IP地址>/phpinfo.php`进行访问。
    
    访问结果如下图所示，表示LNMP环境部署成功。
    
    ![phpinfo](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/3783827071/p130169.png)
    

## 后续步骤

测试访问LNMP配置信息页面后，建议您运行以下命令将`phpinfo.php`文件删除，消除数据泄露风险。

```shell
sudo rm -rf <网站根目录>/phpinfo.php
```

其中，_<网站根目录>_需要替换为您在`nginx.conf`中配置的网站根目录。

本文配置的网站根目录为`/usr/share/nginx/html`，因此需要运行以下命令：

```shell
sudo rm -rf /usr/share/nginx/html/phpinfo.php
```

## 常见问题

### 问题一：如何使用其他版本的Nginx服务器？

1. 使用浏览器访问[Nginx开源社区](https://nginx.org/en/download.html)获取对应的Nginx版本的下载链接。
    
    请根据您的个人需求，选择对应的Nginx版本。本章节以Nginx 1.22.1为例。
    
2. 远程连接需要部署LNMP环境的ECS实例。
    
    具体操作，请参见[使用VNC登录实例](https://help.aliyun.com/zh/ecs/user-guide/log-on-to-an-instance-by-using-vnc#concept-sdk-1jx-wdb)。
    
3. 运行以下命令，安装Nginx相关依赖。
    
    ```shell
    sudo yum install -y gcc-c++
    sudo yum install -y pcre pcre-devel
    sudo yum install -y zlib zlib-devel
    sudo yum install -y openssl openssl-devel
    ```
    
4. 运行`wget`命令下载Nginx 1.22.1。
    
    您可以通过Nginx开源社区直接获取对应版本的安装包URL，然后通过`wget URL`的方式将Nginx安装包下载至ECS实例。例如，Nginx 1.22.1的下载命令如下：
    
    ```shell
    sudo wget http://nginx.org/download/nginx-1.22.1.tar.gz
    ```
    
5. 运行以下命令，解压Nginx 1.22.1安装包，然后进入Nginx所在的文件夹。
    
    ```shell
    sudo tar zxvf nginx-1.22.1.tar.gz
    cd nginx-1.22.1
    ```
    
6. 依次运行以下命令，编译源码。
    
    ```shell
    sudo ./configure \
     --user=nobody \
     --group=nobody \
     --prefix=/usr/local/nginx \
     --with-http_stub_status_module \
     --with-http_gzip_static_module \
     --with-http_realip_module \
     --with-http_sub_module \
     --with-http_ssl_module
    ```
    
    ```shell
    sudo make && make install
    ```
    
7. 运行以下命令，进入Nginx的`sbin`目录，然后启动Nginx。
    
    ```shell
    cd /usr/local/nginx/sbin/
    sudo ./nginx
    ```
    
8. 在本地主机中，使用浏览器访问`ECS实例公网IP`。
    
    出现如下图所示的页面，表示Nginx已成功安装并启动。![nginx](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/7825911161/p228426.png)
    

### 问题二：如何为普通用户添加sudo权限？

1. 使用`root`用户远程连接Linux服务器。
    
    具体操作，请参见[使用VNC登录实例](https://help.aliyun.com/zh/ecs/user-guide/log-on-to-an-instance-by-using-vnc#concept-sdk-1jx-wdb)。
    
2. 运行以下命令，新建一个普通用户`test`并设置密码。
    
    ```shell
    useradd test
    passwd test
    ```
    
3. 运行以下命令，为`/etc/sudoers`文件赋权限。
    
    ```shell
    chmod 750 /etc/sudoers
    ```
    
4. 运行以下命令，编辑`/etc/sudoers`文件。
    
    ```shell
    vim /etc/sudoers
    ```
    
    按`i`键进入编辑模式并添加以下配置：
    
    ```shell
    test ALL=(ALL)  NOPASSWD: ALL
    ```
    
    ![sada45](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/3968248761/p578461.png)输入:wq，保存并退出文件。
    
5. 运行以下命令，切换到`test`用户。
    
    ```shell
    su - test
    ```
    
6. 运行以下命令，测试`sudo`权限。
    
    ```shell
    sudo cat /etc/redhat-release
    ```
    
    如果回显信息类似如下所示，表示`sudo`权限已经添加成功。
    
    ```shell
    [test@iZbp1dqulfhozse3jbp**** ~]$ sudo cat /etc/redhat-release
    CentOS Linux release 7.9.2009 (Core)
    ```

# 开启或关闭SELinux

SELinux是Linux内核的安全子系统，通过严格的访问控制机制增强系统安全性。一般情况下，建议开启SELinux来限制进程的权限，防止恶意程序通过提权等方式对系统进行攻击；然而，由于SELinux的严格访问控制机制，可能会导致一些应用程序或服务无法启动，因此在特定情况下（如开发、调试等），需暂时关闭SELinux。

**说明**

关于SElinux的工作原理和详细说明，请参见[什么是SELinux？](https://www.redhat.com/zh/topics/linux/what-is-selinux)。

## 开启SELinux

本教程以CentOS 7.6 64位操作系统为例，介绍如何开启SELinux。

1. 远程连接ECS实例。
    
    关于连接方式的介绍，请参见[连接方式概述](https://help.aliyun.com/zh/ecs/user-guide/connection-methods#concept-tmr-pgx-wdb)。
    
2. 运行以下命令，查看SELinux状态。
    
    ```shell
    sestatus
    ```
    
    若系统返回的参数信息`SELinux status`显示为`disabled`，则表示SELinux已关闭。![image.png](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/5563080071/p741448.png)
    
3. 在ECS实例上运行以下命令，编辑SELinux的`config`文件。
    
    ```shell
    sudo vi /etc/selinux/config
    ```
    
4. 找到`SELINUX=disabled`字段，按`i`进入编辑模式，通过修改该参数来开启SELinux。
    
    ![SELINUX状态](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/8287688761/p589361.png)
    
    您可以根据实际需求自行调整参数来选择启用SELinux的其中一种模式 ：
    
    - 强制执行模式`SELINUX=enforcing`：表示所有违反安全策略的行为都将被禁止。
        
    - 宽容模式`SELINUX=permissive`：表示所有违反安全策略的行为不被禁止，但会在日志中做记录。
        
    
5. 修改完成后，按下键盘`Esc`键，执行命令`:wq`，保存并退出文件。
    
    **重要**
    
    完成修改`config`文件后，需要重启ECS实例使配置生效，但直接重启实例将会出现系统无法启动的错误。因此，在重启之前需要在根目录下新建`autorelabel`文件，以避免出现该问题。
    
6. 执行以下命令，在根目录下新建隐藏文件`autorelabel`。
    
    ```shell
    sudo touch /.autorelabel
    ```
    
7. 运行以下命令，重启ECS实例。
    
    **说明**
    
    实例重启后，SELinux会自动重新标记所有系统文件。
    
    ```shell
    sudo shutdown -r now
    ```
    

## 关闭SELinux

**重要**

关闭SELinux可能会降低系统的安全性，使系统更容易受到潜在的安全漏洞和攻击。在关闭之前，您应该仔细评估潜在的风险，并确保系统中其他安全措施的有效性。因此，在解决问题后，建议尽快重新启用SELinux，以恢复系统的安全性保护。

1. 远程连接ECS实例。
    
    关于连接方式的介绍，请参见[连接方式概述](https://help.aliyun.com/zh/ecs/user-guide/connection-methods#concept-tmr-pgx-wdb)。
    
2. 运行以下命令，查看SELinux状态。
    
    ```shell
    sestatus
    ```
    
    若系统返回的参数信息`SELinux status`显示为`enabled`，则表示SELinux已启动。![更多SELinux信息](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/8287688761/p589367.png)
    
3. 选择临时关闭或者永久关闭SELinux。
    
    临时关闭SELinux
    
    永久关闭SELinux
    
    执行以下命令，临时关闭SELinux。
    
    ```shell
    setenforce 0
    ```





CentOs7安装nginx
卸载nginx
先查看是否启动了 nginx 服务

ps -ef|grep nginx
1
出现这个则 nginx 没启动服务



出现这个则 nginx 启动了服务



如果 nginx 启动了服务，则需要先关闭 nginx 服务 【没启动就略过这一步】

kill 进程id
1


查看所有与 nginx 有关的文件夹

find / -name nginx
1


删除与 nginx 有关的文件夹

rm -rf file /usr/local/nginx*
1
卸载Nginx相关的依赖

yum remove nginx
1
这样就卸载完成了

安装nginx
查看安装nginx所需要的环境

#查看 C++ 环境是否安装（查看版本号）
gcc -v
#查看 zlib 是否安装
cat /usr/lib64/pkgconfig/zlib.pc
#查看 pcre 是否安装（查版本号）
pcre-config --version
1
2
3
4
5
6
配置 nginx 安装所需的环境

#一次安装4个插件
yum -y install gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel

#一次安装如果有问题，可以试一下分开安装（上面命令执行成功了就无需执行以下命令了）
 	#安装 nginx 需要先将官网下载的源码进行编译，编译依赖 gcc 环境
yum install gcc-c++
 	#pcre是一个perl库，包括perl兼容的正则表达式库，nginx的http模块使用pcre来解析正则表达式，所以需要安装pcre库
yum install -y pcre pcre-devel
 	#zlib库提供了很多种压缩和解压缩方式nginx使用zlib对http包的内容进行gzip，所以需要安装
yum install -y zlib zlib-devel
 	#nginx 不仅支持 http 协议，还支持 https（即在ssl协议上传输http），所以需要在 Centos 安装 OpenSSL 库
yum install -y openssl openssl-devel
1
2
3
4
5
6
7
8
9
10
11
12
安装 nginx

方法一：在官网直接下载.tar.gz安装包，然后通过远程工具拉取到 linux 里面【在 /usr/local 里面创建个nginx文件夹，拉进来。（也可以拉到其他地方）】
方法二：使用wget命令下载，确保系统已经安装了wget，如果没有安装，执行 yum install wget 安装。
这里使用方法二进行安装：

进入 usr/local 里面创建 nginx 文件夹，方便后期删除干净

#进入usr下的local目录
cd usr/local
#在local目录下创建 mysql 文件夹
mkdir nginx
#进入nginx目录
cd nginx
1
2
3
4
5
6


通过 wget 下载 nginx 安装包

wget https://nginx.org/download/nginx-1.21.6.tar.gz
1


解压 并进入解压后的目录

#解压
tar xvf nginx-1.21.6.tar.gz
#进入解压后的目录
cd nginx-1.21.6
1
2
3
4




配置（带有https模块）【需要进入解压后的目录】

./configure --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module
1


编译和安装【需要进入解压后的目录】

#编译
make
#安装
make install
1
2
3
4
启动、关闭 nginx 服务

###启动服务
#需要先进入sbin目录下
cd /usr/local/nginx/sbin
#启动nginx服务
./nginx

###关闭服务
#需要先进入sbin目录下
cd /usr/local/nginx/sbin
#关闭nginx服务
./nginx -s stop
1
2
3
4
5
6
7
8
9
10
11




到这里 nginx 就安装完成了

其他命令

####端口号操作
#查询开启的所有端口
firewall-cmd --list-port
#设置80端口开启
firewall-cmd --zone=public --add-port=80/tcp --permanent
#验证80端口是否开启成功 (单个端口查询)
firewall-cmd --zone=public --query-port=80/tcp
#设置80端口关闭
firewall-cmd --zone=public --remove-port=80/tcp --permanent

####防火墙操作
#检查防火墙是否开启
systemctl status firewalld
#开机自启防火墙
systemctl enable firewalld
#开机禁止自启防火墙
systemctl disable firewalld
#启动
systemctl start firewalld
#关闭
systemctl stop firewalld
#重启
firewall-cmd --reload