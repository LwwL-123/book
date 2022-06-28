# 为什么不建议使用ON DUPLICATE KEY UPDATE？

- 在实习实战开发中，对于一个主键为自增id，和一个唯一索引(aid, 业务key)的表中，我使用了ON DUPLICATE KEY UPDATE，在review的时候，大佬说这个语法有严重的性能和其他隐患问题，必须改成先查询一次分出新增集合和修改集合，再分别进行批量新增和批量修改的方式进行



## 1. 验证

### 1.1 创建一个t1表：

```sql
CREATE TABLE `t1` (
  `a` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键ID',
  `b` int(11),
  `c` int(11),
  PRIMARY KEY (`a`)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8 COMMENT='临时测试表'
```

### 1.2 验证主键插入并更新功能

空表创建好后，多次执行如下sql。(此时只有自增主键a列)

```sql 
INSERT INTO t1 (a,b,c) VALUES (1,2,3) ON DUPLICATE KEY UPDATE c=c+1;
```

- 执行1次的结果:

| a    | b    | c    |
| ---- | ---- | ---- |
| 1    | 2    | 3    |

- 执行2次的结果:

| a    | b    | c    |
| ---- | ---- | ---- |
| 1    | 2    | 4    |

通过观察可知，上面的 sql 在主键已经存在时相当于如下 sql。

```sql
UPDATE t1 SET c=c+1 WHERE a=1;
```



再试下新增的 sql。

```sql
INSERT INTO t1 (b,c) VALUES (20,30) ON DUPLICATE KEY UPDATE c=c+1;
```

| a    | b    | c    |
| ---- | ---- | ---- |
| 1    | 2    | 4    |
| 2    | 20   | 30   |

新增记录成功，id 也自增正常。



### 1.3 验证多字段唯一索引问题

在官方资料中有这样的一句话：

如果列b也是唯一的，那么INSERT等价于这个UPDATE语句

```sql
UPDATE t1 SET c=c+1 WHERE a=1 OR b=2 LIMIT 1;
```

如果a=1 OR b=2匹配多个行，只更新一行。一般来说，应该尽量避免在具有多个唯一索引的表上使用ON DUPLICATE KEY UPDATE子句。

```sql
INSERT INTO t1 (a,b,c) VALUES (3,20,30) ON DUPLICATE KEY UPDATE c=c+1;
```

其 t1 表结果如下：

| a    | b    | c    |
| ---- | ---- | ---- |
| 2    | 20   | 30   |
| 2    | 20   | 31   |

从上面的结果可以看出，其只执行了 update 的操作，从而告诉了我们在使用 on duplicate key update 语句时，应当避免多个唯一索引的场景

当a是一个唯一索引(unique index)时,并且t1表中已经存在a为1的记录时，如下两个sql的效果是一样的。

```sql
INSERT INTO t1 (a,b,c) VALUES (1,2,3) ON DUPLICATE KEY UPDATE c=c+1;
UPDATE t1 SET c=c+1 WHERE a=1;
```

但在innoBD存储类型的表中，当a是一个自增主键时，其效果官方文档中的解释是这样的：

> The effects are not quite identical: For an InnoDB table where a is an auto-increment column, the INSERT statement increases the auto-increment value but the UPDATE does not.
>
> 效果并不完全相同:对于InnoDB表，a是一个自动递增的列，INSERT语句增加自动递增的值，但UPDATE语句不增加。

也就是如果只有一个主键，则会执行新增操作

但当b也是一个唯一索引时，就会执行更新操作, 上面的语句就会变成这样的：

```sql
UPDATE t1 SET c=c+1 WHERE a=1 OR b=2 LIMIT 1;	
```

如果a=1 OR b=2匹配多个行，只更新一行。一般来说，应该尽量避免在具有多个惟一索引的表上使用ON DUPLICATE KEY UPDATE子句。

因此应当避免多唯一索引用on deplicate key update语法



### 1.4 涉及到的锁说明

同时，在查看官网资料中底部对于此语法的说明，从中看到如下描述:

> An INSERT … ON DUPLICATE KEY UPDATE on a partitioned table using a storage engine such as MyISAM that employs table-level locks locks any partitions of the table in which a partitioning key column is updated. (This does not occur with tables using storage engines such as InnoDB that employ row-level locking.) For more information, see Section 22.6.4, “Partitioning and Locking”`https://dev.mysql.com/doc/refman/5.7/en/partitioning-limitations-locking.html`。

主要是说在MyISAM的存储引擎中,on duplicate key update使用的是表级锁来进行实现的，那么就可以存在表级锁时的事务并发性能问题。

但是innoDB引擎中，on duplicate key update是用的行级锁进行实现的。

 	

#  2. 实际情况

```sql
CREATE TABLE IF NOT EXISTS `user_info` (
        id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
        name VARCHAR(20) NOT NULL,
        phone BIGINT(20) UNSIGNED NOT NULL,
        update_time timestamp  NOT NULL,
        UNIQUE KEY phone (phone)
 ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

​    造成死锁的sql如下：

```sql
insert into user_info (name, phone, update_time) values (X,Y,Z) on duplicate key update update_time=Z;
```

​    当我们看到死锁后，在对应数据库中进行分析，”show engine innodb status“，就发现这样的报错信息"lock_mode X locks gap before rec insert intention waiting"。意思就是在等待gap lock（间隙锁）。

​    于是我们开始分析`on duplicate key`这个关键字的sql所可能引入的锁，以及对应我们业务场景中可能触发死锁的问题。



## insert on duplicate key的锁

​    首先insert on duplicate key 这条sql的语义是：如果insert中的对应键值在数据库中没有找到对应的唯一索引记录，即进行插入；如果对表中唯一索引记录冲突，便进行更新，能够很轻松的达到一种效果： 有则直接更新，无则插入。而我们业务中的sql是自增主键id，这样一来冲突的只有可能是 phone这个唯一索引了。

​    首先，在RR的事务隔离级别下，insert on duplicate key这个sql与普通insert只插入意向锁和记录锁不同，insert on duplicate key sql如果没有找到对应的会在唯一键上插入gap lock和插入意向锁（如果有对应记录则会获取next key lock，next key lock 比gap lock多了一个边缘的记录锁）。[Mysql sql lock](https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html)。

​    gap lock即间隙锁，假设目前表中唯一键的数据有以下几个，1，5，10。那么insert的key如果是4,在1-5之间，则获取的gap lock的区间就是（1，5）；如果插入的数据是15，则在10-正无穷之间，因此gap lock的区间就是（10，正无穷），这个gap lock。

​    插入意向锁也是类似于gap lock的一种，生效的范围也一致，只是对应锁上相同范围或者有交集的。横轴为已持有，纵轴为后续申请，是否互斥或兼容。

| 兼容性     | 插入意向锁 | 行锁 | gap lock |
| ---------- | ---------- | ---- | -------- |
| 插入意向锁 | 兼容       | 互斥 | 互斥     |
| 行锁       | 兼容       | 互斥 | 兼容     |
| gap lock   | 兼容       | 兼容 | 兼容     |

​    因此可以看到，在持有gap lock时，在插入的时候如果申请插入意向锁，便会需要等待，而insert on duplicate key的sql在执行时一般就是gap lock和插入意向锁。那么造成死锁的问题就定位到了，肯定是同一时间多个insert事务到来，并且所插入的记录对应的唯一键范围基本一致，所拥有的gap lock和插入意向锁的范围有交集，便可以出现共同持有锁反而造成死锁的问题。

​    那我们大致还原一下对应场景，以下是目前数据库中的数据

| id   | name  | phone       | timestamp |
| ---- | ----- | ----------- | --------- |
| 1    | jack  | 15500000000 | 1970.1.1  |
| 2    | tom   | 15600000000 | 1970.1.1  |
| 3    | hurry | 15700000000 | 1970.1.1  |

| 阶段 | tx1                                                          | tx2                                                          | tx3                                                          |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1    | insert into user_info (name, phone, update_time) values (test1,15700000001,1970.1.1) on duplicate key update update_time=now(); |                                                              |                                                              |
| 1    | 持有（15700000001，正无穷）的插入意向锁以及gap lock          |                                                              |                                                              |
| 2    |                                                              | insert into user_info (name, phone, update_time) values (test2,15700000002,1970.1.1) on duplicate key update update_time=now(); |                                                              |
| 2    |                                                              | 申请（15700000002，正无穷）的插入意向锁失败，申请gap lock成功，等待中 |                                                              |
| 3    |                                                              |                                                              | insert into user_info (name, phone, update_time) values (test3,15700000004,1970.1.1) on duplicate key update update_time=now(); |
| 3    |                                                              |                                                              | 申请（15700000003，正无穷）的插入意向锁失败，申请gap lock成功，等待中 |
| 4    | commit 提交事务，释放锁                                      |                                                              |                                                              |
| 5    |                                                              | 申请插入意向锁成功                                           | 申请插入意向锁成功                                           |
| 6    |                                                              | 死锁                                                         | 死锁                                                         |

​    因此形成死锁，其中一个事务回滚。两个事务都持有gap锁，又想拿到插入意向锁，而插入意向锁与间隙锁是冲突的，所以，产生了死锁



**插入意向锁与间隙锁是冲突的，所以当其它事务持有该间隙的间隙锁时，需要等待其它事务释放间隙锁之后，才能获取到插入意向锁。而间隙锁与间隙锁之间是兼容的，所以所以两个事务中 `select ... for update` 语句并不会相互影响**。

案例中的事务 A 和事务 B 在执行完后 `select ... for update` 语句后都持有范围为`(1006,+∞）`的间隙锁，而接下来的插入操作为了获取到插入意向锁，都在等待对方事务的间隙锁释放，于是就造成了循环等待，导致死锁。



## 为什么间隙锁与间隙锁之间是兼容的？

在MySQL官网上还有一段非常关键的描述：

*Gap locks in InnoDB are “purely inhibitive”, which means that their only purpose is to prevent other transactions from Inserting to the gap. Gap locks can co-exist. A gap lock taken by one transaction does not prevent another transaction from taking a gap lock on the same gap. There is no difference between shared and exclusive gap locks. They do not conflict with each other, and they perform the same function.*

这段话表明间隙锁在本质上是不区分共享间隙锁或互斥间隙锁的，而且间隙锁是不互斥的，即两个事务可以同时持有包含共同间隙的间隙锁。

这里的共同间隙包括两种场景：

- 其一是两个间隙锁的间隙区间完全一样；
- 其二是一个间隙锁包含的间隙区间是另一个间隙锁包含间隙区间的子集。

**间隙锁本质上是用于阻止其他事务在该间隙内插入新记录，而自身事务是允许在该间隙内插入数据的**。也就是说间隙锁的应用场景包括**并发读取、并发更新、并发删除和并发插入**。



## 插入意向锁是什么？

注意！插入意向锁名字虽然有意向锁，但是它并不是意向锁，它是一种特殊的间隙锁。

在MySQL的官方文档中有以下重要描述：

*An Insert intention lock is a type of gap lock set by Insert operations prior to row Insertion. This lock signals the intent to Insert in such a way that multiple transactions Inserting into the same index gap need not wait for each other if they are not Inserting at the same position within the gap. Suppose that there are index records with values of 4 and 7. Separate transactions that attempt to Insert values of 5 and 6, respectively, each lock the gap between 4 and 7 with Insert intention locks prior to obtaining the exclusive lock on the Inserted row, but do not block each other because the rows are nonconflicting.*

这段话表明尽管**插入意向锁是一种特殊的间隙锁，但不同于间隙锁的是，该锁只用于并发插入操作**。

如果说间隙锁锁住的是一个区间，那么「插入意向锁」锁住的就是一个点。因而从这个角度来说，插入意向锁确实是一种特殊的间隙锁。

插入意向锁与间隙锁的另一个非常重要的差别是：尽管「插入意向锁」也属于间隙锁，但两个事务却不能在同一时间内，一个拥有间隙锁，另一个拥有该间隙区间内的插入意向锁（当然，插入意向锁如果不在间隙锁区间内则是可以的）。

另外，我补充一点，插入意向锁的生成时机：

- 每插入一条新记录，都需要看一下待插入记录的下一条记录上是否已经被加了间隙锁，如果已加间隙锁，那 Insert 语句应该被阻塞，并生成一个插入意向锁 