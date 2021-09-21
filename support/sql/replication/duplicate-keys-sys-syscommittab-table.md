---
title: Duplicate keys from sys.syscommittab table
description: This article provides information about resolving a SQL Server Change Tracking issue that can result in duplicate rows in `sys.syscommittab` file.
ms.date: 03/16/2020
ms.prod-support-area-path:
ms.prod: sql
---
# Duplicate key rows from the sys.syscommittab table in SQL Server

This article provides information about resolving a SQL Server Change Tracking issue that can result in duplicate rows in `sys.syscommittab` file.

_Original product version:_ &nbsp; SQL Server 2008 and the later versions  
_Original KB number:_ &nbsp; 3083381

## Symptoms

When you compare the in-memory `SYSCOMMITTABLE` and the on-disk `sys.syscommittab` file in Microsoft SQL Server, you may see duplicate key rows. These duplicate values may cause backup and checkpoint operations to fail.

> "Cannot insert duplicate key row in object 'sys.syscommittab' with unique index 'si_xdes_id'. The duplicate key value is (KeyValue).  
> Error: 3999, Severity: 17, State: 1.  
> Failed to flush the commit table to disk in dbidDatabaseID due to error 2601. Check the errorlog for more information."

## Cause

This problem occurs because of a known issue in SQL Server change tracking.

## Resolve factors that cause the duplicate keys

To resolve the factors that cause the duplicate keys, apply one of the following fixes, as appropriate for your situation:

- [FIX: A backup operation on a SQL Server 2008 or SQL Server 2008 R2 database fails if you enable change tracking on this database](https://support.microsoft.com/help/2522893)

- [FIX: Backup fails in SQL Server 2008, SQL Server 2008 R2, or SQL Server 2012 if you enable change tracking on the database](https://support.microsoft.com/kb/2603910)

- [FIX: Backup operation fails in a SQL Server 2008, SQL Server 2008 R2, or SQL Server 2012 database after you enable change tracking](https://support.microsoft.com/kb/2682488)

Although these fixes prevent duplicate key rows from continuing to appear, they don't automatically remove the duplicate rows. Without removing the duplicate rows, the affected database can't complete database checkpoints, and backups may fail.

## Disable and enable change tracking to remove duplicate rows

1. Disable change tracking on the affected tables and database.
2. Issue a manual database Checkpoint.
3. Enable change tracking on the affected database and tables.

For more information about change tracking, see [Enable and disable change tracking](/sql/relational-databases/track-changes/enable-and-disable-change-tracking-sql-server). For issuing a manual Checkpoint, see [CHECKPOINT (Transact-SQL)](/sql/t-sql/language-elements/checkpoint-transact-sql).

## Manually delete the duplicate rows

1. Copy the Transact-SQL script at the end of the **Transact-SQL script** section into a text editor.
2. Locate the `<AFFECTED_DB>` placeholder in the script, and replace it with the name of the affected database.
3. Save the modified script to your hard disk as a .sql file. For example, `C:\temp\remove_duplicates.sql`.

If you're running SQL Server 2014, you must grant the per-Service SID full control to the `mssqlsystemresource.ldf` and `mssqlsystemresource.mdf` files. To do this, follow these steps:

1. Navigate to the Bin directory that corresponds to your Instance ID. For example:  
`C:\Program Files\Microsoft SQL Server\<Instance ID>\MSSQL\Binn`

2. Open the properties for `mssqlsystemresource.ldf` and `mssqlsystemresource.mdf`, and then click the **Security** tab.
3. Locate the SQL Server service per-Service SID, and note the default permissions:

    - `*Read & execute`
    - `*Read`

4. Grant the SQL Server service per-Service SID Full Control, and then close the permissions dialog boxes.
5. Start SQL Server in Single-User mode. For more information, see [Start SQL Server in Single-User mode](/sql/database-engine/configure-windows/start-sql-server-in-single-user-mode).
6. Use a `sqlcmd` command line to connect to SQL Server under the Dedicated Administrator Connection (DAC). For example:

    ```console
    sqlcmd -S PRODSERV1\MSSQLSERVER -A -E -i c:\temp\remove_duplicates.sql
    ```

   Then, execute the modified Transact-SQL script.

7. [Restart SQL Server in Multi-User mode](/sql/database-engine/configure-windows/start-stop-pause-resume-restart-sql-server-services), and then verify that backup and CHECKPOINT operations against the affected database complete successfully. If step 4 was used, revert the permissions to the default values.

## Transact-SQL script

```sql
/*
Create a temporary database to store the necessary rows
required to remove the duplicate data
*/
if exists(select 1 from sys.databases where name = 'dbChangeTrackingMetadata')
begin
drop database dbChangeTrackingMetadata
end
go
create database dbChangeTrackingMetadata
go

--Table to store the contents of the SYSCOMMITTABLE
use dbChangeTrackingMetadata
go
create table dbo.t_SYSCOMMITTABLE (
commit_ts bigint
,xdes_id bigint
,commit_lbn bigint
,commit_csn bigint
,commit_time datetime
)
go

/*Table to store the duplicate rows to be removed from the
 sys.syscommittab table
*/
create table dbo.t_syscommittab (
commit_ts bigint
,xdes_id bigint
,commit_lbn bigint
,commit_csn bigint
,commit_time datetime
,dbfragid int
)
go

--Enable the usage of OPENROWSET
exec sys.sp_setbuildresource 1
go

--Change <AFFECTED_DB> to the database that contains the duplicate values
USE <AFFECTED DB>
go
declare @rowcount bigint
SET @rowcount = 0

--Copy all rows from the SYSCOMMITTABLE into the temporary database
insert into dbChangeTrackingMetadata.dbo.t_SYSCOMMITTABLE
SELECT commit_ts, xdes_id, commit_lbn, commit_csn, commit_time
FROM OpenRowset (table SYSCOMMITTABLE, db_id (), 0, 0)

--Save the duplicate values into the temporary database
insert into dbChangeTrackingMetadata.dbo.t_syscommittab
select ondisk_ct.* from sys.syscommittab as ondisk_ct
join dbChangeTrackingMetadata.dbo.t_SYSCOMMITTABLE as inmem_ct
on ondisk_ct.xdes_id = inmem_ct.xdes_id

--Delete the duplicate values
delete from sys.syscommittab
where xdes_id in ( select xdes_id from dbChangeTrackingMetadata.dbo.t_syscommittab )
set @rowcount = @@rowcount
if (@rowcount > 0)
begin
print ''
print 'DELETED '+CAST(@rowcount as NVARCHAR(10))+
' rows from sys.syscommittab that were also stored in SYSCOMMITTABLE'
print ''
end
else
begin
print ''
print 'Failed to DELETE DUP rows from sys.syscommittab'
print ''
end
exec sys.sp_setbuildresource 0
go
```

##For SQL Server 2019, If the issue is not getting resolved by following above steps. You can try with the below steps for the error:

Msg 2601, Level 14, State 1, Procedure sys.sp_flush_commit_table, Line 16 [Batch Start Line 7]
Cannot insert duplicate key row in object 'sys.syscommittab' with unique index 'si_xdes_id'. The duplicate key value is (49979609963).
The statement has been terminated.
Msg 2601, Level 14, State 1, Procedure sys.sp_flush_commit_table, Line 16 [Batch Start Line 7]
Cannot insert duplicate key row in object 'sys.syscommittab' with unique index 'si_xdes_id'. The duplicate key value is (49979609963).

1. Verify whether the database is in part of AG. If AG is configured for the same database, First step is to remove the database from AG
2. Disable Change Tracking for the database which is having issue, where you are unable to take Full Backup
3. perform multiple checkpoints until checkpoints completed successfully instead of throwing errors
4. After performing checkpoint sucessfully, we are able to take the full backup for the database without any issue
5. verify the size of the sys.syscommittab table through "sp_spaceused 'sys.syscommittab'"
6. Post that, since the data in the sys.syscommittab will be there we need to flush it with the below command:
sp_flush_commit_table_on_demand
7. Post flusing the table, reverify the size of the sys.syscommittab. Ideally the data in the table would have flushed and there will be less data in the table
8. After fixing the issue, Enable Change Tracking and Always On which are disabled earlier

