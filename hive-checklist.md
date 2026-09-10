# Hive Checklist

Reference for the Hive setup in `spark_local_setup`. Covers the stack, how to connect, and known gotchas (all of which were real bugs hit and fixed while setting this up).

## The stack

| Service | Role | Port |
|---|---|---|
| `hive-metastore` | Stores table metadata (names, schemas, storage locations). PySpark talks to this directly. | 9083 (thrift) |
| `hive-server` | HiveServer2 — SQL query endpoint for `beeline`/JDBC clients. Talks to `hive-metastore` internally. | 10000 (JDBC), 10002 (web UI) |

Both run from the same custom image (`hive/Dockerfile`, built from `apache/hive:4.0.0` + S3 jars).

Config files (baked into the Hive image via `hive/Dockerfile`):
- `hive/hive-site.xml` — metastore URI, warehouse dir, Derby DB path, Tez/notification-poll disables
- `hive/core-site.xml` — umask fix so new files/folders are world-writable automatically

## Connecting

**Via PySpark (Jupyter)** — add `.enableHiveSupport()` to your SparkSession:

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("YourAppNameHere") \
    .master("spark://spark-master:7077") \
    .enableHiveSupport() \
    .config("spark.executor.cores", "2") \
    .config("spark.cores.max", "4") \
    .config("spark.executor.memory", "2g") \
    .getOrCreate()

spark.sparkContext.setLogLevel("ERROR")
```

No extra config needed beyond `.enableHiveSupport()` — `conf/hive-site.xml` (mounted into the `jupyter` container) already points Spark at the right metastore.

```python
spark.sql("CREATE TABLE my_table (id INT, name STRING)")
spark.sql("INSERT INTO my_table VALUES (1, 'example')")
spark.sql("SELECT * FROM my_table").show()
```

End with `spark.stop()` as usual.

**Via `beeline` (separate terminal, not Jupyter):**
```powershell
docker exec -it hive-server beeline -u jdbc:hive2://localhost:10000
```
Then run SQL directly at the `0: jdbc:hive2://localhost:10000>` prompt:
```sql
SHOW DATABASES;
USE default;
SHOW TABLES;
SELECT * FROM your_table;
```
Exit with `!quit`.

**Via browser (lightweight, sessions/queries only — not a query editor):** see URLs section below.

## Restarting Hive services

Prefer this:
```powershell
docker compose up -d --force-recreate hive-metastore hive-server
```
over `docker restart <container>` — a plain restart reuses the container's filesystem, which can leave stale lock/PID files behind and cause the service to refuse to start ("already running").

`hive-server` can take up to ~60-90 seconds to come up after a restart (it retries connecting to the metastore a few times before succeeding) — this is normal, not a hang.

## Known gotchas (already fixed, but useful if they resurface)

- **`Cannot create staging directory` on INSERT** → permissions issue on a newly created table folder. Fixed via `fs.permissions.umask-mode=000` in `core-site.xml`. If it reappears: `docker exec -u root hive-metastore chmod -R 777 /opt/hive/data/warehouse`.
- **`ERROR XBM0J: Directory ... already exists`** during metastore schema init → Derby's `create=true` fails if Docker pre-creates the exact target folder via a volume mount. Fixed by mounting the volume one level up (`metastore-store`) and pointing `javax.jdo.option.ConnectionURL` at a subpath inside it.
- **Tables/metadata disappear after a metastore restart** → the `metastore-store` volume wasn't mounted, or was wiped. Check `docker volume ls` for `spark_local_setup_metastore-store`.
- **HiveServer2 hangs for minutes on startup, never binds port 10000** → was caused by Tez trying to initialize sessions against a non-existent YARN cluster. Fixed via `hive.server2.tez.initialize.default.sessions=false` and `hive.execution.engine=mr`.
- **HiveServer2 fails with `Error initializing notification event poll`** → a materialized-views feature that needs a metastore notification listener we don't have configured. Fixed via `hive.notification.event.poll.interval=0`.
- **Console logs (`docker logs`) show almost nothing useful on failure** → Hive's real error log is a file inside the container, not stdout. Pull it out with:
  ```powershell
  docker cp hive-server:/tmp/root/hive.log .\hive-server.log
  ```

## Quick health check

```powershell
docker ps
```
Both `hive-metastore` and `hive-server` should show `Up`, not `Exited` or restarting repeatedly.

```powershell
docker exec -it hive-server beeline -u jdbc:hive2://localhost:10000 -e "SHOW TABLES;"
```
A one-shot query — if this returns cleanly, the whole chain (beeline → HiveServer2 → metastore) is healthy.

---

## URLs (Hive)

| What | URL |
|---|---|
| HiveServer2 Web UI (sessions, queries, config — not a query editor) | http://localhost:10002 |

Hive has no equivalent of the Spark Master/Application UI for job execution — actual data processing in this stack happens through Spark (see `pyspark-new-file-checklist.md` for Spark UI URLs), not through Hive itself.
