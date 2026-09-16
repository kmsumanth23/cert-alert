# ACM Certificate Renewal & Reporting — Reference and Troubleshooting

Complete documentation for the consolidated AWS ACM certificate automation (v2).
Written to be read cold, months later, by someone who has forgotten how it works
— including the people who built it.

**Two ways to use this file:**

- *"What does this task actually do?"* → §5–§10, one section per play, every task explained.
- *"It's broken."* → §13 Troubleshooting, symptom → cause → fix.

---

## 1. What this system does

Replaces ~25 independent per-client daily AWX workflows (each emailing per
certificate) with **one generated workflow per client group** that renews
certificates and sends **one consolidated HTML digest**.

| | Before | v1 | v2 (now) |
|---|---|---|---|
| Workflows | ~25, one per client | 1 main + 1 per group | same |
| Emails/day | 1 per cert per env | 1 digest + hard failures | same |
| Reported | configured certs only | configured certs only | **every cert AWS will not renew** |
| Renewed | configured certs | configured certs | **configured certs** (unchanged) |
| Shared certs | N rows | N rows | **1 row, envs listed** |
| CloudFront certs | invisible | invisible | **swept when `cdn_certs_enabled`** |

---

## 2. The single most important distinction

**Report scope ≠ renewal scope.** Get this wrong and nothing else makes sense.

| | Report scope | Renewal scope |
|---|---|---|
| What | every ACM cert in the account that AWS will **not** auto-renew | only certs listed in `tools.yml` / the group's DNS records |
| How found | account-wide sweep (`acm_fetch_all_certs`, default **on**) | one ACM lookup per configured domain |
| Marked | `renewal: manual` in the report | `renewal: automated` |
| Can be written to | **never** — `renewal_due` is hard-`false` | yes, when inside the renewal threshold |

The sweep exists for *visibility*: certificates nobody tracks are the ones that
silently expire. It has no Kubernetes secret behind it, so it can never renew
anything. That guard is asserted by VERIFY J4 on every verification run.

---

## 3. The four plays and how they chain

```
  "Create cert renewal workflow template" JT   (job tags: common, certs)
        │
        │  runs the GENERATOR  →  awx-workflow-certs-aws.yml
        │  which BUILDS this workflow object in AWX:
        ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  renew certs - aws - <group|all> clients                      │
  │                                                               │
  │  START ─► [renew-cert-hxsa-awsdevv9] ─► [renew-cert-hxsa-…] ──┐│
  │  START ─► [renew-cert-hxtx-np] ───────────────────────────────┼┼─► [node-digest-report]
  │  START ─► [renew-cert-mosa-np] ─► [renew-cert-mosa-prd] ──────┘│      (converge: ALL)
  └──────────────────────────────────────────────────────────────┘
        │                                               │
        │ each node runs the PARENT PLAYBOOK            │ runs the DIGEST
        │ awsacmcertimportauto.yml                      │ acm-cert-digest.yml
        │   └─ includes the ENGINE                      │
        │      tasks/aws-cert-import.yml                └─► ONE HTML email
        │         └─ set_stats artifact ────────────────────┘
```

| Play | Deployed path | Runs where | Role |
|---|---|---|---|
| Generator | `gcp-shared-services/playbooks/awx/kube/awx-workflow-certs-aws.yml` | generator JT | discovers clients, **builds** the workflow |
| Parent | `aws-v9-automation/awsacmcertimportauto.yml` | once per client-env node | bastion prep, decides where `certificates` comes from |
| Engine | `aws-v9-automation/tasks/aws-cert-import.yml` | included by the parent | ACM lookup, sweep, renewal, artifact |
| Digest | `aws-v9-automation/acm-cert-digest.yml` | convergence node | merge artifacts, dedupe, one email |

- **One branch per client**; that client's envs run **sequentially**
  (`always_nodes`) so two envs never hit the same bastion at once.
- **Clients run in parallel** as independent branches.
- Links are `always` type — a failed env never blocks the digest.
- The digest node sets `all_parents_must_converge: true`, so it waits for every
  branch, including failed ones.

---

## 4. How clients are selected (the thing everyone forgets)

Nothing lists clients by hand. Discovery walks the config repo:

```
hclnow-config-client-environments/clients/<client>/<env>/aws-client-vars-env.yml
                                                          └─ refresh_certs: true
```

**`refresh_certs: true` in that file is the opt-in.** The generator walks every
`<client>/<env>` directory, reads that file, and keeps the ones where the flag is
set. That list becomes both the workflow nodes *and* the digest's
`expected_reports` manifest — which is how silence is distinguished from health.

Four things change that set:

| Var | Effect |
|---|---|
| `refresh_certs: true` | the normal opt-in, per client-env, in the config repo |
| `acm_cert_group: <g>` | **group mode** — take that group's `clients` list from `client-mapping.yml` and bypass `refresh_certs` for them (group membership *is* their opt-in) |
| `acm_cert_env_whitelist` | narrow to listed clients (`"hxtx"`) or envs (`"hxtx/np"`) |
| `acm_test_ignore_refresh_flag` | let whitelisted envs in without the flag (test towers; inert without a whitelist) |

> ⚠️ A whitelist **without** `acm_workflow_name_override` rebuilds the **main**
> workflow containing only those clients, dropping everyone else.

---

## 5. Generator — `awx-workflow-certs-aws.yml`, task by task

Builds the AWX workflow object. Runs once per generation, not per client.

| Task | What it does |
|---|---|
| **Reset AWS consolidated cert workflow facts** | Clears accumulators and sets `acm_group_mode` — true whenever `acm_cert_group` is supplied. The group name exists only as data; no group name appears anywhere in code. |
| **Load the central client mapping** | Reads `client_parent` from `common/vars/client-mapping.yml`, or uses it if the init playbook already loaded it. Group mode only. |
| **Resolve configuration for cert group** | Picks `client_parent[<group>]` — its `clients`, `git_project`, and optional overrides. |
| **Assert the cert group resolved** | Fails loudly, listing the known groups, if the key is missing. Stops a typo producing an empty workflow. |
| **Resolve workflow, schedule and job-template names** | Three-level fallback each: explicit override var → mapping entry → default. This is why a group can have its own JT, workflow name or schedule with no code change. |
| **Resolve group data-file paths** | Builds the path to the group's records file inside the **job's** checkout — so `git_project` must be a submodule of `aws-v9-automation`. |
| **Default group data-file paths in main mode** | Blanks them; nothing reads them outside group mode. |
| **Print resolved cert group configuration** | One line with every resolved value. **First thing to read when a group build looks wrong.** |
| **Discover AWS client environments with refresh_certs enabled** | §4. Walks the config repo with `filetree`, applies the flag / group / whitelist conditions, produces `aws_cert_envs`. |
| **Print discovered AWS cert client-envs** | The list that becomes both the nodes and `expected_reports`. |
| **Build per-client chained workflow nodes** | Groups by client, sorts envs, chains each env to the next via `always_nodes`, last env to the digest. Stamps `extra_data`: `acm_report_client` / `acm_report_env` (the artifact-key contract) plus the group vars in group mode. |
| **Append digest report convergence node** | Adds `node-digest-report` with `all_parents_must_converge: true` and `expected_reports`. ⚠️ ONE `vars:` block only — a second silently replaces the first and leaves `digest_node` undefined. |
| **Print consolidated workflow nodes** | Full node list before creation. |
| **Prepare consolidated AWS certs workflow template** | Assembles the workflow: name, description, `ask_variables_on_launch: true` (this is what lets you pass `acm_report_only` / `acm_verify` at launch), schedule. |
| **Delete target consolidated workflow before rebuild (opt-in)** | `acm_test_workflow_delete: true`. Use after an aborted build left a linkless workflow. |
| **Create consolidated AWS certs workflow** | Hands off to `awx-create-workflow.yml`. |

**Known weakness:** node creation resolves inventories **by name**. A missing
`inventory_client_<client>_<env>` aborts the play *before* the link pass, leaving
every node dangling off START. Rebuild with `acm_test_workflow_delete: true`
after fixing; never patch a half-built workflow.

---

## 6. Parent playbook — `awsacmcertimportauto.yml`, task by task

Runs on the client's bastion. Prepares AWS CLI access, then decides where the
list of certificates to *renew* comes from.

| Task | What it does |
|---|---|
| **Include the default and override variables** | Standard client var loading. |
| **Check if aws is installed** → **Install AWSCLI** | Installs the CLI if absent. |
| **Configure aws for client env**, **Change ownership** | Creates `~/.aws` for the client user. |
| **Creating a file with content for aws profile** | Root credentials file. |
| **Set profile env var** | Builds `profile_<env_base>` = `srvc<env><client>`. |
| **Creating a file … respectively** | Per-env credentials for the client user. |
| *— grouped clients only; the whole block is gated on `acm_cert_group` —* | |
| **Load cert-group conventions** | `include_vars` the group's cert-vars file into `cert_group_vars_raw`. |
| **Snapshot cert-group conventions** | Renders that dict **once** into `cert_group_vars`, with `dns_config_project` / `awx_env` supplied as task-local shims. This task exists because `include_vars` templates lazily — see §13. |
| **Load cert-group data file** | Reads the group's DNS records file. |
| **Extract group records and alert recipients** | Pulls the records list and `cert_report_subscriber`. |
| **Reset certificates list for group sourcing** | Empties `certificates` — the group repo **replaces** `tools.yml`, it does not add to it. |
| **Build certificates list from group records** | Derives `domain_name` / `secret_name` / `secret_namespace` per record. Skips wildcards, `_acme-challenge`, wrong record types, other clients, and **records with no env field** — the client is derivable from the domain, the env is not, and a wrong guess would pull a secret from the wrong cluster. |
| **Show certificates derived for this client env** | Source file, records read, certs derived, records skipped, recipients. **First thing to read when a group run finds nothing.** |
| *— all clients —* | |
| **include tasks to import cert to acm** | Unconditional in v2. The sweep runs by default, so even an env with zero configured certs has a report to produce — it is the env most likely to be hiding an untracked cert. |
| **Publish empty digest artifact when no certificates configured** | Fallback so "nothing configured here" reads as an empty report rather than a NO-report alarm. Only fires when the sweep is explicitly switched off. |

---

## 7. Engine — `tasks/aws-cert-import.yml`, task by task

The heart of the system. Runs once per client-env.

### 7.1 Setup

| Task | What it does |
|---|---|
| **Get authentication session details** | `sts_assume_role` into the client account. |
| **Set AWS STS credentials** | Stores the temporary key / secret / token. |
| **Set python3 interpreter**, **install boto3 and botocore** | Module prerequisites. |
| **Get current timestamp** | Epoch used by the expiry maths. |

### 7.2 Configured path — the renewal scope

| Task | What it does |
|---|---|
| **Get ACM Certificate Details** | One `acm_certificate_info` call per configured domain. **Deliberately not replaced by the sweep:** the sweep filters out certs AWS renews itself, so a configured domain whose cert happens to be AMAZON_ISSUED + ELIGIBLE would drop out of `certs_info`, be reported `not_found_in_acm`, and stop being renewed. |
| **Resolve immediate-alert recipients for this client group** | `cert_report_subscriber` → `smtp_emails_certs_by_group[group]` → `smtp_emails_certs`. |
| **Define renewal threshold (in days)** | From `client_cert_expiry_notice`. |
| **Build certificate information dictionary** | Keyed **by domain**. Computes `renewal_due` (expiry + 11h − threshold < now), stamps `renewal: automated`, carries `secret_name` / `secret_namespace`. `acm_report_only: true` forces `renewal_due` false for every cert — this is the read-only switch. |

### 7.3 Account-wide sweep — the report scope

| Task | What it does |
|---|---|
| **Resolve sweep regions and status filter** | `[aws_region]`, plus `cdn_cert_region` when `cdn_certs_enabled` is true, deduped. Statuses kept: `ISSUED`, `EXPIRED` — `PENDING_VALIDATION` / `FAILED` / `VALIDATION_TIMED_OUT` are ineligible too, but were never live, so they are noise. |
| **Get ALL ACM certificates in each swept region** | One call per region. |
| **Flatten swept certificates and note what the configured path covered** | Builds `acm_swept_certs` and the list of ARNs the configured path already holds. |
| **Add swept certificates the configured path did not already cover** | Keyed **by ARN**, not domain — the same domain can exist as separate certificates in two regions, and keying by domain would collapse them and hide the copy nobody renews. Filter: not already configured, status in the list, and `renewal_eligibility != 'ELIGIBLE'` — falling back to `type == 'IMPORTED'` only if ACM omits the field, because a strict test would otherwise drop every cert and produce a silently empty sweep. Stamps `renewal: manual`, `renewal_due: false`. |
| **Show sweep result** | Counts and **why** certs were dropped: AWS-renewed, by status, already configured. Read this to explain an unexpected total. If *carrying renewal_eligibility* is 0, the fallback did the filtering and AMAZON_ISSUED + INELIGIBLE certs are still invisible. |

> `certs_info` keys are therefore **mixed**: domains for configured certs, ARNs
> for swept ones. Nothing downstream cares, because every renewal task is gated
> on `renewal_due`, and report rows read `cert.value.domain`, not the key.

### 7.4 Renewal — only for certs where `renewal_due` is true

| Task | What it does |
|---|---|
| **Get Kubernetes certificate secret** | `kubectl get secret <secret_name> -n <namespace>` for the TLS key and cert. |
| **Decode TLS key and certificate** | base64-decodes both halves. |
| **Extract primary and intermediate certificates** | Splits the PEM chain on `-----END CERTIFICATE-----`. |
| **Warn if certificate chain length is outside expected range** | Flags chains outside 2–3 certs (leaf + 1–2 intermediates). |
| **Save decoded private key to a file** | `/data/private/<domain>-cert_key.key`, mode 0400. |
| **Remove private key from cert_info var** | Drops the key from memory so it can never reach an artifact. |
| **Save primary certificate to a file** | `/data/private/<domain>-primary_cert.crt` for the AWS CLI. |
| **Save intermediate certificate to a file** | `/data/private/<domain>-intermediate_cert.crt`. Skipped when the chain has no intermediate. |
| **Verify certificate chain locally before import** | `openssl verify -untrusted <intermediate> <primary>`. `failed_when: false` — the result is recorded, not fatal. |
| **Record chain verification result** | Stamps `chain_verify_failed` / `chain_verify_error`. |
| **Notify on chain verification failure** | **Immediate email**, per certificate. Subject carries `[client/env]` because one cert can be shared by several envs and the domain alone would not say which job failed. |
| **Import renewed certificate to ACM** | `aws acm import-certificate --certificate-arn <existing>` — re-imports **to the same ARN**, so nothing downstream needs re-pointing. Skipped when chain verification failed. |

### 7.5 Status and cleanup

| Task | What it does |
|---|---|
| **Set certificate status** | `renewed` / `import_failed`. ⚠️ Requires `item.rc is defined` — without that guard, skipped items fall through to a default and label **every** certificate `expiring_soon`. |
| **Set status for certificates that failed chain verification** | Stamps `chain_verify_failed` explicitly; the import task skipped them, so otherwise they fall through. |
| **Mark certificates as expiring soon or expired** | For everything not already carrying a renewal outcome: `expired` if past, else `expiring_soon` if inside `client_cert_expiry_notice`. |
| **Notify immediately on import failure** | **Immediate email**, per certificate, `[client/env]` in the subject. |
| **Remove temporary private key file** | Deletes the 0400 key file. |
| **Remove temporary primary certificate file** | Cleanup; skipped when chain verification failed, so the file stays for inspection. |
| **Remove temporary intermediate certificate file** | Same, for the intermediate. |

### 7.6 Artifact

| Task | What it does |
|---|---|
| **Note configured domains with no matching certificate in ACM** | Compared against **configured domains only**, since `certs_info` now also holds ARN-keyed sweep rows. |
| **Build sanitized report rows for consolidated digest** | Explicit **whitelist**: `domain`, `renewal`, `cert_arn`, `not_after`, `days_left`, `status`, `error`, `in_use`. `certs_info` still holds PEM material — never publish it wholesale; artifacts land in the AWX database and job detail UI. |
| **Add report rows for configured domains not found in ACM** | `not_found_in_acm`, no ARN. |
| **Publish consolidated digest artifact** | `set_stats` under key `acm_report_<client>_<env>`, built from `acm_report_client` / `acm_report_env` (node `extra_data`) so inventory-var drift cannot cause a false NO-report. |
| **Verify this job's certificate handling** | Inert unless `acm_verify: true`. §10. |

---

## 8. Digest — `acm-cert-digest.yml`, task by task

Runs on the **implicit localhost** of the execution environment. The JT's
inventory contents are irrelevant, but its **Limit field must be empty** or the
play matches no hosts.

| Task | What it does |
|---|---|
| **Assert expected_reports manifest was provided** | Without it the digest cannot tell "healthy" from "never reported". |
| **Initialise digest buckets** | Empty `digest_certs`, `missing_reports`. |
| **Collect artifact from each expected client-env** | Looks up `acm_report_<client>_<env>` for every expected env; envs with no artifact go to `missing_reports`. Stamps `client` / `env` onto each row. |
| **Stamp a dedupe key on every collected row** | Key = **ARN + status**. Status is in the key on purpose: if envs a and b renewed a shared cert and c failed chain verification, a cert-only key produces one row claiming all three failed and loses the successful renewal entirely. ARN-less rows (`not_found_in_acm`) key per client-env, since the same domain missing in two accounts is two real findings. |
| **Collapse rows that share a certificate** | Merges each group — worst status wins, first non-empty error survives, `automated` beats `manual`, lowest `days_left` wins, and `client_env_label` lists the envs (`hxtx / np, prd`). |
| **Replace the collected rows with the collapsed set** | Swaps in the merged list and records `digest_duplicates_collapsed`. That count is measured against the *keyed* list, not `digest_certs`, because this same `set_fact` reassigns `digest_certs` and key evaluation order is not guaranteed. |
| **Classify failed and renewed certificates** | `failed_rows` = import / chain / not-found; `renewed_rows` = renewed. |
| **Identify certificates expiring within N days** | Loop + `when` rather than `selectattr`, because `days_left` can be a string and a lexicographic compare would be wrong. Normalises to int here. |
| **Verify the consolidated digest** | Inert unless `acm_verify: true`. Must sit **after both** the collapse (so it sees merged rows) and the classification (so `failed_rows` exists). |
| **Show digest summary in job output** | One line: counts, distinct certs, duplicates collapsed. |
| **Nothing to report — end without sending email** | All four buckets empty → no email. Healthy days are silent by design. |
| **Load group config for recipients** | Reads the group's data file for `cert_report_subscriber`. |
| **Resolve digest recipients for this client group** | group subscriber → `smtp_emails_certs_by_group` → `smtp_emails_certs`. Accepts a YAML list or a comma-separated string. |
| **Show which recipient list is in use** | Read this when mail reaches the wrong people. |
| **Send consolidated digest email** | Renders `acm_digest_mail.html.j2`. |

---

## 9. Email template — `acm_digest_mail.html.j2`

Inline styles only; mail clients strip `<style>` blocks.

**Columns:** Client / Env · Domain (+ console link) · Expiry · Days left · In use · Renewal · Status

| Element | Rule |
|---|---|
| Colour bands | ≤7 days `#ff9999`, ≤10 `#fdecea`, 11–15 `#fff8e1`. Applied to the **Expiring table only** — `days_left` is the expiry ACM reported *before* the import, so a successfully renewed cert still carries the old date and would otherwise render red in the Renewed table. |
| Client / Env | `client_env_label` from the dedupe, falling back to the single pair for older artifacts. |
| Domain | **Plain text on purpose** — mail clients auto-linkify hostnames and would fight an anchor of ours. |
| Console link | On the line beneath, built from the ARN: `region · account · certid ↗`. Rows with no ARN render no link. |
| Renewal | `Automated` (this workflow renews it) / `Manual` (nobody does — someone must act). |
| In use | From ACM's `in_use_by`. An empty list means the cert is attached to nothing, so an expiry there is usually harmless. |

**Console links and the account problem.** A plain console URL carries **no
account context** — it opens in whichever account the reader's browser session
already holds, so with ~30 accounts every link lands in the same one and reports
"certificate not found". Set these and links route through the IAM Identity
Center portal, which switches account first:

```yaml
aws_console_role: HCLSW_SAAS_ADMIN        # PERMISSION SET name, not the AWSReservedSSO_ role
aws_console_portal: hclsw-aws-saasmstr    # the label before .awsapps.com
```

Unset → falls back to the plain URL with the account id shown beside it. Readers
who lack that permission set land on the portal rather than the certificate — a
soft failure, not a broken link.

---

## 10. Verification tasks

Both are inert unless `acm_verify: true` is passed at workflow launch. A failing
assert stops that branch with the reason in the log, so **the run is the
checklist**. Run read-only the first time:
`acm_verify: true, acm_report_only: true`.

**`tasks/acm-verify-job.yml`** — per client-env:

| | Catches |
|---|---|
| J1 | configured path did not run → renewal scope dead |
| J2 | sweep never resolved its regions |
| **J3** | **ACM not returning `renewal_eligibility`** → the IMPORTED fallback filtered, so AMAZON_ISSUED + INELIGIBLE certs are still invisible. Everything else looks perfect when this fails. |
| J4 | a swept cert marked renewable → would attempt to renew a cert with no secret behind it. **Stop the run.** |
| J5 | report rows missing v2 fields → stale engine on that branch |
| J6 | a configured domain reported nowhere and not flagged not-found |

**`tasks/acm-verify-digest.yml`** — aggregate:

| | Catches |
|---|---|
| D1 | dedupe did not cover every row |
| D2 | duplicate cert/status pairs remain (same ARN with *different* statuses is correct and expected) |
| D3 | a row in a bucket that does not match its status → dedupe moved relative to classification |
| D4 | a `manual` cert carrying a renewal outcome |
| D5 | non-numeric `days_left` |
| D6 | an expected client-env neither reported nor listed as missing |
| warnings | `digest_expiry_days ≠ 15`, `aws_console_role` unset, no manual certs found — reported, never fatal |

**Still needs human eyes:** colour bands render correctly, a renewed row is
green, the Client/Env column is merged, and a console link opens the right
**account**.

---

## 11. Variable reference

### Config repo — `clients/<c>/<env>/aws-client-vars-env.yml`

| Var | Effect |
|---|---|
| `refresh_certs: true` | **the opt-in** — this client-env joins the main workflow |
| `cdn_certs_enabled: true` | this client-env holds CloudFront certs → also sweep `cdn_cert_region` |
| `client_infra_type` | carried into node `extra_data` as `infra_code` |

### `aws-client-vars-base.yml` (aws-v9-automation)

| Var | Effect |
|---|---|
| `client_cert_expiry_notice` | **renewal** threshold in days — when `renewal_due` fires |
| `digest_expiry_days` | **report** window in days — must be `15` for go-live |
| `cdn_cert_region` | default `us-east-1`; AWS requires CloudFront certs to live there |
| `aws_console_role` / `aws_console_portal` | Identity Center deep link (§9) |
| `smtp_*`, `smtp_emails_certs`, `smtp_emails_certs_by_group`, `cloudops_group_emails` | mail |

### Generator launch vars

| Var | Effect |
|---|---|
| `acm_cert_group: <g>` | build that group's workflow; the value **is** the key in `client-mapping.yml` |
| `acm_cert_env_whitelist` | narrow scope — `"hxtx"` or `"hxtx/np"` |
| `acm_workflow_name_override` | build a separate workflow object |
| `acm_schedule_override`, `acm_job_template_override`, `acm_digest_inventory` | name overrides |
| `acm_test_ignore_refresh_flag` | whitelisted envs bypass `refresh_certs` |
| `acm_test_workflow_delete` | delete before rebuild (clean rebuild) |
| `acm_scm_branch_override` | pin **client** nodes to a branch. ⚠️ does **not** reach the digest node — set the branch on the digest JT itself |
| `acm_group_dns_env`, `acm_group_dns_file`, `acm_group_conventions`, `acm_client_mapping_file` | group data overrides |
| `acm_group_exclude_from_main` | subtract every grouped client from the main workflow |

### Workflow launch vars (requires `ask_variables_on_launch`)

| Var | Effect |
|---|---|
| `acm_report_only: true` | force `renewal_due` false — report + digest, no ACM writes |
| `acm_verify: true` | run the verification asserts (§10) |
| `acm_fetch_all_certs: false` | **opt out** of the sweep (default is on) |
| `digest_expiry_days: <n>` | widen or narrow the report window for one run |

> **Precedence:** node `extra_data` (written by the generator) **outranks** the
> launch prompt. Anything pinned at generation time cannot be overridden at run
> time — which is why `acm_report_only` is baked in only when explicitly passed
> to the generator.

---

## 12. Certificate statuses

| Status | Meaning |
|---|---|
| `renewed` | Successfully re-imported to ACM this run |
| `import_failed` | `aws acm import-certificate` returned non-zero |
| `chain_verify_failed` | Local `openssl verify` failed; import deliberately skipped |
| `not_found_in_acm` | Domain configured but no matching certificate in ACM |
| `expired` | Past expiry |
| `expiring_soon` | Within `client_cert_expiry_notice` days |
| `ok` | Outside the notice window |

| Event | Email |
|---|---|
| Chain-verify failure | **Immediate**, per cert (+ digest row) |
| Import failure | **Immediate**, per cert (+ digest row) |
| Expiring / renewed / no-report | Digest only |
| All buckets empty | **No email** |

---

## 13. Troubleshooting

### Workflow build

| Symptom | Cause | Fix |
|---|---|---|
| Nodes all hang off START, no links | Node creation failed mid-build — almost always a missing `inventory_client_<c>_<e>` — aborting before the link pass | Fix or whitelist the inventory, rebuild with `acm_test_workflow_delete: true` |
| Generator cannot resolve the group | `client-mapping.yml` moved or keys renamed | The assert names the expected path and keys |
| The main workflow lost most of its clients | A whitelist was passed **without** `acm_workflow_name_override` | Rebuild without the whitelist |
| Whole "Initialize resources" JT fails on missing inventories | The generator include shares the `initialize` tag | Remove `initialize` from that include (both `tags:` and `apply: tags:`); run generation from a JT tagged `certs` |

### Cert job

| Symptom | Cause | Fix |
|---|---|---|
| `dns_config_project is undefined` | `include_vars` templates values **lazily** — reading one key renders the whole file, including entries referencing DNS-job variables | *Snapshot cert-group conventions* shims `dns_config_project` / `awx_env`. A **third** foreign reference fails in that same task — add the shim there, never on the consumers |
| Vault / Python error reading the cert-vars file | `lookup('file') \| from_yaml` cannot decrypt inline `!vault` values | Use `include_vars`, which decrypts. Do **not** revert to the lookup |
| Group run derives 0 certificates | Wrong data file (`acm_group_dns_env`), SCM branch without the group code, or a client/env filter mismatch | Read **"Show certificates derived for this client env"** — it prints the file path, record count, derived names, skipped records and recipients |
| Every cert reads `expiring_soon` | `Set certificate status` missing the `item.rc is defined` guard | Add it |
| More certs than the console shows | The console view paginates/filters, or several envs share one account and each reports it | Compare against `aws acm list-certificates --query 'length(CertificateSummaryList)'` for the same account + region |
| Sweep returns nothing | `acm_fetch_all_certs` explicitly `false`, or everything filtered | Read **"Show sweep result"** — it prints the drop reason per category |
| VERIFY J3 fails | ACM / the collection is not returning `renewal_eligibility`; the `type == IMPORTED` fallback filtered | AMAZON_ISSUED + INELIGIBLE certs are invisible. Raise it rather than passing the run |
| VERIFY J4 fails | A swept cert is marked `renewal_due` | **Stop the run** — it would try to renew a certificate with no secret behind it |
| CloudFront certs missing from the report | `cdn_certs_enabled` not set for that client-env, or set to a string | `cdn_certs_enabled: true`. A region string there evaluates to **false** through `\| bool` and silently disables the sweep |

### Digest

| Symptom | Cause | Fix |
|---|---|---|
| "does not match any hosts" | A `limit` is set on the digest JT | Clear it — the play targets localhost |
| Asserts `expected_reports` missing | Digest JT has prompt-on-launch Variables disabled | Enable it |
| `failed_rows is undefined` in verification | The verify include sits **before** `Classify failed and renewed certificates` | Move it after classification (and after the collapse) |
| Green client job but a NO-report row | That job never reached the publish task | Check that env's job log |
| False NO-report despite an artifact | Artifact key mismatch | Confirm the node carries `acm_report_client` / `acm_report_env` |
| A shared cert appears once per env | Dedupe not running, or rows missing `cert_arn` | Check `digest_duplicates_collapsed` in the header, and VERIFY D1 / D2 |
| Report header says "365 days" | `digest_expiry_days` left at a test value | Set it back to `15` |
| Mail reached the wrong list | Recipient chain fell through | Read **"Show which recipient list is in use"** |
| No email at all | All four buckets empty | By design — check the "Digest summary" line |

### Email rendering

| Symptom | Cause | Fix |
|---|---|---|
| Console link opens the wrong account | A plain console URL carries no account context | Set `aws_console_role` + `aws_console_portal` (§9) |
| `awsapps.com` will not resolve | Wrong portal label | Copy the host from the address bar while on the portal; only the label before `.awsapps.com` goes in the var |
| A variable reads `(UNSET)` in the template | Name mismatch, or defined inside a nested block in the vars file | Confirm it is a **top-level** key, spelled exactly as the template reads it |
| Renewal column or links blank | Rows published by an older engine | Redeploy the engine; VERIFY J5 catches this |

### Jinja / Ansible traps — every one of these has bitten this project

| Trap | Rule |
|---|---|
| Self-referential default | `vars: x: "{{ x \| default(...) }}"` → "recursive loop detected". Inline the default, or use a distinct name |
| Duplicate `vars:` key | YAML keeps only the last; the earlier block vanishes silently |
| `include_vars` lazy templating | Reading one key renders the **whole** file |
| Macro scoping | A Jinja macro does not reliably see playbook variables — resolve at template top level and pass them **into** the macro as arguments |
| `set_fact` key ordering | Keys in one `set_fact` may not evaluate in order — never read a variable the same task is reassigning |
| `\| bool` on a region string | `'us-east-1' \| bool` is **false**. A boolean flag must hold a boolean |
| `\| default([])` hiding undefined | Silently returns empty instead of failing — the reason a partial merge once read 0 DNS records |
| Anchoring patches | Anchor by task name, never line number — files drift |

---

## 14. Onboarding a new client group

1. **Add the repo as a submodule of `aws-v9-automation`** — not of
   `gcp-shared-services`; the cert job reads from its own checkout. Track
   submodules is on, so `.gitmodules` → `branch` decides what is checked out,
   not the pinned commit.
2. **Add the group** to `common/vars/client-mapping.yml`:
   ```yaml
   client_parent:
     <group>:
       git_project: <submodule dir>
       clients: [<client>, ...]
       # optional: cert_conventions / dns_file / job_template / workflow_name / schedule
   ```
3. **Check the cert-vars file is safe to include** — the step that bit us:
   ```bash
   grep -o '{{[^}]*}}' <cert_vars_file> | sort -u
   ```
   Only `base_src`, `client_code`, `client_env` are in the cert job's scope.
   Anything else needs a shim in *Snapshot cert-group conventions*.
4. **Confirm every `inventory_client_<client>_<env>` exists** in AWX, or the
   build aborts linkless.
5. **Attach a Vault credential** to the Import JT if the cert-vars file holds
   vaulted values.
6. **Records must carry an explicit env field.** Client is derivable from the
   domain; env is not, and records without it are skipped rather than guessed.
7. **Generate scoped to one env**, read-only, on a test branch:
   ```yaml
   acm_cert_group: "<group>"
   acm_cert_env_whitelist: ["<client>/<env>"]
   acm_test_ignore_refresh_flag: true
   acm_group_dns_env: "test"
   acm_workflow_name_override: "TEST renew certs - aws - <group>"
   acm_test_workflow_delete: true
   acm_scm_branch_override: "<test branch>"
   ```
8. **Verify derived secret names** against `kubectl get secrets -n <namespace>`.
   A wrong `valid_suffix` or `secret_name_suffix` produces plausible names that
   simply do not exist — read-only runs look identical either way. This is the
   gate before enabling renewal.
9. **Widen** to the whole group, drop `acm_report_only`, then regenerate without
   the test overrides so the schedule attaches.

---

## 15. Known gaps / future work

1. **The generator trusts the config repo for inventories.** A missing inventory
   aborts the build. A pre-flight check against AWX would make it self-healing.
2. **The digest is a single point of failure for all reporting.** If it dies,
   nothing is reported at all. Attach an AWX on-failure notification to that JT.
3. **Immediate failure alerts are per certificate.** Five failures produce five
   emails plus the digest. Gating them behind a variable (default off) is
   designed but not implemented — the digest already carries every failure row.
4. **`acm_scm_branch_override` does not reach the digest node.** Set the branch
   on the digest JT itself when testing.
5. **Shared certs across envs pull from different clusters.** Several envs may
   import to the same ARN in one run; if their clusters hold different certs for
   that domain, the last write wins.
6. **Volume.** With the sweep on, `digest_certs` carries every ineligible cert
   across all accounts before the expiry filter. If the digest job slows, filter
   to the expiry window before the dedupe rather than after.
7. **The `+11h` constant** in the expiry maths is unexplained — document or
   remove it.
8. **Renewal eligibility column** deferred to v3.
