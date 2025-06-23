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
