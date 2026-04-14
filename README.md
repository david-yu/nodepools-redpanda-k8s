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
# nodepool-blue.yaml
apiVersion: cluster.redpanda.com/v1alpha2
kind: NodePool
metadata:
  name: blue
  namespace: redpanda
spec:
  clusterRef:
    name: redpanda
  replicas: 3
```

```bash
kubectl apply -f nodepool-blue.yaml
```

Wait for the NodePool StatefulSet to be ready:

```bash
kubectl wait --for=condition=Stable redpanda/redpanda -n redpanda --timeout=300s
```

### Step 5c: Verify Migration

```bash
# Check NodePool status
kubectl get nodepool -n redpanda

# Check pods — should see redpanda-blue-{0,1,2}
kubectl get pods -n redpanda -l app.kubernetes.io/name=redpanda

# Check broker health
kubectl exec -n redpanda redpanda-blue-0 -c redpanda -- rpk cluster health
kubectl exec -n redpanda redpanda-blue-0 -c redpanda -- rpk redpanda admin brokers list
```

Expected output: 3 brokers from the blue pool, all active and healthy. NodePool conditions show `Bound=True`, `Deployed=True`, `Stable=True`.

> **Note:** Recreate your test topic after migration since this creates a new cluster:
> ```bash
> kubectl exec -n redpanda redpanda-blue-0 -c redpanda -- rpk topic create test-topic -p 6 -r 3
> kubectl exec -n redpanda redpanda-blue-0 -c redpanda -- \
>   bash -c 'for i in $(seq 1 50); do echo "msg-$i"; done | rpk topic produce test-topic'
> ```

## 6. Upgrade Redpanda via Blue/Green NodePool Migration

NodePools enable blue/green upgrades: bring up a new pool (green) on the target version, wait for all brokers to be healthy, then delete the old pool (blue). The operator decommissions the old brokers one by one automatically, draining partitions to the green brokers. No in-place rolling restart needed.

### Step 6a: Create the Green Pool with the New Version

```yaml
# nodepool-green.yaml
apiVersion: cluster.redpanda.com/v1alpha2
kind: NodePool
metadata:
  name: green
  namespace: redpanda
spec:
  clusterRef:
    name: redpanda
  replicas: 3
  image:
    tag: v26.1.2
```

```bash
kubectl apply -f nodepool-green.yaml
```

Wait for both pools to be healthy (6 brokers temporarily):

```bash
kubectl wait --for=condition=Healthy redpanda/redpanda -n redpanda --timeout=300s
```

Verify all 6 brokers are active:

```bash
kubectl exec -n redpanda redpanda-blue-0 -c redpanda -- rpk redpanda admin brokers list
kubectl exec -n redpanda redpanda-blue-0 -c redpanda -- rpk cluster health
```

### Step 6b: Delete the Blue Pool

Delete the old NodePool. The operator decommissions each blue broker one at a time internally, draining partitions to the green brokers before removing each one:

```bash
kubectl delete nodepool blue -n redpanda
```

Wait for the decommission to complete:

```bash
kubectl wait --for=condition=Stable redpanda/redpanda -n redpanda --timeout=600s
```

### Step 6c: Verify

```bash
# Only green brokers should remain
kubectl exec -n redpanda redpanda-green-0 -c redpanda -- rpk redpanda admin brokers list
kubectl exec -n redpanda redpanda-green-0 -c redpanda -- rpk cluster health

# Verify data survived the migration
kubectl exec -n redpanda redpanda-green-0 -c redpanda -- rpk topic consume test-topic -n 5
```

## 7. Using nodeSelector and Tolerations

In production, each NodePool typically targets a dedicated set of Kubernetes nodes using `nodeSelector` and `tolerations`. This ensures broker pods land on nodes with the right hardware, and that non-Redpanda workloads don't get scheduled onto those nodes.

### Label and Taint Your Nodes

First, label and taint the nodes designated for each pool:

```bash
# Label nodes for the blue pool
kubectl label node <node-1> nodetype=redpanda-blue
kubectl label node <node-2> nodetype=redpanda-blue
kubectl label node <node-3> nodetype=redpanda-blue

# Taint them so only Redpanda pods land there
kubectl taint node <node-1> redpanda-blue=true:NoSchedule
kubectl taint node <node-2> redpanda-blue=true:NoSchedule
kubectl taint node <node-3> redpanda-blue=true:NoSchedule
```

For a kind cluster, label the worker nodes:

```bash
kubectl label node redpanda-worker  nodetype=redpanda-blue
kubectl label node redpanda-worker2 nodetype=redpanda-blue
kubectl label node redpanda-worker3 nodetype=redpanda-blue
```

> **Note:** kind worker nodes don't need taints since there are no competing workloads.

### NodePool with nodeSelector and Tolerations

```yaml
apiVersion: cluster.redpanda.com/v1alpha2
kind: NodePool
metadata:
  name: blue
  namespace: redpanda
spec:
  clusterRef:
    name: redpanda
  replicas: 3
  podTemplate:
    spec:
      nodeSelector:
        nodetype: redpanda-blue
      tolerations:
        - key: redpanda-blue
          value: "true"
          effect: NoSchedule
```

### Blue/Green with Different Node Groups

For blue/green upgrades on dedicated hardware, label a second set of nodes for the green pool:

```bash
# Label and taint nodes for the green pool
kubectl label node <node-4> nodetype=redpanda-green
kubectl label node <node-5> nodetype=redpanda-green
kubectl label node <node-6> nodetype=redpanda-green
kubectl taint node <node-4> redpanda-green=true:NoSchedule
kubectl taint node <node-5> redpanda-green=true:NoSchedule
kubectl taint node <node-6> redpanda-green=true:NoSchedule
```

```yaml
apiVersion: cluster.redpanda.com/v1alpha2
kind: NodePool
metadata:
  name: green
  namespace: redpanda
spec:
  clusterRef:
    name: redpanda
  replicas: 3
  image:
    tag: v26.1.2
  podTemplate:
    spec:
      nodeSelector:
        nodetype: redpanda-green
      tolerations:
        - key: redpanda-green
          value: "true"
          effect: NoSchedule
```

This guarantees blue and green brokers run on entirely separate nodes, which is the intended pattern for hardware migrations and Kubernetes node upgrades.

## 8. (Optional) Remove NodePools — Return to Redpanda CR Management

To revert from NodePools back to direct Redpanda CR management:

### Step 8a: Restore Replicas on the Redpanda CR

```bash
kubectl patch redpanda redpanda -n redpanda --type merge \
  -p '{"spec":{"clusterSpec":{"statefulset":{"replicas":3}}}}'
```

This creates new Redpanda CR-managed brokers alongside the NodePool brokers (6 total temporarily).

Wait for the new brokers to join and become healthy:

```bash
kubectl wait --for=condition=Healthy redpanda/redpanda -n redpanda --timeout=300s
```

### Step 8b: Delete the NodePool

```bash
kubectl delete nodepool green -n redpanda
kubectl wait --for=condition=Stable redpanda/redpanda -n redpanda --timeout=600s
```

### Step 8c: Verify and Clean Up

```bash
# Only redpanda-{0,1,2} should remain
kubectl get pods -n redpanda -l app.kubernetes.io/name=redpanda
kubectl exec -n redpanda redpanda-0 -c redpanda -- rpk redpanda admin brokers list
kubectl exec -n redpanda redpanda-0 -c redpanda -- rpk cluster health

# Verify data
kubectl exec -n redpanda redpanda-0 -c redpanda -- rpk topic consume test-topic -n 5

# Delete the empty NodePool
kubectl delete nodepool green -n redpanda
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
| Old StatefulSet scales to 0 | PASS | Operator stops managing the default StatefulSet |
| NodePool creates new StatefulSet | PASS | Named `redpanda-blue` |
| Blue pool pods `redpanda-blue-{0,1,2}` running | PASS | |
| NodePool conditions (Bound, Deployed, Stable) | PASS | |
| All 3 blue pool brokers in Raft cluster | PASS | All 3 brokers active and healthy |

**Note:** When deploying with `replicas: 0` from the start (as shown in this guide), there is no pre-existing data to migrate. The NodePool creates the only brokers in the cluster. If migrating an existing cluster that was previously running with `replicas > 0`, setting `replicas: 0` will terminate the old brokers. In that scenario, create the NodePool first and allow partitions to replicate before scaling down the old StatefulSet.

### Test 3: Blue/Green Upgrade via NodePools (delete old NodePool)

| Check | Result | Notes |
|-------|--------|-------|
| Create green pool (v26.1.2) alongside blue pool | PASS | 6 brokers running simultaneously |
| All 6 brokers join Raft cluster | PASS | Zero leaderless or under-replicated partitions |
| Delete blue NodePool | PASS | Operator decommissions blue brokers one by one internally |
| Decommission completes without intervention | PASS | No leaderless partitions, no stalls |
| Only green brokers remain | PASS | 3 brokers, all on v26.1.2 |
| Cluster healthy throughout | PASS | |
| Data (50 messages, RF=3) survived | PASS | All messages consumable after migration |

**Key finding:** Deleting the old NodePool is the cleanest upgrade path. The operator handles the decommission internally — it drains partitions from each blue broker to the green brokers one at a time before removing it. No leaderless partitions occurred, no manual intervention needed, data fully preserved.

### Test 4: Remove NodePools — Return to Redpanda CR

| Check | Result | Notes |
|-------|--------|-------|
| Set `replicas: 3` on Redpanda CR | PASS | 3 new CR-managed brokers join alongside green pool |
| 6 brokers running simultaneously | PASS | |
| Delete green NodePool | PASS | Operator decommissions green brokers |
| Decommission completes | PASS | |
| Only `redpanda-{0,1,2}` remain | PASS | |
| Cluster healthy | PASS | |
| Data survived reversal | PASS | |

**Key finding:** Reversal works cleanly. The Redpanda CR brokers join with new broker IDs (continuing from the highest ID in the cluster), and the NodePool brokers are decommissioned. Data is preserved throughout.

## Known Issues and Gotchas

1. **Delete the old NodePool for blue/green upgrades.** The cleanest decommission path is `kubectl delete nodepool <old-pool>`. The operator drains partitions from each broker one at a time internally before removing it. This worked without leaderless partitions or manual intervention in testing.

2. **Migration from Redpanda CR is destructive, not live.** Setting `statefulset.replicas: 0` on a cluster that was previously running with `replicas > 0` terminates old brokers. If you have an existing cluster, create the NodePool first and let partitions replicate before scaling down.

3. **NodePool image vs Redpanda CR image.** The NodePool does not inherit `spec.clusterSpec.image.tag` from the Redpanda CR. You must set `spec.image.tag` on the NodePool explicitly. If not specified, the operator uses its own default image version.

4. **`OnDelete` StatefulSet update strategy.** NodePool StatefulSets use `OnDelete` update strategy. The operator manages rolling restarts directly — changing `spec.image.tag` on an existing NodePool does NOT trigger an in-place upgrade. Use the blue/green pattern (new pool) instead.

5. **Temporary 2x broker count.** Both migration (to and from NodePools) and blue/green upgrades temporarily run double the broker count. Ensure your kind cluster or production environment has sufficient resources.

6. **Use nodeSelector and tolerations in production.** Without scheduling constraints, broker pods may land on the same nodes as other workloads or as brokers from the other pool. See [Section 7](#7-using-nodeselector-and-tolerations) for examples.

## Clean Up

```bash
kubectl delete redpanda redpanda -n redpanda
kubectl delete nodepool --all -n redpanda
kubectl delete pvc --all -n redpanda
helm uninstall redpanda-controller -n redpanda
helm uninstall cert-manager -n cert-manager
kind delete cluster --name redpanda
```
