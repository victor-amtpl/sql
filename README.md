# My SQL Query Documentation

## CHECK TABLE USED IN WHICH SP

```sql
SELECT DISTINCT p.name AS ProcedureName,o.name AS TableName
FROM sys.procedures p
INNER JOIN sys.sql_modules m ON p.object_id = m.object_id
INNER JOIN sys.objects o ON o.type = 'U'  -- 'U' indicates user tables
WHERE m.definition LIKE '%TABLE_NAME%'
AND o.name = 'TABLE_NAME'
ORDER BY p.name;
```

## CHECK TABLE COLUMN USED IN WHICH SP

```sql
SELECT DISTINCT ROUTINE_NAME,ROUTINE_TYPE
FROM INFORMATION_SCHEMA.ROUTINES AS R
INNER JOIN sys.sql_modules AS M ON R.SPECIFIC_NAME = OBJECT_NAME(M.object_id)
WHERE M.definition LIKE '%COLUMN_NAME%' 
AND M.definition LIKE '%TABLE_NAME%'
ORDER BY p.name;
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
