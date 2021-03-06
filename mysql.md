## 逻辑架构图
![](/0d2070e8f84c4801adbfa03bda1f98d9.png "")    

- 连接器    
连接器负责跟客户端建立连接、获取权限、维持和管理连接。    
- 查询缓存    
MySQL 拿到一个查询请求后，会先到查询缓存看看，之前是不是执行过这条语句。之前执行过的语句及其结果可能会以 key-value 对的形式，被直接缓存在内存中。MySQL 8.0 版本直接将查询缓存的整块功能删掉了。    
- 优化器    
优化器是在表里面有多个索引的时候，决定使用哪个索引；或者在一个语句有多表关联（join）的时候，决定各个表的连接顺序。    
- 执行器    
开始执行的时候，要先判断一下你对这个表 T 有没有执行查询的权限，如果没有，就会返回没有权限的错误。    
## redo-log-binlog
- redo log 是 InnoDB 引擎特有的,binlog 是 MySQL 的 Server 层实现的,所有引擎都可以使用.    
- redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。    
- redo log 是循环写的，空间固定会用完；；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。        
sql语句查询流程    


        mysql> create table T(ID int primary key, c int);
        mysql> update T set c=c+1 where ID=2;    
        
![](/2e5bff4910ec189fe1ee6e2ecc7b4bbe.png "流程")

## 隔离性与隔离级别
- 读未提交（read uncommitted）    
一个事务还没提交时，它做的变更就能被别的事务看到。    
- 读提交（read committed）    
一个事务提交之后，它做的变更才会被其他事务看到。    
- 可重复读（repeatable read）    
一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。当然在可重复读隔离级别下，未提交变更对其他事务也是不可见的。    
- 串行化（serializable ）    
顾名思义是对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行。    
![](/7dea45932a6b722eb069d2264d0066f8.png "")     
- 若隔离级别是“读未提交”，则 V1 的值就是 2。这时候事务 B 虽然还没有提交，但是结果已经被 A 看到了。因此，V2、V3 也都是 2。    
- 若隔离级别是“读提交”，则 V1 是 1，V2 的值是 2。事务 B 的更新在提交后才能被 A 看到。所以， V3 的值也是 2。    
- 若隔离级别是“可重复读”，则 V1、V2 是 1，V3 是 2。之所以 V2 还是 1，遵循的就是这个要求：事务在执行期间看到的数据前后必须是一致的。    
- 若隔离级别是“串行化”，则在事务 B 执行“将 1 改成 2”的时候，会被锁住。直到事务 A 提交后，事务 B 才可以继续执行。所以从 A 的角度看，V1、V2 值是 1，V3 的值是 2。    
## InnoDB 基于主键索引和普通索引的查询有什么区别？
- 如果语句是 select * from T where ID=500，即主键查询方式，则只需要搜索 ID 这棵 B+ 树；    
- 如果语句是 select * from T where k=5，即普通索引查询方式，则需要先搜索 k 索引树，得到 ID 的值为 500，再到 ID 索引树搜索一次。这个过程称为回表。    
也就是说，基于非主键索引的查询需要多扫描一棵索引树。
## 锁
根据加锁的范围，MySQL 里面的锁大致可以分成全局锁、表级锁和行锁三类。    
- 全局锁    
顾名思义，全局锁就是对整个数据库实例加锁。MySQL 提供了一个加全局读锁的方法，命令是 Flush tables with read lock (FTWRL)。当你需要让整个库处于只读状态的时候，可以使用这个命令，之后其他线程的以下语句会被阻塞：数据更新语句（数据的增删改）、数据定义语句（包括建表、修改表结构等）和更新类事务的提交语句。    
全局锁的典型使用场景是，做全库逻辑备份。    
- 表级锁    
MySQL 里面表级别的锁有两种：一种是表锁，一种是元数据锁（meta data lock，MDL)。    
表锁的语法是 lock tables … read/write。与 FTWRL 类似，可以用 unlock tables 主动释放锁，也可以在客户端断开的时候自动释放。对于 InnoDB 这种支持行锁的引擎，一般不使用 lock tables 命令来控制并发，毕竟锁住整个表的影响面还是太大。    
另一类表级的锁是 MDL（metadata lock)。MDL 不需要显式使用，访问一个表的时候会被自动加上。MDL 的作用是，保证读写的正确性。当对一个表做增删改查操作的时候，加 MDL 读锁；当要对表做结构变更操作的时候，加 MDL 写锁。    
读锁之间不互斥，因此你可以有多个线程同时对一张表增删改查。    
读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。    
- 行锁    
MySQL 的行锁是在引擎层由各个引擎自己实现的。但并不是所有的引擎都支持行锁，比如 MyISAM 引擎就不支持行锁。InnoDB 是支持行锁的。    
顾名思义，行锁就是针对数据表中行记录的锁。在 InnoDB 事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。这个就是两阶段锁协议。    
InnoDB存储引擎的锁的算法有三种：Record lock：单个行记录上的锁；Gap lock：间隙锁，锁定一个范围，不包括记录本身；Next-key lock：record+gap 锁定一个范围，包含记录本身。    
- 死锁和死锁检测    
当出现死锁以后，有两种策略：一种策略是，直接进入等待，直到超时。这个超时时间可以通过参数innodb_lock_wait_timeout 来设置。另一种策略是，发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。将参数 innodb_deadlock_detect 设置为 on，表示开启这个逻辑。    
