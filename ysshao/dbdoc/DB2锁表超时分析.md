

## 锁定超时分析思路

### 关于Cur_Commit参数

#### 名词解释

看一下ibm官方文档对该参数的描述：“当前已落实”（currently committed semantics，简称 CC）新特性，该新特性的显著特点是在游标稳定性（Cursor stability，简称 CS）隔离级别时可以明显减少锁等待的出现，以及死锁的出现频率。

数据库配置参数 cur_commit：

该数据库配置参数主要是用来控制游标稳定性扫描的行为，默认值为 ON，可选值为：

（1）ON ：打开；

对于新创建的数据库，默认值是 ON，在此情况下，当你试图读取一个正在被其他应用程序修改的行时，将直接返回该行的当前已落实版本数据（也就是首次更改之前的值）。

（2）AVAILABLE：可用；

此值表示你的应用需要显式地请求“当前已落实行为”才能得到“当前已落实”结果。

（3）DISABLED：禁用；

如果数据库是从之前的版本升级而来，这个参数将被设置成 DISABLED，这是为了和以前版本的行为保持一致。如果你希望使用当前已落实来控制游标稳定性扫描的行为，需要将这个参数更改成 ON 。

需要注意的是，注册表变量 DB2_EVALUNCOMMITTED、DB2_SKIPDELETED 和 DB2_SKIPINSERTED 在启用 cur_commit 参数后会受到影响。在绑定（BIND）或预编译（PRECOMPILE）时对 CONCURRENTACCESSRESOLUTION 选项指定 USE CURRENTLY COMMITTED 或 WAIT FOR OUTCOME，那么注册表变量 DB2_EVALUNCOMMITTED、DB2_SKIPDELETED 和 DB2_SKIPINSERTED 将被忽略。

#### 案例

当我们将 cur_commit的值禁用之后的案例：

```sql
db2 update db cfg using cur_commit off  --禁用cur_commit
```

注意：对于这些配置参数，必须在所有应用程序都与此数据库断开连接之后，更改才会生效。

```sql
db2 force application all  --断开连接
```

session1

```sql
 db2 +c "UPDATE CUSTOMER_INFO SET CUSTOMERID='200001' WHERE CUSTOMERID='1001'"
```

注： +c代表禁用自动提交

session2

```sql
db2 +c "SELECT * FROM CUSTOMER_INFO WHERE CUSTOMERID='1001'";
```

此时，就会出现锁等待，session2就会出现以下错误：

```sql
SQL0911N  The current transaction has been rolled back because of a deadlock 
or timeout.  Reason code "68".  SQLSTATE=40001
```

### DB2_CAPTURE_LOCKTIMEOUT 参数 

注：在进行此参数验证时，我将cur_commit的值也禁用了，主要是为了例子的复用

在 DB2 9.5 之后的锁定超时报告功能中，引入了一个新特性，使得锁定超时分析变得非常简单。要激活锁定超时报告，只需将 DB2 注册变量 DB2_CAPTURE_LOCKTIMEOUT 设置为 ON，并重新启动您的 DB2 实例：

```sql
db2set DB2_CAPTURE_LOCKTIMEOUT=ON
db2stop
db2start
```

当 DB2_CAPTURE_LOCKTIMEOUT 设置为 ON 时，DB2 为每个锁定超时事件自动创建一个报告文件。报告文件保存在 DIAGPATH 数据库管理员配置（DBM CFG）参数指向的目录（DIAGPATH）中，包含的信息有：锁定超时的日期和时间、存在问题的锁定、锁定请求程序和锁定拥有者。

#### 设置LOCKTIMEOUT 

只有当数据库配置（DB CFG）参数 LOCKTIMEOUT 未被设置为 -1 时，锁定超时才会发生。-1 意味着一个应用程序可能在无限期地等待一个必需的锁定，在一些情形中，这并不是期望发生的行为，但是 -1 是 LOCKTIMEOUT 的默认设置。假设 LOCKTIMEOUT 被更改为 20 秒（LOCKTIMEOUT 的值以秒为单位）：

```sql
db2 "UPDATE DB CFG FOR DBTEST USING LOCKTIMEOUT 20"
```

session1

```sql
 db2 +c "SELECT * FROM CUSTOMER_INFO WHERE CUSTOMERID='1001'";
```

session2

```sql
db2 +c "UPDATE CUSTOMER_INFO SET CUSTOMERID='200001' WHERE CUSTOMERID='1001'"
```

通过之前的学习，我们可知，此时出现锁等待。因此当我们开启DB2_CAPTURE_LOCKTIMEOUT参数后，就会在DIAGPATH目录下产生一个文件：

```sql

LOCK TIMEOUT REPORT

Date:               31/07/2020
Time:               15:40:48
Instance:           db2iv97
Database:           DBTEST
Database Partition: 0


Lock Information: 

   Lock Name:      04000400000000000000000054
   Lock Type:      Table
   Lock Specifics: Tablespace ID=4, Table ID=4


Lock Requestor: 
   System Auth ID:          DB2IV97 
   Application Handle:      [0-18]
   Application ID:          *LOCAL.db2iv97.200731014911
   Application Name:        db2bp
   Requesting Agent ID:     40
   Coordinator Agent ID:    40
   Coordinator Partition:   0
   Lock timeout Value:      20000 milliseconds
   Lock mode requested:     .IX
   Application Status:      (SQLM_UOWEXEC)
   Current Operation:       (SQLM_EXECUTE_IMMEDIATE)
   Lock Escalation:         No

   Context of Lock Request: 
   Identification:            UOW ID (29); Activity ID (1)
   Activity Information:      
      Package Schema:         (NULLID  )
      Package Name:           (SQLC2H23NULLID  )
      Package Version:        ()
      Section Entry Number:   203
      SQL Type:               Dynamic
      Statement Type:         DML, Insert/Update/Delete
      Effective Isolation:    Cursor Stability
      Statement Unicode Flag: No
      Statement:              UPDATE CUSTOMER SET CUSTOMERID=:L0 WHERE CUSTOMERID=:L1 


Lock Owner (Representative):  
   System Auth ID:          DB2IV97 
   Application Handle:      [0-605]
   Application ID:          192.168.72.116.10340.200731030848
   Application Name:        db2jcc_application
   Requesting Agent ID:     146
   Coordinator Agent ID:    146
   Coordinator Partition:   0
   Lock mode held:          ..S

   List of Active SQL Statements:  Not available

   List of Inactive SQL Statements from current UOW: Not available
```

锁定超时报告包括 4 部分：

- 第一部分提供与锁定超时发生的日期和时间，以及相应的实例和数据库相关的信息。
- *Lock Information* 部分显示导致锁定超时的锁。除了内部锁名以外，还会显示锁的类型（行锁或表锁）和表信息。要确定表名称，需要使用给定的表空间 ID 和表 ID 查询编目视图 SYSCAT.TABLES
- 发生锁定超时的应用程序在 *Lock Requestor* 部分中描述。这部分还包含一些更有趣的条目：用于连接到数据库的身份验证 ID、应用程序名称、请求的锁模式（以及该锁是否由一个锁升级引起）、需要锁的语句的隔离级别，以及 SQL 语句文本本身。
- 最后一部分 *Lock Owner (Representative)* 列出持有有问题的锁的应用程序。使用另一个应用程序，还可以查看身份验证 ID、应用程序名称和锁模式等信息。在一些情形下，可能不止一个应用程序持有（共享）一个锁，这会阻塞对锁的请求。在这些情形下，锁定超时报告中只会显示一个锁持有者。这就是这部分具有额外的 *Representative* 的原因。

#### 死锁事件监视器（锁等待）

为了获得锁持有者的应用程序执行的 SQL 语句的信息，我们使用 DETAILS HISTORY 选项创建一个死锁事件监视器并激活它。例如，可以通过如下方法创建一个恰当的死锁事件监视器并将其激活：

```sql
-- 连接数据库
db2 "connect to 【dbname】"
-- 创建事件监听 path为路径
db2 "create event monitor stmtmon for deadlocks with details history  write to file 'path'"
-- 启动事件监听
db2 "set event monitor stmtmon state=1"
-- 切换至应用程序进行数据库操作
----
----
----
-- 停止事件监听
db2 "set event monitor stmtmon state=0"
-- 删除事件监听
db2 "drop event monitor stmtmon"
```

激活了死锁事件监视器之后，再次运行上面描述的锁定超时场景。这次 DB2 编写一个锁定超时报告，此时因为这个场景是锁等待现象，所以在path路径下生成的文件当中分析不出来任何结果，只有当出现死锁时，path路径下的文件才有意义。

session1

```sql
 db2 +c "SELECT * FROM CUSTOMER_INFO WHERE CUSTOMERID='1001'";
```

session2

```sql
db2 +c "UPDATE CUSTOMER_INFO SET CUSTOMERID='200001' WHERE CUSTOMERID='1001'"
```

```sql

LOCK TIMEOUT REPORT

Date:               31/07/2020
Time:               15:43:51
Instance:           db2iv97
Database:           DBTEST
Database Partition: 0


Lock Information: 

   Lock Name:      04000400060000000000000052
   Lock Type:      Row
   Lock Specifics: Tablespace ID=4, Table ID=4, Row ID=x0600000000000000


Lock Requestor: 
   System Auth ID:          DB2IV97 
   Application Handle:      [0-2108]
   Application ID:          *LOCAL.db2iv97.200731073159
   Application Name:        db2bp
   Requesting Agent ID:     93
   Coordinator Agent ID:    93
   Coordinator Partition:   0
   Lock timeout Value:      20000 milliseconds
   Lock mode requested:     .NS
   Application Status:      (SQLM_UOWEXEC)
   Current Operation:       (SQLM_FETCH)
   Lock Escalation:         No

   Context of Lock Request: 
   Identification:            UOW ID (5); Activity ID (1)
   Activity Information:      
      Package Schema:         (NULLID  )
      Package Name:           (SQLC2H23NULLID  )
      Package Version:        ()
      Section Entry Number:   201
      SQL Type:               Dynamic
      Statement Type:         DML, Select (blockable)
      Effective Isolation:    Cursor Stability
      Statement Unicode Flag: No
      Statement:              SELECT * FROM CUSTOMER WHERE CUSTOMERID=:L0 


Lock Owner (Representative):  
   System Auth ID:          DB2IV97 
   Application Handle:      [0-18]
   Application ID:          *LOCAL.db2iv97.200731014911
   Application Name:        db2bp
   Requesting Agent ID:     40
   Coordinator Agent ID:    40
   Coordinator Partition:   0
   Lock mode held:          ..X

   List of Active SQL Statements:  Not available

   List of Inactive SQL Statements from current UOW:  

      Entry:                  #1
      Identification:         UOW ID (30); Activity ID (1)
      Package Schema:         (NULLID  )
      Package Name:           (SQLC2H23)
      Package Version:        ()
      Section Entry Number:   203
      SQL Type:               Dynamic
      Statement Type:         DML, Insert/Update/Delete
      Effective Isolation:    Cursor Stability
      Statement Unicode Flag: No
      Statement:              UPDATE CUSTOMER SET CUSTOMERID=:L0 WHERE CUSTOMERID=:L1 
```

这个锁定超时报告的开始部分与前面看到的相同。但是，这次的 *Lock Owner* 部分包含额外的、有价值的信息。在标题 *List of Inactive SQL Statements from current UOW* 下边，可以看到在发生锁定超时之前锁持有者的事务执行的所有 SQL 语句。从这组 SQL 语句中，可以找到导致问题锁定的语句。

#### 使用锁定超时报告的提示

带有语句历史功能的死锁事件监视器适用于所有应用程序，会增加 DB2 数据库管理程序对监视器堆的大量使用。所以应该谨慎使用。您应该始终首先设置 DB2_CAPTURE_LOCKTIMEOUT=ON，然后只在必要的时候使用 DETAILS HISTORY 选项激活死锁事件监视器。

使用锁定超时报告时，DIAGPATH 中的锁定超时报告文件的数量在持续增加。DB2 不会删除这些报告文件，所以 DBA 需要删除它们或者将它们移动到不同的位置，以便在 DIAGPATH 的文件系统上始终有足够的空间。



#### 死锁事件监视器（死锁）

当死锁监视器开启之后，会在path路径下产生文件，死锁监视器开始方式见上面。

产生的文件：

```
-rw-r--r-- 1 db2iv97 db2iadm1  79718 Aug  3 10:11 00000000.evt
-rw-r----- 1 db2iv97 db2iadm1     32 Aug  3 10:06 db2event.ctl
```

下面我们模拟一个死锁场景。

session 1

```sql
db2 +c "INSERT INTO CUSTOMER_INFO(CUSTOMERID, CUSTOMERNAME) VALUES('1004', '1114')";
```

session 2

```sql
db2 +c "INSERT INTO CUSTOMER(CUSTOMERID, CUSTOMERNAME) VALUES('1005', '1115')";
```

session 1

```sql
db2 +c "select * from CUSTOMER"
```

session 2

```sql
db2 +c "select * from CUSTOMER_INFO"
```

不用多少时间，就会发现：

session 1

```sql
[db2iv97@dev_20161229 event]$ db2 +c "select * from CUSTOMER"
SQL0911N  The current transaction has been rolled back because of a deadlock 
or timeout.  Reason code "2".  SQLSTATE=40001
```

session 2

```sql
[db2iv97@dev_20161229 ~]$ db2 +c "select * from CUSTOMER_INFO"

CUSTOMERID           CUSTOMERNAME                                                                    
-------------------- -----------------------------
1001                 1111                                                                            
1002                 1112                                                                            
1003                 1113                                                                            

  3 record(s) selected.
```

已经出现了死锁，并且被死锁管理器选择一个application干掉了，解开死锁。

此时我们整理监听事件内容：

```shell
db2evmon -path /home/db2iv97/event > dlockmon.txt
```

```
-rw-r--r-- 1 db2iv97 db2iadm1  7626 Aug  3 10:11 00000000.evt
-rw-r----- 1 db2iv97 db2iadm1    32 Aug  3 10:06 db2event.ctl
-rw-r--r-- 1 db2iv97 db2iadm1 12590 Aug  3 10:13 dlockmon.txt
```

分析dlockmon.txt

```sql
--------------------------------------------------------------------------
                            EVENT LOG HEADER
  Event Monitor name: STMTMON
  Server Product ID: SQL09076
  Version of event monitor data: 11
  Byte order: LITTLE ENDIAN
  Number of nodes in db2 instance: 1
  Codepage of database: 1386
  Territory code of database: 86
  Server instance name: db2iv97
--------------------------------------------------------------------------

--------------------------------------------------------------------------
  Database Name: DBTEST  
  Database Path: /home/db2iv97/tablespace/dbtest/db2iv97/NODE0000/SQL00001/
  First connection timestamp: 08/03/2020 09:20:25.282915
  Event Monitor Start time:   08/03/2020 14:39:06.265864
--------------------------------------------------------------------------

3) Deadlock Event ...
  Deadlock ID:   2   ------>decdlock ID是2
  Number of applications deadlocked: 2   ----->参与deadlock应用2个
  Deadlock detection time: 08/03/2020 14:40:25.220439
  Rolled back Appl participant no: 2  ------>回滚的应用参与者编号是2
  Rolled back Appl Id: *LOCAL.db2iv97.200803063556-->回滚的应用id是                                                                          *LOCAL.db2iv97.200803063556
  Rolled back Appl seq number: 00008

5) Deadlocked Connection ...
  Deadlock ID:   2
  Participant no.: 2
  Participant no. holding the lock: 1
  Appl Id: *LOCAL.db2iv97.200803063556  -->回滚的应用id
  Appl Seq number: 00008
 Appl Id of connection holding the lock: *LOCAL.db2iv97.200803012025--拥有锁的应用appid
  Seq. no. of connection holding the lock: 00001
  Lock wait start time: 08/03/2020 14:40:18.180577
  Lock Name       : 0x04000400090000000000000052
  Lock Attributes : 0x00000000
  Release Flags   : 0x40000000
  Lock Count      : 1
  Hold Count      : 0
  Current Mode    : none
  Deadlock detection time: 08/03/2020 14:40:25.220530
  Table of lock waited on      : CUSTOMER
  Schema of lock waited on     : DB2IV97 
  Data partition id for table  : 0
  Tablespace of lock waited on : DBTESTSPACE
  Type of lock: Row
  Mode of lock: X   - Exclusive
  Mode application requested on lock: NS  - Share (CS/RS)
  Node lock occured on: 0
  Lock object name: 9
  Application Handle: 11935
  Deadlocked Statement:
    Type     : Dynamic
    Operation: Fetch
    Section  : 201
    Creator  : NULLID
    Package  : SQLC2H23
    Cursor   : SQLCUR201
    Cursor was blocking: FALSE
    Text     : select * from CUSTOMER
  List of Locks:
      Lock Name                   : 0x01000000010000000100C02456
      Lock Attributes             : 0x00000000
      Release Flags               : 0x40000000
      Lock Count                  : 1
      Hold Count                  : 0
      Lock Object Name            : 0
      Object Type                 : Internal - Variation
      Data partition id           : -1
      Mode                        : S   - Share

      Lock Name                   : 0x04000500070000000000000052
      Lock Attributes             : 0x00000008
      Release Flags               : 0x40000000
      Lock Count                  : 1
      Hold Count                  : 0
      Lock Object Name            : 7
      Object Type                 : Row
      Tablespace Name             : DBTESTSPACE
      Table Schema                : DB2IV97 
      Table Name                  : CUSTOMER_INFO
      Data partition id           : 0
      Mode                        : X   - Exclusive

      Lock Name                   : 0x41414141414242636F724AC241
      Lock Attributes             : 0x00000000
      Release Flags               : 0x40000000
      Lock Count                  : 1
      Hold Count                  : 0
      Lock Object Name            : 0
      Object Type                 : Internal - Plan
      Data partition id           : -1
      Mode                        : S   - Share

      Lock Name                   : 0x04000400000000000000000054
      Lock Attributes             : 0x00000000
      Release Flags               : 0x40000000
      Lock Count                  : 1
      Hold Count                  : 0
      Lock Object Name            : 4
      Object Type                 : Table
      Tablespace Name             : DBTESTSPACE
      Table Schema                : DB2IV97 
      Table Name                  : CUSTOMER
      Data partition id           : 0
      Mode                        : IS  - Intent Share

      Lock Name                   : 0x04000500000000000000000054
      Lock Attributes             : 0x00000000
      Release Flags               : 0x40000000
      Lock Count                  : 1
      Hold Count                  : 0
      Lock Object Name            : 5
      Object Type                 : Table
      Tablespace Name             : DBTESTSPACE
      Table Schema                : DB2IV97 
      Table Name                  : CUSTOMER_INFO
      Data partition id           : 0
      Mode                        : IX  - Intent Exclusive

  Locks Held:    5
  Locks in List: 5
  Locks Displayed: 5

6) Deadlock statement history ...
  Deadlock ID             : 2
  Participant No          : 2
  Stmt history ID         : 2
  Type                    : Dynamic
  Section No              : 201
  Package cache id        : 1262720385025
  Package creator         : NULLID  
  Package name            : SQLC2H23
  Package version         : 
  Lock timeout value      : 20
  Nesting level of stmt   : 0
  Invocation ID           : 0
  Query ID                : 0
  Source ID               : 0
  UOW Sequence number     : 0008
  Isolation level         : Cursor Stability
  Stmt first use time     : 08/03/2020 14:40:18.180543
  Stmt last use time      : 08/03/2020 14:40:18.180543
  Statement text          : select * from CUSTOMER

7) Deadlock statement history ...
  Deadlock ID             : 2
  Participant No          : 2
  Stmt history ID         : 1
  Type                    : Dynamic
  Section No              : 203
  Package cache id        : 3685081939969
  Package creator         : NULLID  
  Package name            : SQLC2H23
  Package version         : 
  Lock timeout value      : 20
  Nesting level of stmt   : 0
  Invocation ID           : 0
  Query ID                : 0
  Source ID               : 0
  UOW Sequence number     : 0008
  Isolation level         : Cursor Stability
  Stmt first use time     : 08/03/2020 14:39:49.368071
  Stmt last use time      : 08/03/2020 14:39:49.368071
  Statement text          : INSERT INTO CUSTOMER_INFO(CUSTOMERID, CUSTOMERNAME) VALUES(:L0 , :L1 )

8) Connection Header Event ...
  Appl Handle: 10159
  Appl Id: *LOCAL.db2iv97.200803012025
  Appl Seq number: 00003
  DRDA AS Correlation Token: *LOCAL.db2iv97.200803012025
  Program Name    : db2bp
  Authorization Id: DB2IV97 
  Execution Id    : db2iv97
  Codepage Id: 1386
  Territory code: 1
  Client Process Id: 1173
  Client Database Alias: DBTEST
  Client Product Id: SQL09076
  Client Platform: Linux/X8664
  Client Communication Protocol: Local
  Client Network Name: dev_20161229
  Connect timestamp: 08/03/2020 09:20:25.282915

9) Deadlocked Connection ...
  Deadlock ID:   2
  Participant no.: 1
  Participant no. holding the lock: 2
  Appl Id: *LOCAL.db2iv97.200803012025
  Appl Seq number: 00003
  Appl Id of connection holding the lock: *LOCAL.db2iv97.200803063556
  Seq. no. of connection holding the lock: 00001
  Lock wait start time: 08/03/2020 14:40:18.990759
  Lock Name       : 0x04000500070000000000000052
  Lock Attributes : 0x00000000
  Release Flags   : 0x40000000
  Lock Count      : 1
  Hold Count      : 0
  Current Mode    : none
  Deadlock detection time: 08/03/2020 14:40:25.220801
  Table of lock waited on      : CUSTOMER_INFO
  Schema of lock waited on     : DB2IV97 
  Data partition id for table  : 0
  Tablespace of lock waited on : DBTESTSPACE
  Type of lock: Row
  Mode of lock: X   - Exclusive
  Mode application requested on lock: NS  - Share (CS/RS)
  Node lock occured on: 0
  Lock object name: 7
  Application Handle: 10159
  Deadlocked Statement:
    Type     : Dynamic
    Operation: Fetch
    Section  : 201
    Creator  : NULLID
    Package  : SQLC2H23
    Cursor   : SQLCUR201
    Cursor was blocking: FALSE
    Text     : select * from CUSTOMER_INFO
  List of Locks:
      Lock Name                   : 0x01000000010000000100002B56
      Lock Attributes             : 0x00000000
      Release Flags               : 0x40000000
      Lock Count                  : 1
      Hold Count                  : 0
      Lock Object Name            : 0
      Object Type                 : Internal - Variation
      Data partition id           : -1
      Mode                        : S   - Share

      Lock Name                   : 0x04000400090000000000000052
      Lock Attributes             : 0x00000008
      Release Flags               : 0x40000000
      Lock Count                  : 1
      Hold Count                  : 0
      Lock Object Name            : 9
      Object Type                 : Row
      Tablespace Name             : DBTESTSPACE
      Table Schema                : DB2IV97 
      Table Name                  : CUSTOMER
      Data partition id           : 0
      Mode                        : X   - Exclusive

      Lock Name                   : 0x41414141414242636F724AC241
      Lock Attributes             : 0x00000000
      Release Flags               : 0x40000000
      Lock Count                  : 1
      Hold Count                  : 0
      Lock Object Name            : 0
      Object Type                 : Internal - Plan
      Data partition id           : -1
      Mode                        : S   - Share

      Lock Name                   : 0x04000400000000000000000054
      Lock Attributes             : 0x00000000
      Release Flags               : 0x40000000
      Lock Count                  : 1
      Hold Count                  : 0
      Lock Object Name            : 4
      Object Type                 : Table
      Tablespace Name             : DBTESTSPACE
      Table Schema                : DB2IV97 
      Table Name                  : CUSTOMER
      Data partition id           : 0
      Mode                        : IX  - Intent Exclusive

      Lock Name                   : 0x04000500000000000000000054
      Lock Attributes             : 0x00000000
      Release Flags               : 0x40000000
      Lock Count                  : 1
      Hold Count                  : 0
      Lock Object Name            : 5
      Object Type                 : Table
      Tablespace Name             : DBTESTSPACE
      Table Schema                : DB2IV97 
      Table Name                  : CUSTOMER_INFO
      Data partition id           : 0
      Mode                        : IS  - Intent Share

  Locks Held:    5
  Locks in List: 5
  Locks Displayed: 5

10) Deadlock statement history ...
  Deadlock ID             : 2
  Participant No          : 1
  Stmt history ID         : 2
  Type                    : Dynamic
  Section No              : 201
  Package cache id        : 1477468749825
  Package creator         : NULLID  
  Package name            : SQLC2H23
  Package version         : 
  Lock timeout value      : 20
  Nesting level of stmt   : 0
  Invocation ID           : 0
  Query ID                : 0
  Source ID               : 0
  UOW Sequence number     : 0003
  Isolation level         : Cursor Stability
  Stmt first use time     : 08/03/2020 14:40:18.990731
  Stmt last use time      : 08/03/2020 14:40:18.990731
  Statement text          : select * from CUSTOMER_INFO

11) Deadlock statement history ...
  Deadlock ID             : 2
  Participant No          : 1
  Stmt history ID         : 1
  Type                    : Dynamic
  Section No              : 203
  Package cache id        : 1675037245441
  Package creator         : NULLID  
  Package name            : SQLC2H23
  Package version         : 
  Lock timeout value      : 20
  Nesting level of stmt   : 0
  Invocation ID           : 0
  Query ID                : 0
  Source ID               : 0
  UOW Sequence number     : 0003
  Isolation level         : Cursor Stability
  Stmt first use time     : 08/03/2020 14:39:58.383474
  Stmt last use time      : 08/03/2020 14:39:58.383474
  Statement text          : INSERT INTO CUSTOMER(CUSTOMERID, CUSTOMERNAME) VALUES(:L0 , :L1 )
```

