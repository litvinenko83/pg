### AUTOVACUUM когда срабатывает

*https://habr.com/ru/companies/postgrespro/articles/452762/*

```sql
--условие запуска автовакуум 
pg_stat_all_tables.n_dead_tup >= autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * pg_class.reltupes 

--условие запуска аналайза 
pg_stat_all_tables.n_mod_since_analyze >= autovacuum_analyze_threshold + autovacuum_analyze_scale_factor * pg_class.reltupes 
```

### максимальная скорость чтения с диска при AUTOVACUUM-ах, исходя из текущих настроек
основано на формулах из статьи https://pganalyze.com/docs/vacuum-advisor/how-does-the-vacuum-cost-model-work

```sql
--максимальная скорость чтения, исходя из текущих настроек
WITH settings AS (
    SELECT
        (current_setting('block_size'))::int AS block_size,
        (current_setting('autovacuum_vacuum_cost_limit'))::numeric AS cost_limit,
        (current_setting('vacuum_cost_page_miss'))::numeric AS cost_page_miss,
        substring(current_setting('autovacuum_vacuum_cost_delay') FROM '^(\d+(\.\d+)?)')::numeric AS cost_delay_ms
)
SELECT
    block_size,
    cost_limit,
    cost_page_miss,
    cost_delay_ms,

    -- Скорость чтения (байт/с)
    CASE
        WHEN cost_delay_ms > 0 THEN
            (cost_limit / cost_page_miss * block_size) / (cost_delay_ms / 1000)
        ELSE NULL
    END AS read_rate_bytes_per_sec,

    -- Скорость чтения (МБ/с)
    CASE
        WHEN cost_delay_ms > 0 THEN
            ((cost_limit / cost_page_miss * block_size) / (cost_delay_ms / 1000)) / (1024 * 1024)
        ELSE NULL
    END AS read_rate_mb_per_sec,

    -- Текст формулы (с именами переменных)
    '(cost_limit / cost_page_miss * block_size) / (cost_delay_ms / 1000)' AS formula_text,

    -- Текст формулы с подставленными значениями
    CASE
        WHEN cost_delay_ms > 0 THEN
            '(' || cost_limit || ' / ' || cost_page_miss || ' * ' || block_size || ') / (' || cost_delay_ms || ' / 1000)'
        ELSE
            'cost_delay_ms <= 0 – скорость не лимитируется'
    END AS formula_with_values

FROM settings;
```

### максимальная скорость записи на диск при AUTOVACUUM-ах, исходя из текущих настроек
```sql
--максимальная скорость записи на диск при автовакууме, исходя из текущих настроек
WITH settings AS (
    SELECT
        (current_setting('block_size'))::int AS block_size,
        (current_setting('autovacuum_vacuum_cost_limit'))::numeric AS cost_limit,
        (current_setting('vacuum_cost_page_dirty'))::numeric AS cost_page_dirty,
        substring(current_setting('autovacuum_vacuum_cost_delay') FROM '^(\d+(\.\d+)?)')::numeric AS cost_delay_ms
)
SELECT
    block_size,
    cost_limit,
    cost_page_dirty,
    cost_delay_ms,

    -- Скорость записи (байт/с)
    CASE
        WHEN cost_delay_ms > 0 THEN
            (cost_limit / cost_page_dirty * block_size) / (cost_delay_ms / 1000)
        ELSE NULL
    END AS write_rate_bytes_per_sec,

    -- Скорость записи (МБ/с)
    CASE
        WHEN cost_delay_ms > 0 THEN
            ((cost_limit / cost_page_dirty * block_size) / (cost_delay_ms / 1000)) / (1024 * 1024)
        ELSE NULL
    END AS write_rate_mb_per_sec,

    -- Текст формулы (с именами переменных)
    '(cost_limit / cost_page_dirty * block_size) / (cost_delay_ms / 1000)' AS formula_text,

    -- Текст формулы с подставленными значениями
    CASE
        WHEN cost_delay_ms > 0 THEN
            '(' || cost_limit || ' / ' || cost_page_dirty || ' * ' || block_size || ') / (' || cost_delay_ms || ' / 1000)'
        ELSE
            'cost_delay_ms <= 0 – скорость не лимитируется'
    END AS formula_with_values

FROM settings;
```

### Сколько мертвых строк осталось нагенерить update-ами/delete-ами до запуска следующего автовакуума

```sql
SELECT
    stat.schemaname as "схема",
    stat.relname AS "таблица",
    stat.n_dead_tup::BIGINT as n_dead_tup,
    stat.last_autovacuum as last_autovacuum,
--    stat.last_vacuum,
    stat.last_autoanalyze as last_autoanalyze,
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
    stat.schemaname = 'public'
    AND stat.relname = 'goods';
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
