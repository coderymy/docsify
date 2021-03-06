# mysql的事务

## 事务隔离级别

多个事务同时执行,就会产生以下问题

1. `脏读`: (读取之后数据回滚,事务产生脏数据)事务A读取了事务B更新的数据,但是事务B未提交回滚了
2. `幻读`: (读取之后数据又新增,事务没有处理所有数据)系统获取所有学生信息,进行修改操作,但是在获取之后修改之前又新增了其他的学生.这个时候系统结果并未将所有学生信息修改
3. `不可重复读`:(读取之后数据修改,事务处理的数据错了) 事务A读取的数据,在事务B执行期间修改并提交了


为了解决上述问题,所以事务之间就需要有隔离性.那么隔离级别就有以下几个区分

1. `读未提交`: 事务还没提交,数据就可以被别的事务看到
2. `读已提交`: 事务只有提交了,才能看到数据变更
3. `可重复读`: 在读已提交基础上,当前事务的数据总是与事务开始时一致
4. `串行化`: 所有的事务对数据的修改都是上锁操作的,必须一条一条执行

在启动参数中设置`transaction_isolation`即可配置事务隔离级别,查看当前隔离级别` show variables like 'transaction_isolation'


[https://www.jianshu.com/p/4e3edbedb9a8](https://www.jianshu.com/p/4e3edbedb9a8)


> 本文由 [SnailClimb](https://github.com/Snailclimb) 和 [BugSpeak](https://github.com/BugSpeak) 共同完成。
> <!-- TOC -->

- [mysql的事务](#mysql的事务)
  - [事务隔离级别](#事务隔离级别)
  - [事务隔离级别(图文详解)](#事务隔离级别图文详解)
    - [什么是事务?](#什么是事务)
    - [事务的特性(ACID)](#事务的特性acid)
    - [并发事务带来的问题](#并发事务带来的问题)
    - [事务隔离级别](#事务隔离级别-1)
    - [实际情况演示](#实际情况演示)
      - [脏读(读未提交)](#脏读读未提交)
      - [避免脏读(读已提交)](#避免脏读读已提交)
      - [不可重复读](#不可重复读)
      - [可重复读](#可重复读)
      - [防止幻读(可重复读)](#防止幻读可重复读)
    - [参考](#参考)
- [事务](#事务)
  - [事务的四个特性](#事务的四个特性)
  - [事务的隔离级别](#事务的隔离级别)
  - [隔离级别引发的问题](#隔离级别引发的问题)
- [4. 事务的四大特性和隔离级别](#4-事务的四大特性和隔离级别)
- [1. 概念](#1-概念)
  - [mvcc的结构:](#mvcc的结构)
- [2. why to use MVCC](#2-why-to-use-mvcc)
- [3. how to use MVCC](#3-how-to-use-mvcc)
- [4. 问题](#4-问题)

<!-- /TOC -->

## 事务隔离级别(图文详解)

### 什么是事务?

事务是逻辑上的一组操作，要么都执行，要么都不执行。

事务最经典也经常被拿出来说例子就是转账了。假如小明要给小红转账1000元，这个转账会涉及到两个关键操作就是：将小明的余额减少1000元，将小红的余额增加1000元。万一在这两个操作之间突然出现错误比如银行系统崩溃，导致小明余额减少而小红的余额没有增加，这样就不对了。事务就是保证这两个关键操作要么都成功，要么都要失败。

### 事务的特性(ACID)

![事务的特性](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-6/事务特性.png)


1.  **原子性：** 事务是最小的执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么完全不起作用；
2.  **一致性：** 执行事务前后，数据保持一致，多个事务对同一个数据读取的结果是相同的；
3.  **隔离性：** 并发访问数据库时，一个用户的事务不被其他事务所干扰，各并发事务之间数据库是独立的；
4.  **持久性：** 一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。

### 并发事务带来的问题

在典型的应用程序中，多个事务并发运行，经常会操作相同的数据来完成各自的任务（多个用户对统一数据进行操作）。并发虽然是必须的，但可能会导致以下的问题。

- **脏读（Dirty read）:** 当一个事务正在访问数据并且对数据进行了修改，而这种修改还没有提交到数据库中，这时另外一个事务也访问了这个数据，然后使用了这个数据。因为这个数据是还没有提交的数据，那么另外一个事务读到的这个数据是“脏数据”，依据“脏数据”所做的操作可能是不正确的。
- **丢失修改（Lost to modify）:** 指在一个事务读取一个数据时，另外一个事务也访问了该数据，那么在第一个事务中修改了这个数据后，第二个事务也修改了这个数据。这样第一个事务内的修改结果就被丢失，因此称为丢失修改。	例如：事务1读取某表中的数据A=20，事务2也读取A=20，事务1修改A=A-1，事务2也修改A=A-1，最终结果A=19，事务1的修改被丢失。
- **不可重复读（Unrepeatableread）:** 指在一个事务内多次读同一数据。在这个事务还没有结束时，另一个事务也访问该数据。那么，在第一个事务中的两次读数据之间，由于第二个事务的修改导致第一个事务两次读取的数据可能不太一样。这就发生了在一个事务内两次读到的数据是不一样的情况，因此称为不可重复读。
- **幻读（Phantom read）:** 幻读与不可重复读类似。它发生在一个事务（T1）读取了几行数据，接着另一个并发事务（T2）插入了一些数据时。在随后的查询中，第一个事务（T1）就会发现多了一些原本不存在的记录，就好像发生了幻觉一样，所以称为幻读。

**不可重复度和幻读区别：**

不可重复读的重点是修改，幻读的重点在于新增或者删除。

例1（同样的条件, 你读取过的数据, 再次读取出来发现值不一样了 ）：事务1中的A先生读取自己的工资为     1000的操作还没完成，事务2中的B先生就修改了A的工资为2000，导        致A再读自己的工资时工资变为  2000；这就是不可重复读。

 例2（同样的条件, 第1次和第2次读出来的记录数不一样 ）：假某工资单表中工资大于3000的有4人，事务1读取了所有工资大于3000的人，共查到4条记录，这时事务2 又插入了一条工资大于3000的记录，事务1再次读取时查到的记录就变为了5条，这样就导致了幻读。

### 事务隔离级别

**SQL 标准定义了四个隔离级别：**

- **READ-UNCOMMITTED(读取未提交)：** 最低的隔离级别，允许读取尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**。
- **READ-COMMITTED(读取已提交)：** 允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**。
- **REPEATABLE-READ(可重复读)：**  对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生**。
- **SERIALIZABLE(可串行化)：** 最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。

----

|     隔离级别     | 脏读 | 不可重复读 | 幻影读 |
| :--------------: | :--: | :--------: | :----: |
| READ-UNCOMMITTED |  √   |     √      |   √    |
|  READ-COMMITTED  |  ×   |     √      |   √    |
| REPEATABLE-READ  |  ×   |     ×      |   √    |
|   SERIALIZABLE   |  ×   |     ×      |   ×    |

MySQL InnoDB 存储引擎的默认支持的隔离级别是 **REPEATABLE-READ（可重读）**。我们可以通过`SELECT @@tx_isolation;`命令来查看,MySQL 8.0 该命令改为`SELECT @@transaction_isolation;`

```sql
mysql> SELECT @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
```

这里需要注意的是：与 SQL 标准不同的地方在于InnoDB 存储引擎在 **REPEATABLE-READ（可重读）**事务隔离级别下使用的是Next-Key Lock 锁算法，因此可以避免幻读的产生，这与其他数据库系统(如 SQL Server)是不同的。所以说InnoDB 存储引擎的默认支持的隔离级别是 **REPEATABLE-READ（可重读）** 已经可以完全保证事务的隔离性要求，即达到了 SQL标准的**SERIALIZABLE(可串行化)**隔离级别。

因为隔离级别越低，事务请求的锁越少，所以大部分数据库系统的隔离级别都是**READ-COMMITTED(读取提交内容):**，但是你要知道的是InnoDB 存储引擎默认使用 **REPEATABLE-READ（可重读）**并不会有任何性能损失。

InnoDB 存储引擎在 **分布式事务** 的情况下一般会用到**SERIALIZABLE(可串行化)**隔离级别。

### 实际情况演示

在下面我会使用 2 个命令行mysql ，模拟多线程（多事务）对同一份数据的脏读问题。

MySQL 命令行的默认配置中事务都是自动提交的，即执行SQL语句后就会马上执行 COMMIT 操作。如果要显式地开启一个事务需要使用命令：`START TARNSACTION`。

我们可以通过下面的命令来设置隔离级别。

```sql
SET [SESSION|GLOBAL] TRANSACTION ISOLATION LEVEL [READ UNCOMMITTED|READ COMMITTED|REPEATABLE READ|SERIALIZABLE]
```

我们再来看一下我们在下面实际操作中使用到的一些并发控制语句:

- `START TARNSACTION` |`BEGIN`：显式地开启一个事务。
- `COMMIT`：提交事务，使得对数据库做的所有修改成为永久性。
- `ROLLBACK`：回滚会结束用户的事务，并撤销正在进行的所有未提交的修改。

#### 脏读(读未提交)

<div align="center">  
<img src="https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-31-1脏读(读未提交)实例.jpg" width="800px"/>
</div>


#### 避免脏读(读已提交)

<div align="center">  
<img src="https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-31-2读已提交实例.jpg" width="800px"/>
</div>


#### 不可重复读

还是刚才上面的读已提交的图，虽然避免了读未提交，但是却出现了，一个事务还没有结束，就发生了 不可重复读问题。

<div align="center">  
<img src="https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-32-1不可重复读实例.jpg"/>
</div>


#### 可重复读

<div align="center">  
<img src="https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-33-2可重复读.jpg"/>
</div>


#### 防止幻读(可重复读)

<div align="center">  
<img src="https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-33防止幻读(使用可重复读).jpg"/>
</div>


一个事务对数据库进行操作，这种操作的范围是数据库的全部行，然后第二个事务也在对这个数据库操作，这种操作可以是插入一行记录或删除一行记录，那么第一个是事务就会觉得自己出现了幻觉，怎么还有没有处理的记录呢? 或者 怎么多处理了一行记录呢?

幻读和不可重复读有些相似之处 ，但是不可重复读的重点是修改，幻读的重点在于新增或者删除。

### 参考

- 《MySQL技术内幕：InnoDB存储引擎》
- <https://dev.mysql.com/doc/refman/5.7/en/>
- [Mysql 锁：灵魂七拷问](https://tech.youzan.com/seven-questions-about-the-lock-of-mysql/)
- [Innodb 中的事务隔离级别和锁的关系](https://tech.meituan.com/2014/08/20/innodb-lock.html)





# 事务

## 事务的四个特性

1. 原子性:即针对一个事务,要么所有sql都执行成功,要么所有sql都执行失败
2. 一致性:即事务结果要求数据是最终一致的,不能出现一个表数据执行结果与另一个表执行结果不一致的情况
3. 隔离性:即一个事务的未提交前,不会影响其他事务的执行
4. 持久性:事务一旦提交,对数据库的修改就已经产生不会回滚

## 事务的隔离级别

1. 读未提交:即事务未提交前,别的事务能查询到这个事务的执行结果
2. 读已提交:别的事务只能读取该事务提交后的结果
3. 可重复读:别的事务获取的数据,在另一个事务的事务提交前后是一致的
4. 串行化:即每个事务都依次执行,不会有并发执行的情况(MVCC与此刚好对立)

## 隔离级别引发的问题

1. 脏读:事务A读取了事务B更新的数据,但是事务B未提交回滚了
2. 不可重复读:事务A读取的数据,在事务B执行期间修改并提交了
3. 幻读:系统获取所有学生信息,进行修改操作,但是在获取之后修改之前又新增了其他的学生.这个时候系统结果并未将所有学生信息修改

InnoDB支持事务

<font color="red">一个事务以 `begin;` 开始，以 `COMMIT;` 或 `ROLLBACK;` 结束。</font>

> 四个基本特性

1. 原子性:要么全部执行，要么全部不执行；
2. 一致性:事务的执行使得数据库从一种正确状态转化为另一种正确状态；
3. 隔离性:在事务正确提交之前，不允许把该事务对数据的任何改变提供给其他事务；
4. 持久性:事务提交后，其结果永久保存在数据库中。

> 事务隔离级别

```
读未提交
读已提交---3
可重复读---23
串行化---123

READ_UNCOMMITTED（读未提交）: 一个事务还没提交,别人就能查询到我这个事务变更的数据
READ_COMMITTED（读已提交）: 其他流程只能读取到事务提交之后的数据
REPEATABLE_READ（可重复读）: 保证事务提交前和提交后的数据时一致的,也就是可以在提交前和提交后别的流程读取的数据都是一致的
SERIALIZABLE（串行）: 最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

解决问题，1. 脏读。2. 不可重复读。3. 幻读
```

# 4. 事务的四大特性和隔离级别

> 四个基本特性

1. 原子性:要么全部执行，要么全部不执行；
2. 一致性:事务的执行使得数据库从一种正确状态转化为另一种正确状态；
3. 隔离性:在事务正确提交之前，不允许把该事务对数据的任何改变提供给其他事务；
4. 持久性:事务提交后，其结果永久保存在数据库中。

```
对应四大特性的实现方式
原子性:使用undo log实现,原理就是
1. 在操作任何数据之前,都先将数据备份到另一个地方(这个地方称之为undo log)
2. 然后就行数据修改
3. 如果出现异常或者执行了rollback操作,系统就使用undo log中的数据将数据恢复到事物执行之前的状态

持久性:使用的redo log实现的,原理就是
1. 操作任何数据的时候,都将操作之后的数据进行备份到一个地方(这个地方就叫做redo log)
2. 事务提交之前将redo log持久化
3. 如果 系统出现崩溃则可以直接使用redo log恢复数据

隔离性:通过加锁和MVCC去实现的
```



> 事务隔离级别

```
读未提交
读已提交---3
可重复读---23
串行化---123

READ_UNCOMMITTED（读未提交）: 最低的隔离级别，允许读取尚未提交的数据变更，可能会导致脏读、幻读或不可重复读
READ_COMMITTED（读已提交）: 允许读取并发事务已经提交的数据，可以阻止脏读，但是幻读或不可重复读仍有可能发生
REPEATABLE_READ（可重复读）: 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，可以阻止脏读和不可重复读，但幻读仍有可能发生。
SERIALIZABLE（串行）: 最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

解决问题，1. 脏读。2. 不可重复读。3. 幻读
```

> 四个特性如何保证

原子性由undo log日志保证，它记录了需要回滚的日志信息，事务回滚时撤销已经执行成功的sql

一致性一般由代码层面来保证

隔离性由MVCC来保证

持久性由内存+redo log来保证，mysql修改数据同时在内存和redo log记录这次操作，事务提交的时候通过redo log刷盘，宕机的时候可以从redo log恢复





# 1. 概念

首先需要理解几个概念

> MVCC

mvcc:多版本并发控制	

> 当前读和快照读

当前读:会对读取的信息进行加锁的操作

+ 加锁的select操作,比如后面加上关键字`for update`
+ update、insert、delete操作等

快照读:不加锁的一种非阻塞读操作,

+ `不加锁`的select操作



事务的四种隔离级别中

+ 读已提交
+ 可重复读

都是基于MVCC实现的

串行化,因为是基于一步一步操作执行的,这个时候就必须是用到了锁来限制.所以这个串行化是使用当前读实现的

## mvcc的结构:

+ 版本链
+ undo log
+ readview:是去判断版本链中select有效的哪个版本

![](img/MVCC中概念点.png)

在每执行一步快照读操作的时候,执行的字段中包含上面描述的`事务ID`,和`版本回滚指针`

readView的工作模式

```
readview中有四个字段描述
+ m_ids:生成该条readview时当前系统中活跃的读写事务的`事务id`列表.活跃的事务id指的是未commit的事务id
+ min_trx_id:readview中最小的`事务id`
+ max_trx_id:readview时系统分配给下一个事务的id值
+ creator_trx_id:表示生成该条readview的事务id
```

# 2. why to use MVCC

一般需要并发操作有三个场景

+ 读-读:不涉及数据变更,是不会有并发冲突的
+ 读-写:有可能会出现`脏读、重复读、幻读`
+ 写-写:有可能出现更新失败的情况

mvcc就是为了解决第二种`读-写`操作会需要频繁使用锁机制来保证并发冲突情况的另一种解决办法

以此防止使用锁机制造成效率的下降



# 3. how to use MVCC

1. 借助undo log实现.每条undo log中多两个字段,表示出来`事务id`和`版本回滚指针`
2. 使用readview定位到select操作应该执行的版本信息



# 4. 问题

1. 当前读和快照读在RR下的区别

   ```
   当前读并不需要进行一个串性的操作,而是各个语句的执行相互嵌套
   快照读需要串性的锁来管控
   ```

2. #### RC,RR级别下的InnoDB快照读有什么不同？

   ```
   正是Read View生成时机的不同，从而造成RC,RR级别下快照读的结果的不同
   
   在RC隔离级别下，是每个快照读都会生成并获取最新的Read View；
   而在RR隔离级别下，则是同一个事务中的第一个快照读才会创建Read View, 之后的快照读获取的都是同一个Read View。
   ```

   