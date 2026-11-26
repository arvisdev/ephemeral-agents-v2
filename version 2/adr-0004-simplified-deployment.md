# ADR-0004 — Simplified Deployment: Shared Spine + Paired Project VMs

**Status:** ACCEPTED (owner, 2026-11-25) — design discussion closed; build plan below.
**Date:** 2026-11-25
**Origin:** Owner directive after the full V2 milestone sequence closed (TP-M5,
TP-M1-COS, ADR-0003 hardening complete). The milestone-staged install
(m0–m5 `install.sh` chains) is correct as a *gate harness* but wrong as a
*deployment path*: milestone scripts are order-sensitive (ledger 24), a
re-run silently resets later milestones' staged config, and a clean
deployment costs days. The owner's goal: **deploy a new project in under
an hour**, with independently installable and independently testable
components, verified by a smoke suite after each deployment.

This ADR is NOT a new version tree. The repo remains one standalone
product (one deploy tree, one book, one decision log); this wave follows
the ADR-0003 pattern — gated changes folded into `deploy-v2/` — and lands
in the book as revisions of ch14 (build-from-scratch) and ch16
(estates that clone), not as a new edition.

---

## 1. Decisions

| ID | Decision | Rationale |
| :-- | :-- | :-- |
| **D-shared-spine** | Three shared services, each on its OWN VM, serving ALL agent-system pairs estate-wide: **Infisical** (secrets vault), **hindsight** (memory), **docker registry** (images). First deployment = 5 VMs: infisical-VM, hindsight-VM, registry-VM, VM-Xa, VM-Xb. | Secrets are needed by everything → vault deploys first; shared spine stands independently of any project pair. |
| **D-secret-vault** (LANDS here; was deferred) | Secrets' source of truth moves from hand-staged gitignored `.env` files to **self-hosted Infisical (CE, on-prem, LAN-only)** — Infisical Cloud is rejected: the estate is de-identified and LAN-contained; a cloud dependency would break that property. Vault-first ordering: Infisical → hindsight → registry → project VMs. | Vault pattern (modeled on HashiCorp Vault's source-of-truth principle): the vault is authoritative; runtime env files become **derived artifacts** written at deploy time from a vault read (0600, `umask 077`, untracked) and consumed by compose `${VAR}` interpolation unchanged. |
| **D-deploy-roles** | Role-shaped deploy scripts replace the milestone chain as the deployment path: `shared/infisical/`, `shared/hindsight/`, `shared/registry/`, `project/xa/`, `project/xb/` — each **idempotent, self-contained, driven by `hosts.env`, with its own smoke check**. An orchestrator (`deploy.sh`) runs them in dependency order with `--only <component>` / `--project <name>`. Each component is installed and tested **independently** (owner decision). | Kills the ledger-24 order-sensitivity class: no more milestone-era staged config to re-apply; each script produces the FINAL-era state directly. The m0–m5 scripts are retired to gate/evidence harnesses. |
| **D6-REV** | **One project = one (VM-Xa, VM-Xb) pair.** An Xb MAY serve multiple Xa's (the coexistence machinery supports it), but the standard project configuration is the pair — one large project per pair, matching the single-large-project design (owner decision; load/latency rationale). N+1 onboarding = deploy one new pair, shared spine untouched. | Explicit revision of D6 ("one project = one VM-Xa") to name the pairing. Record supersedes, never deletes. |
| **D-deploy-gates** | Deployments verify with a **light smoke suite per component**; the pair's acceptance ALWAYS includes one end-to-end (gateway → manager → queue → executor → report). Full TP batteries remain for changes to the system itself (code, compose topology, credentials contract), not for routine deployments. | Gate rule intact where it matters; under-an-hour goal preserved. |
| (carried) | Container engine stays declared per machine: `config.engine` docker default, **rootless Podman on the execution VM** (ADR-0003 H1b result carried into all new scripts via the `xb_engine()` pattern). | Owner decision: podman deployments stay. |
| (recorded) | Infisical is self-hosted on its own VM (clone of `ubuntu-24-ea-base` 9000, VMID 2100, 2 vCPU / 4 GB / 40 GB per the vm-vault spec). The three shared-VM installs use the same bare base template — no golden-image prerequisite beyond `ubuntu-24-ea-base`. | Keeps the under-an-hour property uniform for shared and project VMs. |

**Secret-model boundary (what stays out of Infisical):** the Infisical
server's own bootstrap credentials (its DB/Redis root secrets) cannot live
inside Infisical — chicken-and-egg. They remain locally staged 0600 env
files on the infisical VM only. Everything downstream of the vault boots —
hindsight, registry TLS/material credentials, project-pair `.env` trees —
reads from Infisical. The `.env` files that remain on disk are **derived
artifacts of a vault read, never hand-authored truth**.

---

## 2. Configuration surface

`deploy-v2/inputs.yaml` extends (rendered by `render-inputs.py` into
`hosts.env`, the single source of truth, as today):

```yaml
machines:
  # ...existing four roles, PLUS:
  - role: vault                  # Infisical — secrets source of truth
    name: vm-vault
    ip: 192.0.2.14
    vmid: 2100

projects:                        # NEW: N project pairs on one estate
  - name: example-project
    tier: xlg                    # isolation_tier per ADR-0002
    vmid_xa: 2010
    ip_xa: 192.0.2.10
    vmid_xb: 2011
    ip_xb: 192.0.2.11
```

Per-project secret material (session token, CoS allow-list credentials,
Hindsight scope key, GitHub App install) lives in a **per-project Infisical
scope**; the pair's install scripts mint/fetch from that scope, and the
D-offboard re-mint rule ("a credential minted for project N never
authenticates project N+1") is enforced by scoping, not by checklist.

---

## 3. Script inventory

```
deploy-v2/shared/infisical/install.sh   # compose: server + Postgres + Redis, TLS, LAN-only policy
deploy-v2/shared/hindsight/install.sh   # refit of the m4-era stack onto the shared-role shape
deploy-v2/shared/registry/install.sh    # refit of the S1-era TLS + basic-auth stack
deploy-v2/project/xa/install.sh         # gateway + manager + CoS profile (final-era state, not milestone replay)
deploy-v2/project/xb/install.sh         # broker + temporal + bridge + worker/lifecycle + executors (podman-capable)
deploy-v2/deploy.sh                     # orchestrator: dependency order, --only, --project, evidence dirs
```

Every script: idempotent (safe re-run), sources `hosts.env` only, writes
its own smoke evidence under `deploy-v2/<role>/evidence/`, and exits
non-zero on first failed smoke assertion. Every vm-xb command branches
through `xb_engine()` (docker | rootless podman). Estate IPs enter tracked
files ONLY as RFC 5737 placeholders per `IP-PLACEHOLDER-MAPPING.md`.

## 4. Smoke suites (D-deploy-gates)

| Component | Smoke assertions |
| :-- | :-- |
| infisical | API `/healthz` 200; auth round-trip (service token → secret read → write → read-back); TLS cert valid |
| hindsight | `/healthz` 200; scope write → pointer recall → read-back; pgvector store reachable |
| registry | TLS login 200; auth-negative 401; push/pull/delete round-trip; plain HTTP 400 |
| project/xa | gateway :8083 200; CoS container healthy with vault-sourced credentials; desktop connection check (manual step recorded in evidence) |
| project/xb | broker queue declared + ≥1 consumer; temporal healthy; executor spawn → report `status: success` → zero survivors (THE end-to-end acceptance) |
| deploy.sh | emits a per-component verdict table; estate summary = all components PASS |

## 5. Build plan (gated, ADR-0003 style)

| Phase | Work | Gate |
| :-- | :-- | :-- |
| **P0** | This ADR + decision-log + WORKING-STATE (owner commit call) | — |
| **P1** | Infisical VM: clone from `ubuntu-24-ea-base`, static IP, UFW, compose (server+PG+Redis), TLS, org/project bootstrap, per-project scopes created | TP-P1: infisical smoke suite + negative (no-token auth fails 401) |
| **P2** | `shared/hindsight` + `shared/registry` refits to the role shape; credential reads switch to Infisical (bootstrap exception documented) | TP-P2: both smoke suites on the LIVE shared VMs (no downtime to running projects) |
| **P3** | `project/xa` + `project/xb` role scripts producing final-era state (M3/M5-era executor config, podman path, H2 wrapper) directly | TP-P3: both scripts re-run idempotent on the LIVE VM-Xa/VM-Xb pair with zero drift (diff staged compose before/after) + full pair smoke |
| **P4** | `deploy.sh` orchestrator + per-component verdicts | TP-P4: orchestrator run against the live estate, all-PASS table |
| **P5** | First 5-VM clean deployment (from-scratch: 3 shared VMs + fresh pair) using ONLY the new scripts | TP-P5: full smoke suite green, end-to-end report success, total wall-clock recorded (target: < 1 h) |
| **P6** | N+1 onboarding: one new pair via `deploy.sh --project <name>`; book ch14/ch16 revisions | TP-P6: pair smoke green; onboarding recipe exercised end-to-end |

**Sequencing rationale:** P1 first because everything needs secrets (owner
directive). P2 refits the proven shared stacks before touching the project
path. P3 is the largest risk (final-era state assembly) and is verified
IDEMPOTENTLY on the live pair before any clean deployment. P5 is the
promise-under-test. P6 closes the loop the owner asked for.

## 6. Consequences and risks

- **Final-era assembly is the hard part of P3.** The m0–m5 install scripts
  each wrote a *milestone-era* shape; the role scripts must emit the final
  state (executor default, engine transform, H2 wrapper, egress policy
  pins) in one pass. The idempotent re-run gate (TP-P3) is the control
  that catches drift — it exists precisely because of ledger 24.
- **Live-estate coexistence unchanged.** Nothing here touches the running
  V2 stack's compose names, queues, or volumes; refits swap the *install
  path*, not the deployed state. Any exception gets a dated decision-log
  row before the change.
- **Capacity note (review finding F9, still open):** one Xb per pair
  concentrates broker + Temporal + up to 5 executors on one VM. The pair
  model does not change that concentration; the §9.3 soak gate remains the
  scaling control.
- **Book timing:** ch14/ch16 revisions land at P6, after the process is
  proven working (standing rule: the book records what the build proved).
