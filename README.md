# skills
My personal directory of skills, straight from my `.claude` directory.

## What is this?
Skills are modular instruction sets that extend Claude's capabilities in specific domains. Each skill is a folder with a `SKILL.md` — a context file Claude reads before tackling tasks in that area — plus optional `references/` with deeper documentation loaded on demand.

Install a `.skill` file in Claude.ai → Settings → Skills.

---

## Skills

### [`devops`](./devops-core/)
Full-stack DevOps and infrastructure engineering.

**Covers:** Kubernetes (K8s/K3s), Docker, Helm, FluxCD v2, GitOps, GitLab CI/CD, Terraform, Terragrunt, Ansible, Bash/Linux, Prometheus/Grafana, Nginx Ingress, cert-manager.

**Use for:** debugging CrashLoopBackOff/OOMKilled/Pending pods, writing production-ready manifests, designing GitOps repo structure, building CI/CD pipelines, IaC modules, Ansible roles, PromQL queries, SLO definitions.

**References:**
- `k8s-networking.md` — CNI, NetworkPolicy, Ingress, DNS debugging
- `fluxcd-patterns.md` — ImageUpdateAutomation, multi-tenant setup, bootstrap
- `gitlab-advanced.md` — DAG pipelines, dynamic environments, partial rebuild, cache
- `terraform-terragrunt.md` — modules, for_each, lifecycle, Terragrunt multi-env
- `ansible.md` — roles structure, handlers, Vault, Jinja2, ad-hoc commands

---

### [`noc`](./noc-engineer/)
Network operations, protocol analysis, and censorship circumvention.

**Covers:** TCP/IP internals, routing, DNS, iptables/nftables, Wireshark/tcpdump, VPN protocols (WireGuard, AmneziaWG 1.5, VLESS+Reality, Hysteria2, Shadowsocks, MTProxy), DPI evasion, ТСПУ/whitelist bypass (РФ), zapret/zapret2, VK TURN proxy, MTU debugging, domain lists, VPN node monitoring.

**Use for:** diagnosing network issues, setting up self-hosted VPN infrastructure (3x-ui, Xray, Amnezia), bypassing РФ censorship (blacklist and whitelist modes), analyzing packet captures, configuring firewalls, selecting the right bypass strategy for specific ISP/region.

**References:**
- `vpn-protocols.md` — AmneziaWG 1.5 params, VLESS+Reality config, Hysteria2, protocol comparison for РФ
- `dpi-censorship.md` — ТСПУ architecture, all blocking models, decision tree for bypass strategy, whitelist IP ranges, VK TURN proxy, tool ecosystem (zapret2, b4, xraycheck, RealiTLScanner, whitebox, itdoginfo/allow-domains)
- `wireshark.md` — display filters, VPN traffic analysis, TCP diagnostics, tshark recipes

---

### [`backend`](./backend/)
Backend API development — TypeScript/Bun and Go.

**Covers:** Hono, Elysia, Express, Fastify, Go chi/net/http, DrizzleORM, pgx, sqlc, PostgreSQL (schema, indexes, migrations), Redis (cache, queues, pub/sub), BullMQ, JWT auth, Zod validation, layered architecture, Repository pattern, DI, Result type, Event bus, Docker, systemd.

**Use for:** writing HTTP APIs, structuring backend projects, code review, N+1 diagnosis, caching strategy, queue setup, auth implementation, CQRS/event-driven design decisions.

**References:**
- `go-backend.md` — chi router, pgx transactions, graceful shutdown, config
- `architecture.md` — layered arch, DI composition root, event-driven, CQRS, антипаттерны

---

### [`bots`](./bots/)
Telegram bot development with GrammY.

**Covers:** GrammY (menus, sessions, middleware, error handling), Telegram Bot API (all methods, limits, types), FSM via session, inline keyboards, payments (Telegram Stars, pre_checkout flow), webhook vs polling, broadcast with rate limiting, Redis session storage, deep links, Mini App integration, admin access control.

**Use for:** building Telegram bots, implementing subscription flows, payments, FSM conversation flows, webhook setup, bot architecture for multi-feature projects.

**References:**
- `bot-api.md` — full Bot API reference: methods, limits (4096 chars, 30 msg/s), parse modes, error codes, inline mode, group/channel patterns

---

### [`frontend`](./frontend/)
Frontend development — React, Svelte 5, Telegram Mini Apps.

**Covers:** React (hooks, context, memoization, Suspense), Svelte 5 (runes: $state/$derived/$effect, snippets), TypeScript on frontend, Vite, TailwindCSS (cn, cva), TanStack Query (optimistic updates), React Hook Form, Telegram Mini App (WebApp API, initData verification), SSR/SSG, bundle optimization, accessibility.

**Use for:** building components, state management decisions, Telegram Mini App integration and backend verification, TanStack Query setup, Svelte 5 migration, performance optimization.

**References:**
- `performance.md` — bundle optimization, lazy loading, Core Web Vitals
- `forms.md` — React Hook Form + Zod, controlled/uncontrolled patterns

---

## Structure

```
skills/
├── devops-core/
│   ├── SKILL.md
│   └── references/
│       ├── k8s-networking.md
│       ├── fluxcd-patterns.md
│       ├── gitlab-advanced.md
│       ├── terraform-terragrunt.md
│       └── ansible.md
├── noc-engineer/
│   ├── SKILL.md
│   └── references/
│       ├── vpn-protocols.md
│       ├── dpi-censorship.md
│       └── wireshark.md
├── backend/
│   ├── SKILL.md
│   └── references/
│       ├── go-backend.md
│       └── architecture.md
├── bots/
│   ├── SKILL.md
│   └── references/
│       └── bot-api.md
├── frontend/
│   ├── SKILL.md
│   └── references/
│       ├── performance.md
│       └── forms.md
└── README.md
```

## Packaged releases

| Skill | File |
|-------|------|
| devops | `devops.skill` |
| noc | `noc.skill` |
| backend | `backend.skill` |
| bots | `bots.skill` |
| frontend | `frontend.skill` |

---

## Notes

- References are **not loaded automatically** — Claude reads them when the topic warrants it
- Skills are provider-agnostic where possible; cloud-specific configs belong in private overlays
- `dpi-censorship.md` includes links to upstream repos that change frequently — Claude is instructed to check latest commits when answering time-sensitive questions about РФ blocking