title: MySQL参数优化配置
date: 2015-08-15 16:56:38
categories: database
tags: [mysql,tuning]
---

随着学习的深入，现在越来越觉得数据库是WEB应用中一个很重要的瓶颈，数据量的不断增加，不得不对MYSQL的配置参数做进一步了解，来提高MYSQL在生产环境中的负载能力，刚好看到一篇不错的文章，这里转载过来，原文如下：
<!--more-->

## 目的
通过根据服务器目前状况，修改Mysql的系统参数，达到合理利用服务器现有资源，最大合理的提高MySQL性能。

## 服务器参数
32G内存、4个CPU,每个CPU 8核。

## MySQL目前安装状况
MySQL目前安装，用的是MySQL默认的最大支持配置。拷贝的是my-huge.cnf.编码已修改为UTF-8.具体修改及安装MySQL,可以参考<<Linux系统上安装MySQL 5.5>>帮助文档

## 修改MySQL配置
打开MySQL配置文件my.cnf
	
> vi  /etc/my.cnf

### MySQL非缓存参数变量介绍及修改
---
#### 修改back_log参数值

**由默认的50修改为500.（每个连接256kb,占用：125M）**

back_log值指出在MySQL暂时停止回答新请求之前的短时间内多少个请求可以被存在堆栈中。也就是说，如果MySql的连接数据达到max_connections时，新来的请求将会被存在堆栈中，以等待某一连接释放资源，该堆栈的数量即back_log，如果等待连接的数量超过back_log，将不被授予连接资源。将会报：unauthenticated user | xxx.xxx.xxx.xxx | NULL | Connect | NULL | login | NULL 的待连接进程时.
back_log值不能超过TCP/IP连接的侦听队列的大小。若超过则无效，查看当前系统的TCP/IP连接的侦听队列的大小命令：cat /proc/sys/net/ipv4/tcp_max_syn_backlog目前系统为1024。对于Linux系统推荐设置为小于512的整数。
修改系统内核参数，）http://www.51testing.com/html/64/n-810764.html
查看mysql 当前系统默认back_log值，命令：
show variables like 'back_log'; 查看当前数量

---

#### 修改wait_timeout参数值
**由默认的8小时，修改为30分钟。**
我对wait-timeout这个参数的理解：MySQL客户端的数据库连接闲置最大时间值。
说得比较通俗一点，就是当你的MySQL连接闲置超过一定时间后将会被强行关闭。MySQL默认的wait-timeout  值为8个小时，可以通过命令show variables like 'wait_timeout'查看结果值;。
设置这个值是非常有意义的，比如你的网站有大量的MySQL链接请求（每个MySQL连接都是要内存资源开销的 ），由于你的程序的原因有大量的连接请求空闲啥事也不干，白白占用内存资源，或者导致MySQL超过最大连接数从来无法新建连接导致“Too many connections”的错误。在设置之前你可以查看一下你的MYSQL的状态（可用show processlist)，如果经常发现MYSQL中有大量的Sleep进程，则需要 修改wait-timeout值了。
interactive_timeout：服务器关闭交互式连接前等待活动的秒数。交互式客户端定义为在mysql_real_connect()中使用CLIENT_INTERACTIVE选项的客户端。
wait_timeout:服务器关闭非交互连接之前等待活动的秒数。在线程启动时，根据全局wait_timeout值或全局 interactive_timeout值初始化会话wait_timeout值，取决于客户端类型(由mysql_real_connect()的连接选项CLIENT_INTERACTIVE定义).
这两个参数必须配合使用。否则单独设置wait_timeout无效

---
#### 修改max_connections参数值
**由默认的151，修改为3000（750M）**
max_connections是指MySql的最大连接数，如果服务器的并发连接请求量比较大，建议调高此值，以增加并行连接数量，当然这建立在机器能支撑的情况下，因为如果连接数越多，介于MySql会为每个连接提供连接缓冲区，就会开销越多的内存，所以要适当调整该值，不能盲目提高设值。可以过'conn%'通配符查看当前状态的连接数量，以定夺该值的大小。
MySQL服务器允许的最大连接数16384；
查看系统当前最大连接数：
show variables like 'max_connections';

---
#### 修改max_user_connections值

**由默认的0，修改为800**

max_user_connections是指每个数据库用户的最大连接
针对某一个账号的所有客户端并行连接到MYSQL服务的最大并行连接数。简单说是指同一个账号能够同时连接到mysql服务的最大连接数。设置为0表示不限制。
目前默认值为：0不受限制。
这儿顺便介绍下Max_used_connections:它是指从这次mysql服务启动到现在，同一时刻并行连接数的最大值。它不是指当前的连接情况，而是一个比较值。如果在过去某一个时刻，MYSQL服务同时有1000个请求连接过来，而之后再也没有出现这么大的并发请求时，则Max_used_connections=1000.请注意与show variables 里的max_user_connections的区别。默认为0表示无限大。
查看max_user_connections值
show variables like 'max_user_connections';

---
#### 修改thread_concurrency值
**由目前默认的8，修改为64**
thread_concurrency的值的正确与否, 对mysql的性能影响很大, 在多个cpu(或多核)的情况下，错误设置了thread_concurrency的值, 会导致mysql不能充分利用多cpu(或多核), 出现同一时刻只能一个cpu(或核)在工作的情况。
thread_concurrency应设为CPU核数的2倍. 比如有一个双核的CPU, 那thread_concurrency  的应该为4; 2个双核的cpu, thread_concurrency的值应为8.
比如：根据上面介绍我们目前系统的配置，可知道为4个CPU,每个CPU为8核，按照上面的计算规则，这儿应为:4*8*2=64
查看系统当前thread_concurrency默认配置命令：
 show variables like 'thread_concurrency';

---
#### 添加skip-name-resolve
**默认被注释掉，没有该参数**
skip-name-resolve：禁止MySQL对外部连接进行DNS解析，使用这一选项可以消除MySQL进行DNS解析的时间。但需要注意，如果开启该选项，则所有远程主机连接授权都要使用IP地址方式，否则MySQL将无法正常处理连接请求！

---
#### skip-networking
**默认被注释掉。没有该参数**
开启该选项可以彻底关闭MySQL的TCP/IP连接方式，如果WEB服务器是以远程连接的方式访问MySQL数据库服务器则不要开启该选项！否则将无法正常连接！

---
#### default-storage-engine
**设置创建数据库及表默认存储类型Innodb**
show table status like 'tablename'显示表的当前存储状态值
查看MySQL有哪些存储状态及默认存储状态
 show engines;
创建表并指定存储类型
CREATE TABLE mytable (id int, title char(20)) ENGINE = INNODB;
修改表存储类型：
  Alter table tableName engine =engineName
 
备注：设置完后把以下几个开启：

> 	innodb_data_home_dir = /var/lib/mysql
> 	#innodb_data_file_path = ibdata1:1024M;ibdata2:10M:autoextend
> 	#（要注释掉上面一句，否则会创建一个新的把原来的替换的。）
> 	innodb_log_group_home_dir = /var/lib/mysql
> 	# You can set .._buffer_pool_size up to 50 - 80 %
> 	# of RAM but beware of setting memory usage too high
> 	innodb_buffer_pool_size = 1000M
> 	innodb_additional_mem_pool_size = 20M
> 	# Set .._log_file_size to 25 % of buffer pool size
> 	innodb_log_file_size = 500M
> 	innodb_log_buffer_size = 20M
> 	innodb_flush_log_at_trx_commit = 0
> 	innodb_lock_wait_timeout = 50

设置完后一定记得把MySQL安装目录地址（我们目前是默认安装所以地址/var/lib/mysql/）下的ib_logfile0和ib_logfile1删除掉。否则重启MySQL起动失败。

---
转载链接：
	http://blog.csdn.net/nightelve/article/details/17393631
