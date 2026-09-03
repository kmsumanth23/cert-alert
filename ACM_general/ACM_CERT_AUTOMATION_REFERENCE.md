# ACM Certificate Renewal & Reporting — Technical Reference

Complete documentation for the consolidated AWS ACM certificate automation.
This file replaces the inline comments in the playbooks: everything the code
comments explained is captured here, so the source can stay lean.

**Audience:** whoever maintains, debugs or extends the cert automation.

---

## 1. What this system does

Replaces ~25 independent per-client daily AWX workflows (each emailing per
certificate) with **one generated workflow per client group** that renews
certificates and sends **one consolidated HTML digest**.

| | Before | After |
|---|---|---|
| Workflows | ~25, one per client | 1 main + 1 per client group |
| Emails | 1 per cert per env per day | 1 digest/day + immediate alerts for hard failures only |
| Client onboarding | Edit AWX objects | Flip `refresh_certs: true` in the config repo |
| Untracked-cert visibility | none | `acm_fetch_all_certs` audit mode |

Requirements it satisfies: consolidated ≤15-day expiry reporting (R1),
delegation via one shared JT + RBAC (R3), bulk/scoped execution (R2 groundwork).

---

## 2. Components

| File | Deployed path | Role |
|---|---|---|
| `awxworkflowcertsawsfinal.yml` | `gcp-shared-services/playbooks/awx/kube/awx-workflow-certs-aws.yml` | **Generator** — discovers client-envs, builds the workflow + schedule |
| `awsacmcertimportauto.yml` | `aws-v9-automation/awsacmcertimportauto.yml` | **Parent playbook** of the cert JT — bastion prep, cert-source selection, empty-report fallback |
| `acmcertimport.yml` | `aws-v9-automation/tasks/aws-cert-import.yml` | **Cert engine** — ACM lookup, renewal, artifact publish |
| `acmcertdigest.yml` | `aws-v9-automation/acm-cert-digest.yml` | **Digest** — convergence node; merges artifacts, sends the email |
| `acm_digest_mail.html.j2` | `aws-v9-automation/templates/acm_digest_mail.html.j2` | Digest email template |

**Data sources**

| File | Repo | Provides |
|---|---|---|
| `clients/<c>/<env>/aws-client-vars-env.yml` | hclnow-config-client-environments | `refresh_certs` (opt-in), `client_infra_type` |
| `clients/<c>/<env>/tools.yml` | hclnow-config-client-environments | `certificates:` list (ungrouped clients) |
| `playbooks/awx/kube/common/vars/client-mapping.yml` | gcp-shared-services | **Per-group config**: `clients`, `git_project`, optional `cert_vars_file`, `dns_file`, `job_template`, `workflow_name`, `schedule` |
| `<group>.cert_vars_file` | gcp-shared-services | **Group conventions**: `namespace`, `valid_suffix`, `secret_name_suffix`, `cert_record_types`, `records_key`, `record_name_field`, `record_env_field` |
| `<group>.dns_file` | `<group>.git_project` (submodule) | Record list + `cert_report_subscriber` |
| `aws-client-vars-base.yml` | aws-v9-automation | SMTP settings, `smtp_emails_certs` |

**AWX objects**

| Object | Notes |
|---|---|
| JT `Import Certificate to ACM in AWS` | One general template serves every group; group behaviour comes from node `extra_data`, not from a different playbook. Override per group via `job_template` in the mapping |
| JT `ACM Certs Digest Report` | Playbook `acm-cert-digest.yml`; inventory `Localhost-HCLNOW-DevOps` |
| Workflow `renew certs - aws - all clients` | Generated, scheduled daily |
| Workflow `renew certs - aws - <group> clients` | Generated with `acm_cert_group: <group>`; name overridable per group |

---

## 3. Architecture

```
START ──► [hxsa-awsdevv9] ─► [hxsa-awsv9m] ──┐
START ──► [hxtx-np] ─────────────────────────┼──► [ACM Certs Digest Report]
START ──► [mosa-np] ─► [mosa-prd] ───────────┘        (converge: ALL)
                                                            │
                                                     ONE HTML email
```

- **One branch per client**; a client's envs run **sequentially** (chained via
  `always_nodes`) so two envs never hit the same client bastion concurrently.
- **Clients run in parallel** as independent branches.
- Links are **always** type: a failed env does not stop its client's chain and
  never blocks the digest.
- Digest node sets `all_parents_must_converge: true` — it waits for every
  branch, including failed ones.
- Node identifiers: `renew-cert-<client>-<env>`, plus `node-digest-report`.

### Artifact flow

Each cert job publishes a sanitized report via `set_stats`:

```
acm_report_<client>_<env> = {client, env, collected_at,
                             renewal_threshold_days, certs: [...]}
```

AWX propagates artifacts down the DAG into the digest job's extra_vars. The
digest compares them against `expected_reports` (injected by the generator) to
detect branches that died before reporting.

**Security:** report rows are an explicit whitelist (`domain`, `cert_arn`,
`not_after`, `days_left`, `status`, `error`, `in_use`). `certs_info` still
holds decoded PEM material for renewal-due certs — never publish it wholesale,
because artifacts land in the AWX database and job detail UI.

### The artifact-key contract

Publisher and consumer must agree on the key. Inventory vars
(`client_code`/`client_env`) can diverge from config-repo directory names, so
the generator — the single discovery authority — stamps `acm_report_client` /
`acm_report_env` into each node's `extra_data`, and the publish task prefers
them, falling back to inventory vars for ad-hoc runs. Without this, a healthy
env can appear as a false "NO report".

---

## 4. Execution flow

1. **Scheduled workflow fires.** Each branch launches the cert JT against
   `inventory_client_<client>_<env>`.
2. **Parent playbook** (`awsacmcertimportauto.yml`) prepares the bastion, then
   decides where `certificates` comes from:
   - default → `tools.yml` for that client-env
   - `acm_cert_group: <group>` → derived from that group's data file (§6)
3. **Cert engine** (`aws-cert-import.yml`): STS assume-role → ACM lookup →
   expiry math → for renewal-due certs: pull k8s TLS secret, split chain,
   `openssl verify`, import to the existing ACM ARN → per-cert status.
4. **Artifact published** — always, even when nothing is due. An env with no
   certificates publishes an **empty** report (parent-playbook fallback), which
   is how "nothing configured here" is distinguished from "job died".
5. **Digest node** runs last: buckets into failed / expiring / renewed /
   no-report and sends one email, or nothing if all buckets are empty.

### Certificate statuses

| Status | Meaning |
|---|---|
| `renewed` | Successfully re-imported to ACM this run |
| `import_failed` | `aws acm import-certificate` returned non-zero |
| `chain_verify_failed` | Local `openssl verify` failed; import deliberately skipped |
| `not_found_in_acm` | Domain configured but no matching ACM certificate |
| `expired` | Past expiry |
| `expiring_soon` | Within `client_cert_expiry_notice` days |
| `ok` (default) | Outside the notice window |

**Status is only stamped when an import actually ran.** `Set certificate
status` requires `item.rc is defined`; without that guard, skipped items fell
through to `default('expiring_soon')` and labelled every certificate
`expiring_soon` regardless of expiry — visible in fetch-all runs where every
import is skipped by design.

### Email behaviour

| Event | Email |
|---|---|
| Chain-verify failure | **Immediate**, per cert (+ digest row) |
| Import failure | **Immediate**, per cert (+ digest row) |
| Expiring / renewed / no-report | Digest only |
| All buckets empty | **No email** |

Digest sections: ❌ Failures · ⏰ Expiring ≤ N days · ✅ Renewed · ❓ NO report.
The **In use** column comes from ACM's `in_use_by` — empty list means the cert
is attached to nothing, so an expiry there is usually harmless.

---

## 5. The two expiry thresholds

They are independent and do **not** need to match:

- **`client_cert_expiry_notice`** (per client) — when renewal is attempted.
- **`digest_expiry_days`** (digest play var, default **15**) — how early a
  *non-renewed* cert appears in the amber bucket.

Renewed certs show green and failures show red **regardless** of
`digest_expiry_days`; the amber bucket is only a safety net for certs the
automation is not renewing. Guideline: keep `digest_expiry_days` ≥ the largest
client's `client_cert_expiry_notice`.

---

## 6. Client groups — certificates from an external repo

A **group** is a set of clients whose certificate data lives in their own repo
instead of `tools.yml`. Activated by `acm_cert_group: <group>` at generation
time; every difference travels in node `extra_data`, so the **cert engine is
unchanged**.

**No group name appears in code.** The value of `acm_cert_group` is the key in
`client_parent`, and everything else — client list, data repo, conventions,
job template, workflow and schedule names — is read from that mapping entry.
Onboarding a new group is a data change only.

### Why extra fields were needed

`tools.yml` gets client + env **for free from its path**
(`clients/<client>/<env>/tools.yml`). A group's data file is one combined file
across clients and envs, so that information had to be re-encoded:

- **client** → derived from the domain (label before `valid_suffix`)
- **env** → explicit `env:` field per record

### Record → certificate derivation

Every rule below is a **default** and can be overridden per group in that
group's `cert_vars_file` (see `cloud-cert-vars-EXAMPLE.yml`).

```
dns_full_name: "cname-autotest.hxsa.now.hclsoftware.cloud."
  strip trailing dot        → cname-autotest.hxsa.now.hclsoftware.cloud
  strip .valid_suffix       → cname-autotest.hxsa
  dots → hyphens (keep ALL labels, client code included)
                            → cname-autotest-hxsa
  append -tls-cert, lower   → cname-autotest-hxsa-tls-cert
```

| Field | Source |
|---|---|
| `domain_name` | FQDN minus trailing dot |
| `secret_name` | derived above, suffix from `secret_name_suffix` (RFC-1123 lowercase) |
| `secret_namespace` | group conventions → `namespace` |

More: `qa-custa.hxtx…` → `qa-custa-hxtx-tls-cert` · `dev.swep.mosa…` →
`dev-swep-mosa-tls-cert`

### Record filters

A record is used only if **all** hold:

1. `dns_full_name` is defined
2. `dns_type` ∈ `cert_record_types` (default {A, CNAME}) — TXT are ACME challenge tokens, never certs
3. not `_acme-challenge.*`
4. not a wildcard (`*` is invalid in a k8s secret name)
5. host has ≥2 labels
6. client label == this job's client
7. **`env` == this job's env** — a record with no `env` is **skipped, never
   guessed**; a wrong guess would pull a secret from the wrong cluster

### Client selection

From `client-mapping.yml` (see `client-mapping-EXAMPLE.yml`):

```yaml
client_parent:
  tx:
    git_project: hclnow-client-hxtx
    clients: [hxtx, mosa, txle, hxsa]   # hxsa = testing only
```

The generator uses `client_parent[acm_cert_group].clients` as the client list
and `.git_project` as the repo directory. It asserts the group resolved, so a
renamed key fails loudly instead of building an empty workflow. If the init
playbook already loaded the file, the in-scope `client_parent` is used.

A group's clients have **no `refresh_certs`** in the config repo — group
membership *is* their opt-in, and the flag is bypassed for them only.

### Recipients

Resolved in priority order, for both immediate alerts and the digest:

1. `cert_report_subscriber` from the group's data file (YAML list or comma string)
2. `smtp_emails_certs_by_group[acm_cert_group]` in `aws-client-vars-base.yml`
3. `smtp_emails_certs` (default — every ungrouped client hits this, unchanged)

The digest needs `acm_group_dns_file` in its node `extra_data` to read (1).

---

## 7. Fetch-all audit mode

`acm_fetch_all_certs: true` scans the **whole account/region** and reports every
**IMPORTED** certificate, instead of only this client's configured domains.
Catches certs nobody tracks — the class that silently expires.

- **Read-only by construction**: `renewal_due` is the literal `false`, so every
  write step skips regardless of `acm_report_only`.
- **AMAZON_ISSUED excluded** — they auto-renew; listing them is noise.
- Keyed by domain, sorted by expiry descending, so if two imported certs share
  a domain the **soonest-expiring** one wins.
- The parent playbook's include runs even when a client has **no** certificates
  configured — otherwise the audit would skip exactly the envs most likely to
  hold untracked certs — and the empty-report fallback is suppressed.
- `not_found_in_acm` comparison is suppressed: a configured domain whose cert is
  AMAZON_ISSUED *is* in ACM, just not imported.

Pair with `digest_expiry_days: 3650` to list everything rather than only
expiring certs. **Region-scoped** to each client-env's `aws_region`.

---

## 8. Variable reference

All are optional; unset means stock behaviour.

### Generator (pass at generator-JT launch)

| Var | Effect |
|---|---|
| `acm_cert_group: <group>` | Build that group's workflow. The value **is** the key in `client_parent`; no group name exists in code |
| `acm_cert_env_whitelist` | Limit scope. Entries are a whole client (`"hxtx"`) or one env (`"hxtx/np"`) |
| `acm_workflow_name_override` | Build a distinct workflow object |
| `acm_schedule_override` | Schedule name (default `Runs daily certs`) |
| `acm_test_ignore_refresh_flag` | Whitelisted envs bypass `refresh_certs` (needs a whitelist) |
| `acm_test_workflow_delete` | Delete the target workflow before rebuilding (clean rebuild) |
| `acm_scm_branch_override` | Pin nodes to an SCM branch (testing) |
| `acm_group_dns_env` | Value substituted into the group's `dns_file` pattern (default `prod`) |
| `acm_group_dns_file` | Full data-file path override |
| `acm_group_vars_file` | Full conventions-file path override |
| `acm_job_template_override` | Job template name override |
| `acm_digest_inventory` | Digest node inventory |
| `acm_group_exclude_from_main` | Subtract every grouped client from the main workflow |
| `acm_client_mapping_file` | Client-mapping path override |

> ⚠️ A whitelist **without** `acm_workflow_name_override` rebuilds the **main**
> workflow with only those clients — dropping everyone else.

### Runtime (pass at workflow launch; needs `ask_variables_on_launch`)

| Var | Effect |
|---|---|
| `acm_report_only: true` | Force `renewal_due` false → read-only run (report + digest, no writes) |
| `acm_fetch_all_certs: true` | Account-wide imported-cert audit (§7) |
| `digest_expiry_days: <n>` | Override the 15-day amber window |

### Where variables can be set

| Layer | Scope | Precedence |
|---|---|---|
| JT → Extra Variables (`base_src`) | every launch of that JT | low |
| **Workflow node `extra_data`** (written by the generator) | that node | **highest** |
| Workflow launch prompt | one run | high |

`acm_report_only` is deliberately **not** pinned into node `extra_data` unless
passed at generation time — node data outranks the launch prompt, so pinning it
would make read-only (or renewing) runs impossible without regenerating.

---

## 9. Operations

### Build / rebuild a workflow

```yaml
# main workflow
awx_type: <value>

# a group's workflow
awx_type: <value>
acm_cert_group: <group>
```

Adding a client to the config repo → next generator run **updates** the existing
workflow (same name = idempotent update). Additions are proven; if a client is
ever *removed*, check the visualizer for orphaned nodes and use
`acm_test_workflow_delete: true` for a clean rebuild.

### Job template requirements

**Cert JT** — `ask_extra_vars: true` (mandatory: without it AWX drops node
`extra_data` and group sourcing silently never happens), `ask_inventory: true`
(nodes set their own inventory), `ask_scm_branch_on_launch: true` (branch
override), plus the bastion `limit` pattern.

**Digest JT** — `ask_extra_vars: true` (mandatory for `expected_reports`) and
**`limit` MUST be empty** (the play targets implicit localhost; a `bastion*`
limit fails with "does not match any hosts"). Inventory contents are irrelevant.

### Delegation (R3)

Grant a team **Execute** on the cert JT and **Use** on only their
`inventory_client_<client>_<env>` inventories. A dedicated JT copy is only
required when the team needs its **own AWS credentials** (credentials attach at
JT level) or when Execute on the shared JT is too wide a surface.

### Submodule handling (grouped clients)

A group's `git_project` must be a submodule of `aws-v9-automation`. This tower has
**track submodules enabled**, so the `branch` field in `.gitmodules` decides
what gets checked out — not the pinned commit. To test a branch, set
`.gitmodules` → `branch = <branch>`, commit **and push**, then sync the AWX
project. Revert to `master` before production.

---

## 10. Onboarding a new client group

No group name exists anywhere in the code. Onboarding is a **data** change in
two files plus AWX object setup. This section is the checklist, and the reasons
behind each item — most were learned the hard way during the TX rollout.

### 10.1 Prerequisites

| # | Requirement | Why |
|---|---|---|
| 1 | A git repo holding the group's DNS/cert records, added as a **submodule of `aws-v9-automation`** | The cert job reads it from its own checkout. A submodule of `gcp-shared-services` is not visible to the cert job |
| 2 | Every record carries the **client label inside the FQDN** and an **explicit env field** | Client is derived from the domain; env is **not derivable**. A record with no env field is skipped, never guessed — a wrong guess pulls a TLS secret from the wrong cluster |
| 3 | A cert-vars file in `gcp-shared-services` declaring at minimum `namespace` and `valid_suffix` | The built-in fallbacks are HCL-specific (§10.4) |
| 4 | That cert-vars file must reference **no variables outside the cert job's scope**, or a shim must be added | `include_vars` renders the whole file (§10.4) |
| 5 | AWX inventory `inventory_client_<client>_<env>` exists for every client-env in the group | A missing inventory aborts the build **before** the link pass, leaving a linkless workflow |
| 6 | The Import JT carries a **Vault credential** if the cert-vars file has vaulted values | `include_vars` decrypts; `lookup('file') \| from_yaml` cannot |
| 7 | One client-env = one Kubernetes cluster | Secrets are pulled per client-env; two clusters behind one env would silently pull from whichever the bastion reaches |
| 8 | `cert_report_subscriber` in the group's data file (optional) | Group-scoped digest recipients; without it the group falls back to `smtp_emails_certs` and mails the shared list |

### 10.2 Steps

1. **Add the submodule** to `aws-v9-automation`, then confirm the AWX project
   syncs submodules. Track-submodules is enabled on this tower, so `.gitmodules`
   → `branch` decides the checkout, not the pinned commit.
2. **Add the group** to `playbooks/awx/kube/common/vars/client-mapping.yml`:

   ```yaml
   client_parent:
     <group>:
       git_project: <submodule dir>
       clients: [<client>, ...]
       # optional overrides:
       # cert_vars_file / dns_file / job_template / workflow_name / schedule
   ```
3. **Point `cert_vars_file`** at the group's conventions file, or leave it
   unset to use `playbooks/dns/cloud-cert-vars.yml`.
4. **Verify the cert-vars file is safe to include** — this is the step that
   bit us:

   ```bash
   grep -o '{{[^}]*}}' <cert_vars_file> | sort -u
   ```

   Only `base_src`, `client_code`, `client_env` are in the cert job's scope.
   Anything else needs a shim in *Snapshot cert-group conventions* (§10.4).
5. **Generate scoped to one env first**, against a test branch:

   ```yaml
   acm_cert_group: "<group>"
   acm_cert_env_whitelist: ["<client>/<env>"]
   acm_test_ignore_refresh_flag: true
   acm_group_dns_env: "test"
   acm_workflow_name_override: "TEST renew certs - aws - <group>"
   acm_test_workflow_delete: true
   acm_scm_branch_override: "<test branch>"
   ```
6. **Run read-only** (`acm_report_only: true` at launch) and read the
   **"Show certificates derived for this client env"** line: source file,
   records read, certificates derived, records skipped for a missing env,
   resolved recipients.
7. **Verify the derived secret names** against
   `kubectl get secrets -n <namespace>`. This is the gate for enabling
   renewal — a wrong `valid_suffix` or `secret_name_suffix` shows up here.
8. **Widen** to the whole group, drop `acm_report_only`, then regenerate
   without the test overrides so the schedule attaches.

### 10.3 Is there any hardcoding left?

No group, client, or repo name appears in any of the four code files — the only
occurrences are in comments and worked examples. What *is* baked in:

| Baked in | Overridable? | Note |
|---|---|---|
| `now.hclsoftware.cloud`, `hclnow-certs` | Yes — `valid_suffix`, `namespace` | **Fails silently**, see §10.4 |
| `dns`, `dns_full_name`, `env`, `[A, CNAME]`, `-tls-cert` | Yes — `records_key`, `record_name_field`, `record_env_field`, `cert_record_types`, `secret_name_suffix` | Same silent-failure class |
| `Import Certificate to ACM in AWS`, `Runs daily certs`, `renew certs - aws - <group> clients` | Yes — `job_template`, `schedule`, `workflow_name` in the mapping | Fails loudly |
| Record shape: a top-level key holding a **flat list** of records | **No** | A nested or per-client-keyed file needs a code change |
| `inventory_client_<client>_<env>` naming | **No** | Pre-existing platform convention |
| `hclnow-config-client-environments` layout, `srvc<env><client>` profiles | **No** | Pre-existing, outside this work |

So: a new external client following the same structure is a data change. A
client with a *differently shaped* records file needs the derivation extended.

### 10.4 Discovered gotchas

**`include_vars` renders the whole file.** It templates values lazily, so
reading one key forces every value in the file to render. `cloud-cert-vars.yml`
belongs to the DNS job and contains
`dns_file_path: "{{ base_src }}/{{ dns_config_project }}/...{{ awx_env }}.yml"`,
which killed the cert job with `dns_config_project is undefined`. Fixed by
rendering the dict **once**, in *Snapshot cert-group conventions*, with
`dns_config_project` and `awx_env` supplied as task-level `vars:` from
`acm_group_git_project` / `acm_group_dns_env` (both injected as node
`extra_data`). A third foreign reference fails in that same task — add the shim
there, not on the consumers.

**Vault forced that switch.** The task originally used
`lookup('file') | from_yaml`, which left those strings inert and never hit the
problem — but it also cannot decrypt inline `!vault` values, which is why the
file had to move to `include_vars` in the first place. Do not "fix" the render
error by reverting.

**The convention defaults fail silently, not loudly.** An external client whose
domain is not `now.hclsoftware.cloud` and who does not set `valid_suffix` gets
*derived certificates with wrong secret names* — a green run, an empty or wrong
report, no error. Treat step 7 above as mandatory, not optional.

**Two-pass workflow build.** Nodes are created first, links second. Any failure
during node creation (almost always a missing inventory) aborts the play and
leaves every node dangling off START. Rebuild with `acm_test_workflow_delete:
true` after fixing; do not patch a half-built workflow.

**A whitelist without `acm_workflow_name_override` rebuilds the MAIN workflow**
with only the whitelisted clients, dropping everyone else. Always pair them.

**Node `extra_data` outranks the launch prompt.** Anything pinned into a node
at generation time cannot be overridden at run time. This is why
`acm_report_only` is only baked in when explicitly passed to the generator.

**Wildcard and `_acme-challenge` records are skipped** — `*.client…` would
derive an invalid RFC-1123 secret name.

---

## 11. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Nodes all hang off START, no links | Node creation failed mid-build (usually a missing inventory) → play aborted before the relation pass | Fix/whitelist inventories, rebuild with `acm_test_workflow_delete: true` |
| Digest: "does not match any hosts" | `limit` set on the digest JT | Clear it |
| Digest asserts `expected_reports` missing | Digest JT has prompt-on-launch Variables disabled | Enable it |
| Green client job but NO-report row | Job never reached the publish task | Check that env's job log |
| False NO-report despite artifact published | Artifact key mismatch | Verify the node carries `acm_report_client`/`acm_report_env` |
| Group run derives 0 certificates | Wrong data file (`acm_group_dns_env`), SCM branch without the group code, or client/env filter mismatch | Read the "Show certificates derived for this client env" debug line — it prints the file path, record count, derived secret names, skipped records and recipients |
| Every cert reads `expiring_soon` | `Set certificate status` missing the `item.rc is defined` guard | Add it |
| "recursive loop detected in template string" | A task-level `vars:` entry defaulting **from a var of the same name** is self-referential | Inline the default inside the expression, or use a distinct name |
| `digest_node is undefined` | Two `vars:` keys on the same task — YAML keeps only the last | Merge into one `vars:` block |
| Generator can't resolve the group | `client-mapping.yml` moved or keys renamed | The assert names the expected path/keys |
| SMTP "Username and Password sent without encryption" | Relay session without STARTTLS (pre-existing) | Accepted for the internal relay; add `secure: starttls` if supported |
| No email on a day you expected one | All four buckets empty | By design — check the "Digest summary" debug line |

---

## 12. Known gaps / future work

1. **Group data-file shape** — the derivation understands one record shape
   (a list of records, each with an FQDN field and an env field). The field
   names, record types, suffix and namespace are configurable, but a group
   whose file is structured differently would need a new derivation path.
2. **Region scope** — one region per client-env. CloudFront certs in
   `us-east-1` won't appear for clients running elsewhere.
3. **Digest is a single point of failure** — if the digest job itself dies, the
   day's reporting vanishes silently. Attach an AWX on-failure notification
   template to the digest JT and the workflow.
4. **Two-pass workflow creation** — the shared `awx-create-workflow.yml`
   pipeline creates nodes then relations; a failure between them leaves a
   linkless workflow. A single `awx.awx.workflow_job_template` call with the
   full `workflow_nodes` list would be atomic, at the cost of diverging from how
   every other workflow on the tower is built.
5. **`+11h` expiry adjustment** — a hardcoded `39600` magic number in the expiry
   math with no explanation of origin.
6. **`CertificateArn` in the group data records** is currently unused; the engine
   still resolves ACM by domain. Using it would help where the ACM cert's domain
   differs from the DNS name (e.g. a wildcard covering the host).
