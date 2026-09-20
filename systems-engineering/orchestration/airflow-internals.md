# Apache Airflow Internals

## 30-Second Intuition

Airflow is a DAG scheduler: you define Python files describing task dependencies, and a scheduler loop continuously parses those files, creates DAG runs at the right intervals, and pushes tasks to workers. The metadata DB (Postgres/MySQL) is the single source of truth for all state — every task state transition, XCom value, and schedule decision is a DB write, which makes it both the heartbeat and the bottleneck.

> **Note (as of July 2026): Airflow 3.0 shipped (GA April 22, 2025) and is now at 3.2.x.** The internals below describe the **Airflow 2.x** model, which is still accurate for 2.x deployments and still the best way to understand the fundamentals (DagRun/TaskInstance state machine, XCom, deferrable operators, etc. are conceptually unchanged). Airflow 3.x keeps these concepts but changes *where* execution and DB access happen — see the dedicated **"Airflow 3.x: What Changed"** section near the end of this doc before assuming workers have direct DB/file access.

---

## DAG Parsing

### How the Scheduler Finds DAG Files

```
$AIRFLOW_HOME/dags/           ← default dag_folder
  my_pipeline.py
  subdir/
    another_dag.py
```

The **DagFileProcessorManager** watches the dag_folder (recursive by default). It:
1. Lists all `.py` files (and `.zip` archives).
2. Spawns `DagFileProcessor` subprocesses — one per file — at intervals.
3. Each subprocess imports the file, collects `DAG` objects, serializes them to the metadata DB.

**Key config knobs:**
```ini
[scheduler]
dag_dir_list_interval = 300      # How often to scan dag_folder for new files (seconds)
min_file_process_interval = 30   # Don't re-parse a file more often than this
file_parsing_timeout = 120       # Kill a parser subprocess after N seconds
max_dagruns_per_loop_to_schedule = 20
```

### DAG File Processor Lifecycle

```
DagFileProcessorManager
  │
  ├─ spawn subprocess: python dag_file.py
  │     (full module import — top-level code runs!)
  │
  ├─ subprocess serializes DAG objects → Pickle/JSON → sends back
  │
  └─ Manager writes SerializedDAG to DB
```

**Critical**: Top-level code in a DAG file runs on every parse. Heavy imports or DB calls at module level bloat parse time. Keep instantiation cheap.

### Serialized DAGs (Airflow 2.0+)

Before 2.0, every scheduler and worker re-parsed DAG files. Now:
- DAGs are serialized to `serialized_dag` table on parse.
- The scheduler reads from DB, not files.
- Workers also read SerializedDAG from DB — **dag_folder does not need to be mounted on workers** (with CeleryExecutor or KubernetesExecutor).

---

## Scheduler Internals

### The Scheduler Loop

```
while True:
    1. Sync DAGs from DB (serialized_dag table)
    2. For each active DAG:
       a. Check if a new DagRun should be created (catchup logic)
       b. Create DagRun if needed
    3. For each DagRun in "running" state:
       a. Evaluate task dependencies
       b. Move eligible tasks: scheduled → queued
    4. Send queued tasks to executor
    5. Sync executor state back to DB
    6. Sleep(scheduler_heartbeat_sec)  # default 5s
```

### DagRun Creation

```python
# DAG definition
dag = DAG(
    dag_id="my_dag",
    schedule_interval="@daily",   # or cron expression
    start_date=datetime(2024, 1, 1),
    catchup=False,
    max_active_runs=1,
)
```

The scheduler computes `logical_date` (formerly `execution_date`) for each run:
- `logical_date` = the **start** of the interval (not when it runs).
- A daily DAG with `start_date=2024-01-01` has its first `logical_date=2024-01-01`, but runs *after* 2024-01-02 00:00 (end of interval).

### Task Scheduling

```
TaskInstance state machine:

none → scheduled → queued → running → success
                                    → failed → up_for_retry → queued ...
                                    → upstream_failed
```

The scheduler evaluates task dependencies by checking all upstream TIs for the same DagRun. If all upstreams are `success`, the task moves to `scheduled`.

---

## Executor Types

### LocalExecutor

```
Scheduler process
  └─ subprocess per task (multiprocessing)
```

- Good for: dev, single-node, small workloads.
- Bad for: any horizontal scale. Single point of failure.
- Config: `parallelism` limits concurrent subprocesses.

### CeleryExecutor

```
Scheduler → pushes task to Celery broker (Redis/RabbitMQ)
               ↓
         Celery Workers (N machines) → pull tasks → execute
               ↓
         Celery result backend (Redis/DB) → report status
```

- Workers share the same Python environment and dag_folder (or use SerializedDAG from DB).
- Flower UI for monitoring workers.
- **Trade-offs**: Broker is another infra component. Worker pool is always-on (cost). Good for steady high-throughput workloads.
- Config: `worker_concurrency` per worker.

### KubernetesExecutor

```
Scheduler → creates Kubernetes Pod per task
                  ↓
            Pod runs task → exits
                  ↓
            Scheduler watches Pod phase → updates TI state
```

- **No always-on workers.** Each task = ephemeral pod.
- Perfect isolation: different Docker images per task, no shared process memory.
- Cold start latency: ~10-30s pod spin-up per task.
- Uses `pod_template_file` to define base pod spec; operator-level overrides merge on top.
- **Bad for**: high-frequency short tasks (pod overhead dominates). Good for: heterogeneous resource requirements, security isolation.

### KubernetesExecutor Deep Dive

```yaml
# pod_template_file.yaml
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: base
      image: apache/airflow:2.8.0
      resources:
        requests:
          memory: "2Gi"
          cpu: "500m"
      volumeMounts:
        - name: dags
          mountPath: /opt/airflow/dags
  volumes:
    - name: dags
      persistentVolumeClaim:
        claimName: airflow-dags
```

Per-task overrides via `executor_config`:
```python
task = PythonOperator(
    task_id="heavy_task",
    python_callable=my_fn,
    executor_config={
        "KubernetesExecutor": {
            "request_memory": "8Gi",
            "request_cpu": "4",
            "image": "my-custom:latest",
        }
    }
)
```

Scheduler reconciliation:
1. Scheduler submits Pod via K8s API.
2. Watches `airflow_worker` label selector for Pod phase changes.
3. Maps Pod phase → TI state (Succeeded → success, Failed → failed).

### CeleryKubernetesExecutor

Hybrid: some tasks run on Celery workers (fast, no cold start), others on K8s pods (isolation). Route via `queue`:
```python
task = PythonOperator(queue="kubernetes")   # → K8s pod
task2 = PythonOperator(queue="default")    # → Celery worker
```

---

## Metadata DB

### Schema (key tables)

| Table | What lives there |
|---|---|
| `dag` | DAG definitions (enabled, paused, schedule) |
| `serialized_dag` | JSON-serialized DAG structure (2.0+) |
| `dag_run` | Each DAG execution (logical_date, state, run_id) |
| `task_instance` | Per-task-per-run state, start/end time, try_number |
| `xcom` | Cross-task communication values |
| `connection` | Hook credentials (conn_id → URI/JSON) |
| `variable` | Key-value store for DAG-accessible config |
| `slot_pool` | Resource pools for concurrency limiting |
| `log` | Task log metadata (if using DB log handler) |

### Why It's the Bottleneck

Every state transition = a DB write. At high scale:
- Scheduler loop: SELECT + UPDATE per task per heartbeat.
- Workers: heartbeats write to `task_instance`.
- Many concurrent DAG runs → DB lock contention.

Mitigations:
- Use `max_dagruns_to_create_per_loop` to throttle DagRun creation.
- Use `parallelism` to cap total concurrent TIs.
- Postgres connection pooling (PgBouncer).
- Separate read replicas for UI queries.

---

## XCom

### How It Works

```python
# Push
def push_fn(**context):
    context['ti'].xcom_push(key='result', value={'rows': 1000})

# Pull
def pull_fn(**context):
    val = context['ti'].xcom_pull(task_ids='push_task', key='result')
```

XComs are stored in the `xcom` table:
```sql
xcom(dag_id, task_id, run_id, key, value BLOB, timestamp)
```

**Default limit**: Postgres `bytea` — effectively limited by your willingness to bloat the DB. Practical community guideline: **48KB max** before performance degrades. The real constraint is that XCom is in the metadata DB — large values thrash Postgres.

### XCom Backend Override

For large payloads (DataFrames, model artifacts), override the backend:

```python
# airflow.cfg
[core]
xcom_backend = airflow.providers.amazon.aws.xcom_backends.s3.S3XComBackend

# Or custom:
class S3XComBackend(BaseXCom):
    @staticmethod
    def serialize_value(value):
        key = f"xcom/{uuid4()}.pkl"
        s3_client.put_object(Bucket=BUCKET, Key=key, Body=pickle.dumps(value))
        return key.encode()

    @staticmethod
    def deserialize_value(result):
        key = result.decode()
        obj = s3_client.get_object(Bucket=BUCKET, Key=key)
        return pickle.loads(obj['Body'].read())
```

Now the `xcom` table stores S3 keys, not raw data.

---

## Task Lifecycle (State Machine)

```
         ┌─────────────────────────────────────────────┐
         │                                             │
  none → scheduled → queued → running → success        │
                                      → failed ────────┤
                                      → up_for_retry → queued (if retries left)
                                      → upstream_failed
                                      → skipped
                                      → removed
```

State transitions:
- `none → scheduled`: scheduler evaluates deps, all satisfied.
- `scheduled → queued`: executor accepts the task.
- `queued → running`: worker picks up the task.
- `running → success/failed`: task exits.
- `failed → up_for_retry`: if `retries > 0` and `try_number < retries`.

**`zombie` detection**: If a running TI's heartbeat stops (worker died), the scheduler marks it `failed` after `scheduler_zombie_task_threshold` (default 5 min).

---

## Triggerer and Deferrable Operators

### The Problem

A traditional sensor holds a worker slot while polling:
```python
# Blocks a worker for hours:
S3KeySensor(task_id="wait_for_file", bucket_key="s3://...", poke_interval=60)
```

With 100 sensors running, you need 100 worker slots — most doing nothing.

### The Solution: Deferrable Operators

```python
from airflow.sensors.base import BaseSensorOperator
from airflow.triggers.temporal import TimeDeltaTrigger

class MyDeferrableSensor(BaseSensorOperator):
    def execute(self, context):
        # Immediately defer — release the worker slot
        self.defer(
            trigger=TimeDeltaTrigger(timedelta(minutes=5)),
            method_name="execute_complete"
        )

    def execute_complete(self, context, event=None):
        # Called when trigger fires — gets a new worker slot
        return event
```

### Triggerer Component

```
Triggerer process (asyncio event loop)
  ├─ Loads all deferred TIs from DB
  ├─ Runs triggers as coroutines (no thread per trigger!)
  └─ When trigger fires → updates TI to "scheduled" → scheduler re-queues
```

The Triggerer uses asyncio — thousands of concurrent async checks in one process. Critical for sensors that poll slowly (S3, GCS, Snowflake query completion).

---

## Dynamic DAGs

### @dag Factory Pattern

```python
from airflow.decorators import dag, task
from datetime import datetime

def create_dag(region: str):
    @dag(dag_id=f"pipeline_{region}", start_date=datetime(2024,1,1))
    def pipeline():
        @task
        def extract(): return f"data from {region}"

        @task
        def load(data): print(data)

        load(extract())

    return pipeline()

for region in ["us-east", "eu-west", "ap-south"]:
    create_dag(region)   # registers 3 DAGs at module level
```

**Gotcha**: Each call to `create_dag()` runs at parse time. 100 dynamic DAGs = 100x parse overhead.

### Dynamic Task Mapping (`expand()`)

```python
@dag(schedule="@daily", start_date=datetime(2024,1,1))
def mapped_pipeline():
    @task
    def get_partitions() -> list[str]:
        return ["2024-01-01", "2024-01-02", "2024-01-03"]

    @task
    def process(partition: str):
        print(f"Processing {partition}")

    @task
    def aggregate(results: list):
        print(f"Done: {len(results)} partitions")

    partitions = get_partitions()
    results = process.expand(partition=partitions)   # fan-out
    aggregate(results)                               # fan-in
```

**How the scheduler handles it:**
1. `get_partitions` runs first, pushes list to XCom.
2. Scheduler reads XCom value, creates N mapped `TaskInstance`s: `process[0]`, `process[1]`, `process[2]`.
3. Each mapped TI is scheduled independently.
4. `aggregate` waits for all mapped TIs to complete (implicit fan-in).

Mapped TIs appear in the UI as `process[0]`, `process[1]`, etc.

---

## Catchup and Backfill

### catchup=True (default)

```python
dag = DAG(
    dag_id="daily_etl",
    start_date=datetime(2023, 1, 1),
    schedule_interval="@daily",
    catchup=True,    # default
    max_active_runs=3,
)
```

When the DAG is unpaused today (2024-06-01), the scheduler creates DagRuns for every missed interval back to `start_date`. With `max_active_runs=3`, at most 3 run concurrently.

**Danger**: A newly deployed DAG with `catchup=True` and a 2-year `start_date` will immediately try to create 730 DagRuns.

**Best practice**: Use `catchup=False` unless you specifically need historical backfill.

### CLI Backfill

```bash
# Backfill specific date range (ignores current schedule state)
airflow dags backfill \
    --start-date 2024-01-01 \
    --end-date 2024-01-31 \
    my_dag_id

# Dry run
airflow dags backfill --dry-run --start-date 2024-01-01 my_dag_id
```

Backfill runs bypass `max_active_runs` (it creates a `backfill` run_type). Use `--max-active-runs` flag to limit.

---

## Scheduler HA (High Availability)

### Airflow 2.0+: Active-Active

Multiple scheduler processes can run simultaneously:
```
Scheduler-1  ─┐
Scheduler-2  ─┤─ Same metadata DB
Scheduler-3  ─┘
```

**How it works (no Zookeeper):**
- Uses DB row-level locking (SELECT FOR UPDATE SKIP LOCKED) on `dag_run` and `task_instance` rows.
- Each scheduler picks up different DagRuns to process.
- No explicit leader election — they all operate and let DB locks prevent double-scheduling.
- If one scheduler dies, the others immediately pick up its work (next heartbeat cycle).

**What can go wrong**: If DB is slow, schedulers back up. Increase `scheduler_heartbeat_sec` under load.

---

## Performance Tuning

```ini
[scheduler]
scheduler_heartbeat_sec = 5          # How often scheduler loop runs
max_dagruns_to_create_per_loop = 10  # DagRun creation throttle
max_tis_per_query = 512              # TIs fetched per scheduler loop
parallelism = 32                     # Max concurrent TIs across ALL DAGs

[core]
dag_concurrency = 16                 # Max concurrent TIs per DAG
max_active_runs_per_dag = 16         # Max concurrent DagRuns per DAG

[celery]
worker_concurrency = 16              # Tasks per Celery worker process
```

**Tuning strategy:**
1. Watch `scheduler.scheduler_loop_duration` metric — should stay < heartbeat_sec.
2. If scheduler is slow: reduce `max_dagruns_to_create_per_loop`, check DB query latency.
3. If workers are bottleneck: add workers or increase `worker_concurrency`.
4. If DB is bottleneck: PgBouncer, read replica for UI, upgrade hardware.

---

## Sensor vs Deferrable Sensor

| | Classic Sensor | Deferrable Sensor |
|---|---|---|
| Worker slot held? | Yes — entire wait | No — released immediately |
| Polling | Thread sleeps (`poke_interval`) | Asyncio coroutine |
| Scalability | 1 slot per waiting sensor | 1000s per Triggerer process |
| Added complexity | None | Requires Triggerer component |
| Use when | Short waits, simple setup | Long waits, many concurrent sensors |

---

## Concrete Example: Data Pipeline with Dynamic Task Mapping

```
Source: 50 S3 partitions/day → transform each → load → aggregate
```

```python
from airflow.decorators import dag, task
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from datetime import datetime

@dag(
    dag_id="s3_partitioned_pipeline",
    schedule_interval="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
    max_active_runs=2,
    default_args={"retries": 2, "retry_delay": timedelta(minutes=5)},
)
def s3_partitioned_pipeline():

    @task
    def discover_partitions(logical_date=None) -> list[str]:
        hook = S3Hook(aws_conn_id="aws_default")
        keys = hook.list_keys(
            bucket_name="data-lake",
            prefix=f"raw/{logical_date.strftime('%Y/%m/%d')}/"
        )
        return keys  # ["raw/2024/01/01/part-0001.parquet", ...]

    @task(
        executor_config={
            "KubernetesExecutor": {
                "request_memory": "4Gi",
                "request_cpu": "2",
            }
        }
    )
    def transform(s3_key: str) -> dict:
        # Heavy transform — runs in isolated K8s pod
        result = run_spark_transform(s3_key)
        return {"key": s3_key, "rows": result.count()}

    @task
    def aggregate(results: list[dict]) -> None:
        total = sum(r["rows"] for r in results)
        notify_slack(f"Pipeline complete: {total} total rows")

    # Execution trace:
    # 1. discover_partitions() runs → returns list of 50 keys
    # 2. Scheduler reads XCom → creates 50 mapped TaskInstances
    # 3. transform[0]..transform[49] run (K8s pods, up to parallelism limit)
    # 4. aggregate() waits for all 50, then runs fan-in

    keys = discover_partitions()
    results = transform.expand(s3_key=keys)
    aggregate(results)

dag_instance = s3_partitioned_pipeline()
```

**Execution trace:**
```
Parse: scheduler reads DAG from serialized_dag table
DagRun created: logical_date=2024-01-01, state=running

TI: discover_partitions → scheduled → queued → running (worker picks up)
  → pushes ["raw/.../part-0001.parquet", ..., "raw/.../part-0050.parquet"] to XCom
  → success

Scheduler reads XCom: creates 50 mapped TIs
TI: transform[0]..transform[49] → scheduled → queued
  → K8s executor creates 50 pods (subject to K8s node capacity + parallelism=32)
  → pods run in batches → success

TI: aggregate → waits (upstream_mapped_count=50, done=50) → scheduled → queued → success

DagRun: state=success
```

---

## DAG Versioning: Airflow 2.x vs 3.x

| Feature | 2.x | 3.x (shipped, GA since April 2025) |
|---|---|---|
| Serialization format | JSON (zlib-compressed) | Extended JSON with versioning (DAG Bundles, AIP-66) |
| DAG versioning | No true versioning — last parse wins | Explicit version history — each DagRun pins to the DAG version active at trigger time |
| Task-level versioning | No | Version pinned to run |
| Scheduler reads | From `serialized_dag` | From versioned DAG bundle store |

In 2.x, re-parsing a DAG file with structural changes immediately affects new runs. In-flight runs use the serialized snapshot from when they were created — but the snapshot in DB is overwritten on re-parse. This is a subtle bug risk for long-running DAGs. **This is one of the problems Airflow 3.0's DAG versioning (see below) directly fixes** — it was the single most-requested feature in the Airflow community survey.

---

## Airflow 3.x: What Changed (GA April 2025, current as of July 2026)

Airflow 3.0 is a major architectural release, not an incremental one. It keeps the core concepts described above (DagRun/TaskInstance state machine, XCom semantics, deferrable operators/Triggerer, scheduler HA via row locking) but changes several load-bearing internals. Airflow 2.x is still widely run in production and the content above remains valid for those deployments — treat this section as a diff, not a replacement.

### Task Execution API / Task SDK (AIP-72)

The biggest change: **workers no longer talk to the metadata DB directly.** In 2.x, workers import Airflow core, open a DB session, and read/write `task_instance`, `xcom`, etc. directly. In 3.x:

```
2.x:  Worker → (direct SQLAlchemy session) → Metadata DB
3.x:  Worker (Task SDK runtime) → Task Execution API (HTTP) → API Server → Metadata DB
```

- A new **API Server** component mediates all task-runtime interactions: state transitions, heartbeats, XCom push/pull, Variable/Connection lookups.
- Task code can no longer import Airflow DB models/sessions directly — enforces least-privilege and isolates user code from the scheduler's DB.
- The **Task SDK** is a lightweight runtime for executing tasks outside the main Airflow codebase — first shipped for Python (backward compatible with existing `PythonOperator`/`@task` code), with **Task SDKs for other languages (starting with Go) planned** for subsequent 3.x releases — this is the foundation for genuinely multi-language DAG task execution, not just Python.
- Net effect: **dag_folder / DAG file parsing is no longer required on workers at all** (2.x already avoided this in practice via SerializedDAG for Celery/K8s executors, but 3.x formalizes it — workers only need the Task SDK and API Server connectivity, not filesystem or DB access).

### DAG Processor Is Now a Standalone Component

In 2.x, the `DagFileProcessorManager` lives inside the Scheduler process. In 3.x, **DAG parsing is split out into its own `airflow dag-processor` process**, started separately. This isolates untrusted user code (top-level DAG file code still runs on every parse — that gotcha is unchanged) from the Scheduler process itself, improving both security and scheduler stability.

### DAG Versioning and DAG Bundles (AIP-66)

- DAGs are now versioned: a DagRun executes to completion using the DAG version that was active when it started, even if the DAG file changes mid-run. This directly fixes the "subtle bug risk for long-running DAGs" called out above.
- DAG source is packaged into **DAG Bundles** (e.g., a git ref, a versioned artifact) rather than a flat `dag_folder` scan, enabling reproducible deploys and rollback.

### Scheduler-Managed Backfills

Backfills move from an external CLI-driven process into the Scheduler itself, with progress visible in the UI (`airflow dags backfill` still works but is now scheduler-native, improving control and diagnostics).

### New UI, Assets, and Event-Driven Scheduling

- Airflow 3.0 ships a rebuilt UI (React + FastAPI) replacing the Flask/Flask-AppBuilder UI.
- **Datasets are renamed to Assets** — same concept (data-aware scheduling), unified with modern data-catalog terminology. Existing `Dataset`-based DAGs continue to work under the `Asset` naming.
- Assets gain **Watchers**, letting Airflow react to external events (e.g., AWS SQS) rather than only polling — a step toward event-driven scheduling, not just interval/cron-based DagRuns.

### What This Means for the Rest of This Document

- The **Scheduler Loop**, **Metadata DB schema**, **XCom mechanics**, **Executor types** (Local/Celery/Kubernetes/CeleryKubernetes), **Triggerer**, **dynamic task mapping**, and **catchup/backfill semantics** sections above remain conceptually correct in 3.x — the state machine and executor model didn't change.
- What changed is the **transport**: task-to-control-plane communication goes through the Task Execution API instead of direct DB/file access, and DAG parsing is an independently-scaled process instead of a Scheduler subroutine.
- If you're running Airflow 3.x, don't rely on `pod_template_file` volume-mounting `dag_folder` for KubernetesExecutor "for DB fallback" reasons — it's unnecessary; the Task SDK doesn't need it.

---

## Key Gotchas

1. **Top-level code in DAG files runs on every parse** — never put API calls, DB connections, or heavy compute at module level.

2. **XCom is not for large data** — it stores in the metadata DB. Use S3/GCS XCom backend for anything > a few KB.

3. **`catchup=True` with old `start_date` = DagRun explosion** — always set `catchup=False` unless you explicitly need backfill.

4. **`execution_date` is now `logical_date`** in 2.2+ — old template `{{ execution_date }}` still works but is deprecated.

5. **KubernetesExecutor cold start** — 15-30s pod startup. Bad for high-frequency short tasks. Use CeleryExecutor for those.

6. **Scheduler HA requires DB row locking** — PostgreSQL handles this well; MySQL has issues under high concurrency. Use Postgres in production.

7. **Dynamic task mapping limit** — `max_map_length` config (default 1024). Large fan-outs hit this.

8. **`max_active_runs` is not `max_active_tasks`** — the former limits concurrent DagRuns; the latter (`dag_concurrency`) limits concurrent TIs within a single DAG run. Both needed.

9. **Deferrable operators require a Triggerer process** — if Triggerer crashes, deferred tasks are stuck until it recovers.

10. **Parsing interval stacking** — if a DAG file takes 30s to parse and `min_file_process_interval=30`, you're constantly re-parsing. Complex DAG files need optimization.
