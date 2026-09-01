# cyberark-api-account-management

Ansible role that idempotently creates, updates, or deletes a CyberArk Privilege Cloud account and manages its credentials via the REST API.

Designed to be called after the [`cyberark_api_authentication`](https://github.com/TobyAnscombe/cyberark-api-management) role, which produces the `cyberark_token` required here.

---

## Requirements

- Ansible 2.14+
- `cyberark_token` present on the play (produced by `tobyanscombe.cyberark_api_authentication`)
- Network access to `https://<subdomain>.privilegecloud.cyberark.cloud`

---

## Role Variables

### Required

| Variable | Description |
|---|---|
| `cyberark_token` | Bearer token from `cyberark_api_authentication` |
| `cyberark_subdomain` | Privilege Cloud subdomain |
| `cyberark_account_safe` | Safe the account lives in |
| `cyberark_account_platform_id` | Platform ID (e.g. `WinDomain`, `UnixSSHKeys`) |
| `cyberark_account_username` | Account username on the target system |

### Account identity

| Variable | Default | Description |
|---|---|---|
| `cyberark_account_name` | `""` | Display name shown in the vault |
| `cyberark_account_address` | `""` | Target hostname, IP, or domain |
| `cyberark_account_state` | `present` | `present` or `absent` |

### Credential

| Variable | Default | Description |
|---|---|---|
| `cyberark_account_secret_type` | `password` | `password` or `key` |
| `cyberark_account_secret` | `""` | Vault-encrypted credential; write-only on the API |
| `cyberark_account_secret_update` | `false` | Push a new credential on every run when `true`; on create only when `false` |
| `cyberark_account_secret_allow_empty` | `false` | Treat an empty `cyberark_account_secret` as an explicit blank credential instead of "no change". Only valid when `cyberark_account_secret_type` is `key`. |

### Platform properties

| Variable | Default | Description |
|---|---|---|
| `cyberark_account_platform_properties` | `{}` | Extra properties created or updated under `platformAccountProperties`; undeclared properties are preserved and accepted keys vary by platform |

### CPM management

| Variable | Default | Description |
|---|---|---|
| `cyberark_account_automatic_management_enabled` | `true` | Let CyberArk CPM rotate the credential automatically |
| `cyberark_account_manual_management_reason` | `""` | Required when `automatic_management_enabled` is `false` |

### PSM remote machine access

| Variable | Default | Description |
|---|---|---|
| `cyberark_account_remote_machines` | `""` | Semicolon-separated list of machines this account may connect to via PSM. Empty = field omitted from API body (no restriction). |
| `cyberark_account_access_restricted_to_remote_machines` | `false` | `true` enforces the list as an allowlist; `false` records preferred machines without blocking others. Has no effect when `remote_machines` is empty. |

### Reconciliation account

| Variable | Default | Description |
|---|---|---|
| `cyberark_account_reconcile_safe` | `""` | Safe containing the reconcile account to link |
| `cyberark_account_reconcile_account_name` | `""` | Display **name** of the reconcile account (its `name` field in the vault, not its username) |

Both must be set together, or both left empty. Linking is additive only — the role does not unlink an existing reconcile account when these are cleared back to `""`.

### Other

| Variable | Default | Description |
|---|---|---|
| `cyberark_validate_certs` | `true` | Validate TLS certificates on API calls |
| `cyberark_no_log` | `true` | Suppress log output on tasks that handle credentials or the bearer token. Override to `false` temporarily to debug a failing task. |

---

## Output Facts

| Fact | Description |
|---|---|
| `cyberark_account_id` | GUID of the created or updated account |
| `cyberark_account_detail` | Full account object returned by the API |

---

## Dependencies

No hard Ansible Galaxy `meta/main.yml` dependency — the role only asserts that `cyberark_token` is already defined and non-empty. Install `cyberark_api_authentication` via `requirements.yml` and call it before this role (or supply `cyberark_token` by any other means):

```yaml
roles:
  - name: cyberark_api_authentication
    src: https://github.com/TobyAnscombe/cyberark-api-management
    version: v1.1.0
  - name: cyberark_account_management
    src: https://github.com/TobyAnscombe/cyberark-api-account-management
    version: v1.5.0
```

---

## Example Playbook — single account

```yaml
- name: Manage CyberArk account
  hosts: localhost
  gather_facts: false

  vars:
    cyberark_identity_tenant: "YOUR_TENANT_ID"
    cyberark_subdomain: "YOUR_SUBDOMAIN"

    cyberark_account_safe: "my-application-safe"
    cyberark_account_platform_id: "WinDomain"
    cyberark_account_address: "corp.example.com"
    cyberark_account_username: "svc_myapp"
    cyberark_account_name: "corp.example.com-svc_myapp"
    cyberark_account_state: present

  roles:
    - cyberark_api_authentication
    - cyberark_account_management

  tasks:
    - name: Show account detail
      debug:
        var: cyberark_account_detail
```

## Example Playbook — SSH key account (Ansible-managed rotation)

```yaml
- name: Vault Linux SSH key account
  hosts: localhost
  gather_facts: false

  vars:
    cyberark_identity_tenant: "YOUR_TENANT_ID"
    cyberark_subdomain: "YOUR_SUBDOMAIN"

    cyberark_account_safe: "linux-safe"
    cyberark_account_platform_id: "UnixSSHKeys"
    cyberark_account_address: "corp.example.com"
    cyberark_account_username: "john.smith"
    cyberark_account_name: "corp.example.com-john.smith"
    cyberark_account_state: present
    cyberark_account_automatic_management_enabled: false
    cyberark_account_manual_management_reason: "SSH key rotation managed by Ansible"
    cyberark_account_secret_type: key
    # cyberark_account_secret: !vault |
    #   $ANSIBLE_VAULT;1.1;AES256
    #   ...

    # To vault this account before a key is available (or to explicitly clear
    # the key), set:
    #   cyberark_account_secret: ""
    #   cyberark_account_secret_allow_empty: true

  roles:
    - cyberark_api_authentication
    - cyberark_account_management
```

## Example Playbook — linked to a reconciliation account

```yaml
- name: Vault Windows account with reconciliation
  hosts: localhost
  gather_facts: false

  vars:
    cyberark_identity_tenant: "YOUR_TENANT_ID"
    cyberark_subdomain: "YOUR_SUBDOMAIN"

    cyberark_account_safe: "windows-safe"
    cyberark_account_platform_id: "WinDomain"
    cyberark_account_address: "corp.example.com"
    cyberark_account_username: "svc_myapp"
    cyberark_account_name: "corp.example.com-svc_myapp"
    cyberark_account_state: present

    cyberark_account_reconcile_safe: "reconciliations"
    cyberark_account_reconcile_account_name: "Operating System-SVC_Reconcile-corp.example.com-SVC_Reconcile"

  roles:
    - cyberark_api_authentication
    - cyberark_account_management
```

## Example Playbook — multiple accounts

See [`examples/multiple_accounts.yml`](examples/multiple_accounts.yml).

---

## Notes

- **Account lookup**: the role resolves the account GUID on every run via a `GET` filtered by `safeName` and then an exact client-side match on `name`. Account names are unique per safe in Privilege Cloud.
- **Update method**: the Accounts API uses JSON Patch (`PATCH`), not `PUT`. The role computes the diff first and skips the PATCH entirely when nothing has changed.
- **Remote machines — pre-clear**: when `remote_machines` needs updating, the role issues a clear PATCH (`remoteMachines: ""`) immediately before the update PATCH. This is required because portal-introduced duplicate machine entries cause a standard PATCH to be silently ignored (the API returns 200 but applies no change). The clear resets the field to a clean state so the subsequent set takes effect.
- **secretManagement fields**: `automaticManagementEnabled` and `manualManagementReason` are only stored and expressed by CyberArk when the safe has a CPM assigned (`managing_cpm` set to a non-empty value). In safes with no CPM these fields are effectively ignored by the platform.
- **Secrets**: the API never returns the stored credential. `cyberark_account_secret_update: false` (default) means the credential is only pushed on create, keeping runs idempotent with respect to the stored value.
- **Provision summary**: the role appends to a `_cyberark_provision_summary` fact on the play. Each changed account adds a row with `action: update` and a `detail` string showing `path: "current" → "proposed"` for each changed field. Callers can render this fact (e.g. as an HTML report) at the end of the playbook.
- **Reconciliation account**: the desired reconcile account name is resolved to an ID (`GET /Accounts?filter=safeName eq ...`, exact client-side name match, `limit=1000` to cover reconcile safes past the API's 50-record default page). The account's current link is read back (`GET /ExtendedAccounts/{id}/LinkedAccounts`, `extraPasswordIndex: 3`), and — because the linked-account ID on that response isn't assumed to be directly comparable to the ID format `GET /Accounts` returns — the current link is resolved through a further `GET /Accounts/{id}` so both sides of the comparison come from the same endpoint. `POST /Accounts/{id}/LinkAccount` only fires when the link is actually missing or different; dry-run previews are computed from the same comparison, and the provision summary shows the change as `"old safe/name" → "new safe/name"` (or `"(none)" → "new safe/name"` for a first-time link). Re-linking to a *different* reconcile account replaces the existing link. There is currently no supported way to remove a reconcile account link via this role. If the reconcile account name can't be resolved, the role fails on a live run; a dry-run instead prints a warning and skips the preview for that account.

---

## License

MIT
