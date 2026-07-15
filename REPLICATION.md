### просмотр статуса физической репликации на лидере (запрос в Postgresql 12 точно работает)

```sql
SELECT pid, application_name, client_addr, state,
       sync_state, -- синхронный или асинхронный режим
       write_lag, flush_lag, replay_lag -- задержки (появятся, если они значительные)
FROM pg_stat_replication;
```

### просмотр статуса физической репликации на репликах (запрос в Postgresql 12 точно работает)

```sql
SELECT
    pid,
    status,
    sender_host || ':' || sender_port AS master_connection,
    last_msg_receipt_time,
    now() - last_msg_receipt_time AS receipt_lag_interval,
    pg_last_xact_replay_timestamp() AS last_replayed_timestamp,
    CASE
        WHEN pg_last_xact_replay_timestamp() IS NOT NULL
        THEN extract(epoch FROM now() - pg_last_xact_replay_timestamp())::numeric(10,3)
        ELSE NULL
    END AS replay_lag_seconds,
    pg_last_wal_replay_lsn() AS last_replayed_lsn,
    received_lsn AS last_received_lsn,
    pg_wal_lsn_diff(received_lsn, pg_last_wal_replay_lsn()) AS received_not_replayed_bytes,
    latest_end_lsn,
    slot_name,
    -- Простая оценка состояния (можно добавить свои критерии)
    CASE
        WHEN status != 'streaming' THEN '⚠️  Не в режиме streaming'
        WHEN now() - last_msg_receipt_time > interval '10 seconds' THEN '⚠️  Большая задержка получения'
        WHEN pg_last_xact_replay_timestamp() IS NOT NULL
             AND extract(epoch FROM now() - pg_last_xact_replay_timestamp()) > 10 THEN '⚠️  Большая задержка воспроизведения'
        WHEN pg_wal_lsn_diff(received_lsn, pg_last_wal_replay_lsn()) > 64 * 1024 * 1024 THEN '⚠️  Много невоспроизведённого WAL (>64MB)'
        ELSE '✅  Работает штатно'
    END AS health_check
FROM pg_stat_wal_receiver;
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
