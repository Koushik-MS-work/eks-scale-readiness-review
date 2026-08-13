# EKS Scale-Readiness Review — answer.md

## 1. Maximum safe warmed-pod RPS

| RPS | App p99 | +max ingress (60ms) | vs 300ms SLO |
|---|---|---|---|
| 400 | 105ms | 165ms | pass |
| 600 | 210ms | **270ms** | **pass (30ms margin)** |
| 700 | 335ms | 395ms | **fail (already fails before ingress)** |
| 850 | 780ms | — | fail |

**Selected: 600 RPS/pod.** At 700 RPS the application's own p99 (335ms) already exceeds the 300ms end-to-end SLO before adding ingress overhead, so 700 and 850 are disqualified outright. 600 RPS clears the SLO even under worst-case 60ms ingress overhead, with a 30ms margin, and its error rate (0.02%) is negligible. 400 RPS is safe but wastes capacity/cost for no benefit.

## 2. Required warmed pods after one AZ loss

- Peak external traffic: 90,000 RPS
- Required reserve (retries + measurement error): +10% → **99,000 RPS** design target
- Pods needed to serve target from the 2 surviving AZs: `ceil(99,000 / 600) = 165 pods`
- To guarantee 165 pods are available from *any* 2 of 3 AZs, distribute evenly:
  `ceil(165 / 2) = 83 pods/AZ` → **249 pods total** (any 2 AZs = 166 ≥ 165 ✓)

## 3. Required pre-launch pods per AZ

**83 pods/AZ, fully warmed (passing `/ready-warm`), before traffic starts.**
The 2-minute ramp is shorter than node provisioning (4 min p95) + warm-up (60s) = ~5 min worst case. Reactive scaling cannot land in time, so the AZ-loss-safe steady-state count (83/AZ) must already exist and be warm at T-0, not triggered by the ramp itself.

## 4. Nodes per AZ

- Allocatable CPU/node = 7.5 − 0.75 (DaemonSet) = 6.75 cores
- Pods/node (CPU-bound, 500m request) = `floor(6.75 / 0.5) = 13` (maxPods=58 not binding: 13+1 DaemonSet = 14)
- **Steady peak:** `ceil(83 / 13) = 7 nodes/AZ` → 21 nodes total
- **10% deployment surge** (applied to the 249-pod steady baseline, per the brief — not combined with AZ loss):
  surge pods = `ceil(249 × 0.10) = 25` → ~9/AZ → 83+9 = **92 pods/AZ**
  surge nodes = `ceil(92 / 13) = 8 nodes/AZ` → 24 nodes total — **exactly at the 8-node/AZ cap**

## 5. Incremental IP use per AZ (current 4 nodes/16 pods → surge state 8 nodes/92 pods)

Incremental = (8−4 nodes) + (92−16 pods) = 4 + 76 = **80 IPs/AZ**

| AZ | Free now | Min must remain | Usable budget | Need | Margin |
|---|---|---|---|---|---|
| 1a | 300 | 30 | 270 | 80 | 190 ✓ |
| 1b | 280 | 30 | 250 | 80 | 170 ✓ |
| 1c | 120 | 30 | 90 | 80 | **10 — tight, flag as launch risk** |

## 6. Regional vCPU and peak hourly cost (worst case = surge state)

- vCPU: `24 nodes × 8 vCPU = 192 vCPU` vs 256 quota → **192 ≤ 256 ✓** (64 vCPU / 8-node headroom)
- Cost: `24 × $0.38/hr = $9.12/hr` vs $10/hr budget → **✓** ($0.88 headroom, <3 nodes worth)

## 7. Constraint checklist — current configuration

| Constraint | Result |
|---|---|
| HPA capacity (`maxReplicas: 150`) vs required ~276 pods (surge) | **FAIL** — cap is only 54% of what's needed |
| Reactive-only node scaling vs 2-min ramp | **FAIL** — 4min node + 60s warm-up = ~5min, cannot land inside the ramp |
| Readiness gate (`/health`, 3s delay) vs true warm-up (60s) | **FAIL** — pods take live traffic while only able to serve 300 RPS, causing latency/error spikes |
| `topologySpreadConstraints` (maxSkew 2, `ScheduleAnyway`) vs even 83/AZ requirement | **FAIL** — doesn't guarantee the split the AZ-loss math depends on |
| Subnet IPs | Pass, but 1c has only 10-IP margin |
| Regional vCPU / cost | Pass (for the *redesigned* capacity — current config never reaches it) |
| Latency/error SLO at 600 RPS/pod | Pass (as a benchmark fact, independent of current config) |

### Verdict: **DO NOT CERTIFY**

The current Deployment/HPA/PDB cannot reach or sustain the required 249–276 warmed pods before or during the ramp: `maxReplicas` is capped below requirement, scaling is purely reactive (too slow for a 2-minute ramp), and readiness is not gated on actual warm-up.

## Forcing capacity to exist before traffic starts

Do not rely on the HPA reacting to the ramp. Instead:

1. **T-15 min:** Pre-scale via a scheduled action (CronJob-triggered `kubectl scale`/HPA patch, or a KEDA `CronScaledObject`) that raises `HorizontalPodAutoscaler.minReplicas` to 249. This precedes the worst-case 4min node-provision + 60s warm-up + buffer.
2. **T-10 min (gate):** Confirm all 249 pods report `/ready-warm` = success and are spread ~83/AZ. This is the first go/no-go check (see Task 3 table).
3. **T-0:** Traffic starts only after the gate passes. The HPA's `minReplicas` floor prevents it from scaling back down mid-event; CPU-based scale-up beyond 249 remains available as extra headroom, not as the primary mechanism.
4. For a deployment during peak, surge capacity (92/AZ, 24 nodes) is likewise pre-validated before the rollout is allowed to start — the rollout does not lean on reactive node scale-out either.

## Resource request/limit

**Unchanged** (`requests: cpu 500m / limits: cpu 1`). At the selected operating point (600 RPS/pod), measured CPU usage is 0.48 cores — the 500m request sits just above measured usage with a small buffer, and the 1-core limit gives headroom without inviting noisy-neighbor CPU throttling. No measurement supports raising or lowering it.

## Task 3: Launch gate

| Measurement | Pass condition | When checked | Action if fail |
|---|---|---|---|
| Warmed pods/AZ | ≥83 pods/AZ passing `/ready-warm` | T-10 min (pre-launch freeze) | Hold launch; force-scale and re-check |
| Nodes/AZ | ≥7 Ready (≥8 if a rollout is planned) per AZ | T-10 min | Manual node scale-up; hold launch |
| Customer p99 latency | <300ms, rolling 1-min window | Continuous, ramp + peak | Throttle/shed traffic, page on-call |
| Error rate | <0.1%, rolling 1-min window | Continuous, ramp + peak | Page on-call; consider halting ramp |
| Per-AZ placement skew | No AZ <80 warmed pods | T-5 min and continuous | Manual pod rebalance/reschedule |
| Subnet free IPs | ≥30 remaining in every AZ | T-10 min | Halt further scale-out in that AZ; escalate to network team |
| HPA headroom | Current replicas ≤90% of `maxReplicas` | Continuous, peak window | Raise `maxReplicas` or add node capacity; page on-call |
| Regional vCPU / hourly cost | ≤256 vCPU and ≤$10/hr | T-10 min and continuous | Freeze scale-out; escalate for a quota/budget exception |

## Assumptions

- IP consumption model treats each pod and each node as consuming exactly one free IP (as stated), regardless of ENI/prefix-delegation mechanics.
- vCPU quota is counted against full instance vCPU (8/node), not allocatable CPU, since node quota is an infrastructure-level limit.
- The 10% deployment surge is applied to the AZ-loss-safe steady baseline (249 pods) rather than to the bare 90,000 RPS target, per the instruction to treat surge and AZ-loss as separate, non-combined scenarios.
- "Traffic target" for AZ-loss purposes includes the 10% retry/measurement reserve (99,000 RPS), since that reserve is a standing requirement, not an event-specific one.
