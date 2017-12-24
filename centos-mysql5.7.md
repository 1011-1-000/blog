title: CentOS7中安装Mysql5.7 
date: 2015-12-01 16:56:38
categories: database
tags: [mysql]
---

前段时间买了阿里的服务器，因为没有什么时间就只是装了个nginx，配了两个tomcat。数据库一直没有装，刚好最近看到mysql5.7的一些新特性，所以干脆就装一下5.7试试。安装过程并不顺利。这里简单记一下：

<!--more-->
失败的就不再写了。全是血泪。

#### 检测一下系统是否安装mysql
	yum list installed | grep mysql

#### 检测系统是否安装Mariadb
CentOS默认会安装Mariadb

	yum list installed | grep Mariadb

#### 如有，请清除
	yum -y remove mysql*
	yum -y remove mariadb*

#### 给系统添加rpm源，并启用mysql5.7

	wget dev.mysql.com/get/mysql-community-release-el6-5.noarch.rpm
	yum localinstall mysql-community-release-el6-5.noarch.rpm
	yum repolist all | grep mysql
	yum-config-manager --disable mysql55-community
	yum-config-manager --disable mysql56-communit
	yum-config-manager --enable mysql57-community-dmr
	yum repolist enabled | grep mysql

#### 安装mysql服务器
	yum install mysql-community-server

#### 启动mysql
	service mysqld start

#### 查看mysql是否自启动，并且设置开机自启动
	chkconfig --list | grep mysqld
	chkconfig mysqld on

#### navicat设置远程连接
	设置bind-address=0.0.0.0
	为root用户授权：
	grant all on *.* to root@"%" identified by '*****';
	flush privileges;
	重启




















