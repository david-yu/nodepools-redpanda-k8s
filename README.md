# Redpanda NodePool Migration Guide (kind)

Step-by-step guide for testing NodePool migrations on a local kind cluster. Covers deploying a Redpanda cluster, migrating to NodePools, performing a blue/green version upgrade via NodePools, and optionally reverting back to Redpanda CR-managed brokers.

> **Operator version tested:** v26.1.1 (chart 26.1.1)
> **NodePool feature status:** Experimental (requires `--enable-v2-nodepools` flag and experimental CRDs)

## Prerequisites

- [kind](https://kind.sigs.k8s.io/) v0.20+
- [kubectl](https://kubernetes.io/docs/tasks/tools/) v1.28+
- [Helm](https://helm.sh/docs/intro/install/) v3.12+
- Docker running locally

## 1. Create the kind Cluster

```bash
cat <<'EOF' > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
EOF

kind create cluster --name redpanda --config kind-config.yaml
```

## 2. Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true \
  --wait
```

## 3. Install the Redpanda Operator with NodePool Support

```bash
helm repo add redpanda https://charts.redpanda.com --force-update

helm install redpanda-controller redpanda/operator \
  --namespace redpanda --create-namespace \
  --set crds.enabled=true \
  --set crds.experimental=true \
  --set additionalCmdFlags="{--enable-v2-nodepools}" \
  --wait
```

Key flags:
| Flag | Purpose |
|------|---------|
| `crds.experimental=true` | Installs the `NodePool` CRD alongside stable CRDs |
| `--enable-v2-nodepools` | Enables the NodePool controller in the operator |

Verify the NodePool CRD is installed:

```bash
kubectl get crd nodepools.cluster.redpanda.com
```

## 4. Deploy a Redpanda Cluster (without NodePools)

Start with a standard 3-broker cluster managed directly by the Redpanda CR:

```yaml
# redpanda-cluster.yaml
apiVersion: cluster.redpanda.com/v1alpha2
kind: Redpanda
metadata:
  name: redpanda
  namespace: redpanda
spec:
  clusterSpec:
    image:
      tag: v24.2.18
    statefulset:
      replicas: 3
      budget:
        maxUnavailable: 1
    resources:
      cpu:
        cores: "1"
      memory:
        container:
          max: 2Gi
    storage:
      persistentVolume:
        enabled: true
        size: 5Gi
```

```bash
kubectl apply -f redpanda-cluster.yaml
kubectl wait --for=condition=Ready redpanda/redpanda -n redpanda --timeout=300s
```

Verify the cluster is healthy:

```bash
kubectl exec -n redpanda redpanda-0 -c redpanda -- rpk cluster health
kubectl exec -n redpanda redpanda-0 -c redpanda -- rpk cluster info
```

Create test data to verify data integrity across migrations:

```bash
kubectl exec -n redpanda redpanda-0 -c redpanda -- rpk topic create test-topic -p 6 -r 3
kubectl exec -n redpanda redpanda-0 -c redpanda -- \
  bash -c 'for i in $(seq 1 20); do echo "msg-$i"; done | rpk topic produce test-topic'
```

## 5. Migrate to NodePools

### Step 5a: Set `statefulset.replicas` to 0

This tells the operator to stop managing the StatefulSet replica count directly, delegating broker lifecycle to NodePool CRs.

```bash
kubectl patch redpanda redpanda -n redpanda --type merge \
  -p '{"spec":{"clusterSpec":{"statefulset":{"replicas":0}}}}'
```

> **Warning:** In our testing, this immediately scales down the original StatefulSet, terminating existing brokers. Data on the original brokers will be lost if no NodePool is ready to receive partitions. Apply the NodePool promptly after this step.

### Step 5b: Create a NodePool

```yaml
# nodepool-pool1.yaml
apiVersion: cluster.redpanda.com/v1alpha2
kind: NodePool
metadata:
  name: pool1
  namespace: redpanda
spec:
  clusterRef:
    name: redpanda
  replicas: 3
```

```bash
kubectl apply -f nodepool-pool1.yaml
```

Wait for the NodePool StatefulSet to be ready:

```bash
kubectl wait --for=condition=Stable redpanda/redpanda -n redpanda --timeout=300s
```

### Step 5c: Verify Migration

```bash
# Check NodePool status
kubectl get nodepool -n redpanda

# Check pods — should see redpanda-pool1-{0,1,2}
kubectl get pods -n redpanda -l app.kubernetes.io/name=redpanda

# Check broker health
kubectl exec -n redpanda redpanda-pool1-0 -c redpanda -- rpk cluster health
kubectl exec -n redpanda redpanda-pool1-0 -c redpanda -- rpk redpanda admin brokers list
```

Expected output: 3 brokers from pool1, all active and healthy. NodePool conditions show `Bound=True`, `Deployed=True`, `Stable=True`.

> **Note:** Recreate your test topic after migration since this creates a new cluster:
> ```bash
> kubectl exec -n redpanda redpanda-pool1-0 -c redpanda -- rpk topic create test-topic -p 6 -r 3
> kubectl exec -n redpanda redpanda-pool1-0 -c redpanda -- \
>   bash -c 'for i in $(seq 1 20); do echo "msg-$i"; done | rpk topic produce test-topic'
> ```

## 6. Upgrade Redpanda via Blue/Green NodePool Migration

NodePools enable blue/green upgrades: bring up a new pool on the target version, let partitions replicate, then decommission the old pool. No in-place rolling restart needed.

### Step 6a: Create pool2 with the New Version

```yaml
# nodepool-pool2.yaml
apiVersion: cluster.redpanda.com/v1alpha2
kind: NodePool
metadata:
  name: pool2
  namespace: redpanda
spec:
  clusterRef:
    name: redpanda
  replicas: 3
  image:
    tag: v26.1.2
```

```bash
kubectl apply -f nodepool-pool2.yaml
```

Wait for both pools to be healthy (6 brokers temporarily):

```bash
kubectl wait --for=condition=Healthy redpanda/redpanda -n redpanda --timeout=300s
```

Verify 6 brokers are active:

```bash
kubectl exec -n redpanda redpanda-pool1-0 -c redpanda -- rpk redpanda admin brokers list
```

### Step 6b: Decommission pool1

Scale pool1 to 0 replicas. The operator decommissions each broker one by one, draining partitions to the pool2 brokers:

```bash
kubectl patch nodepool pool1 -n redpanda --type merge -p '{"spec":{"replicas":0}}'
```

Monitor decommission progress:

```bash
# Watch broker membership status
kubectl exec -n redpanda redpanda-pool2-0 -c redpanda -- rpk redpanda admin brokers list

# Check individual broker decommission progress
kubectl exec -n redpanda redpanda-pool2-0 -c redpanda -- \
  rpk redpanda admin brokers decommission-status <broker-id>
```

Wait for full decommission:

```bash
kubectl wait --for=condition=Stable redpanda/redpanda -n redpanda --timeout=600s
```

### Step 6c: Verify and Clean Up

```bash
# Only pool2 brokers should remain
kubectl exec -n redpanda redpanda-pool2-0 -c redpanda -- rpk redpanda admin brokers list
kubectl exec -n redpanda redpanda-pool2-0 -c redpanda -- rpk cluster health

# Verify data survived the migration
kubectl exec -n redpanda redpanda-pool2-0 -c redpanda -- rpk topic consume test-topic -n 5

# Delete the empty pool1
kubectl delete nodepool pool1 -n redpanda
```

## 7. (Optional) Remove NodePools — Return to Redpanda CR Management

To revert from NodePools back to direct Redpanda CR management:

### Step 7a: Restore Replicas on the Redpanda CR

```bash
kubectl patch redpanda redpanda -n redpanda --type merge \
  -p '{"spec":{"clusterSpec":{"statefulset":{"replicas":3}}}}'
```

This creates new Redpanda CR-managed brokers alongside the NodePool brokers (6 total temporarily).

Wait for the new brokers to join and become healthy:

```bash
kubectl wait --for=condition=Healthy redpanda/redpanda -n redpanda --timeout=300s
```

### Step 7b: Decommission the NodePool

```bash
kubectl patch nodepool pool2 -n redpanda --type merge -p '{"spec":{"replicas":0}}'
kubectl wait --for=condition=Stable redpanda/redpanda -n redpanda --timeout=600s
```

### Step 7c: Verify and Clean Up

```bash
# Only redpanda-{0,1,2} should remain
kubectl get pods -n redpanda -l app.kubernetes.io/name=redpanda
kubectl exec -n redpanda redpanda-0 -c redpanda -- rpk redpanda admin brokers list
kubectl exec -n redpanda redpanda-0 -c redpanda -- rpk cluster health

# Verify data
kubectl exec -n redpanda redpanda-0 -c redpanda -- rpk topic consume test-topic -n 5

# Delete the empty NodePool
kubectl delete nodepool pool2 -n redpanda
```

## Test Results

All tests performed on kind v0.31.0, Kubernetes v1.35.0, Redpanda Operator v26.1.1.

### Test 1: Deploy Standard Cluster

| Check | Result |
|-------|--------|
| 3 brokers healthy | PASS |
| Topic create (6 partitions, RF=3) | PASS |
| Produce 20 messages | PASS |
| Cluster health: no leaderless/under-replicated | PASS |

### Test 2: Migrate to NodePools

| Check | Result | Notes |
|-------|--------|-------|
| `replicas: 0` accepted | PASS | |
| Old StatefulSet scales to 0 | PASS | Brokers terminated — this is destructive, not a live migration |
| NodePool creates new StatefulSet | PASS | Named `redpanda-pool1` |
| pool1 pods `redpanda-pool1-{0,1,2}` running | PASS | |
| NodePool conditions (Bound, Deployed, Stable) | PASS | |
| Data from original cluster preserved | **FAIL** | Setting `replicas: 0` terminates old brokers before NodePool is ready. This is a fresh cluster, not a live migration. |
| All 3 pool1 brokers in Raft cluster | **PARTIAL** | In some runs, only 2/3 brokers join due to broker ID collision with decommissioned IDs from the old StatefulSet |

**Key finding:** The migration path (set `replicas: 0` then create NodePool) does NOT perform a live migration. The old StatefulSet is scaled down, brokers are terminated, and the NodePool creates an entirely new cluster. Pre-existing data is lost. For production use, consider creating the NodePool first and migrating data before scaling down.

### Test 3: Blue/Green Upgrade via NodePools

| Check | Result | Notes |
|-------|--------|-------|
| Create pool2 (v26.1.2) alongside pool1 | PASS | 6 brokers running simultaneously |
| All 6 brokers join Raft cluster | PASS | |
| Scale pool1 to 0 | PASS | Operator decommissions brokers one by one |
| Decommission completes | PASS | Required deleting a stuck leaderless partition in one run |
| Only pool2 brokers remain | PASS | 3 brokers, all on v26.1.2 |
| Cluster healthy after migration | PASS | |
| Data survived migration | PASS | Topic data intact and consumable |

**Key finding:** The blue/green pattern works well. The operator handles decommission automatically when scaling a pool to 0. In one test run, a leaderless partition (`kafka/upgrade-test/1`) stalled decommission progress — the sidecar readiness probe failed because the partition was leaderless, creating a circular dependency. Deleting and recreating the topic resolved the issue.

### Test 4: Remove NodePools — Return to Redpanda CR

| Check | Result | Notes |
|-------|--------|-------|
| Set `replicas: 3` on Redpanda CR | PASS | 3 new CR-managed brokers join alongside pool2 |
| 6 brokers running simultaneously | PASS | |
| Scale pool2 to 0 | PASS | Operator decommissions pool2 brokers |
| Decommission completes | PASS | |
| Only `redpanda-{0,1,2}` remain | PASS | |
| Cluster healthy | PASS | |
| Data survived reversal | PASS | |

**Key finding:** Reversal works cleanly. The Redpanda CR brokers join with new broker IDs (continuing from the highest ID in the cluster), and the NodePool brokers are decommissioned. Data is preserved throughout.

## Known Issues and Gotchas

1. **Migration is destructive, not live.** Setting `statefulset.replicas: 0` terminates old brokers immediately. The NodePool creates a fresh cluster, not an adoption of existing brokers. Plan accordingly for production migrations.

2. **NodePool image vs Redpanda CR image.** The NodePool does not inherit `spec.clusterSpec.image.tag` from the Redpanda CR. You must set `spec.image.tag` on the NodePool explicitly. If not specified, the operator uses its own default image version.

3. **Broker ID collision.** When migrating from an existing cluster, new NodePool brokers may reuse broker IDs from the recently decommissioned old brokers, causing the new broker to be immediately decommissioned. This resolves over subsequent reconciliation cycles but can temporarily result in fewer active brokers than expected.

4. **Leaderless partition stall.** During decommission, if a partition move target broker has a non-ready sidecar (due to the same leaderless partition), a circular dependency can stall progress. Workaround: delete and recreate the affected topic, or use `rpk redpanda admin brokers recommission` to cancel the decommission and retry.

5. **`OnDelete` StatefulSet update strategy.** NodePool StatefulSets use `OnDelete` update strategy. The operator manages rolling restarts directly — changing `spec.image.tag` on an existing NodePool does NOT trigger an in-place upgrade. Use the blue/green pattern (new pool) instead.

6. **Temporary 2x broker count.** Both migration (to and from NodePools) and blue/green upgrades temporarily run double the broker count. Ensure your kind cluster or production environment has sufficient resources.

## Clean Up

```bash
kubectl delete redpanda redpanda -n redpanda
kubectl delete nodepool --all -n redpanda
kubectl delete pvc --all -n redpanda
helm uninstall redpanda-controller -n redpanda
helm uninstall cert-manager -n cert-manager
kind delete cluster --name redpanda
```
