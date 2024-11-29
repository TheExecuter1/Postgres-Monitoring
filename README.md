## This contains some issues and learnings I faced while keeping Postgres DB up and running. Feel Free to add yours and rectify mine :) 


### Connection Pooling
1.  See number of active connections:

`sql>
select count(*) as total_connections  from pg_stat_activity;
`


2. The values of connection pooling should be
: min = no of cores
: max = 2 * no of cores + (no of disk) , After this the transaction per seconds [throttles](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
    
    Eg, for A CPU with 32 cores min ~ 30 and max ~ 75


### High Memory
1. If there are sudden spikes and drops in memory consumptions [VACUUM](https://www.postgresql.org/docs/current/sql-vacuum.html) is most likely the case. 
    Verify this by checking the time of spike and release with Vacuum timings using
    `sql>
    SELECT relname AS table_name, 
       last_vacuum AT TIME ZONE 'UTC' AT TIME ZONE 'Asia/Kolkata' AS last_vacuum_in_kolkata, 
       last_autovacuum AT TIME ZONE 'UTC' AT TIME ZONE 'Asia/Kolkata' AS last_autovacuum_in_kolkata
        FROM pg_stat_user_tables
        ORDER BY last_vacuum_in_kolkata DESC NULLS LAST;
    `
2.    Manually run VACUUM using `sql> VACUUM ` to release memory, takes time. 
3.    <mark>NEVER RUN</mark> `sql> VACCUM FULL` while DB is in active use , it blocks transactions.


### DISK
1. How to see the disk usage:
    - ssh into the container using `shell> docker exec -it <conatiner_name> bash`
    - `shell> df -h` to see over all disk usage for the container
    - Move around and use `du -sh <dir>` to see usage
   
2. DISK Space bloating **WAL**
   Sometimes improperly configured logical replication can fill up DB with wal log files.
   It will keep adding new logs while keeping old unused one. Diagnose this 
   - Run `shell> du -sh /bitnami/postgresql/data/pg_wal`
   - Run `sql> SELECT pg_size_pretty(pg_database_size(current_database())) AS database_size;`
   - IF  wal/db is very high, replication is most likely the issue. 
   Resolution :
   - Check for the inactive slots `sql> SELECT * FROM pg_replication_slots;`
   - Remove the inactive slots `sql> SELECT pg_drop_replication_slot('<slot_name>');`
   - Run `sql> checkpoint` [Checkpoint](https://www.cybertec-postgresql.com/en/postgresql-what-is-a-checkpoint/)
   Other Causes:
   - Check if postgres is archiving the logs using `sql> archive_mode`, if its 'ON' could be an issue.
   - If there is master-slave replication enabled check if the replica is lagging behind.
     This may cause unwarranted writes. Run this in master to verify 
     `sql> SELECT application_name, pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag FROM pg_stat_replication;`
   - Other important variables to help diagnose this issue
          `SHOW wal_keep_size;
          SHOW max_wal_size;
          SHOW min_wal_size;`
3. Monitor WAL using the 'bgwriter' view
    `sql> SELECT * FROM pg_stat_bgwriter;`







