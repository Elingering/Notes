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
