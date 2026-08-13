# EKS Scale-Readiness Review

A capacity-planning exercise: certify (or reject) an existing EKS Deployment/HPA/PDB configuration ahead of a scheduled launch, then produce a corrected patch.

## Scenario

A public, stateless API runs on EKS across 3 Availability Zones. A scheduled launch ramps traffic from 20,000 → 90,000 RPS in 2 minutes and holds peak for ~15 minutes. The configuration must:

- Meet 99.95% availability and <300ms end-to-end p99 latency
- Keep serving the traffic target after losing any one AZ
- Absorb a 10% rolling-deployment surge at peak
- Reserve 10% capacity for retries/measurement error
- Stay within regional vCPU quota, an hourly cost budget, per-AZ node caps, and subnet IP headroom
- Use only on-demand instances

## Verdict: DO NOT CERTIFY

The current config's `HPA maxReplicas: 150` is far below the ~276 pods needed under a peak-time deployment surge, node scaling is purely reactive (4min node provisioning + 60s warm-up can't fit inside a 2-minute ramp), and readiness is gated on basic `/health` rather than actual warm-up — meaning half-warmed pods would take full traffic share during the ramp.

## Approach

- Selected the safe warmed-pod throughput (600 RPS/pod) directly from the load-test table, using the point where measured p99 still clears the SLO even under worst-case ingress overhead — not the highest RPS in the table.
- Sized capacity for the AZ-loss case first (83 warmed pods/AZ, 249 total), then layered the 10% deployment surge on top as a separate, non-combined scenario, per the requirements.
- Treated warm-up as a hard constraint, not a scaling implementation detail: required capacity is pre-provisioned and pre-warmed before traffic starts, rather than relying on the HPA/Cluster Autoscaler to react during the ramp.
- Checked every quota (nodes/AZ, regional vCPU, hourly cost, subnet IPs) against the worst-case (surge) footprint, not just steady-state peak.

## Contents

- [`answer.md`](./answer.md) — full arithmetic, the certify/reject decision, the pre-launch provisioning plan, and an 8-row go/no-go launch gate.
- [`patch.yaml`](./patch.yaml) — the Kubernetes changes (HPA floor/ceiling, strict AZ spread, warm-aware readiness/liveness, surge-safe rollout strategy, tightened PDB) needed to make the numbers hold.

## Note

This is a written planning exercise — no AWS resources are deployed and no credentials/kubeconfigs are included.
