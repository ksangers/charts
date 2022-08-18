# cloudnative-pg

![Version: 0.14.3](https://img.shields.io/badge/Version-0.14.3-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 1.16.1](https://img.shields.io/badge/AppVersion-1.16.1-informational?style=flat-square)

CloudNativePG Helm Chart

**Homepage:** <https://cloudnative-pg.io>

## Maintainers

| Name | Email | Url |
| ---- | ------ | --- |
| phisco | <p.scorsolini@gmail.com> |  |

## Source Code

* <https://github.com/cloudnative-pg/charts>

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| additionalArgs | list | `[]` | Additinal arguments to be added to the operator's args list |
| affinity | object | `{}` | Affinity for the operator to be installed |
| commonAnnotations | object | `{}` | Annotations to be added to all other resources |
| config.create | bool | `true` | Specifies whether the secret should be created |
| config.data | object | `{}` |  |
| config.name | string | `"cnpg-controller-manager-config"` |  |
| config.secret | bool | `false` | Specifies whether it should be stored in a secret, instead of a configmap |
| crds.create | bool | `true` |  |
| fullnameOverride | string | `""` |  |
| image.pullPolicy | string | `"IfNotPresent"` |  |
| image.repository | string | `"ghcr.io/cloudnative-pg/cloudnative-pg"` |  |
| image.tag | string | `""` | Overrides the image tag whose default is the chart appVersion. |
| imagePullSecrets | list | `[]` |  |
| monitoringQueriesConfigMap.name | string | `"cnpg-default-monitoring"` | The name of the default monitoring configmap |
| monitoringQueriesConfigMap.queries | string | `"backends:\n  query: |\n   SELECT sa.datname\n       , sa.usename\n       , sa.application_name\n       , states.state\n       , COALESCE(sa.count, 0) AS total\n       , COALESCE(sa.max_tx_secs, 0) AS max_tx_duration_seconds\n       FROM ( VALUES ('active')\n           , ('idle')\n           , ('idle in transaction')\n           , ('idle in transaction (aborted)')\n           , ('fastpath function call')\n           , ('disabled')\n           ) AS states(state)\n       LEFT JOIN (\n           SELECT datname\n               , state\n               , usename\n               , COALESCE(application_name, '') AS application_name\n               , COUNT(*)\n               , COALESCE(EXTRACT (EPOCH FROM (max(now() - xact_start))), 0) AS max_tx_secs\n           FROM pg_catalog.pg_stat_activity\n           GROUP BY datname, state, usename, application_name\n       ) sa ON states.state = sa.state\n       WHERE sa.usename IS NOT NULL\n  metrics:\n    - datname:\n        usage: \"LABEL\"\n        description: \"Name of the database\"\n    - usename:\n        usage: \"LABEL\"\n        description: \"Name of the user\"\n    - application_name:\n        usage: \"LABEL\"\n        description: \"Name of the application\"\n    - state:\n        usage: \"LABEL\"\n        description: \"State of the backend\"\n    - total:\n        usage: \"GAUGE\"\n        description: \"Number of backends\"\n    - max_tx_duration_seconds:\n        usage: \"GAUGE\"\n        description: \"Maximum duration of a transaction in seconds\"\n\nbackends_waiting:\n  query: |\n   SELECT count(*) AS total\n   FROM pg_catalog.pg_locks blocked_locks\n   JOIN pg_catalog.pg_locks blocking_locks\n     ON blocking_locks.locktype = blocked_locks.locktype\n     AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database\n     AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation\n     AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page\n     AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple\n     AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid\n     AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid\n     AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid\n     AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid\n     AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid\n     AND blocking_locks.pid != blocked_locks.pid\n   JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid\n   WHERE NOT blocked_locks.granted\n  metrics:\n    - total:\n        usage: \"GAUGE\"\n        description: \"Total number of backends that are currently waiting on other queries\"\n\npg_database:\n  query: |\n    SELECT datname\n      , pg_catalog.pg_database_size(datname) AS size_bytes\n      , pg_catalog.age(datfrozenxid) AS xid_age\n      , pg_catalog.mxid_age(datminmxid) AS mxid_age\n    FROM pg_catalog.pg_database\n  metrics:\n    - datname:\n        usage: \"LABEL\"\n        description: \"Name of the database\"\n    - size_bytes:\n        usage: \"GAUGE\"\n        description: \"Disk space used by the database\"\n    - xid_age:\n        usage: \"GAUGE\"\n        description: \"Number of transactions from the frozen XID to the current one\"\n    - mxid_age:\n        usage: \"GAUGE\"\n        description: \"Number of multiple transactions (Multixact) from the frozen XID to the current one\"\n\npg_postmaster:\n  query: |\n    SELECT EXTRACT(EPOCH FROM pg_postmaster_start_time) AS start_time\n    FROM pg_catalog.pg_postmaster_start_time()\n  metrics:\n    - start_time:\n        usage: \"GAUGE\"\n        description: \"Time at which postgres started (based on epoch)\"\n\npg_replication:\n  primary: true\n  query: \"SELECT CASE WHEN NOT pg_catalog.pg_is_in_recovery()\n          THEN 0\n          ELSE GREATEST (0,\n            EXTRACT(EPOCH FROM (now() - pg_catalog.pg_last_xact_replay_timestamp())))\n          END AS lag,\n          pg_catalog.pg_is_in_recovery() AS in_recovery,\n          EXISTS (TABLE pg_stat_wal_receiver) AS is_wal_receiver_up,\n          (SELECT count(*) FROM pg_stat_replication) AS streaming_replicas\"\n  metrics:\n    - lag:\n        usage: \"GAUGE\"\n        description: \"Replication lag behind primary in seconds\"\n    - in_recovery:\n        usage: \"GAUGE\"\n        description: \"Whether the instance is in recovery\"\n    - is_wal_receiver_up:\n        usage: \"GAUGE\"\n        description: \"Whether the instance wal_receiver is up\"\n    - streaming_replicas:\n        usage: \"GAUGE\"\n        description: \"Number of streaming replicas connected to the instance\"\n\npg_stat_archiver:\n  query: |\n    SELECT archived_count\n      , failed_count\n      , COALESCE(EXTRACT(EPOCH FROM (now() - last_archived_time)), -1) AS seconds_since_last_archival\n      , COALESCE(EXTRACT(EPOCH FROM (now() - last_failed_time)), -1) AS seconds_since_last_failure\n      , COALESCE(EXTRACT(EPOCH FROM last_archived_time), -1) AS last_archived_time\n      , COALESCE(EXTRACT(EPOCH FROM last_failed_time), -1) AS last_failed_time\n      , COALESCE(CAST(CAST('x'||pg_catalog.right(pg_catalog.split_part(last_archived_wal, '.', 1), 16) AS pg_catalog.bit(64)) AS pg_catalog.int8), -1) AS last_archived_wal_start_lsn\n      , COALESCE(CAST(CAST('x'||pg_catalog.right(pg_catalog.split_part(last_failed_wal, '.', 1), 16) AS pg_catalog.bit(64)) AS pg_catalog.int8), -1) AS last_failed_wal_start_lsn\n      , EXTRACT(EPOCH FROM stats_reset) AS stats_reset_time\n    FROM pg_catalog.pg_stat_archiver\n  metrics:\n    - archived_count:\n        usage: \"COUNTER\"\n        description: \"Number of WAL files that have been successfully archived\"\n    - failed_count:\n        usage: \"COUNTER\"\n        description: \"Number of failed attempts for archiving WAL files\"\n    - seconds_since_last_archival:\n        usage: \"GAUGE\"\n        description: \"Seconds since the last successful archival operation\"\n    - seconds_since_last_failure:\n        usage: \"GAUGE\"\n        description: \"Seconds since the last failed archival operation\"\n    - last_archived_time:\n        usage: \"GAUGE\"\n        description: \"Epoch of the last time WAL archiving succeeded\"\n    - last_failed_time:\n        usage: \"GAUGE\"\n        description: \"Epoch of the last time WAL archiving failed\"\n    - last_archived_wal_start_lsn:\n        usage: \"GAUGE\"\n        description: \"Archived WAL start LSN\"\n    - last_failed_wal_start_lsn:\n        usage: \"GAUGE\"\n        description: \"Last failed WAL LSN\"\n    - stats_reset_time:\n        usage: \"GAUGE\"\n        description: \"Time at which these statistics were last reset\"\n\npg_stat_bgwriter:\n  query: |\n    SELECT checkpoints_timed\n      , checkpoints_req\n      , checkpoint_write_time\n      , checkpoint_sync_time\n      , buffers_checkpoint\n      , buffers_clean\n      , maxwritten_clean\n      , buffers_backend\n      , buffers_backend_fsync\n      , buffers_alloc\n    FROM pg_catalog.pg_stat_bgwriter\n  metrics:\n    - checkpoints_timed:\n        usage: \"COUNTER\"\n        description: \"Number of scheduled checkpoints that have been performed\"\n    - checkpoints_req:\n        usage: \"COUNTER\"\n        description: \"Number of requested checkpoints that have been performed\"\n    - checkpoint_write_time:\n        usage: \"COUNTER\"\n        description: \"Total amount of time that has been spent in the portion of checkpoint processing where files are written to disk, in milliseconds\"\n    - checkpoint_sync_time:\n        usage: \"COUNTER\"\n        description: \"Total amount of time that has been spent in the portion of checkpoint processing where files are synchronized to disk, in milliseconds\"\n    - buffers_checkpoint:\n        usage: \"COUNTER\"\n        description: \"Number of buffers written during checkpoints\"\n    - buffers_clean:\n        usage: \"COUNTER\"\n        description: \"Number of buffers written by the background writer\"\n    - maxwritten_clean:\n        usage: \"COUNTER\"\n        description: \"Number of times the background writer stopped a cleaning scan because it had written too many buffers\"\n    - buffers_backend:\n        usage: \"COUNTER\"\n        description: \"Number of buffers written directly by a backend\"\n    - buffers_backend_fsync:\n        usage: \"COUNTER\"\n        description: \"Number of times a backend had to execute its own fsync call (normally the background writer handles those even when the backend does its own write)\"\n    - buffers_alloc:\n        usage: \"COUNTER\"\n        description: \"Number of buffers allocated\"\n\npg_stat_database:\n  query: |\n    SELECT datname\n      , xact_commit\n      , xact_rollback\n      , blks_read\n      , blks_hit\n      , tup_returned\n      , tup_fetched\n      , tup_inserted\n      , tup_updated\n      , tup_deleted\n      , conflicts\n      , temp_files\n      , temp_bytes\n      , deadlocks\n      , blk_read_time\n      , blk_write_time\n    FROM pg_catalog.pg_stat_database\n  metrics:\n    - datname:\n        usage: \"LABEL\"\n        description: \"Name of this database\"\n    - xact_commit:\n        usage: \"COUNTER\"\n        description: \"Number of transactions in this database that have been committed\"\n    - xact_rollback:\n        usage: \"COUNTER\"\n        description: \"Number of transactions in this database that have been rolled back\"\n    - blks_read:\n        usage: \"COUNTER\"\n        description: \"Number of disk blocks read in this database\"\n    - blks_hit:\n        usage: \"COUNTER\"\n        description: \"Number of times disk blocks were found already in the buffer cache, so that a read was not necessary (this only includes hits in the PostgreSQL buffer cache, not the operating system's file system cache)\"\n    - tup_returned:\n        usage: \"COUNTER\"\n        description: \"Number of rows returned by queries in this database\"\n    - tup_fetched:\n        usage: \"COUNTER\"\n        description: \"Number of rows fetched by queries in this database\"\n    - tup_inserted:\n        usage: \"COUNTER\"\n        description: \"Number of rows inserted by queries in this database\"\n    - tup_updated:\n        usage: \"COUNTER\"\n        description: \"Number of rows updated by queries in this database\"\n    - tup_deleted:\n        usage: \"COUNTER\"\n        description: \"Number of rows deleted by queries in this database\"\n    - conflicts:\n        usage: \"COUNTER\"\n        description: \"Number of queries canceled due to conflicts with recovery in this database\"\n    - temp_files:\n        usage: \"COUNTER\"\n        description: \"Number of temporary files created by queries in this database\"\n    - temp_bytes:\n        usage: \"COUNTER\"\n        description: \"Total amount of data written to temporary files by queries in this database\"\n    - deadlocks:\n        usage: \"COUNTER\"\n        description: \"Number of deadlocks detected in this database\"\n    - blk_read_time:\n        usage: \"COUNTER\"\n        description: \"Time spent reading data file blocks by backends in this database, in milliseconds\"\n    - blk_write_time:\n        usage: \"COUNTER\"\n        description: \"Time spent writing data file blocks by backends in this database, in milliseconds\"\n\npg_stat_replication:\n  query: |\n   SELECT usename\n     , COALESCE(application_name, '') AS application_name\n     , COALESCE(client_addr::text, '') AS client_addr\n     , EXTRACT(EPOCH FROM backend_start) AS backend_start\n     , COALESCE(pg_catalog.age(backend_xmin), 0) AS backend_xmin_age\n     , pg_catalog.pg_wal_lsn_diff(pg_catalog.pg_current_wal_lsn(), sent_lsn) AS sent_diff_bytes\n     , pg_catalog.pg_wal_lsn_diff(pg_catalog.pg_current_wal_lsn(), write_lsn) AS write_diff_bytes\n     , pg_catalog.pg_wal_lsn_diff(pg_catalog.pg_current_wal_lsn(), flush_lsn) AS flush_diff_bytes\n     , COALESCE(pg_catalog.pg_wal_lsn_diff(pg_catalog.pg_current_wal_lsn(), replay_lsn),0) AS replay_diff_bytes\n     , COALESCE((EXTRACT(EPOCH FROM write_lag)),0)::float AS write_lag_seconds\n     , COALESCE((EXTRACT(EPOCH FROM flush_lag)),0)::float AS flush_lag_seconds\n     , COALESCE((EXTRACT(EPOCH FROM replay_lag)),0)::float AS replay_lag_seconds\n   FROM pg_catalog.pg_stat_replication\n  metrics:\n    - usename:\n        usage: \"LABEL\"\n        description: \"Name of the replication user\"\n    - application_name:\n        usage: \"LABEL\"\n        description: \"Name of the application\"\n    - client_addr:\n        usage: \"LABEL\"\n        description: \"Client IP address\"\n    - backend_start:\n        usage: \"COUNTER\"\n        description: \"Time when this process was started\"\n    - backend_xmin_age:\n        usage: \"COUNTER\"\n        description: \"The age of this standby's xmin horizon\"\n    - sent_diff_bytes:\n        usage: \"GAUGE\"\n        description: \"Difference in bytes from the last write-ahead log location sent on this connection\"\n    - write_diff_bytes:\n        usage: \"GAUGE\"\n        description: \"Difference in bytes from the last write-ahead log location written to disk by this standby server\"\n    - flush_diff_bytes:\n        usage: \"GAUGE\"\n        description: \"Difference in bytes from the last write-ahead log location flushed to disk by this standby server\"\n    - replay_diff_bytes:\n        usage: \"GAUGE\"\n        description: \"Difference in bytes from the last write-ahead log location replayed into the database on this standby server\"\n    - write_lag_seconds:\n        usage: \"GAUGE\"\n        description: \"Time elapsed between flushing recent WAL locally and receiving notification that this standby server has written it\"\n    - flush_lag_seconds:\n        usage: \"GAUGE\"\n        description: \"Time elapsed between flushing recent WAL locally and receiving notification that this standby server has written and flushed it\"\n    - replay_lag_seconds:\n        usage: \"GAUGE\"\n        description: \"Time elapsed between flushing recent WAL locally and receiving notification that this standby server has written, flushed and applied it\"\n\npg_settings:\n  query: |\n    SELECT name,\n    CASE setting WHEN 'on' THEN '1' WHEN 'off' THEN '0' ELSE setting END AS setting\n    FROM pg_catalog.pg_settings\n    WHERE vartype IN ('integer', 'real', 'bool')\n    ORDER BY 1\n  metrics:\n    - name:\n        usage: \"LABEL\"\n        description: \"Name of the setting\"\n    - setting:\n        usage: \"GAUGE\"\n        description: \"Setting value\"\n"` | A string representation of a YAML defining monitoring queries |
| nameOverride | string | `""` |  |
| nodeSelector | object | `{}` | Nodeselector for the operator to be installed |
| podAnnotations | object | `{}` | Annotations to be added to the pod |
| podSecurityContext | object | `{"runAsNonRoot":true,"seccompProfile":{"type":"RuntimeDefault"}}` | Security Context for the whole pod |
| rbac.create | bool | `true` | Specifies whether ClusterRole and ClusterRoleBinding should be created |
| replicaCount | int | `1` |  |
| resources | object | `{}` |  |
| service.name | string | `"cnpg-webhook-service"` | DO NOT CHANGE THE SERVICE NAME as it is currently used to generate the certificate and can not be configured |
| service.port | int | `443` |  |
| service.type | string | `"ClusterIP"` |  |
| serviceAccount.create | bool | `true` | Specifies whether the service account should be created |
| serviceAccount.name | string | `""` | The name of the service account to use. If not set and create is true, a name is generated using the fullname template |
| tolerations | list | `[]` | Tolerations for the operator to be installed |
| webhook.mutating.create | bool | `true` |  |
| webhook.mutating.failurePolicy | string | `"Fail"` |  |
| webhook.port | int | `9443` |  |
| webhook.validating.create | bool | `true` |  |
| webhook.validating.failurePolicy | string | `"Fail"` |  |

----------------------------------------------
Autogenerated from chart metadata using [helm-docs v1.11.0](https://github.com/norwoodj/helm-docs/releases/v1.11.0)
