---
title: "Hardening a $0 Kubernetes Cluster: An SRE Field Guide"
description: "How I took a running-but-unmeasured Kubernetes cluster through four phases of SRE hardening — resource measurement, health probes, resource governance, and remote development tooling — all on Oracle's free ARM tier"
date: 2026-02-09
categories: ["infrastructure"]
tags: ["kubernetes", "sre", "oracle-cloud", "linkerd", "claude-code", "arm64", "devops"]
---

A few weeks ago I wrote about [building a $0/month enterprise platform](https://jcaldwell-labs.github.io/blog/2026-02-04-oracle-mesh-complete/) on Oracle's free ARM tier. Twenty-one pods, service mesh, secrets management, AI observability — the whole works, running on a single ARM instance with 24GB of RAM.

It worked. Everything was up. The pods were green. I could hit my apps through Traefik and feel smug about paying nothing.

But here's the thing about infrastructure: *running* is the easy part. The hard part is knowing whether it's running *well* — and whether it'll keep running when something goes sideways at 3 AM.

I had a cluster with no resource measurements, no health probes, no resource budgets, no backups, and no way to develop on it directly. It was, to borrow a phrase from the SRE handbook, *operating without instrumentation*. Flying blind.

This article is about what I did next: four phases of systematic hardening that took the cluster from "works on my machine" to something an SRE would recognize as *managed*.

---

## The Philosophy: Measure, Then Cut

```
┌─────────┐     ┌────────┐     ┌────────┐     ┌─────────┐     ┌────────┐
│         │     │        │     │        │     │         │     │        │
│ Measure ├────►│ Harden ├────►│ Govern ├────►│ Develop ├────►│ GitOps │
│         │     │        │     │        │     │         │     │        │
└─────────┘     └────────┘     └────────┘     └─────────┘     └────────┘
```

Every SRE knows the temptation: skip straight to the dashboards. Install Prometheus, Grafana, Loki, and ArgoCD. Bask in the green lights.

But dashboards without baselines are decoration. You can't set meaningful alerts if you don't know what "normal" looks like. You can't write resource quotas if you don't know what your pods actually consume. And you definitely can't justify the resource cost of a monitoring stack if you haven't measured how much headroom you've got.

So the phases follow a deliberate order:

1. **Measure** — Find out what's actually happening
2. **Harden** — Make it resilient based on what you measured
3. **Govern** — Set budgets based on what you hardened
4. **Develop** — Make it comfortable to work on
5. **GitOps** — Make it declarative (coming next)

Each phase feeds the next. You can't harden what you haven't measured. You can't govern what you haven't hardened. The data flows forward.

---

## Phase 1: Resource Measurement

**The question:** *What is this cluster actually doing?*

Before this phase, I had no idea how much CPU or memory any of my pods were using. Kubernetes doesn't ship with resource metrics out of the box — you need [metrics-server](https://github.com/kubernetes-sigs/metrics-server) for that. Without it, `kubectl top` just says:

```
error: Metrics API not available
```

Not helpful.

### Deploying metrics-server

The fix was straightforward: deploy metrics-server v0.7.2 with a KIND-specific flag (`--kubelet-insecure-tls`, because KIND uses self-signed kubelet certificates). Three minutes later:

```
$ kubectl top nodes
NAME                        CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
oracle-mesh-control-plane   429m         10%      4113Mi          17%
```

Ten percent CPU. Seventeen percent memory. On a 4-OCPU, 24GB ARM instance. That's a *lot* of headroom I didn't know I had.

### The Resource Inventory

With metrics flowing, I captured a full inventory of every pod. And the numbers told a story:

```
$ kubectl top pods -A
NAMESPACE   NAME                              CPU(cores)   MEMORY(bytes)
langfuse    clickhouse-787d5fc7d4-zbdf7       273m         729Mi
langfuse    langfuse-web-75cd54b748-tb5hv      2m          486Mi
kube-system kube-apiserver-oracle-mesh...      34m         460Mi
langfuse    langfuse-worker-69cddbd4f7-2z8mf   5m         190Mi
kube-system kube-controller-manager...         9m         108Mi
postgres    postgres-67c95bc796-rvnlf          3m         100Mi
...
```

ClickHouse was the elephant in the room — consuming 60% of all pod CPU and bursting up to 997m (nearly a full core). Langfuse's web component was eating 486Mi of memory just sitting there. But the beautiful thing was: *now I knew*. Data beats assumptions every time.

The inventory also revealed that 9 out of 21 deployments had zero resource requests or limits set. Kubernetes was scheduling them blind — no idea how much they needed, no ability to evict them gracefully under pressure.

```
 ________________________________________________
/\                                               \
\_| RESOURCE INVENTORY HEATMAP                   |
  |                                              |
  | CPU and memory usage across 30 pods          |
  | Color-coded by consumption tier              |
  |                                              |
  | [ Future PNG will replace this placeholder ] |
  |   ___________________________________________|_
   \_/_____________________________________________/
```

---

## Phase 2: Cluster Hardening

**The question:** *What happens when something goes wrong?*

With real numbers in hand, I could answer the three hardening questions:

1. **How does Kubernetes know if a pod is healthy?** (Health probes)
2. **How much resource can each pod consume?** (Resource limits)
3. **What happens to my data if a pod dies?** (Backups)

### Health Probes: Teaching Kubernetes to Take a Pulse

Before this phase, Kubernetes had a simple strategy for 9 of my deployments: if the process is running, it's fine. That's like saying a car is working because the engine is on — even if the wheels fell off.

Health probes fix this. They're the Kubernetes equivalent of asking "are you *actually* okay?" every few seconds:

- **Liveness probes:** "Are you alive?" If no → restart the pod.
- **Readiness probes:** "Can you serve traffic?" If no → stop sending requests.
- **Startup probes:** "Have you finished booting?" If no → give it more time before checking liveness.

Each service got probes tailored to its nature:

```yaml
# PostgreSQL: ask the database directly
livenessProbe:
  exec:
    command: ["pg_isready", "-U", "postgres"]
  periodSeconds: 10

# Redis: the canonical ping
livenessProbe:
  exec:
    command: ["redis-cli", "ping"]
  periodSeconds: 10

# Vault: seal-aware (don't restart a sealed vault!)
livenessProbe:
  httpGet:
    path: /v1/sys/health?sealedcode=200&uninitcode=200
    port: 8200
  periodSeconds: 30
```

That Vault probe is worth highlighting. Vault has a concept of being *sealed* — it's running, but it can't serve requests until someone provides unseal keys. A naive liveness probe would see "not responding to health checks" and restart it. Which would seal it again. Which would trigger another restart. Crash loop.

The `sealedcode=200` parameter tells the health endpoint to return 200 (healthy) even when sealed. Vault stays up, patiently waiting for someone to unseal it, instead of fighting Kubernetes in an infinite restart loop.

### The ClickHouse Surprise

Midway through patching, ClickHouse went into CrashLoopBackOff. The reason? Its PersistentVolumeClaim uses `ReadWriteOnce` access mode — only one pod can mount it at a time. When Kubernetes tried a rolling update (start new pod, then stop old pod), both pods tried to mount the same volume simultaneously. Deadlock.

The fix: switch ClickHouse and Vault to `Recreate` deployment strategy — stop the old pod first, then start the new one. It means a brief downtime during updates, but it means updates actually *work*:

```yaml
strategy:
  type: Recreate   # Not RollingUpdate — RWO volumes need this
```

This is the kind of thing you only discover when you actually try to update a deployment. If I'd skipped probes and jumped straight to monitoring, this failure mode would have been lurking — waiting for the worst possible moment to surface.

### Resource Limits: Drawing Lines in the Sand

With measured usage data from Phase 1, I could set intelligent resource requests and limits. The approach: **Burstable QoS** — requests at measured usage, limits at 2-3x measured:

```
Pod measured at 5m CPU / 40Mi memory:
  requests: 5m CPU / 40Mi memory    (what it normally uses)
  limits:   30m CPU / 100Mi memory  (how high it can burst)
```

This tells the Kubernetes scheduler two things: how much to *reserve* (requests), and how much to *allow* (limits). Burstable QoS means pods can use more than their request when the node has spare capacity, but they'll be throttled or evicted before they can starve their neighbors.

Eight infrastructure deployments got limits. One — `local-path-provisioner` — I deliberately skipped. It uses 1m CPU and 7Mi memory. Setting limits on it risks breaking PersistentVolume provisioning for a savings of practically nothing. Sometimes the right decision is to leave well enough alone.

### Backups: Because Disks Are Mortal

The final hardening step: automated backups for every stateful service.

```
┌──────────┐     ┌──────────────────┐     ┌──────────────────┐
│ Cron Job │     │  Backup Script   │     │  Local Storage   │
│ (daily)  ├────►│  pg_dump / tar   ├────►│  ~/backups/data  │
│          │     │  mc mirror       │     │  7-day retention │
└──────────┘     └──────────────────┘     └──────────────────┘
```

Three scripts, staggered across a 2-4 AM ET window to avoid I/O contention:

| Service    | Schedule             | Method                  | Retention |
|------------|----------------------|-------------------------|-----------|
| PostgreSQL | Daily 7:00 AM UTC    | `pg_dump` × 4 databases | 7 days    |
| Vault      | Daily 7:30 AM UTC    | `tar` of file backend   | 7 days    |
| MinIO      | Weekly Sun 8:00 AM   | `mc mirror` via port-forward | 30 days |

The scripts are designed to be *non-fatal*. They try to rsync copies to my WSL machine over Tailscale, but if that fails (and it does — Tailscale SSH from oracle to WSL requires interactive browser auth), they shrug and keep the local copies. A partial backup strategy beats no backup strategy.

---

## Phase 3: Resource Governance

**The question:** *How do I prevent one service from eating the whole cluster?*

Resource limits on individual pods are necessary but not sufficient. What if someone deploys a second replica? Or a third? Pod-level limits don't prevent namespace-level sprawl.

### ResourceQuotas: Namespace-Level Budgets

Kubernetes ResourceQuotas are like departmental budgets in a company. Each namespace gets an allocation, and it cannot exceed it — period:

```
$ kubectl describe quota -n langfuse
Name:            resource-quota
Namespace:       langfuse
Resource         Used    Hard
--------         ----    ----
limits.cpu       2500m   2560m
limits.memory    2560Mi  2620Mi
pods             3       8
requests.cpu     500m    530m
requests.memory  1280Mi  1340Mi
```

Langfuse is using 500m of its 530m CPU request budget. It's living close to the line — which is exactly right. The quota is derived from measured usage plus a small buffer, not an arbitrary guess.

Ten namespaces got quotas. Four infrastructure namespaces (kube-system, linkerd, linkerd-viz, local-path-storage) were deliberately excluded — you don't want a quota preventing CoreDNS from starting during a node reboot.

Each namespace also got a LimitRange: default resource requests and limits for any container that doesn't specify its own. Think of it as a safety net — if someone deploys a container without resource specs, it gets 50m CPU / 64Mi memory request and 200m CPU / 256Mi memory limit instead of unlimited access.

### The Headroom Question

With quotas in place, the natural question: *How much room do I have left?*

```
Node Capacity:    4000m CPU  /  ~24Gi memory
Current Quotas:   1420m CPU  /  3440Mi memory (requests)
Infrastructure:    454m CPU  /  1006Mi memory
Available:        1354m CPU  / 18022Mi memory
```

Over a gigacycle of CPU and 18 GB of memory available for Tier 5 expansion. More than enough for Prometheus, Grafana, and the monitoring stack.

But CPU and memory aren't the real constraint. **Disk is.**

```
Total disk:      ~47 GB
Used:            ~23 GB
Available:       ~24 GB
Tier 5 needs:    11-21 GB (depending on log retention)
```

ClickHouse alone writes several GB of analytics data. Prometheus will want retention. Loki will want log storage. All of them write to the same physical disk. The headroom document I produced maps this out explicitly — which Tier 5 components fit, what retention settings to use, and at what point I'll need to get creative with storage.

This is the value of measuring before expanding: I know *exactly* where the ceiling is before I start building toward it.

---

## Phase 4: Remote Development

**The question:** *Can I develop directly on the server?*

The first three phases were all executed from my WSL workstation, running `ssh oracle 'kubectl ...'` for every command. This works, but it's like performing surgery through a mail slot. I wanted to be *on* the machine — editing manifests, running kubectl, using Claude Code — directly.

### The Developer Toolkit

Step one: make the server comfortable to work on. Oracle's instance ships as "Ubuntu 24.04 Minimal" — emphasis on minimal. No syntax highlighting, no fuzzy finding, no modern CLI tools. Just bash and vi.

I installed eight tools that transform the experience:

```
$ rg --version    # ripgrep 14.1.0  — fast code search
$ fzf --version   # 0.44.1         — fuzzy finder for everything
$ bat --version   # bat 0.24.0     — cat with syntax highlighting
$ fd --version    # fd 9.0.0       — modern find replacement
$ htop --version  # htop 3.3.0     — process monitoring
$ yq --version    # yq v4.44.1     — YAML swiss army knife
$ gh --version    # gh 2.86.0      — GitHub CLI
$ node --version  # v22.22.0       — Node.js for tooling
```

A note about Ubuntu's naming conventions: the `bat` package is actually called `batcat`, and `fd` is called `fdfind`, because Ubuntu already has packages with those names. Symlinks in `~/.local/bin/` fix this:

```bash
ln -sf /usr/bin/batcat ~/.local/bin/bat
ln -sf /usr/bin/fdfind ~/.local/bin/fd
```

Small thing, but it means muscle memory from every other system transfers directly.

### Shell, Tmux, and Vim

Twenty-plus shell aliases turn verbose kubectl commands into quick shortcuts:

```bash
alias k='kubectl'
alias kgp='kubectl get pods -A'
alias klogs='kubectl logs -f'
alias kns='kubectl get ns'
alias ktop='kubectl top pods -A --sort-by=cpu'
alias disk='df -h / /mnt/data 2>/dev/null'
```

Tmux gets pane navigation (h/j/k/l, because vim keybindings should be everywhere) and a status bar that shows the current kubectl context — so you always know which cluster you're talking to:

```
 oracle-mesh | 15:42
```

Vim gets fzf integration (Ctrl+P to open files, Ctrl+G to ripgrep across the project) and YAML-friendly settings (2-space indentation, because Kubernetes manifests are *opinionated* about whitespace).

```
 ________________________________________________
/                                                \
|  TMUX SESSION ON ORACLE                        |
|                                                |
|  Showing: kubectl context in status bar,       |
|  split panes with cluster monitoring,          |
|  and vim editing a manifest                    |
|                                                |
|  [ Future PNG will replace this placeholder ]  |
\__________________________________________  __'\
                                           |/   \\
                                            \    \\  .
                                                 |\\/|
                                                 / " '\
                                                 . .   .
                                                /    ) |
                                               '  _.'  |
                                               '-'/    \
```

### Claude Code on ARM64

Here's where it gets interesting. [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) — Anthropic's CLI tool for AI-assisted development — runs on the oracle instance. Not through SSH tunneling, not through a proxy. *Directly.*

This means I can SSH into oracle, open a tmux session, and have Claude Code help me write manifests, debug deployments, and run kubectl commands — all with full local context. It knows about the cluster topology, the backup schedules, the disk constraints. It even has safety rules to prevent it from doing anything destructive:

```
NEVER run `kubectl delete namespace` without explicit user confirmation
NEVER modify or delete backup scripts or cron jobs
NEVER run `docker system prune` or `kind delete cluster`
NEVER store secrets, passwords, or unseal keys in files
```

These aren't suggestions — they're encoded in a 138-line CLAUDE.md that gets loaded every time Claude Code starts on oracle. Think of it as the AI equivalent of putting warning labels on the circuit breaker panel.

I also enabled auto-updates, so the Claude Code binary stays current without manual intervention. On ARM64. For free. In 2026, this is just a normal thing you can do.

### Git Without GitHub

The oracle repo isn't on GitHub — it contains infrastructure details I'd rather keep private. Instead, I set up a local bare repo on oracle and sync over SSH:

```
                    WSL                         Oracle
               ┌──────────┐              ┌──────────────────┐
               │  oracle/  │  git push   │ repos/oracle.git │
               │ (working  ├────────────►│   (bare repo)    │
               │   copy)   │   via SSH   │                  │
               └──────────┘              └────────┬─────────┘
                                                  │ git clone
                                                  ▼
                                         ┌──────────────────┐
                                         │ projects/oracle  │
                                         │  (working copy)  │
                                         └──────────────────┘
```

`git push oracle --all` from WSL, `git pull` on oracle. No cloud service in the middle. The bare repo acts as a local exchange point — same pattern GitHub uses, minus the web UI and the monthly bill.

---

## The AI Copilot: Structured Execution with Claude Code

I should be transparent about how these phases were executed: each one was planned and run with Claude Code using a structured workflow called GSD (Get Stuff Done). It's a plugin that enforces a discipline I find hard to maintain on my own:

1. **Research** the problem space before planning
2. **Plan** with explicit success criteria (not just tasks)
3. **Execute** with atomic commits per task
4. **Verify** against the original goal, not just the checklist

The workflow spawns subagents — separate Claude Code instances, each with a fresh context window — to execute individual plans in parallel. The orchestrator stays lean (10-15% of its context budget), delegating the heavy lifting to agents that each get a full 200k token window.

Here's what that looks like in practice. This is the actual output from Phase 4 execution:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► EXECUTING PHASE 4
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Phase 4: Remote Development — 3 plans across 2 waves

| Wave | Plans       | What it builds                              |
|------|-------------|---------------------------------------------|
| 1    | 04-01, 04-02| Dev tools + shell, Claude Code + CLAUDE.md  |
| 2    | 04-03       | Git/GitHub access, full verification        |

◆ Spawning 2 agents in parallel...
  → Dev tools + shell config (04-01)
  → Claude Code settings + CLAUDE.md (04-02)

✓ 04-01 complete: 8 tools installed, shell/tmux/vim configured (5 min)
✓ 04-02 complete: Auto-updates enabled, 138-line CLAUDE.md written (3 min)

◆ Spawning executor for wave 2...
  → Git/GitHub access, GSD plugin, full verification (04-03)

╔════════════════════════════════════════════════════════╗
║  CHECKPOINT: Action Required                           ║
╚════════════════════════════════════════════════════════╝
  Add SSH key to GitHub, authenticate gh CLI, clone repo

[user completes manual steps]

✓ 04-03 complete: Environment verified across 7 areas

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► PHASE 4 COMPLETE ✓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

The checkpoint is the key innovation for infrastructure work. Some tasks genuinely require human intervention — adding SSH keys to GitHub, authenticating CLI tools, verifying that tmux status bars look right. The workflow pauses at these points, presents exactly what needs to happen, and resumes when the human signals "done."

This isn't about replacing the operator. It's about structuring the work so the operator spends their attention on the decisions that matter, not on the YAML that doesn't.

### The Numbers

| Phase | Plans | Duration | What it measured / built |
|-------|-------|----------|-------------------------|
| 1. Resource Measurement | 2 | 6 min | metrics-server, full pod/node inventory |
| 2. Cluster Hardening | 3 | 31 min | Probes, limits, backups for all deployments |
| 3. Resource Governance | 2 | 11 min | Quotas for 10 namespaces, headroom projection |
| 4. Remote Development | 3 | ~25 min | Dev tools, Claude Code, Git, full verification |

Total: 10 plans, ~73 minutes of execution time. The cluster went from unmeasured and unprotected to hardened, budgeted, and development-ready in about an hour of actual compute time. (Planning and research took longer, but that's time well spent — measure twice, cut once.)

---

## What I Actually Learned

### SRE Is Mostly About Knowing What You Don't Know

Before Phase 1, I would have guessed my cluster was using "not much" CPU. The actual number — 11% — confirmed the guess, but the *distribution* was surprising. ClickHouse alone was eating 60% of all pod CPU. That's the kind of insight that changes how you think about capacity planning.

### Health Probes Are Cheaper Than You Think

Adding probes to 9 deployments took 20 minutes and prevented at least one class of outage (the ClickHouse RWO volume deadlock). The ROI on health probes is absurd — a few lines of YAML prevent hours of debugging. Every deployment should have them from day one.

### ResourceQuotas Are Guardrails, Not Walls

The total quota limits (6540m CPU / 7664Mi memory) *exceed* the node's actual capacity (4000m CPU / 24Gi memory). That's intentional. With Burstable QoS, pods can exceed their requests when the node has spare capacity. The quotas prevent runaway growth, not normal bursting. It's like a credit limit — you're allowed to spend up to the limit, but you're expected to pay it back.

### Disk Is the Real Constraint

Every conversation about Kubernetes resources focuses on CPU and memory. On a single-node cluster with persistent volumes, disk is the bottleneck you don't see coming. I have 24GB available. Tier 5 (Prometheus, Grafana, Loki, ArgoCD) needs 11-21GB depending on retention settings. That's a *tight* fit. CPU and memory have 80%+ headroom. Disk has maybe 50%. The headroom document makes this explicit.

### The AI Copilot Changes the Economics

The structured workflow — research, plan, execute, verify — sounds like heavyweight process. In practice, each plan averaged 7 minutes of execution. The overhead of structure is paid back in:

- **No rework:** Every task has explicit success criteria checked before completion
- **No drift:** Each task is committed atomically, so you can `git log` the exact sequence
- **No context loss:** STATE.md tracks decisions and blockers across sessions
- **Parallel execution:** Independent plans run simultaneously, cutting wall-clock time

I wouldn't hand-write 10 Kubernetes manifests and 3 backup scripts in 73 minutes. But I would *review* them in that time — which is exactly what happened.

---

## The Cluster Today

```
$ kubectl top nodes
NAME                        CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
oracle-mesh-control-plane   1220m        30%      4579Mi          19%

$ kubectl get pods -A | wc -l
31

$ kubectl get ns | wc -l
15
```

Thirty pods. Fourteen namespaces. Every service with health probes, resource limits, and namespace-level quotas. Backups running nightly. A development environment on the server itself, with Claude Code, modern CLI tools, and git sync.

Monthly cost: still $0.

---

## What's Next: Phase 5 — GitOps

The cluster is hardened and governed, but deployments are still imperative: `kubectl apply -f manifest.yaml`. If someone (or something) modifies a resource directly, the cluster drifts from its declared state with no way to detect it.

Phase 5 introduces GitOps — probably ArgoCD Core, watching the git repo for changes and automatically syncing them to the cluster. If a manifest in git says "3 replicas" and the cluster has 2, ArgoCD will fix it. If someone `kubectl edit`s a deployment directly, ArgoCD will flag the drift.

The headroom analysis from Phase 3 confirmed ArgoCD fits within the resource budget. The dev environment from Phase 4 means I can develop directly on oracle. Everything is in place.

But that's an article for another day. Right now, I'm going to SSH into oracle, open tmux, and enjoy the view.

```
ubuntu@monitoring-arm:~$ k get pods -A
NAMESPACE            NAME                                                READY   STATUS
demo                 flask-demo-599b4f64d4-5knvb                         2/2     Running
finance              actual-budget-9977665bb-trft4                       2/2     Running
jobsearch            planka-6cf79cdf74-ksvc8                             2/2     Running
jupyter              jupyterlab-64768bdc48-ws868                         2/2     Running
langfuse             clickhouse-787d5fc7d4-zbdf7                         2/2     Running
langfuse             langfuse-web-75cd54b748-tb5hv                       2/2     Running
langfuse             langfuse-worker-69cddbd4f7-2z8mf                    2/2     Running
minio                minio-6cd7c56f9-5xf7q                               2/2     Running
postgres             postgres-67c95bc796-rvnlf                           2/2     Running
redis                redis-5cffc4fdfd-bm22v                              2/2     Running
traefik              traefik-796489dc45-5bwtq                            2/2     Running
vault                vault-5f4f745486-rtvpp                              1/2     Running
```

Every pod 2/2 — application container plus Linkerd sidecar, mTLS encrypted. Except Vault, sitting at 1/2 because it's sealed and the readiness probe correctly reports it as not ready for traffic.

Even the failures are intentional now.

---

*The oracle-mesh project is tracked in a private repository. The manifests, phase guides, and lessons learned are documented in the [oracle repo](https://github.com/jcaldwell-labs). Find the first article in this series: [I Built a $0/Month Enterprise Platform](https://jcaldwell-labs.github.io/blog/2026-02-04-oracle-mesh-complete/).*
