title: MySQL缓存变量介绍及修改
date: 2015-08-15 16:56:38
categories: database
tags: [mysql,tuning]
---

数据库属于IO密集型的应用程序，其主职责就是数据的管理及存储工作。而我们知道，从内存中读取一个数据库的时间是微秒级别，而从一块普通硬盘上读取一个 IO是在毫秒级别，二者相差3个数量级。所以，要优化数据库，首先第一步需要优化的就是IO，尽可能将磁盘IO转化为内存IO。本文先从MySQL数据库 IO相关参数(缓存参数)的角度来看看可以通过哪些参数进行IO优化.

<!--more-->

---
#### 全局缓存
启动MySQL时就要分配并且总是存在的全局缓存。目前有：key_buffer_size(默认值：402653184,即384M)、innodb_buffer_pool_size(默认值：134217728即：128M)、innodb_additional_mem_pool_size（默认值：8388608即：8M）、innodb_log_buffer_size(默认值：8388608即：8M)、query_cache_size(默认值：33554432即：32M)等五个。总共：560M.
这些变量值都可以通过命令如：show variables like '变量名';查看到。

---
**key_buffer_size本系统目前为384M,可修改为400M**
key_buffer_size是用于索引块的缓冲区大小，增加它可得到更好处理的索引(对所有读和多重写)，对MyISAM(MySQL表存储的一种类型，可以百度等查看详情)表性能影响最大的一个参数。如果你使它太大，系统将开始换页并且真的变慢了。严格说是它决定了数据库索引处理的速度，尤其是索引读的速度。对于内存在4GB左右的服务器该参数可设置为256M或384M.
怎么才能知道key_buffer_size的设置是否合理呢，一般可以检查状态值Key_read_requests和Key_reads   ，比例key_reads / key_read_requests应该尽可能的低，比如1:100，1:1000 ，1:10000。其值可以用以下命令查得：show status like 'key_read%';
比如查看系统当前key_read和key_read_request值为：
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Key_read_requests | 28535 |
| Key_reads         | 269   |
+-------------------+-------+
可知道有28535个请求，有269个请求在内存中没有找到直接从硬盘读取索引.
未命中缓存的概率为：0.94%=269/28535*100%.  一般未命中概率在0.1之下比较好。目前已远远大于0.1，证明效果不好。若命中率在0.01以下，则建议适当的修改key_buffer_size值。
http://dbahacker.com/mysql/innodb-myisam-compare(InnoDB与MyISAM的六大区别)
http://kb.cnblogs.com/page/99810/（查看存储引擎介绍）
MyISAM、InnoDB、MyISAM Merge引擎、InnoDB、memory(heap)、archive

---
**innodb_buffer_pool_size(默认128M)(1G)**
innodb_buffer_pool_size:主要针对InnoDB表性能影响最大的一个参数。功能与Key_buffer_size一样。InnoDB占用的内存，除innodb_buffer_pool_size用于存储页面缓存数据外，另外正常情况下还有大约8%的开销，主要用在每个缓存页帧的描述、adaptive hash等数据结构，如果不是安全关闭，启动时还要恢复的话，还要另开大约12%的内存用于恢复，两者相加就有差不多21%的开销。假设：12G的innodb_buffer_pool_size，最多的时候InnoDB就可能占用到14.5G的内存。若系统只有16G，而且只运行MySQL，且MySQL只用InnoDB，
那么为MySQL开12G，是最大限度地利用内存了。
另外InnoDB和 MyISAM 存储引擎不同， MyISAM 的 key_buffer_size 只能缓存索引键，而 innodb_buffer_pool_size 却可以缓存数据块和索引键。适当的增加这个参数的大小，可以有效的减少 InnoDB 类型的表的磁盘 I/O 。
当我们操作一个 InnoDB 表的时候，返回的所有数据或者去数据过程中用到的任何一个索引块，都会在这个内存区域中走一遭。
可以通过 (Innodb_buffer_pool_read_requests – Innodb_buffer_pool_reads）除以Innodb_buffer_pool_read_requests 乘以 100% 计算缓存命中率，并根据命中率来调整 innodb_buffer_pool_size 参数大小进行优化。值可以用以下命令查得：show status like 'Innodb_buffer_pool_read%';
比如查看当前系统中系统中
| Innodb_buffer_pool_read_requests      | 1283826 |
| Innodb_buffer_pool_reads              | 519     |
+---------------------------------------+---------+
其命中率99.959%=（1283826-519）/1283826*100%  命中率越高越好。

---
**innodb_additional_mem_pool_size(默认8M)(20M)**
innodb_additional_mem_pool_size 设置了InnoDB存储引擎用来存放数据字典信息以及一些内部数据结构的内存空间大小，所以当我们一个MySQL Instance中的数据库对象非常多的时候，是需要适当调整该参数的大小以确保所有数据都能存放在内存中提高访问效率的。
这个参数大小是否足够还是比较容易知道的，因为当过小的时候，MySQL会记录Warning信息到数据库的error log中，这时候你就知道该调整这个参数大小了。
查看当前系统mysql的error日志  cat  /var/lib/mysql/机器名.error 发现有很多waring警告。所以要调大为20M.
根据MySQL手册，对于2G内存的机器，推荐值是20M。32G内存的 100M.

---
**innodb_log_buffer_size(默认8M)(20M)**
innodb_log_buffer_size  这是InnoDB存储引擎的事务日志所使用的缓冲区。类似于Binlog Buffer，InnoDB在写事务日志的时候，为了提高性能，也是先将信息写入Innofb Log Buffer中，当满足innodb_flush_log_trx_commit参数所设置的相应条件(或者日志缓冲区写满)之后，才会将日志写到文件 (或者同步到磁盘)中。可以通过innodb_log_buffer_size 参数设置其可以使用的最大内存空间。
InnoDB 将日志写入日志磁盘文件前的缓冲大小。理想值为 1M 至 8M。大的日志缓冲允许事务运行时不需要将日志保存入磁盘而只到事务被提交(commit)。 因此，如果有大的事务处理，设置大的日志缓冲可以减少磁盘I/O。 在 my.cnf中以数字格式设置。
默认是8MB，系的如频繁的系统可适当增大至4MB～8MB。当然如上面介绍所说，这个参数实际上还和另外的flush参数相关。一般来说不建议超过32MB
注：innodb_flush_log_trx_commit参数对InnoDB Log的写入性能有非常关键的影响,默认值为1。该参数可以设置为0，1，2，解释如下：
0：log buffer中的数据将以每秒一次的频率写入到log file中，且同时会进行文件系统到磁盘的同步操作，但是每个事务的commit并不会触发任何log buffer 到log file的刷新或者文件系统到磁盘的刷新操作;
1：在每次事务提交的时候将log buffer 中的数据都会写入到log file，同时也会触发文件系统到磁盘的同步;
2：事务提交会触发log buffer到log file的刷新，但并不会触发磁盘文件系统到磁盘的同步。此外，每秒会有一次文件系统到磁盘同步操作。
实际测试发现，该值对插入数据的速度影响非常大，设置为2时插入10000条记录只需要2秒，设置为0时只需要1秒，而设置为1时则需要229秒。因此，MySQL手册也建议尽量将插入操作合并成一个事务，这样可以大幅提高速度。根据MySQL手册，在存在丢失最近部分事务的危险的前提下，可以把该值设为0。

---
**query_cache_size(默认32M)(40M)**
主要用来缓存MySQL中的ResultSet，也就是一条SQL语句执行的结果集，所以仅仅只能针对select语句。当我们打开了 Query Cache功能，MySQL在接受到一条select语句的请求后，如果该语句满足Query Cache的要求(未显式说明不允许使用Query Cache，或者已经显式申明需要使用Query Cache)，MySQL会直接根据预先设定好的HASH算法将接受到的select语句以字符串方式进行hash，然后到Query Cache中直接查找是否已经缓存。也就是说，如果已经在缓存中，该select请求就会直接将数据返回，从而省略了后面所有的步骤(如SQL语句的解析，优化器优化以及向存储引擎请求数据等)，极大的提高性能。根据MySQL用户手册，使用查询缓冲最多可以达到238%的效率。
当然，Query Cache也有一个致命的缺陷，那就是当某个表的数据有任何任何变化，都会导致所有引用了该表的select语句在Query Cache中的缓存数据失效。所以，当我们的数据变化非常频繁的情况下，使用Query Cache可能会得不偿失
Query Cache的使用需要多个参数配合，其中最为关键的是query_cache_size和query_cache_type，前者设置用于缓存 ResultSet的内存大小，后者设置在何场景下使用Query Cache。在以往的经验来看，如果不是用来缓存基本不变的数据的MySQL数据库，query_cache_size一般256MB是一个比较合适的大小。当然，这可以通过计算Query Cache的命中率(Qcache_hits/(Qcache_hits+Qcache_inserts)100))来进行调整。 query_cache_type可以设置为0(OFF)，1(ON)或者2(DEMOND)，分别表示完全不使用query cache，除显式要求不使用query cache(使用sql_no_cache)之外的所有的select都使用query cache，只有显示要求才使用query cache(使用sql_cache)。如果Qcache_lowmem_prunes的值非常大，则表明经常出现缓冲. 如果Qcache_hits的值也非常大，则表明查询缓冲使用非常频繁，此时需要增加缓冲大小；
根据命中率(Qcache_hits/(Qcache_hits+Qcache_inserts)100))进行调整，一般不建议太大，256MB可能已经差不多了，大型的配置型静态数据可适当调大.
可以通过命令：show status like 'Qcache_%';查看目前系统Query catch使用大小
| Qcache_hits             | 1892463  |
| Qcache_inserts          | 35627    |
命中率98.17%=1892463/(1892463 +35627 )*100

---
#### 局部缓存
除了全局缓冲，MySql还会为每个连接发放连接缓冲。个连接到MySQL服务器的线程都需要有自己的缓冲。大概需要立刻分配256K，甚至在线程空闲时，它们使用默认的线程堆栈，网络缓存等。事务开始之后，则需要增加更多的空间。运行较小的查询可能仅给指定的线程增加少量的内存消耗，然而如果对数据表做复杂的操作例如扫描、排序或者需要临时表，则需分配大约read_buffer_size，
sort_buffer_size，read_rnd_buffer_size，tmp_table_size 大小的内存空间. 不过它们只是在需要的时候才分配，并且在那些操作做完之后就释放了。有的是立刻分配成单独的组块。tmp_table_size 可能高达MySQL所能分配给这个操作的最大内存空间了
。注意，这里需要考虑的不只有一点——可能会分配多个同一种类型的缓存，例如用来处理子查询。一些特殊的查询的内存使用量可能更大——如果在MyISAM表上做成批的插入
时需要分配 bulk_insert_buffer_size 大小的内存；执行 ALTER TABLE， OPTIMIZE TABLE， REPAIR TABLE 命令时需要分配 myisam_sort_buffer_size 大小的内存。

---
**read_buffer_size（默认值：2097144即2M）(4M)**
read_buffer_size 是MySql读入缓冲区大小。对表进行顺序扫描的请求将分配一个读入缓冲区，MySql会为它分配一段内存缓冲区。read_buffer_size变量控制这一
缓冲区的大小。如果对表的顺序扫描请求非常频繁，并且你认为频繁扫描进行得太慢，可以通过增加该变量值以及内存缓冲区大小提高其性能.

---
**sort_buffer_size（默认值：2097144即2M）(4M)**
sort_buffer_size是MySql执行排序使用的缓冲大小。如果想要增加ORDER BY的速度，首先看是否可以让MySQL使用索引而不是额外的排序阶段。如果不能，可以尝试增加sort_buffer_size变量的大小

---
**read_rnd_buffer_size(默认值：8388608即8M)(8M)**
ead_rnd_buffer_size 是MySql的随机读缓冲区大小。当按任意顺序读取行时(例如，按照排序顺序)，将分配一个随机读缓存区。进行排序查询时，MySql会首先扫描一遍该缓冲，以避免磁盘搜索，提高查询速度，如果需要排序大量数据，可适当调高该值。但MySql会为每个客户连接发放该缓冲空间，所以应尽量适当设置该值，以避免内存开销过大。

---
**tmp_table_size(默认值：8388608 即：16M)(16M)**
tmp_table_size是MySql的heap （堆积）表缓冲大小。所有联合在一个DML指令内完成，并且大多数联合甚至可以不用临时表即可以完成。大多数临时表是基于内
存的(HEAP)表。具有大的记录长度的临时表 (所有列的长度的和)或包含BLOB列的表存储在硬盘上。如果某个内部heap（堆积）表大小超过tmp_table_size，MySQL可以根据需要自
动将内存中的heap表改为基于硬盘的MyISAM表。还可以通过设置tmp_table_size选项来增加临时表的大小。也就是说，如果调高该值，MySql同时将增加heap表的大小，可达到提高联接查询速度的效果。

---
**record_buffer:（默认值：128K）**
record_buffer每个进行一个顺序扫描的线程为其扫描的每张表分配这个大小的一个缓冲区。如果你做很多顺序扫描，你可能想要增加该值。默认数值是131072(128K)

---
#### 其它缓存
**table_cache(默认值：512)**
TABLE_CACHE(5.1.3及以后版本又名TABLE_OPEN_CACHE)
table_cache指定表高速缓存的大小。每当MySQL访问一个表时，如果在表缓冲区中还有空间，该表就被打开并放入其中，这样可以更快地访问表内容。通过检查峰值时间的状态值Open_tables和Opened_tables，可以决定是否需要增加table_cache的值。如果你发现open_tables等于table_cache，并且opened_tables在不断增长，那么你就需要增加table_cache的值了（上述状态值可以使用SHOW STATUS LIKE ‘Open%tables’获得）。注意，不能盲目地把table_cache设置成很大的值。如果设置得太高，可能会造成文件描述符不足，从而造成性能不稳定或者连接失败。
SHOW STATUS LIKE 'Open%tables';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Open_tables   | 356   |
| Opened_tables | 0     |
+---------------+-------+
2 rows in set (0.00 sec)
open_tables表示当前打开的表缓存数，如果执行flush tables操作，则此系统会关闭一些当前没有使用的表缓存而使得此状态值减小；
opend_tables表示曾经打开的表缓存数，会一直进行累加，如果执行flush tables操作，值不会减小。
在mysql默认安装情况下，table_cache的值在2G内存以下的机器中的值默认时256到512，如果机器有4G内存,则默认这个值 是2048，但这决意味着机器内存越大，这个值应该越大，因为table_cache加大后，使得mysql对SQL响应的速度更快了，不可避免的会产生 更多的死锁（dead lock），这样反而使得数据库整个一套操作慢了下来，严重影响性能。所以平时维护中还是要根据库的实际情况去作出判断，找到最适合你维护的库的 table_cache值。
由于MySQL是多线程的机制,为了提高性能,每个线程都是独自打开自己需要的表的文件描 述符,而不是通过共享已经打开的.针对不同存储引擎处理的方法当然也不一样
在myisam表引擎中,数据文件的描述符 (descriptor)是不共享的,但是索引文件的描述符却是所有线程共享的.Innodb中和使用表空间类型有关,假如是共享表空间那么实际就一个数 据文件,当然占用的数据文件描述符就会比独立表空间少.
mysql手册上给的建议大小 是:table_cache=max_connections*n
n表示查询语句中最大表数, 还需要为临时表和文件保留一些额外的文件描述符。
这个数据遭到很多质疑,table_cache够用就好,检查 Opened_tables值,如果这个值很大,或增长很快那么你就得考虑加大table_cache了.
table_cache：所有线程打开的表的数目。增大该值可以增加mysqld需要的文件描述符的数量。默认值是64.

---
**thread_cache_size (服务器线程缓存)(64)**
默认的thread_cache_size=8，但是看到好多配置的样例里的值一般是32，64，甚至是128，感觉这个参数对优化应该有帮助，于是查了下：
根据调查发现以上服务器线程缓存thread_cache_size没有进行设置，或者设置过小,这个值表示可以重新利用保存在缓存中线程的数量,当断开连接时如果缓存中还有空间,那么客户端的线程将被放到缓存中,如果线程重新被请求，那么请求将从缓存中读取,如果缓存中是空的或者是新的请求，那么这个线程将被重新创建,如果有很多新的线程，增加这个值可以改善系统性能.通过比较 Connections 和 Threads_created 状态的变量，可以看到这个变量的作用。(–>表示要调整的值)根据物理内存设置规则如下：
1G —> 8
2G —> 16
3G —> 32     >3G —> 64
  mysql> show status like 'thread%';
+——————-+——-+
| Variable_name     | Value |
+——————-+——-+
| Threads_cached    | 0     |  <—当前被缓存的空闲线程的数量
| Threads_connected | 1     |  <—正在使用（处于连接状态）的线程
| Threads_created   | 1498  |  <—服务启动以来，创建了多少个线程
| Threads_running   | 1     |  <—正在忙的线程（正在查询数据，传输数据等等操作）
+——————-+——-+
查看开机起来数据库被连接了多少次？
mysql> show status like '%connection%';
+———————-+——-+
| Variable_name        | Value |
+———————-+——-+
| Connections          | 1504  |          –>服务启动以来，历史连接数
| Max_used_connections | 2     |
+———————-+——-+
通过连接线程池的命中率来判断设置值是否合适？命中率超过90%以上,设定合理。
 (Connections -  Threads_created) / Connections * 100 %

---
转载链接：
	http://blog.csdn.net/nightelve/article/details/17393631
