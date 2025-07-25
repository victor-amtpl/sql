## SQL CURRENT JOBS SPS
```sql
SELECT 
    j.name AS JobName,
    s.step_id AS StepID,
    s.step_name AS StepName,
    s.subsystem AS Subsystem,
    s.command AS CommandText
FROM 
    msdb.dbo.sysjobs j
INNER JOIN 
    msdb.dbo.sysjobsteps s ON j.job_id = s.job_id
WHERE 
    s.command LIKE '%EXEC%'
    OR s.command LIKE '%usp_%'
ORDER BY 
    j.name, s.step_id;
```

## TABLE TYPE DEFINITION
```sql
DECLARE @TypeName SYSNAME = 'YOUR_USERDEFINED_TABLE_TYPE'; -- Change this
DECLARE @SchemaName SYSNAME = (SELECT SCHEMA_NAME(schema_id) 
                               FROM sys.table_types WHERE name = @TypeName);

DECLARE @SQL NVARCHAR(MAX) = '';
SET @SQL = 'CREATE TYPE [' + @SchemaName + '].[' + @TypeName + '] AS TABLE(' + CHAR(13);

SELECT @SQL = @SQL +
    '    [' + c.name + '] ' +
    t.name +
    CASE 
        WHEN t.name IN ('varchar', 'char', 'varbinary', 'binary', 'nvarchar', 'nchar') 
            THEN '(' + CASE WHEN c.max_length = -1 THEN 'MAX' 
                            ELSE CAST(
                                CASE 
                                    WHEN t.name IN ('nchar', 'nvarchar') 
                                        THEN c.max_length / 2 
                                    ELSE c.max_length 
                                END AS VARCHAR) END + ')'
        WHEN t.name IN ('decimal', 'numeric') 
            THEN '(' + CAST(c.precision AS VARCHAR) + ',' + CAST(c.scale AS VARCHAR) + ')'
        ELSE ''
    END + 
    CASE WHEN c.is_nullable = 1 THEN ' NULL' ELSE ' NOT NULL' END + ',' + CHAR(13)
FROM sys.table_types tt
JOIN sys.columns c ON tt.type_table_object_id = c.object_id
JOIN sys.types t ON c.user_type_id = t.user_type_id
WHERE tt.name = @TypeName
ORDER BY c.column_id;

-- Remove trailing comma and newline
SET @SQL = LEFT(@SQL, LEN(@SQL) - 2) + CHAR(13) + ')';

-- Output the CREATE TYPE statement
PRINT @SQL;
```

## TABLE TYPE DEPENDENCIES
```sql
SELECT OBJECT_NAME(referencing_id) AS dependent_object
FROM sys.sql_expression_dependencies
WHERE referenced_entity_name = 'YOUR_USER_DEFINED_TABLE_TYE';
```

## TABLE DEPENDENCIES IN SP
```sql
SELECT DISTINCT p.name AS ProcedureName,o.name AS TableName
FROM sys.procedures p
INNER JOIN sys.sql_modules m ON p.object_id = m.object_id
INNER JOIN sys.objects o ON o.type = 'U'  -- 'U' indicates user tables
WHERE m.definition LIKE '%TABLE_NAME%'
AND o.name = 'TABLE_NAME'
ORDER BY p.name;
```

## TABLE COLUMN DEPENDENCIES IN SP
```sql
SELECT DISTINCT ROUTINE_NAME,ROUTINE_TYPE
FROM INFORMATION_SCHEMA.ROUTINES AS R
INNER JOIN sys.sql_modules AS M ON R.SPECIFIC_NAME = OBJECT_NAME(M.object_id)
WHERE M.definition LIKE '%COLUMN_NAME%' 
AND M.definition LIKE '%TABLE_NAME%'
```
## LAZY QUERY

```sql
SELECT req.session_id,req.total_elapsed_time AS duration_ms,req.cpu_time AS cpu_time_ms,
req.total_elapsed_time - req.cpu_time AS wait_time,req.logical_reads,
SUBSTRING (REPLACE (REPLACE (SUBSTRING (ST.text, (req.statement_start_offset/2) + 1, ( (CASE statement_end_offset WHEN -1 THEN DATALENGTH(ST.text) ELSE req.statement_end_offset END - req.statement_start_offset)/2) + 1) , CHAR(10), ' '), CHAR(13), ' '), 1, 512) AS statement_text
FROM sys.dm_exec_requests AS req
CROSS APPLY sys.dm_exec_sql_text (req.sql_handle) AS ST
ORDER BY total_elapsed_time DESC
```

## HEAD BLOCKING
```sql
SET NOCOUNT ON
GO
SELECT SPID, BLOCKED, REPLACE (REPLACE (T.TEXT, CHAR(10), ' '), CHAR (13), ' ' ) AS BATCH
INTO #T
FROM sys.sysprocesses R CROSS APPLY sys.dm_exec_sql_text(R.SQL_HANDLE) T
GO
WITH BLOCKERS (SPID, BLOCKED, LEVEL, BATCH)
AS
(
SELECT SPID,
BLOCKED,
CAST (REPLICATE ('0', 4-LEN (CAST (SPID AS VARCHAR))) + CAST (SPID AS VARCHAR) AS VARCHAR (1000)) AS LEVEL,
BATCH FROM #T R
WHERE (BLOCKED = 0 OR BLOCKED = SPID)
AND EXISTS (SELECT * FROM #T R2 WHERE R2.BLOCKED = R.SPID AND R2.BLOCKED <> R2.SPID)
UNION ALL
SELECT R.SPID,
R.BLOCKED,
CAST (BLOCKERS.LEVEL + RIGHT (CAST ((1000 + R.SPID) AS VARCHAR (100)), 4) AS VARCHAR (1000)) AS LEVEL,
R.BATCH FROM #T AS R
INNER JOIN BLOCKERS ON R.BLOCKED = BLOCKERS.SPID WHERE R.BLOCKED > 0 AND R.BLOCKED <> R.SPID
)
SELECT N'    ' + REPLICATE (N'|         ', LEN (LEVEL)/4 - 1) +
CASE WHEN (LEN(LEVEL)/4 - 1) = 0
THEN 'HEAD -  '
ELSE '|------  ' END
+ CAST (SPID AS NVARCHAR (10)) + N' ' + BATCH AS BLOCKING_TREE
FROM BLOCKERS ORDER BY LEVEL ASC
GO
DROP TABLE #T
GO
```


## TABLE LOCK
```sql

SELECT request_session_id,resource_type, resource_associated_entity_id,
request_status, request_mode,request_session_id,
resource_description, o.object_id, o.name, o.type_desc 
FROM sys.dm_tran_locks l with(nolock), sys.objects o with(nolock)
WHERE l.resource_associated_entity_id = o.object_id and resource_database_id = DB_ID()

SELECT request_session_id, * FROM sys.dm_tran_locks
WHERE resource_database_id = DB_ID()
AND resource_associated_entity_id = OBJECT_ID(N'dbo.TABLE_NAME');

kill request_session_id

SELECT t.name AS TableName,r.request_mode AS LockType,r.request_status AS LockStatus,
r.request_session_id AS SessionID,s.login_name AS SessionLoginName,
er.command AS BlockingCommand,st.text AS BlockingQueryText
FROM sys.dm_tran_locks AS r
INNER JOIN sys.dm_exec_requests AS er ON r.request_request_id = er.request_id
INNER JOIN sys.dm_exec_sessions AS s ON er.session_id = s.session_id
INNER JOIN sys.objects AS t ON r.resource_associated_entity_id = t.object_id
CROSS APPLY sys.dm_exec_sql_text(er.sql_handle) AS st
WHERE r.resource_type = 'OBJECT' AND er.blocking_session_id IS NOT NULL;
```

## ROW NO
```sql
ROW_NUMBER() OVER (PARTITION BY MPC.MemberNo ORDER BY PMA.FromDate desc) Sno
```

## TRANSACTIONS
```sql
SELECT session_id,blocking_session_id,wait_type,wait_time,wait_resource,percent_complete,* 
FROM sys.dm_exec_requests WHERE status = 'running';
```
## MODIFIED SPs & TABLES
```sql
SELECT o.name AS ObjectName,SCHEMA_NAME(o.schema_id) AS SchemaName,o.type_desc,o.modify_date
FROM sys.objects o
WHERE o.is_ms_shipped = 0 AND CAST(o.modify_date AS DATE) = CAST(GETDATE() AS DATE)
ORDER BY o.modify_date DESC;
```

## SPID
```sql
select P.spid,right(convert(varchar, dateadd(ms, datediff(ms, P.last_batch, getdate()), '1900-01-01'), 121), 12) as 'batch_duration',
P.program_name,P.hostname,P.loginame,P.*
from master.dbo.sysprocesses P
where P.spid > 50 and P.status not in ('background', 'sleeping')
and P.cmd not in ('AWAITING COMMAND','MIRROR HANDLER','LAZY WRITER','CHECKPOINT SLEEP','RA MANAGER')
order by batch_duration desc
```
## SQL DETAILS
```sql
declare
    @spid int
,   @stmt_start int
,   @stmt_end int
,   @sql_handle binary(20)

set @spid = 000 -- Fill this in

select  top 1
    @sql_handle = sql_handle
,   @stmt_start = case stmt_start when 0 then 0 else stmt_start / 2 end
,   @stmt_end = case stmt_end when -1 then -1 else stmt_end / 2 end
from    sys.sysprocesses
where   spid = @spid
order by ecid

SELECT
    SUBSTRING(  text,
            COALESCE(NULLIF(@stmt_start, 0), 1),
            CASE @stmt_end
                WHEN -1
                    THEN DATALENGTH(text)
                ELSE
                    (@stmt_end - @stmt_start)
                END
        )
FROM ::fn_get_sql(@sql_handle)
```

## DELETE DUPLICATE RECORD
```sql
WITH CTE AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY COL1, COL2 ORDER BY (SELECT NULL)) AS rn
    FROM YOUR_TABLE_NAME
)
SELECT * FROM CTE
--DELETE FROM CTE
WHERE rn = 2;
```

