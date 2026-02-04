---
title: "I Built a $0/Month Enterprise Platform (And You Can Too)"
description: "How I turned Oracle's free tier into a production-grade Kubernetes platform with service mesh, secrets management, and AI observability"
date: 2026-02-04
categories: ["infrastructure"]
tags: ["kubernetes", "oracle-cloud", "linkerd", "vault", "langfuse"]
---

I wanted to run real applications—not toy projects, but actual software I'd use daily. Finance tracking. Job search management. Jupyter notebooks. AI observability for my Claude Code sessions.

But I also wanted to do it *right*. Not "throw it in a Docker container and hope for the best" right. Actually right:

- Encrypted service-to-service communication
- Centralized secrets management
- Automatic failover and recovery
- Observable, debuggable, maintainable

The catch? I didn't want to pay $50-200/month for a "real" Kubernetes cluster.

## The Solution

Oracle Cloud's Always Free tier includes an ARM instance with 4 OCPUs and 24GB RAM. That's not a typo—24 gigabytes of memory, free forever.

I spent a day turning it into **oracle-mesh**: a complete application platform running Kubernetes, Linkerd service mesh, Traefik ingress, HashiCorp Vault, and four production applications.

Total monthly cost: **$0**.

---

## What's Running

### The Infrastructure Layer

```
┌─────────────────────────────────────────────────────────────┐
│  KIND (Kubernetes)                                          │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Linkerd  │  │ Traefik  │  │  Vault   │  │PostgreSQL│   │
│  │  (mesh)  │  │(ingress) │  │(secrets) │  │  Redis   │   │
│  └──────────┘  └──────────┘  └──────────┘  │  MinIO   │   │
│                                             │ClickHouse│   │
│                                             └──────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Linkerd** wraps every pod with a sidecar proxy. All traffic between services is mTLS-encrypted automatically. I didn't write a single line of TLS code—the mesh handles it.

**Traefik** routes external traffic to the right services. Everything lives under path prefixes (`/budget/`, `/jobs/`, `/jupyter/`), so I only need one hostname.

**Vault** stores every secret—database passwords, API keys, tokens. Applications never see credentials in environment variables. They're injected at runtime from Vault.

### The Data Layer

- **PostgreSQL** with four isolated databases (finance, jobsearch, langfuse, jupyter)
- **Redis** for caching and queues
- **MinIO** for S3-compatible object storage
- **ClickHouse** for analytics and time-series data

All credentials are Vault-managed. When I deploy a new app, I create a database, store the password in Vault, and the app gets a connection string. No credentials in git, ever.

### The Applications

**Actual Budget** — A local-first personal finance app. Think YNAB but self-hosted. Every transaction, every budget, stored on my infrastructure.

**Planka** — A Trello-like kanban board I use for job search tracking. Cards for applications, columns for stages (Applied, Interviewing, Offer, Rejected). Uses the PostgreSQL database directly.

**JupyterLab** — Data science notebooks with direct access to all my databases. I can query my finance data, job search history, or AI traces from a single notebook.

**Langfuse** — This is the interesting one.

---

## AI Observability: Why Langfuse Matters

I use Claude Code daily. It writes code, runs commands, manages files. But until now, those interactions were ephemeral—I couldn't see what prompts were sent, how many tokens were used, or debug why something went wrong.

Langfuse changes that.

It's an open-source observability platform for LLM applications. Every Claude Code session can be traced:

- What prompts were sent
- What responses came back
- Token counts and latency
- Tool calls and their results

The entire conversation becomes observable, searchable, and debuggable.

### Why Self-Host?

Langfuse has a cloud offering, but I wanted the data on my infrastructure. AI prompts often contain proprietary code, business logic, or sensitive context. Self-hosting means:

- Complete data ownership
- No usage limits
- Integration with my existing database infrastructure
- The satisfaction of running it myself

The Langfuse stack requires PostgreSQL (have it), Redis (have it), MinIO (have it), and ClickHouse (added it). The platform I'd already built made adding AI observability trivial.

---

## The Hard Parts

### Linkerd Needs Gateway API CRDs

The Linkerd installation failed immediately:

```
The Gateway API CRDs must be installed prior to installing Linkerd.
```

This is a relatively new requirement (2024+). The fix is simple—install the CRDs first—but it wasn't in any of the quick-start guides I'd read.

### Vault Hates Read-Only ConfigMaps

Vault's Docker image has an entrypoint script that tries to `chown` the config directory. Kubernetes ConfigMaps are always read-only. The pod crash-looped until I bypassed the entrypoint:

```yaml
command: ["/bin/vault"]
args:
  - server
  - -config=/vault/config/vault.hcl
```

### Tailscale MagicDNS Doesn't Do Subdomains

I originally planned to use `demo.monitoring-arm.ts.net`, `vault.monitoring-arm.ts.net`, etc. MagicDNS doesn't resolve subdomains—only exact hostnames registered in the tailnet.

The fix was path-based routing: everything at `monitoring-arm/demo/`, `monitoring-arm/vault/`, etc. This required Traefik's `stripPrefix` middleware to remove the path prefix before forwarding to backends.

### PostgreSQL 15+ Changed Default Permissions

When Planka tried to create its database tables:

```
permission denied for schema public
```

PostgreSQL 15 changed the default behavior—regular users can no longer create objects in the public schema. I had to explicitly grant permissions:

```sql
GRANT ALL ON SCHEMA public TO planka_app;
```

### ClickHouse Single-Node vs Cluster

Langfuse's migrations assume a ClickHouse cluster with Zookeeper. My single-node setup doesn't have Zookeeper. The fix:

```yaml
env:
  - name: CLICKHOUSE_CLUSTER_ENABLED
    value: "false"
```

---

## What I Learned

### Service Meshes Are Worth It

I was skeptical of Linkerd's value for a personal project. But having automatic mTLS between every service is genuinely useful. I don't think about encryption—it just happens. The observability (which service is talking to which, success rates, latency percentiles) is a bonus.

### Vault Is Overkill (But I'm Glad I Did It)

For a personal project, Kubernetes secrets would be fine. But using Vault taught me patterns I'll use professionally:

- Secrets have versions
- Secrets can be rotated without redeploying apps
- Access can be audited
- The same patterns work at any scale

### KIND Is Production-Ready (For Personal Use)

KIND (Kubernetes IN Docker) is usually positioned as a development tool. But for a single-node personal cluster, it works great. The cluster survives host reboots. Persistent volumes work. The only limitation is no multi-node HA—which I don't need.

### Free Tier ARM Is Seriously Capable

24GB of RAM is substantial. After deploying 21 pods across all tiers, I'm using 3.9GB. That's 84% headroom. I could add several more applications before needing to optimize.

---

## The Numbers

| Metric | Value |
|--------|-------|
| Total pods | 21 |
| Applications | 7 (demo, vault, minio, budget, jobs, jupyter, langfuse) |
| Databases | 4 PostgreSQL + 1 ClickHouse |
| RAM used | 3.9 GB / 24 GB |
| Disk used | 18 GB / 45 GB |
| Monthly cost | $0 |
| mTLS coverage | 100% |

---

## Try It Yourself

The entire setup is documented in my oracle repository:

- **Design document**: Full architecture and decision rationale
- **Phase guides**: Step-by-step deployment instructions
- **Lessons learned**: Every gotcha I hit and how to avoid them
- **Manifests**: All Kubernetes YAML, ready to apply

The core insight: enterprise patterns aren't just for enterprises. Service mesh, secrets management, and observability are tools that improve security and reliability at any scale. The barrier isn't capability—it's knowing how to start.

Oracle's free tier removes the cost barrier. The documentation removes the knowledge barrier. What's left is just doing it.

---

## What's Next

The platform is complete, but it's also a foundation. Next steps I'm considering:

- **Prometheus + Grafana** for metrics visualization
- **Loki** for centralized logging
- **ArgoCD** for GitOps deployments
- **n8n** for workflow automation

Each addition follows the same pattern: store credentials in Vault, deploy to the mesh, expose via Traefik. The hard work is done.

---

## Final Thoughts

A year ago, running this infrastructure would have cost $100-300/month and required significant DevOps expertise. Today, it's free and documented.

The tools have matured. The free tiers have expanded. The documentation has improved. What used to require a team now requires an afternoon.

If you've been waiting to learn Kubernetes, service mesh, or infrastructure-as-code—stop waiting. The barrier is lower than you think. And the skills transfer directly to professional work.

I built a $0/month enterprise platform. You can too.

---

*oracle-mesh is open source. Find the documentation, manifests, and guides in the [oracle repository](https://github.com/jcaldwell-labs).*
