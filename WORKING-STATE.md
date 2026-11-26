# WORKING-STATE.md — ephemeral-agents-v2 session bootstrap

> **Read this first in every new session.** This file is the living continuity
> record: current state, decisions, open questions, next actions. Update it in
> the same turn a decision is made or a step completes. Context lives HERE
> (cheap, read once) — not in the conversation (expensive, resent every turn).

*Last updated: 2026-11-23 (seventeenth) — Second-Edition reorganization
complete; deploy-v2 transform
COMPLETE: 33-file tree ea-named end-to-end (paths repointed to
`deploy-v2/mN`, broker user `ea:`, ea-tasks queue family, ea labels,
`ea/executor-stub`, demo repo `ephemeral-v2-demo`). Coexistence
amendments: m0 §3/§4 reuse live V1 broker/Temporal when present;
§1 no longer renames hostnames on already-provisioned VMs
(`vm-hermes-agent`/`vm-ephemeral-agents` kept). m0 Tier-1 harness
9/9 PASS (stub now records PUTs; `5 declares + 1 probe` assertions).
services/ response-queue sync to `ea-tasks.response` — all 5 modules
build+test green. Ready to commit, then staged build stages 1–2.)*

*Prior: 2026-11-13 (fifth) — S4 OPEN. Owner chose `deploy-v2/` as the
V2 deploy-tree name. Tree created (33 files, ea-named); broker wiring
done (`ea` user + permissions, `ea-tasks.dlq` declared,
`agent.tasks.#` binding verified on the live broker). Credential sweep
miss found during the S4 wiring review and fixed: the tracked
`deploy/m1|2/m5` composes carried a literal 16-char broker password in
`RABBITMQ_URL` (stale V1-era value — does NOT match the live broker
password; live value only in gitignored .env) — now parameterized
`${RABBITMQ_PASS:?set RABBITMQ_PASS in .env}` in V2 (`0734c57`), V1
backport `c2649a7`; deploy-v2 copies hardened (`5deb77c` + ledger
entry 5). Verified byte-level: zero credential bytes remain in either
repo's tracked files.)*

*Prior: 2026-11-13 — BUILD PHASE DECLARED OPEN (owner, this
session). Rename estate half steps 1–2 EXECUTED: registry re-tag
3/3 digest-identical (correction: only 3 V1 images were ever
registry-resident — the other 6 of the listed 9 were built at install
time, never pushed; `rename-to-ea.sh` IMAGES list corrected), stale
`agent.tasks.dlq` (35 msgs) deleted, `ea-tasks` declared durable with
`x-max-priority: 3` (the single OQ4 declaration point). V1
`agent.tasks.*` queues untouched pending switch-over. Step 3 (baseline
path under ea names) defers to staged-build stages 1–2. Registry VM
recovered: it was up but had shifted address after downtime (DHCP
lease move — pinned-static item now a build-start security item);
drive-cycled once. HEAD pushed after rename-estate commit.)*

*Addendum (same day, `d214a4b`): final gap analysis
(`version 2/V2-gap-analysis.md` — 20 gaps, all enumerated/classified/
owned, none unknown) and feasibility study
(`version 2/V2-feasibility-study.md` — FR-1..9 + NFR-1..9 each evidenced;
verdict feasible/useful/successful, recommends build start). Design-side
record now complete; the declare-start gate stands.*

## Current position

- **V2 build phase: OPEN.** Repo-side half of the `ea-` rename micro-gate
  done (2026-11-11, `d2afc5d`): mapping table at
  `version 2/ea-rename-mapping.md`; grep audit over V2-owned trees clean
  (the deploy layer is born `ea-`-named; the estate's pre-rename names are
  documented in the mapping table per §1 of the platform design). Estate half is separately gated
  and pending owner go-ahead: registry re-tag of the 9 images under
  `ea/…` (digest equality), queue re-create under `ea-*` with the single
  `x-max-priority` declaration, baseline test path re-run green under
  `ea` names. Then: file-level impact inventory + endpoint table
  (§9.5) → build-start security items (§9.1) → staged build → M1.

- **Second-Edition rule (owner, 2026-11-18):** this repo is a standalone
  Second Edition; version 1 (the POC) will be archived and unavailable —
  no references or pointers to it remain anywhere in this repo. The
  repo's own record (design docs, book, deploy layer, decision log) is
  the complete provenance.
- **V2** (this repo, private, LIVE): the live iteration — design record,
  deploy layer, schemas, services. Ready to begin.
- **Estate** (version-agnostic hardware, healthy as of 2026-11-08):
  - front-of-house VM: gateway (:8080) + manager only — deterministic Go, no
    agent (deliberate boundary)
  - execution VM: rabbitmq, temporal+pg, temporal-ui (:8080), broker mgmt UI
    (:15672, LAN-open by owner decision 2026-11-08), worker, bridge,
    lifecycle, llm-proxy (v3, mounts from /opt/poc/llm-proxy/)
  - memory VM: hindsight API :8081; registry VM: private registry :5000
  - post-reboot bring-up: `bash deploy-v2/restart-stack.sh <execution-host>`
  - **2026-11-13 (second) — S0 CHASSIS EXECUTED + fixed.** Owner §9.1
  decisions at build start: dashboard control = interface binding
  (no schema touch); registry transport = TLS + basic-auth primary;
  coexistence = preserve V1 credentials (broker pass reused, temporal
  aligned via socket-auth ALTER USER to the fresh V2 pass). S0 apply
  surfaced 4 issues, all fixed and logged in the new
  `version 2/V2-issue-ledger.md` (netplan network-root bug; temporal
  compose-declared on poc-net ONLY — rabbitmq-pattern, dual-network DNS
  SERVFAIL; credential-regen vs persistent volume; registry VM DHCP
  drift). Temporal healthy on poc-net, worker registered on poc-tasks,
  bridge consuming, DLQ drained (3 benign `m0-cross-test` smoke
  publishes the bridge rightly rejected). Registry VM: `vm-registry`
  ssh alias added; static netplan pin needs ONE owner-run command (its
  user lacks passwordless sudo) — S0 close-out = first reboot-and-reach.
  S0 evidence: WORKING-STATE shows HEAD `9ad275f`+S0-notes. Next: S1
  registry TLS + basic-auth (compose.tls.yaml, .env.registry, GC
  keep-last-N), then S2 images (rename step1 already green 3/3), S3+.
- **2026-11-13 (fourth) — S2/S3 CLOSED.** S2 gate green: 3/3 ea-images
  digest-identical to their poc sources (re-verified from manifest
  headers); the other 6 images are install-built by the V2 deploy layer
  (never registry-resident in V1 — nothing to re-tag). Interim
  restart-policy flip done via `docker update --restart unless-stopped`
  on all 8 live containers across vm-xb + memory VM (no V1 evidence
  files mutated); the V2 layer bakes it in compose. S3 gate green:
  cross-VM AMQP publish test — publish from vm-xa (the manager path)
  over :5672/15672 to `ea-tasks` returned routed:true, message landed
  (queue depth 1), fetched+acked with correct payload. Note: the
  rabbitmq management API's queue-depth stats lag briefly after a
  publish — rabbitmqctl is authoritative. **Open owner decision before
  S4:** the V2 deploy-tree directory name (the docs say "born
  ea-named"; the impact inventory used `deploy-v2/` as a placeholder —
  owner decides the real name before the first S4 install script).
- **2026-11-13 (third) — S1 EXECUTED + verified.** Registry VM: TLS
  (internal CA, leaf cert IP-SAN 192.0.2.13) + bcrypt htpasswd
  (passlib; registry v3.1.2 requires bcrypt — the passlib default
  $apr1$ scheme was rejected) + REGISTRY_HTTP_TLS_CERTIFICATE (v3 env
  var name — the v2-style CERT name is silently ignored) + delete
  enabled. Registry data preserved (existing bind ./data, 2.2G).
  Consumers: CA cert distributed + docker login verified on vm-xa,
  vm-xb, vm-hindsight. Workstation: docker daemon still needs one
  owner sudo command for /etc/docker/certs.d, but the workstation is
  fully validated via the rootless podman context (CA in
  ~/.config/containers/certs.d/192.0.2.13:5000/ — no sudo): login,
  pull, push, delete all green (podman rejects underscore-leading
  repo names where docker accepted them — noted). Tests green: pull with creds (200), auth-negative
  (401), push+delete (202, test-s1 image deleted), plain HTTP 400.
  Registry UI: joxit via nginx proxy_pass to https://registry:5000
  with fixed basic-auth header (NGINX_PROXY_HEADER_Authorization) —
  catalog 200 through the proxy, UI page 200 from the workstation.
  GC: weekly cron (Mon 03:00) running registry garbage-collect against
  /etc/distribution/config.yml (v3 path) + disk alert at 80% (current
  45%). Repo template corrected to match reality (CERTIFICATE env var
  name, ./data bind path, pinned storage root). Next: S2 images, S3+.
- **2026-11-13 estate recovery + pre-start sweep (owner-requested fix):**
    ran `deploy/restart-stack.sh vm-xb` — temporal+pg back up (healthy),
    worker registered on `poc-tasks`, bridge consuming, dashboards 200;
    root cause: the temporal stack (V1-era, `restart: no`) never came back
    after the host reboot and its first auto-start raced its own DB (one
    manual `docker start` cleared it). Memory VM: hindsight-pg had the
    same orphan (exited 255, 38 h) — brought up, healthy, `/healthz` 200.
    Registry VM verified (API 200). Residuals noted: temporal stack still
    `restart: no` (V2 build bakes `unless-stopped` — interim flip offered);
    `agent.tasks.dlq` holds 35 stale V1 messages (drained at rename
    step 2); no ssh alias exists for the registry VM (IP-only access);
    chassis inventory re-verify, registry TLS/auth, secret staging, and
    the desktop registry entry remain the open declare-start items.
    gh auth verified (`arvisdev`). Desktop-gateway-connection setup was
    started early at owner request and **parked at owner direction
    (2026-11-13)** until the build completes: the desktop side needs a
    running `hermes serve` CoS backend on VM-Xa (built at stage S7), so
    the connection is created then — procedure already pinned in the S7
    runbook row (`9712193`, from the official multi-connection-desktop
    docs).
    (order: engine+store → broker+bridge → worker+lifecycle → dashboards)

## Active milestone — V2-M1: the Chief-of-Staff agent

Persistent resident Hermes agent with the bus-org chief-of-staff persona on
the front-of-house VM, between the unchanged gateway and manager. **The agent
proposes, the code disposes** — the manager validates every proposal.

### Decided (2026-11-07/08)

1. Pipeline: desktop → gateway (unchanged) → CoS (new) → manager (unchanged,
   still validates) → queue → ephemeral executor → report
2. Persistent resident agent, NOT the ephemeral `eph-chief-of-staff` worker
   (that roster entry stays for principals needing coordination artifacts)
3. No new VM, no new credentials class — CoS submits through the gateway API
4. CoS hydrates like eph-workers do: persona files, llm-wiki, Hindsight —
   the hydration contract gains its first persistent consumer
5. V1 stays frozen; this milestone lives only here; its own TP + gates
   required before it counts (gate rule carried forward from V1)

### Open questions

- **Q1 — desktop↔CoS connection:** REVISED 2026-11-09 → native Hermes
  Desktop gateway registry (named remote connections, session-token auth);
  the bespoke HTTP API of the morning draft is rejected.
- **Q2 — approval mode:** ANSWERED 2026-11-09 → ask-before-submit.
- **2026-11-09 design-v2 decisions D3–D11** (runtime shape, repo as source
   of truth, desktop stateless, one CoS per project VM, management model,
   priority field, parallelism, change management) — recorded in
   `version 2/V2-M1-cos-design.md`; plus OQ1–OQ4 there.
- **2026-11-09 (second):** harness set stays at two (claude-code + hermes;
   third-harness criteria in design §8); VM-Xb carries over unchanged for
   the V2 test run (design §9); V2 gets its own demo repo. Model
   specification & configurables = deferred topic.
- **2026-11-09 (third):** single-repo-per-project default convention with
   per-project manifest (design §10); multi-flow concurrency resolved —
   worktrees rejected, M1 sequential same-repo, later scope-declaration
   milestone (design §11). Deferred topics queued: model specification &
   configurables, secure key management.
- **2026-11-09 (fourth):** 9-item open-items queue adopted, worked
   one-by-one: (1) model spec & configurables — OPENED AND RESET at owner
   direction, restarts from scratch later; (2) secure key management —
   deferred; (3) isolation level; (4) embeddings provider; (5) wiki
   curation automation; (6) dependency refresh policy; (7) umbrella
   compose; (8) contract test suite; (9) production hardening.
- **2026-11-10 platform decisions (owner), recorded in
  `version 2/V2-platform-design.md` (DRAFT v1):**
  (a) **`ea-` naming family** replaces all V1 "poc" names (`ea-net`,
  `ea/hermes-ephemeral`, `ea/claude-coder`, `EA_*` envs) — decided now,
  executed as the opening action of the V2 build phase; baseline
  `deploy/` keeps V1 names until then; docs reference only V2 names
  after the rename. (b) **Memory pointer-store convention:** documents
  live only in the project repo; Hindsight stores pointers only; CoS
  writes, memory VM stores, repo is content. (c) **Concurrency default
  5** ephemeral agents in flight, manifest-configurable
  (`max_parallel_agents`); M1 same-repo sequential (design §11).
  (d) **Embeddings:** evidence-driven — zero-vector fallback for M1;
  small self-hosted model on the memory VM only if pointer recall
  proves poor. (e) **Queue policy:** message-v1 additive-only evolution;
  breaking change ⇒ versioned message-v2; new message types as gated
  additions. (f) **Isolation ladder:** A (containers) for M1; A+
  (rootless runtime + egress allow-list) as a separately-gated change
  right after M1's gate; B (gVisor) / C (micro-VM) trigger-gated.
  (g) **V1 lessons pre-applied** in the V2 build, lesson→build-step
  mapped (platform-design §7). Queue items 3 and 4 are thereby resolved
  as policy; item 3's A+ implementation is a post-M1 build milestone.
- **2026-11-10 (second): END-TO-END DESIGN REVIEW performed and findings
  ACCEPTED (owner).** 28 findings (4 critical / 10 major / 9 minor /
  5 observation), verdict **approve with conditions**. Applied in the
  docs: schema-evolution mechanics corrected (platform §5); CoS state
  persistence specified as `cos-state/state.json` in the project repo
  per-transition (M1 §6.8); failure behaviors completed (M1 §6); gate
  assertions 9–12 added (no-submit-without-approval, approval-payload
  hash pinning, credential capability check, desktop reconnect) + the
  judgment corpus as a non-signed pre-gate eval (M1 §7); registry
  TLS+digest pinning, llm-proxy/Hindsight auth, dashboard publisher
  control, token custody pulled forward as build-start controls
  (platform §9.1); A+ re-specified as segmentation + rootless, both
  gate-asserted (§9.2); capacity model + soak test (§9.3); CoS
  observability minimum (§9.3); D6 supersedes the 2026-11-08 "no new VM"
  note for multi-project growth (§9.4); rename-before-OQ4 ordering +
  rename micro-gate (§9.7). New build-phase commitments (owner
  directive): **file-level impact inventory** before the first milestone
  (§9.5) and the **full V1-style build process with a V2 issue ledger**
  (§9.6). **V2 book timing decided (owner):** the V2 book is written
  after the full installation/deployment is proven working — the closing
  deliverable, containing V2's findings and corrective actions from the
  record. **Second-pass review performed (2026-11-10, third): verdict
  ready-with-conditions; report persisted at `version 2/audits/2026-11-10-design-review.md`.**
  Convergent critical: the cos-state/state.json mechanism (M1 §6.8)
  contradicts the §5 CoS credential contract (no GitHub token) — owner
  decision required: scoped CoS repo credential (amend §5) OR local
  volume + Hindsight pointer (amend §6.8). Plus: approval-hash verifier
  must be named (manager; enumerated gated change), assertion 11 marked
  conditional to A+, assertion 2 OQ4-conditional, A+ memory-hydration +
  Hindsight token-split decisions recorded, doc drift fixed. Next gate:
  owner rules on the credential question + conditions applied, then the
  build phase opens with the rename + inventory.
- **2026-11-10 (fourth):** V2 feature list created — `version 2/V2-feature-list.md`
  (DRAFT v1): the full V2 offer as a status-marked feature inventory
  (decided / trigger-gated / deferred) across intake, pipeline, work
  management, memory, security/isolation, operations, and documentation,
  closing with the one-line offer.
- **2026-11-10 (fourth): D12 ADOPTED (owner) — kanban board as the CoS
  work board.** One board per project CoS; card = work package; columns
  proposed → approved → submitted → in-flight → reported → rolled up;
  report events move cards. **Desk, never the door:** queue is the only
  execution truth; cards reach "submitted" only via the approval flow
  (gate-asserted, TP 9 amended). **Resolves the second-pass review's
  condition 1 (critical):** the board replaces `cos-state/state.json` as
  the CoS state store of record — lives on VM-Xa with the profile,
  reboot-durable, no GitHub credential needed; repo stays source of
  truth for project content. Board history = submit-side audit trail;
  desktop board visibility = build-time verification item. Remaining
  review conditions: (2) reconcile assertions 2/3/10/11 (name the
  approval-hash verifier, mark 11 conditional to A+), (3) record A+
  memory-hydration + Hindsight token-split choices, (4) doc drift.
- **2026-11-10 (fifth): THIRD-PASS REVIEW performed** (report updated in
  `version 2/audits/2026-11-10-design-review.md`). D12 confirmed correct by both
  panels; verdict **ready-with-conditions (narrowing)**. Five conditions
  before the build gate: (1) restate §5's credential allow-list (the CoS
  now holds a Hindsight write token + llm-proxy CoS-class token —
  "exactly three credentials" is stale and a conforming build would fail
  assertion 6); (2) fix the approval-hash mechanism (message-v1 hash
  field; M1 manager check = retry integrity + mismatch-reject; §7.10
  reworded to tamper-evidence — a message-carried hash cannot prevent
  mutation; canonicalization defined; gateway-side approval registration
  = later enforcement change); (3) board durability contract (storage as
  build-time verification item, backup/export, fail-safe-to-no-submit on
  unreadable/ambiguous state, write-ahead ordering, corruption case in
  TP 7); (4) make the judgment corpus runnable (fixtures, location,
  machine-checkable fields, threshold — carried through both passes);
  (5) drift pass (audit-trail provenance: board = intent trail,
  gateway/manager log = enforcement trail; board/queue reconcile +
  terminal card states; TP board-inspection path; advisory attribution
  header; A+ hydration re-verification + CoS as the memory write-back
  actor; new CoS→VM-Xb llm-proxy cross-VM dependency in the endpoint
  table; M1 §1 "unchanged" wording; stale next-action items).
- **2026-11-10 (sixth): DE-POC SWEEP (owner directive).** Removed the two
  root POC documents (`Initial-POC-Findings-Report.md`,
  `POC-Retrospective-and-Insights.md`) — they live verbatim in the frozen
  V1 record (HEAD `2c97797…`, verified); README + book/README pointers
  now cite that location; V2's own findings report + issue ledger are
  build-phase closing deliverables. Retained deliberately: `book/`
  (V1-record provenance) and `deploy/` (V1 evidence baseline; V1 names
  until the `ea-` rename micro-gate). `schemas/message-v1.json` metadata
  de-POC'd (title + description; no contract semantics touched — recorded
  as the §5 "first touch").
- **2026-11-10 (seventh): D12-REV — TWO-TIER WORK BOARD (owner).** Tier 1
  master plan on an external tracker (GitHub Issues default, ADO/Jira
  swap-in; exporter holds the credential, CoS never does) = plan of
  record, human-reviewable, intent audit trail; Tier 2 local Hermes board
  = the pulled working subset driven through approval → submit → report.
  Tracker = plan truth; queue = execution truth; local board = cache.
  Desk-never-door unchanged; fail-safe recovery; write-ahead ordering.
  Full record: `version 2/adr-0001-cos-work-board.md` (ADR-0001 — first
  of a standing ADR class; owner directive: the V2 book gives the
  work-board/approval areas special plain-language attention). Conditions
  1–3 of the third-pass review now applied (§5 allow-list; honest
  approval-hash = retry integrity + tamper-evidence; board durability
  contract). Remaining: condition 4 (runnable judgment corpus) and 5
  (drift pass). Both applied — see the decision-log (eighth) entry.
- **2026-11-10 (ninth): OQ1/OQ4 CLOSED (owner) — no open questions block
  the build.** OQ1: `priority` encoding = coarse enum `high`/`normal`/
  `low` (manager rejects out-of-range; logs the declared value). OQ4:
  the priority field lands **with M1** (field+consumer same gated
  change; `x-max-priority` declared once under the `ea-*` name during
  the rename's queue re-creation). TP assertions 2 and 3 now
  unconditional. **The build phase may open** — sequence: `ea-` rename
  micro-gate → file-level impact inventory + endpoint table →
  build-start security items (registry TLS, service tokens) → staged
  build → M1 / TP-V2-M1.

### Next actions

1. `version 2/V2-M1-cos-design.md` rewritten as DRAFT v2 — **owner review /
   continue design discussion is the next gate**
2. Build: VM-Xa Hermes Agent profile with the CoS persona, desktop gateway
   registration, message-v1 `priority` extension (restart:
   unless-stopped, persona from roster repo, reboot-safe per ch14 §14.8)
3. Gate: TP-V2-M1 — **as amended** (see the M1 design doc §7): structural
   assertions 1–12 including no-submit-without-approval, approval-payload
   hash verification, board-is-not-the-door, credential env + (A+-deferred)
   capability checks, priority (coarse 3-level, with M1 — OQ1/OQ4 closed),
   the judgment corpus as a non-signed pre-gate eval — plus one end-to-end
   (market-analysis example → packages → reports). Prerequisite
   artifacts: the contract test suite and the file-level impact inventory
   (platform §9.5/§9.6).
4. Later open items (from the early decision log) — *status-corrected per the
   third-pass drift pass:* concurrency beyond sequential (resolved:
   manifest-configurable, default 5; scope-declaration milestone later);
   embeddings provider (resolved as policy: evidence-triggered);
   reboot-surviving stack and contract test suite (**subsumed** — platform
   §9.2/§9.6, no longer separate queue items); dedicated V2 demo repo (+
   1-min GitHub App install when coder-style work arrives); key vault
   (D-secret-vault) on the front-of-house VM (owner-deferred)


## Sixteenth update — live-estate sequence complete: S1, m1–m5, TP-M5 PASS (2026-11-22)

- **The full milestone sequence re-ran on the LIVE estate** (vm-xa .233 / vm-xb
  .224 / vm-hindsight .223 / vm-registry .198) and closed: S1 registry TLS gate,
  m1 T1–T5, m2 V1–V5, m3 B1–B5, m4 G1–G6 (commits 564250d..80344c3), and
  **m5/TP-M5 PASS** (commit `caf9f9d`): E2E task `83229c03` success via the real
  hermes executor, F1–F5 (F1/F2/F5 PASS; F3/F4 CHECK→recover-and-complete),
  trivy 9/9 clean, token issue→200→revoke→401. Evidence:
  `deploy-v2/m5/evidence/` (`tp-m5-results.md` live pack; the test-estate pack
  preserved as `*-test-estate.*`).
- **Ledger 17 (layer cache):** a cached apt RUN layer ships the build-day
  package snapshot — the needrestart CVE survived a "rebuild". Rule: a
  CVE-remediation rebuild must bust the base-image layer cache.
- **Ledger 18 (privilege path):** §6 minted its token under sudo but revoked as
  poc (root-owned state file unreadable) — silent `set -e` death. Rule: gate
  steps sharing state must share a privilege path.
- **Repo hygiene:** m0 decision-log had 33 duplicate install-run blocks
  (append-loop bug) — deduped; `inputs-test.yaml` (MAC-corrected test inventory,
  901=vm-xlg-xa/.203, 902=registry/.200, 903=exec/.202, 904=hindsight/.201)
  recovered from session records and committed (`afa9420`).
- **Estate:** all 4 live VMs were found STOPPED on the PVE host — booted
  2000/1000/2001/1001; the m0-era Temporal stack and hindsight-pg had the known
  reboot-orphan race, restarted in dependency order, healthy. Test clones
  901–904 stay stopped.
- **Remaining:** TP-M1-COS only — prereq is the owner-side step (add the Hermes
  Desktop gateway connection `http://192.0.2.10:8084` with the
  `COS_SESSION_TOKEN` from `/opt/ea/deploy-v2/m5/.env` on vm-xa), then
  `deploy-v2/m1-cos/install.sh --gate`.


## Seventeenth update — the CoS milestone closed; TP-M1-COS signed (2026-11-23)

- **TP-M1-COS PASS and signed** (`deploy-v2/m1-cos/evidence/tp-m1-cos-results.md`):
  judgment corpus 13 fixtures / 0 errors; the manager approval-hash suite
  17/17 (retry integrity, mismatch-reject, whitespace-exactness); the
  interactive assertions exercised by the owner's real desktop use of the
  CoS (live analysis request over the `ea-cos` gateway connection);
  desktop reconnect proven across the 2026-11-22 full estate restart.
- **Ledger 20 (gate-caught defect):** the D12 board state lived in the
  container FS — a recreate wiped it. Fixed: `HERMES_HOME=/state` in the
  cos compose + entrypoint; state migrated live (2.9 GB, no loss).
  Rule: a durable-mount claim must name the mechanism and the env that
  activates it, asserted by the compose.
- **The full V2 milestone sequence is complete:** S1 registry TLS, M0–M5
  (live estate, TP-M5 PASS), and the CoS milestone (TP-M1-COS PASS).
- **Publication prep in progress:** real LAN addresses swept to RFC 5737
  placeholders across files AND full history (`git filter-repo`; mapping
  table at `deploy-v2/IP-PLACEHOLDER-MAPPING.md`; all commit hashes
  changed — HEAD `18c9a24`). Remaining before the visibility flip:
  owner's two topology diagrams (ch11/ch14 slots — post-publication OK),
  AUTHORSHIP V2-era sign-off line, README honesty pass.

## Working protocol (cost reduction — ENABLED 2026-11-08, owner directive)

1. **Bootstrap-by-document:** every session starts by reading this file.
   Decisions and completions are written here IN THE SAME TURN they happen.
2. **One session per workstream;** end sessions at milestone boundaries.
   Retire long sessions freely — continuity lives here, not in the chat.
3. **Delegate bulk work** (repo-wide scans, log analysis, multi-file edits,
   estate inspections) to subagents — they consume their own context; only
   compact summaries return to the main conversation.
4. **In-session output discipline:** targeted queries over full dumps;
   aggregate programmatically and print verdicts; batch tool calls and
   owner questions.
5. **Model discipline:** economical default model; expensive reasoning is the
   exception. Check provider prompt-caching eligibility to flatten the
   resend-cost curve.
6. **De-identification standing rule:** no IPs, usernames, passwords, or API
   keys in any tracked file (placeholders + gitignored env files); the
   Apache copyright headers / LICENSE / NOTICE / AUTHORSHIP author line are
   the deliberate exceptions.

## Repo map (V2)

```
WORKING-STATE.md      this file — read first every session
version 2/            V2 design docs (V2-M1 design lands here)
version 2/book/       Second Edition book (15 chapters)
deploy-v2/            V2 deploy layer (ea-named; m0-m5 + m1-cos)
schemas/              message-v1 contract
decision-log.md       V2 decision log (append-only)
services/             Go services (evolve here as milestones require)
```

## Seventh update — S4 staged build: m0 apply green + m1 (topic-bridge) install + gate T1–T5 PASS (2026-11-14)

- **m0 apply green** (commit `1f98b82`): cross-VM publish test rewritten V2-native — publish from vm-xa via AMQP 5672, valid message-v1 payload, fetch+ack from `ea-tasks` via rabbitmqctl; net-zero V1 pollution (the earlier "valid payload would start a real V1 workflow" concern resolved: the bridge tests only assert the V2 path, and the T3 kill target is the V2 container). Also fixed: coexistence-branch `RABBITMQ_PASS` load, `--no-gate` flag now implies `ASSUME_YES`, netplan `network:` root. All 5 m0 gates PASS; both `ea-tasks` and `agent.tasks.dlq` drained to 0 post-test.
- **m1 install + gate T1–T5 PASS** (commit `2c5cedf`): V2 bridge `ea/topic-bridge:1.0.0` deployed as compose project `ea-m1` on vm-xb (container `ea-m1-topic-bridge-1`), consuming `ea-tasks` (1 consumer). T1 50→50 PASS, T2 dup +0 PASS, T3 kill-9 +20 PASS (redelivery, no doubles), T4 poison +3 DLQ PASS, T5 temporal-restart +5 PASS. Evidence: `deploy-v2/m1/evidence/m1-tests.txt`.
- **Fixes made during the run:**
  - `services/topic-bridge/cmd/adapter/main.go`: queue declare for `ea-tasks` now passes `x-max-priority: 3` — inequivalent re-declare was rejected with PRECONDITION_FAILED and the bridge could never consume.
  - `m1/install.sh`: §0 health check greps `SERVING` (CLI output) and exports `TEMPORAL_ADDRESS` for the CLI; T3 kills `ea-m1-topic-bridge-1` (was the V1 container name).
  - `.gitignore`: `services/**/adapter` masked the topic-bridge source tree (`cmd/adapter/`) — added `!services/**/cmd/adapter/` negation; adapter source now tracked (`cmd/adapter/main.go`, `adapter_test.go`).
  - `services/topic-bridge/schemas/message-v1.json` re-copied from the authoritative `schemas/message-v1.json` (drift guard caught it).
- **Estate state after the run**: V1 bridge `m1-topic-bridge-1` still up, consuming `agent.tasks.durable` (V1 untouched); `ea-tasks.dlq` holds the 3 T4 poison pills (expected, correct DLQ routing); `agent.tasks.dlq` 0 — no V1 pollution. Both brokers share the `agent.tasks` exchange; V1's `agent.tasks.durable`/`agent.tasks.dlq` and V2's `ea-tasks`/`ea-tasks.dlq` are distinct consumers.
- **Next**: run TP-M1 with file evidence from `deploy-v2/m1/evidence/`, then M2 install (executor-stub → worker lifecycle).

## Eighth update — TP-M1 signed (2026-11-14)

- **TP-M1 gate: PASS** — all 6 mandatory cases PASS (TC-M1-01 poison-pill→DLQ with reason property, TC-M1-02 50→50, TC-M1-03 duplicate idempotency +0, TC-M1-04 kill-9 no loss, TC-M1-05 Temporal restart recovery, TC-M1-06 rollback/re-deploy: 5 messages published while bridge down → durable queue held them, drained to 0 on restart, workflows started). TC-M1-06 was run after T1–T5 as a separate, stronger variant.
- **Evidence pack:** `deploy-v2/m1/evidence/` — `m1-tests.txt` (T1–T5), `m1-dlq-sample.json` (x-reject-reason header), `tp-m1-results.md` (signed results table). Signed on owner's behalf per standing authorization (2026-11-05).
- **M2 installation authorized.** Next: `deploy-v2/m2/install.sh` (executor-stub → worker lifecycle).

## Ninth update — S4 staged build: m2 (lifecycle) install + TP-M2 signed (2026-11-14)

- **m2 install + gate V1–V5 all PASS** (second run): V1 clean, V2 failure path, V3 timeout, V4 crash recovery (load-bearing — independent reaper reaped the orphan), V5 10 mixed runs zero survivors. TC-M2-06 (no-pull rule) run separately: gate refuses to serve without the pinned image, restores cleanly. Evidence: `deploy-v2/m2/evidence/`.
- **TP-M2: PASS — M3 installation authorized.** Signed on owner's behalf per standing authorization. Evidence pack: `tp-m2-results.md`.
- **Root-cause fixed during the gate:** the first m2 gate run failed V1/V5 — the V2 `m2/compose.yaml` carried the M5-era executor default (`EA_EXECUTOR_DEFAULT: hermes`), so M2-era gate tests spawned the real hermes executor (not in scope at M2, exits 2 with unstaged m3.env). Fixed by restoring the M2-era stub-only compose (the M3 install writes the M3-era executor switch per the compose-evolution rule); also fixed m2/install.sh prereq (`SERVING` + `TEMPORAL_ADDRESS`), T-container names (`ea-m2-*`), and pre-staged `/opt/ea/secrets/m3.env` + m3 mount dirs for the M3-era compose forward-compat. Recorded in the V2 issue ledger.
- **Estate state:** `ea-m2-lifecycle-1` + `ea-m2-worker-1` on vm-xb (compose `ea-m2`, ea-net); V1 stack untouched. ea-tasks.dlq reset during re-run.
- **Next:** M3 install (executors: hermes-ephemeral + claude-coder, m3.env staging, token issuance) → TP-M3, then M4 (memory), M5 (gateway+manager), TP-V2-M1 closing the staged build.

## Tenth update — S4 staged build: m3 (executors) + m4 (memory) installs, coexistence fix (2026-11-14)

- **M3 complete — TP-M3: PASS (all 7 mandatory cases), signed on owner's behalf.** Evidence in `deploy-v2/m3/evidence/` (boot-time tables p50 1433 ms + reproducibility Δ≈34%, persona match to `agent-roster`, failure-path exits 2/2, coder commit `d36d49f` on `ea-demo-repo` task branch, token issue→revoke→401, trivy secrets = 0). Committed `2adc046`.
- **M4 complete — hindsight deployed on vm-hindsight as project `ea-hindsight` on port 8082; gate G1–G6 all PASS.** G4 proves end-to-end hydration from the fresh V2 store (single t-g1 entry — V1 rows absent → isolation + 8082 routing correct).
- **Coexistence fix (commit `0209ffb`):** the m4 compose initially kept V1's project name `hindsight` and took over V1's stack (V1 containers replaced, V1 data volume reused). Fixed: V2 compose is now `name: ea-hindsight`, containers `ea-hindsight-*`, own volume `ea-hindsight-pg-data` (fresh, initdb), publish `8082:8081`. Restoration sequence executed: takeover stack down → V1 up from `/opt/poc/deploy/m4/hindsight` (8081, healthy) → V2 up (8082, healthy). Both stacks coexist with distinct volumes.
- **install.sh updates:** `HINDSIGHT_PORT` env-overridable (default 8082); UFW rule uses the port var; §2 wiring check now reads m3.env (env_file mechanism) instead of grepping the compose; evidence snippet masks the LLM_WIKI token (middle elided).
- **m3-era env:** `HINDSIGHT_URL=http://192.0.2.12:8082` in `/opt/ea/secrets/m3.env`; lifecycle+worker recreated and verified. Known open item: EMBEDDING_API_KEY unset → zero-vector fallback (D-embed, same as V1).
- **Next:** TP-M4 sign-off from `deploy-v2/m4/evidence/`, then M5 (gateway + manager) → TP-V2-M1 closing the staged build.


## Eleventh update — M4 gate: TP-M4 signed (2026-11-14)

- **TP-M4: PASS — all 7 mandatory cases green, M5 authorized.** Evidence pack in `deploy-v2/m4/evidence/`: readback.txt (TC-01/02 raw JSON, q-driven cross-task recall), retrieval-hits.md (TC-03/06 clone lookups, SHA 425b5bcd9df0 / curation bc1fcd25a7af + CHANGELOG 6776fd1888e2), latency-table.txt (3 runs, totals 0.79–0.92 s ≈ 99.8% TTL headroom), isolation-check.txt (distinct PG engine+host+volume, zero wiki artifacts), tp-m4-results.md (signed table + root-cause ledger).
- **TC-M4-06 curation cycle exercised the D-curation loop end-to-end:** new standard doc (`dev-best-practices/coexistence-port-allocations.md`) pushed to llm-wiki, CHANGELOG entry added (HITL per D-curation), fresh spawned-container clone verified new+old docs readable.
- **Signed on owner's behalf** per standing authorization (all-PASS with evidence).
- V1 untouched throughout (its hindsight stack on 8081 with its own volume; llm-wiki curation is additive content).
- **Next: M5 install (gateway + manager on vm-xa), then TP-V2-M1 closes the staged build.**


## Twelfth update — M5 install: bring-up green, E2E coexistence-race fix in flight (2026-11-14)

- **M5 ea-transform complete and check-only green**: deploy-v2/m5 (compose.yaml, install.sh, e2e-deploy.sh, Dockerfile) ported. Fixes: stale GATEWAY_URL 8080→8083 in both hosts.env (gitignored); sudo mkdir+chown for /opt/ea on vm-xa (m2 pattern); V1's broken §6 token check (literal *** → real issue→200→revoke→401 on the catalog repo); .env staging now writes AGENT_LIST_REPO + BROKER_HOST; a real `set -euo pipefail` bug fixed (`[ -n ... ] && printf` short-circuit inside the .env pipe block killed the script before compose-up); E2E capability "dev" → exact roster ref ephemeral-org/eph-backend-engineer (real 22-agent index has no dev role); compose RESPONSE_QUEUE agent.tasks.response → ea-tasks.response.
- **§1–§3 ALL PASS live**: ea/gateway:1.0.0 + ea/manager:1.0.0 built and deployed on vm-xa (gateway 8083, V1 keeps 8080), UFW 8083 open, e2e-deploy S1–S7 all PASS.
- **§4 E2E root cause (confirmed)**: shared `agent.tasks` exchange — BOTH V1's bridge (m1-topic-bridge-1, poc/topic-bridge, Up) and V2's bridge are bound `agent.tasks.#`; both derive the same WorkflowId from task_id; first-start wins. V1's bridge won for bb41c4cd and cceeb912 → executed on V1's poc-tasks queue (artifact /var/poc/...). The bridge "invalid character 'm'" errors were the m0 gate's plain-text test payloads, a red herring. S7 smoke passed because run-task.sh publishes directly to the ea-tasks queue.
- **Fix applied (code, not yet rebuilt/redeployed)**: services/manager/main.go now publishes DIRECTLY to the ea-tasks queue (default exchange, routing key = queue name; new cfg tasksQueue/EA_TASKS_QUEUE default ea-tasks; exchange declare skipped when empty). V1 untouched. NOT done: manager image rebuild+redeploy, E2E re-run, §5–§7, TP-M5.
- **Unverified flag**: a /tmp/bind.py run on vm-xa printed "bound ea.tasks.# -> ea-tasks" — unintended (vm-xa has no broker); verify the vm-xb broker binding table (agent.tasks ← ea-tasks) before trusting it.
- **Known issue (pre-existing)**: /opt/ea/deploy-v2/m5/.env missing RABBITMQ_PASS on vm-xa? (grep said "No such file" twice mid-session — verify; install writes it at §1).
- Next session: 1) rebuild ea/manager:1.0.0 from services/manager, docker save|ssh vm-xa docker load, compose up -d --force-recreate manager; 2) re-run §4 E2E → expect workflow on ea-tasks + report; 3) §5 failure injections, §6 hardening, §7 docs → TP-M5; 4) record the D-routing coexistence decision (V2 queue-direct publishing) in deploy-v2/m0/decision-log.md; 5) commit everything.
- TP-M4 remains closed 7/7; M5 pending E2E; V1 stack untouched throughout.

## Thirteenth update — M5 COMPLETE: §5–§7 green, TP-M5 signed (2026-11-15)

- **TP-M5: PASS — the V2 staged build (M0–M5) is complete.** Evidence pack:
  `deploy-v2/m5/evidence/` (`failure-injections.txt`, `install-m5.log`,
  `e2e-report-*.json`, `trivy-*`, `tp-m5-results.md` — signed on owner's behalf
  per standing authorization, 2026-11-05).
- **§4 E2E green on the V2 path (queue-direct publish):** hermes-executor
  acceptance runs (d13105db, 17bd78e3, 205b4e2a, ea49dc5b, 93b74eab) all
  COMPLETED with reports written and zero survivors. TC-M5-02(c) commit
  evidence: coder-routed acceptance `7c9a32c` on
  `origin/task/f9e6c71d-25d5-4ad8-aa66-b276a5e9f191` (healthz handler + 39
  lines of tests), produced via `EA_EXECUTOR_DEFAULT=coder` flip through the
  identical gateway→manager→queue path, restored to hermes after.
- **§5 F1–F5:** F1/F2 PASS (nft block with forward-hook — container egress
  verified dead — produces failure-shaped reports); F5 PASS (report after
  manager restart); F3/F4 CHECK→policy **recover-and-complete** (M2 retry
  absorbs crash/TTL; zero survivors) — same CHECK-marks precedent V1 signed.
- **§6 green:** trivy 9/9 clean (accepted residuals recorded), env sweep
  clean, token issue→200→revoke→401.
- **Root causes fixed this gate (issue ledger):** gitignore masked the entire
  gateway/manager source trees (c4e1282's fix never committed, didn't compile)
  → `28b8616`; default-exchange routing key must be the queue name → `28b8616`;
  deployed compose still raced V1 on agent.tasks → `a710d08`; the M5-era
  `EA_EXECUTOR_DEFAULT=hermes` switch was never implemented (§5 silently ran
  the stub executor) → `15d0e15`; §5 runbook stub-era defects (dev capability,
  45 s sleeps, pkill names, nft OUTPUT hook) → `b26a589`, `f2162b7`; §6
  undefined `$tok` under `set -u` + revoke propagation race → `4dad6d0`.
- **Next:** owner topology re-draw in ARCHITECTURE/foundation docs (standing
  directive), then the V2 book (closing deliverable) and the findings report;
  TP-V2-M1 CoS milestone per the standing plan.

## Fourteenth update — closing deliverables: findings report + V2 book first edition (2026-11-15)

- **Findings report written** — `version 2/V2-findings-report.md`: what was
  built, gate table M0–M5, five findings (coexistence as cheapest isolation;
  gates catching every major error pre-ship; stage-gate re-assertion rule;
  display-masking hazard; executor assertability), commit trail of record.
- **Issue ledger extended 5 → 13** — m1 x-max-priority declare (6), m2
  future-era compose (7), m4 V1-takeover (8), m5 gitignore source-masking (9),
  routing-key bug (10), stub-executor §5 (11), runbook stub-era defects (12),
  §6 $tok/propagation (13) — each symptom → wrong assumption → root cause → rule.
- **V2 book first edition complete** — `version 2/book/` (commit `4dc5bd1`):
  15 chapters (V2 deltas for ch4–10; NEW ch11 Two Organizations One Estate —
  the coexistence centerpiece; ch12 failure ledger with the honest 13-entry
  histogram; ch13 what remains; ch14 operator's chapter; ch15 evaluator's
  chapter), the two owner-directed plain-language specials
  (`special-1-who-may-start-work.md`, `special-2-what-does-approved-mean.md`),
  and Part I pointer stubs. All grounded in the committed record,
  de-identified, with owner-drawn diagram slots in ch11/ch14/ch15.
- **Awaiting owner:** the two topology diagrams (ch11 + ch14 slots), then
  the book is final; next milestone TP-V2-M1 (CoS on vm-xa).

## Fifteenth update — Second-Edition reorganization: repo-wide V1-reference sweep (2026-11-18)

- **Owner's rule (2026-11-18):** the Second Edition is standalone. Version 1
  (the POC) will be archived and unavailable; no references or pointers to it
  may remain anywhere in this repo — provenance citations, "V1 @ hash"
  references, and "carried from V1" phrasing included. Every V1-derived
  rule/lesson now reads as this build's own statement of fact, without naming
  version 1 or pointing at any V1 artifact. Live coexistence identifiers on
  the shared estate (the first organization's `poc-*` containers, `poc-net`,
  `POC_*` env vars, `/opt/poc` paths, the shared `agent.tasks` exchange) keep
  their real names but are attributed to the first organization / the
  neighboring stack, never to "V1".
- **Three passes executed (all repo-wide, uncommitted):**
  1. *Book pass:* `version 2/book/` made fully self-contained; the root
     `book/` copy removed; all V1 references rewritten as V2 fact.
  2. *Deploy pass:* the deploy layer consolidated into `deploy-v2/`; the
     V1-baseline `deploy/` tree deleted (evidence confirmed duplicated).
  3. *Sweep pass (this update):* repo-wide grep for `V1|version 1|first
     edition|version 1/|book/ch|poc/|/opt/poc`; every hit categorized —
     contract identifiers kept (`message-v1`), live coexistence identifiers
     kept and re-attributed, dated history left as dated history
     (decision-log.md, deploy-v2/m*/decision-log.md, evidence dirs), all
     other references rewritten as V2 fact (design docs, runbooks, install
     scripts, compose comments, Go comments, README).
- **Standing rule superseded:** the rule that the version-1 record is always
  cited by pinned commit hash is retired — version 1 references are being
  eliminated from the repo entirely. The repo's own record is the provenance.
- **Verification:** final grep clean except the categorized exceptions
  (message-v1 contract identifier, dated-history files, live coexistence
  identifiers, M2 gate test-case names V1–V5).

Seventeenth update (2026-11-23): post-publication architecture audit
recorded at `version 2/audits/2026-11-23-architecture-audit.md`
(7-law review; 4 compliant, 3 P1 — egress filtering/A+ rung,
memory-injection governance, continuous evaluation; remediation roadmap
phases recorded). Aligns with the existing post-publication backlog:
A+ isolation was already first.

Eighteenth update (2026-11-23): audits folder convention established.
Independent review/audit documents now live in `version 2/audits/` with
date-led filenames: `2026-11-10-design-review.md` (the multi-pass design
review, previously V2-design-review-report.md) and
`2026-11-23-architecture-audit.md` (the 7-law post-publication audit).
Build-record documents (V2-findings-report.md, V2-issue-ledger.md) stay
in `version 2/` — the record/review distinction is deliberate. Future
audits land in `audits/` by convention, dated in the filename.

Nineteenth update (2026-11-24): ADR-0003 H1a executed — executor egress
allow-list live on the execution VM (DOCKER-USER policy, systemd-persisted;
TP-H1a all-PASS, evidence `deploy-v2/h1/evidence/`, decision D-APlus-egress).
llm-proxy found down (container lost at reboot) and recreated from the
repo-staged source; ledger follow-up opened to give it a compose/unit home.
Remaining hardening sequence per ADR-0003: H2 memory governance, H3
continuous evaluation, H1b rootless Podman.

Twentieth update (2026-11-24): ADR-0003 H2 executed — memory-injection
governance (D-mem-governance, TP-H2 all-PASS): untrusted-memory wrapper in
the executor hydration contract, CoS rules 9–11 (untrusted data, 90-day
staleness re-validation, CoS-only load-bearing writes) deployed live to the
CoS profile, platform design §2 amended, canary injection test evidence in
`deploy-v2/h2/`. Executor image rebuild deferred to H1b. Remaining: H3
continuous evaluation, H1b rootless Podman.

Twenty-first update (2026-11-24): ADR-0003 H3 executed — continuous
evaluation (D-eval-ci, TP-H3 all-PASS): corpus-as-CI-gate live
(.github/workflows/judgment-corpus.yml + ci-roster-index.json), Layer-2 +
trajectory-sampling cadence recorded in deploy-v2/RUNBOOK-V2-EVAL-CADENCE.md,
sampler at deploy-v2/h3/trajectory-sample.sh. Remaining: H1b rootless
Podman (executor image rebuild carries the H2 wrapper live at the same time).

Twenty-second update (2026-11-25): ADR-0003 H1b executed — rootless Podman
live on the execution VM (TP-H1b all-PASS, decision D-APlus-podman):
full M2 V-suite V1–V5 PASS incl. V4 crash recovery + zero survivors, live
e2e af4489c5 exit-0. Along the way: root-caused the exit-254 spawn failures
(podman-compat maps docker CLI's PidsLimit:0 to pids.max=1; fix = explicit
--pids-limit 512, commit 374b821); rebuilt/reloaded the executor image so
the H2 untrusted-memory wrapper is live (grep-verified in the image); the
TP-H1b V-suite re-run of m2/install.sh reset the M3/M5-era executor config
to the stub (ledger 24) — re-staged M3-era compose + M5 §2' default,
verified hermes hydration live (boot_ms=1004). Ledger 23 records the
podman egress follow-up (pasta traffic bypasses DOCKER-USER; enforcement
verified empirically, formal podman-path ruleset = open follow-up).
ADR-0003 hardening complete: H1a + H1b + H2 + H3 all signed.

Twenty-third update (2026-11-25): ADR-0004 ACCEPTED (owner) — simplified
deployment as a shared spine + paired project VMs. Three shared services
on their own VMs (Infisical, hindsight, registry) serve all (VM-Xa, VM-Xb)
pairs (D6-REV: one project = one pair); Infisical lands as D-secret-vault
(self-hosted CE, LAN-only; .env files demote to derived artifacts of vault
reads); role-shaped idempotent install scripts + orchestrator replace the
m0–m5 chain as the deployment path (D-deploy-roles, kills the ledger-24
class); light per-component smoke suites with a mandatory end-to-end on
every pair deployment (D-deploy-gates); podman carried via xb_engine().
NOT a version tree — ADR-0003 pattern inside the standalone repo; book
revisions ch14/ch16 at P6. Build plan P0–P6 with TP gates in the ADR
(P5 = first clean 5-VM deployment, target < 1 h). ADR + decision-log +
this update written but UNCOMMITTED pending the owner's commit call
(ADR-0003 also still untracked).
