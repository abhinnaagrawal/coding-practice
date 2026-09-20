# Argo Events: Event-Driven Automation for Kubernetes

## 30-Second Intuition

Argo Events is a plumbing layer: it listens to external event sources (GitHub webhooks, S3 notifications, Kafka topics, cron schedules), buffers them on an EventBus (NATS), and triggers Kubernetes actions (Argo Workflows, HTTP calls, resource creation). Think of it as AWS EventBridge, but Kubernetes-native and composable with Argo Workflows.

---

## Architecture Overview

```
External World          Argo Events                  Kubernetes
────────────            ──────────                   ──────────
GitHub webhook   →  EventSource  →  EventBus  →  Sensor  →  Argo Workflow
S3 notification →  EventSource  →  (NATS)    →  Sensor  →  HTTP call
Kafka topic      →  EventSource  →            →  Sensor  →  K8s resource
SQS queue        →  EventSource  →            →          →  Slack notification
Calendar cron    →  EventSource  →
```

**Three primitives:**
1. **EventSource**: Connects to an external system, emits cloud events.
2. **EventBus**: Durable message bus (NATS) — decouples sources from sensors.
3. **Sensor**: Watches EventBus, applies conditions/filters, fires triggers.

---

## Core CRDs

### EventSource

Connects to external systems and publishes events to EventBus:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: my-sources
spec:
  eventBusName: default
  github:
    my-repo-push:
      repositories:
        - owner: my-org
          names: [my-repo]
      webhook:
        endpoint: /push
        port: "12000"
        method: POST
      events:
        - push
      apiToken:
        name: github-secret
        key: token
      insecure: false

  s3:
    data-uploads:
      bucket:
        name: my-data-bucket
      region: us-east-1
      filter:
        prefix: "uploads/"
        suffix: ".parquet"
      events:
        - s3:ObjectCreated:*
      credentials:
        accessKey:
          name: aws-secret
          key: accessKey
        secretKey:
          name: aws-secret
          key: secretKey
```

Each EventSource runs as a Pod that subscribes to the source and publishes CloudEvents to the EventBus.

---

## EventSource Types

### Webhook

Receives HTTP POST calls:
```yaml
webhook:
  my-hook:
    port: "12000"
    endpoint: /webhook
    method: POST
    # optionally: authSecret, serverCertSecret, serverKeySecret for mTLS
```

Argo Events creates a Service + Pod. You point your system at `http://event-source-svc:12000/webhook`.

### GitHub

```yaml
github:
  push-events:
    repositories:
      - owner: my-org
        names: [repo1, repo2]
    webhook:
      endpoint: /push
      port: "12000"
    events: [push, pull_request]
    apiToken:
      name: github-access-token
      key: token
    # Argo auto-registers webhook in GitHub if apiToken has admin:repo_hook scope
```

### S3

Uses S3 event notifications (SNS+SQS under the hood or direct SQS):
```yaml
s3:
  data-bucket-events:
    bucket:
      name: my-bucket
    region: us-east-1
    events:
      - s3:ObjectCreated:*
      - s3:ObjectRemoved:*
    filter:
      prefix: "ingest/"
      suffix: ".json"
```

### Kafka

```yaml
kafka:
  orders-topic:
    url: "kafka-broker:9092"
    topic: orders
    partition: "0"      # or omit for all partitions
    consumerGroup: argo-events-group
    tls:
      clientCertSecret: {name: kafka-tls, key: tls.crt}
      clientKeySecret: {name: kafka-tls, key: tls.key}
      caCertSecret: {name: kafka-tls, key: ca.crt}
```

### SQS

```yaml
sqs:
  job-queue:
    region: us-east-1
    queue: my-job-queue
    waitTimeSeconds: 20    # long polling
    credentials:
      accessKey: {name: aws-creds, key: accessKey}
      secretKey: {name: aws-creds, key: secretKey}
```

### NATS

Subscribes to NATS subjects:
```yaml
nats:
  nats-events:
    url: nats://nats:4222
    subject: my.events.#
```

### Calendar

Cron-based event generation:
```yaml
calendar:
  daily-trigger:
    schedule: "0 2 * * *"     # cron expression
    timezone: "America/New_York"
  interval-trigger:
    interval: 5m               # every 5 minutes
```

### Resource

Watches Kubernetes resource state changes:
```yaml
resource:
  pod-completed:
    namespace: my-namespace
    group: ""
    version: v1
    resource: pods
    eventTypes:
      - ADD
      - UPDATE
    filter:
      fields:
        - key: status.phase
          operation: "=="
          value: Succeeded
```

---

## EventBus

The EventBus is the backbone — a durable message bus that decouples EventSources from Sensors.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventBus
metadata:
  name: default
spec:
  nats:
    native:
      replicas: 3          # HA: 3-node NATS cluster
      auth: token
      # or: jetstream for persistence
```

**NATS Streaming (default):**
- EventSource pods publish CloudEvents to NATS subjects.
- Sensor pods subscribe to relevant subjects.
- Events are durably buffered: if Sensor is down, events queue up in NATS.

**NATS JetStream (recommended for production):**
```yaml
spec:
  jetstream:
    version: "2.10.5"
    replicas: 3
    persistence:
      storageClassName: gp3
      volumeSize: 20Gi
```

JetStream adds durable consumers, replay, and ack-based processing — ensures at-least-once delivery even across Sensor restarts.

---

## Sensor

The Sensor watches EventBus for events and fires triggers when conditions are met.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: my-sensor
spec:
  template:
    serviceAccountName: argo-events-sa
  dependencies:
    - name: s3-event
      eventSourceName: my-sources
      eventName: data-uploads   # matches EventSource key

    - name: calendar-event
      eventSourceName: my-sources
      eventName: daily-trigger

  triggers:
    - template:
        name: trigger-workflow
        argoWorkflow:
          operation: submit
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: process-upload-
              spec:
                workflowTemplateRef:
                  name: data-pipeline
                arguments:
                  parameters:
                    - name: s3-key
                      value: "replace-me"   # will be overridden by event data
          parameters:
            - src:
                dependencyName: s3-event
                dataKey: notification.s3.object.key   # extract from CloudEvent payload
              dest: spec.arguments.parameters.0.value  # inject into workflow arg
```

---

## Trigger Types

### Argo Workflow Trigger

```yaml
triggers:
  - template:
      name: submit-workflow
      argoWorkflow:
        operation: submit       # submit, retry, resume, suspend, terminate
        source:
          resource:
            apiVersion: argoproj.io/v1alpha1
            kind: Workflow
            metadata:
              generateName: my-workflow-
            spec:
              workflowTemplateRef:
                name: my-template
```

### HTTP Trigger

```yaml
triggers:
  - template:
      name: http-trigger
      http:
        url: http://my-service/api/notify
        method: POST
        headers:
          Content-Type: application/json
        payload:
          - src:
              dependencyName: s3-event
              dataKey: notification.s3.object.key
            dest: body.s3_key
        timeout: 10s
```

### Kubernetes Resource Trigger

Create/update any K8s resource:
```yaml
triggers:
  - template:
      name: create-job
      k8s:
        operation: create
        source:
          resource:
            apiVersion: batch/v1
            kind: Job
            metadata:
              generateName: data-job-
            spec:
              template:
                spec:
                  containers:
                    - name: worker
                      image: my-worker:latest
                  restartPolicy: Never
```

### Kafka Trigger

Publish a message to Kafka:
```yaml
triggers:
  - template:
      name: kafka-trigger
      kafka:
        url: kafka-broker:9092
        topic: processed-events
        payload:
          - src:
              dependencyName: s3-event
              dataAll: true    # entire event payload
            dest: body
        partitioningKey: "my-key"
```

### Slack Trigger

```yaml
triggers:
  - template:
      name: slack-notify
      slack:
        slackToken:
          name: slack-secret
          key: token
        channel: "#alerts"
        message: "Pipeline triggered for: {{.dependencies.s3-event.notification.s3.object.key}}"
```

---

## Event Payload Transformation

Extract specific fields from the CloudEvent payload and inject into triggers:

```yaml
# CloudEvent payload (s3 notification):
{
  "notification": {
    "eventName": "ObjectCreated:Put",
    "s3": {
      "bucket": {"name": "my-bucket"},
      "object": {"key": "uploads/data_20240101.parquet", "size": 12345678}
    }
  }
}

# Extract key using dataKey (JSONPath syntax):
parameters:
  - src:
      dependencyName: s3-event
      dataKey: notification.s3.object.key   # → "uploads/data_20240101.parquet"
    dest: spec.arguments.parameters.0.value

  - src:
      dependencyName: s3-event
      dataKey: notification.s3.bucket.name  # → "my-bucket"
    dest: spec.arguments.parameters.1.value

# Use dataAll: true to pass entire event payload as JSON string
  - src:
      dependencyName: s3-event
      dataAll: true
    dest: spec.arguments.parameters.0.value
```

**Transformation with template:**
```yaml
parameters:
  - src:
      dependencyName: s3-event
      dataKey: notification.s3.object.key
      # Transform: extract date partition from path
      template: "{{ (split \"/\" .Input)[1] }}"   # "uploads/2024-01-01/file.parquet" → "2024-01-01"
    dest: spec.arguments.parameters.0.value
```

---

## Dependency Logic: Circuit (AND) vs Conditions

### Default: OR Logic

Each trigger fires independently when ANY listed dependency fires:
```yaml
dependencies:
  - name: source-a
    eventSourceName: sources
    eventName: event-a
  - name: source-b
    eventSourceName: sources
    eventName: event-b

triggers:
  - template:
      name: trigger-1
      conditions: "source-a"   # fires on source-a only
  - template:
      name: trigger-2
      conditions: "source-b"   # fires on source-b only
```

### AND Logic via `conditions`

```yaml
dependencies:
  - name: s3-upload
    eventSourceName: sources
    eventName: data-uploads
  - name: calendar
    eventSourceName: sources
    eventName: daily-trigger

triggers:
  - template:
      name: combined-trigger
      conditions: "s3-upload && calendar"   # fires only when BOTH fired
      argoWorkflow:
        ...
```

### Circuit with Time Window

```yaml
triggers:
  - template:
      name: combined-trigger
      conditions: "s3-upload && calendar"
      conditionsReset:
        - byTime:
            cron: "0 0 * * *"    # reset condition state daily
            timezone: "UTC"
```

Without `conditionsReset`, once a condition fires, it stays "met" until the trigger fires. With reset, the window closes and conditions must re-fire in the next cycle.

---

## Common Pattern: GitHub Push → Argo Workflow (CI/CD)

```
GitHub push
  → GitHub sends webhook POST to EventSource
  → EventSource publishes to EventBus (NATS)
  → Sensor receives event
  → Sensor submits Argo Workflow (build + test)
```

```yaml
# event-source.yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
spec:
  github:
    push:
      repositories:
        - owner: my-org
          names: [my-service]
      webhook:
        endpoint: /github
        port: "12000"
      events: [push]
      apiToken:
        name: github-token
        key: token
      secretToken:
        name: github-webhook-secret
        key: secret

---
# sensor.yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
spec:
  dependencies:
    - name: push-event
      eventSourceName: ci-sources
      eventName: push
      filters:
        data:
          - path: body.ref
            type: string
            value:
              - refs/heads/main    # only main branch

  triggers:
    - template:
        name: ci-workflow
        argoWorkflow:
          operation: submit
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: ci-build-
              spec:
                workflowTemplateRef:
                  name: ci-pipeline
                arguments:
                  parameters:
                    - name: repo
                      value: "placeholder"
                    - name: sha
                      value: "placeholder"
          parameters:
            - src:
                dependencyName: push-event
                dataKey: body.repository.clone_url
              dest: spec.arguments.parameters.0.value
            - src:
                dependencyName: push-event
                dataKey: body.after
              dest: spec.arguments.parameters.1.value
```

---

## Common Pattern: S3 Upload → Data Pipeline

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
spec:
  s3:
    uploads:
      bucket:
        name: raw-data-bucket
      region: us-east-1
      events:
        - s3:ObjectCreated:*
      filter:
        suffix: ".parquet"
      credentials:
        accessKey: {name: aws-creds, key: access_key}
        secretKey: {name: aws-creds, key: secret_key}
---
apiVersion: argoproj.io/v1alpha1
kind: Sensor
spec:
  dependencies:
    - name: s3-upload
      eventSourceName: data-sources
      eventName: uploads

  triggers:
    - template:
        name: data-pipeline-trigger
        argoWorkflow:
          operation: submit
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: data-pipeline-
              spec:
                workflowTemplateRef:
                  name: etl-pipeline
                arguments:
                  parameters:
                    - name: s3-bucket
                      value: placeholder
                    - name: s3-key
                      value: placeholder
          parameters:
            - src:
                dependencyName: s3-upload
                dataKey: notification.s3.bucket.name
              dest: spec.arguments.parameters.0.value
            - src:
                dependencyName: s3-upload
                dataKey: notification.s3.object.key
              dest: spec.arguments.parameters.1.value
```

---

## Concrete Example: Multi-Source Trigger (S3 AND Calendar)

Use case: Run a data pipeline only when BOTH a new file arrived AND it's the right time window.

```yaml
# event-source.yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: multi-source
spec:
  eventBusName: default
  s3:
    daily-file:
      bucket:
        name: data-bucket
      region: us-east-1
      events: [s3:ObjectCreated:*]
      filter:
        prefix: "daily/"
        suffix: ".parquet"
      credentials:
        accessKey: {name: aws-creds, key: access_key}
        secretKey: {name: aws-creds, key: secret_key}

  calendar:
    processing-window:
      schedule: "0 3 * * *"   # 3am UTC daily
      timezone: "UTC"
---
# sensor.yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: multi-source-sensor
spec:
  eventBusName: default
  template:
    serviceAccountName: argo-events-sa

  dependencies:
    - name: file-arrived
      eventSourceName: multi-source
      eventName: daily-file

    - name: time-window
      eventSourceName: multi-source
      eventName: processing-window

  triggers:
    - template:
        name: pipeline-trigger
        conditions: "file-arrived && time-window"   # BOTH must fire
        conditionsReset:
          - byTime:
              cron: "0 0 * * *"   # reset daily
              timezone: "UTC"
        argoWorkflow:
          operation: submit
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: daily-pipeline-
                namespace: production
              spec:
                workflowTemplateRef:
                  name: daily-etl
                arguments:
                  parameters:
                    - name: s3-key
                      value: placeholder
                    - name: run-date
                      value: placeholder
          parameters:
            - src:
                dependencyName: file-arrived
                dataKey: notification.s3.object.key
              dest: spec.arguments.parameters.0.value
            - src:
                dependencyName: time-window
                dataKey: eventTime              # ISO timestamp of calendar event
                template: '{{ (split "T" .Input)[0] }}'  # extract date portion
              dest: spec.arguments.parameters.1.value
```

**Execution flow:**
```
Day 1, 2:47am: S3 file arrives
  → EventSource sees ObjectCreated event
  → Publishes to NATS: subject "default:multi-source:daily-file"
  → Sensor receives: marks "file-arrived" = met
  → conditions: "file-arrived && time-window" → false (time-window not yet met)
  → No trigger

Day 1, 3:00am: Calendar fires
  → EventSource emits calendar event
  → Publishes to NATS: subject "default:multi-source:processing-window"
  → Sensor receives: marks "time-window" = met
  → conditions: "file-arrived && time-window" → true!
  → Trigger fires: submits Argo Workflow with s3-key and run-date
  → conditionsReset at midnight → both conditions reset for next day
```

---

## Key Gotchas

1. **EventBus must exist before EventSource/Sensor** — create the EventBus first; other components reference it by name.

2. **NATS Streaming vs JetStream** — NATS Streaming (legacy) doesn't guarantee delivery if the Sensor pod restarts mid-processing. Use JetStream for production.

3. **Webhook port exposure** — EventSource pods need to be reachable. Create a LoadBalancer/Ingress for external webhooks. Use NodePort for internal only.

4. **GitHub webhook auto-registration** — requires `apiToken` to have `admin:repo_hook` scope. Otherwise, register the webhook manually in GitHub pointing to the EventSource service.

5. **`conditions` are stateful per Sensor pod** — if the Sensor pod restarts, in-memory condition state resets. Use JetStream EventBus to persist condition state.

6. **dataKey uses JSONPath-like syntax** — nested keys use `.` notation. Arrays use index: `notification.records.0.s3.object.key`.

7. **One EventSource pod per EventSource CRD** — if you have 20 GitHub repo webhooks, they all funnel through one EventSource pod. The pod is a potential bottleneck; consider splitting across multiple EventSource CRDs.

8. **Rate limiting** — Sensors don't have built-in rate limiting. A flood of S3 events can trigger a flood of Argo Workflows. Add a `rateLimit` to the trigger:
   ```yaml
   triggers:
     - template:
         rateLimit:
           requestsPerUnit: 10
           unit: MINUTE
   ```

9. **Filter evaluation is client-side** — for SQS/Kafka, all messages are consumed and filtered locally. You can't push filters down to the broker in most cases.
