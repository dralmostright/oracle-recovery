## Automatic Block Media Recovery in a DataGuard

Starting from Oracle 12.2 in Data Guard Configuration the corrupted data blocks are automatically recovered with uncorrupted copies among the instances participating in the configuration. However there are few things:
1. Oracle Active Data Guard Lisence is required as the standby database must be in Real-Time Query Mode.
2. The pysical standby database must be operating MANAGED REAL TIME APPLY with READ ONLY WITH APPLY mode.

Automatic repair is supported with any Data Guard protection mode and works in two directions but one of the instance participating in Data Guard must have healthy block. When an automatic block repair has been performed, a message is written to the database alert log. If automatic block repair is not possible, an ORA-1578 error is returned.

We will be using below configuration to test this scenario:

#### Data Guard Configuration

<table>
<tr><th>Primary </th><th>Standby</th></tr>
<tr><td>

| Parameter      | Value |
| ----------- | ----------- |
| Version      | 19.3       |
| Database Type   | Standalone        |
| ASM | NO |
|Source host | mysqlvm1.localdomain|
|Source database |orcl|
|Source unique name |orcl|

</td><td>

| Parameter      | Value |
| ----------- | ----------- |
| Version      | 19.3       |
| Database Type   | Standalone        |
|ASM | NO|
|Source host | mysqlvm4.localdomain|
|Source database |orcl|
|Source unique name |orcldr|

</td></tr> </table>

<ins>Step 1: Verify the primary (As we are simulating the block corruption, we are just verifying)</ins>
```
SQL> select name, open_mode,database_role from v$database;

NAME      OPEN_MODE            DATABASE_ROLE
--------- -------------------- ----------------
ORCL      READ WRITE           PRIMARY

SQL> select * from v$database_block_corruption;

no rows selected

SQL> col destination format a60
SQL> col dest_name format a20
SQL> select dest_name,DESTINATION,status from v$archive_dest where destination is not null; 
DEST_NAME            DESTINATION                                                  STATUS
-------------------- ------------------------------------------------------------ ---------
LOG_ARCHIVE_DEST_1   /u01/app/oracle/product/19.0.0.0/dbhome_1/dbs/arch           VALID
LOG_ARCHIVE_DEST_2   orcldr                                                       VALID

SQL>
```
***
<ins>Step 2: Verify Log shipping mode in <strong>Primary</strong> </ins>
```
SQL> select dest_id , recovery_mode from  v$archive_dest_status where recovery_mode!='IDLE';

   DEST_ID RECOVERY_MODE
---------- ----------------------------------
         2 MANAGED REAL TIME APPLY WITH QUERY

SQL>
```
***

<ins>Step 3: Lets Verify the Standby database ot see if its in READ ONLY WITH APPLY and in Real TIME APPLY mode </ins>
```
SQL> select name, open_mode, database_role from v$database;

NAME      OPEN_MODE            DATABASE_ROLE
--------- -------------------- ----------------
ORCL      READ ONLY WITH APPLY PHYSICAL STANDBY

SQL> select * from v$database_block_corruption;

no rows selected

SQL> set lines 222
SQL> col destination format a60
SQL> col dest_name format a20
SQL> select dest_name,DESTINATION,status from v$archive_dest where destination is not null;

DEST_NAME            DESTINATION                                                  STATUS
-------------------- ------------------------------------------------------------ ---------
LOG_ARCHIVE_DEST_1   /u01/app/oracle/product/19.0.0.0/dbhome_1/dbs/arch           VALID
STANDBY_ARCHIVE_DEST /u01/app/oracle/product/19.0.0.0/dbhome_1/dbs/arch           VALID

SQL>

SQL> select dest_id , recovery_mode from  v$archive_dest_status where recovery_mode!='IDLE';

   DEST_ID RECOVERY_MODE
---------- ----------------------------------
         1 MANAGED REAL TIME APPLY

SQL>
SQL> col value format a50
SQL> select name, value from v$dataguard_stats;

NAME                             VALUE
-------------------------------- --------------------------------------------------
transport lag                    +00 00:00:00
apply lag                        +00 00:00:00
apply finish time
estimated startup time           7

SQL>
```
***

<ins>Step 4: Lets identify the object and its block for which we want to simulate corruption (We are doing this in <strong>primary</strong>)</ins>
```
SQL> col table_name format a30
SQL> select table_name,tablespace_name from dba_tables where owner='SUMAN' and table_name='TEST';

TABLE_NAME                     TABLESPACE_NAME
------------------------------ ------------------------------
TEST                           USERS

SQL> col file_name format a60
SQL> set lines 222
SQL> select file_name, tablespace_name from dba_data_files where tablespace_name='USERS';

FILE_NAME                                                    TABLESPACE_NAME
------------------------------------------------------------ ------------------------------
/u01/app/oracle/oradata/ORCL/users01.dbf                     USERS

SQL>
SQL> select * from (select distinct dbms_rowid.rowid_block_number(rowid)  from suman.test);

DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID)
------------------------------------
                                 350
                                 351
                                 349
                                 347

SQL>
```
***

<ins>Step 5: Before simulating lets verify that datafile has no corruption on both primary and standby</ins>
* Primary
```
[oracle@mysqlvm1 ~]$ dbv file=/u01/app/oracle/oradata/ORCL/users01.dbf blocksize=8192

DBVERIFY: Release 19.0.0.0.0 - Production on Sun Dec 17 12:16:59 2023

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

DBVERIFY - Verification starting : FILE = /u01/app/oracle/oradata/ORCL/users01.dbf


DBVERIFY - Verification complete

Total Pages Examined         : 640
Total Pages Processed (Data) : 65
Total Pages Failing   (Data) : 0
Total Pages Processed (Index): 15
Total Pages Failing   (Index): 0
Total Pages Processed (Other): 467
Total Pages Processed (Seg)  : 0
Total Pages Failing   (Seg)  : 0
Total Pages Empty            : 93
Total Pages Marked Corrupt   : 0
Total Pages Influx           : 0
Total Pages Encrypted        : 0
Highest block SCN            : 2186796 (0.2186796)
[oracle@mysqlvm1 ~]$
```
```
[oracle@mysqlvm1 ~]$ rman target /

Recovery Manager: Release 19.0.0.0.0 - Production on Sun Dec 17 12:18:01 2023
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

connected to target database: ORCL (DBID=1683545872)

RMAN> backup validate check logical datafile 7 SECTION SIZE 1024M;

Starting backup at 17-DEC-23
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=85 device type=DISK
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00007 name=/u01/app/oracle/oradata/ORCL/users01.dbf
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
List of Datafiles
=================
File Status Marked Corrupt Empty Blocks Blocks Examined High SCN
---- ------ -------------- ------------ --------------- ----------
7    OK     0              93           641             2194605
  File Name: /u01/app/oracle/oradata/ORCL/users01.dbf
  Block Type Blocks Failing Blocks Processed
  ---------- -------------- ----------------
  Data       0              65
  Index      0              15
  Other      0              467

Finished backup at 17-DEC-23

RMAN>
```
* Standby
```
[oracle@mysqlvm4 ~]$ dbv file=/u01/app/oracle/oradata/ORCL/users01.dbf blocksize=8192

DBVERIFY: Release 19.0.0.0.0 - Production on Sun Dec 17 12:19:11 2023

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

DBVERIFY - Verification starting : FILE = /u01/app/oracle/oradata/ORCL/users01.dbf


DBVERIFY - Verification complete

Total Pages Examined         : 640
Total Pages Processed (Data) : 65
Total Pages Failing   (Data) : 0
Total Pages Processed (Index): 15
Total Pages Failing   (Index): 0
Total Pages Processed (Other): 467
Total Pages Processed (Seg)  : 0
Total Pages Failing   (Seg)  : 0
Total Pages Empty            : 93
Total Pages Marked Corrupt   : 0
Total Pages Influx           : 0
Total Pages Encrypted        : 0
Highest block SCN            : 2186788 (0.2186788)
[oracle@mysqlvm4 ~]$ 
```
```
RMAN> backup validate check logical datafile 7 SECTION SIZE 1024M;

Starting backup at 17-DEC-23
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=67 device type=DISK
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00007 name=/u01/app/oracle/oradata/ORCL/users01.dbf
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
List of Datafiles
=================
File Status Marked Corrupt Empty Blocks Blocks Examined High SCN
---- ------ -------------- ------------ --------------- ----------
7    OK     0              93           641             2194605
  File Name: /u01/app/oracle/oradata/ORCL/users01.dbf
  Block Type Blocks Failing Blocks Processed
  ---------- -------------- ----------------
  Data       0              65
  Index      0              15
  Other      0              467

Finished backup at 17-DEC-23

RMAN>
```
We are good as there is no corruption in primary as well as standby as of now.
***
<ins>Step 6: Lets mark one of the block as corrupt in <strong>Primary</strong> </ins>
```
[oracle@mysqlvm1 ~]$ dd of=/u01/app/oracle/oradata/ORCL/users01.dbf bs=8192 seek=351 conv=notrunc count=1 if=/dev/zero
1+0 records in
1+0 records out
8192 bytes (8.2 kB, 8.0 KiB) copied, 4.2965e-05 s, 191 MB/s
[oracle@mysqlvm1 ~]$
```
Note: This will not be propagated to standby as Database is not aware of this change. Hence the block in standby still will be healthy.
***
<ins>Step 7: Lets verify if above corrupted some blocks or not in <strong>Primary</strong> </ins>
```
[oracle@mysqlvm1 ~]$ dbv file=/u01/app/oracle/oradata/ORCL/users01.dbf blocksize=8192

DBVERIFY: Release 19.0.0.0.0 - Production on Sun Dec 17 12:23:33 2023

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

DBVERIFY - Verification starting : FILE = /u01/app/oracle/oradata/ORCL/users01.dbf
Page 351 is marked corrupt
Corrupt block relative dba: 0x01c0015f (file 7, block 351)
Completely zero block found during dbv:



DBVERIFY - Verification complete

Total Pages Examined         : 640
Total Pages Processed (Data) : 64
Total Pages Failing   (Data) : 0
Total Pages Processed (Index): 15
Total Pages Failing   (Index): 0
Total Pages Processed (Other): 467
Total Pages Processed (Seg)  : 0
Total Pages Failing   (Seg)  : 0
Total Pages Empty            : 93
Total Pages Marked Corrupt   : 1
Total Pages Influx           : 0
Total Pages Encrypted        : 0
Highest block SCN            : 2186796 (0.2186796)
[oracle@mysqlvm1 ~]$
```
Ok now we can see one of the block is corrupted : <strong>Total Pages Marked Corrupt   : 1 </strong>

```
RMAN> backup validate check logical datafile 7 SECTION SIZE 1024M;

Starting backup at 17-DEC-23
using channel ORA_DISK_1
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00007 name=/u01/app/oracle/oradata/ORCL/users01.dbf
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
List of Datafiles
=================
File Status Marked Corrupt Empty Blocks Blocks Examined High SCN
---- ------ -------------- ------------ --------------- ----------
7    FAILED 0              93           641             2186796
  File Name: /u01/app/oracle/oradata/ORCL/users01.dbf
  Block Type Blocks Failing Blocks Processed
  ---------- -------------- ----------------
  Data       0              64
  Index      0              15
  Other      1              468

validate found one or more corrupt blocks
See trace file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_5229.trc for details
Finished backup at 17-DEC-23

RMAN>
```
Validate command also reported some blocks are corrupted.

Now lets verify at database level how may blocks are corrupted.
```
SQL> select name, open_mode, database_role from v$database;

NAME      OPEN_MODE            DATABASE_ROLE
--------- -------------------- ----------------
ORCL      READ WRITE           PRIMARY

SQL> select * from v$database_block_corruption;

     FILE#     BLOCK#     BLOCKS CORRUPTION_CHANGE# CORRUPTIO     CON_ID
---------- ---------- ---------- ------------------ --------- ----------
         7        351          1                  0 ALL ZERO           0

SQL>
```

<ins>Step 8: Ok now how to fix it. Its simple we just need to query and fetch the data from corrputed blocks.</ins>
```
SQL> alter system flush buffer_cache;

System altered.

SQL> select count(*) from suman.test;

  COUNT(*)
----------
       185

SQL> select * from v$database_block_corruption;

     FILE#     BLOCK#     BLOCKS CORRUPTION_CHANGE# CORRUPTIO     CON_ID
---------- ---------- ---------- ------------------ --------- ----------
         7        351          1                  0 ALL ZERO           0

SQL> select name, open_mode, database_role from v$database;

NAME      OPEN_MODE            DATABASE_ROLE
--------- -------------------- ----------------
ORCL      READ WRITE           PRIMARY

SQL>
```
Ok while querying we didn't encounter any errors or we are not reported it has corrputed blocks and its due to Automatic Block Media Recovery in a DataGuard feature. And during this operation oracle will throw some messages in alert log as below:

```
Corrupt block relative dba: 0x01c0015f (file 7, block 351)
Completely zero block found during multiblock buffer read

Reading datafile '/u01/app/oracle/oradata/ORCL/users01.dbf' for corrupt data at rdba: 0x01c0015f (file 7, block 351)
Reread (file 7, block 351) found same corrupt data (no logical check)
Automatic block media recovery requested for (file# 7, block# 351)
2023-12-17T12:44:55.769222+05:45
Corrupt Block Found
         TIME STAMP (GMT) = 12/17/2023 12:44:55
         CONT = 0, TSN = 4, TSNAME = USERS
         RFN = 7, BLK = 351, RDBA = 29360479
         OBJN = 73231, OBJD = 73231, OBJECT = TEST, SUBOBJECT =
         SEGMENT OWNER = SUMAN, SEGMENT TYPE = Table Segment
2023-12-17T12:44:55.871650+05:45
Automatic block media recovery successful for (file# 7, block# 351)
2023-12-17T12:44:55.872183+05:45
Automatic block media recovery successful for (file# 7, block# 351)
```

Eventhough the block is repaired why are we still seeing block corrupt rows in v$database_block_corruption, for this we will need to validate the block from RMAN and lets see if actually resolved the corruption.
```
[oracle@mysqlvm1 ~]$ dbv file=/u01/app/oracle/oradata/ORCL/users01.dbf blocksize=8192

DBVERIFY: Release 19.0.0.0.0 - Production on Sun Dec 17 12:48:09 2023

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

DBVERIFY - Verification starting : FILE = /u01/app/oracle/oradata/ORCL/users01.dbf


DBVERIFY - Verification complete

Total Pages Examined         : 640
Total Pages Processed (Data) : 65
Total Pages Failing   (Data) : 0
Total Pages Processed (Index): 15
Total Pages Failing   (Index): 0
Total Pages Processed (Other): 467
Total Pages Processed (Seg)  : 0
Total Pages Failing   (Seg)  : 0
Total Pages Empty            : 93
Total Pages Marked Corrupt   : 0
Total Pages Influx           : 0
Total Pages Encrypted        : 0
Highest block SCN            : 2194605 (0.2194605)
[oracle@mysqlvm1 ~]$
```
DBV says the datafile is green. Now lets verify from RMAN too.
```
RMAN> backup validate check logical datafile 7 SECTION SIZE 1024M;

Starting backup at 17-DEC-23
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=61 device type=DISK
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00007 name=/u01/app/oracle/oradata/ORCL/users01.dbf
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
List of Datafiles
=================
File Status Marked Corrupt Empty Blocks Blocks Examined High SCN
---- ------ -------------- ------------ --------------- ----------
7    OK     0              93           641             2194605
  File Name: /u01/app/oracle/oradata/ORCL/users01.dbf
  Block Type Blocks Failing Blocks Processed
  ---------- -------------- ----------------
  Data       0              65
  Index      0              15
  Other      0              467

Finished backup at 17-DEC-23

RMAN>
```
RMAN to says there are not corrputed blocks in datafile. Lets verify in database too.
```
[oracle@mysqlvm1 ~]$ sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Sun Dec 17 12:50:04 2023
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> select name, open_mode, database_role from v$database;

NAME      OPEN_MODE            DATABASE_ROLE
--------- -------------------- ----------------
ORCL      READ WRITE           PRIMARY

SQL> select * from v$database_block_corruption;

no rows selected

SQL>
```
Database side also the rows which were reporting block corruption are gone.
