# Argo Workflows Internals

## 30-Second Intuition

Argo Workflows is a Kubernetes-native workflow engine: each step runs as a container in a Pod, and the workflow definition is a CRD stored in etcd. The controller watches Workflow objects and drives their execution by creating/monitoring Pods. Unlike Airflow, there's no separate scheduler process, no metadata DB — Kubernetes IS the state machine.

---

## CRD Architecture

```
Kubernetes API Server (etcd)
  ├─ Workflow              ← a running or completed workflow instance
  ├─ WorkflowTemplate      ← reusable template (namespace-scoped)
  ├─ ClusterWorkflowTemplate ← reusable template (cluster-scoped)
  ├─ CronWorkflow          ← cron-scheduled Workflow creator
  └─ WorkflowArtifactGCTask ← garbage collection tasks
```

**Workflow controller** watches these CRDs and reconciles:
```
Controller loop:
  1. List/Watch Workflow objects
  2. For each Workflow not in terminal state:
     a. Evaluate which templates need to run
     b. Create Pods for ready templates
     c. Watch Pod completions → update Workflow status
     d. Continue until all nodes complete or error
```

### CronWorkflow

```yaml
apiVersion: argoproj.io/v1alpha1
kind: CronWorkflow
metadata:
  name: daily-pipeline
spec:
  schedule: "0 2 * * *"
  timezone: "America/New_York"
  concurrencyPolicy: Forbid    # Allow, Forbid, Replace
  startingDeadlineSeconds: 0
  workflowSpec:
    entrypoint: main
    templates:
      - name: main
        container:
          image: my-job:latest
          command: [python, main.py]
```

---

## Template Types

### 1. Container Template

Runs a single container:
```yaml
- name: train-model
  container:
    image: ml-trainer:v2
    command: [python, train.py]
    args: ["--epochs", "100"]
    resources:
      requests:
        memory: 16Gi
        cpu: "8"
        nvidia.com/gpu: "1"
    env:
      - name: LEARNING_RATE
        value: "0.001"
```

### 2. Script Template

Inline script — Argo writes it to a file and executes it:
```yaml
- name: process-data
  script:
    image: python:3.11
    command: [python]
    source: |
      import json, sys
      data = json.loads(open('/tmp/input.json').read())
      result = {"count": len(data)}
      print(json.dumps(result))
```

### 3. DAG Template

Dependency graph — steps declare dependencies:
```yaml
- name: ml-pipeline
  dag:
    tasks:
      - name: preprocess
        template: preprocess-data
      - name: train
        template: train-model
        dependencies: [preprocess]
        arguments:
          artifacts:
            - name: dataset
              from: "{{tasks.preprocess.outputs.artifacts.dataset}}"
      - name: evaluate
        template: eval-model
        dependencies: [train]
      - name: register
        template: register-model
        dependencies: [evaluate]
        when: "{{tasks.evaluate.outputs.parameters.accuracy}} > 0.95"
```

### 4. Steps Template

Sequential steps with optional parallelism within a step:
```yaml
- name: pipeline
  steps:
    - - name: step1          # sequential step
        template: fetch-data
    - - name: step2a         # parallel within step
        template: process-a
      - name: step2b
        template: process-b
    - - name: step3          # sequential step
        template: aggregate
        arguments:
          parameters:
            - name: result_a
              value: "{{steps.step2a.outputs.parameters.result}}"
```

**DAG vs Steps:**
- DAG: declare dependencies explicitly, any topology, easier to read for complex graphs.
- Steps: think of it as a list of "phases"; steps within the same phase run in parallel. Simpler for linear pipelines.

### 5. Resource Template

Manipulate Kubernetes resources directly:
```yaml
- name: create-spark-job
  resource:
    action: create       # create, apply, delete, get, patch
    successCondition: status.applicationState.state == COMPLETED
    failureCondition: status.applicationState.state == FAILED
    manifest: |
      apiVersion: sparkoperator.k8s.io/v1beta2
      kind: SparkApplication
      metadata:
        name: my-spark-job
      spec:
        type: Python
        mainApplicationFile: s3://bucket/job.py
```

### 6. Suspend Template

Pause workflow for manual approval or timed wait:
```yaml
- name: await-approval
  suspend:
    duration: "24h"   # optional auto-resume after duration
```

Resume via: `argo resume my-workflow --node-field-selector templateName=await-approval`

### 7. HTTP Template

Make an HTTP call as a workflow step:
```yaml
- name: notify-slack
  http:
    url: "https://hooks.slack.com/services/XXX"
    method: POST
    headers:
      - name: Content-Type
        value: application/json
    body: '{"text": "Pipeline complete!"}'
    successCondition: "response.statusCode == 200"
```

---

## Artifacts

Artifacts are files passed between workflow steps, stored in an artifact repository (S3/GCS/MinIO/Artifactory).

### Configuration

```yaml
# values.yaml or ConfigMap
artifactRepository:
  s3:
    bucket: my-argo-artifacts
    endpoint: s3.amazonaws.com
    keyFormat: "{{workflow.namespace}}/{{workflow.name}}/{{pod.name}}/{{outputs.artifacts.name}}"
```

### Passing Artifacts Between Steps

```yaml
templates:
  - name: preprocess
    outputs:
      artifacts:
        - name: dataset          # name within workflow
          path: /tmp/dataset.parquet   # file path inside container
    container:
      image: preprocess:latest
      command: [python, preprocess.py, --output, /tmp/dataset.parquet]

  - name: train
    inputs:
      artifacts:
        - name: dataset
          path: /tmp/dataset.parquet   # where to mount inside this container
    container:
      image: trainer:latest
      command: [python, train.py, --data, /tmp/dataset.parquet]
```

**Mechanism:**
1. `preprocess` container writes to `/tmp/dataset.parquet`.
2. `wait` container (sidecar, see Pod Lifecycle) copies it to S3.
3. `train` step starts: init container (or wait container) downloads from S3 to `/tmp/dataset.parquet`.
4. `train` container runs with the file already in place.

---

## Parameters

### Input/Output Parameters (lightweight — stored in etcd via Workflow status)

```yaml
- name: compute-stats
  inputs:
    parameters:
      - name: date
  outputs:
    parameters:
      - name: row_count
        valueFrom:
          path: /tmp/count.txt   # container writes value to this file
  container:
    image: stats:latest
    command: [python, compute.py, "--date", "{{inputs.parameters.date}}"]
```

### Global Parameters

```yaml
spec:
  arguments:
    parameters:
      - name: env
        value: production
  templates:
    - name: main
      container:
        env:
          - name: ENV
            value: "{{workflow.parameters.env}}"
```

---

## Workflow Controller: Reconciliation Loop

```
Controller watches:
  - Workflow objects (Create/Update/Delete)
  - Pod objects with argo label selector

On Workflow change:
  1. Deserialize Workflow spec + status
  2. Build execution graph (DAG or Steps)
  3. Determine which nodes are "ready" (all deps complete)
  4. For each ready node:
     a. Create Pod with correct spec
     b. Inject artifact download init containers
     c. Record node as "Pending" in Workflow.status.nodes
  5. Patch Workflow object back to API server

On Pod status change:
  1. Find parent Workflow
  2. Map Pod phase → node status (Succeeded/Failed)
  3. Update Workflow.status.nodes
  4. Requeue Workflow for next reconciliation
```

**State is in etcd** — the Workflow object's `status.nodes` field is the ground truth. If the controller crashes and restarts, it re-reads all Workflows and resumes.

---

## Pod Lifecycle: The Sidecar Model

Every Argo step runs as a Pod with multiple containers:

```
Pod for a workflow step:
  ├─ init container: "init"     ← copies argo binary into shared volume
  ├─ container: "main"          ← your actual workload
  └─ container: "wait"          ← argo agent (emissary executor)
```

### Emissary Executor (default since Argo 3.1)

The `wait` container is the **Argo executor**. It:
1. Waits for `main` to complete (by watching a shared process namespace or file signals).
2. Collects output parameters (reads files at specified paths).
3. Collects output artifacts (uploads to S3).
4. Reports status back by patching the Workflow object.

**Why emissary replaced Docker executor:**
- Docker executor required mounting the Docker socket (huge security risk).
- PNS executor used ptrace (Linux-specific, fragile).
- Emissary uses a shared volume to inject the binary — no host access required.
- Works on any OCI-compliant runtime (containerd, CRI-O).

### The `/var/run/argo` shared volume

```
init container: cp /usr/bin/argoexec /var/run/argo/argoexec
main container: entrypoint replaced with: /var/run/argo/argoexec emissary -- <original_command>
wait container: /var/run/argo/argoexec wait
```

The `argoexec emissary` wrapper runs your command as a subprocess, then signals the `wait` container when done. This is how artifact/parameter collection happens without Docker socket access.

---

## Retry Strategies

```yaml
- name: flaky-step
  retryStrategy:
    limit: "3"
    retryPolicy: "OnFailure"    # Always, OnFailure, OnError, OnTransientError
    backoff:
      duration: "30s"           # initial wait
      factor: "2"               # exponential backoff multiplier
      maxDuration: "5m"
    expression: "lastRetry.exitCode == 1"   # retry only on exit code 1
  container:
    image: my-job:latest
```

Retry policies:
- `OnFailure`: retry if exit code != 0.
- `OnError`: retry on infrastructure errors (pod eviction, OOM kill).
- `OnTransientError`: retry on network errors, pod scheduling failures.
- `Always`: always retry regardless of exit code.

---

## Workflow of Workflows

Submit child workflows from within a workflow:

```yaml
- name: spawn-child
  resource:
    action: create
    manifest: |
      apiVersion: argoproj.io/v1alpha1
      kind: Workflow
      metadata:
        generateName: child-workflow-
      spec:
        workflowTemplateRef:
          name: my-workflow-template
        arguments:
          parameters:
            - name: input
              value: "{{inputs.parameters.input}}"
    successCondition: status.phase == Succeeded
    failureCondition: status.phase in (Failed, Error)
```

Or use the `submit` workflow step:
```yaml
- name: submit-child
  steps:
    - - name: run-child
        templateRef:
          name: my-template
          template: entrypoint
          clusterScope: false
```

---

## Parallelism Control

```yaml
spec:
  parallelism: 5          # max concurrent pods in this workflow

templates:
  - name: parallel-dag
    parallelism: 3        # max concurrent tasks in this DAG template
    dag:
      tasks:
        - name: task-{{item}}
          template: worker
          arguments:
            parameters: [{name: item, value: "{{item}}"}]
          withItems: [a, b, c, d, e, f, g, h]  # 8 tasks, max 3 concurrent
```

---

## Garbage Collection

### TTL Strategy

```yaml
spec:
  ttlStrategy:
    secondsAfterCompletion: 86400    # delete workflow 24h after done
    secondsAfterSuccess: 3600        # override for successful workflows
    secondsAfterFailure: 604800      # keep failed for 7 days (debugging)
```

### Pod GC

```yaml
spec:
  podGC:
    strategy: OnPodSuccess           # OnPodCompletion, OnPodSuccess, OnWorkflowCompletion, OnWorkflowSuccess
    labelSelector:
      matchLabels:
        app: my-app
```

Without GC, completed pods accumulate in the namespace. At scale (1000s of workflows/day) this kills API server performance.

---

## Argo Workflows vs Airflow

| Dimension | Argo Workflows | Airflow |
|---|---|---|
| Execution unit | Container/Pod | Python callable |
| State store | etcd (via K8s CRDs) | Postgres/MySQL |
| Language | YAML (workflow spec) | Python (DAG definition) |
| Dynamic tasks | `withItems`/`withParam` | `expand()` (2.3+) |
| Worker model | One pod per step | Shared worker pool |
| Scheduling | CronWorkflow (external trigger) | Built-in scheduler |
| Isolation | Strong (container per task) | Weak (shared worker process) |
| Dependency on K8s | Required | Optional |
| UI | Argo UI | Airflow UI |
| Ecosystem | Cloud-native, CNCF Graduated project (since Dec 2022) | Mature, extensive providers |

**Use Argo Workflows when:**
- Steps need strong isolation or different images.
- You're already on Kubernetes and want native integration.
- You need artifact passing between steps.
- ML pipelines with GPU tasks.

**Use Airflow when:**
- You have complex scheduling logic (catchup, backfill, SLAs).
- Most tasks are Python-based and share dependencies.
- You need rich sensors (waiting on external events).
- Your team is Python-first.

---

## Concrete Example: ML Training Pipeline

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: ml-training-
spec:
  entrypoint: ml-pipeline
  arguments:
    parameters:
      - name: dataset-path
        value: "s3://data/training/2024-01-01/"
      - name: model-name
        value: "my-classifier-v2"

  artifactRepositoryRef:
    configMap: artifact-repositories
    key: default

  templates:
    # ─────────── Orchestrator ───────────
    - name: ml-pipeline
      dag:
        tasks:
          - name: prepare-data
            template: data-prep
            arguments:
              parameters:
                - name: source
                  value: "{{workflow.parameters.dataset-path}}"

          - name: train
            template: model-training
            dependencies: [prepare-data]
            arguments:
              artifacts:
                - name: dataset
                  from: "{{tasks.prepare-data.outputs.artifacts.processed-dataset}}"

          - name: evaluate
            template: model-eval
            dependencies: [train]
            arguments:
              artifacts:
                - name: model
                  from: "{{tasks.train.outputs.artifacts.model}}"

          - name: register
            template: model-registry
            dependencies: [evaluate]
            when: "{{tasks.evaluate.outputs.parameters.f1-score}} > 0.90"
            arguments:
              parameters:
                - name: model-name
                  value: "{{workflow.parameters.model-name}}"
                - name: score
                  value: "{{tasks.evaluate.outputs.parameters.f1-score}}"
              artifacts:
                - name: model
                  from: "{{tasks.train.outputs.artifacts.model}}"

    # ─────────── Step Templates ───────────
    - name: data-prep
      inputs:
        parameters:
          - name: source
      outputs:
        artifacts:
          - name: processed-dataset
            path: /tmp/dataset.parquet
      container:
        image: data-pipeline:v3
        command: [python, prep.py]
        args: ["--source", "{{inputs.parameters.source}}", "--output", "/tmp/dataset.parquet"]
        resources:
          requests: {memory: 8Gi, cpu: "4"}

    - name: model-training
      inputs:
        artifacts:
          - name: dataset
            path: /tmp/dataset.parquet
      outputs:
        artifacts:
          - name: model
            path: /tmp/model.pkl
      container:
        image: ml-trainer:v5
        command: [python, train.py]
        args: ["--data", "/tmp/dataset.parquet", "--output", "/tmp/model.pkl"]
        resources:
          requests: {memory: 16Gi, cpu: "8", "nvidia.com/gpu": "1"}
          limits: {"nvidia.com/gpu": "1"}
      retryStrategy:
        limit: "2"
        retryPolicy: OnError

    - name: model-eval
      inputs:
        artifacts:
          - name: model
            path: /tmp/model.pkl
      outputs:
        parameters:
          - name: f1-score
            valueFrom:
              path: /tmp/score.txt
        artifacts:
          - name: eval-report
            path: /tmp/eval_report.html
      container:
        image: ml-evaluator:v2
        command: [python, evaluate.py]
        args: ["--model", "/tmp/model.pkl", "--score-output", "/tmp/score.txt"]
        resources:
          requests: {memory: 4Gi, cpu: "2"}

    - name: model-registry
      inputs:
        parameters:
          - name: model-name
          - name: score
        artifacts:
          - name: model
            path: /tmp/model.pkl
      container:
        image: model-registry-client:v1
        command: [python, register.py]
        args:
          - "--model", "/tmp/model.pkl"
          - "--name", "{{inputs.parameters.model-name}}"
          - "--score", "{{inputs.parameters.score}}"
        env:
          - name: REGISTRY_URL
            valueFrom:
              secretKeyRef:
                name: registry-credentials
                key: url
```

**Execution trace:**
```
1. Workflow submitted → controller sees it → entrypoint=ml-pipeline
2. DAG template: prepare-data has no dependencies → Pod created immediately
   - init container injects argoexec
   - main: data-pipeline:v3 runs prep.py
   - wait: argoexec uploads /tmp/dataset.parquet to S3 → records artifact ref in Workflow status

3. prepare-data Succeeded → train dependencies satisfied → Pod created
   - init/wait: argoexec downloads dataset.parquet from S3 to /tmp/
   - main: trainer:v5 runs train.py with GPU
   - wait: uploads model.pkl to S3

4. train Succeeded → evaluate Pod created → runs eval → writes score to /tmp/score.txt
   - wait: reads /tmp/score.txt → stores as parameter in Workflow.status.nodes[evaluate].outputs.parameters

5. evaluate Succeeded, f1-score=0.93 → 0.93 > 0.90 → register Pod created
   - Downloads model.pkl → registers with ML platform

6. All nodes Succeeded → Workflow.phase = Succeeded
```

---

## Key Gotchas

1. **etcd size limit** — Workflow objects (including `status.nodes`) must stay under ~1.5MB. Deep workflows with many nodes can hit this. Use workflow archiving to offload to Postgres.

2. **Artifact download is blocking** — if S3 is slow, your pod just sits in init stage. Build in retries on artifact config.

3. **`when` conditions are string comparisons** — `"{{tasks.eval.outputs.parameters.score}} > 0.9"` is evaluated as a string expression. Use proper numeric formatting.

4. **Pod GC must be configured** — without it, namespaces accumulate thousands of completed pods, killing K8s API server.

5. **emissary requires shared process namespace** or a specific pod security configuration — check your PSP/PSA policies.

6. **`withItems` loops vs dynamic fan-out** — `withItems` is static (items in spec); `withParam` is dynamic (items from previous step output as JSON array). Use `withParam` for runtime fan-out.

7. **Workflow controller is a single point of failure** — run multiple replicas with leader election (HA mode available since 3.4).

8. **`globalParameters` vs per-step parameters** — global params are accessible everywhere via `{{workflow.parameters.*}}`; step outputs require explicit threading through DAG task arguments.

9. **TTL is required at scale** — set it or your cluster fills with completed Workflow objects that slow down etcd.
