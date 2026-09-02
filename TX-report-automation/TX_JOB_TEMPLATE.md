# TX job template — `Import Certificate to ACM in AWS - TX`

The TX JT runs the **same playbook** as the existing cert JT. It exists for
isolation (RBAC, credentials, SCM branch, blast radius), not for different
logic — every TX/non-TX difference is driven by node `extra_data`.

## How to create it

In `gcp-shared-services/playbooks/awx/kube/common/jobs/job-vars.yml`, copy the
existing `Import Certificate to ACM in AWS` stanza and change **only** the two
fields below. Copying (rather than writing fresh) is deliberate — it inherits
project, playbook, EE, credentials, `job_tags`, `limit` and become settings,
which must stay identical.

```yaml
  - name: "Import Certificate to ACM in AWS - TX"
    description: >-
      Certificate renewal for TX-group clients (hxtx, mosa, txle). Same
      playbook as the standard cert job; TX behaviour is driven by node
      extra_data (acm_certs_source, acm_cert_group, acm_tx_dns_file).
      Separate template so the TX team can be granted Execute without
      access to the DevOps-owned job.
    # ---- everything below copied verbatim from the existing cert JT ----
    # project / playbook: awsacmcertimportauto.yml
    # execution_environment, credentials, become settings
    # job_tags: certs
    # limit: "<same bastion pattern as the existing JT>"
```

## Fields that MUST be true (verify after copying)

| Field | Why |
|---|---|
| `ask_extra_vars: true` | Without it AWX drops the node's `acm_certs_source` / `acm_cert_group` / `acm_tx_dns_file` and TX sourcing silently never happens — the job falls back to `tools.yml` and looks green |
| `ask_inventory: true` | A workflow node can only set its inventory if the JT allows it |
| `ask_scm_branch_on_launch: true` | Needed for `acm_scm_branch_override` while testing |
| `limit` | Same bastion pattern as the existing cert JT — TX runs against the same bastion inventories |

## After creating it

Drop `acm_tx_job_template` from the generator launch vars — TX mode defaults to
`Import Certificate to ACM in AWS - TX`.

## RBAC (when handing over to the TX team)

- AWX Team `ext-cert-team`
- **Execute** on `Import Certificate to ACM in AWS - TX`
- **Use** on only `inventory_client_{hxtx,mosa,txle}_*`
- No access to the generator, the workflows, or the digest JT
