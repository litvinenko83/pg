### запрос для бысрой выборки параметров pg (сравнивал параметры PROD и STAGE), которые могут повлиять на поведение SELECT/INSERT/UPDATE при нагрузочном тестировании

SELECT
    name,
    setting,
    unit,
    short_desc
FROM pg_settings
WHERE name IN (
    -- Память и кэши
    'shared_buffers',
    'effective_cache_size',
    'work_mem',
    'maintenance_work_mem',
    'autovacuum_work_mem',
    'wal_buffers',
    'temp_buffers',

    -- Параллельные workers
    'max_connections',
    'max_worker_processes',
    'max_parallel_workers',
    'max_parallel_workers_per_gather',
    'max_parallel_maintenance_workers',

    -- Автовакуум
    'autovacuum_max_workers',
    'autovacuum_vacuum_scale_factor',
    'autovacuum_analyze_scale_factor',
    'autovacuum_vacuum_cost_limit',

    -- WAL и дисковые операции
    'wal_level',
    'fsync',
    'synchronous_commit',
    'wal_compression',
    'checkpoint_timeout',
    'max_wal_size',
    'min_wal_size',
    'wal_writer_delay',
    'commit_delay',
    'commit_siblings',

    -- Планировщик (cost)
    'random_page_cost',
    'seq_page_cost',
    'cpu_tuple_cost',
    'cpu_index_tuple_cost',
    'cpu_operator_cost',
    'effective_io_concurrency',
    'maintenance_io_concurrency',

    -- JIT
    'jit',

    -- Таймауты (могут влиять на поведение, но не на скорость)
    'statement_timeout',
    'lock_timeout',
    'idle_in_transaction_session_timeout',

    -- Другое
    'default_statistics_target',
    'geqo_threshold',
    'constraint_exclusion'
)
ORDER BY name;
