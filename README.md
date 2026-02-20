# pg
## полезные запросы Postgres


### Сколько мертвых строк осталось нагенерить update-ами/delete-ами до запуска следующего автовакуума

```sql
SELECT
    stat.schemaname as "схема",
    stat.relname AS "таблица",
    stat.n_dead_tup::BIGINT as n_dead_tup,
    stat.last_autovacuum as last_autovacuum,
--    stat.last_vacuum,
    stat.last_autoanalyze as last_autovacuum,
--    stat.last_analyze,
    cls.reltuples::BIGINT AS "оценка живых, строк",
    current_setting('autovacuum_vacuum_threshold')::bigint AS autovacuum_vacuum_threshold,
    current_setting('autovacuum_vacuum_scale_factor')::real AS autovacuum_vacuum_scale_factor,
    (current_setting('autovacuum_vacuum_threshold')::bigint +
     current_setting('autovacuum_vacuum_scale_factor')::real * cls.reltuples) AS "пороговое значение, строк",
    CASE
        WHEN (current_setting('autovacuum_vacuum_threshold')::bigint +
              current_setting('autovacuum_vacuum_scale_factor')::real * cls.reltuples)::numeric > 0
        THEN round((stat.n_dead_tup::numeric /
                   (current_setting('autovacuum_vacuum_threshold')::bigint +
                    current_setting('autovacuum_vacuum_scale_factor')::real * cls.reltuples)::numeric) * 100, 2)
        ELSE NULL
    END AS "n_dead_tup % от порога, %",
    GREATEST((current_setting('autovacuum_vacuum_threshold')::bigint +
              current_setting('autovacuum_vacuum_scale_factor')::real * cls.reltuples) - stat.n_dead_tup, 0) AS "до автовакуума мертвых строк осталось",
    CASE
        WHEN (current_setting('autovacuum_vacuum_threshold')::bigint +
              current_setting('autovacuum_vacuum_scale_factor')::real * cls.reltuples)::numeric > 0
        THEN round(GREATEST(100 - (stat.n_dead_tup::numeric /
                                  (current_setting('autovacuum_vacuum_threshold')::bigint +
                                   current_setting('autovacuum_vacuum_scale_factor')::real * cls.reltuples)::numeric) * 100, 0), 2)
        ELSE NULL
    END AS "остаток, процентов",
    CASE
        WHEN stat.n_dead_tup >= (current_setting('autovacuum_vacuum_threshold')::bigint +
                                 current_setting('autovacuum_vacuum_scale_factor')::real * cls.reltuples)
        THEN 'Yes'
        ELSE 'No'
    END AS "нужен автовакуум"
FROM
    pg_stat_all_tables stat
JOIN
    pg_class cls ON stat.relid = cls.oid
WHERE
    stat.schemaname = 'public';
    --AND stat.relname = '';
```

### Проверка прогресса autovacuum

https://github.com/lesovsky/uber-scripts/blob/master/postgresql/sql/vacuum_activity.sql

```sql
SELECT
        p.pid,
        now() - a.xact_start AS duration,
        coalesce(wait_event_type ||'.'|| wait_event, 'f') AS waiting,
        CASE 
                WHEN a.query ~ '^autovacuum.*to prevent wraparound' THEN 'wraparound' 
                WHEN a.query ~ '^vacuum' THEN 'user'
                ELSE 'regular'
        END AS mode,
        p.datname AS database,
        p.relid::regclass AS table,
        p.phase,
        pg_size_pretty(p.heap_blks_total * current_setting('block_size')::int) AS table_size,
        pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
        pg_size_pretty(p.heap_blks_scanned * current_setting('block_size')::int) AS scanned,
        pg_size_pretty(p.heap_blks_vacuumed * current_setting('block_size')::int) AS vacuumed,
        round(100.0 * p.heap_blks_scanned / p.heap_blks_total, 1) AS scanned_pct,
        round(100.0 * p.heap_blks_vacuumed / p.heap_blks_total, 1) AS vacuumed_pct,
        p.index_vacuum_count,
        round(100.0 * p.num_dead_tuples / p.max_dead_tuples,1) AS dead_pct
FROM pg_stat_progress_vacuum p
RIGHT JOIN pg_stat_activity a ON a.pid = p.pid
WHERE (a.query ~* '^autovacuum:' OR a.query ~* '^vacuum') AND a.pid <> pg_backend_pid()
ORDER BY now() - a.xact_start DESC;
```

### Проверка прогресса построения/перестроения индексов

https://stackoverflow.com/questions/21546542/retrieve-a-progress-of-index-creation-process-in-postgresql

```sql
SELECT 
       clock_timestamp() - a.xact_start                          AS duration_so_far,
       coalesce(a.wait_event_type ||'.'|| a.wait_event, 'false') AS waiting,
       a.state,
       p.phase,
       CASE p.phase
           WHEN 'initializing' THEN '1 of 12'
           WHEN 'waiting for writers before build' THEN '2 of 12'
           WHEN 'building index: scanning table' THEN '3 of 12'
           WHEN 'building index: sorting live tuples' THEN '4 of 12'
           WHEN 'building index: loading tuples in tree' THEN '5 of 12'
           WHEN 'waiting for writers before validation' THEN '6 of 12'
           WHEN 'index validation: scanning index' THEN '7 of 12'
           WHEN 'index validation: sorting tuples' THEN '8 of 12'
           WHEN 'index validation: scanning table' THEN '9 of 12'
           WHEN 'waiting for old snapshots' THEN '10 of 12'
           WHEN 'waiting for readers before marking dead' THEN '11 of 12'
           WHEN 'waiting for readers before dropping' THEN '12 of 12'
       END AS phase_progress,
       format(
           '%s (%s of %s)',
           coalesce(round(100.0 * p.blocks_done / nullif(p.blocks_total, 0), 2)::text || '%', 'not applicable'),
           p.blocks_done::text,
           p.blocks_total::text
       ) AS scan_progress,
       format(
           '%s (%s of %s)',
           coalesce(round(100.0 * p.tuples_done / nullif(p.tuples_total, 0), 2)::text || '%', 'not applicable'),
           p.tuples_done::text,
           p.tuples_total::text
       ) AS tuples_loading_progress,
       format(
           '%s (%s of %s)',
           coalesce((100 * p.lockers_done / nullif(p.lockers_total, 0))::text || '%', 'not applicable'),
           p.lockers_done::text,
           p.lockers_total::text
       ) AS lockers_progress,
       format(
           '%s (%s of %s)',
           coalesce((100 * p.partitions_done / nullif(p.partitions_total, 0))::text || '%', 'not applicable'),
           p.partitions_done::text,
           p.partitions_total::text
       ) AS partitions_progress,
       p.current_locker_pid,
       trim(trailing ';' from l.query) AS current_locker_query
  FROM pg_stat_progress_create_index   AS p
  JOIN pg_stat_activity                AS a ON a.pid = p.pid
  LEFT JOIN pg_stat_activity           AS l ON l.pid = p.current_locker_pid
 ORDER BY clock_timestamp() - a.xact_start DESC;

```


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

### Блоат таблиц

```sql
-- new table bloat query
-- still needs work; is often off by +/- 20%
WITH constants AS (
    -- define some constants for sizes of things
    -- for reference down the query and easy maintenance
    SELECT current_setting('block_size')::numeric AS bs, 23 AS hdr, 8 AS ma
),
no_stats AS (
    -- screen out table who have attributes
    -- which dont have stats, such as JSON
    SELECT table_schema, table_name, 
        n_live_tup::numeric as est_rows,
        pg_table_size(relid)::numeric as table_size
    FROM information_schema.columns
        JOIN pg_stat_user_tables as psut
           ON table_schema = psut.schemaname
           AND table_name = psut.relname
        LEFT OUTER JOIN pg_stats
        ON table_schema = pg_stats.schemaname
            AND table_name = pg_stats.tablename
            AND column_name = attname 
    WHERE attname IS NULL
        AND table_schema NOT IN ('pg_catalog', 'information_schema')
    GROUP BY table_schema, table_name, relid, n_live_tup
),
null_headers AS (
    -- calculate null header sizes
    -- omitting tables which dont have complete stats
    -- and attributes which aren't visible
    SELECT
        hdr+1+(sum(case when null_frac <> 0 THEN 1 else 0 END)/8) as nullhdr,
        SUM((1-null_frac)*avg_width) as datawidth,
        MAX(null_frac) as maxfracsum,
        schemaname,
        tablename,
        hdr, ma, bs
    FROM pg_stats CROSS JOIN constants
        LEFT OUTER JOIN no_stats
            ON schemaname = no_stats.table_schema
            AND tablename = no_stats.table_name
    WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
        AND no_stats.table_name IS NULL
        AND EXISTS ( SELECT 1
            FROM information_schema.columns
                WHERE schemaname = columns.table_schema
                    AND tablename = columns.table_name )
    GROUP BY schemaname, tablename, hdr, ma, bs
),
data_headers AS (
    -- estimate header and row size
    SELECT
        ma, bs, hdr, schemaname, tablename,
        (datawidth+(hdr+ma-(case when hdr%ma=0 THEN ma ELSE hdr%ma END)))::numeric AS datahdr,
        (maxfracsum*(nullhdr+ma-(case when nullhdr%ma=0 THEN ma ELSE nullhdr%ma END))) AS nullhdr2
    FROM null_headers
),
table_estimates AS (
    -- make estimates of how large the table should be
    -- based on row and page size
    SELECT schemaname, tablename, bs,
        reltuples::numeric as est_rows, relpages * bs as table_bytes,
    CEIL((reltuples*
            (datahdr + nullhdr2 + 4 + ma -
                (CASE WHEN datahdr%ma=0
                    THEN ma ELSE datahdr%ma END)
                )/(bs-20))) * bs AS expected_bytes,
        reltoastrelid
    FROM data_headers
        JOIN pg_class ON tablename = relname
        JOIN pg_namespace ON relnamespace = pg_namespace.oid
            AND schemaname = nspname
    WHERE pg_class.relkind = 'r'
),
estimates_with_toast AS (
    -- add in estimated TOAST table sizes
    -- estimate based on 4 toast tuples per page because we dont have 
    -- anything better.  also append the no_data tables
    SELECT schemaname, tablename, 
        TRUE as can_estimate,
        est_rows,
        table_bytes + ( coalesce(toast.relpages, 0) * bs ) as table_bytes,
        expected_bytes + ( ceil( coalesce(toast.reltuples, 0) / 4 ) * bs ) as expected_bytes
    FROM table_estimates LEFT OUTER JOIN pg_class as toast
        ON table_estimates.reltoastrelid = toast.oid
            AND toast.relkind = 't'
),
table_estimates_plus AS (
-- add some extra metadata to the table data
-- and calculations to be reused
-- including whether we cant estimate it
-- or whether we think it might be compressed
    SELECT current_database() as databasename,
            schemaname, tablename, can_estimate, 
            est_rows,
            CASE WHEN table_bytes > 0
                THEN table_bytes::NUMERIC
                ELSE NULL::NUMERIC END
                AS table_bytes,
            CASE WHEN expected_bytes > 0 
                THEN expected_bytes::NUMERIC
                ELSE NULL::NUMERIC END
                    AS expected_bytes,
            CASE WHEN expected_bytes > 0 AND table_bytes > 0
                AND expected_bytes <= table_bytes
                THEN (table_bytes - expected_bytes)::NUMERIC
                ELSE 0::NUMERIC END AS bloat_bytes
    FROM estimates_with_toast
    UNION ALL
    SELECT current_database() as databasename, 
        table_schema, table_name, FALSE, 
        est_rows, table_size,
        NULL::NUMERIC, NULL::NUMERIC
    FROM no_stats
),
bloat_data AS (
    -- do final math calculations and formatting
    select current_database() as databasename,
        schemaname, tablename, can_estimate, 
        table_bytes, round(table_bytes/(1024^2)::NUMERIC,3) as table_mb,
        expected_bytes, round(expected_bytes/(1024^2)::NUMERIC,3) as expected_mb,
        round(bloat_bytes*100/table_bytes) as pct_bloat,
        round(bloat_bytes/(1024::NUMERIC^2),2) as mb_bloat,
        table_bytes, expected_bytes, est_rows
    FROM table_estimates_plus
)
-- filter output for bloated tables
SELECT databasename, schemaname, tablename,
    can_estimate,
    est_rows,
    pct_bloat, mb_bloat,
    table_mb
FROM bloat_data
-- this where clause defines which tables actually appear
-- in the bloat chart
-- example below filters for tables which are either 50%
-- bloated and more than 20mb in size, or more than 25%
-- bloated and more than 4GB in size
WHERE ( pct_bloat >= 50 AND mb_bloat >= 10 )
    OR ( pct_bloat >= 25 AND mb_bloat >= 1000 )
ORDER BY mb_bloat DESC;
```


### Дубли индексов

```sql
SELECT ni.nspname || '.' || ct.relname AS "table", 
       ci.relname AS "dup index",
       pg_get_indexdef(i.indexrelid) AS "dup index definition", 
       i.indkey AS "dup index attributes",
       cii.relname AS "encompassing index", 
       pg_get_indexdef(ii.indexrelid) AS "encompassing index definition",
       ii.indkey AS "enc index attributes"
  FROM pg_index i
  JOIN pg_class ct ON i.indrelid=ct.oid
  JOIN pg_class ci ON i.indexrelid=ci.oid
  JOIN pg_namespace ni ON ci.relnamespace=ni.oid
  JOIN pg_index ii ON ii.indrelid=i.indrelid AND
                      ii.indexrelid != i.indexrelid AND
                      (array_to_string(ii.indkey, ' ') || ' ') like (array_to_string(i.indkey, ' ') || ' %') AND
                      (array_to_string(ii.indcollation, ' ')  || ' ') like (array_to_string(i.indcollation, ' ') || ' %') AND
                      (array_to_string(ii.indclass, ' ')  || ' ') like (array_to_string(i.indclass, ' ') || ' %') AND
                      (array_to_string(ii.indoption, ' ')  || ' ') like (array_to_string(i.indoption, ' ') || ' %') AND
                      NOT (ii.indkey::integer[] @> ARRAY[0]) AND -- Remove if you want expression indexes (you probably don't)
                      NOT (i.indkey::integer[] @> ARRAY[0]) AND -- Remove if you want expression indexes (you probably don't)
                      i.indpred IS NULL AND -- Remove if you want indexes with predicates
                      ii.indpred IS NULL AND -- Remove if you want indexes with predicates
                      CASE WHEN i.indisunique THEN ii.indisunique AND
                         array_to_string(ii.indkey, ' ') = array_to_string(i.indkey, ' ') ELSE true END
  JOIN pg_class ctii ON ii.indrelid=ctii.oid
  JOIN pg_class cii ON ii.indexrelid=cii.oid
 WHERE ct.relname NOT LIKE 'pg_%' AND
       NOT i.indisprimary
 ORDER BY 1, 2, 3;
```

### Need Indexes

```sql
WITH 
index_usage AS (
    SELECT  sut.relid,
            current_database() AS database,
            sut.schemaname::text as schema_name, 
            sut.relname::text AS table_name,
            sut.seq_scan as table_scans,
            sut.idx_scan as index_scans,
            pg_total_relation_size(relid) as table_bytes,
            round((sut.n_tup_ins + sut.n_tup_del + sut.n_tup_upd + sut.n_tup_hot_upd) / 
                (seq_tup_read::NUMERIC + 2), 2) as writes_per_scan
    FROM pg_stat_user_tables sut
),
index_counts AS (
    SELECT sut.relid,
        count(*) as index_count
    FROM pg_stat_user_tables sut LEFT OUTER JOIN pg_indexes
    ON sut.schemaname = pg_indexes.schemaname AND
        sut.relname = pg_indexes.tablename
    GROUP BY relid
),
too_many_tablescans AS (
    SELECT 'many table scans'::TEXT as reason, 
        database, schema_name, table_name,
        table_scans, pg_size_pretty(table_bytes) as table_size,
        writes_per_scan, index_count, table_bytes
    FROM index_usage JOIN index_counts USING ( relid )
    WHERE table_scans > 1000
        AND table_scans > ( index_scans * 2 )
        AND table_bytes > 32000000
        AND writes_per_scan < ( 1.0 )
    ORDER BY table_scans DESC
),
scans_no_index AS (
    SELECT 'scans, few indexes'::TEXT as reason,
        database, schema_name, table_name,
        table_scans, pg_size_pretty(table_bytes) as table_size,
        writes_per_scan, index_count, table_bytes
    FROM index_usage JOIN index_counts USING ( relid )
    WHERE table_scans > 100
        AND table_scans > ( index_scans )
        AND index_count < 2
        AND table_bytes > 32000000   
        AND writes_per_scan < ( 1.0 )
    ORDER BY table_scans DESC
),
big_tables_with_scans AS (
    SELECT 'big table scans'::TEXT as reason,
        database, schema_name, table_name,
        table_scans, pg_size_pretty(table_bytes) as table_size,
        writes_per_scan, index_count, table_bytes
    FROM index_usage JOIN index_counts USING ( relid )
    WHERE table_scans > 100
        AND table_scans > ( index_scans / 10 )
        AND table_bytes > 1000000000  
        AND writes_per_scan < ( 1.0 )
    ORDER BY table_bytes DESC
),
scans_no_writes AS (
    SELECT 'scans, no writes'::TEXT as reason,
        database, schema_name, table_name,
        table_scans, pg_size_pretty(table_bytes) as table_size,
        writes_per_scan, index_count, table_bytes
    FROM index_usage JOIN index_counts USING ( relid )
    WHERE table_scans > 100
        AND table_scans > ( index_scans / 4 )
        AND table_bytes > 32000000   
        AND writes_per_scan < ( 0.1 )
    ORDER BY writes_per_scan ASC
)
SELECT reason, database, schema_name, table_name, table_scans, 
    table_size, writes_per_scan, index_count
FROM too_many_tablescans
UNION ALL
SELECT reason, database, schema_name, table_name, table_scans, 
    table_size, writes_per_scan, index_count
FROM scans_no_index
UNION ALL
SELECT reason, database, schema_name, table_name, table_scans, 
    table_size, writes_per_scan, index_count
FROM big_tables_with_scans
UNION ALL
SELECT reason, database, schema_name, table_name, table_scans, 
    table_size, writes_per_scan, index_count
FROM scans_no_writes;
```

### Unused Indexes

```sql
WITH table_scans as (
    SELECT relid,
        tables.idx_scan + tables.seq_scan as all_scans,
        ( tables.n_tup_ins + tables.n_tup_upd + tables.n_tup_del ) as writes,
                pg_relation_size(relid) as table_size
        FROM pg_stat_user_tables as tables
),
all_writes as (
    SELECT sum(writes) as total_writes
    FROM table_scans
),
indexes as (
    SELECT idx_stat.relid, idx_stat.indexrelid,
        idx_stat.schemaname, idx_stat.relname as tablename,
        idx_stat.indexrelname as indexname,
        idx_stat.idx_scan,
        pg_relation_size(idx_stat.indexrelid) as index_bytes,
        indexdef ~* 'USING btree' AS idx_is_btree
    FROM pg_stat_user_indexes as idx_stat
        JOIN pg_index
            USING (indexrelid)
        JOIN pg_indexes as indexes
            ON idx_stat.schemaname = indexes.schemaname
                AND idx_stat.relname = indexes.tablename
                AND idx_stat.indexrelname = indexes.indexname
    WHERE pg_index.indisunique = FALSE
),
index_ratios AS (
SELECT schemaname, tablename, indexname,
    idx_scan, all_scans,
    round(( CASE WHEN all_scans = 0 THEN 0.0::NUMERIC
        ELSE idx_scan::NUMERIC/all_scans * 100 END),2) as index_scan_pct,
    writes,
    round((CASE WHEN writes = 0 THEN idx_scan::NUMERIC ELSE idx_scan::NUMERIC/writes END),2)
        as scans_per_write,
    pg_size_pretty(index_bytes) as index_size,
    pg_size_pretty(table_size) as table_size,
    idx_is_btree, index_bytes
    FROM indexes
    JOIN table_scans
    USING (relid)
),
index_groups AS (
SELECT 'Never Used Indexes' as reason, *, 1 as grp
FROM index_ratios
WHERE
    idx_scan = 0
    and idx_is_btree
UNION ALL
SELECT 'Low Scans, High Writes' as reason, *, 2 as grp
FROM index_ratios
WHERE
    scans_per_write <= 1
    and index_scan_pct < 10
    and idx_scan > 0
    and writes > 100
    and idx_is_btree
UNION ALL
SELECT 'Seldom Used Large Indexes' as reason, *, 3 as grp
FROM index_ratios
WHERE
    index_scan_pct < 5
    and scans_per_write > 1
    and idx_scan > 0
    and idx_is_btree
    and index_bytes > 100000000
UNION ALL
SELECT 'High-Write Large Non-Btree' as reason, index_ratios.*, 4 as grp 
FROM index_ratios, all_writes
WHERE
    ( writes::NUMERIC / ( total_writes + 1 ) ) > 0.02
    AND NOT idx_is_btree
    AND index_bytes > 100000000
ORDER BY grp, index_bytes DESC )
SELECT reason, schemaname, tablename, indexname,
    index_scan_pct, scans_per_write, index_size, table_size
FROM index_groups;
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
