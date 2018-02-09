---
title: nginx服务器安装启动及配置文件详解
date: 2018-01-30 11:47:11
categories: 
- 服务器运维
tags: 
- nginx
---
## 选择稳定nginx版本

centos的yum不提供nginx安装，通过配置官方yum源的方式获取到的也只是源码包。所以我们找到了Nginx官网看下官方提供的安装方式：Nginx源码包下载的官网地址（http://nginx.org/en/download.html）
从官网上提供三个类型的版本，分别是Mainline version、Stable version、Legacy versions
<!--more-->
 - Mainline version：Mainline 是 Nginx 目前主力在做的版本，可以说是开发版
 - Stable version：最新稳定版，生产环境上建议使用的版本
 - Legacy versions：遗留的老版本的稳定版
在这里我们选择Stable 版本的  `nginx-1.12.2.tar.gz`
安装环境是CentOS 6.5 安装过程所**执行的命令需要root权限**，所以我们选择使用root用户安装。
## 安装依赖包和必要的库
### 安装依赖包

```
shell> yum -y install gcc gcc-c++ make libtool
```

这些依赖包是编译所必需的，如果yum上没有的话可以去下载源码来安装
### 安装必要的库
这些库都是nginx中一些功能模块所依赖的，如without-http_gzip_module需要zlib库来构建和运行此模块，http_rewrite_module需要pcre库来构建和运行此模块，http_ssl_module需要OpenSSL库来构建和运行此模块

#### 安装pcre库（版本8.40）

```
shell> cd /opt/software
shell> wget http://ftp.pcre.org/pub/pcre/pcre-8.40.tar.gz
shell> tar -zxvf pcre-8.40.tar.gz
shell> cd pcre-8.40
shell> ./configure --enable-utf8 --enable-unicode-properties
shell> make && make install
```
#### 安装zlib库（版本1.2.11）

```
shell> cd /opt/software
shell> wget http://www.zlib.net/zlib-1.2.11.tar.gz
shell> tar -zxvf zlib-1.2.11.tar.gz
shell> cd zlib-1.2.11
shell> ./configure
shell> make && make install
```
#### 安装OpenSSL（版本1.0.2k）

```
shell> cd /opt/software
shell> wget http://www.openssl.org/source/old/1.0.2/openssl-1.0.2k.tar.gz
shell> tar -zxvf openssl-1.0.2k.tar.gz
shell> cd openssl-1.0.2k
shell> ./config
shell> make && make install
```

## 编译安装Nginx
### nginx编译选项详解

```
shell> cd /opt
shell> wget http://nginx.org/download/nginx-1.12.2.tar.gz
shell> tar -zxvf nginx-1.12.2.tar.gz
shell> cd nginx-1.12.2
shell> ./configure --help
```

可以在编译时./configure --help列出大部分常用模块和编译选项，列出的编译选项中以--without开头的都默认安装，以PATH结尾的需要手动指定依赖库**源码**目录。

> ./configure --help   
> --help                             print this message
> --prefix=PATH                      set installation prefix  
> --sbin-path=PATH                   set nginx binary pathname
> --with-select_module               enable select module   
> --without-select_module            disable select module   
> --with-poll_module                 enable poll module
> ......

下面是编译选项的说明

 - `--prefix=PATH`： 指定nginx的**安装目录**。默认 /usr/local/nginx
 - `--sbin-path=PATH`：设置nginx可执行文件的名称。默认情况下，文件指向 安装目录/sbin/nginx
 - `--conf-path=PATH`：设置nginx.conf配置文件的名称。nginx允许使用不同的配置文件启动，通过在命令行参数中指定它 。默认情况下，文件指向 安装目录/conf/nginx.conf
 - `--pid-path=PATH`：设置存储主进程ID文件nginx.pid的名称。默认情况下，文件指向 安装目录/logs/nginx.pid
 - `--error-log-path=PATH`：设置错误，警告和诊断文件的名称。默认情况下，文件指向 安装目录/logs/error.log
 - `--http-log-path=PATH`：设置HTTP服务器的请求日志文件的名称。默认情况下，文件指向 安装目录/logs/access.log
 - `--lock-path=PATH`：安装文件锁定，防止安装文件被别人利用，或自己误操作。
 - `--user=nginx`：指定程序运行时的非特权用户。安装完成后，可以随时在nginx.conf配置文件更改user。默认用户名为nobody
 - `--group=nginx`：指定程序运行时的非特权用户所在组名称。默认情况下，组名称设置为非root用户的名称。
 - `--with-http_realip_module` 启用`ngx_http_realip_module`支持（这个模块允许从请求标头更改客户端的IP地址值，默认为关）
 - `--with-http_ssl_module` ：启用`ngx_http_ssl_module`支持（使支持https请求，需已安装openssl）
 - `--with-http_stub_status_module` ：启用`ngx_http_stub_status_module`支持（获取nginx自上次启动以来的工作状态）
 - `--with-http_gzip_static_module` ：启用`ngx_http_gzip_module`支持（该模块同--without-http_gzip_module功能一样）
 - `--http-client-body-temp-path=PATH` ：设定http客户端请求临时文件路径
 - `--http-proxy-temp-path=PATH`：设定http代理临时文件路径
 - `--http-fastcgi-temp-path=PATH`：设定http fastcgi临时文件路径
 - `--http-uwsgi-temp-path=PATH`：设定http scgi临时文件路径
 - `--with-pcre`：设置pcre库的源码路径，如果已通过yum方式安装，使用--with-pcre自动找到库文件。使用`--with-pcre=PATH`时，需要从PCRE网站下载pcre库的源码（版本8.4）并解压，剩下的就交给Nginx的./configure和make来完成。perl正则表达式使用在location指令和 `ngx_http_rewrite_module`模块中。
 - `--with-zlib=PATH`：指定 zlib（版本1.2.11）的源码解压目录。在默认就启用的网络传输压缩模块ngx_http_gzip_module时需要使用zlib
 - `--with-http_ssl_module`：使用https协议模块。默认情况下，该模块没有被构建。前提是openssl已安装
 - `--add-module=PATH` ： 添加第三方外部模块，如`nginx-sticky-module-ng`或缓存模块。每次添加新的模块都要重新编译（Tengine可以在新加入module时无需重新编译）

完整编译指令

```
shell> ./configure \
--prefix=/opt/nginx-1.12.2 \
--sbin-path=/opt/nginx-1.12.2/sbin/nginx \
--conf-path=/opt/nginx-1.12.2/conf/nginx.conf \
--pid-path=/opt/nginx-1.12.2/logs/nginx.pid \
--lock-path=/opt/nginx-1.12.2/logs/nginx.lock \
--error-log-path=/opt/nginx-1.12.2/logs/error.log \
--http-log-path=/opt/nginx-1.12.2/logs/access.log \
--user=nginx \
--group=nginx \
--with-poll_module \
--with-http_realip_module \
--with-http_ssl_module \
--with-http_stub_status_module \
--with-http_gzip_static_module \
--with-zlib=/opt/software/zlib-1.2.11 \
--with-pcre=/opt/software/pcre-8.40 \
--with-openssl=/opt/software/openssl-1.0.2k \
--with-pcre-jit
```
### nginx安装

```
shell> make && make install
```

## 启动Nginx

### 检查配置文件是否正确

注意，启动前和修改了配置文件nginx.conf后最好先检查一下修改过的配置文件是否正确，以免重启后Nginx出现错误影响服务器稳定运行。判断Nginx配置是否正确命令如下：
```
shell> /opt/nginx-1.12.2/sbin/nginx -t
```
### 启动

```   	
shell> groupadd nginx   #创建nginx用户组
shell> useradd -r -g nginx -s /bin/false nginx

```
```
shell> /opt/nginx-1.12.2/sbin/nginx        # 默认配置文件 conf/nginx.conf， -c指定其他配置文件
```
启动成功会在logs目录中出现nginx.pid文件，即nginx进程号
若未成功可以查看logs目录下的error.log文件查看报错日志
### 平滑重启nginx

如果更改了配置就要重启Nginx，要先关闭Nginx再打开？不是的，可以向Nginx 发送信号，平滑重启。
```
shell> /opt/nginx-1.12.2/sbin/nginx -s reload
```
或

```
shell> kill -HUP 进程号或进程号文件路径
```
### 停止Nginx

停止操作是通过向nginx进程发送信号来进行的
步骤1：查询nginx主进程号

```
shell> ps -ef | grep nginx
```
步骤2：发送信号
1.正常关闭

```
shell> kill -QUIT 主进程号
```
2.快速停止Nginx：

```
shell> kill -TERM 主进程号
```
3.强制停止Nginx：

```
shell> pkill -9 nginx
```

### Nginx服务自启
把nginx添加到系统服务中，使其可以使用service nginx start/stop/restart等。
创建一个名称为nginx的脚本，添加如下内容，更改里面的nginx路径和配置文件路径，给执行权限。

如果想添加脚本用service启动，必须要脚本里面包含这2行

```
# chkconfig: - 85 15
# description: nginx is a World Wide Web server. It is used to serve
```
以下是写好的脚本
```
#!/bin/sh
# chkconfig: - 85 15
# description: nginx is a World Wide Web server. It is used to serve
# description: Nginx is an HTTP(S) server, HTTP(S) reverse \
#               proxyand IMAP/POP3 proxy server
# processname: nginx
# config:      /etc/nginx/nginx.conf
# config:      /etc/sysconfig/nginx
# pidfile:     /var/run/nginx.pid
# Source function library.
. /etc/rc.d/init.d/functions
# Source networking configuration.
. /etc/sysconfig/network
# Check that networking is up.
[ "$NETWORKING" = "no" ] && exit 0
nginx="/usr/local/nginx-1.12.2/sbin/nginx"
prog=$(basename $nginx)
NGINX_CONF_FILE="/usr/local/nginx-nginx-1.12.2/conf/nginx.conf"
[ -f /etc/sysconfig/nginx ] && . /etc/sysconfig/nginx
lockfile=/var/lock/subsys/nginx
start() {
    [ -x $nginx ] || exit 5
    [ -f $NGINX_CONF_FILE ] || exit 6
    echo -n $"Starting $prog: "
    daemon $nginx -c $NGINX_CONF_FILE
    retval=$?
    echo
    [ $retval -eq 0 ] && touch $lockfile
    return $retval
}
stop() {
    echo -n $"Stopping $prog: "
    killproc $prog -QUIT
    retval=$?
    echo
    [ $retval -eq 0 ] && rm -f $lockfile
    return $retval
killall -9 nginx
}
restart() {
    configtest || return $?
    stop
    sleep 1
    start
}
reload() {
    configtest || return $?
    echo -n $"Reloading $prog: "
    killproc $nginx -HUP
RETVAL=$?
    echo
}
force_reload() {
    restart
}
configtest() {
$nginx -t -c $NGINX_CONF_FILE
}
rh_status() {
    status $prog
}
rh_status_q() {
    rh_status >/dev/null 2>&1
}
case "$1" in
    start)
        rh_status_q && exit0
    $1
        ;;
    stop)
        rh_status_q || exit 0
        $1
        ;;
    restart|configtest)
        $1
        ;;
    reload)
        rh_status_q || exit 7
        $1
        ;;
    force-reload)
        force_reload
        ;;
    status)
        rh_status
        ;;
    condrestart|try-restart)
        rh_status_q || exit 0
            ;;
    *)  
      echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
        exit 2
esac
```

```
shell> cp nginx /etc/init.d/
shell> chmod 755 /etc/init.d/nginx
shell> chkconfig --add nginx
shell> chkconfig -level 35 nginx
```

测试启动

```
shell> service nginx start
shell> service nginx stop
shell> service nginx reload
```

参考：

> http://nginx.org/en/docs/configure.html
> http://www.jianshu.com/p/d5114a2a2052
> http://seanlook.com/2015/05/17/nginx-install-and-config/
> http://www.ttlsa.com/nginx/nginx-configure-descriptions/
