# ClickHouse Operator — Complete Guide

> **Operator covered:** Official ClickHouse Inc. operator
> **GitHub:** https://github.com/ClickHouse/clickhouse-operator
> **Docs:** https://clickhouse.com/docs/clickhouse-operator/overview
> **Version at writing:** v0.0.3 (March 2026)

---

## Table of Contents

1. [Quick Start](#1-quick-start) ← *video tutorial section*
2. [Detailed Start](#2-detailed-start)
3. [Customisation](#3-customisation)
4. [Operations](#4-operations)
5. [Troubleshooting](#5-troubleshooting)
6. [Comparison: Official vs Altinity Operator](#6-comparison-official-vs-altinity-operator)
7. [Appendix](#7-appendix)

---

## 1. Quick Start

> **Goal:** Get a working ClickHouse cluster running in Kubernetes in under 5 minutes.
> **Prerequisites:** A running Kubernetes cluster, `kubectl` configured.

### Step 1 — Install cert-manager (required dependency)

The operator uses webhooks; cert-manager issues the TLS certificates for them.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

Wait for cert-manager to be ready:

```bash
kubectl wait --for=condition=Ready pods --all -n cert-manager --timeout=120s
```

### Step 2 — Install the ClickHouse Operator

```bash
kubectl apply -f https://github.com/ClickHouse/clickhouse-operator/releases/latest/download/clickhouse-operator.yaml
```

This installs:
- The operator `Deployment` in the `clickhouse-operator-system` namespace
- Two CRDs: `ClickHouseCluster` and `KeeperCluster`
- RBAC roles, service accounts, and webhook configurations

Verify it is running:

```bash
kubectl get pods -n clickhouse-operator-system
```

Expected output:
```
NAME                                    READY   STATUS    RESTARTS   AGE
clickhouse-operator-xxxx-xxxx           1/1     Running   0          30s
```

### Step 3 — Deploy a ClickHouse Cluster

Apply the minimal example (includes ClickHouse + Keeper):

```bash
kubectl apply -f https://raw.githubusercontent.com/ClickHouse/clickhouse-operator/refs/heads/main/examples/minimal.yaml
```

This creates:
- A **KeeperCluster** with 1 replica (coordination layer)
- A **ClickHouseCluster** with 1 shard, 2 replicas, 1Gi storage

Wait for the cluster to be ready:

```bash
kubectl get clickhouseclusters -w
kubectl get pods -w
```

### Step 4 — Connect and Test

Get the service name:

```bash
kubectl get svc
```

Port-forward to the ClickHouse HTTP interface:

```bash
kubectl port-forward svc/<cluster-name> 8123:8123
```

Test the connection:

```bash
curl http://localhost:8123/ping
# Expected: Ok.

curl "http://localhost:8123/?query=SELECT+version()"
# Expected: 25.x.x.x
```

Connect with the native client:

```bash
kubectl exec -it <clickhouse-pod-name> -- clickhouse-client
```

Run a quick smoke test inside the client:

```sql
CREATE DATABASE test;
CREATE TABLE test.events (id UInt64, name String) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}') ORDER BY id;
INSERT INTO test.events VALUES (1, 'hello'), (2, 'world');
SELECT * FROM test.events;
```

---

## 2. Detailed Start

### 2.1 Architecture Overview

The operator manages two distinct Custom Resources:

| Resource | Purpose | Key fields |
|---|---|---|
| `KeeperCluster` | ClickHouse Keeper — distributed coordination (replaces ZooKeeper) | replicas (must be odd), storage, TLS |
| `ClickHouseCluster` | ClickHouse database nodes | shards, replicas per shard, storage, keeper reference |

**Deployment order:** Always create the `KeeperCluster` before the `ClickHouseCluster`, since ClickHouse nodes register with Keeper at startup.

### 2.2 Installing via Helm (alternative)

```bash
helm install clickhouse-operator oci://ghcr.io/clickhouse/clickhouse-operator-helm \
  --create-namespace \
  -n clickhouse-operator-system
```

Helm is recommended for production because it gives you:
- Version-pinned installs
- Easy upgrades via `helm upgrade`
- Custom values for operator configuration

### 2.3 Namespace Strategy

By default the operator watches **all namespaces**. You can restrict it at install time via Helm values:

```bash
helm install clickhouse-operator oci://ghcr.io/clickhouse/clickhouse-operator-helm \
  -n clickhouse-operator-system \
  --set watchNamespace=production
```

For multi-tenant clusters, run one operator per namespace.

### 2.4 Deploying a Production-Grade Cluster

**keeper.yaml** — HA Keeper with 3 replicas:

```yaml
apiVersion: clickhouse.com/v1alpha1
kind: KeeperCluster
metadata:
  name: keeper
  namespace: default
spec:
  replicas: 3
  storage:
    size: 10Gi
    className: fast-ssd
  podTemplate:
    spec:
      resources:
        requests:
          cpu: "500m"
          memory: "1Gi"
        limits:
          cpu: "1"
          memory: "2Gi"
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: keeper
```

**clickhouse.yaml** — 2-shard, 2-replica-per-shard cluster:

```yaml
apiVersion: clickhouse.com/v1alpha1
kind: ClickHouseCluster
metadata:
  name: clickhouse
  namespace: default
spec:
  shards: 2
  replicas: 2
  keeper:
    name: keeper
  storage:
    size: 100Gi
    className: fast-ssd
  podTemplate:
    spec:
      resources:
        requests:
          cpu: "2"
          memory: "8Gi"
        limits:
          cpu: "4"
          memory: "16Gi"
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: clickhouse
```

Apply both:

```bash
kubectl apply -f keeper.yaml
kubectl apply -f clickhouse.yaml
```

### 2.5 Understanding the Pods and Services Created

For a 2-shard, 2-replica cluster named `clickhouse`, the operator creates:

| Resource | Name pattern | Purpose |
|---|---|---|
| Pods | `clickhouse-shard-X-replica-Y` | ClickHouse instances |
| StatefulSets | One per shard | Manages pod lifecycle |
| Services | `clickhouse` | Load-balanced entry point |
| Services | `clickhouse-headless` | DNS for pod-to-pod communication |
| PVCs | One per pod | Persistent data storage |

### 2.6 Verifying Replication

Inside any ClickHouse pod:

```sql
-- Check Keeper connectivity
SELECT * FROM system.zookeeper WHERE path = '/';

-- Check cluster topology
SELECT * FROM system.clusters WHERE cluster = 'default';

-- Check replica status
SELECT database, table, replica_name, is_leader FROM system.replicas;
```

---

## 3. Customisation

### 3.1 Custom ClickHouse Configuration

Inject ClickHouse server settings via `extraSettings`:

```yaml
apiVersion: clickhouse.com/v1alpha1
kind: ClickHouseCluster
metadata:
  name: clickhouse
spec:
  shards: 1
  replicas: 2
  keeper:
    name: keeper
  extraSettings:
    max_connections: "4096"
    max_concurrent_queries: "200"
    mark_cache_size: "5368709120"    # 5GB
    uncompressed_cache_size: "8589934592"  # 8GB
    max_server_memory_usage_to_ram_ratio: "0.9"
```

### 3.2 User Management

```yaml
spec:
  users:
    - name: admin
      password: "secure-password-here"
      networks:
        - "10.0.0.0/8"
        - "::/0"
      profile: default
      quota: default
    - name: readonly
      password: "readonly-pass"
      profile: readonly
      quota: default
```

For production, reference passwords from Kubernetes Secrets:

```yaml
spec:
  users:
    - name: admin
      passwordSecret:
        name: clickhouse-admin-secret
        key: password
```

### 3.3 TLS / SSL Encryption

Requires cert-manager with a configured `Issuer` or `ClusterIssuer`.

```yaml
apiVersion: clickhouse.com/v1alpha1
kind: KeeperCluster
metadata:
  name: keeper
spec:
  replicas: 3
  tls:
    enabled: true
    required: true
    certSecret: keeper-tls-cert   # Secret containing tls.crt and tls.key
---
apiVersion: clickhouse.com/v1alpha1
kind: ClickHouseCluster
metadata:
  name: clickhouse
spec:
  shards: 1
  replicas: 2
  keeper:
    name: keeper
  tls:
    enabled: true
    required: true
    certSecret: clickhouse-tls-cert
```

See the full working TLS example:

```bash
kubectl apply -f https://raw.githubusercontent.com/ClickHouse/clickhouse-operator/refs/heads/main/examples/cluster_with_ssl.yaml
```

### 3.4 Cloud-Specific Storage

**AWS EKS with GP3:**

```yaml
spec:
  storage:
    size: 500Gi
    className: gp3
```

Apply the AWS-optimised example:

```bash
kubectl apply -f https://raw.githubusercontent.com/ClickHouse/clickhouse-operator/refs/heads/main/examples/aws_eks_gp3.yaml
```

**GCP GKE with SSD:**

```bash
kubectl apply -f https://raw.githubusercontent.com/ClickHouse/clickhouse-operator/refs/heads/main/examples/gcp_gke_ssd.yaml
```

### 3.5 Pod Anti-Affinity and Zone Spreading

Spread pods across availability zones to tolerate zone failures:

```yaml
spec:
  podTemplate:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: clickhouse
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app.kubernetes.io/name: clickhouse
              topologyKey: kubernetes.io/hostname
```

### 3.6 Resource Tuning

```yaml
spec:
  podTemplate:
    spec:
      resources:
        requests:
          cpu: "4"
          memory: "32Gi"
        limits:
          cpu: "8"
          memory: "64Gi"
      env:
        - name: CLICKHOUSE_MAX_SERVER_MEMORY_USAGE
          value: "30000000000"    # 30GB
```

### 3.7 PodDisruptionBudget

Protect cluster availability during node drains:

```yaml
spec:
  podDisruptionBudget:
    maxUnavailable: 1
```

---

## 4. Operations

### 4.1 Checking Cluster Status

```bash
# List all ClickHouse clusters
kubectl get clickhouseclusters

# Describe a specific cluster (shows events, status, conditions)
kubectl describe clickhousecluster clickhouse

# List Keeper clusters
kubectl get keeperclusters
kubectl describe keepercluster keeper

# Watch pods
kubectl get pods -l app.kubernetes.io/name=clickhouse -w
```

### 4.2 Scaling — Adding Replicas

Edit the cluster spec and increase `replicas`:

```bash
kubectl patch clickhousecluster clickhouse --type='merge' -p '{"spec":{"replicas":3}}'
```

The operator will:
1. Create new pods and PVCs
2. Wait for ClickHouse to start
3. Trigger database/table synchronisation from existing replicas

### 4.3 Scaling — Adding Shards

```bash
kubectl patch clickhousecluster clickhouse --type='merge' -p '{"spec":{"shards":3}}'
```

**Note:** Adding shards does not automatically rebalance existing data. You must manually move data using `ALTER TABLE ... MOVE PARTITION` or use a distributed table that routes new inserts to all shards.

### 4.4 Upgrading ClickHouse Version

```bash
kubectl patch clickhousecluster clickhouse --type='merge' \
  -p '{"spec":{"image":"clickhouse/clickhouse-server:25.3"}}'
```

The operator performs a rolling upgrade — one replica at a time per shard — so the cluster remains available.

### 4.5 Upgrading the Operator

**Helm:**

```bash
helm upgrade clickhouse-operator oci://ghcr.io/clickhouse/clickhouse-operator-helm \
  -n clickhouse-operator-system
```

**Manifest:**

```bash
kubectl apply -f https://github.com/ClickHouse/clickhouse-operator/releases/latest/download/clickhouse-operator.yaml
```

Upgrading the operator does **not** restart ClickHouse pods.

### 4.6 Backup and Restore

The operator does not include a built-in backup tool. Recommended approaches:

**Option A — clickhouse-backup (Altinity community tool):**

```bash
# Run as a sidecar or separate job
clickhouse-backup create backup-$(date +%Y%m%d)
clickhouse-backup upload backup-$(date +%Y%m%d)
```

**Option B — Volume snapshots (cloud-native):**

```bash
kubectl apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: clickhouse-data-snapshot
spec:
  source:
    persistentVolumeClaimName: data-clickhouse-shard-0-replica-0
EOF
```

**Option C — Freeze + copy:**

```sql
-- Inside ClickHouse
ALTER TABLE my_table FREEZE;
-- Then copy files from /var/lib/clickhouse/shadow/
```

### 4.7 Prometheus Metrics

Apply the Prometheus scraper example:

```bash
kubectl apply -f https://raw.githubusercontent.com/ClickHouse/clickhouse-operator/refs/heads/main/examples/prometheus_secure_metrics_scraper.yaml
```

Key metrics to monitor:

| Metric | What it tells you |
|---|---|
| `clickhouse_uptime` | Pod restarts |
| `clickhouse_metrics_Query` | Active queries |
| `clickhouse_metrics_Merge` | Active merges |
| `clickhouse_async_metrics_MemoryResident` | RSS memory |
| `clickhouse_events_FailedQuery` | Query errors |
| `clickhouse_parts_*` | Part counts per table |

### 4.8 Deleting a Cluster

```bash
# Delete ClickHouse cluster (PVCs are retained by default)
kubectl delete clickhousecluster clickhouse

# Delete Keeper cluster
kubectl delete keepercluster keeper

# To also delete PVCs (data loss!):
kubectl delete pvc -l app.kubernetes.io/name=clickhouse
kubectl delete pvc -l app.kubernetes.io/name=keeper
```

---

## 5. Troubleshooting

### 5.1 Operator Pod Not Starting

```bash
kubectl logs -n clickhouse-operator-system deploy/clickhouse-operator
kubectl describe pod -n clickhouse-operator-system <operator-pod>
```

**Common cause:** cert-manager webhooks not ready. Wait for cert-manager pods and re-apply:

```bash
kubectl wait --for=condition=Ready pods --all -n cert-manager --timeout=120s
kubectl rollout restart deploy/clickhouse-operator -n clickhouse-operator-system
```

### 5.2 ClickHouse Pods Stuck in Pending

```bash
kubectl describe pod <clickhouse-pod>
```

**Cause: Insufficient resources**
Check events for `Insufficient cpu` or `Insufficient memory`. Reduce resource requests or add nodes.

**Cause: PVC not bound**
```bash
kubectl get pvc
kubectl describe pvc <pvc-name>
```
Check that your `storageClassName` matches an available `StorageClass`:
```bash
kubectl get storageclass
```

**Cause: Topology spread constraint unsatisfiable**
You have fewer nodes/zones than required by `whenUnsatisfiable: DoNotSchedule`. Change to `ScheduleAnyway` or add nodes.

### 5.3 ClickHouse Pod CrashLoopBackOff

```bash
kubectl logs <clickhouse-pod>
kubectl logs <clickhouse-pod> --previous
```

**Common causes:**

| Log message | Fix |
|---|---|
| `Exception: Cannot connect to ZooKeeper` | Keeper cluster not running; check `kubectl get keepercluster` |
| `DB::Exception: Can't get data for node /clickhouse/...` | Keeper path conflict; check for existing data from a previous cluster |
| `Out of memory` | Increase memory limits or tune `max_server_memory_usage` |
| `Permission denied: /var/lib/clickhouse` | PVC permissions issue; check `securityContext.fsGroup` |

### 5.4 Keeper Pods Forming a Quorum

```bash
# Check Keeper logs
kubectl logs keeper-0

# Check quorum status from inside a Keeper pod
kubectl exec -it keeper-0 -- clickhouse-keeper-client -h localhost -p 9181
# Then inside the client:
stat
```

Keeper requires a majority quorum. With 3 replicas, 2 must be running. If you have 1/3 running, the cluster is read-only.

### 5.5 Replication Not Working

```sql
-- Check replication queue
SELECT * FROM system.replication_queue WHERE is_currently_executing = 1;

-- Check replica errors
SELECT database, table, last_exception FROM system.replicas WHERE last_exception != '';

-- Check Keeper connection
SELECT * FROM system.zookeeper WHERE path = '/';
```

**Fix for stuck replication:**

```sql
-- Detach and re-attach the table to reset replication state
ALTER TABLE my_table DETACH;
ALTER TABLE my_table ATTACH;
```

### 5.6 Cluster Reconciliation Stuck

```bash
kubectl describe clickhousecluster clickhouse
# Look at the Status.Conditions section
```

Force reconciliation by adding/removing an annotation:

```bash
kubectl annotate clickhousecluster clickhouse reconcile=$(date +%s) --overwrite
```

### 5.7 Useful Debug Commands

```bash
# Exec into a ClickHouse pod
kubectl exec -it <pod> -- clickhouse-client

# Check system tables
# Inside clickhouse-client:
SELECT * FROM system.disks;
SELECT * FROM system.processes;
SELECT * FROM system.errors ORDER BY last_error_time DESC LIMIT 20;
SELECT * FROM system.query_log ORDER BY event_time DESC LIMIT 10;
```

---

## 6. Comparison: Official vs Altinity Operator

| Dimension | Official (ClickHouse Inc.) | Altinity Operator |
|---|---|---|
| **GitHub** | github.com/ClickHouse/clickhouse-operator | github.com/Altinity/clickhouse-operator |
| **Maintained by** | ClickHouse Inc. | Altinity Inc. |
| **Maturity** | v0.0.3 — new (2025) | v0.26 — mature (since 2019) |
| **Stars / adoption** | Growing | ~2,500 stars, widely adopted |
| **Primary CRD** | `ClickHouseCluster` | `ClickHouseInstallation` (CHI) |
| **Coordination CRD** | `KeeperCluster` (built-in) | Separate ZooKeeper setup / manual Keeper |
| **API version** | `clickhouse.com/v1alpha1` | `clickhouse.altinity.com/v1` |
| **cert-manager required** | Yes (for webhooks) | No |
| **Helm chart** | Yes (OCI registry) | Yes |
| **Config granularity** | Moderate — clean, opinionated API | Very high — exposes almost all ClickHouse XML config |
| **Replication templates** | Handled by operator | Configurable shard/replica topology |
| **Cluster topology** | `shards` + `replicas` integers | Full shard/replica definition with named groups |
| **TLS/SSL** | Native cert-manager integration | Supported but manual cert management |
| **Monitoring** | Prometheus example provided | Prometheus + Grafana dashboards provided |
| **Schema migration** | Not built-in | Schema propagation on scale-out |
| **Cloud examples** | AWS EKS, GCP GKE | Generic / community contributed |
| **Commercial support** | ClickHouse Cloud / ClickHouse Inc. | Altinity.Cloud / Altinity Enterprise |
| **Best for** | New projects, tight ClickHouse Inc. ecosystem alignment | Large existing deployments, maximum configurability |

### When to choose the Official Operator

- Starting a new project and want the most up-to-date, vendor-maintained solution
- You are already using cert-manager
- You prefer a simpler, more opinionated API
- You want tight integration with ClickHouse Inc. roadmap and future features

### When to choose the Altinity Operator

- You need battle-tested stability (7 years of production use)
- Your cluster needs fine-grained ClickHouse XML configuration control
- You are migrating an existing Altinity deployment
- You want the richest community resources, examples, and forum support
- You need schema propagation on scale-out out of the box

---

## 7. Appendix

### A. Custom Resource — ClickHouseCluster Full Schema

```yaml
apiVersion: clickhouse.com/v1alpha1
kind: ClickHouseCluster
metadata:
  name: <name>
  namespace: <namespace>
spec:
  # Cluster topology
  shards: 1                         # Number of shards (horizontal scale-out)
  replicas: 2                       # Replicas per shard (HA)

  # ClickHouse Keeper reference
  keeper:
    name: <keeper-cluster-name>     # Must be a KeeperCluster in the same namespace

  # Container image
  image: clickhouse/clickhouse-server:latest

  # Persistent storage per pod
  storage:
    size: 100Gi
    className: fast-ssd

  # Pod template (applied to all pods)
  podTemplate:
    spec:
      resources:
        requests:
          cpu: "2"
          memory: "8Gi"
        limits:
          cpu: "4"
          memory: "16Gi"
      env: []
      topologySpreadConstraints: []
      affinity: {}
      tolerations: []

  # TLS configuration
  tls:
    enabled: false
    required: false
    certSecret: ""                  # K8s Secret with tls.crt / tls.key

  # User definitions
  users:
    - name: default
      password: ""                  # Or use passwordSecret
      passwordSecret:
        name: ""
        key: ""

  # Raw ClickHouse server settings
  extraSettings:
    max_connections: "4096"

  # PodDisruptionBudget
  podDisruptionBudget:
    maxUnavailable: 1
```

### B. Custom Resource — KeeperCluster Full Schema

```yaml
apiVersion: clickhouse.com/v1alpha1
kind: KeeperCluster
metadata:
  name: <name>
  namespace: <namespace>
spec:
  # Must be odd: 1, 3, 5, 7, 9, 11, 13, or 15
  replicas: 3

  storage:
    size: 10Gi
    className: fast-ssd

  podTemplate:
    spec:
      resources:
        requests:
          cpu: "500m"
          memory: "1Gi"
        limits:
          cpu: "1"
          memory: "2Gi"
      topologySpreadConstraints: []
      affinity: {}

  tls:
    enabled: false
    required: false
    certSecret: ""

  podDisruptionBudget:
    maxUnavailable: 1
```

### C. Altinity Operator — ClickHouseInstallation Schema Overview

The Altinity operator uses a single `ClickHouseInstallation` (CHI) CRD with a deeply nested structure:

```yaml
apiVersion: clickhouse.altinity.com/v1
kind: ClickHouseInstallation
metadata:
  name: my-cluster
spec:
  defaults:
    replicasUseFQDN: "yes"
    distributedDDL:
      profile: default

  configuration:
    zookeeper:
      nodes:
        - host: zookeeper.default.svc
          port: 2181

    users:
      admin/password: "password"
      admin/networks/ip:
        - "0.0.0.0/0"

    profiles:
      default/max_memory_usage: 10000000000

    settings:
      max_connections: 4096

    clusters:
      - name: mycluster
        layout:
          shardsCount: 2
          replicasCount: 2

  templates:
    podTemplates:
      - name: default
        spec:
          containers:
            - name: clickhouse
              image: clickhouse/clickhouse-server:25.3
              resources:
                requests:
                  cpu: "2"
                  memory: "8Gi"

    volumeClaimTemplates:
      - name: data
        spec:
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 100Gi
```

The Altinity CHI gives you control over each shard and replica individually if needed — a level of granularity the official operator does not yet expose.

### D. DNS and Connectivity

Inside the cluster, pods discover each other via headless service DNS:

| Pattern | Example |
|---|---|
| ClickHouse pod | `clickhouse-shard-0-replica-0.clickhouse-headless.default.svc.cluster.local` |
| Keeper pod | `keeper-0.keeper-headless.default.svc.cluster.local` |
| ClickHouse service (LB) | `clickhouse.default.svc.cluster.local` |

ClickHouse is configured with these DNS names automatically by the operator.

### E. Ports Reference

| Port | Protocol | Purpose |
|---|---|---|
| 8123 | HTTP | REST queries, ping, metrics |
| 9000 | TCP | Native ClickHouse client protocol |
| 9009 | TCP | Inter-replica data transfer |
| 9181 | TCP | ClickHouse Keeper client port |
| 9234 | TCP | Keeper Raft port (inter-Keeper) |
| 9363 | HTTP | Prometheus metrics endpoint |

### F. Useful kubectl Aliases

```bash
# Add to ~/.zshrc or ~/.bashrc
alias kch="kubectl get clickhouseclusters"
alias kk="kubectl get keeperclusters"
alias kpods="kubectl get pods -l app.kubernetes.io/name=clickhouse"
alias klogs="kubectl logs -l app.kubernetes.io/name=clickhouse --all-containers"

# Exec into the first ClickHouse pod
alias chclient='kubectl exec -it $(kubectl get pod -l app.kubernetes.io/name=clickhouse -o jsonpath="{.items[0].metadata.name}") -- clickhouse-client'
```

### G. Minimum Viable Production Checklist

- [ ] Keeper has 3 replicas across 3 zones
- [ ] ClickHouse has at least 2 replicas per shard
- [ ] PVCs use a durable storage class (retain policy)
- [ ] Resource requests and limits set
- [ ] Pod anti-affinity configured
- [ ] PodDisruptionBudget set (`maxUnavailable: 1`)
- [ ] TLS enabled for inter-node and client traffic
- [ ] Prometheus scraping configured
- [ ] Backup job running on schedule
- [ ] `default` user password changed
- [ ] Operator version pinned (not `latest`)

### H. Further Reading

- [Official docs — clickhouse-operator overview](https://clickhouse.com/docs/clickhouse-operator/overview)
- [GitHub — ClickHouse/clickhouse-operator](https://github.com/ClickHouse/clickhouse-operator)
- [GitHub — Altinity/clickhouse-operator](https://github.com/Altinity/clickhouse-operator)
- [Altinity operator docs](https://docs.altinity.com/clickhouseonkubernetes/)
- [ClickHouse Keeper docs](https://clickhouse.com/docs/guides/sre/keeper/clickhouse-keeper)
- [cert-manager installation](https://cert-manager.io/docs/installation/)
