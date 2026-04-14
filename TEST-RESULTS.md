# Same-AZ Traffic Preference Test Results

**Date:** 2026-04-14
**Cluster:** kind `same-az-test` (1 CP + 3 workers)
**Istio:** 1.28.0 ambient profile

## Cluster Topology

| Node | Zone | Workloads |
|------|------|-----------|
| same-az-test-worker | us-east-1a | httpbin-zone-a, curl-client |
| same-az-test-worker2 | us-east-1b | httpbin-zone-b |
| same-az-test-worker3 | us-east-1c | httpbin-zone-c, waypoint |

## Key Configuration

Service with `spec.trafficDistribution: PreferClose`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  namespace: demo
spec:
  trafficDistribution: PreferClose
  selector:
    app: httpbin
  ports:
    - port: 80
```

## Test Results

### Test 1: Same-AZ Preference (ztunnel path, no waypoint)
- **50/50 requests -> zone-a httpbin** (same zone as client)
- **0 cross-AZ traffic**
- **PASS**

### Test 2: Failover (zone-a httpbin scaled to 0)
- **50/50 requests succeeded** (0 failures)
- Traffic split: **27 zone-c + 23 zone-b** (evenly across remaining zones)
- **PASS**

### Test 2b: Recovery (zone-a httpbin scaled back to 1)
- **50/50 requests -> zone-a** (traffic returns to local zone)
- **PASS**

### Test 3: Control (trafficDistribution removed)
- Traffic split: **37 zone-a + 27 zone-b + 36 zone-c** (~even spread)
- Confirms PreferClose is doing real work, not an artifact
- **PASS**

### Test 4a: Waypoint Path — Same-AZ Preference
- Waypoint on zone-c, client on zone-a
- **50/50 requests -> zone-c httpbin** (waypoint's local zone, NOT client's zone)
- This is expected: waypoint makes the backend selection based on **its own** locality
- **PASS**

### Test 4b: Waypoint Failover (zone-c httpbin scaled to 0)
- **50/50 requests succeeded**
- Traffic split: **25 zone-a + 25 zone-b** (even across remaining zones)
- **PASS**

## Waypoint EDS Priority Dump (for reference)

```
Cluster: inbound-vip|80|http|httpbin.demo.svc.cluster.local
  locality: {"region": "us-east-1", "zone": "us-east-1c"}  priority: 0  <- waypoint's zone
  locality: {"region": "us-east-1", "zone": "us-east-1a"}  priority: 1  <- other zones
  locality: {"region": "us-east-1", "zone": "us-east-1b"}  priority: 1
```

## ztunnel Locality Dump (what "good" looks like)

```json
{"name": "curl-client-...",      "locality": {"region": "us-east-1", "zone": "us-east-1a"}}
{"name": "httpbin-zone-a-...",   "locality": {"region": "us-east-1", "zone": "us-east-1a"}}
{"name": "httpbin-zone-b-...",   "locality": {"region": "us-east-1", "zone": "us-east-1b"}}
{"name": "httpbin-zone-c-...",   "locality": {"region": "us-east-1", "zone": "us-east-1c"}}
```

**Compare with a broken setup (locality empty):**
```json
{"locality": {}}
```

When locality is empty, PreferClose has nothing to work with and silently falls back to even distribution.

## Key Findings

1. **PreferClose works** when locality is populated — both ztunnel-only and waypoint paths.
2. **Empty locality is the #1 root cause** when PreferClose doesn't work. Nodes must have `topology.kubernetes.io/region` and `topology.kubernetes.io/zone` labels for ztunnel to pick up locality. Cloud providers (EKS, AKS, GKE) set these automatically — investigate if they're not propagating.
3. **No global PreferClose setting exists** — must be set per-Service via `spec.trafficDistribution: PreferClose` or the annotation.
4. **Waypoint uses its OWN locality** for backend selection, not the originating client's. If waypoint is in zone-c but client is in zone-a, traffic goes to zone-c backend.
5. **Failover is automatic and instant** when local zone has no healthy endpoints.

## Reproduction

See the [README](README.md) for full step-by-step instructions, or run:

```bash
kind create cluster --config kind-config.yaml
istioctl install --set profile=ambient --skip-confirmation
kubectl apply -f workload.yaml
```
