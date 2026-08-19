# Scale-Readiness Review: 90,000 RPS API Cluster Sizing
## Task 1: The Capacity & Network Infrastructure Audit

**Final Verdict: DO NOT CERTIFY**

### 1. Architectural Reasoning & Mathematical Walkthrough

#### A. Single Pod Performance Sweet-Spot Sizing
The Service Level Objective (SLO) mandates an end-to-end p99 latency below **300 ms**. Network routing infrastructure introduces a fixed tax of **35 ms to 60 ms** of latency overhead. To calculate our maximum allowable application-layer processing budget, we subtract the worst-case network overhead:
$$\text{300 ms (Max Budget)} - \text{60 ms (Max Network Overheard)} = \mathbf{240\text{ ms Max App Processing Budget}}$$

Evaluating the load-test benchmark table:
* **400 RPS:** 105 ms p99 latency / 0.00% error rate / 0.33 CPU cores.
* **600 RPS:** 210 ms p99 latency / 0.02% error rate / 0.48 CPU cores.
* **700 RPS:** 335 ms p99 latency / 0.15% error rate / 0.62 CPU cores.

The **600 RPS per pod** tier is selected as our core engineering anchor. It delivers a 210 ms application latency (safely below our 240 ms limit), keeps the error rate practically at zero (0.02%), and consumes **0.48 CPU cores** per pod. Pushing to 700 RPS is discarded as it immediately breaches our latency SLO.

#### B. Total Target Traffic Calculation with Compounding Buffers
The business requirements define a scheduled peak workload of 90,000 RPS. We must compound two independent 10% safety margins:
1. **Measurement Error and Retry Storm Buffer (+10%):** Protects against tracking error and immediate client-side retries.
   $$90,000\text{ RPS} \times 1.10 = \mathbf{99,000\text{ RPS (Traffic Demand Target)}}$$
2. **Live Rolling Deployment Surge Layer (+10%):** Handles the physical pod expansion required to transition containers during an active peak code roll out.

#### C. Multi-AZ Blast Radius Sizing (N+1 Failure Model)
To ensure compliance with the strict N+1 Availability Zone failure rule, the remaining **2 surviving AZs** must absorb 100% of our 99,000 RPS traffic demand target if any single zone drops completely offline.
* **Traffic Demand Per Surviving Zone:** $99,000\text{ RPS} \div 2\text{ AZs} = 49,500\text{ RPS per zone}$
* **Required Active Pods Per Zone:** $49,500\text{ RPS} \div 600\text{ RPS per pod} = 82.5 \rightarrow \mathbf{83\text{ pods per AZ}}$

To prevent catastrophic latency degradation and a 2.40% error wave during the 5-minute node boot/warm-up delay of a reactive autoscaling sequence, **all 3 Availability Zones must be symmetrically pre-provisioned with 83 pods each before the event launches**.
* **Steady-State Baseline Pod Floor:** $83\text{ pods per zone} \times 3\text{ AZs} = \mathbf{249\text{ total active pods}}$

#### D. Sizing the Peak Deployment Surge Footprint
If an urgent code update is deployed during peak traffic, Kubernetes will spin up 10% extra pods across the cluster to manage the transition:
* **Peak Deployment Surge Footprint Ceiling:** $249\text{ base pods} \times 1.10 = 273.9 \rightarrow \mathbf{276\text{ pods total across the cluster}}$
* **Peak Pod Count Per Zone During Surge:** $276\text{ pods} \div 3\text{ AZs} = \mathbf{92\text{ pods per AZ}}$

---

### 2. Physical Infrastructure Constraints & Audit Checklist

To translate these pod numbers into virtual hardware servers, we evaluate the packing density on our locked `c7i.2xlarge` instances (8 vCPUs total).
* **Usable Compute Resources:** Background system tooling and DaemonSets consume a fixed overhead, leaving exactly **7.5 usable CPU cores per node** for application workloads.
* **Max Pod Packing Density Per Server Node:** 
  $$7.5\text{ usable cores} \div 0.48\text{ cores per pod} = 15.625 \rightarrow \mathbf{15\text{ pods max per node}}$$

#### Compliance Verification Gateways:

1. **The Regional Auto-Scaling Ceiling Check:**
   * *Required Nodes Per Zone (Steady-State):* $83\text{ pods} \div 15\text{ pods/node} = 5.53 \rightarrow \mathbf{6\text{ nodes per AZ}}$
   * *Required Nodes Per Zone (Peak Deployment Surge):* $92\text{ pods} \div 15\text{ pods/node} = 6.13 \rightarrow \mathbf{7\text{ nodes per AZ}}$
   * *Hard Constraint Limit:* Max 8 nodes per AZ.
   * *Status:* **✅ PASS** (7 nodes $\le$ 8 nodes).

2. **The Financial Budget Cap Check:**
   * *Maximum Worldwide Node Footprint:* 7 nodes per AZ $\times$ 3 AZs = **21 total servers**.
   * *Hourly Compute Financial Cost:* $21\text{ total nodes} \times \$0.38\text{ per node-hour} = \mathbf{\$7.98\text{ per hour}}$
   * *Hard Constraint Limit:* Max \$10.00/hour.
   * *Status:* **✅ PASS** (\$7.98 $\le$ \$10.00).

3. **The Private IP Address Exhaustion Check (Subnet `1c`):**
   * *Available Private IP Runway:* Subnet `1c` reports 120 total unallocated IPs. Business compliance mandates keeping at least 30 IPs completely unallocated at all times.
     $$\text{120 Total Available IPs} - \text{30 Reserved Safety IPs} = \mathbf{90\text{ Usable IPs Max}}$$
   * *AWS VPC CNI Network Architecture Allocation Formula:* Because Amazon EKS maps a real, unique private IP directly to every single EC2 Host Node and every single application Container Pod, the total network address consumption formula is:
     $$\text{Total IPs Demanded} = \text{Required Node Count} + \text{Required Pod Count}$$
   * *Steady-State IP Consumption in 1c:* $6\text{ Nodes} + 83\text{ Pods} = \mathbf{89\text{ IP addresses used}}$ (Leaves exactly 1 available address in the pool).
   * *Peak Deployment Surge IP Consumption in 1c:* $7\text{ Nodes} + 92\text{ Pods} = \mathbf{99\text{ IP addresses demanded}}$

#### Why We Cannot Certify:
The architectural footprint requires **99 private IP addresses** in Subnet `1c` to execute a safe, balanced rolling code deployment at peak traffic or to cleanly handle a zone failover. Since the subnet imposes an absolute legal ceiling of **90 usable IP addresses**, our infrastructure is short by exactly **9 IP addresses**. 

The AWS VPC CNI will face network address exhaustion. The remaining 9 containers and nodes will stall permanently in a `CreateContainerConfigError` or `Pending` status. This will unbalance the cluster, trigger traffic routing mismatches, drop availability below our 99.95% SLO, and cause a cascading blackout.

---

## Task 3: In-Event Runbook & Go/No-Go Gate Playbook

| Step | Scope / Target Metric | Binary Gate Threshold Trigger | Immediate Playbook Mitigation Action |
| :--- | :--- | :--- | :--- |
| **1** | **Subnet IP Pool Verification** | Subnet `1c` reports **< 96 available private IP addresses** at T-30m. | **ABORT LAUNCH**. Immediately cancel the marketing event and scale back baseline replicas to prevent a un-routable `Pending` pod lockup. |
| **2** | **Pre-Warmed Cluster Readiness** | Deployment reports **< 249 fully available and running replicas** at T-15m. | **IMPOSE 5-MINUTE HOLD**. If pods do not clear readiness probes within 5 minutes, engage front-end rate limits to restrict traffic entry. |
| **3** | **Symmetrical Topology Balance** | Symmetrical distribution skews **outside of an exact 83-83-83 pod split** at T-10m. | **EXECUTE CORRECTION**. Force a targeted rolling restart (`kubectl rollout restart`) to re-trigger the scheduler's topology spread constraints. |
| **4** | **Baseline Core Performance** | Core API p99 latency under baseline traffic (20,000 RPS) **exceeds 110 ms** at T-5m. | **CANCEL THE EVENT**. A high baseline indicates an unoptimized software layer or database lock that will fail under the 90,000 RPS spike. |
| **5** | **Application CPU Throttling** | Prometheus tracking metrics log **> 2% CPU throttling** across active containers. | **HOT-PATCH ALLOCATION**. Instantly use a patch command to modify and increase the container `limits.cpu` value from 1.0 to 1.5 cores. |
| **6** | **HTTP Server-Side Success Rate** | Edge network metrics log a total **HTTP 5xx error rate spike > 0.05%** at peak. | **ACTIVATE EDGE TRAFFIC SHEDDING**. Engage pre-configured WAF rules at the edge to immediately drop 15% of non-essential traffic with a 429 code. |
| **7** | **Physical Availability Zone Loss** | AWS logs report a **total power or network infrastructure blackout** in Zone `1b`. | **EXECUTE ISOLATION PROTOCOL**. Instruct the ingress load balancer to drop all paths to `1b`. Do not change pod scales; surviving zones carry the load. |
| **8** | **Backend Connection Runway** | Database connection pooling or proxy limits **breach 85% capacity** at peak. | **SCALE STORAGE CAP**. Immediately scale up the database compute layer or trigger an elastic pool expansion to prevent database exhaustion. |
