# PG-Important-Queries

--------------- Check Dead Tuples  -----------------------------------
```
SELECT relname, n_dead_tup, n_live_tup 
FROM pg_stat_all_tables 
WHERE n_dead_tup > (n_live_tup * 0.2)
ORDER BY n_dead_tup DESC;



-------- Long Running Query ------

SELECT datname, pid, state, query, age(clock_timestamp(), query_start) AS age
FROM pg_stat_activity
WHERE state <> 'idle'
AND query NOT LIKE '% FROM pg_stat_activity %'
ORDER BY age;

---Monitor Vacuum & REINDEX Progress---

SELECT pid, query, state, wait_event
FROM pg_stat_activity
WHERE query LIKE '%VACUUM%' OR query LIKE '%REINDEX%';

------------ Monitor Last autovacuum |vacuum and analyze |autoanalyzer status ----------------

SELECT schemaname, relname, last_autovacuum, last_autoanalyze,last_analyze,last_vacuum 
FROM pg_stat_all_tables;

-----------Check Current use lock------------------------

SELECT * FROM pg_locks WHERE NOT granted;

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
--- Pg_dump by Python ---
```

import psycopg2
import subprocess
import os

host='host_name'
port='port'
passwd='password'
user='username'
backup_dir="/path/to/backup"
backup_format="plain" # custome,plain,tar,directory 

db_list =[db1, db2,db3, db4,b5]


for db in db_list:
	timestamp = datetime.datetime.now().strtime("%Y%m%d_%H%M%S")
	backup_file= f"{db}_backup_{timestamp}.backup"
	backup_path = os.path.join(backup_dir,backu_file)

	# pg_dump command
	
	pg_dump_cmd = ["pg_dump","-U",user,"-h",host,"-p",port,"-F",backup_format,"-f",backup_path,db ]
	
	#set environment variable for password (optional)

	os.environ['PGPASSWORD']= "passwd"
	

	try:

	   print(f"staring backup of '{db}' to '{backup_path}'")
	   subprocess.run()pg_dump_cmd,check=True)
	   print("backup completed successfully.")
       except exception as error:
	    print("Backup Failed {db}!")

```
----- Vacuum Verbose By Python Scrip -----
```
import psycopg2
host = "host_name"
port = "port"
user = "user_name"
pass = "password"
DB_list = ["db1","db2","db3","db4","db5"]
for db in DB_list:
     try:
	conn = psycopg2.connect(host=host,port=port,user=user, pass=pass,db=db)
	conn.autocommit()
	cursor = conn.cursor()
	cursor.execute("VACUUM VERBOSE;")
	print(f"vauum is completed successfully on {Db}")
	cursor.close()
	conn.close()
	Print("Database Connection Closed {db} ")
    except exception as error:
	print(f"Dtaabase Connection Failed: {error}")
print("Vacuum Completed Successfully on all :- {DB_list}")

```
---------------------------------REINDEX on index which has bloating wastebytes >0 by python script ----------------------------
```
import psycopg2

# The SQL query to analyze index bloat
QUERY = """
SELECT current_database(),
       schemaname,
       tablename,
       ROUND((CASE WHEN otta=0 THEN 0.0 ELSE sml.relpages::float/otta END)::numeric, 1) AS tbloat,
       CASE WHEN relpages < otta THEN 0 ELSE bs*(sml.relpages-otta)::BIGINT END AS wastedbytes,
       iname,
       ROUND((CASE WHEN iotta=0 OR ipages=0 THEN 0.0 ELSE ipages::float/iotta END)::numeric, 1) AS ibloat,
       CASE WHEN ipages < iotta THEN 0 ELSE bs*(ipages-iotta) END AS wastedibytes
FROM (
    SELECT schemaname, tablename, cc.reltuples, cc.relpages, bs,
           CEIL((cc.reltuples * ((datahdr + ma - (CASE WHEN datahdr % ma = 0 THEN ma ELSE datahdr % ma END)) + nullhdr2 + 4)) / (bs - 20::float)) AS otta,
           COALESCE(c2.relname, '?') AS iname,
           COALESCE(c2.reltuples, 0) AS ituples,
           COALESCE(c2.relpages, 0) AS ipages,
           COALESCE(CEIL((c2.reltuples * (datahdr - 12)) / (bs - 20::float)), 0) AS iotta
    FROM (
        SELECT ma, bs, schemaname, tablename,
               (datawidth + (hdr + ma - (CASE WHEN hdr % ma = 0 THEN ma ELSE hdr % ma END)))::numeric AS datahdr,
               (maxfracsum * (nullhdr + ma - (CASE WHEN nullhdr % ma = 0 THEN ma ELSE nullhdr % ma END))) AS nullhdr2
        FROM (
            SELECT schemaname, tablename, hdr, ma, bs,
                   SUM((1 - null_frac) * avg_width) AS datawidth,
                   MAX(null_frac) AS maxfracsum,
                   hdr + (SELECT 1 + count(*) / 8
                          FROM pg_stats s2
                          WHERE null_frac <> 0
                          AND s2.schemaname = s.schemaname
                          AND s2.tablename = s.tablename) AS nullhdr
            FROM pg_stats s,
                 (SELECT (SELECT current_setting('block_size')::numeric) AS bs,
                         CASE WHEN substring(v, 12, 3) IN ('8.0', '8.1', '8.2') THEN 27 ELSE 23 END AS hdr,
                         CASE WHEN v ~ 'mingw32' THEN 8 ELSE 4 END AS ma
                  FROM (SELECT version() AS v) AS foo) AS constants
            GROUP BY 1, 2, 3, 4, 5
        ) AS foo
    ) AS rs
    JOIN pg_class cc ON cc.relname = rs.tablename
    JOIN pg_namespace nn ON cc.relnamespace = nn.oid
                          AND nn.nspname = rs.schemaname
                          AND nn.nspname <> 'information_schema'
    LEFT JOIN pg_index i ON indrelid = cc.oid
    LEFT JOIN pg_class c2 ON c2.oid = i.indexrelid
) AS sml
ORDER BY wastedbytes DESC;
"""

# Database connection details
DB_HOST = "bpay-pgflexi-pr-dam.postgres.database.azure.com"
DB_PORT = "5432"
DB_NAME = "dsaudit"
DB_USER = "psqladmin"
DB_PASSWORD = "P@ssw0rd@5432@2023"

# List to store bloated indexes
index_list = []

# Connect to PostgreSQL server
try:
    conn = psycopg2.connect(host=DB_HOST, port=DB_PORT, dbname=DB_NAME, user=DB_USER, password=DB_PASSWORD)
    conn.autocommit = True  # Ensure queries execute immediately
    cursor = conn.cursor()

    # Execute query to fetch index bloat details
    cursor.execute(QUERY)
    data = cursor.fetchall()

    # Extract indexes with wasted space
    for row in data:
        if row[7] > 0:
            index_list.append(row[5])

    # Set PostgreSQL parameters for optimized reindexing
    cursor.execute("SET max_parallel_workers_per_gather = 8;")
    cursor.execute("SET maintenance_work_mem = '20GB';")
    print("Database parameters set: max_parallel_workers_per_gather = 8, maintenance_work_mem = 20GB")

    # Loop through indexes and reindex them concurrently
    for index in index_list:
        try:
            query = f"REINDEX INDEX CONCURRENTLY {index};"
            print(f"Reindexing: {index}")
            cursor.execute(query)
            print(f"Successfully reindexed: {index}")
        except Exception as e:
            print(f"Error reindexing {index}: {e}")

    # Close the database connection
    cursor.close()
    conn.close()
    print("Database connection closed.")

except Exception as db_error:
    print(f"Database connection failed: {db_error}")
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
------Monitor TPS And CPU by Python Script -----
```
import psycopg2
import time

# PostgreSQL connection details
conn = psycopg2.connect(
    dbname="your_database",
    user="your_user",
    password="your_password",
    host="your_host",
    port="your_port"
)
cursor = conn.cursor()

prev_txns = None  # Store previous transaction count

while True:
    # Query PostgreSQL for transaction count
    cursor.execute("""
        SELECT xact_commit + xact_rollback AS total_txns
        FROM pg_stat_database
        WHERE datname = 'your_database';
    """)
    current_txns = cursor.fetchone()[0]

    if prev_txns is not None:
        tps = current_txns - prev_txns  # Calculate TPS
        print(f"Transactions Per Second: {tps}")

    prev_txns = current_txns  # Update previous count
    time.sleep(1)  # Run every second
```


