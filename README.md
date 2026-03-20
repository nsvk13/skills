# skills

My personal directory of skills, straight from my `.claude` directory.

## What is this?

Skills are modular instruction sets that extend Claude's capabilities in specific domains. Each skill is a folder with a `SKILL.md` — a context file Claude reads before tackling tasks in that area — plus optional `references/` with deeper documentation loaded on demand.

Install a `.skill` file in Claude.ai → Settings → Skills.

---

## Skills

### [`devops`](./devops/)

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

### [`noc`](./noc/)

Network operations, protocol analysis, and censorship circumvention.

**Covers:** TCP/IP internals, routing, DNS, iptables/nftables, Wireshark/tcpdump, VPN protocols (WireGuard, AmneziaWG 1.5, VLESS+Reality, Hysteria2, Shadowsocks, MTProxy), DPI evasion, ТСПУ/whitelist bypass (РФ), zapret/zapret2, VK TURN proxy, MTU debugging, domain lists, VPN node monitoring.

**Use for:** diagnosing network issues, setting up self-hosted VPN infrastructure (3x-ui, Xray, Amnezia), bypassing РФ censorship (blacklist and whitelist modes), analyzing packet captures, configuring firewalls, selecting the right bypass strategy for specific ISP/region.

**References:**
- `vpn-protocols.md` — AmneziaWG 1.5 params, VLESS+Reality config, Hysteria2, protocol comparison for РФ
- `dpi-censorship.md` — ТСПУ architecture, all blocking models, decision tree for bypass strategy, whitelist IP ranges, VK TURN proxy, tool ecosystem (zapret2, b4, xraycheck, RealiTLScanner, whitebox, itdoginfo/allow-domains)
- `wireshark.md` — display filters, VPN traffic analysis, TCP diagnostics, tshark recipes

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
└── README.md
```

## Packaged releases

| Skill | File |
|-------|------|
| devops-core | `devops-core.skill` |
| noc-engineer | `noc-engineer.skill` |

---

## Notes

- References are **not loaded automatically** — Claude reads them when the topic warrants it
- Skills are provider-agnostic where possible; cloud-specific configs belong in private overlays
- `dpi-censorship.md` includes links to upstream repos that change frequently — Claude is instructed to check latest commits when answering time-sensitive questions about РФ blocking