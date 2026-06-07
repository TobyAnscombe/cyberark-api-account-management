# cyberark-api-account-management

Ansible role that idempotently creates, updates, or deletes a CyberArk Privilege Cloud account and manages its credentials via the REST API.

Designed to be called after the [`cyberark_auth`](https://github.com/TobyAnscombe/cyberark-api-management) role, which produces the `cyberark_token` required here.

---

## Requirements

- Ansible 2.9+
- `cyberark_token` present on the play (produced by `tobyanscombe.cyberark_auth`)
- Network access to `https://<subdomain>.privilegecloud.cyberark.cloud`

---

## Role Variables

### Required

| Variable | Description |
|---|---|
| `cyberark_token` | Bearer token from `cyberark_auth` |
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

### Platform properties

| Variable | Default | Description |
|---|---|---|
| `cyberark_account_platform_properties` | `{}` | Extra properties passed under `platformAccountProperties` — keys vary by platform |

### CPM management

| Variable | Default | Description |
|---|---|---|
| `cyberark_account_automatic_management_enabled` | `true` | Let CyberArk CPM rotate the credential automatically |
| `cyberark_account_manual_management_reason` | `""` | Required when `automatic_management_enabled` is `false` |

### Other

| Variable | Default | Description |
|---|---|---|
| `cyberark_validate_certs` | `true` | Validate TLS certificates on API calls |

---

## Output Facts

| Fact | Description |
|---|---|
| `cyberark_account_id` | GUID of the created or updated account |
| `cyberark_account_detail` | Full account object returned by the API |

---

## Dependencies

None declared. Install `cyberark_auth` via `requirements.yml` and call it before this role:

```yaml
roles:
  - name: cyberark_auth
    src: https://github.com/TobyAnscombe/cyberark-api-management
    version: main
  - name: cyberark_account_management
    src: https://github.com/TobyAnscombe/cyberark-api-account-management
    version: main
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
    - cyberark_auth
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

  roles:
    - cyberark_auth
    - cyberark_account_management
```

## Example Playbook — multiple accounts

See [`examples/multiple_accounts.yml`](examples/multiple_accounts.yml).

---

## Notes

- **Account lookup**: accounts have no human-readable unique key — the role resolves the GUID on every run via a `GET` filtered by `safeName`, `userName`, and `address`.
- **Update method**: the Accounts API uses JSON Patch (`PATCH`), not `PUT`.
- **Secrets**: the API never returns the stored credential. `cyberark_account_secret_update: false` (default) means the credential is only pushed on create, keeping runs idempotent with respect to the stored value.

---

## License

MIT
