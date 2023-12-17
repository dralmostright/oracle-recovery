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
<ins>Step 3: Lets identify the object and its block for which we want to simulate corruption </ins>
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
<ins>Step 2: </ins>

![image description](imgs/primary-2.png)
![image description](imgs/primary-3.png)
![image description](imgs/primary-4.png)
![image description](imgs/primary-5.png)