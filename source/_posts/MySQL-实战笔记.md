---
title: MySQL 实战学习笔记  
tags: MySQL
categories: Tech
date: 2019-01-18 09:48:56  
---


# 01 概览  

## 概览
MySQL分为服务层和存储引擎层，服务层连接器、查询缓存、分析器、优化器、执行器等，包括内置功能函数（日期时间、数学、加密函数等），包括所有跨存储引擎的功能，比如存储过程、触发器、视图等。存储引擎负责数据的存储和提取。

MySQL 逻辑视图：
{% asset_img 01.png %}  
<!-- more -->
# 02 日志系统  

## redo log
> InnoDB特有的日志

WAL：Write-Ahead Logging，先写日志，再写磁盘  

当有一条记录需要更新的时候，InnoDB先把记录写到redo log，并更新内存，次数更新就算完成了。同时，InnoDB在适当的时候，将这个操作记录更新到磁盘，这个更新往往是在系统比较空闲的时候做  

InnoDB的redo log是固定大小的，比如可以配置为一组4个文件，每个文件大小1GB。从头开始写，写到末尾就回到开头循环写。

{% asset_img 02.png %}

write pos是当前记录的位置，一边写一边后移，checkpoint是当前要擦除的位置，也是往后推移并循环的，擦除记录前把记录更新到数据文件  

write pos和checkpoint之间的是空着的部分，可以用来记录新的操作。如果write pos追上了checkpoint，则表示记录满了，不能在执行新的更新，此时需要擦除一些记录  

有了redo log，InnoDB就可以保证即时数据库异常重启，之前提交的记录都不会丢失，这个能力称为crash-safe  

innode_flush_log_at_trx_commit参数，设置成1的时候，表示每次事物的redo log都直接持久化到磁盘

sync_binlog设置成1，表示每次事务的binlog都持久化到磁盘

### redo log的写入机制

redo log存在的三种状态
{% asset_img 25.png %}

- 存在redo log buffer中，物理上实在MySQL的进程内存中
- 写到磁盘(write)，但是没有持久化(fsync), 物理上在文件系统的page cache里面
- 持久化到磁盘，对应hard disk

redo log写入策略控制：innodb_flush_log_at_trx_commit
- 0：每次事务提交时都只把redo log留在redo log buffer
- 1：每次事务提交时都把redo log直接持久化到磁盘（如果设置为1，redo log在prepare阶段就要持久化一次）
- 2：每次事务提交时都把redo log写到page cache

InnoDB有一个后台线程，每隔1s把redo log buffer中的日志写到page buffer，再写到磁盘

事务执行中间过程的redo log也是直接写在redo log buffer的，这些redo log也会被后台线程一起持久化到磁盘

还有两种场景也会让一个没有提交的事务的redo log持久化到磁盘
- redo log buffer 占用的空间即将达到innodb_log_buffer_size一半的时候，后台线程会主动写盘。（只是write，没有fsync，只留在了page buffer）
- 并行的事务提交的时候，顺带将这个事务的redo log buffer持久化到磁盘（innodb_flush_log_at_trx_commit = 1时，另一个事务提交时会写盘，此时会将未提交事务的redo log一并连带写入）

## binlog
> server层自己的日志  

最开始的MySQL并没有INNODB引擎，自带引擎是MyISAM，没有crash-safe能力，binlog日志只用于归档

与redo log的区别
1. redo log是InnoDB特有的，binlog是MySQL的server层实现的，所有引擎都可以使用
2. redo log是物理日志，记录的是“在某个数据页上做了什么修改”，binlog是逻辑日志，记录的是这个语句的原始逻辑，比如“给ID=2这一行的c字段加1”
3. redo log是循环写的，空间固定会用完，binlog是可以追加写的，写到一定大小后会切换到下一个，不会覆盖以前的日志


在更新语句时的INNODB流程：
> update T set c=c+1 where ID=2;

1. 执行器先找引擎取ID=2这一行。先读内存，再度磁盘 
2. 执行器拿到行数据，将值加1，得到新数据，再调用引擎写入新数据
3. 引擎将新数据更新到内存，同时将更新记录记录到redo log，此时redo log为prepare状态。然后告知执行器执行完了，可以提交事务
4. 执行器生成这个操作的binlog，并把binlog写入磁盘
5. 执行器调用引擎的提交事务接口，把redo log状态更新为提交状态，更新完成02

{% asset_img 03.png %}

### binlog的写入机制

事务执行过程中，写到binlog cache，提交的时候再把binlog cache写到binlog文件中

系统给binlog cache分配了一片内存，每个线程一个，如果超过了设置的大小，就要暂存到磁盘

{% asset_img 24.png %}

每个线程有自己的binlog cache，但是共用一个binlog文件

write和fsync的时机，有参数sync_binlog控制
- 0：每次提交事务都只write，不fsync
- 1：每次提交事务都会fsybc
- N(N>1)：每次提交事务都write，但积累N个事务后才fsync

## 事务隔离

### 事务隔离级别

- read uncommitted 读未提交：一个事务还没提交时，它做的变更就能被别的事务看到
- read committed 读提交：一个事务提交后，它做的变更才会被其他事务看到
- repeatable read 可重复读：一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。（未提交变更对其他事物也是不可见的）
- serializable 串行化：对于同一行记录，写会加写锁，读会加读锁，出现读写锁冲突时，后访问的事务必须等前一个事务执行完成才能继续执行

相关参数：transaction-isolation

### 事务隔离的实现

每条记录在更新的时候都会同时记录一条回滚操作。记录上的最新值，通过回滚操作，都可以得到前一个状态的值

{% asset_img 04.png %}

当前值是4，但是在查询的时候，不同时刻启动的事务会有不同的read-view。在视图A B C里，这一个记录的值分别是1 2 4，同一条记录在系统中可以存在多个版本，就是数据库的多版本并发控制（MVCC）。对于read-view A，要得到1，就必须将当前值依次执行图中所有的回滚操作得到。

同时即使现在有另外一个事务正在将4改成5，这个事物跟read-view A B C对应的事务是不会冲突的

回滚日志在不需要的时候会被删除。即当系统里没有比这个回滚日志更早的read-view的时候

### 为什不不建议使用长事务

长事务意味着系统里会存在很老的事务视图。由于这些事务随时可能访问数据库里面的任何数据，所以这个事务提交之前，数据库里面他可能用到的回滚记录都必须保留，这就会导致大量占用存储空间

长事务还会占用锁资源，也可能拖垮整个库

### 事务的启动方式

1 显示启动事务语句，begin或start transaction。配套的提交语句是commit，回滚语句是rollback
2 set autocommit = 0，这个命令会将这个线程的自动提交关掉。意味着如果你只执行一个select语句，这个事务就启动了，而且不会自动提交。这个事务持续存在直到你主动执行commit或rollback语句，或者断开连接

可以在information_schema库的innodb_trx这个表中查询长事务

``` 
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60
```

### 索引

每一个索引在INNODB里面对应一棵B+树

```
mysql> create table T(
id int primary key, 
k int not null, 
name varchar(16),
index (k))engine=InnoDB;
```

对应的索引结构：
{% asset_img 05.png %}

根据叶子节点的内容，索引类型分为主键索引和非主键索引

主键索引的叶子节点存的是整行数据。在INNODB，主键索引也被称为聚簇索引（clustered index）

非主键索引的叶子节点内容是主键的值。在INNODB，非主键索引也被称为二级索引（secondary index）

主键查询方式，只需要搜索ID这棵树

普通索引查询方式，需要先搜索k索引树，得到ID值，再到ID索引树搜索一次，这个过程成为回表

#### insert 流程  
{% asset_img 10.JPG %}

#### 索引选择错误
{% asset_img 11.JPG %}

#### 字符串 前缀索引
{% asset_img 12.JPG %}
{% asset_img 13.JPG %}

### 索引维护

以上面的图为例，如果插入的新的行ID值为700，在只需要在R5的记录后面插入一个新纪录。如果新插入的ID值为400，需要逻辑上挪动后面的数据，空出位置

如果R5所在的数据页已经满了，这个时候需要申请一个新的数据页，然后挪动部分数据过去。这个过程成为页分裂

页分裂除了影响性能，还影响数据页的利用率

当相邻两个页由于删除了数据。原本放在一个页的数据，利用率很低之后，会将数据页做合并。（分裂过程的逆过程）

主键长度越小，普通索引的叶子节点就越小，普通索引占用的空间也就越小。从性能和存储空间方面考量，自增主键往往是更合理的选择

### 覆盖索引

```
mysql> create table T (
ID int primary key,
k int NOT NULL DEFAULT 0, 
s varchar(16) NOT NULL DEFAULT '',
index k(k))
engine=InnoDB;

insert into T values(100,1, 'aa'),(200,2,'bb'),(300,3,'cc'),(500,5,'ee'),(600,6,'ff'),(700,7,'gg');
```

{% asset_img 06.png %}

如果执行的语句是 select * from T where k between 3 and 5

1. 在k索引树上找到k=3的记录，取的ID=300
2. 再到ID索引树上找到ID=300对应的R3
3. 在k索引树取下一个值k=5，取得ID=500
4. 再回到ID索引树查到ID=500对应的R4
5. 在k索引树取下一个值k=6，不满足条件，循环结束

在这个过程中，回到主键索引树搜索的过程，称为回表

如果执行的语句是 select ID from T where k between 3 and 5
这时只需要查ID的值，而ID的值已经在k索引树上了，因此不需要回表

也就是说，索引k已经覆盖了我们的查询需求，我们称为覆盖索引

覆盖索引可以减少树的搜索次数，显著提升查询性能

### 最左前缀原则

可以利用索引的“最左前缀”，来定位记录

{% asset_img 07.jpg %}

当查询所有名字是“张三”的人时，可以快速定位到ID4，然后向后遍历得到所有需要的结果

当查询的是所有名字第一个字是“张”得人（where name like ’张%‘”。这时也可以用到这个索引

在建立联合索引时，如何安排索引的顺序：
- 索引的复用能力：如果通过调整顺序，可以少维护一个索引，那么这个顺序往往就是需要优先考虑采用的

### 索引下推

MySQL 5.6 引入的优化，在索引遍历过程中，对索引包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数

联合索引（name, age）
```
mysql> select * from tuser where name like '张 %' and age=10 and ismale=1;
```

5.6之前：从ID3开始一个个回表，到主键索引上找出数据行，再对比字段值
5.6及之后：在索引遍历过程中，对索引包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数

无索引下推执行流程：
{% asset_img 08.jpg %}

有索引下推执行流程：
{% asset_img 09.jpg %}

## 锁

### 全局锁

对整个数据库实例加锁

Flush tables with read lock（FTWRL）：加全局读锁的方法，阻塞数据更新语句、数据定义语句、更新类事务的提交语句

### 表级锁

表级别的锁有两种：表锁、元数据锁

#### 表锁

lock tables ... read/write，可以用unlock tables主动释放锁，断开连接时自动释放

#### MDL 元数据锁

不需要显示使用，在访问一个表时自动加上

作用是保证读写的正确性，当对一个表做增删改查的时候，加MDL读锁，当对表结构做变更的时候，加MDL写锁

- 读锁之间不互斥，因此你可以有多个线程同时对一张表进行增删改查
- 读写锁之间、写锁之间互斥，保障变更表结构操作的安全性

### 行锁

两段锁协议：INNODB事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放

当出现死锁后，有两种策略:
- 一种策略是，直接进入等待，直到超时
- 另一种策略是，发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行

正常和默认情况下采用第二种，但会带来性能损耗，每个新来的被堵住的线程，都需要发起死锁检测，需要消耗大量CPU资源。可以通过临时关闭检测或控制并发度来控制资源消耗

### 脏页 FLUSH

当内存数据页跟磁盘数据页内容不一致的时候，称这个内存页为“脏页”。内存数据写入到磁盘后，内存和磁盘上的数据页就一致了，成为“干净页”

flush的时机

- redo log写满了
- 系统内存不足，当需要新的内存页，而内存不够的时候，需要淘汰一些数据页，空出内存给别的数据页使用。如果淘汰的是“脏页”，就要先将脏页写到磁盘。这种情况是常态
- MySQL认为系统空闲的时候
- MySQL正常关闭的时候

InnoDB用缓冲池（buffer pool）管理内存，缓冲池中的内存页有三种状态
- 还没有使用的
- 使用了并且是干净页
- 使用了并且是脏页

InnoDB的策略是尽量使用内存，因此对于一个长时间运行的库，未被使用的页面很少

刷脏页虽然是常态，但是出现以下情况，会明显影响性能
- 一个查询要淘汰的脏页个数太多
- 日志写满，更新全部堵住，写性能跌为0

#### InnoDB刷脏页的控制策略

需要正确告诉InnoDB所在主机的IO能力，使用innodb_io_capacity，建议设置成磁盘的IOPS

InnoDB的刷盘速度参考两个因素
- 脏页比例
- redo log写盘速度

InnoDB会根据这两个因素先单独算出两个数字

参数innodb_max_dirty_pages_pct是脏页比例上限，默认值是75%。InnoDB会根据当前的脏页比例M，算出一个范围在0到100之间的数字

InnoDB每次写入的日志都有一个序号，当前写入的序号跟checkpoint对应的序号之间的差值假设为N，InnoDB会根据N算出一个范围在0到100间的数字（N越大，算出来的值越大）

然后，根据f1(M)和f2(N)算的的两个值，取其中较大的值记为R，之后引擎就可以按照innodb_io_capcity定义的能力乘以R%来控制刷脏页的速度

{% asset_img 14.png %}

一旦一个查询请求需要在执行过程中先flush掉一个脏页时，如果这个数据页旁边的数据页刚好是脏页，就会把这个邻居也带着一起刷掉；而且这个逻辑还可以继续蔓延。innodb_flush_neighbors就是用来控制这个行为的，值为1的时候会有上述机制

上述策略在机械硬盘是有意义的，可以减少很多随机IO，如果是SSD建议关闭（8.0默认为0）

### 表数据删除

表数据既可以存在共享表空间里，也可以是单独的文件。由参数innodb_file_per_table控制的，OFF表示表的数据放在系统共享表空间，也就是和数据字典放在一起；ON表示每个InnoDB表数据存储在一个已.ibd为后缀的文件中（默认ON）

建议设置为ON。一个表单独存储更容易管理，而且如果放在共享表空间中，及时表删掉了，空间也是不会回收的

在删除整个表的时候，可以使用drop table回收表空间。但是如果删除某些行，表中的数据被删除了，但是表空间却没有被回收

#### 数据删除流程

假设要删掉一个记录，InnoDB只会把这个记录标记为删除，如果之后在相同位置插入一个新纪录时，可能会复用这个位置。磁盘文件的大小不会缩小

如果删掉了一个数据页上的所有记录，整个数据页就可以被复用了

数据页的复用和记录的复用是不同的

记录的复用，只限于符合范围条件的数据
当整个页从B+树里摘掉之后，可以复用到任何位置

如果相邻的两个数据页利用率都很小，系统就会把这两个页上的数据合到其中一个页上，另外一个数据页就被标记为可复用

如果我们用delete命令把整个表的数据删除，结果就是所有的数据页都会被标记为可复用，但磁盘上文件不会变小

这些可以复用，而没有被使用的空间，看起来就像是“空洞”

不只是删除数据会造成空洞，插入数据也会

如果数据是随机插入的，可能造成索引的数据页分裂

{% asset_img 15.png %}

另外，更新索引上的值，可以理解为删除一个旧的值，再插入一个新的值，也可能会造成空洞

经过大量增删改的表，都是可能存在空洞的。重建表可以把这些空洞去掉，达到收缩表空间的目的

#### 重建表

alter table
新建一个相同结构的表，按照主键ID递增顺序一行一行读出来再插入到新表
在整个DDL过程中，原表不能有更新

{% asset_img 16.png %}

Oneline DDL：
{% asset_img 17.png %}

### COUNT * 

为什么InnoDB不合MyISAM一样，也把数字存起来
由于MVCC的原因，InnoDB表应该返回多少行是不确定的，每一行记录都要判断自己是否对这个会话可见

MySQL优化器会找到最小的索引树来遍历，尽量减少扫描的数据量

show table status 中的 TABLE_ROWS 是通过采样来估算的，误差可能达到40%~50%

### ORDER BY

#### 全字段排序

explain之后，Extra中的“Using filesort”表示的就是需要排序，MySQL会给每个线程分配一块内存用于排序，称为sort buffer

```
select city, name, age from t where city = 'A' order by name limit 1000;
```

执行流程：
1. 初始化sort buffer，确定放入需要查询的字段
2. 从索引找到第一个满足查询条件的id
3. 到主键索引取出整行，取查询字段的值，存入sort buffer
4. 从索引中取下一个记录的id
5. 重复3、4步骤，直到不满足查询条件为止
6. 对sort_buffer中的数据按照排序条件做快速排序
7. 按照排序结果取前N行返回

{% asset_img 18.jpg %}

其中的排序动作，可能在内存中完成，也可能需要使用外部排序，取决于排序所需的内存和参数sort_buffer_size

sort_buffer_size: sort_buffer的大小

#### rowid排序

max_length_for_sort_data: 如果单行的长度超过这个值，MySQL会认为单行太大，需要换一种算法
新的算法放入sort_buffer的字段，只有需要排序的字段和主键id

执行流程：
1. 初始化sort buffer，确定放入排序字段和主键字段
2. 从索引找到第一个满足查询条件的id
3. 到主键索引中取出郑航，取排序字段和主键的值，存入sort buffer
4. 从索引中取下一个记录的id
5. 重复3、4步骤，直到不满足查询条件为止
6. 对sort buffer中的数据按照排序字段排序
7. 遍历排序结果，取前N行，并按照id的值回到原表中取出查询字段返回

{% asset_img 19.jpg %}

rowid排序多访问了一次主键索引



如果可以保证从索引上取出的行，天然是按照排序字段排好序的话，就不需要排序操作了

有时，可以通过建立联合索引解决

{% asset_img 20.png %}

执行流程：
1. 从索引找到第一个满足查询条件的id
2. 到主键索引中根据id取出整行，作为结果集的一部分直接返回
3. 从索引中取下一个主键id
4. 重复2、3步骤，直到查到第N条记录，或者当不满足查询条件时循环结束

{% asset_img 21.jpg %}

此时，explain中的extra为"Using index condition"

可以利用覆盖索引，使过程更加简化
eg 可以创建一个city、name、age的联合索引，这样的话就不需要回表了

{% asset_img 22.jpg %}

此时，explain中的extra为“Using index”


### 幻读

一个事务在前后两次查询同一个范围的时候，后一个查询看到了前一个查询没有看到的行
- 可重复读级别下，普通的查询是快照读，是不会看到别的事务插入的数据的。因此，幻读在“当前读”下才会出现
- 幻读仅专指新插入的行

#### 间隙锁

{% asset_img 23.png %}

跟间隙锁存在冲突关系的，是“往间隙中插入记录”这个操作

间隙锁和行锁合称next-key lock，每个next-key lock都是前开后闭区间
eg (-∞,0]、(0,5]、(5,10]、(10,15]、(15,20]、(20, 25]、(25, +supremum]

间隙锁是在可重复读级别下才会生效的
如果把隔离级别设置为读提交就没有间隙锁了，但同时，需要解决可能出现的数据和日志不一致的情况，需要把binlog格式设置为row

### 加锁规则
隔离级别：可重复读

- 原则1：加锁的基本单位是next-key lock (前开后闭区间)
- 原则2：查找过程中访问到的对象才会加锁
- 优化1：索引上的等值查询，给唯一索引加锁的时候，next-key lock退化为行锁
- 优化2：索引上的等值查询，向右遍历时且最后一个值不满足等值条件时，next-key lock退化为间隙锁
- 唯一索引上的范围查询会访问到不满足条件的第一个值为止

## 主备一致

主备切换：
{% asset_img 29.png %}

建议备库设置成readOnly模式
- 防止备库发生写操作
- 防止切换逻辑有bug，比如切换过程中出现双写，造成主备不一致
- 可以用readonly状态来判断节点的角色

readonly模式对超级权限用户是无效的，所以不影响同步更新线程（拥有超级权限）

内部流程：
{% asset_img 30.png %}

主库接收到客户端的更新请求后，执行内部事务的更新逻辑，同时写binlog
备库和主库之间维持了一个长连接。主库内部有一个线程，专门用于服务备库的这个长连接

事务日志同步的完整过程
1. 在备库上通过change master命令，设置主库的IP、端口、用户名、密码，以及要从那个位置开始请求binlog，这个位置包含文件名和日志偏移量
2. 在备库上执行start slave命令，这时备库会启动两个线程，就是图中的io_thread和sql_thread。其中io_thread负责与主库建立连接
3. 主库校验完用户名、密码后，开始按照备库传过来的位置，从本地读取binlog，发给备库
4. 备库拿到binlog后，写到本地文件，称为中转日志（replay log）
5. sql_thread读取replay log，解析出日志里的命令，并执行

### binlog的三种格式对比

格式：statement, row, mixed

1 statement
{% asset_img 31.png %}

记录了原始的SQL语句

可能产生主备不一致的情况

eg

```
delete from t where a>=4 and t_modified<='2018-11-10' limit 1;
```

主库选择了a索引，从库执行的时候选择了t_modified索引

2 row
{% asset_img 32.png %}

没有SQL语句的原文，替换成了两个event：Table_map（说明接下来操作的库表）和Delete_rows（定义删除的行为）

需要借助mysqlbinlog解析，才能看到详细信息

```  
mysqlbinlog -vv data/master.000001 --start-position=8900
```

{% asset_img 33.png %}

- server id 1: 这个事务是在server_id=1这个库上执行的
- Table_map event，和上面相同。如果操作多张表，每个表都会有一个Table_map event，都会map到一个单独的数字，用于区分对不同表的操作
- 命令中使用了-vv参数（把内容都解析出来），所以从结果里面可以看到各字段的值（@1=4、@2=4）
- Xid enent，表示事务被正确的提交了

当binlog_format使用row格式的时候，binlog里面记录了真实删除行的主键id，这样binlog传到备库去的时候，就肯定会删除id=4的行，不会有主备删除不同行的问题

3 mixed

- row格式的缺点是很占空间，且可能影响执行速度。eg delete 10W行数据，statement格式就是一个SQL语句的记录，row格式需要记录10W条

mixed格式，MySQL自己会判断这条SQL语句是否可能引起主备不一致，如果有可能，就用row格式，否则就用statement格式


现在越来越多的场景要求设置为row格式。好处之一是恢复数据

- delete：row格式会把删掉的整行信息保存下来，可以直接将其转成insert语句，插入回去恢复
- insert：道理同上
- update：会记录修改前郑航的数据和修改后的郑航数据，所以当误删时，将这个event前后的两行信息对调一下，再去执行，就能恢复

### 循环复制问题

双M结构，主备切换流程

{% asset_img 34.png %}

节点A和B之间总是互为主备，这样在切换的时候就不用再修改主备关系

循环复制：
当A更新了一条语句，把生成的binlog发给B，B执行完后也会生成binlog（建议把参数log_slave_updates设置为on，表示备库执行replay log后生成binlog）

如果A同时是B的备库，相当于又把B新生成的binlog拿过来执行了一次，然后A和B之间会不断循环执行这个更新语句

解决：
MySQL在binlog中记录了这个命令第一次执行时所在实例的server id
1. 规定两个库的server id必须不同
2. 一个备库在接到binlog并在重放的过程中，生成与原binlog的server id相同的新的binlog
3. 每个库在收到从自己的主库发过来的日志后，先判断server id，如果跟自己的相同，表示这个日志是自己生成的，就直接丢弃这个日志


## 高可用相关

### 主备延迟

同步延迟，数据同步相关的时间点：

- T1: 主库A执行完一个事务，写入binlog
- T2: 传给备库B，B接受完这个binlog
- T3: 备库B执行完成这个事务

主备延迟的时间：T3 - T1

在备库上执行show slave status可以显示seconds_behind_master

T1的时间往往很短，正常情况下，主备延迟的主要来源是备库接收完binlog和执行完这个事务之间的时间差
所以主备延迟的直接表现是，备库消费中转日志relay log的速度，比主库生产binlog的速度要慢

### 主备延迟的来源

**1 备库所在机器的性能要比主库所在机器的性能差**

**2 备库的压力大**
使用主库克制，但是忽略了备库的压力控制

**3 大事务**
主库上必须等事务执行完才会写入binlog，再传给备库。所以如果主库上的语句执行了10分钟，那这个事务可能就会导致从主库延迟10分钟
典型的大事务场景：
- 一次性的用delete语句删除太多数据
- 大表DDL（计划内的DDL，建议使用gh-ost方案）

**4 备库的并行复制能力**

### 主备切换的不同策略

#### 可靠性优先策略

在双M结构下，切换的过程：
1. 判断备库B现在的seconds_behind_master, 如果小于某个值继续下一步，否则持续重试这一步
2. 把主库A改成只读状态，即把readonly设置为true
3. 判断备库的seconds_behind_master，直到这个值变成0为止
4. 把备库B改成可读写状态，也就是把readonly设置为false
5. 把业务请求切换到备库B

{% asset_img 35.png %}

这个切换流程存在不可用时间（步骤2-5）

#### 可用性优先策略

强行把步骤4和5调整到最开始执行，也就是说不等主备数据同步，直接把连接切到备库B，并且让备库B可以读写，那么系统几乎就没有不可用时间了

可能出现数据不一致的情况

## 备库并行复制能力

主备流程
{% asset_img 36.png %}

如果sql_thread更新数据使用单线程，就会导致备库应用日志不够快，造成主备延迟

多线程模型：
{% asset_img 37.png %}

coordinator就是原来的sql_thread, 不过现在它不再直接更新数据，只负责读取中转日志和分发事务，真正更新日志的变成了worker线程。worker线程的个数，由参数slave_parallel_workers决定的

事务不能按照轮询的方式分发给各个worker：分发之后，不同的worker就独立执行了，由于cpu的调度策略，很可能第二个事务最终比第一个事务限制性，导致主备不一致
同一个事务的多个更新语句不能分给不同的worker执行：虽然最终结果是主备一致的，但是中间过程中如果备库有查询的话，就会看到这个事务更新了一半的结果，破坏了事务逻辑的隔离性

coordinator在分发的时候，需要满足的要求：
- 不能造成更新覆盖。要求更新同一行的两个事务，必须被分发到同一个worker中
- 同一个事务不能拆开，必须放到同一个worker中

### 5.5 版本的并行复制策略

官方不支持并行复制

### 5.6 版本的并行复制策略

支持并行复制，支持的粒度是按库并行

### MariaDB 的并行复制策略

利用组提交
- 能够在同一组里提交的事务，一定不会修改同一行
- 主库上可以并行执行的事务，备库上也一定是可以并行执行的

1. 在一组里面一起提交的事务，有一个相同的commit_id，下一组就是commit_id+1
2. commit_id直接写到binlog里面
3. 传到备库应用的时候，相同commit_id的事务分发到多个worker执行
4. 这一组全部执行完成后，coordinator再去取下一批

主库并行事务：
{% asset_img 38.png %}

备库执行效果：
{% asset_img 39.png %}

在备库上执行的时候，要等第一组事务完全执行完成后，第二组事务才能开始执行，这样系统的吞吐量就不够
另外，很容易被大事务拖后腿，大事务执行期间，只有一个worker线程在工作，浪费资源

### 5.7 版本的并行复制策略

有参数slave-parallel-type控制策略
- DATABASE：表示使用5.6版本的按库并行策略
- LOGICAL_LOCK：病史类似MariaDB的策略（做了优化）

只要能够达到redo log prepare阶段，就表示事务已经通过锁冲突的检验了
思想：
- 同时处于prepare状态的事务，在备库执行时是可以并行的
- 处于prepare状态的事务，与处于commit状态的事务之间，在备库执行时也是可以并行的

- binlog_group_commit_sync_delay：延迟多少微秒后才调用fsync
- binlog_group_commit_sync_no_delay_count：积累多少次以后才调用fsync

这两个参数用于拉长binlog从write到fsync的时间，减少binlog的写盘次数。他们可以用来制造更多的“同时处于prepare阶段的事务”。这样就增加了备库复制的并行度

### 5.7.22 版本的并行复制策略

新增了一个参数binlog-transaction-dependency-tracking，用来控制是否启用这个新策略
- COMMIT_ORDER: 根据同时进入prepare和commit来判断是否可以并行的策略
- WRITESET: 对于事务涉及更新的每一行，计算出这一行的hash值，组成集合writeset，如果两个事务没有操作相同的行，也就是说他们的writeset没有交集，就可以并行
- WRITESET_SESSION：在WRITESET基础上多了一个约束，在主库上同一个线程先后执行的两个事务，在备库执行的时候，要保证相同的先后顺序

hash值通过“库名+表名+索引名+值”计算出来的。对于每一个唯一索引，insert语句对应的writeset就要多增加一个hash值


## 主库故障后的主备切换问题

一主多从基本结构：
{% asset_img 40.png %}

A和A'互为主备

主库发生故障，主备切换后的结果：
{% asset_img 41.png %}

A'成为新的主库，从库B，C，D改接到A'

### 基于位点的主备切换

备库执行change master命令

```
CHANGE MASTER TO
MASTER_HOST=$host_name
MASTER_PORT=$port
MASTER_USER=$user_name
MASTER_PASSWORD=$password
MASTER_LOG_FILE=$master_log_name
MASTER_LOG_POS=$master_log_pos
```

问题：如何定位MASTER_LOG_POS

不精确的定位方式：
1. 等待A'把中转日志relay log全部同步完成
2. A'执行show master status命令，得到A'最新的file和position
3. 取A故障的时刻T
4. 用mysqlbinlog工具解析A'的file，得到T时刻的位点

```
mysqlbinlog File --stop-datetime=T --start-datetime=T
```

mysqlbinlog输出内容
{% asset_img 43.png %}
end_log_pos后面的值123，表示A’这个实例，在T时刻写入binlog的位置。然后我们就可以把123这个值作为$master_log_pos

位点不精确的情况：
假设在T时刻，主库A已经完成执行了一个insert语句插入了一行数据R，并且已经将binlog传给了A'和B，在传完的瞬间A的主机掉电了
此时系统的状态：
- 在B上，由于同步了binlog，R这一行已经存在
- 在A'上，R也存在，日志是写在123位置之后的
- 在B执行change master命令，指向A'的123位置，就会把R这行的binlog又同步到B去执行

此时，B会提示出现主键冲突，停止同步

通常情况，在切换的时候，有两种常用的方法主动跳过错误

1 跳过一个事务

```
set global sql_slave_skip_counter=1;
start slave;
```

需要在B开始接到A'时，持续观察，每次遇到这些错误就停下来，执行一次跳过命令，直到不出现停下来的情况

2 设置slave_skip_errors, 直接设置跳过指定的错误

- 1062错误：插入时唯一键冲突
- 1032错误：删除时找不到数据

### GTID: Global Transaction Identifier

全局事务ID，是一个事务提交时生成的，是这个事务的唯一标识

由两部分组成：
GTID = server_uuid:gno

- server_uuid: 一个实例第一次启动时自动生成的，是一个全局唯一的值
- gno: 一个整数，初始值是1，每次提交事务时分配给这个事务，并加1

GTID模式的启动：启动一个MySQL实例的时候，加上参数gtid_mode=on和enforce_gtid_consistency=on

gtid_next: 决定GTID的生成方式
- gtid_next=automatic，代表使用默认值，这时MySQL就会把server_uu id:gno分配给这个事务
- - 记录binlog的时候，先记录一行SET @@SESSION.GTID_NEXT='server_uuid:gno'
- - 把这个GTID加入本实例的GTID集合
- gtid_next是一个指定的GTID的值，比如通过set gtid_next='current_gtid'指定为current_gtid, 那么就有两种可能
- - 如果current_gtid已经存在于本实例的GTID集合，接下来执行的这个事务就会被忽略
- - 如果current_gtid不存在于本实例的GTID集合，就将这个current_gtid分配给接下来要执行的事务，也就是说系统不需要给这个事务生成新的GTID，gno也不同加1

一个current_gtid只能给一个事务使用，这个事务提交后，如果要执行下一个事务，就要执行set命令，把gtid_next设置为另外一个gtid或者automatic

这样，每个MySQL实例都维护了一个GTID集合，用来对应“这个实例执行过的所有事务”

### 基于GTID的主备切换

```
CHANGE MASTER TO
MASTER_HOST=$master_host
MASTER_PORT=$master_port
MASTER_USER=$master_user
MASTER_PASSWORD=$password
master_auto_position=1
```

master_auto_position=1表示这个主备关系表示的是GTID协议

将A'的GTID集合记为set_a，将B的GTID集合记为set_b
主备切换逻辑：
1. 实例B指定主库A'，基于主备协议建立连接
2. 实例B把set_b发给主库A'
3. A'算出set_a和set_b的差集，判断A'本地是否已经包含了差集需要的所有binlog事务
    3.1 如果不包含，表示A'已经把B所需要的binlog删掉了，返回错误
    3.2 如果确认全部包含，A'从自己的binlog文件里面，找出一个不在set_b的事务，发给B
4. 之后就从这个事务开始，往后读文件，按顺序取binlog发给B去执行

在基于GTID的主备关系里，系统认为只要建立主备关系，就必须保证主库发给备库的日志是完整的，因此，如果实例B需要的日志不存在，A'就拒绝把日志发给B

切换后，主库A'自己生成的binlog中的GTID集合格式是server_uuid_of_A':1-N
如果B之前的GTID集合格式是server_uuid_of_A:1-N，那么切换后GTID集合格式就变成了server_uuid_of_A:1-N, server_uuid_of_A':1-M

因为A'此前也是A的备库，因此A'和B的GTID集合是一样的，这就达到了我们的预期


## 读写分离

读写分离两种架构：客户端直连和proxy

客户端直连
client主动做负载均衡，选择数据库进行查询

{% asset_img 44.png %}

proxy
client连接至proxy，proxy进行分发路由

{% asset_img 45.png %}

### 过期读

由于主从延迟，更新完立即查询，查询的是从库的话，会读到系统的一个过期状态

处理方案
- 强制读主
- sleep：读从库前sleep一下
- 判断主备无延迟
- 配合semi-sync
- 等主库位点
- 等GTID

sleep的另一种实现：更新后直接将输入内容进行展示，这样再刷新的时候读从就相当于间接的sleep了

判断主备无延迟 (无法解决主库已经完成事务但备库还没收到binlog的情况, 持续延迟情况下会出现过度等待)：
1 判断seconds_behind_master是否为0
2 对比位点判断
show slave status命令，其中Master_Log_File和Read_Master_Log_Pos表示读到的主库的最新位点，Relay_Master_Log_File和Exec_Master_Log_Pos表示备库执行的最新位点
3 对比GTID集合
show slave status命令，其中Auto_Position=1表示使用了GTID协议，Retrieved_Gtid_Set表示备库收到的所有日志的GTID集合，Executed_Gtid_Set表示备库所有已经执行完成的GTID集合

{% asset_img 46.png %}

**semi-sync**
1. 事务提交的时候，主库发送binlog给从库
2. 从库收到之后，返回给主库ack
3. 主库收到ack后才会确认“事务完成”

一主一从可以解决过期读，但是一主多从不能

**等主库位点**
等待主库执行到某一个位置
1. 事务更新完成后，执行show master status得到主库当前执行到的位点
2. 在从库上执行select master_pos_wait
3. 如果返回值>=0，则在这个从库执行查询语句
4. 否则（超时或异常）到主库查询语句

**等主库GTID**
1. 事务更新完成后，从返回包直接获取这个事物的GTID
2. 在从库上执行select wait_for_executed_gtid_set
3. 如果返回值是0，则在这个从库上执行查询
4. 否侧（超时）到主库查询
