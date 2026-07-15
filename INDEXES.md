### SQL-запрос для вывода всех индексов (основной таблицы и её TOAST-таблицы)
```sql
-- индексы основной таблицы
SELECT
    n.nspname AS schema_name,
    c.relname AS table_name,
    i.relname AS index_name,
    pg_get_indexdef(i.oid) AS index_def,
    idx.indisvalid AS is_valid
FROM pg_class i
JOIN pg_index idx ON i.oid = idx.indexrelid
JOIN pg_class c ON idx.indrelid = c.oid
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE c.oid = 'public.goods_sell_2602'::regclass

UNION ALL

-- индексы TOAST-таблицы
SELECT
    n.nspname AS schema_name,
    c.relname AS table_name,
    i.relname AS index_name,
    pg_get_indexdef(i.oid) AS index_def,
    idx.indisvalid AS is_valid
FROM pg_class i
JOIN pg_index idx ON i.oid = idx.indexrelid
JOIN pg_class c ON idx.indrelid = c.oid
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE c.oid = (SELECT reltoastrelid FROM pg_class WHERE oid = 'public.goods_sell_2602'::regclass);
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
--WHERE table_schema = 'public' 
ORDER BY total_bytes DESC; 
```
*http://www.gilev.ru/sizetablepostgre/*



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

