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
| Workflows | ~25, one per client | 1 per group (`all clients`, `tx clients`) |
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
| `clients/<c>/<env>/tools.yml` | hclnow-config-client-environments | `certificates:` list (non-TX clients) |
| `playbooks/awx/kube/common/vars/client-mapping.yml` | gcp-shared-services | `client_parent.<group>.{clients, git_project}` |
| `playbooks/dns/cloud-cert-vars.yml` | gcp-shared-services | `namespace`, `valid_suffix` (TX) |
| `hclsoftware-cloud-dns-<env>.yml` | hclnow-client-hxtx (submodule) | `dns:` records + `cert_report_subscriber` (TX) |
| `aws-client-vars-base.yml` | aws-v9-automation | SMTP settings, `smtp_emails_certs` |

**AWX objects**

| Object | Notes |
|---|---|
| JT `Import Certificate to ACM in AWS` | One general template serves every group; TX behaviour comes from node `extra_data`, not from a different playbook |
| JT `ACM Certs Digest Report` | Playbook `acm-cert-digest.yml`; inventory `Localhost-HCLNOW-DevOps` |
| Workflow `renew certs - aws - all clients` | Generated, scheduled daily |
| Workflow `renew certs - aws - tx clients` | Generated with `acm_cert_source: tx` |

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
   - `acm_certs_source: tx` → derived from the TX DNS config (§6)
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

## 6. TX group — certificates from an external repo

TX clients (`hxtx`, `mosa`, `txle`) own their certificate data in
`hclnow-client-hxtx` instead of `tools.yml`. Activated by
`acm_cert_source: tx` at generation time; every difference travels in node
`extra_data`, so the **cert engine is unchanged**.

### Why extra fields were needed

`tools.yml` gets client + env **for free from its path**
(`clients/<client>/<env>/tools.yml`). The TX DNS file is one combined file
across clients and envs, so that information had to be re-encoded:

- **client** → derived from the domain (label before `valid_suffix`)
- **env** → explicit `env:` field per record

### Record → certificate derivation

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
| `secret_name` | derived above (RFC-1123 lowercase) |
| `secret_namespace` | `cloud-cert-vars.yml` → `namespace` (`hclnow-certs`) |

More: `qa-custa.hxtx…` → `qa-custa-hxtx-tls-cert` · `dev.swep.mosa…` →
`dev-swep-mosa-tls-cert`

### Record filters

A record is used only if **all** hold:

1. `dns_full_name` is defined
2. `dns_type` ∈ {A, CNAME} — TXT are ACME challenge tokens, never certs
3. not `_acme-challenge.*`
4. not a wildcard (`*` is invalid in a k8s secret name)
5. host has ≥2 labels
6. client label == this job's client
7. **`env` == this job's env** — a record with no `env` is **skipped, never
   guessed**; a wrong guess would pull a secret from the wrong cluster

### Client selection

From `client-mapping.yml`:

```yaml
client_parent:
  tx:
    git_project: hclnow-client-hxtx
    clients: [hxtx, mosa, txle, hxsa]   # hxsa = testing only
```

The generator uses `client_parent[acm_tx_parent].clients` as the client list
and `.git_project` as the repo directory. It asserts the group resolved, so a
renamed key fails loudly instead of building an empty workflow. If the init
playbook already loaded the file, the in-scope `client_parent` is used.

TX clients have **no `refresh_certs`** in the config repo — group membership
*is* their opt-in, and the flag is bypassed for them only.

### Recipients

Resolved in priority order, for both immediate alerts and the digest:

1. `cert_report_subscriber` from the TX DNS config (YAML list or comma string)
2. `smtp_emails_certs_by_group[acm_cert_group]` in `aws-client-vars-base.yml`
3. `smtp_emails_certs` (default — every non-TX client hits this, unchanged)

The digest needs `acm_tx_dns_file` in its node `extra_data` to read (1).

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
| `acm_cert_source: tx` | Build the TX workflow (client list from `client-mapping.yml`) |
| `acm_tx_parent` | Group key in `client_parent` (default `tx`) |
| `acm_cert_env_whitelist` | Limit scope. Entries are a whole client (`"hxtx"`) or one env (`"hxtx/np"`) |
| `acm_workflow_name_override` | Build a distinct workflow object |
| `acm_schedule_override` | Schedule name (default `Runs daily certs`) |
| `acm_test_ignore_refresh_flag` | Whitelisted envs bypass `refresh_certs` (needs a whitelist) |
| `acm_test_workflow_delete` | Delete the target workflow before rebuilding (clean rebuild) |
| `acm_scm_branch_override` | Pin nodes to an SCM branch (testing) |
| `acm_tx_dns_env` | `test` or `prod` DNS file (default `prod`) |
| `acm_tx_dns_file` | Full DNS file path override |
| `acm_tx_job_template` | Job template name (default: the general cert JT) |
| `acm_digest_inventory` | Digest node inventory |
| `acm_tx_exclude_from_main` | Subtract TX clients from the main workflow |
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

# TX workflow
awx_type: <value>
acm_cert_source: tx
```

Adding a client to the config repo → next generator run **updates** the existing
workflow (same name = idempotent update). Additions are proven; if a client is
ever *removed*, check the visualizer for orphaned nodes and use
`acm_test_workflow_delete: true` for a clean rebuild.

### Job template requirements

**Cert JT** — `ask_extra_vars: true` (mandatory: without it AWX drops node
`extra_data` and TX sourcing silently never happens), `ask_inventory: true`
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

### Submodule handling (TX)

`hclnow-client-hxtx` is a submodule of `aws-v9-automation`. This tower has
**track submodules enabled**, so the `branch` field in `.gitmodules` decides
what gets checked out — not the pinned commit. To test a branch, set
`.gitmodules` → `branch = <branch>`, commit **and push**, then sync the AWX
project. Revert to `master` before production.

---

## 10. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Nodes all hang off START, no links | Node creation failed mid-build (usually a missing inventory) → play aborted before the relation pass | Fix/whitelist inventories, rebuild with `acm_test_workflow_delete: true` |
| Digest: "does not match any hosts" | `limit` set on the digest JT | Clear it |
| Digest asserts `expected_reports` missing | Digest JT has prompt-on-launch Variables disabled | Enable it |
| Green client job but NO-report row | Job never reached the publish task | Check that env's job log |
| False NO-report despite artifact published | Artifact key mismatch | Verify the node carries `acm_report_client`/`acm_report_env` |
| TX run derives 0 certificates | Wrong DNS file (`acm_tx_dns_env`), SCM branch without the TX code, or client/env filter mismatch | Read the "Show TX certificates derived" debug line — it prints the file path, top-level keys, record count and skipped records |
| Every cert reads `expiring_soon` | `Set certificate status` missing the `item.rc is defined` guard | Add it |
| "recursive loop detected in template string" | A task-level `vars:` entry defaulting **from a var of the same name** is self-referential | Inline the default inside the expression, or use a distinct name |
| `digest_node is undefined` | Two `vars:` keys on the same task — YAML keeps only the last | Merge into one `vars:` block |
| Generator can't resolve the TX group | `client-mapping.yml` moved or keys renamed | The assert names the expected path/keys |
| SMTP "Username and Password sent without encryption" | Relay session without STARTTLS (pre-existing) | Accepted for the internal relay; add `secure: starttls` if supported |
| No email on a day you expected one | All four buckets empty | By design — check the "Digest summary" debug line |

---

## 11. Known gaps / future work

1. **Generalisation** — the group mechanism is keyed to the literal `'tx'`
   (`acm_cert_source == 'tx'` in generator and parent playbook). A second group
   with the same repo shape needs a small refactor: treat `acm_cert_source` as
   the group *name*. Then a new client group is pure data.
2. **Latent self-reference** — `acm_tx_dns_file` still uses the
   `vars:`-defaulting-from-itself pattern. It works only because the generator
   always passes it as an extra var; an ad-hoc TX run without it would hit the
   recursion described in §10.
3. **Region scope** — one region per client-env. CloudFront certs in
   `us-east-1` won't appear for clients running elsewhere.
4. **Digest is a single point of failure** — if the digest job itself dies, the
   day's reporting vanishes silently. Attach an AWX on-failure notification
   template to the digest JT and the workflow.
5. **Two-pass workflow creation** — the shared `awx-create-workflow.yml`
   pipeline creates nodes then relations; a failure between them leaves a
   linkless workflow. A single `awx.awx.workflow_job_template` call with the
   full `workflow_nodes` list would be atomic, at the cost of diverging from how
   every other workflow on the tower is built.
6. **`+11h` expiry adjustment** — a hardcoded `39600` magic number in the expiry
   math with no explanation of origin.
7. **`CertificateArn` in the TX DNS records** is currently unused; the engine
   still resolves ACM by domain. Using it would help where the ACM cert's domain
   differs from the DNS name (e.g. a wildcard covering the host).
