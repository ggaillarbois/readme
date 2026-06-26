# Azure Arc Agent Migration on AWS Spot Instances

## Remediation Plan for Ephemeral Workloads

---

## 1. Problem Statement

The standard Azure Arc agent migration flow (uninstall → wait for MCC scan → reinstall) assumes long-lived instances. On AWS Spot instances, this assumption breaks down because:

- Spot instances can be reclaimed with only a **2-minute termination warning**
- The **Multi-Cloud Connector (MCC) scan runs on an interval** (typically hourly), creating a gap between uninstall and reinstall
- Spot fleets churn constantly — new instances spawn with the **old agent baked in** faster than migration can complete
- Terminated instances leave **orphaned Azure Arc resources** in Azure if not deregistered cleanly

### Why the Classic Flow Fails on Spot

```mermaid
sequenceDiagram
    participant SSM as SSM Document
    participant EC2 as Spot Instance
    participant MCC as Azure Arc MCC
    participant Azure as Azure Arc

    SSM->>EC2: 1. Uninstall old agent
    EC2-->>SSM: ✓ Uninstalled
    Note over EC2,MCC: ⏳ Waiting for next MCC scan (up to 1h)
    Note over EC2: ⚠️ AWS reclaims instance<br/>(2-min warning)
    EC2--xMCC: ❌ Instance gone before scan
    MCC-xAzure: ❌ No reinstall triggered
    Note over Azure: 🧟 Orphaned resource remains
```

---

## 2. Core Principle: Migrate the Source, Not the Instances

For ephemeral workloads, the mental shift is: **don't chase running instances — fix the provisioning path and let natural churn do the work.**

```mermaid
flowchart LR
    A[Golden AMI /<br/>Launch Template /<br/>User-data] -->|New agent baked in| B[New Spot Instances]
    B -->|Register with<br/>new agent| C[Azure Arc]
    D[Old Spot Instances] -->|Natural churn<br/>hours to days| E[Terminated]
    E -->|Replaced by| B

    style A fill:#2d5a3d,color:#fff
    style B fill:#2d5a3d,color:#fff
    style C fill:#1e3a5f,color:#fff
    style D fill:#5a2d2d,color:#fff
    style E fill:#3a3a3a,color:#fff
```

---

## 3. Remediation Strategy — Four Parallel Tracks

```mermaid
flowchart TB
    Start([Migration Kickoff]) --> T1
    Start --> T2
    Start --> T3
    Start --> T4

    T1[Track 1:<br/>Fix Provisioning Source]
    T2[Track 2:<br/>Migrate Long-Lived Fleet<br/>via SSM]
    T3[Track 3:<br/>Graceful Offboarding<br/>on Termination]
    T4[Track 4:<br/>Orphan Reconciliation]

    T1 --> T1a[Update AMI / user-data<br/>with new agent version]
    T1a --> T1b[Roll launch template<br/>new version]
    T1b --> T1c[New instances = new agent]

    T2 --> T2a[Scope SSM target:<br/>on-demand OR<br/>LaunchTime > 24h]
    T2a --> T2b[Run inline<br/>uninstall+install document]
    T2b --> T2c[No MCC wait gap]

    T3 --> T3a[EventBridge rule:<br/>Spot Interruption Warning]
    T3a --> T3b[Trigger SSM Run Command:<br/>azcmagent disconnect]
    T3b --> T3c[Clean Azure Arc exit]

    T4 --> T4a[Scheduled Lambda]
    T4a --> T4b[Compare Azure Arc inventory<br/>vs live EC2 instance IDs]
    T4b --> T4c[Delete stragglers]

    T1c --> Done([Migration Complete])
    T2c --> Done
    T3c --> Done
    T4c --> Done

    style Start fill:#1e3a5f,color:#fff
    style Done fill:#2d5a3d,color:#fff
    style T1 fill:#3d3d1e,color:#fff
    style T2 fill:#3d3d1e,color:#fff
    style T3 fill:#3d3d1e,color:#fff
    style T4 fill:#3d3d1e,color:#fff
```

---

## 4. Track Details

### Track 1 — Fix the Provisioning Source *(highest priority)*

Stop the bleed first. Every new spot instance launched with the old AMI/user-data is a new problem.

**Actions:**
- Identify the provisioning path (Packer/EC2 Image Builder, Launch Template user-data, Karpenter NodePool, EMR bootstrap, AWS Batch compute environment, etc.)
- Bake the new Azure Arc agent version into the image, OR update the `user-data` installation script
- Publish a new Launch Template version and update ASG / Karpenter / Batch references
- Validate: launch one instance, confirm new agent connects to Azure Arc

### Track 2 — Migrate Long-Lived Instances via SSM

For on-demand and long-running instances only. **Skip MCC in the migration path entirely** — use an inline document that uninstalls and reinstalls back-to-back.

```mermaid
flowchart LR
    A[SSM Automation Document] --> B{Target filter}
    B -->|Tag: lifecycle != spot<br/>OR LaunchTime > 24h| C[Eligible Instance]
    C --> D[Step 1: Uninstall old agent]
    D --> E[Step 2: Install new agent inline]
    E --> F[Step 3: azcmagent connect]
    F --> G[Verify connection]

    B -->|Spot + recent| H[Skip — let churn handle it]

    style A fill:#1e3a5f,color:#fff
    style C fill:#2d5a3d,color:#fff
    style H fill:#3a3a3a,color:#fff
    style G fill:#2d5a3d,color:#fff
```

**Why inline:** removes the MCC scan-interval dependency and eliminates the vulnerability window where an instance could die between uninstall and reinstall.

### Track 3 — Graceful Offboarding on Spot Interruption

The most important hygiene piece. Without this, Azure Arc accumulates zombie resources indefinitely.

```mermaid
sequenceDiagram
    participant AWS as AWS Spot Service
    participant EB as EventBridge
    participant SSM as SSM Run Command
    participant EC2 as Spot Instance
    participant Azure as Azure Arc

    AWS->>EC2: ⚠️ 2-min interruption warning
    AWS->>EB: EC2 Spot Instance<br/>Interruption Warning event
    EB->>SSM: Trigger deregistration document
    SSM->>EC2: Run: azcmagent disconnect
    EC2->>Azure: Deregister resource
    Azure-->>EC2: ✓ Clean exit
    Note over EC2: ~30-60s remaining
    AWS->>EC2: 🛑 Terminate
    Note over Azure: ✅ No orphan left behind
```

**Implementation:**
- EventBridge rule on event pattern: `aws.ec2` → `EC2 Spot Instance Interruption Warning`
- Target: SSM Run Command invoking `AWS-RunShellScript` (Linux) or `AWS-RunPowerShellScript` (Windows)
- Script: `azcmagent disconnect --force` with appropriate service principal credentials from Secrets Manager
- Keep the script lean — you have ~90 usable seconds after event propagation

### Track 4 — Orphan Reconciliation Safety Net

Even with Track 3, some instances will die without warning (hardware failure, network partition during the 2-min window). A periodic reconciler catches these.

```mermaid
flowchart TB
    A[EventBridge Scheduled Rule<br/>every 6 hours] --> B[Reconciliation Lambda]
    B --> C[List Azure Arc resources<br/>tagged source=aws-spot]
    B --> D[List live EC2 instances<br/>via DescribeInstances]
    C --> E{Compare}
    D --> E
    E -->|In Azure, not in EC2| F[Orphaned resource]
    E -->|In both| G[Healthy — skip]
    F --> H[Delete from Azure Arc]
    H --> I[Log + metric to CloudWatch]

    style A fill:#1e3a5f,color:#fff
    style B fill:#1e3a5f,color:#fff
    style F fill:#5a2d2d,color:#fff
    style G fill:#2d5a3d,color:#fff
    style H fill:#5a2d2d,color:#fff
```

---

## 5. Rollout Sequence

```mermaid
gantt
    title Migration Timeline
    dateFormat YYYY-MM-DD
    axisFormat %b %d

    section Preparation
    Inventory provisioning paths     :a1, 2026-04-28, 3d
    Build + test new AMI/user-data   :a2, after a1, 4d

    section Track 1 — Source
    Roll new Launch Template         :b1, after a2, 2d
    Validate new instances register  :b2, after b1, 2d

    section Track 3 — Safety Net
    Deploy EventBridge + SSM hook    :c1, after a2, 3d
    Deploy reconciliation Lambda     :c2, after c1, 2d

    section Track 2 — Long-lived
    Run inline SSM on on-demand      :d1, after b2, 3d

    section Track 4 — Verification
    Monitor Azure Arc inventory      :e1, after d1, 7d
    Confirm spot churn complete      :e2, after d1, 7d
```

**Critical ordering note:** Deploy Track 3 (graceful offboarding) **before or alongside** Track 1. Otherwise, the natural churn from the new Launch Template will terminate old-agent instances without deregistering them, creating a flood of orphans.

---

## 6. Decision Matrix — Which Instances Get Which Treatment

| Instance Type | LaunchTime | Treatment | Rationale |
|---|---|---|---|
| On-demand | Any | Inline SSM (Track 2) | Stable enough for uninstall+install |
| Spot | > 24h | Inline SSM (Track 2) | Survived long enough; likely to survive migration |
| Spot | < 24h | Let churn handle it (Track 1) | Will be replaced soon anyway |
| Any | New (post-AMI roll) | Already migrated | Baked in at provisioning |

---

## 7. Pre-Flight Check

Before executing, confirm one thing that changes the whole plan:

> **Are spot instances discovered by MCC, or do they self-onboard via `azcmagent connect` in user-data?**

- **Self-onboard (user-data):** MCC scan interval is irrelevant. Migration = Track 1 only (update user-data). Tracks 3 and 4 still apply for hygiene.
- **MCC discovery:** Full four-track plan applies.

---

## 8. Success Criteria

- [ ] 100% of newly-launched spot instances run new agent version (measured daily)
- [ ] Zero on-demand instances on old agent version after Track 2 completion
- [ ] Azure Arc inventory count matches live EC2 count (±5% tolerance for in-flight terminations)
- [ ] Orphan reconciliation Lambda reports < 10 orphans per run after week 2
- [ ] No customer-visible monitoring/compliance gaps during migration window

---

## 9. Rollback Plan

If the new agent version causes issues:

```mermaid
flowchart LR
    A[Issue detected] --> B{Scope?}
    B -->|Single instance| C[Terminate — ASG<br/>replaces with healthy one]
    B -->|Fleet-wide| D[Revert Launch Template<br/>to previous version]
    D --> E[New instances launch<br/>with old agent]
    E --> F[Run inline SSM<br/>on new-agent instances<br/>to downgrade]

    style A fill:#5a2d2d,color:#fff
    style C fill:#2d5a3d,color:#fff
    style F fill:#3d3d1e,color:#fff
```

The "migrate the source" approach makes rollback trivially fast — you just roll the Launch Template back and let churn reverse the migration.