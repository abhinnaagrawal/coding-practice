# Argo Rollouts: Progressive Delivery

## 30-Second Intuition

Argo Rollouts replaces Kubernetes `Deployment` with a `Rollout` CRD that gives you traffic-aware progressive delivery: you define a strategy (canary or blue-green), Argo gradually shifts traffic while running analysis (Prometheus/Datadog queries), and automatically aborts if metrics degrade. Standard Deployments just swap pods — they have no understanding of traffic or success metrics.

---

## Why Standard Deployments Are Insufficient

```
Standard Deployment rollout:
  v1 pods: 10 → 8 → 6 → 4 → 2 → 0
  v2 pods:  0 → 2 → 4 → 6 → 8 → 10

Problems:
  1. Traffic split = replica ratio (coarse, not percentages)
  2. No traffic control layer (all traffic goes to both versions)
  3. No automated metric analysis → no auto-abort
  4. No manual pause/gate between stages
  5. No preview environment
```

With canary: 1% of traffic hits v2 while 99% stays on v1, regardless of replica count.

---

## Rollout CRD

The `Rollout` spec is nearly identical to `Deployment` — it extends the deployment concept:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-service
spec:
  replicas: 10
  selector:
    matchLabels:
      app: my-service
  template:                    # Pod template — same as Deployment
    metadata:
      labels:
        app: my-service
    spec:
      containers:
        - name: app
          image: my-service:v2   # update this to trigger rollout
          ports:
            - containerPort: 8080
  strategy:
    canary:                    # or blueGreen
      steps: [...]
```

**Migration from Deployment:**
1. Change `kind: Deployment` → `kind: Rollout`.
2. Add `strategy` block.
3. Delete the original Deployment (or Argo will conflict).

The Rollout controller manages ReplicaSets just like the Deployment controller — but with traffic awareness added.

---

## Blue-Green Strategy

```
                    ┌─────────────────────┐
  Users ─── active  │  Service (active)   │──→ v1 ReplicaSet (10 pods)
  service →         └─────────────────────┘

  Preview            ┌─────────────────────┐
  service  →         │  Service (preview)  │──→ v2 ReplicaSet (10 pods)
  (testers only)     └─────────────────────┘
```

```yaml
strategy:
  blueGreen:
    activeService: my-service           # production traffic
    previewService: my-service-preview  # test traffic
    autoPromotionEnabled: false         # require manual promotion
    autoPromotionSeconds: 0
    prePromotionAnalysis:               # run analysis before switching
      templates:
        - templateName: success-rate
    postPromotionAnalysis:              # run after switch (optional rollback window)
      templates:
        - templateName: success-rate
    scaleDownDelaySeconds: 30           # keep old ReplicaSet alive 30s after switch
```

**Blue-Green flow:**
```
1. Update Rollout image → v2 ReplicaSet created (full replica count)
2. v2 pods become Ready → preview service now routes to v2
3. prePromotionAnalysis runs → checks metrics
4. If analysis passes + manual approval: active service switches to v2
5. scaleDownDelay: old v1 pods stay alive briefly for graceful drain
6. v1 ReplicaSet scaled down
```

**Abort blue-green:**
```bash
kubectl argo rollouts abort my-service
# → active service stays on v1, v2 scaled down
```

---

## Canary Strategy

Progressive traffic shifting with gates:

```yaml
strategy:
  canary:
    stableService: my-service-stable
    canaryService: my-service-canary
    trafficRouting:
      istio:
        virtualService:
          name: my-service-vs
          routes:
            - primary
    steps:
      - setWeight: 5          # 5% to canary
      - pause: {duration: 5m}
      - analysis:             # run analysis at 5%
          templates:
            - templateName: success-rate
      - setWeight: 25
      - pause: {duration: 10m}
      - setWeight: 50
      - pause: {}             # pause indefinitely (manual approval)
      - setWeight: 100
```

**Canary flow:**
```
Step 1: setWeight: 5    → Istio VirtualService: 95% stable, 5% canary
Step 2: pause 5m        → wait
Step 3: analysis        → query Prometheus for 5 minutes
        if error rate > threshold → abort → revert to stable
        if OK → continue
Step 4: setWeight: 25   → 75% stable, 25% canary
...
Step 7: pause {}        → wait for: kubectl argo rollouts promote my-service
Step 8: setWeight: 100  → stable service updated to point to new ReplicaSet
```

---

## Traffic Management Integrations

### How Weight Splitting Works

Argo Rollouts manages your traffic layer's weights, not just replica counts.

#### Istio VirtualService

```yaml
# Before rollout: Argo manages this
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-service-vs
spec:
  http:
    - name: primary
      route:
        - destination:
            host: my-service-stable
          weight: 95     # ← Argo updates these
        - destination:
            host: my-service-canary
          weight: 5      # ← based on canary step
```

Argo patches the VirtualService weights at each step. The stable service always points to the current stable ReplicaSet; canary service to the new ReplicaSet.

#### NGINX Ingress

```yaml
trafficRouting:
  nginx:
    stableIngress: my-ingress   # Argo creates a canary ingress with annotation
    annotationPrefix: nginx.ingress.kubernetes.io
    additionalIngressAnnotations:
      canary-by-header: X-Canary
```

Argo creates a second Ingress with `nginx.ingress.kubernetes.io/canary: "true"` and `canary-weight: 5` annotations.

#### AWS ALB

```yaml
trafficRouting:
  alb:
    ingress: my-alb-ingress
    servicePort: 80
    annotationPrefix: alb.ingress.kubernetes.io
```

Argo updates ALB's weighted target group split via annotations.

---

## AnalysisTemplate

Defines how to measure success/failure during a rollout:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
    - name: canary-hash
  metrics:
    - name: success-rate
      interval: 2m               # query every 2 minutes
      count: 5                   # total of 5 measurements
      successCondition: result[0] >= 0.95
      failureLimit: 2            # fail if 2 out of 5 below threshold
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(
              http_requests_total{
                service="{{args.service-name}}",
                status!~"5.."
              }[2m]
            )) /
            sum(rate(
              http_requests_total{
                service="{{args.service-name}}"
              }[2m]
            ))

    - name: latency-p99
      interval: 1m
      count: 10
      successCondition: result[0] < 0.5   # < 500ms
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            histogram_quantile(0.99,
              rate(http_request_duration_seconds_bucket{
                service="{{args.service-name}}"
              }[1m])
            )
```

**Analysis phases:**
- `Pending` → analysis hasn't started.
- `Running` → actively querying metrics.
- `Successful` → all metrics passed.
- `Failed` → failureLimit exceeded → rollout aborts.
- `Error` → couldn't query metrics → configurable behavior.
- `Inconclusive` → not enough data.

### Background Analysis

Run analysis continuously from start (not just at `analysis` steps):

```yaml
strategy:
  canary:
    analysis:
      startingStep: 2           # begin after step 2
      templates:
        - templateName: success-rate
      args:
        - name: service-name
          value: my-service
    steps:
      - setWeight: 5
      - pause: {duration: 10m}
      - setWeight: 50
      - pause: {duration: 10m}
```

---

## Experiment CRD

Run multiple versions simultaneously for A/B testing:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Experiment
metadata:
  name: ab-test-experiment
spec:
  duration: 24h
  templates:
    - name: baseline
      replicas: 5
      spec:
        containers:
          - name: app
            image: my-service:v1
    - name: candidate
      replicas: 5
      spec:
        containers:
          - name: app
            image: my-service:v2
  analyses:
    - name: compare-success-rate
      templateName: ab-analysis
      args:
        - name: baseline-hash
          valueFrom:
            type: PodTemplateHash
            podTemplateHashValue: Stable
        - name: canary-hash
          valueFrom:
            type: PodTemplateHash
            podTemplateHashValue: Latest
```

Experiments are usually embedded within a canary step:
```yaml
steps:
  - experiment:
      duration: 1h
      templates:
        - name: experiment-v2
          specRef: canary
      analyses:
        - name: ab-test
          templateName: ab-success-rate
```

---

## kubectl Plugin

```bash
# Install
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-darwin-amd64
chmod +x kubectl-argo-rollouts-darwin-amd64 && mv it /usr/local/bin/kubectl-argo-rollouts

# Status
kubectl argo rollouts get rollout my-service --watch

# Promote (advance past a pause step)
kubectl argo rollouts promote my-service

# Abort (revert to stable)
kubectl argo rollouts abort my-service

# Pause
kubectl argo rollouts pause my-service

# Resume
kubectl argo rollouts resume my-service

# Retry failed rollout
kubectl argo rollouts retry rollout my-service

# Set image (triggers rollout)
kubectl argo rollouts set image my-service app=my-service:v3

# List all rollouts
kubectl argo rollouts list rollouts -n my-namespace
```

---

## Integration with ArgoCD

ArgoCD manages GitOps sync; Argo Rollouts manages progressive delivery. They compose naturally:

```
Git repo:           Argo CD:               Argo Rollouts:
  rollout.yaml  →   syncs to cluster  →    manages traffic/analysis
  analysis.yaml
  virtualservice.yaml

Flow:
1. Dev merges PR (bumps image tag in rollout.yaml)
2. ArgoCD detects diff → syncs → applies new Rollout spec
3. Rollout controller sees image change → begins canary steps
4. Analysis runs against Prometheus
5. If success: Rollout promotes to stable
6. ArgoCD sees Rollout as "Healthy" → sync complete
```

**Important**: ArgoCD considers a Rollout "Progressing" (not Healthy) during a rollout. Don't configure ArgoCD to auto-sync before a rollout completes — it could interrupt mid-rollout.

ArgoCD Rollouts integration adds rich UI showing traffic weights and analysis results in the ArgoCD UI.

---

## Concrete Example: Canary with Prometheus Analysis

Full YAML for a canary rollout with automated abort on error rate:

```yaml
# analysis-template.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: http-success-rate
spec:
  args:
    - name: canary-service
  metrics:
    - name: success-rate
      interval: 1m
      count: 10
      successCondition: result[0] >= 0.99
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus-operated.monitoring.svc:9090
          query: |
            sum(rate(
              istio_requests_total{
                destination_service_name="{{args.canary-service}}",
                response_code!~"5.."
              }[1m]
            )) /
            sum(rate(
              istio_requests_total{
                destination_service_name="{{args.canary-service}}"
              }[1m]
            ))
---
# rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api-service
  namespace: production
spec:
  replicas: 20
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      containers:
        - name: api
          image: api-service:v2.1.0     # update this to trigger rollout
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
  strategy:
    canary:
      stableService: api-service-stable
      canaryService: api-service-canary
      trafficRouting:
        istio:
          virtualService:
            name: api-service-vs
            routes:
              - primary
      steps:
        - setWeight: 1                      # 1% canary
        - pause: {duration: 5m}
        - analysis:                         # gate: check success rate at 1%
            templates:
              - templateName: http-success-rate
            args:
              - name: canary-service
                value: api-service-canary
        - setWeight: 10
        - pause: {duration: 10m}
        - setWeight: 30
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {}                         # manual gate: ops reviews dashboards
        - setWeight: 100
      analysis:                             # background analysis throughout
        startingStep: 1
        templates:
          - templateName: http-success-rate
        args:
          - name: canary-service
            value: api-service-canary
---
# services.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service-stable
spec:
  selector:
    app: api-service
  ports: [{port: 80, targetPort: 8080}]
---
apiVersion: v1
kind: Service
metadata:
  name: api-service-canary
spec:
  selector:
    app: api-service
  ports: [{port: 80, targetPort: 8080}]
---
# virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-service-vs
spec:
  http:
    - name: primary
      route:
        - destination:
            host: api-service-stable
          weight: 100    # Argo manages these weights
        - destination:
            host: api-service-canary
          weight: 0
```

**Rollout execution trace:**
```
t=0:   image updated to v2.1.0
       → Argo creates canary ReplicaSet (2 pods for 1% weight)
       → VirtualService: stable=99%, canary=1%

t=5m:  setWeight: 1 pause expires
       → AnalysisRun created: queries Prometheus for success rate
       → 10 measurements over 10 minutes
       → result: 0.9998 → pass

t=15m: setWeight: 10
       → VirtualService: stable=90%, canary=10%
       → canary RS scaled up to ~2 pods (10% of 20)

t=25m: setWeight: 30 → 70/30 split

t=35m: setWeight: 50 → 50/50 split

t=35m: pause {} → ops team sees 50/50 healthy for 10min → promotes:
       kubectl argo rollouts promote api-service

t=35m+: setWeight: 100
        → VirtualService: stable=0%, canary=100%
        → stable service selector updated to new RS
        → old RS scaled down
        → Rollout phase: Healthy
```

If at any point success-rate < 0.99 for 3 consecutive measurements:
```
AnalysisRun: Failed
→ Rollout: Abort
→ VirtualService instantly reverts to 100% stable
→ Canary RS scaled to 0
```

---

## Rollout vs Deployment: Migration Path

```bash
# 1. Convert Deployment to Rollout
kubectl get deployment my-service -o yaml | \
  sed 's/kind: Deployment/kind: Rollout/' | \
  sed '/  strategy:/,/    type:.*/d' > rollout.yaml

# 2. Add strategy block to rollout.yaml manually

# 3. Apply Rollout
kubectl apply -f rollout.yaml

# 4. Delete original Deployment (important: Rollout manages RS now)
kubectl delete deployment my-service

# 5. Verify
kubectl argo rollouts get rollout my-service
```

---

## Key Gotchas

1. **Delete the Deployment after creating Rollout** — if both exist, they fight over the same ReplicaSets.

2. **Services must have distinct names** — stable and canary services must be different Services. Don't try to reuse the same service.

3. **Weight 100 ≠ rollout complete** — after `setWeight: 100`, Argo still needs to update the stable service selector and scale down the old RS. Rollout is "Healthy" only when this is done.

4. **Analysis args must match AnalysisTemplate** — mismatch causes AnalysisRun to fail with an error (not a metric failure).

5. **`pause: {}` blocks indefinitely** — requires explicit `kubectl argo rollouts promote` or the rollout never advances. Add a timeout with `pause: {duration: 24h}` if you want auto-advance.

6. **Prometheus query must return a scalar** — `result[0]` is the first element. If your PromQL returns no data, the analysis fails with "Inconclusive" (configurable).

7. **VirtualService must have exactly the named route** — Argo looks for the route name specified in `routes`. If it doesn't match, traffic splitting silently fails.

8. **ReplicaSet count during canary** — during a 10% canary with 20 total pods: canary RS has 2 pods, stable RS has 18 pods. The `replicas` in the Rollout spec is total desired.
