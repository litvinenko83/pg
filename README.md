# pg
## полезные запросы Postgres


### Update JSON
```sql
update the_table 
   set attr['is_default'] = to_jsonb(false); 
```
*https://dba.stackexchange.com/questions/295298/how-to-update-a-property-value-of-a-jsonb-field*


### Поиск по тексту хранимых процедур и функций
```sql
SELECT 
    n.nspname AS schema_name, 
    pp.proname AS function_name, 
    pp.prosrc AS function_source 
FROM 
    pg_proc pp 
JOIN 
    pg_namespace n ON n.oid = pp.pronamespace 
WHERE 
    pp.prosrc ILIKE '%shkfor%'; 
```


### Table+Index size
```sql
SELECT 
*, 
pg_size_pretty(table_bytes) AS table, 
pg_size_pretty(index_bytes) AS index, 
pg_size_pretty(total_bytes) AS total 
FROM ( 
SELECT 
*, total_bytes - index_bytes - COALESCE(toast_bytes, 0) AS table_bytes 
FROM ( 
SELECT 
c.oid, 
nspname AS table_schema, 
relname AS table_name, 
c.reltuples AS row_estimate, 
pg_total_relation_size(c.oid) AS total_bytes, 
pg_indexes_size(c.oid) AS index_bytes, 
pg_total_relation_size(reltoastrelid) AS toast_bytes 
FROM 
pg_class c 
LEFT JOIN pg_namespace n ON n.oid = c.relnamespace 
WHERE relkind = 'r' 
) a 
) a 
WHERE table_schema != 'public' 
ORDER BY total_bytes DESC; 
```
*http://www.gilev.ru/sizetablepostgre/*


### AUTOVACUUM  

*https://habr.com/ru/companies/postgrespro/articles/452762/*

```sql
--условие запуска автовакуум 
pg_stat_all_tables.n_dead_tup >= autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * pg_class.reltupes 

--условие запуска аналайза 
pg_stat_all_tables.n_mod_since_analyze >= autovacuum_analyze_threshold + autovacuum_analyze_scale_factor * pg_class.reltupes 
```


### Неиспользуемые индексы (с размерами) 

```sql
WITH index_stats AS ( 
  SELECT 
    psui.schemaname, 
    psui.relname AS tablename, 
    psui.indexrelname, 
    pg_size_pretty(pg_relation_size(psui.indexrelid)) AS index_size, 
    psui.indexrelid, 
    psui.idx_scan, 
    psui.idx_tup_fetch, 
    psui.idx_tup_read 
  FROM 
    pg_stat_user_indexes psui 
  JOIN 
    pg_index pi 
  ON 
    psui.indexrelid = pi.indexrelid 
) 
SELECT 
  schemaname, 
  tablename, 
  indexrelname, 
  index_size, 
  idx_scan, 
  idx_tup_fetch, 
  idx_tup_read 
FROM 
  index_stats 
WHERE 
  idx_scan = 0 
ORDER BY 
  pg_relation_size(indexrelid) DESC; 
```


### блоат таблиц 

```sql
select schemaname, 
relname, 
pg_size_pretty(pg_relation_size(schemaname|| '.' || relname)) as size, 
n_live_tup, 
n_dead_tup, 
CASE WHEN n_live_tup > 0 THEN round((n_dead_tup::float / 
n_live_tup::float)::numeric, 4) END AS dead_tup_ratio, 
last_autovacuum, 
last_autoanalyze 
from pg_stat_user_tables 
WHERE schemaname not in ('public') 
order by dead_tup_ratio desc NULLS LAST; 
```


### отвал сети вручную --закрывает все сессии 

```sql
--если в бд особо нет хождений по апи в принципе пользователи даже не заметят, а синки просто переотправят, считай как отвал сети) 
SELECT PG_TERMINATE_BACKEND(pid)   
  FROM pg_stat_activity   
WHERE pid <> PG_BACKEND_PID(); 
```


### дерево блокировок 

```sql
select 
  bda.pid as blocked_pid, 
  bda.query as blocked_query, 
  bda.query_start blocked_query_start, 
  bga.pid as blocking_pid, 
  bga.query as blocking_query, 
  bda.query_start blocking_query_start 
from pg_catalog.pg_locks bdl 
  join pg_stat_activity bda 
    on bda.pid = bdl.pid 
  join pg_catalog.pg_locks bgl 
    on bgl.pid != bdl.pid 
    and bgl.transactionid = bdl.transactionid 
  join pg_stat_activity bga 
    on bga.pid = bgl.pid 
where not bdl.granted 
```


### дерево ожидания блокировок 

```sql
with recursive activity as ( 
  select 
    pg_blocking_pids(pid) blocked_by, 
    *, 
    age(clock_timestamp(), xact_start)::interval(0) as tx_age, 
    -- "pg_locks.waitstart" – PG14+ only; for older versions:  age(clock_timestamp(), state_change) as wait_age 
    age(clock_timestamp(), (select max(l.waitstart) from pg_locks l where a.pid = l.pid))::interval(0) as wait_age 
  from pg_stat_activity a 
  where state is distinct from 'idle' 
), blockers as ( 
  select 
    array_agg(distinct c order by c) as pids 
  from ( 
    select unnest(blocked_by) 
    from activity 
  ) as dt(c) 
), tree as ( 
  select 
    activity.*, 
    1 as level, 
    activity.pid as top_blocker_pid, 
    array[activity.pid] as path, 
    array[activity.pid]::int[] as all_blockers_above 
  from activity, blockers 
  where 
    array[pid] <@ blockers.pids 
    and blocked_by = '{}'::int[] 
  union all 
  select 
    activity.*, 
    tree.level + 1 as level, 
    tree.top_blocker_pid, 
    path || array[activity.pid] as path, 
    tree.all_blockers_above || array_agg(activity.pid) over () as all_blockers_above 
  from activity, tree 
  where 
    not array[activity.pid] <@ tree.all_blockers_above 
    and activity.blocked_by <> '{}'::int[] 
    and activity.blocked_by <@ tree.all_blockers_above 
) 
select 
  pid, 
  blocked_by, 
  case when wait_event_type <> 'Lock' then replace(state, 'idle in transaction', 'idletx') else 'waiting' end as state, 
  wait_event_type || ':' || wait_event as wait, 
  wait_age, 
  tx_age, 
  to_char(age(backend_xid), 'FM999,999,999,990') as xid_age, 
  to_char(2147483647 - age(backend_xmin), 'FM999,999,999,990') as xmin_ttf, 
  datname, 
  usename, 
  (select count(distinct t1.pid) from tree t1 where array[tree.pid] <@ t1.path and t1.pid <> tree.pid) as blkd, 
  format( 
    '%s %s%s', 
    lpad('[' || pid::text || ']', 9, ' '), 
    repeat('.', level - 1) || case when level > 1 then ' ' end, 
    left(query, 1000) 
  ) as query 
from tree 
order by top_blocker_pid, level, pid 
```


### Поиск индексов, не используемых ни на одной партиции таблицы

```sql
WITH index_stats AS ( 
  SELECT 
    psui.schemaname, 
    psui.relname AS tablename, 
    psui.indexrelname, 
    pg_relation_size(psui.indexrelid) AS index_size, 
    psui.indexrelid, 
    psui.idx_scan 
  FROM 
    pg_stat_user_indexes psui 
  JOIN 
    pg_index pi 
  ON 
    psui.indexrelid = pi.indexrelid 
  WHERE 
    psui.schemaname = 'warehouse' 
    --AND psui.relname ILIKE 'shktracking%' 
), 
base_names AS ( 
  SELECT 
    schemaname, 
    regexp_replace(tablename, '(00|01|02|03|04|05|06|07|08|09|10|11|12|13|14|15|16|17|18|19|20|21|22|23|24|25|26|27|28|29|30|31|32|33|34|35|36|37|38|39)', '', 'g') AS base_tablename, 
    regexp_replace(indexrelname, '(00|01|02|03|04|05|06|07|08|09|10|11|12|13|14|15|16|17|18|19|20|21|22|23|24|25|26|27|28|29|30|31|32|33|34|35|36|37|38|39)', '', 'g') AS base_index_name, 
    index_size, 
    idx_scan 
  FROM 
    index_stats 
) 
SELECT 
  schemaname, 
  base_tablename, 
  base_index_name, 
  pg_size_pretty(SUM(index_size)) AS total_index_size 
FROM 
  base_names 
GROUP BY 
  schemaname, base_tablename, base_index_name 
HAVING 
  SUM(idx_scan) = 0 
ORDER BY 
  total_index_size DESC; 
```


### блоат индексов 

*https://github.com/pgexperts/pgx_scripts/blob/master/bloat/index_bloat_check.sql*

```sql
WITH btree_index_atts AS ( 
    SELECT nspname, 
        indexclass.relname as index_name, 
        indexclass.reltuples, 
        indexclass.relpages, 
        indrelid, indexrelid, 
        indexclass.relam, 
        tableclass.relname as tablename, 
        regexp_split_to_table(indkey::text, ' ')::smallint AS attnum, 
        indexrelid as index_oid 
    FROM pg_index 
    JOIN pg_class AS indexclass ON pg_index.indexrelid = indexclass.oid 
    JOIN pg_class AS tableclass ON pg_index.indrelid = tableclass.oid 
    JOIN pg_namespace ON pg_namespace.oid = indexclass.relnamespace 
    JOIN pg_am ON indexclass.relam = pg_am.oid 
    WHERE pg_am.amname = 'btree' and indexclass.relpages > 0 
         AND nspname NOT IN ('pg_catalog','information_schema') 
    ), 
index_item_sizes AS ( 
    SELECT 
    ind_atts.nspname, ind_atts.index_name, 
    ind_atts.reltuples, ind_atts.relpages, ind_atts.relam, 
    indrelid AS table_oid, index_oid, 
    current_setting('block_size')::numeric AS bs, 
    8 AS maxalign, 
    24 AS pagehdr, 
    CASE WHEN max(coalesce(pg_stats.null_frac,0)) = 0 
        THEN 2 
        ELSE 6 
    END AS index_tuple_hdr, 
    sum( (1-coalesce(pg_stats.null_frac, 0)) * coalesce(pg_stats.avg_width, 1024) ) AS nulldatawidth 
    FROM pg_attribute 
    JOIN btree_index_atts AS ind_atts ON pg_attribute.attrelid = ind_atts.indexrelid AND pg_attribute.attnum = ind_atts.attnum 
    JOIN pg_stats ON pg_stats.schemaname = ind_atts.nspname 
          -- stats for regular index columns 
          AND ( (pg_stats.tablename = ind_atts.tablename AND pg_stats.attname = pg_catalog.pg_get_indexdef(pg_attribute.attrelid, pg_attribute.attnum, TRUE)) 
          -- stats for functional indexes 
          OR   (pg_stats.tablename = ind_atts.index_name AND pg_stats.attname = pg_attribute.attname)) 
    WHERE pg_attribute.attnum > 0 
    GROUP BY 1, 2, 3, 4, 5, 6, 7, 8, 9 
), 
index_aligned_est AS ( 
    SELECT maxalign, bs, nspname, index_name, reltuples, 
        relpages, relam, table_oid, index_oid, 
        coalesce ( 
            ceil ( 
                reltuples * ( 6 
                    + maxalign 
                    - CASE 
                        WHEN index_tuple_hdr%maxalign = 0 THEN maxalign 
                        ELSE index_tuple_hdr%maxalign 
                      END 
                    + nulldatawidth 
                    + maxalign 
                    - CASE /* Add padding to the data to align on MAXALIGN */ 
                        WHEN nulldatawidth::integer%maxalign = 0 THEN maxalign 
                        ELSE nulldatawidth::integer%maxalign 
                      END 
                )::numeric 
              / ( bs - pagehdr::NUMERIC ) 
              +1 ) 
         , 0 ) 
      as expected 
    FROM index_item_sizes 
), 
raw_bloat AS ( 
    SELECT current_database() as dbname, nspname, pg_class.relname AS table_name, index_name, 
        bs*(index_aligned_est.relpages)::bigint AS totalbytes, expected, 
        CASE 
            WHEN index_aligned_est.relpages <= expected 
                THEN 0 
                ELSE bs*(index_aligned_est.relpages-expected)::bigint 
            END AS wastedbytes, 
        CASE 
            WHEN index_aligned_est.relpages <= expected 
                THEN 0 
                ELSE bs*(index_aligned_est.relpages-expected)::bigint * 100 / (bs*(index_aligned_est.relpages)::bigint) 
            END AS realbloat, 
        pg_relation_size(index_aligned_est.table_oid) as table_bytes, 
        stat.idx_scan as index_scans 
    FROM index_aligned_est 
    JOIN pg_class ON pg_class.oid=index_aligned_est.table_oid 
    JOIN pg_stat_user_indexes AS stat ON index_aligned_est.index_oid = stat.indexrelid 
), 
format_bloat AS ( 
SELECT dbname as database_name, nspname as schema_name, table_name, index_name, 
        round(realbloat) as bloat_pct, round(wastedbytes/(1024^2)::NUMERIC) as bloat_mb, 
        round(totalbytes/(1024^2)::NUMERIC,3) as index_mb, 
        round(table_bytes/(1024^2)::NUMERIC,3) as table_mb, 
        index_scans 
FROM raw_bloat 
) 
-- final query outputting the bloated indexes 
-- change the where and order by to change 
-- what shows up as bloated 
SELECT concat('REINDEX INDEX CONCURRENTLY ',schema_name,'.',index_name,';'), * 
FROM format_bloat 
WHERE ( bloat_pct > 50 and bloat_mb > 10 ) 
ORDER BY bloat_pct DESC; 
```


### Размер WAL

```sql
SELECT pg_size_pretty(SUM(size)) AS total_wal_size 
FROM pg_ls_waldir();
```


### Отставание репликации

```sql
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS replication_lag
FROM
    pg_stat_replication;
```

### Размер полей таблицы

```sql
SELECT
    pg_size_pretty(SUM(pg_column_size(field1))) AS field1_total_size,
    pg_size_pretty(SUM(pg_column_size(field2))) AS field2_total_size
FROM
    shemaname.tablename;
```
