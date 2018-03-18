--- 
layout: post
title: Execution plan of never ending query
comments: true
date: 2012-08-21
---
 
Have you ever had situation that a query runs fine on one server and slow on different one – although specification is the same or better? Well I saw it more than once (yes you guess I'm no sql guru ;). Usually when it happen first stop to figure out what is going on is - actual execution plan. In Management Studio we can include it clicking 'Include Actual Execution Plan' (Ctrl+M) - but it will be displayed only after query finish to execute. And this may take a while ;)

Other way to get it is to use [sys.dm_exec_requests][1] view to obtain plan handle and then use [sys.dm_exec_query_paln][2] to get actual execution plan:

``` sql 
SELECT * FROM sys.dm_exec_query_plan(0x050001003B644F5A40015286000000000000000000000000)
```

We can get xml content of *query_plan* column and paste it to new file with extension \*.sqlplan. 
Now we can double click it and we will get graphical representation of the plan in SQL Management Studio.
 
The 0x0500... is actual plan handle obtained from the first view. View [sys.dm_exec_requests][1] holds all actually executed requests - it might be a bit of a problem to determine which entry represent our running query. Column *command* might be helpful to narrow the search as it determines type of command (SELECT, INSERT etc.). What I like to do when I work in SQL Management Studio is to first obtain session id of window in which I'll run long running query using:

``` sql
SELECT @@SPID
```

Then in the new window (in the previous one, we execute our long running query) we just query:

``` sql
SELECT * FROM sys.dm_exec_requests WHERE session_id = <our_session_id>
```
 
If the query is executed by our application we can use [sys.dm_exec_sql_text][3] view to determine sql statement and from that identify which request is our query:

``` sql
SELECT * FROM sys.dm_exec_sql_text(0x0200000002C6D42F7AAACE541C95D637FEB5A6185258B435)
```
 
You can also put above views together and use this:

``` sql
SELECT
    [spid] = r.session_id,
    [database] = DB_NAME(r.database_id),
    r.start_time,
    r.[status],
    r.command,
    [object] = QUOTENAME(OBJECT_SCHEMA_NAME(t.objectid, t.[dbid]))
    + '.' + QUOTENAME(OBJECT_NAME(t.objectid, t.[dbid])),
    t.[text] AS query_text,
    p.query_plan
FROM
    sys.dm_exec_requests AS r
CROSS APPLY
    sys.dm_exec_sql_text(r.[sql_handle]) AS t
CROSS APPLY
    sys.dm_exec_query_plan(r.[plan_handle]) AS p
WHERE
    r.session_id <> @@SPID
    AND 
    r.session_id > 50
```
 
 
 
 
 
 
[1]:http://msdn.microsoft.com/en-us/library/ms177648%28v=sql.90%29.aspx
[2]: http://msdn.microsoft.com/en-us/library/ms189747.aspx
[3]: http://msdn.microsoft.com/en-us/library/ms181929.aspx
