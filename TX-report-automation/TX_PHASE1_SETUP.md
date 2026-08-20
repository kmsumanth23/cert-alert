# TX Clients — Phase 1 (read-only report) setup

Phase 1 gives the TX clients their own workflow, sourcing certificate details
from the TX repo, and produces a **read-only** digest — no k8s access, no ACM
writes. Phase 2 (enabling renewal) waits on the env-mapping answer.

Everything is **additive and gated**: without `acm_cert_source: tx`, the
generator and the cert playbook behave exactly as they do in production today.

---

## What changed in the code

| File | Change |
|---|---|
| `awxworkflowcertsaws-final.yml` (generator) | New **TX mode**: `acm_cert_source: tx` reads the group from `playbooks/awx/kube/common/vars/client-mapping.yml` → `client_parent.tx.{clients,git_project}`, bypasses `refresh_certs` for those clients only, points nodes at the TX job template, defaults the workflow name to `renew certs - aws - tx clients`, and injects `acm_certs_source` / `acm_tx_dns_file` / `acm_report_only` into each node |
| `aws-acm-cert-import-auto.yml` (parent playbook) | New gated block **before** the cert include that builds `certificates` from the TX DNS file (domain + derived secret name + `hclnow-certs` namespace) |
| `awscertimport.yml` (cert engine) | **No change** — see the one-line read-only guard below |

## Prerequisites

1. **Submodule** — add `hclnow-client-hxtx` to `aws-v9-automation` so it lands
   in the cert job's checkout at `{{ base_src }}/hclnow-client-hxtx/`.
   Confirm the AWX project for `aws-v9-automation` is set to update
   submodules on sync, then sync it.
2. **TX job template** — create `Import Certificate to ACM in AWS - TX` as a
   copy of the existing cert JT (same project, same playbook
   `awsacmcertimportauto.yml`). Keep prompt-on-launch settings identical.
   Point it at your test SCM branch while validating.
3. **Read-only guard** (recommended, one line in `tasks/aws-cert-import.yml`):
   in the *Build certificate information dictionary* task, wrap `renewal_due`:

   ```yaml
   renewal_due: "{{ false if (acm_report_only | default(false) | bool) else ( ...existing expression... ) }}"
   ```

   The generator sets `acm_report_only: true` on TX nodes in Phase 1, so this
   guarantees no k8s secret pull and no ACM import even if a TX cert is inside
   its renewal window. Default `false` ⇒ zero change for every other client.

## Source files this reads

| File | Provides | Read by |
|---|---|---|
| `playbooks/awx/kube/common/vars/client-mapping.yml` | `client_parent.tx.clients` (client list) and `client_parent.tx.git_project` (TX repo dir) | generator |
| `playbooks/dns/cloud-cert-vars.yml` | `namespace` (secret namespace), `valid_suffix` | job |
| `<git_project>/hclsoftware-cloud-dns-<test\|prod>.yml` | `dns:` records, combined across TX clients | job (path passed by the generator) |

Override any of them with `acm_client_mapping_file`, `acm_tx_vars_file`,
`acm_tx_dns_file` / `acm_tx_dns_env`. If the init playbook already loads the
mapping, the in-scope `client_parent` var is used and no file read happens.

## First test — hxsa on the test tower

`hxsa` is in the TX client list for exactly this purpose, and its test records
live in the **test** DNS file (e.g. `cname-autotest.hxsa.now.hclsoftware.cloud.`
→ secret `cname-autotest-tls`). On a test tower the other TX clients' inventories
usually don't exist, and a missing inventory aborts the build before links are
created — so scope the run to hxsa:

```yaml
awx_type: <your usual value>
acm_cert_source: tx
acm_cert_env_whitelist: ["hxsa/awsv9m"]        # only this env
acm_tx_dns_env: "test"                          # hxsa records are in the test file
acm_workflow_name_override: "TEST renew certs - aws - tx clients"
acm_test_workflow_delete: true                  # clean rebuild
acm_scm_branch_override: "<your test branch>"   # TX nodes run your branch

awx_type: awx_test
acm_cert_source: tx
acm_tx_job_template: "Import Certificate to ACM in AWS"   # reuse existing JT --> about this its not required right as by default our generator picks this JT
acm_cert_env_whitelist: ["hxsa/awsv9m"]
acm_tx_dns_env: "test"
acm_workflow_name_override: "TEST renew certs - aws - tx clients"
acm_test_workflow_delete: false ---> because i dont want to delete existing Workflow
acm_scm_branch_override: "sum-cert-alert-aws" 
```

Expected result: a workflow with a single `node-hxsa-awsv9m` branch converging
on the digest node, that node's extra_data carrying `acm_certs_source: tx`,
`acm_tx_dns_file` ending in `-test.yml`, and `acm_report_only: true`.

Then widen: drop the whitelist (or list the other TX clients) once their
inventories exist on the tower you're testing against.

## Build the TX workflow

Launch the generator JT (`Initialize resources in awx`) with:

```yaml
awx_type: <your usual value>
acm_cert_source: tx
```

That is the whole invocation — client selection comes from the file, not from
extra vars. Useful additions:

```yaml
acm_test_workflow_delete: true          # clean rebuild of the TX workflow
acm_scm_branch_override: "<test branch>" # pin TX nodes to your branch
acm_tx_dns_file: "/runner/project/hclnow-client-hxtx/hclsoftware-cloud-dns-test.yml"
acm_workflow_name_override: "TEST renew certs - aws - tx clients"
```

Defaults if you pass nothing extra: workflow `renew certs - aws - tx clients`,
JT `Import Certificate to ACM in AWS - TX`, DNS file
`{{ base_src }}/hclnow-client-hxtx/hclsoftware-cloud-dns-prod.yml`,
`acm_report_only: true`.

## Verify before running

- Node set = one branch per TX client (hxtx, mosa, txle — and `hxsa` while it
  remains in `dns_sub_domains`), envs chained, all converging on the digest
- Digest node: convergence **ALL**, `expected_reports` present
- A TX node's extra_data contains `acm_certs_source: tx`, `acm_tx_dns_file`,
  `acm_report_only: true`
- The **main** workflow is untouched (different name, same node set as before)

## Run it

Launch the TX workflow manually. In a TX client's job log, look for
**"Show TX certificates derived for this client env"** — it prints how many DNS
records were read and the derived secret names, e.g.:

```
TX source: 8 DNS record(s) read, 7 certificate(s) derived for hxtx/np
-> ['qa-custa-tls', 'qa-custb-tls', 'dev-custa-tls', ...]
```

Then check the digest email. Every TX domain appears with its ACM status;
domains with no imported ACM certificate show as `not_found_in_acm`.

## What the run tells you

| Observation | Meaning |
|---|---|
| Derived secret names match `kubectl get secrets -n hclnow-certs` | naming rule confirmed → Phase 2 can proceed |
| A domain shows `not_found_in_acm` | no imported ACM cert for that hostname — either genuinely absent, or the cert is AMAZON_ISSUED |
| A client's branch derives 0 certificates | no records for that client in the chosen DNS file (expected for `txle` today) |
| `no report` row for a TX env | job failed before publishing — check that env's job log |

## Known gaps (Phase 2)

1. **Env mapping** — the DNS file has no notion of `client_env`, so if a TX
   client has several envs, every env currently derives the *same* cert list.
   Harmless while read-only; must be resolved before renewal (one cluster per
   client / prefix convention / explicit field in the TX repo).
2. **Wildcard records** are skipped — `*.client…` would derive an invalid k8s
   secret name. None seen so far; needs a rule if they appear.
3. **`hxsa` in `dns_sub_domains`** — it will get a branch in the TX workflow.
   If hxsa also has `refresh_certs: true` anywhere, it is in both workflows.
   Remove it from the manifest when testing is done. Note
   `acm_tx_exclude_from_main` is deliberately **off** for this reason —
   enabling it today would drop hxsa from the main production workflow.
4. **Digest recipients** — TX workflow currently emails the standard cert
   group. Add the TX team's address once confirmed.

```yml
   ---
# SNIPPET for aws-client-vars-base.yml — not a complete file.
# Add smtp_emails_certs_by_group next to the existing smtp_emails_certs.

# ACM certificate list to check/renew overridden in client env tools.yml
certificates: []

# Default recipients — unchanged. Used by:
#   * every immediate alert (import failure / chain-verify failure)
#   * the digest of any workflow that does NOT pass acm_cert_group
smtp_emails_certs: >-
  existing-cert-group@hcl.com

# Per-client-group digest recipients, keyed to match client_parent in
# common/vars/client-mapping.yml. The workflow's digest node passes
# acm_cert_group (e.g. 'tx'); the digest playbook then uses:
#
#   smtp_emails_certs_by_group[acm_cert_group] | default(smtp_emails_certs)
#
# so an unlisted or absent group silently falls back to the default list.
#
# TX policy: the TX team acts on alerts, the current cert group is kept on
# the list for monitoring — so BOTH addresses belong here.
smtp_emails_certs_by_group:
  tx: >-
    tx-team@hcl.com,existing-cert-group@hcl.com

# NOTE: immediate failure alerts inside the cert job still use
# smtp_emails_certs — they are per-cert and fire before the digest node
# exists. If TX hard failures should also page the TX team directly, that is
# a separate change in tasks/aws-cert-import.yml (the two `mail:` tasks).
```
