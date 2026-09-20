# KEDA: Kubernetes Event-Driven Autoscaling

## 30-Second Intuition

KEDA extends Kubernetes HPA to scale workloads based on external event sources — queue depth, Kafka consumer lag, Prometheus metrics, cron schedules. The key superpower: **scale to zero** when idle, then wake up on demand. HPA can only scale between 1 and N replicas based on CPU/memory; KEDA can scale 0↔N based on anything with an API.

---

## Why HPA Isn't Enough

```
Standard HPA:
  - Metrics: CPU utilization, memory, custom metrics (limited)
  - Min replicas: 1 (can't scale to zero)
  - Scale trigger: pod-level resource metrics
  - Latency: slow scale-up (CPU builds before HPA reacts)

KEDA:
  - Metrics: queue depth, consumer lag, Prometheus query, cron, any custom scaler
  - Min replicas: 0 (true scale-to-zero)
  - Scale trigger: external event source state
  - Latency: fast scale-up (triggered by queue depth, not CPU)
```

**Classic problem KEDA solves:**
```
SQS queue has 10,000 messages at 2am.
HPA: can't see SQS → workers sit at min=1 → processing takes hours.
KEDA: sees queue depth=10,000 → scale to 100 workers → process in minutes.
```

---

## Architecture

```
                                        K8s API
                                           │
KEDA Operator     ────────────────────────>│  manages ScaledObject/ScaledJob CRDs
      │                                    │
      │  reads ScaledObject                │  creates/updates HPA
      │  ↓                                 │
      └─ Metrics Server (keda-adapter) ───>│  external.metrics.k8s.io API
              │                            │  (HPA queries this for scaler metrics)
              │
         ScalerFactory
              │
   ┌──────────┼──────────┐
   Kafka    Prometheus  SQS    (50+ scalers)
   scaler   scaler      scaler
```

**Two components:**
1. **KEDA Operator**: Watches ScaledObject/ScaledJob CRDs, creates/manages HPAs, handles scale-to-zero (KEDA directly manages 0→1 transition, HPA handles 1→N).
2. **KEDA Metrics Adapter**: Implements `external.metrics.k8s.io` API — HPA queries this to get current metric values (queue depth, lag, etc.).

**Scale-to-zero mechanism:**
- HPA minimum replicas = 1. KEDA handles 0↔1 separately.
- When queue is empty: KEDA sets Deployment replicas=0 directly (bypasses HPA).
- When first message arrives: KEDA sets replicas=1, hands off to HPA for 1→N.
- HPA uses KEDA's external metrics to scale 1→N based on load.

---

## ScaledObject

Targets a Deployment (or StatefulSet, Custom Resource):

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
  namespace: processing
spec:
  scaleTargetRef:
    apiVersion: apps/v1        # default
    kind: Deployment           # or StatefulSet
    name: kafka-consumer

  pollingInterval: 30          # check external metrics every 30s
  cooldownPeriod: 300          # wait 5min before scaling down
  minReplicaCount: 0           # scale to zero when idle
  maxReplicaCount: 50

  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka.default.svc:9092
        consumerGroup: my-consumer-group
        topic: orders
        lagThreshold: "100"    # target: 100 messages per replica
        offsetResetPolicy: latest

  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleUp:
          stabilizationWindowSeconds: 0    # scale up immediately
          policies:
            - type: Percent
              value: 100
              periodSeconds: 15
        scaleDown:
          stabilizationWindowSeconds: 300  # stabilize for 5min before scale-down
```

---

## ScaledJob

For batch workloads — scale Kubernetes Jobs (not Deployments):

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: s3-processor
spec:
  jobTargetRef:
    template:
      spec:
        containers:
          - name: processor
            image: file-processor:latest
            args: ["--single-message"]    # process one message and exit
        restartPolicy: Never
    backoffLimit: 3
    completions: 1
    parallelism: 1

  pollingInterval: 10
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5
  maxReplicaCount: 100       # max concurrent Jobs

  scalingStrategy:
    strategy: accurate        # accurate, default, custom
    # accurate: one job per pending message (if possible)

  triggers:
    - type: sqs
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123/my-queue
        queueLength: "1"    # 1 job per message in queue
        awsRegion: us-east-1
      authenticationRef:
        name: aws-trigger-auth
```

**ScaledObject vs ScaledJob:**
- ScaledObject: long-running consumer that processes messages in a loop (Deployment/StatefulSet).
- ScaledJob: short-lived worker that processes one batch and exits (Job).
- ScaledJob avoids "stuck consumer" issues — each Job is fresh.

---

## Scale-to-Zero: How It Works

```
State: queue empty, replicas=0

1. KEDA polls scaler every pollingInterval seconds
2. Scaler returns metric=0 (queue empty) → stay at 0

State: message arrives, queue depth=5

1. KEDA scaler returns metric=5
2. KEDA operator: 0 replicas → set Deployment.spec.replicas=1
3. Pod starts → consumer begins processing
4. KEDA creates HPA targeting the ScaledObject's external metric
5. HPA computes desired replicas: ceil(5 / lagThreshold=1) = 5
6. HPA scales Deployment from 1 → 5

State: queue drains to 0

1. KEDA scaler returns metric=0
2. HPA would scale to 0, but HPA min=1 (KEDA overrides this)
3. KEDA operator waits cooldownPeriod=300s
4. After cooldown: KEDA directly sets Deployment.spec.replicas=0
5. HPA suspended by KEDA (it patches HPA.spec.minReplicas=0 temporarily)
```

**The 0→1 transition is KEDA's responsibility, not HPA's.** HPA only handles 1→N.

---

## Scalers Catalog

### Kafka Scaler

```yaml
triggers:
  - type: kafka
    metadata:
      bootstrapServers: "kafka-0.kafka:9092,kafka-1.kafka:9092"
      consumerGroup: my-group
      topic: events
      lagThreshold: "50"       # desired lag per replica
      activationLagThreshold: "1"   # activate (0→1) when lag >= 1
      offsetResetPolicy: latest     # or earliest
      allowIdleConsumers: "false"   # don't scale if no active consumers
    authenticationRef:
      name: kafka-trigger-auth      # TLS credentials
```

**Kafka scaler logic:**
```
total_lag = sum of (latest_offset - committed_offset) across all partitions

desired_replicas = ceil(total_lag / lagThreshold)

Example:
  10 partitions × 200 messages each = 2000 total lag
  lagThreshold = 100
  desired_replicas = ceil(2000/100) = 20
```

**Important**: KEDA queries the Kafka consumer group's committed offsets directly via the Kafka protocol — no Kafka admin API needed.

### SQS Scaler

```yaml
triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123456/my-queue
      queueLength: "10"          # target: 10 messages per replica
      awsRegion: us-east-1
      activationTargetQueueLength: "1"  # activate when ≥1 message
      scaleOnInFlight: "true"           # include in-flight messages in count
      scaleOnDelayed: "false"           # exclude delayed messages
    authenticationRef:
      name: aws-trigger-auth
```

SQS scaler calls `GetQueueAttributes` for `ApproximateNumberOfMessages` (and optionally `ApproximateNumberOfMessagesNotVisible` for in-flight).

### Prometheus Scaler

```yaml
triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring.svc:9090
      metricName: http_requests_pending
      threshold: "100"           # scale 1 replica per 100 pending requests
      activationThreshold: "10"  # activate when ≥10
      query: |
        sum(rate(http_requests_pending_total[1m]))
      namespace: my-service      # label filter (optional)
```

Prometheus scaler evaluates the PromQL query and divides the result by `threshold` to get desired replicas.

### Cron Scaler

```yaml
triggers:
  - type: cron
    metadata:
      timezone: "America/New_York"
      start: "30 7 * * 1-5"    # 7:30am weekdays
      end: "0 20 * * 1-5"      # 8pm weekdays
      desiredReplicas: "10"    # scale to 10 during hours
```

Useful for predictable load patterns — scale up before business hours, down after.

### Redis Scaler

```yaml
triggers:
  - type: redis
    metadata:
      address: redis-master.default.svc:6379
      listName: job-queue
      listLength: "10"       # target items per replica
    authenticationRef:
      name: redis-trigger-auth
```

### Azure Service Bus

```yaml
triggers:
  - type: azure-servicebus
    metadata:
      namespace: my-servicebus-ns
      queueName: my-queue          # or topicName + subscriptionName
      messageCount: "5"
      activationMessageCount: "1"
    authenticationRef:
      name: azure-servicebus-auth
```

---

## KEDA + Argo: Trigger Argo Workflow Pods via KEDA

Architecture: KEDA watches a queue → scales Argo Workflow runner pods:

```yaml
# Pattern 1: KEDA scales a Deployment that picks up messages and submits Argo Workflows
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: workflow-submitter-scaler
spec:
  scaleTargetRef:
    name: workflow-submitter   # Deployment that reads from queue and submits workflows
  minReplicaCount: 0
  maxReplicaCount: 10
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123/workflow-triggers
        queueLength: "1"

---
# Pattern 2: KEDA + Argo Events — use Argo Events as the bridge
# Argo Events EventSource watches SQS
# Argo Events Sensor triggers Argo Workflow
# KEDA scales the Argo Events Sensor deployment based on queue depth
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
spec:
  scaleTargetRef:
    name: argo-events-sensor-deployment
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123/events
        queueLength: "5"
```

---

## TriggerAuthentication

How credentials are injected into scalers:

### Kubernetes Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kafka-secret
type: Opaque
data:
  sasl_username: <base64>
  sasl_password: <base64>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: kafka-trigger-auth
spec:
  secretTargetRef:
    - parameter: username      # parameter name expected by scaler
      name: kafka-secret       # secret name
      key: sasl_username       # secret key
    - parameter: password
      name: kafka-secret
      key: sasl_password
```

### Pod Identity (IRSA for AWS)

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: aws-trigger-auth
spec:
  podIdentity:
    provider: aws    # uses IRSA — pod's IAM role for SQS/CloudWatch access
```

The KEDA metrics adapter pod must have the IAM role (via service account annotation). No secret needed.

### HashiCorp Vault

```yaml
spec:
  hashiCorpVault:
    address: http://vault.default.svc:8200
    authentication: token
    role: keda-role
    credential:
      token:
        secretRef:
          name: vault-token
          key: token
    secrets:
      - parameter: password
        path: secret/data/kafka
        key: password
```

### ClusterTriggerAuthentication

Cluster-scoped version — usable across all namespaces:
```yaml
apiVersion: keda.sh/v1alpha1
kind: ClusterTriggerAuthentication
metadata:
  name: aws-cluster-auth
spec:
  podIdentity:
    provider: aws
```

Reference from ScaledObject:
```yaml
authenticationRef:
  name: aws-cluster-auth
  kind: ClusterTriggerAuthentication
```

---

## Concrete Example: Kafka Consumer Scaling

Full end-to-end setup for a Kafka consumer Deployment:

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-processor
  namespace: ecommerce
spec:
  replicas: 0           # KEDA manages this
  selector:
    matchLabels:
      app: order-processor
  template:
    metadata:
      labels:
        app: order-processor
    spec:
      serviceAccountName: order-processor-sa
      containers:
        - name: processor
          image: order-processor:v3
          env:
            - name: KAFKA_BROKERS
              value: "kafka.kafka.svc:9092"
            - name: CONSUMER_GROUP
              value: "order-processors"
            - name: TOPIC
              value: "orders"
          resources:
            requests:
              memory: 512Mi
              cpu: 250m
            limits:
              memory: 1Gi
              cpu: 500m
---
# trigger-auth.yaml
apiVersion: v1
kind: Secret
metadata:
  name: kafka-tls-secret
  namespace: ecommerce
type: Opaque
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
  ca.crt: <base64-encoded-ca>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: kafka-auth
  namespace: ecommerce
spec:
  secretTargetRef:
    - parameter: tls
      name: kafka-tls-secret
      key: tls.crt
    - parameter: tlsKey
      name: kafka-tls-secret
      key: tls.key
    - parameter: ca
      name: kafka-tls-secret
      key: ca.crt
---
# scaled-object.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
  namespace: ecommerce
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-processor

  pollingInterval: 15        # check Kafka every 15 seconds
  cooldownPeriod: 120        # 2 min before scaling down
  minReplicaCount: 0         # scale to zero when no messages
  maxReplicaCount: 30        # cap at 30 consumers

  triggers:
    - type: kafka
      metadata:
        bootstrapServers: "kafka-0.kafka.svc:9092,kafka-1.kafka.svc:9092"
        consumerGroup: order-processors
        topic: orders
        lagThreshold: "100"             # 1 replica per 100 messages of lag
        activationLagThreshold: "5"     # start from 0 when lag ≥ 5
        offsetResetPolicy: latest
        allowIdleConsumers: "false"
        scaleToZeroOnInvalidOffset: "true"
        partitionLimitation: ""          # scale across all partitions
        tls: "enable"
      authenticationRef:
        name: kafka-auth

  advanced:
    horizontalPodAutoscalerConfig:
      name: order-processor-hpa         # explicit HPA name for observability
      behavior:
        scaleUp:
          stabilizationWindowSeconds: 0
          policies:
            - type: Pods
              value: 5                   # add up to 5 pods per 15s
              periodSeconds: 15
            - type: Percent
              value: 50                  # or 50% of current, whichever is larger
              periodSeconds: 15
          selectPolicy: Max
        scaleDown:
          stabilizationWindowSeconds: 120
          policies:
            - type: Pods
              value: 2                   # remove at most 2 pods per 30s
              periodSeconds: 30
```

**Scaling behavior trace:**
```
t=0:   Kafka topic orders has 0 messages
       KEDA: activationLagThreshold=5 not met → replicas=0

t=30s: 50 messages arrive
       KEDA polls: lag=50, activationLag met (50>5)
       KEDA: 0→1 (direct Deployment patch, bypasses HPA)
       Consumer pod starts, begins consuming

t=45s: KEDA metrics adapter reports lag=50 to HPA
       HPA: desired = ceil(50/100) = 1 (already at 1)

t=1m:  500 messages arrive (backlog = 550)
       HPA: desired = ceil(550/100) = 6
       scaleUp policy: max(5 pods, 50% of 1=1) = 5 pods → add 5
       Deployment: 1 → 6

t=1m15s: HPA: lag still 500+
         desired = ceil(500/100) = 5 (being consumed)
         6 consumers eating through queue

t=5m:  Queue drained to 0
       HPA: desired = ceil(0/100) = 0, but min=1 (KEDA sets this)
       stays at 1 consumer

t=7m:  Queue still empty (cooldownPeriod=120s elapsed)
       KEDA: directly patches Deployment.spec.replicas=0
       Last consumer pod terminates gracefully
       replicas=0 ✓
```

---

## Key Gotchas

1. **Scale-to-zero has cold start latency** — from KEDA detecting a message to your pod being ready: 15s (poll) + 30s (pod startup) = 45s minimum. Messages may time out or pile up. Mitigate with `activationLagThreshold` to keep 1 replica warm.

2. **pollingInterval vs HPA sync** — HPA syncs every 15s by default. KEDA's `pollingInterval` is separate. The effective scaling reaction time is max(pollingInterval, HPA sync interval).

3. **Kafka scaler needs consumer group to exist** — KEDA queries committed offsets. If the consumer group hasn't committed yet (brand new deployment), lag calculation may be inaccurate.

4. **`lagThreshold` is not a hard limit per partition** — it's a desired average across all partitions. With 10 partitions, `lagThreshold=100` means scale to ceil(total_lag/100) replicas, regardless of how lag is distributed.

5. **`allowIdleConsumers: false`** prevents scaling above partition count — you can't have more consumers than partitions in a consumer group (Kafka assigns one partition per consumer).

6. **ScaledJob creates N Jobs, not scales 1 Job** — ScaledJob spawns new Job objects per scaler trigger. Monitor job history limits or completed jobs accumulate.

7. **KEDA modifies the HPA** — if you also manage an HPA for the same Deployment, KEDA will overwrite it. Let KEDA own the HPA.

8. **Cooldown period prevents thrashing** — but means over-provisioning persists for `cooldownPeriod` seconds. Balance cost (shorter cooldown) vs stability (longer).

9. **TriggerAuthentication namespace** — TriggerAuthentication is namespace-scoped. Use ClusterTriggerAuthentication for cross-namespace secrets.

10. **Prometheus scaler requires accessible endpoint** — KEDA metrics adapter calls Prometheus directly. If Prometheus is in a different namespace/cluster, configure network policy accordingly.
