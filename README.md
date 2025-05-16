# PG-Important-Queries

--------------- Check Dead Tuples  -----------------------------------
```
select schemaname, relname AS table_name, n_dead_tup AS dead_tuples
from pg_stat_user_tables ORDER BY n_dead_tup DESC;
```


-------------Script Remove Files from foder windows os----------------------------------
```
$folder = "C:\Users\FMGS10009RJ26\Documents\ds_log_ds-000006"
$keepDate = "2025_05_14"

Get-ChildItem -Path $folder -Filter "CoreLog_*" | ForEach-Object {
    if ($_ -match "CoreLog_.*?_(\d{4}_\d{2}_\d{2})") {  # Adjusted regex pattern to match YYYY_MM_DD
        $fileDate = $matches[1]  # Extract matched date portion

        if ($fileDate -ne $keepDate) {  # Ensure files with 2025_04_30 are NOT deleted
            Write-Host "Deleting file: $($_.FullName)"
            Remove-Item $_.FullName
        }
    }
}
```
-------------------------------  Blocking ----------------------------------------
```
SELECT blocked.pid AS blocked_pid, blocker.pid AS blocking_pid
FROM pg_locks blocked
JOIN pg_locks blocker ON blocked.locktype = blocker.locktype
WHERE blocked.granted = FALSE;


SELECT pid, usename, application_name, state, wait_event, query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL;

```

----------------- -Blocking Query -------------------------
```
SELECT
  blocked_locks.pid AS blocked_pid,
  blocked_activity.usename AS blocked_user,
  blocking_locks.pid AS blocking_pid,
  blocking_activity.usename AS blocking_user,
  blocked_activity.query AS blocked_query,
  blocking_activity.query AS blocking_query,
  blocking_activity.state,
  now() - blocking_activity.query_start AS blocking_duration
FROM pg_locks blocked_locks
JOIN pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
  AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
  AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
  AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
  AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
  AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
  AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
  AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
  AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
  AND blocking_locks.pid != blocked_locks.pid
JOIN pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```
---------------------------------Check Current use lock----------------------------
```

SELECT * FROM pg_locks WHERE NOT granted;
```
------------------------------------------------------------------------------------------

