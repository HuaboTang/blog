---
title: 如何处理MySQL死锁（翻译）
---

> 原文链接：https://dzone.com/articles/how-deal-mysql-deadlocks


当二个或以上的事务操作，同时持有或请求锁，并产生了循环依赖，此时将产生死锁。在事务系统中，死锁是无法完全避免的。InnoDB引擎会自动检测死锁，在死锁发生时回滚事务并返回一个错误。它使用回滚来简单的度量事务（It uses a metric to pick the easiest transaction to rollback）。不需要担心偶然发生的死锁，但频繁出现的，则需要重视。

在MySQL5.6之前，使用`SHOW ENGINE INNODBS STATUS`来查看最后一条死锁记录。`Percona Toolkits`提供的`pt-deadlock-logger`，可以定时通过`SHOW ENGINE INNODBS STATUS`来获取死锁，并记录到文件或表格中，供后续分析。更多关于`pt-deadlock-logger`请参考[这里](https://www.percona.com/doc/percona-toolkit/2.2/pt-deadlock-logger.html)。MySQL5.6，你可以使用新的变量`innodb_print_all_deadlocks`来获取所有InnoDB记录在mysqld错误日志中的死锁信息。

我们建议在应用中捕获死锁，并进行重试。

## 如何诊断MySQL死锁

一个MySQL死锁，可能由超过两个事务产生，但`LASTEST DETECTED DEADLOCK`仅显示最后的两个事务。而且只显示这两个事务中，最后执行的语句，以及这两个事务产生循环依赖的锁。可能会遗漏获取锁的早期信息。我将展示一些指引，来收集这些丢失的语句。

让我们来看看两个实际的案例，例子1：
```
1 141013 6:06:22
2 *** (1) TRANSACTION:
3 TRANSACTION 876726B90, ACTIVE 7 sec setting auto-inc lock
4 mysql tables in use 1, locked 1
5 LOCK WAIT 9 lock struct(s), heap size 1248, 4 row lock(s), undo log entries 4
6 MySQL thread id 155118366, OS thread handle 0x7f59e638a700, query id 87987781416 localhost msandbox update
7 INSERT INTO t1 (col1, col2, col3, col4) values (10, 20, 30, 'hello')
8 *** (1) WAITING FOR THIS LOCK TO BE GRANTED:
9 TABLE LOCK table `mydb`.`t1` trx id 876726B90 lock mode AUTO-INC waiting
10 *** (2) TRANSACTION:
11 TRANSACTION 876725B2D, ACTIVE 9 sec inserting
12 mysql tables in use 1, locked 1
13 876 lock struct(s), heap size 80312, 1022 row lock(s), undo log entries 1002
14 MySQL thread id 155097580, OS thread handle 0x7f585be79700, query id 87987761732 localhost msandbox update
15 INSERT INTO t1 (col1, col2, col3, col4) values (7, 86, 62, "a lot of things"), (7, 76, 62, "many more")
16 *** (2) HOLDS THE LOCK(S):
17 TABLE LOCK table `mydb`.`t1` trx id 876725B2D lock mode AUTO-INC
18 *** (2) WAITING FOR THIS LOCK TO BE GRANTED:
19 RECORD LOCKS space id 44917 page no 529635 n bits 112 index `PRIMARY` of table `mydb`.`t2` trx id 876725B2D lock mode S locks rec but not gap waiting
20 *** WE ROLL BACK TRANSACTION (1)
```
第1行，给出了死锁发生的时间。我们再次建议在应用代码中捕获死锁的信息，那么你可以在日志中匹配上这两个时间。应当从这里开始回滚，回滚事务中已经执行的所有语句。

第3行和11行，给出了事务的编号和生效的时间。如果你定期记录了`SHOW ENGINE INNODB STATUS`的内容，那么你可能通过编号来寻找到事务内更早的执行语句。`ACTIVE sec`提示了事务是单个语句还是多个语句。

第4行和第12行，仅显示与当前语句相关，锁定和使用的表数量。因此，使用1个表，并不一定意味着该事务仅涉及一个表。

第5行和第13行，这是有价值的一条信息，`undo log entries`展示了事务已经产生多少变化，`row lock(s)`展示了事务持有了多少行锁。这些信息给出了当前事务的复杂性。

第6行和第14行，记录了线程的ID、连接的host和用户。建议在不同的应用中，使用不同的用户进行连接，可以根据此来判断事务是来处哪个应用。

第9行，代表第一个事务正在等待锁，在这个示例中是表`t1`上的`AUTO-INC`锁。其它的可能性是`S`，代表共享锁；`X`代表独占锁，以及有或没有间隙锁。

第16和17行，代表第二个事务持有了锁，在这个示例里，它正持有事务1等待的锁。

第18和19行，表示第二个事务正在等待的锁。在这个示例里，它是一个在另外一个表的主键上的共享锁，并且无间隙锁。InnoDB中，共享锁只有以下几个来源：
1. 使用`SELECT ... LOCK IN SHARE MODE`
2. 外键关联记录
3. 在`INSERT INTO ... SELECT`的情况下，来源表会加上分享锁
事务2只是简单的执行了往表t1中插入记录的操作，那可以排除1和3，只剩下2。可以通过`SHOW CREATE TABLE t1`来确认是否有外键关联，即可判断S锁产生原因。

示例2：使用MySQL社区版本，每个记录锁的内容都将被输出
```
1 2014-10-11 10:41:12 7f6f912d7700
2 *** (1) TRANSACTION:
3 TRANSACTION 2164000, ACTIVE 27 sec starting index read
4 mysql tables in use 1, locked 1
5 LOCK WAIT 3 lock struct(s), heap size 360, 2 row lock(s), undo log entries 1
6 MySQL thread id 9, OS thread handle 0x7f6f91296700, query id 87 localhost ro ot updating
7 update t1 set name = 'b' where id = 3
8 *** (1) WAITING FOR THIS LOCK TO BE GRANTED:
9 RECORD LOCKS space id 1704 page no 3 n bits 72 index `PRIMARY` of table `tes t`.`t1` trx id 2164000 lock_mode X locks rec but not gap waiting
10 Record lock, heap no 4 PHYSICAL RECORD: n_fields 5; compact format; info bit s 0
11 0: len 4; hex 80000003; asc ;;
12 1: len 6; hex 000000210521; asc ! !;;
13 2: len 7; hex 180000122117cb; asc ! ;;
14 3: len 4; hex 80000008; asc ;;
15 4: len 1; hex 63; asc c;;
16
17 *** (2) TRANSACTION:
18 TRANSACTION 2164001, ACTIVE 18 sec starting index read
19 mysql tables in use 1, locked 1
20 3 lock struct(s), heap size 360, 2 row lock(s), undo log entries 1
21 MySQL thread id 10, OS thread handle 0x7f6f912d7700, query id 88 localhost r oot updating
22 update t1 set name = 'c' where id = 2
23 *** (2) HOLDS THE LOCK(S):
24 RECORD LOCKS space id 1704 page no 3 n bits 72 index `PRIMARY` of table `tes t`.`t1` trx id 2164001 lock_mode X locks rec but not gap
25 Record lock, heap no 4 PHYSICAL RECORD: n_fields 5; compact format; info bit s 0
26 0: len 4; hex 80000003; asc ;;
27 1: len 6; hex 000000210521; asc ! !;;
28 2: len 7; hex 180000122117cb; asc ! ;;
29 3: len 4; hex 80000008; asc ;;
30 4: len 1; hex 63; asc c;;
31
32 *** (2) WAITING FOR THIS LOCK TO BE GRANTED:
33 RECORD LOCKS space id 1704 page no 3 n bits 72 index `PRIMARY` of table `tes t`.`t1` trx id 2164001 lock_mode X locks rec but not gap waiting
34 Record lock, heap no 3 PHYSICAL RECORD: n_fields 5; compact format; info bit s 0
35 0: len 4; hex 80000002; asc ;;
36 1: len 6; hex 000000210520; asc ! ;;
37 2: len 7; hex 17000001c510f5; asc ;;
38 3: len 4; hex 80000009; asc ;;
39 4: len 1; hex 62; asc b;;
```

第9、10行：`space id`是表空间的id，`page no`给出了表空间内记录锁定的页面。The ‘n bits’ is not the page offset, instead the number of bits in the lock bitmap。页面的偏移量是第10行中的`heap no`。

第11~15行：以16进制的方式展示了行数据。`0`行是主键索引。忽略最高位，值是3.`1`行是最后修改改记录的事务ID，10进行值是2164001，即事务2。行`2`是回滚点。从`3`开始，是行数据的其它部分。`3`是整形列，值是8.行`4`是字符串类型，值是c。通过观察这些值，可以获知哪行被锁以及当前值是什么。

### 通过分析我们还可以得到什么？
当MySQL死锁在两个事务间产生时，我们应当基于以下的假设开始分析。在示例1中，事务2等待一个分享锁，那么事务1持有在表`t2`主键上的共享锁或者独占锁。`col2`是外键列，通过检查事务1的操作，它并没有需求相同的记录锁，因此必然是事务1中之前的操作需要事务2主键记录的`S`或者`X`锁。事务1在7秒内，仅变更了4行数据。因此可以得出一些问题：事务1进行了大量操作，但较少的变更；这些改变涉及t1和t2，单一的记录写入到t2中。这些信息将帮助开发人员定位问题位置。

### 从哪里我们可以获取事务之前的操作？

除了应用的日志和之前的`SHOW ENGINE INNODB STATUS`输出，你也可以利用binlog、慢日志和查询日志。使用binlog，如果设置binlog_format=statement，每一条日志将记录线程ID。仅提交的事务才会被记录到binlog中，因此我们可以查询事务2的日志。在示例1中，我们知道死锁发生的事情，并且知道事务2在9秒之前启动。我们可以使用`mysqlbinlog`来查询线程ID为155097580的执行语句。
```
$ mysqlbinlog -vvv --start-datetime=“2014-10-13 6:06:12” \
--stop-datatime=“2014-10-13 6:06:22” mysql-bin.000010 > binlog_1013_0606.out
```

在`Percona Server`5.5及以上版本，我们可以通过设置`log_show_verbosity`使得慢日志中包含事务id。如果设置long_query_time = 0，则可以记录所有的执行语句，包括回滚的。在正常的查询日志已经包含线程的ID，可以直接进行查询。

### 如何避免MySQL死锁

- 对于有变更操作的应用。有时候，你应当将长事务的操作，拆分成多个较少的事务操作，让锁尽快的释放。另一些情况，因为多个事务，以不同的顺序操作了在一个或多个表中的相同的数据集。此时可以调整这些操作，让它们保持一致的操作顺序或者序列化操作，将死锁变成锁的等待。
- 改变表结构，移除外键，增加索引降低扫描和加锁的行数
- In case of gap locking, you may change transaction isolation level to read committed for the session or transaction to avoid it. But then the binlog format for the session or transaction would have to be ROW or MIXED.你可以调整会话或者事务的隔离级别为`read commited`来避免间隙锁问题。但回话和事务的biglog日志格式需指定为ROW或者MIXED
