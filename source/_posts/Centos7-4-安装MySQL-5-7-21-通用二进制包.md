---
title: Centos7.4 安装MySQL 5.7.21 (通用二进制包)
date: 2018-02-01 15:41:00
categories: 
- 服务器
tags: 
- mysql
---
## 下载安装包
MySQL 官方下载地址：https://dev.mysql.com/downloads/mysql/
MySQL 5.7官方安装文档：https://dev.mysql.com/doc/refman/5.7/en/binary-installation.html

本文完全按照官方步骤配置安装
<!--more-->
选择Linux - generic 64位安装包
![](http://img.blog.csdn.net/20180117211028862?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvem1sMzcyMQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
MySQL 5.7.21 二进制包下载地址：https://dev.mysql.com//Downloads/MySQL-5.7/mysql-5.7.21-linux-glibc2.12-x86_64.tar.gz

```
wget --no-check-certificate https://dev.mysql.com//Downloads/MySQL-5.7/mysql-5.7.21-linux-glibc2.12-x86_64.tar.gz
```

## 安装依赖包
MySQL依赖于libaio 库。如果这个库没有在本地安装，数据目录初始化和后续的服务器启动步骤将会失败。请使用适当的软件包管理器进行安装。例如，在基于Yum的系统上：
```
shell> yum search libaio  
shell> yum install libaio 
```
注意
SLES 11：从MySQL 5.7.19开始，Linux通用tar包的格式是EL6而不是EL5。以致于MySQL客户端bin / mysql需要libtinfo.so.5。

解决方法是创建软链接，例如64位系统上的`ln -s libncurses.so.5.6 /lib64/libtinfo.so.5`或32 位系统上的`ln -s libncurses.so.5.6 /lib/libtinfo.so.5`。
## 创建一个mysql用户和组

```
shell> groupadd mysql
shell> useradd -r -g mysql -s /bin/false mysql
```
注意
此用户仅用于运行mysql服务，而不是登录，因此使用useradd -r和-s /bin/false命令选项来创建对服务器主机没有登录权限的用户。

## 解压到指定目录

```
shell> tar -zxvf mysql-5.7.21-linux-glibc2.12-x86_64.tar.gz -C /opt
shell> cd /opt
shell> mv mysql-5.7.21-linux-glibc2.12-x86_64 mysql-5.7.21
```
## 配置环境变量

```
echo "export PATH=$PATH:/opt/mysql-5.7.21/bin" >> /etc/profile
```
## 配置数据库目录
数据目录：`/opt/mysql-5.7.21/data`
参数文件my.cnf：`/etc/my.cnf`
错误日志log-error：`/opt/mysql-5.7.21/log/mysql_error.log`
二进制日志log-bin：`/opt/mysql-5.7.21/log/mysql_bin.log`
慢查询日志slow_query_log_file：`/opt/mysql-5.7.21/log/mysql_slow_query.log`
套接字socket文件：`/opt/mysql-5.7.21/run/mysql.sock`
pid文件：`/opt/mysql-5.7.21/run/mysql.pid`
创建目录：

```
shell> mkdir -p /opt/mysql-5.7.21/{data,log,etc,run}
shell> chown -R mysql:mysql /opt/mysql-5.7.21
shell> chmod 750 /opt/mysql-5.7.21/{data,log,etc,run}
```
## 配置my.cnf文件
在/etc/下创建my.cnf文件，加入如下参数，其他参数根据需要配置（以下配置按默认配置设置）

```
shell> touch /etc/my.cnf   
shell> chown mysql:mysql /etc/my.cnf     
```

```
[client]
port = 3306
socket = /opt/mysql-5.7.21/run/mysql.sock

[mysqld]
port = 3306
socket = /opt/mysql-5.7.21/run/mysql.sock
pid_file = /opt/mysql-5.7.21/run/mysql.pid
datadir = /opt/mysql-5.7.21/data
default_storage_engine = InnoDB
max_allowed_packet = 128M
max_connections = 2048
open_files_limit = 65535

skip-name-resolve
lower_case_table_names=1

character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
init_connect='SET NAMES utf8mb4'


innodb_buffer_pool_size = 128M
innodb_log_file_size = 128M
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit = 0


key_buffer_size = 16M

log-error = /opt/mysql-5.7.21/log/mysql_error.log
log-bin = /opt/mysql-5.7.21/log/mysql_bin.log
slow_query_log = 1
slow_query_log_file = /opt/mysql-5.7.21/log/mysql_slow_query.log
long_query_time = 5


tmp_table_size = 16M
max_heap_table_size = 16M
query_cache_type = 0
query_cache_size = 0

server-id=1

```
## 初始化

```
shell> mysqld --initialize --user=mysql --basedir=/opt/mysql-5.7.21 --datadir=/opt/mysql-5.7.21/data
```
此时会生成一个临时密码，可以在mysql_error.log文件找到

```
shell> grep 'temporary password' /opt/mysql-5.7.21/log/mysql_error.log 
```
生成ssl

```
shell> mysql_ssl_rsa_setup --basedir=/opt/mysql-5.7.21 --datadir=/opt/mysql-5.7.21/data/
```
## 配置服务，使用systemctl管理

```
shell> cd /usr/lib/systemd/system
shell> touch mysqld.service 
```
文件内容如下

```
# Copyright (c) 2015, 2016, Oracle and/or its affiliates. All rights reserved.
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; version 2 of the License.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA  02110-1301 USA
#
# systemd service file for MySQL forking server
#

[Unit]
Description=MySQL Server
Documentation=man:mysqld(8)
Documentation=http://dev.mysql.com/doc/refman/en/using-systemd.html
After=network.target
After=syslog.target

[Install]
WantedBy=multi-user.target

[Service]
User=mysql
Group=mysql

Type=forking

PIDFile=/opt/mysql-5.7.21/run/mysql.pid

# Disable service start and stop timeout logic of systemd for mysqld service.
TimeoutSec=0

# Execute pre and post scripts as root
PermissionsStartOnly=true

# Needed to create system tables
#ExecStartPre=/usr/bin/mysqld_pre_systemd

# Start main service
ExecStart=/opt/mysql-5.7.21/bin/mysqld --daemonize --pid-file=/opt/mysql-5.7.21/run/mysql.pid $MYSQLD_OPTS

# Use this to switch malloc implementation
EnvironmentFile=-/etc/sysconfig/mysql

# Sets open_files_limit
LimitNOFILE = 65535

Restart=on-failure

RestartPreventExitStatus=1

PrivateTmp=false
```
让systemctl加载配置服务

```
shell> systemctl daemon-reload
shell> systemctl enable mysqld.service
shell> systemctl is-enabled mysqld
```
## 启动MySQL服务

```
shell> systemctl start mysqld.service 
```
## MySQL用户初始化
重置密码(上一步已经重置过了 这次可以忽略)
删除匿名用户
关闭root用户的远程登录
删除测试数据库

```
shell> /opt/mysql-5.7.21/bin/mysql_secure_installation

Securing the MySQL server deployment.

Enter password for user root:  # 输入初始化mysql时产生的密码，查看/opt/mysql-5.7.21/log/mysql_error.log 文件

The existing password for the user account root has expired. Please set a new password.

New password: # 新密码

Re-enter new password: # 重新输入新密码

VALIDATE PASSWORD PLUGIN can be used to test passwords
and improve security. It checks the strength of password
and allows the users to set only those passwords which are
secure enough. Would you like to setup VALIDATE PASSWORD plugin?

Press y|Y for Yes, any other key for No: Y   # 是否启用密码安全验证插件

There are three levels of password validation policy:

LOW    Length >= 8
MEDIUM Length >= 8, numeric, mixed case, and special characters
STRONG Length >= 8, numeric, mixed case, special characters and dictionary                  file

Please enter 0 = LOW, 1 = MEDIUM and 2 = STRONG: 2
Using existing password for root.

Estimated strength of the password: 100 
Change the password for root ? ((Press y|Y for Yes, any other key for No) : N # 是否修改root密码

 ... skipping.
By default, a MySQL installation has an anonymous user,
allowing anyone to log into MySQL without having to have
a user account created for them. This is intended only for
testing, and to make the installation go a bit smoother.
You should remove them before moving into a production
environment.

Remove anonymous users? (Press y|Y for Yes, any other key for No) : Y # 删除匿名用户
Success.


Normally, root should only be allowed to connect from
'localhost'. This ensures that someone cannot guess at
the root password from the network.

Disallow root login remotely? (Press y|Y for Yes, any other key for No) : Y # 关闭root用户的远程登录
Success.

By default, MySQL comes with a database named 'test' that
anyone can access. This is also intended only for testing,
and should be removed before moving into a production
environment.


Remove test database and access to it? (Press y|Y for Yes, any other key for No) : Y # 删除测试数据库
 - Dropping test database...
Success.

 - Removing privileges on test database...
Success.

Reloading the privilege tables will ensure that all changes
made so far will take effect immediately.

Reload privilege tables now? (Press y|Y for Yes, any other key for No) : Y # 重新加载表
Success.

All done!
```
## 导入时区

```
shell> mysql_tzinfo_to_sql /usr/share/zoneinfo | mysql -u root -p mysql
```
## 验证安装

```
shell> mysqladmin version -u root -p
```


参考
1.https://www.jianshu.com/p/0d628b2f7476
2.https://dev.mysql.com/doc/refman/5.7/en/binary-installation.html