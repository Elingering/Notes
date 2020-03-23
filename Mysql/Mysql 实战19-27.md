# 19 | 为什么我只查一行的语句，也执行这么慢？
为了便于描述，我还是构造一个表，基于这个表来说明今天的问题。这个表有两个字段 id 和 c，并且我在里面插入了 10 万行记录。
```sql
mysql> CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=100000) do
    insert into t values(i,i);
    set i=i+1;
  end while;
end;;
delimiter ;

call idata();
```

## 第一类：查询长时间不返回
> mysql> select * from t where id=1;
一般碰到这种情况的话，大概率是表 t 被锁住了。接下来分析原因的时候，一般都是首先执行一下 show processlist 命令，看看当前语句处于什么状态。

### 等 MDL 锁
这类问题的处理方式，就是找到谁持有 MDL 写锁，然后把它 kill 掉。

但是，由于在 show processlist 的结果里面，session A 的 Command 列是“Sleep”，导致查找起来很不方便。不过有了 performance_schema 和 sys 系统库以后，就方便多了。（MySQL 启动时需要设置 performance_schema=on，相比于设置为 off 会有 10% 左右的性能损失)

通过查询 sys.schema_table_lock_waits 这张表，我们就可以直接找出造成阻塞的 process id，把这个连接用 kill 命令断开即可。
![title](https://raw.githubusercontent.com/Elingering/note-images/master/note-images/2019/07/17/1563349225332-1563349225337.png)

### 等 flush
![title](https://raw.githubusercontent.com/Elingering/note-images/master/note-images/2019/07/17/1563349497517-1563349497522.png)
出现 Waiting for table flush 状态的可能情况是：有一个 flush tables 命令被别的语句堵住了，然后它又堵住了我们的 select 语句。
复现 Waiting for table flush
![title](https://raw.githubusercontent.com/Elingering/note-images/master/note-images/2019/07/17/1563349564023-1563349564026.png)
![title](https://raw.githubusercontent.com/Elingering/note-images/master/note-images/2019/07/17/1563349577259-1563349577262.png)

### 等行锁
> mysql> select * from t where id=1 lock in share mode; 
由于访问 id=1 这个记录时要加读锁，如果这时候已经有一个事务在这行记录上持有一个写锁，我们的 select 语句就会被堵住。
复现：
![title](https://raw.githubusercontent.com/Elingering/note-images/master/note-images/2019/07/17/1563349671045-1563349671048.png)
![title](https://raw.githubusercontent.com/Elingering/note-images/master/note-images/2019/07/17/1563349712050-1563349712055.png)
如果你用的是 MySQL 5.7 版本，可以通过 sys.innodb_lock_waits 表查到是谁占着这个写锁。
> mysql> select * from t sys.innodb_lock_waits where locked_table=`'test'.'t'`\G
![title](https://raw.githubusercontent.com/Elingering/note-images/master/note-images/2019/07/17/1563349912598-1563349912629.png)
> 解决 kill 4

## 第二类：查询慢
> mysql> select * from t where c=50000 limit 1;
由于字段 c 上没有索引，这个语句只能走 id 主键顺序扫描，因此需要扫描 5 万行。

这里为了把所有语句记录到 slow log 里，我在连接后先执行了 set long_query_time=0，将慢查询日志的时间阈值设置为 0。
![title](https://raw.githubusercontent.com/Elingering/note-images/master/note-images/2019/07/17/1563350099920-1563350099926.png)
全表扫描 5 万行的 slow log

> mysql> select * from t where id=1；
虽然扫描行数是 1，但执行时间却长达 800 毫秒。
![title](https://raw.githubusercontent.com/Elingering/note-images/master/note-images/2019/07/17/1563350185816-1563350185824.png)
复现：
![title](https://raw.githubusercontent.com/Elingering/note-images/master/note-images/2019/07/17/1563350386724-1563350386734.png)
session B 更新完 100 万次，生成了 100 万个回滚日志 (undo log)。

==带 lock in share mode 的 SQL 语句，是当前读==，因此会直接读到 1000001 这个结果，所以速度很快；而 select * from t where id=1 这个语句，是一致性读，因此需要从 1000001 开始，依次执行 undo log，执行了 100 万次以后，才将 1 这个结果返回。

## 小结
今天我给你举了在一个简单的表上，执行“查一行”，可能会出现的被锁住和执行慢的例子。这其中涉及到了表锁、行锁和一致性读的概念。

## 问题
我们在举例加锁读的时候，用的是这个语句，select * from t where id=1 lock in share mode。由于 id 上有索引，所以可以直接定位到 id=1 这一行，因此读锁也是只加在了这一行上。

但如果是下面的 SQL 语句，
```sql
begin;
select * from t where c=5 for update;
commit;
```
这个语句序列是怎么加锁的呢？加的锁又是什么时候释放呢？

答：读提交隔离级别下，在语句执行完成后，是只有行锁的。而且语句执行完成后，InnoDB就会把不满足条件的行行锁去掉。

当然了，c=5这一行的行锁，还是会等到commit的时候才释放的。

# 20 | 幻读是什么，幻读有什么问题？

## 幻读是什么？
假设一个场景，只在 id=5 这一行加行锁
![title](https://raw.githubusercontent.com/Elingering/note-images/master/note-images/2019/07/17/1563351369274-1563351369286.png)
其中，Q3 读到 id=1 这一行的现象，被称为“幻读”。也就是说，幻读指的是一个事务在前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有看到的行。
1. 在可重复读隔离级别下，普通的查询是快照读，是不会看到别的事务插入的数据的。因此，==幻读在“当前读”下才会出现==。
2. 上面 session B 的修改结果，被 session A 之后的 select 语句用“当前读”看到，不能称为幻读。==幻读仅专指“新插入的行”==。

## 幻读有什么问题？
首先是语义上的。session A 在 T1 时刻就声明了，“我要把所有 d=5 的行锁住，不准别的事务进行读写操作”。而实际上，这个语义被破坏了。

如果现在这样看感觉还不明显的话，我再往 session B 和 session C 里面分别加一条 SQL 语句，你再看看会出现什么现象。
![title](https://raw.githubusercontent.com/Elingering/note-images/master/note-images/2019/07/17/1563351546471-1563351546479.png)
三个查询都是加了==for update，都是当前读==。而当前读的规则，就是要==能读到所有已经提交的记录的最新值==。

## 如何解决幻读
产生幻读的原因是，行锁只能锁住行，但是新插入记录这个动作，要更新的是记录之间的“间隙”。因此，为了解决幻读问题，InnoDB只好引入新的锁，也就是==间隙锁==(Gap Lock)。

跟间隙锁存在冲突关系的，是“往这个间隙中==插入==一个记录”这个操作。==间隙锁之间都不存在冲突关系==。

间隙锁和行锁合称next-key lock，每个next-key lock是前开后闭区间。

==间隙锁的引入==，可能会导致同样的语句锁住更大的范围，这其实是==影响了并发度==的。

间隙锁是在可重复读隔离级别下才会生效的。

==如果把隔离级别设置为读提交的话，就没有间隙锁了。但同时，你要解决可能出现的数据和日志不一致问题，需要把binlog格式设置为row。==

## 问题
![title](https://raw.githubusercontent.com/Elingering/note-images/master/gitnote/2020/03/22/Snipaste_2020-03-22_14-40-34-1584860159587.png)

# 21 | 为什么我只改一行的语句，锁这么多

我总结的加锁规则里面，包含了两个“原则”、两个“优化”和一个“bug”。

1. 原则1：加锁的基本单位是next-key lock。希望你还记得，next-key lock是前开后闭区间。
2. 原则2：查找过程中访问到的对象才会加锁。
3. 优化1：索引上的等值查询，给唯一索引加锁的时候，next-key lock退化为行锁。
4. 优化2：索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock退化为间隙锁。
5. 一个bug：唯一索引上的范围查询会访问到不满足条件的第一个值为止。

在==删除数据的时候尽量加limit==。这样不仅可以==控制删除数据的条数==，让操作更安全，还==可以减小加锁的范围==。

在读提交隔离级别下还有一个优化，即：语句执行过程中加上的行锁，在语句执行完成后，就要把“不满足条件的行”上的行锁直接释放了，不需要等到事务提交。

也就是说，读提交隔离级别下，锁的范围更小，锁的时间更短，这也是不少业务都默认使用读提交隔离级别的原因。

# 22 | MySQL有哪些“饮鸩止渴”提高性能的方法

## 慢查询性能问题
在MySQL中，会引发性能问题的慢查询，大体有以下三种可能：
1. 索引没有设计好；
2. SQL语句没写好；
3. MySQL选错了索引。

## QPS突增问题
有时候由于业务突然出现高峰，或者应用程序bug，导致某个语句的QPS突然暴涨，也可能导致MySQL压力过大，影响服务。

我之前碰到过一类情况，是由一个新功能的bug导致的。当然，最理想的情况是让业务把这个功能下掉，服务自然就会恢复。

# 23 | MySQL是怎么保证数据不丢的

## binlog的写入机制
binlog的写入逻辑比较简单：事务执行过程中，先把日志写到binlog cache，事务提交的时候，再把binlog cache写到binlog文件中。

一个事务的binlog是不能被拆开的，因此不论这个事务多大，也要确保一次性写入。

## redo log的写入机制
事务在执行过程中，生成的redo log是要先写到redo log buffer，然后再flush到redo log

## 如果你的MySQL现在出现了性能瓶颈，而且瓶颈在IO上，可以通过哪些方法来提升性能呢？
1. 设置 binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count参数，减少binlog的写盘次数。这个方法是基于“额外的故意等待”来实现的，因此可能会增加语句的响应时间，但没有丢失数据的风险。
2. 将sync_binlog 设置为大于1的值（比较常见是100~1000）。这样做的风险是，主机掉电时会丢binlog日志。
3. 将innodb_flush_log_at_trx_commit设置为2。这样做的风险是，主机掉电的时候会丢数据。

# 24 | MySQL是怎么保证主备一致的

## MySQL主备的基本原理
1. 在备库B上通过change master命令，设置主库A的IP、端口、用户名、密码，以及要从哪个位置开始请求binlog，这个位置包含文件名和日志偏移量。
2. 在备库B上执行start slave命令，这时候备库会启动两个线程，就是图中的io_thread和sql_thread。其中io_thread负责与主库建立连接。
3. 主库A校验完用户名、密码后，开始按照备库B传过来的位置，从本地读取binlog，发给B。
4. 备库B拿到binlog后，写到本地文件，称为中转日志（relay log）。
5. sql_thread读取中转日志，解析出日志里的命令，并执行。

建议你把节点B（也就是备库）设置成只读（readonly）模式。这样做，有以下几个考虑：
1. 有时候一些运营类的查询语句会被放到备库上去查，设置为只读可以防止误操作；
2. 防止切换逻辑有bug，比如切换过程中出现双写，造成主备不一致；
3. 可以用readonly状态，来判断节点的角色。

因为readonly设置对超级(super)权限用户是无效的，而用于同步更新的线程，就拥有超级权限。

## binlog的三种格式对比
1. 因为有些==statement格式==的binlog可能会导致主备不一致，所以要使用row格式。
2. 但==row格式==的缺点是，很占空间。比如你用一个delete语句删掉10万行数据，用statement的话就是一个SQL语句被记录到binlog中，占用几十个字节的空间。但如果用row格式的binlog，就要把这10万条记录都写到binlog中。这样做，不仅会占用更大的空间，同时写binlog也要耗费IO资源，影响执行速度。
3. 所以，MySQL就取了个折中方案，也就是有了==mixed格式==的binlog。mixed格式的意思是，MySQL自己会判断这条SQL语句是否可能引起主备不一致，如果有可能，就用row格式，否则就用statement格式。

现在越来越多的场景要求把MySQL的binlog格式设置成row。这么做的理由有很多，我来给你举一个可以直接看出来的好处：==恢复数据==。

==用binlog来恢复数据的标准做法==是，用 mysqlbinlog工具解析出来，然后把解析结果整个发给MySQL执行。类似下面的命令：
```sql
mysqlbinlog master.000001  --start-position=2738 --stop-position=2973 | mysql -h127.0.0.1 -P13000 -u$user -p$pwd;
```
这个命令的意思是，将 master.000001 文件里面从第2738字节到第2973字节中间这段内容解析出来，放到MySQL去执行。

## 循环复制问题
节点A和B之间总是互为主备关系。这样在切换的时候就不用再修改主备关系。
解决两个节点间的循环复制的问题：
1. 规定两个库的server id必须不同，如果相同，则它们之间不能设定为主备关系；
2. 一个备库接到binlog并在重放的过程中，生成与原binlog的server id相同的新的binlog；
3. 每个库在收到从自己的主库发过来的日志后，先判断server id，如果跟自己的相同，表示这个日志是自己生成的，就直接丢弃这个日志。

# 25 | MySQL是怎么保证高可用的
## 主备延迟
“同步延迟”。与数据同步有关的时间点主要包括以下三个：
1. 主库A执行完成一个事务，写入binlog，我们把这个时刻记为T1;
2. 之后传给备库B，我们把备库B接收完这个binlog的时刻记为T2;
3. 备库B执行完成这个事务，我们把这个时刻记为T3。

所谓主备延迟，就是同一个事务，在备库执行完成的时间和主库执行完成的时间之间的差值，也就是T3-T1。

你可以在备库上执行show slave status命令，它的返回结果里面会显示seconds_behind_master，用于表示当前备库延迟了多少秒。

主备延迟最直接的表现是，备库消费中转日志（relay log）的速度，比主库生产binlog的速度要慢。

## 主备延迟的来源
1. 首先，有些部署条件下，备库所在机器的性能要比主库所在的机器性能差。
2. 第二种常见的可能了，即备库的压力大。解决：
- 一主多从。除了备库外，可以多接几个从库，让这些从库来分担读的压力。
- 通过binlog输出到外部系统，比如Hadoop这类系统，让外部系统提供统计类查询的能力。
3. 第三种可能了，即大事务。原因：
- 一次性地用delete语句删除太多数据
- 大表DDL
4. 备库的并行复制能力

## 可靠性优先策略
![title](https://raw.githubusercontent.com/Elingering/note-images/master/gitnote/2020/03/23/Snipaste_2020-03-23_16-08-20-1584951489196.png)
## 可用性优先策略
有没有哪种情况数据的可用性优先级更高呢？
我曾经碰到过这样的一个场景：

- 有一个库的作用是记录操作日志。这时候，如果数据不一致可以通过binlog来修补，而这个短暂的不一致也不会引发业务问题。
- 同时，业务系统依赖于这个日志写入逻辑，如果这个库不可写，会导致线上的业务操作无法执行。
这时候，你可能就需要选择先强行切换，事后再补数据的策略。

当然，事后复盘的时候，我们想到了一个改进措施就是，让业务逻辑不要依赖于这类日志的写入。也就是说，日志写入这个逻辑模块应该可以降级，比如写到本地文件，或者写到另外一个临时库里面。

这样的话，这种场景就又可以使用可靠性优先策略了。

## 小结
在满足数据可靠性的前提下，MySQL高可用系统的可用性，是依赖于主备延迟的。延迟的时间越小，在主库故障的时候，服务恢复需要的时间就越短，可用性就越高。

在实际的应用中，我更建议使用可靠性优先的策略。毕竟保证数据准确，应该是数据库服务的底线。在这个基础上，通过减少主备延迟，提升系统的可用性。

# 26 | 备库为什么会延迟好几个小时
## 备库并行复制能力
在官方的5.6版本之前，MySQL只支持单线程复制，由此在主库并发高、TPS高时就会出现严重的主备延迟问题。
![title](https://raw.githubusercontent.com/Elingering/note-images/master/gitnote/2020/03/23/Snipaste_2020-03-23_16-29-34-1584952207524.png)
coordinator在分发的时候，需要满足以下这两个基本要求：
1. 不能造成更新覆盖。这就要求更新同一行的两个事务，必须被分发到同一个worker中。
2. 同一个事务不能被拆开，必须放到同一个worker中。

## MySQL 5.5版本的并行复制策略
官方MySQL 5.5版本是==不支持并行复制==的。但是，在2012年的时候，我自己服务的业务出现了严重的主备延迟，原因就是备库只有单线程复制。然后，我就先后写了两个版本的并行策略。

- 按表分发策略
按表分发事务的基本思路是，如果两个事务更新不同的表，它们就可以并行。因为数据是存储在表里的，所以按表分发，可以保证两个worker不会更新同一行。

- 按行分发策略
要解决热点表的并行复制问题，就需要一个按行并行复制的方案。按行复制的核心思路是：如果两个事务没有更新相同的行，它们在备库上可以并行执行。显然，这个模式要求binlog格式必须是row。

## MySQL 5.6版本的并行复制策略


# 27 | 主库出问题了，从库怎么办