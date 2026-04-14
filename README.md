# Same-AZ Traffic Preference in Istio Ambient Mode

This guide covers how to configure Istio ambient mesh to prefer routing traffic to endpoints in the same availability zone, with automatic failover to other zones when local endpoints are unavailable. This applies to both the ztunnel (L4) and waypoint proxy (L7) data paths.

> **Validated on:** Istio 1.28 ambient profile, single-cluster with 3 AZs. See [test results](TEST-RESULTS.md) for full details.

---

## Table of Contents

1. [How It Works](#how-it-works)
2. [Prerequisites](#prerequisites)
3. [Configuration](#configuration)
4. [Verifying Locality Is Populated](#verifying-locality-is-populated)
5. [Waypoint Behavior](#waypoint-behavior)
6. [Multicluster Considerations](#multicluster-considerations)
7. [Troubleshooting](#troubleshooting)
8. [Testing It Yourself](#testing-it-yourself)

---

## How It Works

In ambient mode, ztunnel handles L4 routing between workloads. When `PreferClose` is enabled on a Service, ztunnel categorizes endpoints by locality and routes to the closest healthy ones using this priority ladder:

| Priority | Locality Match |
|----------|---------------|
| 0 (highest) | Same network, region, zone, and subzone |
| 1 | Same network, region, and zone |
| 2 | Same network and region |
| 3 | Same network |
| 4 (lowest) | Any available endpoint |

If all endpoints in a closer locality become unhealthy, traffic **automatically fails over** to the next level. When closer endpoints recover, traffic **automatically returns**.

### Two Data Paths

There are two distinct routing hops in ambient where locality matters:

```
Client Pod → [ztunnel] → Backend Pod          (ztunnel-only path, L4)
Client Pod → [ztunnel] → Waypoint → Backend   (waypoint path, L4+L7)
```

- **ztunnel path:** ztunnel on the client's node selects the backend based on the **client's locality**.
- **Waypoint path:** ztunnel routes to the waypoint, then the **waypoint selects the backend** based on **its own locality** (not the client's).

This distinction matters for cost optimization — see [Waypoint Behavior](#waypoint-behavior).

---

## Prerequisites

### 1. Node Topology Labels

Nodes **must** have standard Kubernetes topology labels. Without these, ztunnel cannot determine locality and `PreferClose` silently does nothing.

```
topology.kubernetes.io/region: us-east-1
topology.kubernetes.io/zone: us-east-1a
```

**Cloud providers set these automatically:**
- **EKS:** Set by the cloud controller manager based on the EC2 instance's AZ.
- **AKS:** Set automatically based on the VM's zone.
- **GKE:** Set automatically.
- **kind (local testing):** Must be set manually in the kind config or via `kubectl label node`.

Verify with:

```bash
kubectl get nodes -L topology.kubernetes.io/region,topology.kubernetes.io/zone
```

Expected output:

```
NAME       STATUS   ROLES    AGE   VERSION   REGION      ZONE
worker-1   Ready    <none>   10m   v1.32.0   us-east-1   us-east-1a
worker-2   Ready    <none>   10m   v1.32.0   us-east-1   us-east-1b
worker-3   Ready    <none>   10m   v1.32.0   us-east-1   us-east-1c
```

### 2. Istio Ambient Mode Installed

Istio must be installed with the `ambient` profile. ztunnel and CNI must be running on all worker nodes.

```bash
istioctl install --set profile=ambient --skip-confirmation
```

### 3. Namespace Enrolled in Ambient

The application namespace must have the ambient dataplane label:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    istio.io/dataplane-mode: ambient
```

---

## Configuration

### Option A: `spec.trafficDistribution` Field (Recommended)

Available on Kubernetes 1.31+ (stable). Set directly on the Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: my-app
spec:
  trafficDistribution: PreferClose    # ← This is all you need
  selector:
    app: my-app
  ports:
    - port: 80
```

### Option B: Annotation (Older Kubernetes or ServiceEntry)

For Kubernetes clusters that don't have the `trafficDistribution` field, or for `ServiceEntry` resources:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: my-app
  annotations:
    networking.istio.io/traffic-distribution: PreferClose
spec:
  selector:
    app: my-app
  ports:
    - port: 80
```

### Available Values

| Value | Behavior |
|-------|----------|
| `PreferClose` | Prefer same zone, fall back through region → network → any |
| `PreferSameZone` | Same as PreferClose but Istio-specific; prefer same zone with full fallback chain |
| `PreferSameNode` | Prefer endpoints on the exact same node |
| `PreferNetwork` | Prefer same network (useful for multicluster) |
| *(not set)* | Default — ztunnel treats all endpoints equally, no locality preference |

### Namespace-Level Default

You can set a default for all services in a namespace:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  annotations:
    networking.istio.io/traffic-distribution: PreferClose
```

Individual services can override with their own annotation.

### No Global Setting

There is currently **no global mesh-wide setting** to enable PreferClose for all services. It must be configured per-Service or per-Namespace.

---

## Verifying Locality Is Populated

This is the most common failure point. If ztunnel doesn't see locality for workloads, PreferClose has no effect.

### Check ztunnel's View of Workloads

```bash
# Pick any ztunnel pod
ZTUNNEL=$(kubectl -n istio-system get pod -l app=ztunnel -o jsonpath='{.items[0].metadata.name}')

# Dump workloads and check locality
istioctl ztunnel-config workloads $ZTUNNEL -ojson | \
  jq '.[] | select(.namespace=="my-app") | {name, node, locality}'
```

**Healthy output (locality populated):**

```json
{
  "name": "my-app-abc123",
  "node": "ip-10-0-1-50.ec2.internal",
  "locality": {
    "region": "us-east-1",
    "zone": "us-east-1a"
  }
}
```

**Broken output (locality empty):**

```json
{
  "name": "my-app-abc123",
  "node": "my-node",
  "locality": {}
}
```

If locality is empty, PreferClose will be ignored and traffic will spread evenly. See [Troubleshooting](#troubleshooting).

### Check the Service Has trafficDistribution Set

```bash
kubectl get svc my-service -n my-app -o jsonpath='{.spec.trafficDistribution}'
# Should print: PreferClose
```

Or for annotation-based:

```bash
kubectl get svc my-service -n my-app -o jsonpath='{.metadata.annotations.networking\.istio\.io/traffic-distribution}'
```

---

## Waypoint Behavior

When a waypoint proxy is in the traffic path, there are **two locality-aware hops**:

### Hop 1: Client ztunnel → Waypoint

ztunnel routes the client's request to a waypoint. The waypoint itself is a pod, and ztunnel applies `PreferClose` to the waypoint selection — so ztunnel prefers the waypoint in the **client's zone**.

> By default, `trafficDistribution: PreferClose` is automatically applied to waypoint traffic. You don't need to configure this.

### Hop 2: Waypoint → Backend

The waypoint (Envoy) selects the backend pod. It uses **its own locality** to determine priority:

```
Priority 0: backends in the waypoint's zone
Priority 1: backends in the waypoint's region (different zone)
Priority 2: everything else
```

### Implication for Cost

If you have 3 AZs and a single waypoint replica in zone-c:
- A client in zone-a sends traffic cross-AZ to the waypoint in zone-c (hop 1).
- The waypoint then selects the zone-c backend (hop 2, same-AZ).
- **Net result:** 1 cross-AZ hop instead of 0.

**To minimize cross-AZ traffic with waypoints**, ensure waypoint replicas are spread across all AZs using `topologySpreadConstraints`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-waypoint
  namespace: my-app
  annotations:
    # Spread waypoint replicas across zones
    gateway.istio.io/deployment-topology-spread-constraints: |
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
spec:
  gatewayClassName: istio-waypoint
  listeners:
    - name: mesh
      port: 15008
      protocol: HBONE
```

With a waypoint replica per zone, the client's ztunnel picks the same-zone waypoint, and that waypoint picks the same-zone backend — zero cross-AZ hops.

### Inspecting Waypoint EDS Priorities

Port-forward to the waypoint and dump EDS to see how it ranks endpoints:

```bash
WAYPOINT=$(kubectl -n my-app get pod -l gateway.networking.k8s.io/gateway-name=my-waypoint -o jsonpath='{.items[0].metadata.name}')

kubectl -n my-app port-forward $WAYPOINT 15000:15000 &
curl -s 'http://localhost:15000/config_dump?include_eds' | \
  jq '.configs[] | select(.dynamic_endpoint_configs) |
      .dynamic_endpoint_configs[] |
      select(.endpoint_config.cluster_name | contains("my-service")) |
      .endpoint_config.endpoints[] |
      {locality, priority, endpoints: [.lb_endpoints[].endpoint.address]}'
```

You should see priority 0 for the waypoint's own zone and priority 1+ for other zones.

---

## Multicluster Considerations

In a multicluster ambient mesh, traffic distribution involves additional components:

### East-West Gateway Labels

The east-west gateway **must** have topology labels for remote endpoints to receive correct locality information:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: istio-eastwest
  namespace: istio-eastwest
  labels:
    topology.istio.io/cluster: cluster-1
    topology.istio.io/network: cluster-1-network
    topology.kubernetes.io/region: us-east-1          # ← Critical for locality
spec:
  gatewayClassName: istio-eastwest
  listeners:
    - name: cross-network
      port: 15008
      protocol: HBONE
      tls:
        mode: Passthrough
```

Without `topology.kubernetes.io/region` on the east-west gateway, remote `WorkloadEntry` objects lose their locality values and all remote endpoints get the same priority — a common root cause when multicluster locality routing doesn't work.

### DestinationRule Failover (Cross-Region)

For multicluster with waypoints, you can define explicit region failover order:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: my-service-failover
  namespace: my-app
spec:
  host: my-service.my-app.svc.cluster.local
  trafficPolicy:
    outlierDetection:
      baseEjectionTime: 30s
      consecutiveGatewayErrors: 3
      consecutive5xxErrors: 10
      interval: 10s
      maxEjectionPercent: 100
    loadBalancer:
      localityLbSetting:
        enabled: true
        failover:
          - from: us-east-1
            to: us-east-2
          - from: us-east-2
            to: us-east-1
```

> `outlierDetection` is **required** alongside `localityLbSetting` — it's what triggers endpoint ejection and drives the failover.

> DestinationRule failover is evaluated based on the **waypoint's locality**, not the client's. The `from:` field must match the region where the waypoint is running.

---

## Troubleshooting

### Problem: Traffic spreads evenly despite PreferClose

**Check 1:** Is locality populated?

```bash
ZTUNNEL=$(kubectl -n istio-system get pod -l app=ztunnel -o jsonpath='{.items[0].metadata.name}')
istioctl ztunnel-config workloads $ZTUNNEL -ojson | \
  jq '.[] | select(.namespace=="my-app") | {name, locality}'
```

If `locality: {}`, fix the node labels:

```bash
# Verify nodes have topology labels
kubectl get nodes -L topology.kubernetes.io/region,topology.kubernetes.io/zone

# If missing, add them (shouldn't be needed on EKS/AKS/GKE)
kubectl label node <node-name> topology.kubernetes.io/region=us-east-1
kubectl label node <node-name> topology.kubernetes.io/zone=us-east-1a
```

After labeling, restart ztunnel to pick up the change:

```bash
kubectl -n istio-system rollout restart daemonset/ztunnel
```

**Check 2:** Is `trafficDistribution` set on the Service?

```bash
kubectl get svc my-service -n my-app -o yaml | grep -A1 trafficDistribution
```

**Check 3:** Are you using an annotation when you meant the spec field (or vice versa)?

The `spec.trafficDistribution` field uses `PreferClose`. The annotation `networking.istio.io/traffic-distribution` also accepts `PreferSameZone`, `PreferSameNode`, and `PreferNetwork`. Make sure you're using the right mechanism for your Kubernetes version.

**Check 4:** Is the annotation on the Service as an annotation, not a label?

A common mistake is putting `networking.istio.io/traffic-distribution` as a **label** instead of an **annotation**. Labels are ignored — it must be an annotation.

### Problem: Multicluster traffic ignores locality

**Check 1:** East-west gateway has `topology.kubernetes.io/region` label.

**Check 2:** Remote peer gateway objects have correct region labels.

**Check 3:** `istio-system` namespace has `topology.istio.io/network` label matching the cluster.

**Check 4:** Trust domain is configured correctly (cluster-specific recommended for multicluster).

### Problem: Waypoint routes to wrong zone

The waypoint selects backends based on **its own** locality. If the waypoint is in zone-c, it will prefer zone-c backends regardless of where the client is.

**Fix:** Scale waypoint replicas across AZs with topology spread constraints so each zone has a local waypoint.

---

## Testing It Yourself

You can reproduce the full test locally with kind in under 5 minutes. All manifests are included in this repo.

### 1. Create a kind Cluster with AZ Labels

```bash
kind create cluster --config kind-config.yaml
```

### 2. Install Istio Ambient

```bash
istioctl install --set profile=ambient --skip-confirmation
```

### 3. Deploy Test Workload

```bash
kubectl apply -f workload.yaml
kubectl -n demo wait --for=condition=ready pod -l app=httpbin --timeout=120s
kubectl -n demo wait --for=condition=ready pod -l app=curl-client --timeout=120s
```

### 4. Verify Locality

```bash
ZTUNNEL=$(kubectl -n istio-system get pod -l app=ztunnel -o jsonpath='{.items[0].metadata.name}')
istioctl ztunnel-config workloads $ZTUNNEL -ojson | \
  jq '.[] | select(.namespace=="demo") | {name, node, locality}'
```

### 5. Test Same-AZ Preference

```bash
# Send 50 requests
kubectl -n demo exec deploy/curl-client -- sh -c \
  'for i in $(seq 1 50); do curl -s http://httpbin.demo.svc.cluster.local/get >/dev/null; done'

# Check ztunnel logs for destination
ZTUNNEL_CLIENT_NODE=$(kubectl -n istio-system get pods -l app=ztunnel \
  --field-selector spec.nodeName=$(kubectl -n demo get pod -l app=curl-client \
  -o jsonpath='{.items[0].spec.nodeName}') -o jsonpath='{.items[0].metadata.name}')

kubectl -n istio-system logs $ZTUNNEL_CLIENT_NODE --since=30s | \
  grep outbound | grep httpbin | grep -o 'dst.workload="[^"]*"' | sort | uniq -c
```

Expected: all requests go to the zone-a httpbin pod.

### 6. Test Failover

```bash
# Kill zone-a backend
kubectl -n demo scale deploy httpbin-zone-a --replicas=0
sleep 5

# Send requests again
kubectl -n demo exec deploy/curl-client -- sh -c \
  'for i in $(seq 1 50); do curl -s http://httpbin.demo.svc.cluster.local/get >/dev/null; done'

# Check distribution
kubectl -n istio-system logs $ZTUNNEL_CLIENT_NODE --since=15s | \
  grep outbound | grep httpbin | grep -o 'dst.workload="[^"]*"' | sort | uniq -c
```

Expected: traffic splits across zone-b and zone-c.

### 7. Test Control (Remove PreferClose)

```bash
kubectl -n demo patch svc httpbin --type='json' \
  -p='[{"op":"remove","path":"/spec/trafficDistribution"}]'
sleep 5

kubectl -n demo exec deploy/curl-client -- sh -c \
  'for i in $(seq 1 100); do curl -s http://httpbin.demo.svc.cluster.local/get >/dev/null; done'

kubectl -n istio-system logs $ZTUNNEL_CLIENT_NODE --since=20s | \
  grep outbound | grep httpbin | grep -o 'dst.workload="[^"]*"' | sort | uniq -c
```

Expected: roughly even split across all 3 zones (~33% each).

### 8. Cleanup

```bash
kind delete cluster --name same-az-test
```

---

## References

- [Istio Ambient Traffic Distribution](https://istio.io/latest/docs/ambient/usage/traffic-distribution/)
- [Ambient Mesh Load Balancing](https://ambientmesh.io/docs/traffic/load-balancing/)
- [Kubernetes Topology Aware Routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/)
- [Istio Multicluster Failover (Ambient)](https://istio.io/latest/docs/ambient/install/multicluster/failover/)
