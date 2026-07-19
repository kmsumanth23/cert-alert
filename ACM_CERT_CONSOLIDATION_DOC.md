# AWS ACM Certificate Renewal — Consolidated Reporting & Delegation

**Status:** PoC complete and verified on the AWX test tower · production cutover prepared
**Scope:** all AWS client clusters managed via the `hclnow-config-client-environments` config repo
**Repos touched:** `aws-v9-automation` (cert playbooks), `gcp-shared-services` (AWX generator), config repo (per-env `refresh_certs` flag)

---

## 1. Problem

The certificate renewal automation ran as **~25 independent per-client daily workflows**, each executing the same `Import Certificate to ACM in AWS` job template against one client inventory. Every job emailed the group individually — one mail per certificate per environment per client per day for expiring/renewed certs — spamming the mailbox and making fleet-wide tracking impossible. All 25+ workflows were owned and babysat by the DevOps team.

### Requirements (Praveen)

| # | Requirement | Priority | Outcome |
|---|---|---|---|
| 1 | **Consolidated Reporting** — one mechanism identifying all certs expiring within 15 days across all client clusters; reduce email volume | Primary | ✅ **Delivered & PoC-verified** |
| 3 | **Delegation & Shared Access** — shared jobs/templates so other teams execute renewals against their own inventories; stop DevOps managing 25+ daily jobs | Primary | ✅ **Delivered** (architecture + runbook; RBAC grants are a cutover admin step — §7) |
| 2 | **Bulk Execution** — trigger renewals for multiple clusters (e.g. all non-prod) in one operation | Lower | ⏸ Deferred; ~80% of machinery already exists (§8) |

---

## 2. Architecture

**One generated workflow replaces ~25.** Clients are parallel branches; a client's environments run sequentially in a chain; every branch converges on a single digest node that sends one HTML email.

```
                 "renew certs - aws - all clients"  (daily schedule)

START ──► [hxsa-awsdevv9] ─► [hxsa-awsfred] ─► [hxsa-awsv9m] ──┐
START ──► [hxtx-np] ───────────────────────────────────────────┼──► [ACM Certs Digest Report]
START ──► [mosa-np] ─► [mosa-prd] ─────────────────────────────┘        (converge: ALL)
            ... every client with refresh_certs: true ...                     │
                                                                              ▼
                                                                     ONE HTML email
```

Key properties (all verified in the PoC):

- **Always-links** between nodes: a failed env does not stop its client's chain, and never blocks the digest.
- **Convergence = ALL** (`all_parents_must_converge: true`): the digest waits for every branch, including failed ones.
- **Artifacts** (`set_stats`): each cert job publishes a sanitized report; AWX propagates artifacts down the DAG into the digest job's extra_vars.
- **Manifest** (`expected_reports`): the generator injects the full discovered client-env list into the digest node, so a branch that dies silently is *detected*, never assumed healthy.
- **Single source of truth:** the same discovery loop builds the workflow nodes AND the manifest AND the artifact-key identities — they cannot drift apart.

### Components

| File (this repo) | Deploys to | Role |
|---|---|---|
| `awxworkflowcertsaws.yml` | `gcp-shared-services/playbooks/awx/kube/awx-workflow-certs-aws.yml` | **Generator** — discovers AWS client-envs, builds the one consolidated workflow + schedule. Include ONCE per generator run (outside the per-client loop) |
| `aws-acm-cert-import-auto.yml` | `aws-v9-automation/awsacmcertimportauto.yml` | **Parent playbook** of the cert JT — bastion prep, then includes cert-import; publishes an EMPTY report when no certificates are configured |
| `awscertimport.yml` | `aws-v9-automation/tasks/aws-cert-import.yml` | **Cert engine** — STS assume-role, ACM expiry check, k8s secret pull, chain verify, ACM import, per-cert status, artifact publish |
| `acmcertdigest.yml` | `aws-v9-automation/acm-cert-digest.yml` | **Digest playbook** — the convergence node; merges all artifacts against the manifest, buckets, sends one email |
| `acm_digest_mail.html.j2` | `aws-v9-automation/templates/acm_digest_mail.html.j2` | HTML email template |
| `SETUP.md` | — | Original setup instructions from the PoC |

### AWX objects

| Object | Notes |
|---|---|
| JT `Import Certificate to ACM in AWS` | Existing; unchanged shape. Prompts for inventory (that's how one JT serves all clients) |
| JT `ACM Certs Digest Report` | New. Playbook `acm-cert-digest.yml`; inventory **`Localhost-HCLNOW-DevOps`** (contents never used — play targets implicit localhost on the EE); **Limit field EMPTY** (a `bastion*` limit filters out localhost → "does not match any hosts"); **Prompt on launch → Variables: ENABLED** (required for `expected_reports` extra_data) |
| Workflow `renew certs - aws - all clients` | Generated; scheduled daily via `Runs daily certs` |

---

## 3. How a daily run works

1. **Scheduled workflow fires.** Each client branch launches the cert JT against `inventory_client_<client>_<env>`.
2. **Per env:** STS assume-role into the client account → `acm_certificate_info` per configured domain → expiry math against `client_cert_expiry_notice` → if renewal due: pull TLS secret from k8s, split chain, `openssl verify`, import to ACM under the existing ARN → per-cert status.
3. **Artifact publish** (always, even when nothing is due): key `acm_report_<client>_<env>`, payload = whitelisted fields only (`domain`, `cert_arn`, `not_after`, `days_left`, `status`, `error`). Never PEM material — artifacts land in the AWX DB and job UI. Envs with **no certificates configured** publish an **empty** report (parent playbook fallback).
4. **Digest node** runs last: for each manifest entry, look up the artifact → bucket into *failed* / *expiring ≤ 15d* / *renewed* / *no-report* → send one email, or nothing if every bucket is empty.

### Artifact-key contract

The key must be agreed between the job (publisher) and the digest (consumer). The generator — the single discovery authority — stamps `acm_report_client` / `acm_report_env` (config-repo directory names) into each node's `extra_data`; the publish task prefers those and falls back to inventory vars (`client_code`/`client_env`) for ad-hoc runs. This prevents false "NO report" alarms when inventory vars diverge from directory names (they do in this estate — e.g. `awsuni/group_vars/hxsa_np`).

### Email semantics

| Section | Rows | Meaning |
|---|---|---|
| ❌ Failures (red) | `import_failed`, `chain_verify_failed`, `not_found_in_acm` | Manual action; the first two ALSO fire immediate individual emails |
| ⏰ Expiring ≤ 15d (amber, sorted by days left) | seen, in window, not yet renewed | ≤ 7 days styled red/bold |
| ✅ Renewed (green) | successfully re-imported this run | — |
| ❓ NO report (red) | branch died before publishing | "certs NOT checked — treat as unknown"; **empty reports do NOT land here** |
| *(no email at all)* | all four buckets empty | All-healthy days are silent by design |

**Email volume:** from up to (certs × envs × clients) per day → **one digest per day** + rare immediate alerts for the two kept hard-failure types.

---

## 4. Requirement 1 — Consolidated Reporting: HOW IT IS FULFILLED

- One scheduled workflow covers **all** AWS client clusters; the digest's amber section is exactly "all certificates expiring within the next 15 days across all client clusters" (`digest_expiry_days: 15`, overridable per-run via extra var for testing).
- Per-cert `renewed`/`expiring_soon` emails **removed** from the cert job; only `import_failed` and `chain_verify_failed` still alert immediately (deliberate — they need human hands).
- Tracking is simplified structurally: the manifest guarantees every expected env appears in the accounting (reported / empty / NO-report) — silence is impossible.

**PoC evidence:** multi-branch run (hxtx + hxsa chain) with two genuinely-failed envs → one email containing the expiring hxtx cert, the empty hxsa/awsv9m env correctly absent from buckets, and the two dead envs as NO-report rows. Convergence ALL and always-links verified in the AWX visualizer.

## 5. Requirement 3 — Delegation & Shared Access: HOW IT IS FULFILLED

Two halves:

**(a) "Stop DevOps managing 25+ individual daily jobs"** — solved by the consolidation itself: 25 scheduled workflows collapse into 1, owned by DevOps, driven entirely by the per-env `refresh_certs` flag in the config repo. Onboarding/offboarding a client = one flag change + generator run; no AWX surgery.

**(b) "Shared templates other teams execute against their inventories"** — the renewal logic already lives in ONE job template parameterized purely by inventory, so delegation is **pure AWX RBAC, zero new code**:

1. Create an AWX **Team** for the external group.
2. Grant the team **Execute** on the cert job template (or a dedicated copy — see §7).
3. Grant the team **Use** on *only their* `inventory_client_<client>_<env>` inventories. AWX then physically restricts the inventory picker to those.
4. Keep **Prompt on launch → Variables OFF** on the delegated JT so nobody injects extra vars.
5. Never grant teams the generator or the workflow — those stay DevOps-owned.

Known, accepted behavior: an ad-hoc delegated run happens outside the workflow, so it won't appear in that day's digest; hard failures still alert immediately, and the next daily digest reflects true ACM state.

---

## 6. Production cutover runbook

Files in this repo are already in production form (TEST name removed, schedule restored, test vars stripped to inert extra-var overrides, `digest_expiry_days: 15`, no pinned `scm_branch`).

1. **Merge to production branches:** `awscertimport.yml` + `aws-acm-cert-import-auto.yml` + template → `aws-v9-automation` main; `awxworkflowcertsaws.yml` → `gcp-shared-services` main. Sync both AWX projects.
2. **Config repo:** verify the production AWX projects track the branch where real clients have `refresh_certs: true`. This flag is the ONLY production scope switch.
3. **Prod digest JT:** confirm `ACM Certs Digest Report` exists with inventory `Localhost-HCLNOW-DevOps`, empty Limit, Prompt-on-launch Variables ON, playbook `acm-cert-digest.yml`.
4. **Inventory precheck (important):** every discovered client-env must have its `inventory_client_<client>_<env>` in prod AWX — a missing inventory aborts the two-pass build before link creation, leaving a linkless workflow. If unsure, do a first run with `acm_cert_env_whitelist` covering verified envs, then widen.
5. **First production generator run** with extra var `acm_test_workflow_delete: true` (one-time: removes the leftover PoC `TEST renew certs - aws - all clients` workflow). Subsequent runs: no extra vars at all.
6. **Verify in AWX:** node count = discovered envs + 1; per-client chains; all links Always; digest node convergence ALL; `expected_reports` in digest node extra_data; schedule `Runs daily certs` attached.
7. **Parallel-run window (2–3 days):** old per-client AWS workflows and the consolidated one coexist. Overlap is tolerable (a second run sees post-import expiry, so no double renewal) but failures alert twice — keep it short. Compare the digest against known cert state daily.
8. **Decommission:** disable/remove the old per-client AWS workflows (teach `awx-workflow-certs.yml` to skip `cloud_provider == 'aws'`, or set them `state: absent`). GCP path untouched.
9. **Recipients:** digest and immediate alerts use the same `smtp_emails_certs` / `cloudops_group_emails` from `aws-client-vars-base.yml` — remove any personal test addresses.

### Maintenance extra-vars reference (all inert unless passed)

| Var | Consumed by | Effect |
|---|---|---|
| `acm_cert_env_whitelist: ["client/env", ...]` | Generator | Restrict discovery to listed envs (test towers, emergency scope-cut, team-scoped builds) |
| `acm_test_ignore_refresh_flag: true` | Generator | Whitelisted envs bypass `refresh_certs` (only works WITH a whitelist) |
| `acm_test_workflow_delete: true` | Generator | Delete the leftover PoC TEST workflow before building |
| `acm_scm_branch_override: <branch>` | Generator | Pin cert-JT nodes to a test SCM branch (PoC used `sum-cert-alert-aws`) |
| `acm_digest_inventory: <name>` | Generator | Digest node inventory (default `Localhost-HCLNOW-DevOps`) |
| `digest_expiry_days: <n>` | Digest JT | Override the 15-day window for a run (PoC used 45 to force an email) |

### How to pass extra vars in the AWX tower

There are two places to put them, and the difference matters:

- **Launch prompt (preferred for all test/maintenance vars):** the generator's job template (`Initialize resources in awx`) already prompts for variables — it requires `awx_type` at launch — so paste the ACM vars into the **Extra Variables** box on the launch screen, alongside `awx_type`, in YAML. They apply to that one run only and leave nothing behind.
- **Job template → Extra Variables field:** persists for **every** launch, including scheduled ones, until someone removes it. Use only for a deliberate standing override (e.g. keeping `digest_expiry_days` widened during a bake-in period). A test var left here is the moral equivalent of the hardcoded `set_fact` block we stripped at cutover — just relocated. Always clean it up.

Why this works with zero code changes: **extra vars have the highest precedence in Ansible** — they beat the playbook's `vars:`, every `vars_files`, and all inventory/group vars. Every override in the table exists as a `default(...)`-guarded reference in the code, so an unset var means stock behavior.

**Example 1 — test-tower generator run** (exactly what the PoC did, previously hardcoded in the reset `set_fact`):
```yaml
awx_type: <your usual value>
acm_cert_env_whitelist:
  - "hxtx/np"
  - "hxsa/awsv9m"
  - "hxsa/awsdevv9"
  - "hxsa/awsfred"
acm_test_ignore_refresh_flag: true
acm_test_workflow_delete: true
acm_scm_branch_override: "sum-cert-alert-aws"
```

**Example 2 — first production generator run** (one-time cleanup of the PoC TEST workflow):
```yaml
awx_type: <your usual value>
acm_test_workflow_delete: true
```

**Example 3 — steady-state production run:** no ACM vars at all. Scope comes exclusively from `refresh_certs: true` in the config repo.

**Example 4 — emergency scope-cut** (a client's account is broken and its branch fails daily; rebuild the workflow without it until fixed):
```yaml
awx_type: <your usual value>
acm_cert_env_whitelist:
  - "hxtx/np"
  - "mosa/prd"
  # ...every env EXCEPT the broken one
```
The whitelist shrinks the digest manifest too, so the excluded env raises no NO-report noise while out. Re-run the generator **without** the whitelist to restore full scope — don't forget, or the fleet silently stays shrunk.

**Example 5 — digest window override** (digest JT → Extra Variables field, or via a temporary edit while testing):
```yaml
digest_expiry_days: 45
```

**Cert-import JT launched directly (ad-hoc / delegated runs):** the JT prompts for inventory; pass the established launch vars for this JT family — `infra_code`, `now_cloud`, and `prod_ticket` where applicable (the same three the workflow nodes inject). Do **not** pass `acm_report_client` / `acm_report_env` manually: they exist so the *generator* can pin artifact keys inside the workflow. An ad-hoc run correctly falls back to the inventory's `client_code`/`client_env`, and its artifact is consumed by nobody — harmless.

---

## 7. External Team Access — hxtx, mosa & txle

**Ask:** a separate job for a different team to check certificates on three specific client accounts (hxtx, mosa, txle), authenticating with **their own admin account** which has **restricted access**.

### Design: dedicated JT copy + RBAC + credential override

Do **not** reuse the DevOps cert JT directly — a dedicated copy isolates their credentials, their SCM pin, and their blast radius:

1. **Create JT `Import Certificate to ACM in AWS - ExtTeam`** — same project + playbook (`awsacmcertimportauto.yml`) as the DevOps JT. `ask_inventory_on_launch: true`, **Prompt-on-launch Variables OFF**, Limit empty.
2. **AWX Team + RBAC:** create Team `ext-cert-team`; grant it **Execute** on this JT and **Use** on only `inventory_client_hxtx_*`, `inventory_client_mosa_*`, `inventory_client_txle_*`. AWX's inventory prompt then shows them exactly those and nothing else. No access to the generator, the consolidated workflow, the digest JT, or any other inventory.
3. **Their own AWS admin account:** the playbook authenticates via `access_val` / `sec_val` / `arn_auto_admin_val` (STS assume-role). Inject the team's own values via a **custom AWX Credential Type** attached to their JT:
   - Inputs: access key, secret key (secret: true), role ARN.
   - Injector: extra_vars `access_val`, `sec_val`, `arn_auto_admin_val` — extra vars outrank the client vars files, so their credentials cleanly override the DevOps ones for their runs, and the secret is stored encrypted in AWX rather than in any vars file.
   - One credential per team (or per account if their role ARNs differ per client account — then three credentials and they pick the matching one at launch, or three thin JT copies each pinning inventory+credential).
4. **Restricted IAM needs (minimum for the role they assume):** `acm:DescribeCertificate`, `acm:ListCertificates`, `acm:ImportCertificate` (re-import to existing ARN), plus `sts:AssumeRole` on the target role from their user. If they are check-only (no renewals), drop `acm:ImportCertificate` — expiry checks still work and an attempted import fails cleanly with an immediate alert.
5. **Reporting interplay:** their ad-hoc runs are outside the consolidated workflow — no digest interference, artifacts published under fallback keys are harmless. hxtx/mosa/txle remain in the DevOps daily digest regardless (their `refresh_certs` flags are independent). Immediate failure emails from their runs go to the standard cert alert group; add the team's address to `smtp_emails_certs`-equivalent if they should receive them too.
6. **Optional later (Requirement 2 preview):** if the team wants their three clients in ONE click instead of per-inventory launches, run the generator with `acm_cert_env_whitelist` covering the hxtx/mosa/txle envs and a distinct workflow name + their recipients — a team-scoped mini consolidated workflow. Not needed for handover v1.

### Handover checklist

- [ ] Custom credential type created; team credential(s) entered by the team (DevOps never sees the secret)
- [ ] JT `Import Certificate to ACM in AWS - ExtTeam` created (prompts inventory, vars off, empty limit)
- [ ] Team `ext-cert-team` with Execute on the JT, Use on the 3 clients' inventories only
- [ ] Their IAM role validated: ACM describe/list (+import if renewals allowed) in each of the 3 accounts
- [ ] One supervised test launch per client with the team watching
- [ ] Short how-to for the team: Templates → launch → pick inventory → read job output; failures also email

---

## 8. Requirement 2 (deferred) — how to build it when prioritized

The generator's discovery + whitelist filter is already a bulk-selection mechanism. "Renew all non-prod in one operation" = a second generator invocation emitting an **unscheduled** workflow (e.g. `renew certs - aws - nonprod`) whose discovery filters env names by class (or a `env_class` var in the env files), same digest pattern, launched on demand. No new architecture required.

---

## 9. Troubleshooting quick reference

| Symptom | Cause | Fix |
|---|---|---|
| Workflow nodes all hang off START, no links | Node-creation failed mid-build (usually missing inventory) → play aborted before the relation pass | Fix/whitelist inventories; rebuild (delete workflow first so nodes report `changed` and relations re-create) |
| Digest job: "does not match any hosts" | Limit field set on the digest JT | Clear it — the play targets implicit localhost |
| Digest asserts `expected_reports` missing | Digest JT's Prompt-on-launch Variables disabled → AWX dropped node extra_data | Enable it |
| Green client job but NO-report row | Job never reached the publish task (e.g. `certificates` empty pre-fallback, or died mid-play) | Check that env's job log; empty-cert envs publish empty reports since the parent-playbook fallback |
| False NO-report despite artifact published | Artifact key mismatch (inventory vars ≠ directory names) | Fixed by design: generator passes `acm_report_client/env`; verify the node's extra_data carries them |
| "Username and Password sent without encryption" warning on mail | SMTP relay session without STARTTLS — same as the legacy per-cert mails | Accepted (internal relay); optionally add `secure: starttls` to mail tasks if the relay supports it |
| No email on a day you expected one | All four buckets empty — healthy day | By design; check the digest job's "Digest summary" debug line |

---

## 10. Design note — why the generator uses `set_fact` instead of calling `awx.awx.workflow_job_template` / `workflow_job_template_node` directly

The generator deliberately does **data assembly only** (build the list of node dicts via `set_fact` + Jinja) and then hands the finished structure to the repo's existing shared includes — `awx-create-workflow.yml`, `awx-create-workflow-nodes.yml`, `awx-create-schedule.yml` — which own the actual AWX module calls. Reasons:

1. **Contract compatibility.** The per-client generator (`awx-workflow-certs.yml`) feeds those same includes with the same node-dict schema. The consolidated generator plugs into identical machinery, so both paths create workflows the same way, with the same module versions, same controller auth, same relation handling. One creation pipeline, two data sources.
2. **Single change point for tower plumbing.** Controller hostname, credentials, module choice (`awx.awx.*` vs legacy `tower_*`), TLS options — all live once in the shared includes. If the generator called modules directly it would duplicate all of that (the PoC's Step 4b delete task is a small taste: it had to carry its own `controller_host/username/password`).
3. **Separation of concerns = testability.** The hard part of this design is the *data*: chains, identifiers, extra_data, the manifest. As pure `set_fact` output it's printable (`Print consolidated workflow nodes`) and verifiable before anything touches AWX — which is exactly how the PoC debugged topology issues.

### The optimization worth considering (later, not at cutover)

The existing shared pipeline creates workflows in **two passes** — nodes first, then relations — and the PoC exposed its weakness twice: a node-creation failure aborts the play **before** the relation pass (leaving a linkless workflow), and the relation pass may skip unchanged nodes on rebuilds. A single call to `awx.awx.workflow_job_template` with the full `workflow_nodes:` list (the schema our node dicts already follow, including `related.always_nodes` by identifier and `all_parents_must_converge`) replaces both passes atomically: the module diffs and applies nodes + links + convergence in one idempotent operation, eliminating the linkless-workflow failure mode and the rebuild-delete dance entirely.

Trade-offs to weigh before adopting it:
- It bypasses the shared includes, so the consolidated path would diverge from how every other workflow on the tower is created (point 1 above) — or you migrate the shared includes themselves, which touches the GCP path too.
- Per-node error behavior changes: today a missing inventory fails that one loop item and the log names it precisely; the single-call module fails the whole operation with one (less pinpointed) error.
- Requires a recent `awx.awx` collection in the EE and its schema quirks validated against your AWX version.

Verdict: correct future optimization, wrong cutover-day change. Ship production on the proven two-pass pipeline; revisit as a hygiene item once stable, ideally as a migration of the shared includes so *all* workflows benefit.
