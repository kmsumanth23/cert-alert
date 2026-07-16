# AWS ACM Cert Renewal — Consolidated Digest Setup

Redesign of the AWS certificate renewal automation: one consolidated workflow
(per-client branches converging on a digest node) replacing ~25 per-client
daily workflows, and one daily HTML digest email replacing per-cert spam.

## Target topology

```
[hxsa-np] -> [hxsa-prd] ---------.
[mosa-np] -> [mosa-prd] ---------+--> [ACM Certs Digest Report] --> 1 HTML email
[hxco-np] -----------------------'          (converge: ALL)
   ...all other AWS clients...
```

- One branch per client; a client's envs run **sequentially** (always-linked,
  preserving today's per-client serialization).
- Clients run in **parallel** as independent branches.
- The digest node has `all_parents_must_converge: true` and is reached via
  **always** links, so it runs after every branch finishes — including failed ones.
- Each client-env job publishes a sanitized report artifact
  (`acm_report_<client_code>_<client_env>`) via `set_stats`; AWX propagates
  artifacts down the DAG, so the digest job receives all of them.

## Email behavior after this change

| Event | Before | After |
|---|---|---|
| Cert expiring soon | 1 email per cert per env per day | Digest row only |
| Cert renewed | 1 email per cert | Digest row only |
| Import failed | 1 email per cert | **Immediate email (kept)** + digest row |
| Chain verify failed | 1 email per cert | **Immediate email (kept)** + digest row |
| Domain not found in ACM | silent | Digest row (`not_found_in_acm`) — **new** |
| Client job died before reporting | silent | Digest "NO report" row — **new** |
| Everything healthy | (no email) | No email |

## File map

| File in this package | Destination |
|---|---|
| `aws-v9-automation/tasks/aws-cert-import.yml` | `aws-v9-automation/tasks/aws-cert-import.yml` (replace) |
| `aws-v9-automation/acm-cert-digest.yml` | `aws-v9-automation/` — place next to the repo's other playbooks; adjust `vars_files` path inside (see step 2) |
| `aws-v9-automation/templates/acm_digest_mail.html.j2` | `aws-v9-automation/templates/acm_digest_mail.html.j2` |
| `gcp-shared-services/playbooks/awx/kube/awx-workflow-certs-aws.yml` | `gcp-shared-services/playbooks/awx/kube/awx-workflow-certs-aws.yml` (new file) |

## Setup steps

### 1. `aws-cert-import.yml` (replace existing)

Diff vs current version — review before replacing:

- **Removed:** per-cert emails for `renewed` and `expiring_soon` (the old
  "Notify user about certificate status" task is narrowed to `import_failed`
  only and renamed "Notify immediately on import failure").
- **Kept:** immediate emails for `chain_verify_failed` and `import_failed`.
- **Added (end of file):** three tasks building a whitelisted report
  (`domain`, `cert_arn`, `not_after`, `days_left`, `status`, `error` — never
  PEM material) plus a `set_stats` publish keyed
  `acm_report_{{ client_code }}_{{ client_env }}`. Requires `client_code` and
  `client_env` inventory vars (confirmed present in client inventories).

### 2. Digest playbook + job template

1. Commit `acm-cert-digest.yml` and `templates/acm_digest_mail.html.j2` to
   `aws-v9-automation`. Fix the `vars_files` path in the playbook to wherever
   `aws-client-vars-base.yml` actually lives relative to the playbook
   (it provides `smtp_server`, `smtp_port`, `smtp_auth_user`,
   `smtp_auth_pswd`, `smtp_default_sender`, `smtp_emails_certs`,
   `cloudops_group_emails`).
2. Create job template **`ACM Certs Digest Report`** in AWX (name must match
   the generator exactly):
   - Project: the `aws-v9-automation` project; Playbook: `acm-cert-digest.yml`
   - Inventory: a shared-services inventory whose host/EE can reach the SMTP
     server (e.g. `inventory_client_hxss_awsgss`) — the mail task delegates to
     localhost, so the EE/controller needs SMTP reachability
   - **Prompt on launch → Variables: ENABLED** — required, otherwise AWX
     silently drops the `expected_reports` extra_data from the workflow node
     (the playbook asserts on it, so a misconfig fails loudly, not silently)
   - Credentials: none needed beyond source control (no AWS access required)
3. For testing: add your own address to the recipients in
   `aws-client-vars-base.yml` on your test branch (or temporarily override
   `smtp_emails_certs` on the test job template).

### 3. Workflow generator

1. Commit `awx-workflow-certs-aws.yml` to `gcp-shared-services/playbooks/awx/kube/`.
2. Include it **once per generator run** from the same play that currently
   includes `awx-workflow-certs.yml` — *outside* the per-client loop (it
   discovers all clients itself from `{{ hclnow_config_client_env }}`).
3. In `awx-workflow-certs.yml` (the existing per-client generator), skip the
   AWS path once cutover happens — see rollout below.
4. Verify `awx-create-workflow.yml` passes node dicts through to the AWX
   module unmodified. If it maps fields explicitly, add
   `all_parents_must_converge` to the mapping
   (`awx.awx.workflow_job_template`'s `workflow_nodes` and the
   `workflow_job_template_node` module both support it).
5. Set `acm_digest_inventory` if the default shared-services inventory name
   in the file isn't right for your AWX.

**Version note:** `all_parents_must_converge` requires AWX ≥ 19.2 / AAP ≥ 2.1.
The PoC below verifies it empirically on your controller.

## PoC plan (test branch, before touching production workflows)

1. **Artifact only:** deploy the modified `aws-cert-import.yml` to the test
   branch. It changes no behavior except fewer emails + the `set_stats` at the
   end. Run one client's existing workflow; open the job in AWX → the
   *Artifacts* panel should show `acm_report_<client>_<env>` with sanitized
   rows only (no PEM blocks — verify this explicitly).
2. **Digest job template:** create it per step 2, recipient = you.
3. **Hand-build a test workflow in the AWX UI** (skip the generator for now):
   2 clients × their envs, chained per client, both branches always-linked to
   the digest node, "All parents must converge" checked. Paste into the digest
   node's extra_data:
   ```yaml
   expected_reports:
     - {client: hxsa, env: awsv9m, infra_code: v9m}
     - {client: mosa, env: np, infra_code: v9}
   ```
4. Run it. Verify: digest job's extra_vars contain both `acm_report_*` keys;
   one HTML email arrives; renders correctly in Outlook + Gmail.
5. **Failure drill:** break one branch (e.g. bad credential) and re-run. The
   digest must still send, with that client-env under "NO report".
6. **Duplicate-run sanity:** run the workflow twice back-to-back; confirm the
   second run's digest shows the renewed cert as healthy (fresh expiry), not
   re-renewed.
7. **Generator:** run `awx-workflow-certs-aws.yml`, diff the generated
   workflow against your hand-built one in the AWX UI.

## Rollout / cutover

1. Merge; let the consolidated workflow run on its daily schedule **in
   parallel with** the old per-client workflows for 2–3 days. Overlap is
   safe-ish (a second run sees the fresh post-import expiry, so no double
   renewal) but failures would alert twice — keep the overlap short.
2. Point real recipients at the digest (`smtp_emails_certs` already used).
3. Decommission the old per-client AWS workflows: either disable their
   schedules in AWX, or teach the per-client generator to emit them with
   `state: absent` for `cloud_provider == 'aws'`. Leave GCP untouched.

## Requirement 3 — delegation (no new code needed)

The renewal logic lives in one job template parameterized purely by inventory,
so delegation is AWX RBAC:

- Grant the team **Execute** on `Import Certificate to ACM in AWS`.
- Grant the team **Use** on *only their* `inventory_client_<client>_<env>`
  inventories.
- Either enable `ask_inventory_on_launch` (AWX limits the picker to
  inventories the user can Use), or — safer and consistent with your
  objects-as-code pattern — generate thin per-team job templates with pinned
  inventories.
- Keep **Prompt on launch → Variables disabled** on the renewal template's
  delegated copies so nobody injects extra vars.
- Ad-hoc team runs happen outside the consolidated workflow, so they won't
  appear in that day's digest; immediate failure alerts still fire, and the
  next daily digest reflects true ACM state.

## Requirement 2 — bulk execution (future, cheap now)

The generator's discovery loop is the hook: add an env filter (e.g.
`env != 'prd'`) and emit a second, unscheduled workflow
"renew certs - aws - all nonprod" for on-demand launch. Same digest node
pattern applies.
