## MySQL
### 1. MyISAM 与 InnoDB区别
					MyISAM                  Innodb
		事物支持：   不支持					支持
		锁的粒度：   table					Row
		存储容量：   没有上限               	64TB
		哈希索引：   不支持                 	支持
		全文索引：   支持						不支持
		外键：      不支持					支持
	
### 2. Mysql 语句优化, 慢的原因查找
	a. 查看正在进行的SQL
		mysql> show processlist;
		+----+------+-----------------+------+---------+------+-------+------------------+
		| Id | User | Host            | db   | Command | Time | State | Info             |
		+----+------+-----------------+------+---------+------+-------+------------------+
		|  4 | root | localhost:51784 | NULL | Query   |    0 | init  | show processlist |
		+----+------+-----------------+------+---------+------+-------+------------------+
		1 row in set (0.00 sec)
	
	当MySQL繁忙的时候运行show processlist，会发现有很多行输出，每行输出对应一个MySQL连接。怎么诊断发起连接的进程是哪个？它当前正在干嘛呢？

	首先，需要通过TCP Socket而不是Unix Socket连接MySQL，这样在show processlist的输出中就会有来源端口号。如下，
		mysql> show processlist;
		+——–+——–+—————–+——+———+——+——-+——————+
		| Id | User | Host | db | Command | Time | State | Info |
		+——–+——–+—————–+——+———+——+——-+——————+
		| 277801 | mydbuser | localhost:35558 | mydb | Sleep | 1 | | NULL |
		| 277804 | mydbuser | localhost:35561 | mydb | Sleep | 1 | | NULL |
		| 277805 | mydbuser | localhost:35562 | mydb | Sleep | 0 | | NULL |
		+——–+——–+—————–+——+———+——+——-+——————+
		
		在Host列有来源IP和端口号，然后我们从连接机器查看端口号是谁打开的，
		
		[root@localhost ~]# netstat -ntp | grep 35558
		… 124.115.0.68:35558 ESTABLISHED 18783/httpd
		
		可知进程18783发起的MySQL连接来源端口是35558，然后就可以用strace观察这个进程了。如果是Apache的PHP脚本，还可以 用proctitle模块( http://pecl.php.net/package/proctitle/ )设置脚本的状态信息。
		
		lsof也能根据端口号显示进程号，细节请参考手册。
		
		

### 3. Mysql 索引命中/如何不命中
### 4. 数据库设计要则
	a. 选择正确的引擎
		a.1.是否需要支持事务
		a.2.是否需要外键
		a.3.是否需要全文索引
		a.4.经常使用什么样的查询模式（大量的查，还是大量的写）
		a.5.数据量的大小，大体量的话，建议
		经验上来说，小应用或项目使用MyISAM就足够了，如果你正在计划使用一个超大数据量的项目，而且需要事务处理或外键支持，那么你真的应该直接使用InnoDB方式。但需要记住InnoDB 的表需要更多的内存和存储，转换100GB 的MyISAM 表到InnoDB 表可能会让你有非常坏的体验。
	b. 固定长度的表（static表 或 静态 表）会更快
		何为固定长度？简单来讲就是表中的所有字段都是定长的，
		如表中有varchar，text, blob 这些字段都不是固定长度的表
	c. 垂直分表，把数据库中的表按列变成几张表，小表，可以降低表的复杂度。如把常查询的列，变成静态表。
	d. 越小的列，速度会越快。
		对于大多数的数据库引擎来说，硬盘操作可能是最重大的瓶颈。
		所以，把你的数据变得紧凑会对这种情况非常有帮助，因为这减少了对硬盘的访问。
		如果一个表只会有几列罢了（比如说字典表，配置表），那么，我们就没有理由使用 INT 来做主键，
		使用 MEDIUMINT, SMALLINT 或是更小的 TINYINT 会更经济一些。
		如果你不需要记录时间，使用 DATE 要比 DATETIME 好得多。
	
## PHP
### 1. PHP数组性能高--内部都是链表（双向链表，可快速查找）
### 2. PHP 检测email === 正则 和 内置函数
### 3. Web 安全 SQL 注入，XSS , 跨站攻击
### 4. 写一个方法快速翻转一个数组，并说明思路
	思路是：以中间为对折，然后左右两边替换
	function reverse(&$a){
        $len = count($a);
        $first = 0;
        $last = $len -1;
        while($last > $first){
                $tmp = $a[$first];
                $a[$first] = $a[$last];
                $a[$last] = $tmp;
                $last --;
                $first ++;
        }
        return $a;
	｝
### 5. 有两个数组A和B, 都是有序的，如何以最快的方法判定B是A的子集？

## Redis
### 1. 数据类型
### 2. 使用场景
### 3. 优化

## Linux
### 1. 统计日志文件数
### 2. 查看系统调用
>lsof -p pid

	[root@fcuh ~]# lsof -p 2154
	COMMAND  PID  USER   FD   TYPE DEVICE SIZE/OFF    NODE NAME
	php-fpm 2154 nginx  cwd    DIR  253,0     4096       2 /
	php-fpm 2154 nginx  rtd    DIR  253,0     4096       2 /
	php-fpm 2154 nginx  txt    REG  253,0  4033872 2367037 /usr/sbin/php-fpm
	php-fpm 2154 nginx  DEL    REG    0,4            10872 /dev/zero
	php-fpm 2154 nginx  mem    REG  253,0    65928 3145770 /lib64/libnss_files-2.12.so
	php-fpm 2154 nginx  DEL    REG    0,4                0 /SYSV00000000
	php-fpm 2154 nginx  mem    REG  253,0   116192 2366953 /usr/lib64/php/modules/zip.so
	php-fpm 2154 nginx  mem    REG  253,0    14288 3145818 /lib64/libgpg-error.so.0.5.0
	php-fpm 2154 nginx  mem    REG  253,0   478496 3145831 /lib64/libgcrypt.so.11.5.3
	php-fpm 2154 nginx  mem    REG  253,0   248072 2363976 /usr/lib64/libxslt.so.1.1.26
	php-fpm 2154 nginx  mem    REG  253,0    81688 2363973 /usr/lib64/libexslt.so.0.8.15
	php-fpm 2154 nginx  mem    REG  253,0    33928 2367033 /usr/lib64/php/modules/xsl.so

>说明：

	lsof输出各列信息的意义如下：
	COMMAND：进程的名称
	PID：进程标识符
	PPID：父进程标识符（需要指定-R参数）
	USER：进程所有者
	PGID：进程所属组
	FD：文件描述符，应用程序通过文件描述符识别该文件。如cwd、txt等
	（1）cwd：表示current work dirctory，即：应用程序的当前工作目录，这是该应用程序启动的目录，除非它本身对这个目录进行更改
	（2）txt ：该类型的文件是程序代码，如应用程序二进制文件本身或共享库，如上列表中显示的 /sbin/init 程序
	（3）lnn：library references (AIX);
	（4）er：FD information error (see NAME column);
	（5）jld：jail directory (FreeBSD);
	（6）ltx：shared library text (code and data);
	（7）mxx ：hex memory-mapped type number xx.
	（8）m86：DOS Merge mapped file;
	（9）mem：memory-mapped file;
	（10）mmap：memory-mapped device;
	（11）pd：parent directory;
	（12）rtd：root directory;
	（13）tr：kernel trace file (OpenBSD);
	（14）v86  VP/ix mapped file;
	（15）0：表示标准输出
	（16）1：表示标准输入
	（17）2：表示标准错误
	一般在标准输出、标准错误、标准输入后还跟着文件状态模式：r、w、u等
	（1）u：表示该文件被打开并处于读取/写入模式
	（2）r：表示该文件被打开并处于只读模式
	（3）w：表示该文件被打开并处于
	（4）空格：表示该文件的状态模式为unknow，且没有锁定
	（5）-：表示该文件的状态模式为unknow，且被锁定
	同时在文件状态模式后面，还跟着相关的锁
	（1）N：for a Solaris NFS lock of unknown type;
	（2）r：for read lock on part of the file;
	（3）R：for a read lock on the entire file;
	（4）w：for a write lock on part of the file;（文件的部分写锁）
	（5）W：for a write lock on the entire file;（整个文件的写锁）
	（6）u：for a read and write lock of any length;
	（7）U：for a lock of unknown type;
	（8）x：for an SCO OpenServer Xenix lock on part      of the file;
	（9）X：for an SCO OpenServer Xenix lock on the      entire file;
	（10）space：if there is no lock.
	TYPE：文件类型，如DIR、REG等，常见的文件类型
	（1）DIR：表示目录
	（2）CHR：表示字符类型
	（3）BLK：块设备类型
	（4）UNIX： UNIX 域套接字
	（5）FIFO：先进先出 (FIFO) 队列
	（6）IPv4：网际协议 (IP) 套接字
	DEVICE：指定磁盘的名称
	SIZE：文件的大小
	NODE：索引节点（文件在磁盘上的标识）
	NAME：打开文件的确切名称
 
### 3. 查看系统负载
### 4. 查看网络连接 waitout 状态
### 5. 自动代码部署 rsync 的使用
### 6. 高可用系统架设
### 7. 高并发解决方案

## MQ
### 1. 使用场景
### 2. 配置
						
