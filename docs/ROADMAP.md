# Roadmap — OT Data Collector & Traceability Platform (lab)

- **Version:** v0.1 (draft, to refine together)
- **Date:** 2026-11-03
- **Related:** ADR-0001 (working model with Claude), ADR-0002 (public repository)

## Legend

- **M** — milestone: visible outcome → acceptance test. Fits a 1–3 h session.
- **C** — checkpoint: rebuild from zero, then context audit (`/context`, `/doctor prompt-audit`, ADR index).
- **◆** — decision point: choose between options, record the result as an ADR.
- **L0–L4** — Claude access levels (ADR-0001, A5).
- **⏸** — good place for a long break. The lab is rebuildable from the repo at this point.
- **Done** means: I ran the acceptance test and confirmed it. Claude may propose closing, never mark it alone.

## Constraints (as of 2026-11-03)

| Item | Value |
|---|---|
| Hosts | 2× HP Elite Mini 805 G8, 16 GB RAM, 1× ~476 GB NVMe each |
| Host 1 | Proxmox VE, default install: `local` (dir, ~94 GiB, ISO/templates only), `local-lvm` (LVM-thin, ~349 GiB, VM disks) |
| Host 2 | Dedicated to the lab: Proxmox VE + Proxmox Backup Server (backup target and disaster-recovery target). No cluster |
| Network | 1 onboard NIC per host (USB NIC planned), unmanaged switch, Huawei B535 LTE router (CGNAT, no VLAN), no public IP |
| Storage | D1: LVM-thin on both hosts, no ZFS. One disk per host means no redundancy: the backup on host 2 is the only protection |
| Exposure | Nothing exposed to the internet. Remote access through an overlay (Tailscale or similar) |
| Time | ~8–9 h/week, mostly 1 h evenings, sometimes 3 h, rarely a weekend block |

## Network design (N1)

- `vmbr0` on the physical NIC = home LAN: OPNsense WAN + Proxmox management IP. Management stays reachable if OPNsense fails.
- `vmbr1` without a physical port = lab LAN. Zones (services, sandbox, agent) are VLANs on `vmbr1`, fully virtual.
- Host 2 stays on the home LAN. Proxmox backups go host-to-host over the home LAN, not through OPNsense. An OPNsense rule is needed only for backup clients inside lab VMs.
- RAM budget host 1 (GB): OPNsense 2, Docker host 4–6, agent VM 2–4, sandbox 1–2, monitoring 2.

## Tracks

- **INFRA** — at the desk, at home, on Proxmox.
- **APP** — local docker compose or cloud sessions, so it can also move forward from the phone.

The tracks meet in **P5 (CI/CD)**.

```
P0 ──┬── INFRA: I1 → I2 → I3 ──┬── G2 (L2 gate)
     │                         │
     └── APP:   A1 ────────────┴── P5 CI/CD → P6 Monitoring → P7 Backup (L3) → P8 E2E (L4)
```

---

## P0 — Repo and working model (L0) · ~4 h

| ID | Outcome | Acceptance test |
|---|---|---|
| M0.1 | Public GitHub repo skeleton (ADR-0002): protected `main`, `.gitattributes`, `.gitignore`, PR template, `.claude/settings.json` (base deny rules, `autoMemoryEnabled: false`), secret scanning | Direct push to `main` is rejected. A commit with a fake secret is blocked. `git ls-files --eol` shows LF. `/permissions` lists the deny rules from `.claude/settings.json`. Auto memory shows off in `/memory` |
| M0.2 | `CLAUDE.md`, `STATE.md`, ADR template, ADR-0001, `ROADMAP.md`, session checklist in the repo. Account memory cleanup (M1) | A fresh session answers "what is next?" from `STATE.md` only. `/context` shows ≤ ~150 lines of always-loaded context |
| M0.3 | First task from the phone: cloud session → PR with "State delta" | PR merged. `STATE.md` updated by me on `main` |

⏸ after M0.3

## INFRA — I1: Proxmox host baseline · ~4 h

| ID | Outcome | Acceptance test |
|---|---|---|
| I1.1 | Host hardened and documented (updates, repositories, SSH keys, storage layout) | Runbook "host 1 from scratch" exists. `/etc/pve` backup is stored off the host |
| I1.2 | Cloud-init VM template, made by hand to understand it | A clone boots with my SSH key in < 5 min |

## INFRA — I2: Network and security · ~8 h

| ID | Outcome | Acceptance test |
|---|---|---|
| I2.1 | OPNsense VM: WAN on `vmbr0`, LAN on `vmbr1` | A lab VM gets DHCP and internet, and cannot reach the home router or PC |
| I2.2 | Remote access. ◆ Tailscale on OPNsense or a separate subnet-router VM | Lab reachable from the phone on LTE. An external port scan finds nothing |
| I2.3 | Zones as VLANs on `vmbr1` and a firewall rule matrix (documented) | A connectivity-matrix script passes 100% |
| **C1** | Rebuild OPNsense from its config backup | The matrix script passes again |

⏸ after C1

## INFRA — I3: Platform as code (L0: Claude writes, I run) · ~10 h

| ID | Outcome | Acceptance test |
|---|---|---|
| I3.1 | Terraform with a scoped Proxmox API token. ◆ Which Proxmox provider | `apply`, then `destroy`, then `plan` is empty |
| I3.2 | Ansible baseline role (users, SSH hardening, updates) | Second run gives `changed=0` |
| I3.3 | Docker host VM fully from code | Docker host from zero with one command sequence |
| **C2** | Destroy all VMs except OPNsense, rebuild from the repo | Everything works again. Rebuild time recorded in `STATE.md` |

⏸ after C2

## APP — A1: Simulators and data (L1 from A1.1) · ~20 h

| ID | Outcome | Acceptance test |
|---|---|---|
| A1.0 | ◆ Stack review (FastAPI + Pydantic, HTMX + Chart.js, PostgreSQL + TimescaleDB, Alembic, Docker Compose) | ADR accepted |
| A1.1 | Modbus simulator in docker compose (2–3 devices) | A Modbus client reads changing values |
| A1.2 | Collector writes to a TimescaleDB hypertable | Row count matches the polling rate. No data loss after a restart |
| A1.3 | Traceability model (relational) + Alembic migrations | Upgrade and downgrade work on an empty database |
| A1.4 | API | `pytest` suite passes. OpenAPI spec complete |
| A1.5 | Dashboard (HTMX + Chart.js) | Live chart updates. Lookup by serial number works |
| **C3** | `git clone` + `docker compose up` on a clean machine | Working demo, time recorded |

- L1 lesson required before G2: see Claude break something locally, restore it, write a `lessons/` entry.
- **W3 experiment** (Projects in Claude Code) happens in this phase, when the Ready queue is longest.

⏸ after C3

## G2 — Gate to access level L2 · ~6 h · after I2 and the L1 lesson

| ID | Outcome | Acceptance test |
|---|---|---|
| G2.1 | Hooks: `PreToolUse` blocks, `SessionStart` state injection (L2/F3 from ADR-0001) | A blocked command is refused even when I ask for it. A stale `STATE.md` triggers a warning |
| G2.2 | Agent VM (E3) + sandbox VM: own user, SSH key valid only for the sandbox, OPNsense rule "agent → sandbox only", snapshot before each session | The agent VM reaches only the sandbox. Drill "Claude breaks the sandbox → restore from snapshot" recorded in `lessons/` |

Enter **L2** after G2.2. Write an ADR with lessons learned.

## P5 — CI/CD (tracks meet) · ~8 h · needs I3 + C3

| ID | Outcome | Acceptance test |
|---|---|---|
| P5.0 | ◆ CI/CD tool: GitHub Actions + self-hosted runner in the lab (pull-based, nothing exposed) / Gitea + Woodpecker in the lab / Jenkins. The repo is public: the ADR must address the self-hosted runner risk (ADR-0002) | ADR accepted |
| P5.1 | CI on every PR: lint, tests, image build, "State delta" check (F2) | A failing test blocks the merge |
| P5.2 | CD to the lab | After a merge to `main`, `/version` in the lab shows the new commit SHA |
| P5.3 | First cloud scheduled task (e.g. weekly STATE/dependency check) | Report delivered on schedule |

◆ Split `app/` into its own repo? (R3)

⏸ after P5.2

## P6 — Monitoring · ~6 h

| ID | Outcome | Acceptance test |
|---|---|---|
| P6.1 | Metrics and alerts. ◆ Prometheus + Grafana (known) or something new | Stopping the collector fires an alert |
| P6.2 | Logs + MCP with read-only database access (A6) | Claude answers a telemetry question through MCP, without copy-paste |

## P7 — Backup and restore · ~6 h · opens L3

| ID | Outcome | Acceptance test |
|---|---|---|
| P7.1 | Host 2: Proxmox VE + PBS. Scheduled VM backups from host 1, verify jobs | A VM restored under a new ID works |
| P7.2 | Database backup (inside a lab VM, needs an OPNsense rule to PBS) | After a restore, row counts match |
| P7.3 | Disaster-recovery runbook. ◆ Copy of backups to a USB disk outside host 2 · ◆ N2/N4 (lab segment across both hosts) if needed | Runbook reviewed and dry-run done |

Enter **L3** after P7 (scoped Proxmox token, `terraform apply/destroy` as `ask`). Write an ADR with lessons learned.

## P8 — End-to-end from zero · ~4 h · opens L4

| ID | Outcome | Acceptance test |
|---|---|---|
| **C-final** | Wipe host 1, rebuild everything from the repo and PBS on host 2, using only the runbooks | Dashboard shows data again. Traceability data restored. RTO measured and recorded |

Enter **L4** after C-final: read-only by default, changes only in planned chaos exercises. Write an ADR with lessons learned.

After P8: ◆ game server on host 2? ◆ NAS?

---

## Estimate

~75–80 h of work in total, about 10 weeks at full pace. Realistically **4–6 months** with breaks.

## Open decision points (summary)

I2.2 remote access · I3.1 Terraform provider · A1.0 stack · P5.0 CI/CD tool · split `app/` · P6.1 monitoring stack · P7.3 USB copy, N2/N4 · W3 experiment (A1) · WSL2 sandbox on L1 (optional) · game server, NAS (after P8)
